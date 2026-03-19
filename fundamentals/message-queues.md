# Message Queues

> Message queues decouple producers from consumers, absorb traffic spikes, and enable async processing, but the architectural difference between a queue and a log fundamentally changes what your system can do with the messages.

## Core Concept

A message queue is a form of asynchronous service-to-service communication. Producers send messages to a queue; consumers read from the queue and process them. This decoupling provides:

- **Temporal decoupling**: Producer and consumer don't need to be available simultaneously.
- **Rate decoupling**: Consumer can process at its own rate; the queue absorbs bursts.
- **Failure isolation**: If the consumer crashes, messages wait in the queue rather than being lost.

But not all message systems are the same. The architectural choice between a **queue** (messages are consumed and deleted) and a **log** (messages are retained and replayable) determines what patterns you can build on top.

## Queue vs Log: The Fundamental Architectural Difference

### Queue (RabbitMQ, SQS, ActiveMQ)

A queue is a buffer. Messages are delivered to one consumer, and once acknowledged, they are deleted. The queue has no memory of delivered messages.

**Characteristics**:
- Each message is processed by exactly one consumer (in a competing consumers model).
- Once consumed and acknowledged, the message is gone.
- Ordering is generally FIFO but not strictly guaranteed across concurrent consumers.
- No ability to replay past messages.
- Well-suited for work distribution: divide a batch of jobs across N workers.

**Best for**: Task queues, job processing, work distribution. "Send an email," "resize this image," "process this order."

### Log (Kafka, Kinesis, Apache Pulsar)

A log is a persistent, ordered, append-only sequence of records. Messages are retained after consumption. Consumers track their own position (offset) in the log.

**Characteristics**:
- Messages are retained for a configurable period (days, weeks, or indefinitely).
- Multiple independent consumer groups can read the same log independently, each at their own position.
- Any consumer can replay the log from any offset (time-travel debugging, backfill, reprocessing).
- Ordering is guaranteed within a partition.
- Built for high throughput (millions of messages/second).

**Best for**: Event streaming, event sourcing, CDC (change data capture), audit logs, feeding multiple downstream systems from a single event stream.

### The Decision

| Need | Choose |
|---|---|
| Distribute work across workers, message consumed once | Queue (SQS, RabbitMQ) |
| Multiple independent consumers of same data | Log (Kafka, Kinesis) |
| Replay past events | Log |
| Real-time stream processing | Log |
| Simple task queue with low throughput | Queue |
| Audit trail / event sourcing | Log |
| Fan-out to many subscribers | Either (Log is more natural for many consumers) |

## Delivery Semantics

### At-Most-Once
A message is delivered zero or one times. If the consumer crashes after receiving the message but before processing it, the message is not redelivered.

**Implementation**: The broker marks the message as delivered when it is sent, before the consumer acknowledges processing. Crash between receive and process = message lost.

**Cost**: Potential message loss.

**When appropriate**: Metrics, telemetry, analytics events — workloads where losing some data is acceptable and redelivery would cause worse problems (double-counting metrics, duplicate events in analytics).

### At-Least-Once
A message is delivered one or more times. If the consumer crashes before acknowledging, the broker redelivers. Guarantees no message loss, but the consumer may process the same message multiple times.

**Implementation**: The consumer explicitly acknowledges (ACKs) messages after successful processing. The broker only marks a message as consumed after receiving the ACK. If the consumer crashes, the broker redelivers.

**Cost**: Consumer must be idempotent — processing the same message twice must produce the same result as processing it once.

**When appropriate**: The default and most common model. Email sending, payment processing (with idempotency), order fulfillment. Almost always preferable to at-most-once unless you genuinely cannot afford duplicate processing.

**Idempotency implementation**: Use a database record or a distributed deduplication store keyed by message ID. Before processing, check if the message ID has been seen. After processing, record it.

### Exactly-Once
A message is delivered exactly one times — no losses, no duplicates. The hardest guarantee to provide.

**Implementation**: Requires coordination between the broker and the consumer's downstream system. Kafka's implementation uses:
- **Idempotent producers**: Each producer is assigned an ID; the broker deduplicates retried writes from the same producer.
- **Transactional APIs**: Kafka transactions allow atomic writes across multiple partitions + consumer offset commits. Either all writes succeed and the offset advances, or none of them do.

**Cost**: Significantly higher latency than at-least-once. Requires both the producer and consumer to use the transactional API correctly. Errors in the surrounding code (writing to a DB after the Kafka transaction but outside of it) break exactly-once guarantees.

**When appropriate**: Financial transactions, billing systems, anything where duplicate processing has direct monetary impact. Even here, many teams prefer at-least-once with idempotency over exactly-once due to the complexity.

### Practical Recommendation
Design for **at-least-once with idempotent consumers**. This is simpler to implement correctly than exactly-once, handles all practical scenarios, and is the approach used by most large-scale systems.

## Consumer Groups and Partitioning

### Kafka Partitions
A Kafka topic is divided into partitions — ordered, immutable sequences of records. Each partition is stored on a specific broker (with replicas on others).

**Ordering guarantee**: Within a single partition, messages are strictly ordered by offset. Across partitions, there is no ordering guarantee.

**Partition assignment**: You control which partition a message goes to via the message key. Messages with the same key always go to the same partition (consistent hashing on the key). This is how you preserve ordering for a specific entity (e.g., all events for user 123 in order).

### Consumer Groups
A consumer group is a set of consumers that collectively consume a topic. Kafka assigns each partition to exactly one consumer within the group. If a group has 3 consumers and a topic has 6 partitions, each consumer handles 2 partitions.

**Horizontal scaling**: Add more consumers to the group (up to the number of partitions) to increase throughput.

**Independent consumption**: Multiple consumer groups can consume the same topic independently. Group A is a fraud detection service; Group B is an analytics pipeline. Both consume the same event stream at their own rate and offset. Neither affects the other.

**Consumer group rebalance**: When a consumer joins or leaves the group, Kafka triggers a rebalance — redistributing partitions across the group. During rebalance, consumption pauses briefly. Design for this: consumers must commit offsets before a rebalance, and processing must be resumable from any committed offset.

## Backpressure

### What It Is
Backpressure is the mechanism by which a slow consumer signals to the upstream system to slow down. Without backpressure, a fast producer can overwhelm a slow consumer, filling buffers until the system runs out of memory or starts dropping messages.

### How Queues Help
A queue naturally absorbs burst traffic. If the consumer is processing at 100 msg/s and the producer is sending at 1000 msg/s, the queue grows. This is queue-based backpressure — the queue grows instead of the system failing. The consumer catches up during quieter periods.

**Risk**: Unbounded queue growth. If the consumer is consistently slower than the producer, the queue grows without bound. Eventually you hit memory or disk limits. The queue itself becomes a failure mode.

**Mitigations**:
- **Queue depth alerts**: Alert when the queue depth exceeds a threshold. This is a leading indicator of a consumer bottleneck.
- **Consumer autoscaling**: Scale out consumers based on queue depth (SQS + Lambda autoscaling, Kubernetes KEDA).
- **Producer rate limiting**: Apply backpressure upstream — if the queue is too deep, the producer slows down or returns errors to callers.
- **Message TTL**: Messages that are too old are discarded. Acceptable for some workloads (stale notifications, old metrics).

### Reactive Streams Backpressure
In streaming systems (Kafka Streams, Akka Streams), backpressure is propagated upstream: a slow sink signals to the source to slow production. This prevents unbounded buffering.

## Dead Letter Queues (DLQ)

### What They Are
A dead letter queue is a separate queue where messages are moved when they fail to be processed after N attempts. Instead of blocking the main queue or silently dropping failed messages, the DLQ captures them for investigation.

### Why They Matter
Without a DLQ:
- A malformed message ("poison pill") blocks the queue forever — no subsequent messages are processed.
- Failed messages are silently discarded (at-most-once) or cause infinite retry loops that block the queue.

With a DLQ:
- Failed messages after N retries are moved to the DLQ.
- The main queue continues processing other messages.
- Operations team can inspect the DLQ, fix the root cause, and replay messages.

### Implementation
- **SQS**: Set `maxReceiveCount` on the queue's redrive policy. After N deliveries without deletion, the message is moved to the configured DLQ.
- **Kafka**: No native DLQ, but the pattern is common: a consumer that fails to process a message after N retries publishes it to a `topic-name.DLQ` topic.
- **RabbitMQ**: Dead letter exchanges (DLX) + message TTL.

### DLQ Operational Discipline
Monitor DLQ depth. A growing DLQ is an alarm: messages are failing to process. Have a runbook for common DLQ failure modes. Replay mechanisms must be tested, not improvised during incidents.

## Message Ordering Guarantees

- **Kafka**: Strict ordering within a partition. No ordering across partitions. To preserve per-entity ordering, use the entity ID as the message key — all messages for that entity land in the same partition.
- **SQS Standard**: Best-effort ordering. No ordering guarantees. Messages may arrive out of order and may be delivered more than once.
- **SQS FIFO**: Strict FIFO ordering within a message group. Exactly-once processing (within 5 minutes). Lower throughput than Standard queues (up to 3,000 msg/s with batching vs unlimited for Standard).
- **RabbitMQ**: FIFO ordering per queue with a single consumer. Ordering breaks with multiple consumers competing on the same queue.

**Practical note**: If you need strict ordering AND high throughput, Kafka with partitioning by key is the right architecture. SQS FIFO is suitable for lower-throughput ordered workloads.

## Push vs Pull Consumers

### Push (Broker Delivers to Consumer)
The broker proactively delivers messages to registered consumers. The consumer must be available and ready to receive.

**Examples**: SQS with Lambda triggers, RabbitMQ consumers with a channel push model, webhooks.

**Advantages**: Low latency from message arrival to processing. Simpler consumer logic (no polling loop).

**Disadvantages**: Consumer must be able to accept the delivery rate. If consumer is slow, the broker continues to push, overwhelming it. Backpressure is harder to implement.

### Pull (Consumer Fetches from Broker)
The consumer polls the broker for messages at its own pace.

**Examples**: Kafka consumers, SQS consumers polling via `ReceiveMessage`, Kinesis consumers.

**Advantages**: Consumer controls its own processing rate. Natural backpressure — if consumer is slow, it just polls less frequently. No risk of being overwhelmed by the broker.

**Disadvantages**: Polling overhead (especially for low-throughput queues with frequent empty responses). Slightly higher latency (time between message arrival and next poll).

**Kafka's approach**: Consumers pull in batches. Configurable `max.poll.records` and `fetch.min.bytes` control batch size and wait time. Long polling (`fetch.wait.max.ms`) reduces empty poll overhead.

## When to Use a Queue vs a Direct API Call

Use a queue when:
- The operation can be async (user doesn't need the result immediately).
- You want to absorb traffic spikes (smooth load over time).
- The operation might fail and needs retry logic.
- Multiple systems need to react to the same event.
- The downstream system is slow or unreliable.

Use a direct API call when:
- The user needs the result to continue (synchronous user request).
- The operation must complete before the response is returned.
- Low latency is required (queue adds 10–100ms).
- The operation is simple and the caller and callee are equally reliable.

**Example**: "Send a confirmation email" should be queued — it's async, can be retried, and the user doesn't need to wait for it. "Check if the username is available" should be a direct API call — the result is needed immediately.

## Common Patterns

### Work Queue (Competing Consumers)
A single queue with multiple consumers. Each message is processed by one consumer. Used for distributing jobs: video encoding, email sending, report generation.

### Pub/Sub (Publish-Subscribe)
A producer publishes to a topic. Multiple independent subscribers receive every message. Used for event notification: "User signed up" published once, consumed by email service, analytics service, onboarding service.

### Fan-Out
A single message triggers processing in many parallel consumers. Similar to pub/sub but emphasizes parallelism. SNS + SQS: a single SNS topic fans out to N SQS queues, each consumed by a different service.

### Event Streaming
Continuous stream of events processed in real time or near-real time. Kafka Streams, Flink, Spark Streaming. Used for: fraud detection on transaction streams, real-time dashboards, CDC from database to downstream systems.

## Kafka Deep Dive

### Topics, Partitions, Offsets
- **Topic**: A named stream of records.
- **Partition**: A topic is divided into N partitions. Each partition is an ordered log.
- **Offset**: The position of a record within a partition. Consumers track their offset.
- **Consumer group offset commit**: Consumers periodically commit their offset to Kafka (or to a custom store). If a consumer crashes and restarts, it resumes from the committed offset.

### Replication
Each partition has a leader and N-1 followers. Writes go to the leader; followers replicate. ISR (In-Sync Replicas) tracks which replicas are fully caught up. `acks=all` requires all ISR members to acknowledge before the producer gets a success response.

### Log Compaction
Kafka can compact topics — for each key, only the latest value is retained. Older messages with the same key are deleted. This enables Kafka to act as a changelog: consuming a compacted topic from the beginning gives you the current state of every key.

**Use case**: CDC topics, configuration topics, database-like state in a Kafka topic.

### Consumer Group Rebalance and Session Timeout
- `session.timeout.ms`: If a consumer doesn't send a heartbeat within this window, it is considered dead and its partitions are reassigned. Too short = false positives during GC pauses. Too long = slow failure detection.
- `max.poll.interval.ms`: If a consumer doesn't call `poll()` within this interval, it is considered stuck. Set this long enough for your slowest processing job.

## Pitfalls

### Large Messages
Kafka is not designed for large payloads. Default `message.max.bytes` is 1MB. Sending multi-MB messages causes broker memory pressure. Use the "claim check" pattern: store the large object in S3, put the S3 reference in the queue.

### Poison Pills
A malformed message that causes every consumer to crash when processing it. The consumer restarts, picks up the same message, crashes again. Infinite loop. Fix: DLQ after N retries, message deserialization error handling that skips and logs rather than crashing, graceful error handling in consumer.

### Unbounded Queue Growth
Producer consistently outpaces consumer. Queue grows unbounded. Fix: consumer autoscaling on queue depth, producer rate limiting, message TTL for time-sensitive messages.

### Committing Offsets Too Early
Committing the offset before the message is fully processed means a crash after commit but before completion loses that message. If you must use at-least-once, commit after processing, not before.

### Schema Evolution Without Compatibility
Changing the schema of a Kafka topic message in a backward-incompatible way breaks existing consumers. Use a schema registry (Confluent Schema Registry) with backward or forward compatibility enforcement.

## SRE Lens: What This Looks Like in Production

### Kafka Consumer Lag Alert Triggering
An SRE sees a consumer lag alert: `consumer_group_orders_processor` is 500,000 messages behind on topic `orders`. The consumer service had an OOM crash 30 minutes ago and restarted, but it's processing slowly due to a downstream dependency (payment service) having high latency. Fix: scale out consumers, investigate payment service latency, consider adding a circuit breaker.

### DLQ Growing During Downstream Outage
A downstream API that a consumer depends on has an outage. Consumer retries exhaust the maxReceiveCount. Messages flood into the DLQ. When the downstream recovers, operations team must replay the DLQ. Fix: automate DLQ replay on downstream recovery, set up DLQ depth alerting, design for graceful retry with exponential backoff.

### Kafka Partition Rebalance Causing Duplicate Processing
A consumer group processes payments. A consumer GC pause exceeds `session.timeout.ms`. Kafka reassigns the partition. The paused consumer resumes and also continues processing. Two consumers now process the same partition for a brief window — duplicate payment processing. Fix: tune `session.timeout.ms` and `max.poll.interval.ms` to be compatible with GC pause durations, implement idempotency at the payment processor.

## Common Misconceptions

- **"Kafka replaces all other queues"**: Kafka is excellent for event streaming and multi-consumer logs. SQS/RabbitMQ are simpler and cheaper for basic work queues. Don't operate a Kafka cluster for a 100 msg/day task queue.
- **"Exactly-once means no bugs"**: Exactly-once in Kafka is exactly-once within the Kafka transaction boundary. If you write to a database outside that transaction, you're back to at-least-once semantics.
- **"A queue is reliable, so I don't need idempotency"**: At-least-once delivery means duplicates are possible. Build idempotent consumers.
- **"Consumer groups scale linearly with consumers"**: You can only have as many active consumers as partitions. Adding more consumers than partitions results in idle consumers. Increase partitions to increase parallelism.
- **"DLQs are a safety net I can ignore"**: A growing DLQ is an operational emergency. Messages in a DLQ represent lost business value. Monitor and act on DLQ growth.

## Key Takeaways

- Queues (SQS, RabbitMQ) delete messages after consumption. Logs (Kafka, Kinesis) retain messages for replay by multiple independent consumers.
- At-most-once: may lose messages. At-least-once: may deliver duplicates, requires idempotent consumers. Exactly-once: complex, high latency, use only where truly required.
- Consumer groups in Kafka allow multiple independent services to consume the same topic. Each group tracks its own offset.
- Backpressure is handled by queue depth — monitor it as a leading indicator of consumer problems.
- DLQs are non-negotiable for production message systems. Monitor them and have a replay process.
- Kafka partitioning by message key preserves ordering for a given entity (e.g., all events for a user go to the same partition).
- Push consumers risk being overwhelmed; pull consumers control their own rate. Kafka uses pull.
- Design for at-least-once with idempotent consumers. It is simpler and more correct than exactly-once in practice.

## References

- Narkhede, N., Shapira, G., Palino, T. — *Kafka: The Definitive Guide* (O'Reilly)
- Kleppmann, M. — *Designing Data-Intensive Applications*, Chapter 11
- Kreps, J. (2013). "The Log: What every software engineer should know about real-time data's unifying abstraction." LinkedIn Engineering Blog.
- AWS SQS documentation — Message lifecycle, DLQ, FIFO queues
- Confluent documentation — Consumer groups, exactly-once semantics

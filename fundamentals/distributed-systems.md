# Distributed Systems

> Distributed systems are multiple computers coordinating to appear as a single coherent system — and nearly every assumption you make about how programs behave locally is wrong in a distributed setting.

## Core Concept

A distributed system is a collection of autonomous computing nodes that communicate via a network and coordinate their actions to provide a unified service. The appeal is clear: horizontal scale, fault tolerance, geographic distribution. The challenge is equally clear: partial failures, asynchrony, and the impossibility of a globally shared state.

Understanding distributed systems is understanding failure modes. Everything that can be wrong about a distributed system differs from local computing in fundamental, non-intuitive ways.

## The 8 Fallacies of Distributed Computing

Originally articulated by Peter Deutsch and James Gosling at Sun Microsystems. Every senior engineer has been burned by at least half of these.

### 1. The network is reliable
**Reality**: Networks drop packets, experience congestion, have link failures, and suffer from misconfigured routing rules. A call to a remote service may return a success, a failure, or simply never return — and you cannot always distinguish "slow" from "down" without a timeout.

**Consequence**: You must design for retries, timeouts, and idempotency. A "fire and forget" RPC to a remote service is not "fire and forget" — it's "fire and hope."

### 2. Latency is zero
**Reality**: Even within a single data center, a network call takes 0.5–1ms. Cross-AZ is 1–2ms. Cross-region is 50–150ms. Compared to memory access (100ns) or L1 cache (1ns), network calls are astronomically slow.

**Consequence**: Chatty protocols — tight loops of small synchronous RPCs — kill performance. Design for batching, async, and minimize the number of round trips per user request.

### 3. Bandwidth is infinite
**Reality**: Network links have capacity limits. Saturating a 10Gbps link with replication traffic or large payloads degrades all other traffic on that link.

**Consequence**: Be explicit about payload sizes. Avoid sending full object snapshots when diffs suffice. Compress data in transit. Monitor egress costs and bandwidth utilization.

### 4. The network is secure
**Reality**: Networks are shared infrastructure. Traffic can be intercepted, man-in-the-middle attacks are real, and internal networks are not inherently trustworthy.

**Consequence**: Use mTLS for service-to-service communication. Don't trust requests just because they're from "inside the network." Zero-trust architecture exists for good reason.

### 5. Topology doesn't change
**Reality**: Nodes are added and removed. IP addresses change. Load balancers cycle backends. Cloud auto-scaling changes the set of instances continuously.

**Consequence**: Service discovery must be dynamic. Hard-coded IPs are a maintenance nightmare. Health checks must be continuous, not one-time.

### 6. There is one administrator
**Reality**: Large systems span teams, organizations, cloud providers, and data centers. Change management is fragmented. A change in one team's service can break another team's dependency with no coordinated notification.

**Consequence**: Design APIs with backwards compatibility. Use explicit versioning. Monitor your dependencies, not just your own service.

### 7. Transport cost is zero
**Reality**: Serialization, deserialization, encryption, and network transfer all have CPU and memory costs. Sending a 1MB payload 10,000 times per second has real infrastructure cost.

**Consequence**: Choose efficient serialization formats (protobuf vs JSON). Understand the cost of your RPC layer. Don't serialize more than you need.

### 8. The network is homogeneous
**Reality**: Networks mix hardware vendors, OS versions, MTU settings, and protocol stacks. IPv4 and IPv6 coexist. Some paths have jumbo frames enabled and others don't.

**Consequence**: Test across network configurations. Don't assume behavior observed in one environment will hold in another.

## Clock Drift and Logical Clocks

### Physical Clocks Are Not Reliable for Ordering

Physical clocks in computers drift. Even with NTP, two servers in the same data center can have clocks that differ by tens of milliseconds. Across data centers, the drift can be hundreds of milliseconds. You cannot use wall-clock timestamps to establish a reliable causal ordering of events across machines.

**Example**: Server A writes record X at T=100. Server B writes record Y at T=99 (but B's clock is 1ms slow). If you order by timestamp, Y appears to come before X. But Y was written after X. This is the fundamental problem with "last-write-wins" timestamp-based conflict resolution.

### Lamport Timestamps

Proposed by Leslie Lamport in 1978. A logical clock that assigns a monotonically increasing integer to each event, tracking causal ordering.

**Rules**:
1. Each process maintains a counter, initialized to 0.
2. Before each event, increment the counter.
3. When sending a message, include the current counter value.
4. When receiving a message, set your counter to `max(local, received) + 1`.

**Property**: If event A causally precedes event B, then `timestamp(A) < timestamp(B)`. But the converse is not necessarily true — a lower timestamp doesn't mean causal precedence if the events are on different processes with no communication.

**Limitation**: Lamport timestamps establish partial ordering. You cannot determine if two events are concurrent or causally related from the timestamp alone.

### Vector Clocks

An extension of Lamport timestamps that captures full causal relationships. Each process maintains a vector of counters, one per process.

**Rules**:
1. Each process `i` maintains `V[i]` incremented on each event.
2. When sending a message, include the full vector.
3. When receiving a message, `V[j] = max(V[j], received[j])` for all `j`, then `V[i]++`.

**Property**: You can determine if two events are concurrent, or if one causally precedes the other, by comparing vectors:
- `A → B` (A causally precedes B): all `V_A[i] <= V_B[i]` and at least one is strictly less.
- `A || B` (concurrent): neither A → B nor B → A.

**Used in**: DynamoDB (for conflict detection), Riak, and CRDTs. When two writes are concurrent (neither causally precedes the other), you have a conflict that requires resolution.

## Leader Election

Many distributed systems need exactly one leader — a coordinator to avoid split-brain decisions. Leader election is the mechanism for choosing and maintaining that leader.

### Why You Need It
- A database primary that accepts writes.
- A job scheduler that runs a cron job exactly once.
- A distributed lock server.
- A partition leader in Kafka.

### Raft

Raft was designed specifically to be understandable (unlike Paxos). It decomposes consensus into three problems: leader election, log replication, and safety.

**Election Process**:
1. Nodes start as followers. If a follower doesn't hear from a leader within an election timeout (randomized, typically 150–300ms), it becomes a candidate.
2. The candidate increments its term and sends `RequestVote` RPCs to all other nodes.
3. A node votes for a candidate if it hasn't voted in this term and the candidate's log is at least as up-to-date as the voter's log.
4. A candidate becomes leader if it receives votes from a majority of nodes.
5. The leader sends heartbeats to prevent new elections.

**Log Replication**: The leader receives all writes, appends them to its log, and replicates them to followers. An entry is committed once acknowledged by a majority.

**Used in**: etcd, CockroachDB, TiKV, Consul, many internal systems at large tech companies.

### Paxos

The original consensus algorithm, notoriously difficult to understand and implement correctly. There are many variants (Basic Paxos, Multi-Paxos, Cheap Paxos).

**Conceptually**: Paxos has two phases — Prepare/Promise and Accept/Accepted. The proposer prepares a proposal with a ballot number, collects promises from a quorum, then sends the accept request. It tolerates up to `f` failures in a cluster of `2f+1` nodes.

**Used in**: Google Chubby, Bigtable's distributed metadata, Spanner (as a component).

**Practical note**: Most new systems use Raft because it's easier to implement correctly and reason about. Paxos shows up more in older systems and research papers.

## Consensus and Why It's Hard

**The FLP Impossibility Result** (Fischer, Lynch, Paterson, 1985): In a fully asynchronous system where even one process may fail, no deterministic algorithm can guarantee reaching consensus.

This is not a proof that consensus is impossible in practice — it's a proof that in a pure asynchronous model you can't guarantee *both* safety (correct decision) and liveness (eventually decides). Real systems work around this by:
- Using timeouts (which introduce synchrony assumptions).
- Using randomization (which breaks determinism in a useful way).
- Assuming partial synchrony (Raft's approach — bounded message delays in practice).

**Byzantine Generals Problem**: A more general form of consensus where some nodes may behave arbitrarily (send wrong messages, lie). Solutions require 3f+1 nodes to tolerate f Byzantine failures. BFT consensus is much more expensive than crash-failure consensus.

## Two-Phase Commit vs Two-Phase Locking (Common Confusion)

These sound similar and are often confused, but they solve completely different problems.

### Two-Phase Locking (2PL) — Concurrency Control

A protocol for controlling concurrent access to shared data within a single database.

**Phases**:
1. **Growing phase**: Acquire locks as needed. Do not release any.
2. **Shrinking phase**: Release locks. Do not acquire any new ones.

**Purpose**: Ensures serializability — transactions execute as if they ran one at a time. Prevents dirty reads, non-repeatable reads, and phantoms.

**Problem**: Can cause deadlocks when two transactions wait for each other's locks.

### Two-Phase Commit (2PC) — Distributed Atomicity

A protocol for atomically committing a transaction across multiple distributed nodes.

**Phases**:
1. **Prepare phase**: The coordinator sends `PREPARE` to all participants. Each participant ensures it *can* commit and writes a prepare record to its WAL. Responds with `YES` or `NO`.
2. **Commit phase**: If all participants voted `YES`, the coordinator sends `COMMIT` to all. Otherwise, sends `ABORT`.

**Problem**: The coordinator is a single point of failure. If the coordinator crashes after participants vote YES but before sending COMMIT, participants are stuck — they hold locks and cannot proceed or roll back without the coordinator's decision. This is called the "blocking" problem of 2PC.

**Used in**: XA transactions (Java EE), distributed SQL systems, Postgres foreign data wrappers.

### 3PC (Three-Phase Commit)
An attempt to make 2PC non-blocking by adding a `PRE-COMMIT` phase. It avoids the blocking problem in certain failure scenarios but is still not safe in network partition scenarios. Rarely used in practice.

## Distributed Transaction Patterns

### 2PC (Two-Phase Commit)
As above. Provides atomicity across multiple services but has blocking behavior under coordinator failure. Suitable for databases in the same availability zone with shared fate.

### Saga Pattern

Break a distributed transaction into a sequence of local transactions, each with a compensating transaction that can undo it.

**Two coordination styles**:
- **Choreography**: Each service publishes events that trigger the next service's action. No central coordinator. Simpler to implement, harder to visualize the overall flow.
- **Orchestration**: A central saga orchestrator tells each service what to do and handles failures. Easier to reason about, creates a central dependency.

**Example — Order Service Saga**:
1. Order Service creates order (local tx).
2. Payment Service charges customer (local tx). On failure → cancel order.
3. Inventory Service reserves item (local tx). On failure → refund payment, cancel order.
4. Shipping Service schedules delivery (local tx). On failure → unreserve inventory, refund, cancel.

**Tradeoff**: Sagas sacrifice atomicity for availability. The system is briefly in inconsistent states between steps. Compensating transactions must be idempotent.

### Outbox Pattern

Solves the "dual write" problem: atomically writing to your database and publishing to a message queue.

**Problem**: A service writes to its DB and then publishes to Kafka. If the service crashes between the DB commit and the Kafka publish, the event is lost.

**Solution**:
1. Write to your DB table AND an `outbox` table in the same local transaction.
2. A separate process (transactional outbox reader) polls the outbox table and publishes messages to the queue.
3. Once successfully published, mark the outbox record as processed.

**Used in**: Debezium (CDC-based outbox), custom implementations in services needing exactly-once semantics for event publishing.

## Byzantine vs Crash Failures

**Crash-Stop Failures**: A node fails by stopping. It stops sending messages, stops responding. All other nodes will eventually detect the failure via timeout.

**Crash-Recovery Failures**: A node crashes and may restart. It may have partially completed state from before the crash. Logs and WALs are designed to handle this.

**Byzantine Failures**: A node behaves arbitrarily — it might send incorrect data, send conflicting messages to different nodes, or actively try to subvert the system. Named after the Byzantine Generals Problem.

**Practical implications**:
- Most distributed databases and consensus algorithms assume crash failures only. This simplifies design enormously.
- Byzantine fault tolerance is required for blockchains, systems with untrusted participants, or safety-critical systems.
- BFT protocols (PBFT, Tendermint) require 3f+1 nodes for f failures — much more expensive than the 2f+1 required for crash-fault tolerance.

## Network Partitions: How They Actually Happen in Production

Partitions are not exotic failures. They happen regularly in large-scale systems:

**Physical cable failure**: A fiber cut in a data center severs connectivity between racks. Half the nodes can't reach the other half.

**Switch/router failure**: A core switch fails, splitting the network into islands.

**Network congestion**: A switch's input buffer fills up. Packets are dropped. TCP connections time out. The system can't distinguish "partition" from "very slow network."

**Asymmetric partition**: Node A can send to node B, but B can't send to A. Particularly insidious — A thinks B is healthy (its own messages succeed), B thinks A is dead (B's messages to A are dropped). This causes split-brain.

**EC2 / Cloud network events**: AWS periodically experiences network events that partition subsets of a VPC. This is documented in postmortems from major cloud customers.

**Long GC pause**: A JVM node pauses for 30 seconds during garbage collection. From the network's perspective, it stopped responding. Other nodes may elect a new leader. When the GC pause ends, the old node may think it's still the leader — creating a split-brain scenario.

**Mitigation in design**:
- Leader leases with expiration (leader yields leadership after N seconds without contact with quorum).
- Epoch numbers — any message from a "superseded" leader is rejected.
- Fencing tokens — a monotonically increasing token issued to leaders. Storage systems reject operations from holders of an old token.

## How This Appears in Real Systems

**Raft in etcd/Kubernetes**: Every write to the Kubernetes API server goes through etcd's Raft log. The cluster requires a majority of etcd nodes to be available to accept writes. Losing a majority → control plane stops accepting changes.

**Kafka's replication**: Kafka topics have a replication factor. Each partition has an ISR (In-Sync Replicas) — replicas that are fully caught up. A write is only acknowledged when `acks=all` and all ISR members have persisted the message. If the ISR shrinks due to lag, availability vs durability trade-offs kick in.

**Saga in e-commerce**: Stripe's payment + order service saga is a classic — charge the card, then create the order, with compensation logic to refund if order creation fails.

## SRE Lens: What This Looks Like in Production

### Split-Brain After Network Partition
Two Cassandra nodes lose connectivity to each other. Both continue accepting writes. When the partition heals, they have diverging data. Cassandra's last-write-wins (based on client-side timestamps) picks a winner. The "loser" write is silently discarded. This is consistent with Cassandra's design, but it means writes during a partition may be lost. An SRE must know which partition behavior their database is designed for.

### etcd Quorum Loss Halting Kubernetes
A 3-node etcd cluster loses 2 nodes simultaneously (one crashed, one network partition). The Kubernetes API server becomes read-only — it can serve cached state but rejects all mutations. Pods cannot be scheduled, ConfigMaps cannot be updated. The SRE runbook for this is "restore etcd quorum before anything else."

### Vector Clock Conflict at DynamoDB Scale
A DynamoDB table with inconsistent read pattern shows "siblings" (multiple versions of the same key) under high concurrency. The application was not implementing conflict resolution — it was discarding all but the latest version. Audit log shows write conflicts during a traffic spike. Fix: switch to conditional writes or redesign the conflict resolution strategy.

## Common Misconceptions

- **"Distributed transactions are just like local transactions but slower"**: They are fundamentally different. 2PC can block indefinitely on coordinator failure. Sagas sacrifice atomicity entirely.
- **"Vector clocks tell me what the correct value is"**: Vector clocks tell you which writes are concurrent (conflicting). They don't tell you which value is correct. Your application or merge function must decide.
- **"Network partitions only happen in poorly operated systems"**: Every large-scale cloud operator experiences partitions. The question is not "if" but "what does your system do when it happens?"
- **"Raft guarantees no split-brain"**: Raft prevents split-brain in the consensus log, but JVM GC pauses, long OS delays, and misconfigured timeouts can still cause operationally split-brain-like scenarios.

## Key Takeaways

- The 8 fallacies describe the false assumptions that cause distributed system bugs. The most dangerous are "the network is reliable" and "latency is zero."
- Physical clocks cannot be used for causal ordering. Lamport timestamps provide partial ordering; vector clocks provide full causal ordering.
- Raft is the practical consensus algorithm to know. It decomposes into leader election and log replication, with safety provided by quorum writes and term numbers.
- 2PC and 2PL are different: 2PL is concurrency control within a DB; 2PC is distributed atomicity across DBs.
- Sagas trade atomicity for availability. They require compensating transactions and idempotency.
- The Outbox pattern solves the dual-write problem between a database and a message bus.
- Network partitions in production have known shapes: cable cuts, switch failures, GC pauses, asymmetric failures. Design your system knowing these happen.

## References

- Lamport, L. (1978). "Time, Clocks, and the Ordering of Events in a Distributed System." Communications of the ACM.
- Fischer, M., Lynch, N., Paterson, M. (1985). "Impossibility of Distributed Consensus with One Faulty Process." JACM.
- Ongaro, D. and Ousterhout, J. (2014). "In Search of an Understandable Consensus Algorithm." USENIX ATC. — The Raft paper.
- Kleppmann, M. — *Designing Data-Intensive Applications*, Chapters 8 and 9
- Richardson, C. — *Microservices Patterns* — comprehensive coverage of Saga and Outbox
- Deutsch, P. — "The Eight Fallacies of Distributed Computing" (Sun Microsystems)

# Scalability Patterns

> Scalability is not just about handling more load — it's about maintaining performance, reliability, and operational sanity as the system grows, and the patterns you choose at design time determine how gracefully you can grow later.

## Core Concept

Scalability is the property of a system to handle growing amounts of work by adding resources. The question is not "can it scale" (anything can be scaled with enough money and engineering) but "how does it scale, and at what cost?"

The patterns in this document are architectural decisions that either enable or constrain how a system can grow. Most of them involve tradeoffs: complexity vs independence, flexibility vs performance, strong consistency vs availability.

## Horizontal vs Vertical Scaling

### Vertical Scaling (Scale Up)
Add more resources to a single machine: more CPU cores, more RAM, faster SSDs, higher-bandwidth NICs.

**Advantages**:
- Simple: no distributed systems complexity.
- No code changes required.
- Works with stateful systems.
- Cheaper per unit of work at moderate scale.

**Limits**:
- Physical ceiling: the largest available AWS EC2 instance has 192 vCPUs and 24TB RAM (u-24tb1.metal). At some scale, you hit the hardware ceiling.
- Single point of failure: one machine failing takes down the entire service.
- Vertical scaling often requires downtime (instance type change, RAM upgrade).
- Diminishing returns: doubling cores does not double throughput for I/O-bound workloads.

**When to use**: Until you actually need horizontal scale. Many systems that "need" to be distributed don't. A well-tuned PostgreSQL on an 8xl instance handles millions of OLTP operations per second. Scale vertically first; add horizontal complexity only when you have a concrete bottleneck.

### Horizontal Scaling (Scale Out)
Add more machines to a pool. Distribute load across the pool.

**Advantages**:
- Near-linear scaling for stateless workloads.
- No single point of failure.
- Can use commodity hardware.
- Elastic: add and remove machines dynamically.

**Requirements**:
- Stateless services (or externalized state): any instance must be able to serve any request.
- Load balancer to distribute requests.
- Distributed state management (databases, caches, queues).

**When to apply**: After you've exhausted vertical scaling options, or when you need the fault tolerance of multiple instances (at least 2 for HA), or when your traffic patterns require elastic scaling.

## Stateless Services

### Why Statelessness Enables Horizontal Scale
A stateless service holds no client-specific state in memory between requests. Every request contains all the information needed to serve it. Any instance can serve any request.

**Stateless implications**:
- Session state → Redis or DynamoDB.
- Uploaded files → S3, not local disk.
- In-progress computation → handed off via queue or DB, not held in memory.
- JWT tokens → carry claims without server-side session lookup.

**Deployment benefits**:
- Auto-scaling: spin up more instances under load, terminate them when load drops.
- Deploy without sticky sessions.
- Health checks are simple: any instance is equivalent.
- No "draining" concerns beyond in-flight request completion.

### Stateless Design Checklist
- No in-memory session state. (Use Redis or a cookie.)
- No local disk writes for user data. (Use S3.)
- No node-local cache that is required for correctness. (Caches are acceptable; they can be cold.)
- No hardcoded hostname dependencies.
- Configuration from environment variables, not hard-coded.

## CQRS (Command Query Responsibility Segregation)

### The Pattern
Separate the model used for writes (Commands) from the model used for reads (Queries). Instead of one unified data model for all operations, you have:

- **Command side**: Accepts writes (create, update, delete). Validates business rules. Writes to the write store (often normalized, relational).
- **Query side**: Accepts reads. Returns data in the shape the consumer needs. Reads from one or more read-optimized stores (denormalized, projected views, search indexes).

### Why It Exists
OLTP workloads often have very different read and write patterns:
- Writes need strict validation, consistency, transactional atomicity.
- Reads need different projections of the same data for different consumers (a mobile app shows a summary; an analytics dashboard needs an aggregation).

In a single-model system, optimizing for reads (denormalize, index heavily) conflicts with optimizing for writes (normalize to avoid update anomalies).

CQRS resolves this by maintaining separate models and synchronizing them asynchronously.

### Example
An e-commerce order service:
- **Write model**: Normalized orders, order_items, inventory tables. ACID transactions for inventory decrement + order creation.
- **Read model**: Denormalized `order_summary` view materialized in Redis. A separate `product_search_index` in Elasticsearch. A `user_order_history` projection in DynamoDB.

When an order is created, the command handler writes to the normalized tables and publishes an `OrderCreated` event. Subscribers update each read model asynchronously.

### Trade-offs
- **Complexity**: Two models to maintain, synchronization logic between them.
- **Eventual consistency**: Read models lag behind the write model. Acceptable for most reads; not for read-your-writes if the user expects to see their order immediately after placing it.
- **When worth it**: High read-to-write ratio, diverse consumer data needs, significant read performance requirements.

## Event Sourcing

### What It Is
Instead of storing the current state of a record, store the sequence of events that led to that state. The current state is derived by replaying events.

**Traditional (state-based)**:
```
orders table: { id: 123, status: "shipped", total: 99.00, updated_at: ... }
```

**Event-sourced**:
```
events: 
  { order_id: 123, type: "OrderCreated", payload: {...}, timestamp: T1 }
  { order_id: 123, type: "PaymentReceived", payload: {...}, timestamp: T2 }
  { order_id: 123, type: "OrderShipped", payload: {...}, timestamp: T3 }
```

The current order state is derived by replaying these events.

### Benefits
- **Perfect audit trail**: Every change is recorded. You can reconstruct what happened and when.
- **Temporal queries**: "What was the state of this order 3 weeks ago?" Replay events up to that timestamp.
- **Event replay**: If a bug corrupts downstream state, replay the events to rebuild it correctly.
- **Multiple projections**: The same event stream can be consumed by multiple read models, each projecting different views.
- **Debugging**: The full event history explains how you got to the current state.

### Costs
- **Complexity**: Significant increase in architectural complexity.
- **Snapshot requirement**: For high-event entities, replaying from the beginning on every read is slow. Periodically snapshot the current state and replay only from the last snapshot.
- **Schema evolution**: Changing event schemas for old events is painful. Events are immutable; you can't migrate them like rows.
- **Query difficulty**: "List all orders over $100" requires reading all order events or maintaining a read-projected query model.
- **Not for every domain**: Use event sourcing where the event history itself is business value (financial ledgers, legal records, audit logs). Don't use it just because it's architecturally interesting.

### When to Use
- Financial systems where audit trails are mandatory.
- Systems with complex business logic where business analysts need to understand what happened.
- Collaborative editing (Google Docs-style) where the document is built from operations.
- Where undo/redo semantics are required.

## Saga Pattern

### The Problem
Distributed transactions across microservices. You can't use 2PC reliably across independent services. How do you ensure multi-step operations are atomic?

**Example**: Booking a trip requires:
1. Reserve flight.
2. Reserve hotel.
3. Charge credit card.

If the credit card charge fails after steps 1 and 2 succeed, you need to cancel the flight and hotel reservations.

### Compensating Transactions
A Saga is a sequence of local transactions. Each transaction has a corresponding compensating transaction that undoes its effect. If any step fails, the saga executes compensating transactions for all preceding successful steps.

**Important**: Compensating transactions are not rollbacks. They are new forward-running transactions that logically undo the effect. They must themselves succeed, which means they must be idempotent and resilient.

### Choreography vs Orchestration

**Choreography**: No central coordinator. Each service listens for events and reacts. When step 1 succeeds, it publishes `FlightReserved`. The hotel service listens and reserves the hotel. When `HotelReserved` fires, the payment service charges the card.

```
FlightService → [FlightReserved] → HotelService → [HotelReserved] → PaymentService
                                                                         ↓ failure
                                                                   [PaymentFailed]
                                                                     → HotelService (cancel)
                                                                     → FlightService (cancel)
```

**Advantages**: Simple services, no central point of failure.
**Disadvantages**: Hard to trace the overall flow. Cyclic dependencies are possible. Debugging is complex.

**Orchestration**: A central saga orchestrator tells each service what to do.

```
SagaOrchestrator:
  1. Tell FlightService: Reserve flight
  2. Tell HotelService: Reserve hotel
  3. Tell PaymentService: Charge card
  On failure at step 3:
    Tell HotelService: Cancel hotel
    Tell FlightService: Cancel flight
```

**Advantages**: The overall flow is explicit and visible in one place. Easier to monitor and debug.
**Disadvantages**: The orchestrator is a central dependency. Business logic leaks into the orchestrator.

### Practical Guidance
Use orchestration for complex sagas where visibility and debuggability matter. Use choreography for simple event chains where each service can independently decide what to do.

## Cell-Based Architecture

### What It Is
Divide the system into isolated "cells," each a complete, independent stack serving a subset of users. Cells share no state with each other. A failure in one cell does not affect others.

**Analogy**: A navy ship is divided into watertight compartments. A breach in one compartment doesn't sink the ship. Cell-based architecture is the same principle: blast radius containment.

### How It Works
- Users are assigned to a cell (typically based on a hash of user ID or tenant ID).
- Each cell has its own instance of every service tier: API, application servers, database, cache.
- Cells do not call each other directly.
- A "control plane" manages cell assignment and global metadata.

**Example**: Slack divides customers into cells (called "shards"). A Slack workspace belongs to one shard. An incident in shard 7 affects only workspaces assigned to shard 7. Other workspaces are completely unaffected.

**Uber uses cells** at the city level: the London cell serves London rides. A bug or outage in the London cell doesn't affect New York.

### Benefits
- **Blast radius containment**: Incidents affect a fraction of users, not all users.
- **Independent deployments**: Deploy updates to one cell, verify, then roll out to others.
- **Independent scaling**: Scale cells based on their load (dense urban cells need more capacity than rural cells).
- **Testing at scale**: A new cell can be a "test cell" with production traffic before wide rollout.

### Trade-offs
- **Increased infrastructure cost**: N cells means N times the infrastructure (at minimum).
- **Cross-cell features**: Features that require data from multiple cells (e.g., global search, cross-workspace communication) require a separate mechanism.
- **Cell routing logic**: Every request must be routed to the correct cell.

## Bulkhead Pattern

### What It Is
Isolate components of a system so that a failure in one doesn't cause failures in others. Named after the bulkheads in ships that prevent flooding from spreading between compartments.

### Implementation Forms

**Thread pool isolation**: Assign separate thread pools to different dependencies. If the payment service is slow, its thread pool fills up, but the thread pool for the inventory service is unaffected. Applications can still process inventory-related requests.

**Connection pool isolation**: Separate database connection pools per service or feature. A slow bulk analytics query doesn't exhaust connections for transactional operations.

**Process/container isolation**: Run different services in separate processes or containers. A memory leak in service A doesn't kill service B.

**Kubernetes resource limits**: CPU and memory limits prevent one pod from starving others on the same node.

**Example**: Netflix Hystrix uses thread pool isolation. Each downstream service dependency (users, recommendations, billing) gets its own thread pool. If the recommendations service is down and its thread pool is exhausted, calls to recommendations fail fast, but all other user requests continue normally.

## Strangler Fig Pattern

### The Problem
You have a large, monolithic legacy system. You need to migrate to microservices (or a new architecture). You cannot rewrite everything at once — that's the "big bang" rewrite, which historically fails.

### The Solution
Incrementally replace the monolith. New functionality goes into new services. Old functionality is gradually extracted. The monolith "strangles" over time, like a strangler fig vine growing around and eventually replacing a tree.

**Implementation**:
1. Put a facade (API gateway or reverse proxy) in front of the monolith.
2. For a feature you want to migrate, build it as a new service.
3. Update the facade to route requests for that feature to the new service instead of the monolith.
4. Gradually migrate features one by one.
5. When the monolith handles nothing, retire it.

**Used at**: Martin Fowler documented this pattern at ThoughtWorks. Used extensively in e-commerce platforms, banks, and any organization migrating from legacy systems.

## Database Read Replicas Pattern

### How It Works
The primary database handles all writes. One or more read replicas receive a copy of every write via replication. Read traffic is distributed across replicas.

**Typical ratio**: 1 primary + 2–5 read replicas. Route 80–90% of read traffic to replicas.

**Limitations**:
- Replication lag: replicas may be 10–500ms behind the primary. Operations that require reading your own writes must go to the primary.
- Not a substitute for a proper sharding strategy at very high write volumes — all writes still go to one primary.

**Use for**: Analytics queries, reporting, background batch jobs, any read that can tolerate slight staleness.

## Async Processing: Offloading Work to Queues

### The Principle
Synchronous user requests should only do the minimum work required to generate a response. Anything that doesn't need to complete before the response can be deferred to an async worker.

**Sync path**: Validate input, write to DB, return response.

**Async deferred**: Send email, generate PDF, resize images, update search index, update analytics, send webhooks, notify other services.

### Benefits
- **Lower p99 latency**: The sync path is fast because it's minimal.
- **Fault isolation**: A failure in email sending doesn't fail the API request.
- **Retryability**: Async jobs can be retried without impacting the user.
- **Scalability**: Async workers can be scaled independently of API servers.

### The Pattern
```
POST /orders
  → Validate input
  → Write order to DB (sync)
  → Publish "OrderCreated" event to queue (sync, fast)
  → Return 201 with order ID

AsyncWorker (consuming queue):
  → Send order confirmation email
  → Update inventory cache
  → Trigger fulfillment workflow
  → Update search index
```

## Fan-out Patterns: Fan-out on Write vs Fan-out on Read

### The Twitter Home Timeline Problem
When User A (with 10M followers) posts a tweet, how is it shown in all 10M followers' home timelines?

### Fan-out on Write (Push Model)
When User A posts a tweet, the system immediately copies the tweet ID into the inbox of all 10M followers. The home timeline read is a simple lookup of the user's pre-computed inbox.

**Read**: O(1) — just read from the user's pre-computed inbox.
**Write**: O(N) — N = number of followers. For a celebrity with 10M followers, one tweet triggers 10M writes.

**Used for**: Regular users with a manageable follower count. Twitter used fan-out on write for most users.

**Problem with celebrities**: 10M writes per tweet. At Katy Perry scale (100M followers), this is 100M writes per tweet. Not feasible synchronously.

### Fan-out on Read (Pull Model)
When a user requests their home timeline, the system fetches recent tweets from all followed users and merges them in real-time.

**Read**: O(N) — N = number of followed accounts. Scan each followed user's recent tweets, merge, sort.
**Write**: O(1) — just write the tweet to the author's own timeline.

**Problem**: For users following 5,000 accounts, reading the home timeline requires 5,000 lookups and a merge. Very slow at read time.

### Twitter's Hybrid Approach
- **Regular users**: Fan-out on write. Pre-compute inbox. Fast reads.
- **Celebrity users** (millions of followers): Fan-out on read. When you request your home timeline, the system injects celebrity tweets at read time by querying the celebrities you follow.
- **Most users see fast pre-computed timelines, with late-injected celebrity tweets.**

This is a practical example of a hybrid architecture that makes pragmatic trade-offs based on real-world data distributions (most accounts have few followers; a small fraction have many).

## How These Patterns Appear in Real Systems

**Uber's cell-based architecture**: Uber operates geographically isolated cells per city/region. Each cell handles rides in that region independently. The architecture enables them to roll out new features city by city, isolate outages, and scale cells independently based on ride volume.

**Netflix Bulkhead with Hystrix**: Netflix's Hystrix library implements bulkhead isolation. Each external service call (recommendations, ratings, metadata) has a dedicated thread pool. If the recommendations service degrades, Netflix gracefully degrades by showing a default row instead of a personalized one. The rest of the UI continues normally.

**Event Sourcing at Banks**: Core banking systems are natural candidates for event sourcing — every transaction is an event (debit, credit, fee). The account balance is derived by replaying events. Banks have used this model (though not always called "event sourcing") for decades in their ledger systems.

## SRE Lens: What This Looks Like in Production

### Stateful Service Blocking Autoscaling
An SRE is trying to autoscale a service during a traffic spike. The autoscaler adds 10 new instances, but the load doesn't distribute. Investigation reveals the service stores user session data in-process (in-memory HashMap). Existing users are sticky to specific instances. New instances handle no traffic. All load remains on the original instances. They OOM. Fix: externalize session state to Redis, make the service stateless, redeploy.

### Saga Compensating Transaction Failure
During an order saga, the hotel reservation step fails after the flight reservation succeeded. The saga triggers a compensating transaction to cancel the flight. The flight cancellation API is down. The compensating transaction can't complete. The saga is stuck: the flight is reserved, the hotel is not, and the user has been charged. Fix: compensating transactions need their own retry queues and eventually-consistent resolution. Design the Saga orchestrator to retry compensation indefinitely (with alerts for stuck sagas requiring manual intervention).

### Fan-out Write Spike During Celebrity Post
A major celebrity (50M followers) posts on a social platform. Fan-out workers must write to 50M user inboxes. The fan-out queue depth hits 50M. Workers are saturated. The system falls 30 minutes behind on delivering other users' posts. Fix: apply the hybrid fan-out model — switch high-follower accounts to fan-out on read, or rate-limit/throttle celebrity fan-out writes, or dramatically over-provision fan-out workers for expected viral events.

## Common Misconceptions

- **"Microservices automatically scale better than monoliths"**: A stateless monolith can scale horizontally just as well as microservices. Microservices add distributed systems complexity. Scale within a monolith first.
- **"Event sourcing is just using Kafka"**: Event sourcing is an architectural pattern about how you persist state. Kafka may be the event store, but event sourcing requires a specific approach to state reconstruction and querying.
- **"CQRS requires event sourcing"**: CQRS and event sourcing are often used together but are independent patterns. You can have CQRS with a traditional state-based write model.
- **"Cell-based architecture is only for huge scale"**: Cell-based architecture's primary benefit is blast radius containment, not scale. Even at moderate scale, cells reduce the percentage of users affected by any given incident.
- **"Sagas replace transactions"**: Sagas provide eventual consistency for multi-service operations. They do not provide atomicity. During the saga's execution, the system is in inconsistent intermediate states.

## Key Takeaways

- Scale vertically until you have a concrete bottleneck. Horizontal scale requires stateless services and distributed state management.
- Statelessness is the prerequisite for horizontal scaling. Externalize all state: sessions to Redis, files to S3.
- CQRS separates write and read models, enabling each to be optimized for its workload. Appropriate when reads and writes have significantly different requirements.
- Event sourcing provides a perfect audit trail and replayability at the cost of significant architectural complexity. Use it where the event history is genuinely valuable.
- Sagas manage distributed transactions via compensating transactions, trading atomicity for availability. Both choreography and orchestration have valid use cases.
- Cell-based architecture contains blast radius: a failure in one cell affects only users assigned to it. Used by Slack, Uber, and large-scale platforms.
- Bulkheads isolate failure: thread pool isolation per dependency prevents one slow dependency from degrading unrelated features.
- Fan-out on write is fast for reads but expensive for high-follower accounts. The hybrid model (fan-out on write for normal users, fan-out on read for celebrities) is the production solution.

## References

- Newman, S. — *Building Microservices* (O'Reilly) — comprehensive microservice patterns
- Richardson, C. — *Microservices Patterns* (Manning) — CQRS, Event Sourcing, Saga in depth
- Fowler, M. — *Patterns of Enterprise Application Architecture* — foundational patterns
- Fowler, M. — "Strangler Fig Application" blog post — https://martinfowler.com/bliki/StranglerFigApplication.html
- Hohpe, G. and Woolf, B. — *Enterprise Integration Patterns* — message-based integration patterns
- Twitter Engineering Blog — "The Infrastructure Behind Twitter: Scale" — fan-out architecture details

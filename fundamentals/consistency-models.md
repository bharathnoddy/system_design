# Consistency Models

> Consistency models define the rules a distributed system makes about what values reads can return relative to writes — understanding the spectrum from linearizability to eventual consistency determines what guarantees your users can depend on.

## Core Concept

A consistency model is a contract between a storage system and the programmers who use it. It specifies which values are legal to return for a read operation given the history of writes. Choosing the wrong model causes user-visible bugs; choosing a model stronger than necessary pays unnecessary latency and availability costs.

The consistency spectrum, from strongest to weakest:

```
Linearizability → Sequential → Causal → Read-Your-Writes / Monotonic Reads → Eventual
```

Moving left: stronger guarantees, higher latency, lower availability during partitions.  
Moving right: weaker guarantees, lower latency, higher availability.

## Linearizability (Strongest)

**What it guarantees**: Every operation appears to take effect atomically at a single point in real time between its invocation and completion. The system appears as a single, non-replicated store.

More precisely: if operation A completes before operation B begins, then B must see A's effects. Concurrent operations can be ordered arbitrarily, but the ordering must be consistent with real time for non-overlapping operations.

**Analogy**: Think of a single register (a variable) with a lock. Every read or write grabs the lock, performs the operation atomically, and releases it. The entire distributed system must behave as if this is how it works.

**Cost**:
- Requires coordination on every write (or read, depending on implementation).
- Latency = at least one round trip to ensure all nodes agree.
- Unavailable during partitions (CP in CAP terms).
- According to the FLP impossibility theorem, in an asynchronous network with even one crash failure, no algorithm can guarantee both liveness and linearizable consensus.

**Real examples**:
- **Google Spanner**: Uses TrueTime (GPS + atomic clocks with bounded uncertainty) to provide external consistency (a form of linearizability). Before committing a transaction, it waits for the uncertainty window to pass, ensuring global ordering.
- **etcd**: All reads/writes go through Raft consensus. Every read that requests linearizability contacts the leader and forces a log sync.
- **ZooKeeper**: Provides linearizable writes. Reads from followers can be stale unless you call `sync()` first.
- **CockroachDB**: Serializable isolation implemented with distributed consensus.

**When to use**: Distributed locks, leader election, financial transactions, inventory decrement, any operation where acting on a stale value causes incorrect behavior.

## Sequential Consistency

**What it guarantees**: All operations appear to execute in some sequential order that is consistent with the program order of each individual process. Operations from the same process appear in program order relative to each other, but the global ordering does not need to match real time.

**The key difference from linearizability**: In sequential consistency, you lose the real-time constraint. Two different clients may see operations interleaved in different orders, as long as each client sees its own operations in the right order and both see *some* consistent global sequence.

**Analogy**: Everyone at a conference sees the presentations in the same order, but that order might not match when the presentations were actually scheduled — it just needs to be a consistent sequence everyone agrees on.

**Cost**: Lower than linearizability because you don't need to synchronize on a global real-time clock. But still requires global ordering.

**Real examples**:
- **Multi-core CPUs** (with memory barriers): CPU memory models are often sequential consistency with explicit fence instructions.
- Some distributed databases use sequential consistency for multi-object transactions.

**When to use**: Scenarios where program-order consistency per client is sufficient, but you don't need real-time ordering guarantees across clients.

## Causal Consistency

**What it guarantees**: Operations that are causally related must be seen by all nodes in causal order. Concurrent operations (with no causal relationship) can be seen in different orders by different nodes.

Causality is tracked explicitly: if A wrote X and B read X and then wrote Y, then Y is causally dependent on X. Any node that sees Y must already have seen X.

**Analogy**: In a conversation thread, if Alice's post replies to Bob's post, everyone must see Bob's post before they see Alice's reply. But two unrelated posts can appear in any order.

**Cost**: Significantly cheaper than sequential consistency. You only need to coordinate operations that are causally related. Unrelated operations can proceed independently.

**Real examples**:
- **MongoDB (causal sessions)**: Causal consistency is available within a session. If you read a value and then write based on it, later reads in that session will reflect the write.
- **COPS (Clusters of Order-Preserving Servers)**: A research system that provides causal+ consistency across geo-replicated data stores.
- **Dynamo-style systems with vector clocks**: Detect causal relationships and ensure causal ordering.

**When to use**: Social feeds, comment threads, collaborative editing — anywhere you need "replies come after their parents" but can tolerate reordering of unrelated items.

## Read-Your-Writes Consistency

**What it guarantees**: After a client writes a value, subsequent reads by the *same client* will always see that value (or a newer one). Other clients might still see stale data.

This is a session guarantee, not a system-wide guarantee.

**Analogy**: If you update your profile picture on a social network, you should always see your new picture when you look at your own profile — even if other users still see the old one for a bit.

**Cost**: Can be implemented cheaply by routing reads from a client to the replica it last wrote to, or by tracking a "read-after-write" version token.

**Real examples**:
- **DynamoDB**: Read-your-writes is guaranteed when you use strongly consistent reads or when you use the same connection to the same replica.
- **AWS RDS with read replicas**: If you write to the primary and immediately redirect your next read to a read replica, you may not read your own write. Applications must implement stickiness or use version tokens.
- **Facebook TAO**: Uses a similar mechanism — after a write, the client is temporarily "sticky" to a primary datacenter to ensure read-your-writes.

**When to use**: User-facing CRUD operations. Users expect to see their own changes immediately. This is one of the most common expectations to violate accidentally in distributed systems.

## Monotonic Reads

**What it guarantees**: If a client reads value V at time T, it will never read a value older than V in any subsequent read. Reads are monotonically non-decreasing in terms of data freshness.

Without this, a user could see data "go backward in time" — seeing a newer value, then a stale value, which is extremely confusing.

**Cost**: Requires routing a client's reads to replicas that have caught up to at least the point the client last read from. Usually implemented with session tokens or read-your-writes plus replica staleness tracking.

**Real examples**:
- **Cassandra**: When reading from multiple replicas, you need QUORUM or similar to ensure monotonic reads. LOCAL_ONE reads from a random node per request can violate monotonic reads.
- **MongoDB causal sessions**: Ensures monotonic reads within a session.
- **Zookeeper**: After calling `sync()`, reads from a follower are monotone relative to the sync point.

**When to use**: Any sequential workflow — onboarding steps, wizard UIs, any multi-step process where the user must not see previous steps "undo."

## Eventual Consistency

**What it guarantees**: If no new writes are made to an object, all reads will eventually return the last written value. All replicas will converge to the same value — eventually.

What it does NOT guarantee: how long "eventually" takes, what value you'll read before convergence, or anything about ordering between different objects.

**This is not the same as "basically inconsistent" or "probably correct."** Eventual consistency is a precise mathematical guarantee about convergence. The engineering challenge is defining what "correct" means during the convergence window.

**Conflict resolution strategies under eventual consistency**:
- **Last-Write-Wins (LWW)**: The write with the highest timestamp wins. Simple, but can lose writes on concurrent updates. Clocks must be synchronized (or logical clocks used).
- **Multi-value register**: Store all conflicting versions and surface them to the application for resolution (Dynamo's approach).
- **CRDTs (Conflict-free Replicated Data Types)**: Data structures that are mathematically guaranteed to merge correctly — G-Counters, OR-Sets, LWW-Sets. Suitable for specific use cases like counters, collaborative text editing.
- **Application-level merge**: The application defines how to resolve conflicts. Maximum control, maximum complexity.

**Real examples**:
- **Cassandra (with ONE or LOCAL_ONE consistency level)**: A read from one replica. If that replica is behind, you get stale data.
- **DynamoDB (eventually consistent reads)**: Reads may return data up to 1 second old. Cheaper and faster than consistent reads.
- **DNS**: A DNS update propagates globally over minutes to hours. Every DNS server is eventually consistent.
- **S3 (historical)**: For a long time, S3 had read-after-write consistency only for new objects, and eventual consistency for overwrites. AWS updated this to strong consistency in December 2020.
- **Couchbase, Riak**: Designed for eventual consistency with vector clock conflict detection.

**When to use**: Caches, CDN content, social activity feeds, recommendation systems, shopping cart totals, user analytics — operations where the cost of stale data is low and the benefit of high availability and low latency is high.

## How These Models Relate to User-Visible Bugs

| Consistency Model Violated | User Experience |
|---|---|
| Read-your-writes | User updates their email. Refreshes the page. Old email is shown. User submits the update again. Now there are two update events in the audit log. |
| Monotonic reads | User sees 150 unread messages. Clicks refresh. Now sees 140. Messages appear to "unread" themselves. |
| Causal | User posts a reply to a comment. Other users see the reply before the original comment. The page is confusing. |
| Linearizability | Two users simultaneously claim the last available ticket to an event. Both transactions succeed. The venue is oversold. |
| Eventual (wrong assumption) | Inventory shows 1 item available. 50 concurrent users add it to their cart. All 50 get an "Added to cart" success. 49 will be disappointed at checkout. |

## When Each Model Is Appropriate

| Use Case | Minimum Required Model |
|---|---|
| Distributed lock / leader election | Linearizability |
| Bank account balance | Linearizability (or serializable transactions) |
| Inventory decrement | Linearizability |
| User profile reads (own profile) | Read-your-writes |
| Activity feed | Causal consistency |
| Social follower count | Eventual consistency |
| Product catalog | Eventual consistency |
| DNS lookup | Eventual consistency |
| Shopping cart | Eventual consistency with CRDT or conflict detection |
| Search index | Eventual consistency |
| Configuration management | Sequential or linearizable |

## SRE Lens: What This Looks Like in Production

### The "Phantom Update" Bug
An SRE is on-call for a customer complaints spike. Users report their account settings revert after saving. Root cause: the application writes to a primary DB, then redirects the user to a page that reads from a read replica. Replication lag is 200ms. The read arrives before the write is replicated. The user sees their old settings and thinks the save failed. They save again. No data loss, but terrible UX. Fix: implement read-your-writes routing for post-write redirects.

### Cassandra Inconsistency During Rolling Restart
During a maintenance window, an SRE performs a rolling restart of a Cassandra cluster. Because of the timing, a read coordinator briefly has a node marked as up that is actually restarting. A read that should return the latest value returns a slightly older value from a node that was mid-restart. The consistency level was LOCAL_QUORUM, but the quorum included the restarting node. This is a transient consistency violation, not data corruption — it resolves itself — but it caused a brief anomaly in a report.

### DynamoDB Consistent vs Eventually Consistent Read Confusion
A team migrates from a relational DB to DynamoDB and uses eventually consistent reads everywhere by default (they're cheaper). Their payment processing flow reads an account balance with an eventually consistent read immediately after a write. Under high load, the replication lag is 300ms — enough to cause the balance to appear unchanged, triggering a duplicate payment attempt.

## Common Misconceptions

- **"Eventual consistency is 'basically consistent'"**: No. Eventual consistency is a precise guarantee about convergence. The operational question is: what does your application do during the convergence window?
- **"Linearizability is always the safe choice"**: It is the safest for correctness, but it's expensive in latency and unavailability during partitions. Many systems pay this cost unnecessarily.
- **"Read-your-writes is easy to get for free"**: It requires either routing reads to the node you wrote to (adds coupling) or tracking write versions and enforcing them on reads (adds complexity).
- **"Eventual consistency means the system might be wrong forever"**: If you stop writing, all replicas converge. The system does not stay wrong forever. The question is behavior *during* writes.
- **"These models only matter for databases"**: Caches, CDNs, message queues, file systems — all have consistency models. Ignoring them causes bugs in any distributed component.

## Key Takeaways

- Consistency models are contracts between a storage system and its clients about what values reads are allowed to return.
- Linearizability is the strongest model: the system appears as a single atomic register. It's correct but expensive.
- Eventual consistency is not "broken consistency" — it's a precise convergence guarantee with a defined window of staleness.
- Most user-visible consistency bugs come from violating read-your-writes or monotonic reads — often from reading after a write from a replica.
- Choose the weakest model your application can tolerate. Strong consistency costs latency; weak consistency costs complexity in conflict resolution.
- Causal consistency is often the sweet spot for social and collaborative applications: strong enough to be intuitive, cheap enough to be fast.
- DynamoDB, Cassandra, and most NoSQL databases default to eventual consistency. Opt-in to stronger models where you need them, with awareness of the cost.

## References

- Herlihy, M. and Wing, J. (1990). "Linearizability: A Correctness Condition for Concurrent Objects." ACM TOPLAS.
- Terry, D. et al. (1994). "Session Guarantees for Weakly Consistent Replicated Data." IEEE PDIS.
- Vogels, W. (2008). "Eventually Consistent." ACM Queue / Communications of the ACM.
- Kleppmann, M. — *Designing Data-Intensive Applications*, Chapter 9
- Bailis, P. et al. (2013). "Highly Available Transactions: Virtues and Limitations." VLDB.
- Shapiro, M. et al. (2011). "CRDTs: A Principled Approach to Distributed Data Structures." — for CRDT fundamentals

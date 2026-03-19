# CAP Theorem

> CAP theorem states that a distributed system can only guarantee two of three properties — Consistency, Availability, and Partition Tolerance — and understanding which two you're choosing defines your system's failure behavior.

## Core Concept

CAP theorem, formalized by Eric Brewer in 2000 and proved by Gilbert and Lynch in 2002, states that a distributed data store can provide at most two of the following three guarantees simultaneously:

- **Consistency (C)**: Every read receives the most recent write or an error. All nodes see the same data at the same time. This is linearizability — not eventual consistency or "consistent enough."
- **Availability (A)**: Every request receives a (non-error) response — not a guarantee that it contains the most recent write, but a response nonetheless.
- **Partition Tolerance (P)**: The system continues to operate even when network messages are lost or delayed between nodes.

The key insight: **network partitions are not optional in distributed systems**. Networks fail. Cables are cut. Data centers lose connectivity. The choice is never really "CA, CP, or AP" in a vacuum — it's "when a partition occurs, do we sacrifice consistency or availability?"

### The Partition Scenario

Imagine two database nodes, A and B. A client writes a new value to node A. Before replication completes, the network link between A and B is severed. Now another client reads from node B.

You face an unavoidable choice:
1. **Return the stale value from B** — you preserved availability, sacrificed consistency.
2. **Return an error or block** — you preserved consistency, sacrificed availability.

There is no third option that preserves both while the partition exists.

## CA, CP, and AP Systems

### CA (Consistency + Availability) — Partitions Are Fatal

CA systems assume the network will not partition. They optimize for consistency and availability under normal conditions but fail catastrophically when a partition occurs.

**Real Examples:**
- **PostgreSQL / MySQL** (single-node or synchronous replication): A standalone relational database is fully consistent and highly available. But it's not distributed across failure domains — it's a single point of failure. Synchronous multi-master setups that refuse writes when a node is unavailable are also CA.
- **RDBMS in a single data center**: Within a tightly controlled network, you can often ignore partition risk. Once you span availability zones or data centers, you're back to CAP's constraints.

**In practice**: Choosing CA means you've decided not to distribute state across failure-independent nodes. This is a legitimate architectural choice for many workloads.

### CP (Consistency + Partition Tolerance) — Sacrifice Availability

CP systems remain consistent when a partition occurs, but they become unavailable for some operations — typically refusing writes or reads that cannot be verified as up-to-date.

**Real Examples:**
- **ZooKeeper**: A quorum-based coordination service. If fewer than N/2+1 nodes can communicate, ZooKeeper stops accepting writes. This is intentional — it would rather say "I can't answer" than give a stale answer about who holds a lock.
- **etcd**: Same quorum model, used by Kubernetes. If the control plane loses quorum, the cluster stops accepting changes to state.
- **HBase**: Built on HDFS with strong consistency. During a region server failure, the affected regions are unavailable until the master reassigns them.
- **MongoDB (default write concern w:majority)**: Writes require acknowledgment from a majority of replicas. If the primary is partitioned from the majority, it steps down rather than accept writes that might conflict.

**When to choose CP**: Coordination services, financial ledgers, configuration management, anything where acting on stale data is worse than temporarily blocking.

### AP (Availability + Partition Tolerance) — Sacrifice Consistency

AP systems remain available during partitions but may return stale or conflicting data. They resolve conflicts after the partition heals through mechanisms like last-write-wins, vector clocks, or application-level merge logic.

**Real Examples:**
- **Cassandra**: Designed to be always-on. With a replication factor of 3 and a QUORUM read, you can tolerate one node failure. But with LOCAL_ONE reads, you'll happily return stale data to keep writes and reads flowing. Cassandra resolves conflicts with last-write-wins based on timestamps.
- **DynamoDB (default)**: Eventually consistent reads are cheaper and faster. Strongly consistent reads are available but cost more and are blocked during partitions.
- **CouchDB**: Embraces AP with MVCC. Conflicts are stored explicitly and must be resolved by the application.
- **DNS**: Returns cached (potentially stale) answers. The system stays available even if authoritative servers are unreachable.
- **Shopping cart systems (Amazon)**: Bezos's famous paper — let customers add to their cart even if you can't verify inventory. Reconcile later. Availability over consistency.

**When to choose AP**: User-facing services where downtime is worse than showing slightly stale data — social feeds, shopping carts, user profiles, recommendation systems.

## PACELC: The Extension That Matters More

CAP only describes behavior during partitions. PACELC (proposed by Daniel Abadi, 2012) extends this:

**If Partition: choose between Availability and Consistency**  
**Else (no partition): choose between Latency and Consistency**

The "Else" case is where most systems actually operate. Even without a partition, maintaining strong consistency requires coordination — which adds latency.

| System | Partition Behavior | Normal Behavior |
|---|---|---|
| Cassandra | AP | EL (low latency, eventual consistency) |
| Spanner | CP | HC (high consistency, higher latency) |
| DynamoDB | AP | EL |
| ZooKeeper | CP | HC |
| MySQL (single node) | CA | HC |
| Riak | AP | EL |

**Example**: Spanner uses TrueTime (GPS + atomic clocks) to provide external consistency globally. Even without a partition, a write to Spanner involves multiple rounds of Paxos consensus across datacenters — adding tens of milliseconds of latency. You're paying latency for consistency even in the happy path.

This is the design tradeoff that matters daily. CAP is the failure-time tradeoff. PACELC is the steady-state tradeoff.

## Common Misconceptions

### "CAP means I pick two properties and tune a knob"
CAP is not a dial. It describes an impossibility during a network partition. You don't configure your system to be "CA" — you design it with the understanding that partitions will happen and decide what breaks when they do.

### "Partitions are rare, so I can mostly ignore P"
Partitions are not rare — they are inevitable at scale. AWS, Google, and Azure all have documented partition events. Even a well-operated data center will experience network drops, switch failures, and overloaded links. If your design assumes partitions won't happen, you're building on a false premise.

### "Eventual consistency means the system will be inconsistent forever"
Eventual consistency means: if writes stop, all nodes will converge to the same value. The window of inconsistency depends on replication lag — often milliseconds to seconds in practice. The real question is what happens if a client reads during that window.

### "CP systems are always safer"
CP systems are correct during partitions, but they can fail hard. A ZooKeeper cluster that loses quorum provides zero service. Depending on your use case, an AP system that returns slightly stale data may cause less harm than a CP system that returns errors or blocks entirely.

### "CAP applies to individual operations"
CAP applies to the system level, not individual operations. A system can offer strongly consistent reads for some operations and eventually consistent reads for others (DynamoDB's consistent vs. eventually consistent reads are an example).

## How to Reason About CAP When Designing

Ask these questions:

1. **What happens if two nodes disagree on the current value?**
   - Can your application tolerate seeing stale data? (AP is fine)
   - Would acting on stale data cause financial or safety harm? (CP required)

2. **What is worse for your users — an error or stale data?**
   - A 503 during checkout is usually worse than a slightly stale inventory count.
   - A 503 during a bank transfer is better than double-crediting an account.

3. **What is your conflict resolution strategy?**
   - Last-write-wins (LWW) is simple but loses data on concurrent writes.
   - CRDTs (Conflict-free Replicated Data Types) allow merging without conflicts for specific data types (counters, sets).
   - Application-level merge requires more complex code but maximum control.

4. **What operations must be atomic?**
   - Inventory decrement + order creation must be atomic.
   - Appending to an activity feed does not need to be atomic across users.

5. **What is your partition tolerance strategy?**
   - Can you queue writes and replay when partition heals?
   - Can you serve reads from a local replica with known staleness?

## SRE Lens: What CAP Violations Look Like in Production

### Cassandra: Stale Reads During Node Failure
An on-call engineer sees customers receiving outdated profile data after a Cassandra node fails. The cluster is AP — it stays up, but with only 2/3 replicas available, some reads are served from nodes that haven't received the latest write. Fix: temporarily switch to QUORUM reads until the node recovers, or investigate if the consistency level tuning matches the replication factor.

### ZooKeeper: Quorum Loss During Network Partition
A ZooKeeper ensemble of 5 nodes loses 3 nodes to a network partition. The entire service coordination layer stops working — services can't acquire locks, leader elections halt. This is correct CP behavior, but the blast radius is enormous. The SRE lesson: CP systems need careful capacity planning. A 5-node ZooKeeper ensemble losing 3 nodes is a severe failure. Consider whether quorum sizes are sized correctly for your failure domain assumptions.

### PostgreSQL Replication Lag Causing Read-Your-Writes Violations
An application reads from a read replica immediately after a write to the primary. The replication lag (even 50ms) means the replica hasn't received the write yet. The user sees their changes disappear. This is a consistency model failure, not a CAP violation per se — but it's caused by the same underlying tension. Fix: route writes and immediate post-write reads to the primary, use read replicas only for operations tolerant of lag.

### Split-Brain in Multi-Master MySQL
Two MySQL masters temporarily lose connectivity. Both continue accepting writes. When the partition heals, they have diverging data. Conflict resolution is undefined. This is the worst CAP outcome: you thought you had a CA system but you actually had an AP system with no conflict resolution strategy.

## Key Takeaways

- CAP theorem describes what must be sacrificed during a **network partition**, not a general design philosophy.
- Partitions are inevitable in distributed systems. The choice is how your system fails, not whether it fails.
- CA systems are single-node or tightly coupled systems that accept a single point of failure in exchange for simplicity.
- CP systems (ZooKeeper, etcd, HBase) stop serving requests when they can't guarantee consistency.
- AP systems (Cassandra, DynamoDB, DNS) stay available but may return stale or conflicting data.
- PACELC is more useful for daily design decisions: even without a partition, strong consistency costs latency.
- Design decisions should be driven by the business impact of each failure mode, not by a label.
- Strong consistency is expensive. Only pay for it where the cost of inconsistency is demonstrably higher.

## References

- Gilbert, S. and Lynch, N. (2002). "Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services." ACM SIGACT News.
- Abadi, D. (2012). "Consistency Tradeoffs in Modern Distributed Database System Design." IEEE Computer.
- Kleppmann, M. — *Designing Data-Intensive Applications*, Chapter 9 (Consistency and Consensus)
- Brewer, E. (2012). "CAP Twelve Years Later: How the 'Rules' Have Changed." IEEE Computer.
- Bailis, P. and Kingsbury, K. (2014). "The Network is Reliable." ACM Queue. — documents real partition events at scale

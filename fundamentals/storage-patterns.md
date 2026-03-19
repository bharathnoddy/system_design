# Storage Patterns

> The on-disk data structures and replication strategies that databases use determine whether your system is fast for reads, fast for writes, or resilient to failures — and you often cannot be all three simultaneously.

## Core Concept

Most developers interact with databases through SQL or an API and treat the storage layer as a black box. But the storage engine's internal data structures have direct consequences for performance, operational behavior, and failure modes. Knowing whether a database uses a B-tree or an LSM-tree tells you whether it will perform better on reads or writes, whether compaction will cause latency spikes, and whether a crash will cause data loss.

## B-Trees vs LSM-Trees

This is the most fundamental storage engine choice in databases today.

### B-Trees

**How they work**: Data is organized in a balanced tree of fixed-size pages (typically 4KB or 8KB on disk). The tree root points to child pages, branching until leaf pages are reached. Leaf pages contain the actual data (or pointers to it in clustered vs non-clustered index distinction).

**Reads**: Navigate the tree from root to leaf. For a balanced tree with N records, this is O(log N) page reads. For a database with 1 billion records and pages of 1000 entries, the tree is 3–4 levels deep. Reading a record requires 3–4 disk I/O operations.

**Writes**: Find the correct leaf page, update it in place. If the page is full, split it into two pages and update the parent. In the worst case, splitting propagates up the tree. The key property: **writes are in-place mutations of existing pages**.

**Write-Ahead Log (WAL) dependency**: Because pages are mutated in place, a crash in the middle of a write can leave the page in a corrupt state. The WAL records the intended change before modifying any page. On recovery, the WAL is replayed to redo or undo the operation.

**Read performance**: Excellent. A key lookup navigates the tree and reads the page. B-trees are well-optimized for random reads.

**Write performance**: Each write may result in several random disk I/O operations (read page, modify, write page back). Under high write throughput, this random I/O pattern becomes a bottleneck, especially on spinning disks. SSDs mitigate this significantly but don't eliminate it.

**Used in**: PostgreSQL, MySQL (InnoDB), SQLite, LMDB, most relational databases.

**Analogy**: A B-tree is like a library with a card catalog. Finding a book (read) is fast. Adding a new book (write) requires finding the right shelf, potentially reshuffling books to make room.

### LSM-Trees (Log-Structured Merge Trees)

**How they work**: Writes go to an in-memory buffer (the memtable), which is a sorted structure. When the memtable fills up, it is flushed to disk as an immutable, sorted file called an SSTable (Sorted String Table). Reads must check the memtable and potentially all SSTables. Periodically, SSTables are merged and compacted.

**Writes**: All writes go to the memtable (in-memory, fast), plus the WAL (append-only, sequential disk write). No random I/O on the write path. **Writes are sequential appends** — the most efficient disk operation.

**Reads**: Check memtable first (most recent data). Then check SSTables from newest to oldest until the key is found. In the worst case, you read from all SSTables. Bloom filters per SSTable avoid reading SSTables that cannot contain the key.

**Compaction**: Without compaction, LSM-trees accumulate many SSTables. Reads degrade as more SSTables must be checked. Compaction merges multiple SSTables into one, discarding deleted keys and resolving updates. Compaction is a background process that consumes I/O and CPU, sometimes causing write latency spikes.

**Write performance**: Excellent. All writes are sequential. LSM-trees are optimized for write-heavy workloads.

**Read performance**: Good with bloom filters and compaction, but a worst-case read requires checking multiple levels of SSTables. B-trees have more predictable read latency.

**Space amplification**: LSM-trees temporarily use more disk space (old versions exist until compaction) — this is space amplification. B-trees have lower space amplification (data is stored once) but higher write amplification.

**Write amplification**: In LSM-trees, each write is written multiple times as it moves through levels during compaction. This is write amplification. It wears out SSDs faster than B-trees in theory, though the sequential nature of LSM writes is gentler than the random writes of B-trees per unit of data.

**Used in**: Cassandra, HBase, LevelDB, RocksDB, Bigtable, InfluxDB.

### Summary

| Property | B-Tree | LSM-Tree |
|---|---|---|
| Write performance | Moderate (random I/O) | Excellent (sequential) |
| Read performance | Excellent (predictable) | Good (varies with compaction) |
| Space amplification | Low | Higher (during compaction) |
| Write amplification | Low-moderate | Higher (data rewritten during compaction) |
| Crash safety | WAL + in-place pages | WAL + immutable SSTables |
| Best for | Read-heavy, OLTP | Write-heavy, time-series, log data |

## SSTables and Memtables

### Memtable
The in-memory write buffer of an LSM-tree storage engine. Writes go here first. Typically implemented as a red-black tree or skip list (both maintain sorted order). When the memtable exceeds a configured size (e.g., 64MB–256MB in Cassandra), it is flushed to disk as an SSTable.

**Durability**: If the process crashes before the memtable is flushed, writes in the memtable would be lost. This is prevented by the WAL — every write to the memtable is first appended to the WAL. On recovery, the WAL is replayed to reconstruct the memtable.

### SSTable (Sorted String Table)
An immutable, sorted file on disk. Structure:
- **Data block**: Key-value pairs, sorted by key.
- **Index block**: A sparse index of keys pointing to positions in the data block.
- **Bloom filter**: A space-efficient probabilistic structure that answers "is key K definitely NOT in this SSTable?" Used to skip SSTables that can't contain a key.
- **Metadata**: Compression info, creation time, etc.

**Why sorted?**: Merging sorted files is O(N) using a k-way merge (like mergesort). This makes compaction efficient.

## Write-Ahead Log (WAL)

### Purpose
The WAL is a sequential, append-only log of all intended changes to the database, written to disk before any actual data structure modification. It is the foundation of crash recovery in both B-tree and LSM-tree databases.

**Sequence of events for a write in PostgreSQL**:
1. Write the intended change to the WAL (sequential disk write).
2. Apply the change to the in-memory buffer pool.
3. At some later point, write the dirty buffer page to its location on disk.
4. Once the page is written to disk, the corresponding WAL entry is no longer needed for recovery.

**On crash**: Replay the WAL from the last checkpoint. Redo committed transactions. Undo uncommitted transactions (using the undo log, which is also in the WAL in some implementations).

### WAL in Replication
The WAL is not just for crash recovery — it is the mechanism for replication in PostgreSQL:
- **Physical replication (streaming replication)**: The WAL stream is sent to standby servers, which replay it to stay in sync.
- **Logical replication**: WAL is decoded into logical change events (INSERT/UPDATE/DELETE rows) for selective replication or CDC (Change Data Capture).

Tools like Debezium tap into the logical WAL to stream database changes to Kafka.

## Sharding Strategies

Sharding is the process of horizontally partitioning data across multiple nodes. Each shard holds a subset of the total data.

### Range-Based Sharding
Partition by a contiguous range of key values. Example: Users A–M on shard 1, N–Z on shard 2.

**Advantages**: Range queries are efficient — they often touch only one shard.  
**Disadvantages**: Hotspots. If your partition key is a timestamp and you're always writing recent data, all writes go to the shard holding the latest range. Also, shards can become unbalanced if data distribution is skewed.

**Used in**: HBase (row key ranges), DynamoDB (sort key ranges within a partition), MongoDB (range sharding).

### Hash-Based Sharding
Apply a hash function to the key. Use the hash value (modulo number of shards) to determine the shard.

**Advantages**: Even distribution of data. No hotspots if the key has high cardinality.  
**Disadvantages**: Range queries touch all shards (scatter-gather). Resharding is expensive — adding a shard requires remapping and moving data.

**Used in**: Cassandra (consistent hashing), Redis Cluster.

### Directory-Based Sharding (Lookup Table)
A central lookup table maps keys to shard IDs. The application queries the directory to find the correct shard.

**Advantages**: Flexibility. You can arbitrarily assign keys to shards, making resharding easier (just update the directory).  
**Disadvantages**: The directory is a single point of failure and a potential bottleneck if every request must consult it.

### Consistent Hashing
A variant of hash sharding that minimizes data movement when nodes are added or removed. Keys and nodes are mapped to positions on a ring. A key is served by the first node clockwise from its position on the ring.

When a node is added, only the keys in the range between the new node and its predecessor on the ring need to be remapped. When a node is removed, only its keys need to be redistributed.

**Used in**: Cassandra (with virtual nodes for better balance), Amazon DynamoDB, Chord DHT.

## Hotspot Problems and Mitigations

### What Causes Hotspots
- **Range sharding on monotonic keys**: All new writes to the latest range.
- **Popular keys in hash sharding**: A viral celebrity's Twitter feed. One partition key receiving 10,000x the average traffic.
- **Celebrity rows**: A single user with 50 million followers. Any operation on their data is disproportionately expensive.

### Mitigations
- **Key salting**: Append a random suffix to hotspot keys. `user:12345` → `user:12345:3`. Distribute the load across multiple keys, then aggregate reads. Complexity: writes are fast, reads require aggregation.
- **Consistent hashing with virtual nodes**: Each physical node owns multiple virtual nodes on the ring. New nodes can be inserted at multiple points, taking load from multiple physical nodes instead of just one neighbor.
- **Application-level caching**: Cache the result of the hotspot key at the application layer (local in-process cache). Reduces DB load.
- **Read replicas for hotspot data**: Direct reads for the celebrity user to dedicated read replicas.
- **DynamoDB adaptive capacity**: DynamoDB automatically redistributes capacity from cold to hot partitions. Helps with moderate hotspots.

## Replication

Replication serves two purposes: fault tolerance (data survives node failure) and read scaling (distribute reads across replicas).

### Single-Leader Replication
One node (the leader/primary) accepts all writes. Changes are replicated to followers (replicas). Followers serve reads.

**Replication lag**: Async replication means followers are behind the leader by some amount (milliseconds to seconds). During this lag window, reads from followers may return stale data.

**Failover**: If the leader fails, a follower must be promoted. This involves:
1. Detecting the leader failure (timeout-based, usually 30 seconds in practice).
2. Electing a new leader (Raft/Paxos or manual).
3. Reconfiguring all followers to replicate from the new leader.
4. Clients must redirect writes to the new leader.

**Risk**: If the old leader comes back before learning it was replaced, you have two nodes that believe they are the primary — split-brain. Fencing tokens or STONITH (Shoot The Other Node In The Head) power control mechanisms are used to prevent this.

**Used in**: PostgreSQL streaming replication, MySQL replication, MongoDB replica sets.

### Multi-Leader Replication
Multiple nodes accept writes simultaneously. Changes are replicated to all other leaders.

**Use case**: Multi-datacenter deployments. Each datacenter has a local leader. Writes go to the local leader (low latency). Leaders replicate to each other async (cross-datacenter is slow but non-blocking).

**Problem**: Write conflicts. If two leaders accept writes to the same key concurrently, how is the conflict resolved? Options: last-write-wins (data loss), custom merge, operational transformation (for documents).

**Used in**: MySQL Group Replication (Galera), CouchDB, some multi-region PostgreSQL setups.

### Leaderless Replication
Any node can accept reads and writes. Writes go to N replicas (determined by configuration). Reads query R replicas and take the most recent value.

**Quorum consistency**: If W + R > N, at least one node in every read set must have seen the latest write. For N=3, W=2, R=2: any read will overlap with any write by at least 1 node.

**Sloppy quorum and hinted handoff**: If some nodes are unavailable, writes go to other nodes temporarily (sloppy quorum). When the target node recovers, the write is forwarded (hinted handoff). This improves availability but may cause reads to miss recent writes temporarily.

**Used in**: Cassandra, Riak, DynamoDB (under the hood).

## Replication Lag and Its Consequences

Replication lag is the delay between a write being applied to the primary and being visible on a replica.

**Operational causes of high lag**:
- Slow replica (underpowered hardware).
- Long-running transactions blocking replication.
- High write volume saturating the replication link.
- Network congestion.

**User-visible consequences**:
- **Read-your-writes violation**: User updates their profile, reads from replica, sees old profile.
- **Monotonic reads violation**: User sees 100 messages, refreshes, sees 95 messages (from a more lagged replica).
- **Causality violation**: User A sends a message. User B replies. User C sees B's reply before A's original message.

**Monitoring**: In PostgreSQL: `pg_stat_replication` view shows replication lag in bytes and seconds. Alert when lag exceeds your SLO.

## Compaction Strategies

Compaction is the process of merging SSTables in LSM-tree storage engines. The strategy affects read amplification, write amplification, and space amplification.

### Size-Tiered Compaction (STCS)
Group SSTables by size. When there are N SSTables of similar size, merge them into one larger SSTable.

**Characteristics**: Low write amplification (few rewrites). High read amplification and space amplification (many SSTables to search). Good for write-heavy workloads. Default in Cassandra for some table types.

### Leveled Compaction (LCS)
Organize SSTables into levels. Each level has a size limit. Level 0 is the most recently flushed data. When a level fills up, SSTables are merged into the next level.

**Characteristics**: Low read amplification (at most one SSTable per level for any key). Higher write amplification (keys are rewritten as they move through levels). Good for read-heavy workloads. Default in LevelDB and RocksDB.

### TWCS (Time Window Compaction Strategy)
Used in Cassandra for time-series data. SSTables are grouped by time window (e.g., 1 hour). Data within a time window is compacted together. Old time windows are never compacted with new ones.

**Characteristics**: Enables efficient TTL expiration (an entire time-window SSTable can be deleted at once). Low compaction overhead for time-series workloads. Breaks down if data is not written in time order.

## Bloom Filters

### What Problem They Solve
In an LSM-tree, reading a key that doesn't exist (a negative lookup) is expensive — you must check every SSTable and read through them to confirm the key is absent.

A Bloom filter is a space-efficient probabilistic data structure that answers: **"Is key K definitely NOT in this SSTable?"**

### How It Works
- A Bloom filter is a bit array of M bits, all initialized to 0.
- K hash functions map each key to K positions in the bit array, and set those bits to 1.
- **To query**: Hash the key with all K functions. If any bit position is 0, the key is **definitely not** in the set.
- **If all bits are 1**: The key is **probably** in the set (false positive possible).

**There are never false negatives**: If a key is in the set, all its bits are 1. A Bloom filter will never say "not present" for a key that is present.

**False positive rate**: Depends on M (array size), K (hash functions), and N (number of elements inserted). More elements = more false positives. Adding space reduces false positive rate.

### Use in Databases
- **Cassandra**: Each SSTable has a Bloom filter. Before reading from an SSTable, query its Bloom filter. If the result is "definitely not here," skip the SSTable. Under typical settings, Bloom filters eliminate 90–99% of unnecessary SSTable reads.
- **RocksDB**: Bloom filters per SSTable level.
- **PostgreSQL**: No bloom filters by default (B-tree doesn't need them for the same reason). The `bloom` extension provides a bloom filter index type for multi-column equality queries.

### False Positive Trade-off
A typical Bloom filter with 10 bits per element achieves a ~1% false positive rate. Doubling to 20 bits per element reduces it to ~0.01%. There's a memory-accuracy trade-off: larger filter = fewer false positives = more reads skipped.

## Column-Oriented Storage

### Why It's Faster for Analytics

Row-oriented storage (OLTP): A row's columns are stored contiguously. When you read a row, you get all its columns.

Column-oriented storage (OLAP): Each column is stored separately. All values for column A, then all values for column B.

**For analytical queries that read few columns across many rows**:
```sql
SELECT AVG(price), category FROM orders GROUP BY category;
```
A row store reads every column of every row and discards most of the data. A column store reads only the `price` and `category` columns — potentially 10–100x less I/O.

**Compression**: Column values tend to be similar (same category strings, prices in a range). Column storage achieves dramatically better compression:
- **Run-length encoding**: `[200 × "US", 150 × "EU"]` stores 350 values in 4 bytes.
- **Dictionary encoding**: Store unique values once, use integer IDs elsewhere.
- **Bitpacking**: Fixed-width integers can be packed densely.

**Vectorized processing**: Queries operate on column arrays. Modern CPUs can apply SIMD (Single Instruction, Multiple Data) operations to process 256 or 512 bits at a time. Working on a column array of integers is highly cache-friendly.

**Used in**: Apache Parquet (columnar file format), ClickHouse, Redshift, BigQuery, Snowflake, Apache ORC, DuckDB.

## How This Appears in Real Systems

**Cassandra LSM-Tree in Production**: A Cassandra cluster is write-heavy (IoT telemetry). The team notices occasional high read latency every few hours. The culprit is size-tiered compaction running in the background, consuming disk I/O. Fix: switch to leveled compaction for better read latency, or tune compaction throughput limits to prevent I/O saturation.

**PostgreSQL WAL for CDC**: A team uses Debezium to stream PostgreSQL changes to Kafka. Debezium reads the logical WAL replication slot. If the consumer falls behind, PostgreSQL retains WAL segments to ensure the consumer can catch up — this can fill the disk. Monitor replication slot lag as a critical operational metric.

## SRE Lens: What This Looks Like in Production

### Compaction-Induced Write Stalls
An SRE observes periodic 30-second spikes in Cassandra write latency every 4 hours. `nodetool compactionstats` shows major compactions running. The compaction is consuming the disk I/O budget. Fix: configure `compaction_throughput_mb_per_sec` to throttle compaction, stagger compaction schedules across nodes, or add faster storage.

### B-Tree Bloat After DELETE-Heavy Workload
A PostgreSQL table storing events has 50% of rows deleted as events are processed. The table is 500GB but only 250GB is live data. Index scans are slow because index pages are half empty (fragmented). `VACUUM FULL` reclaims space but takes an exclusive lock. Fix: use partial indexes, partition the table by time and drop old partitions, or schedule `CLUSTER` operations during low-traffic windows.

### Replication Slot Bloat Killing Disk Space
A PostgreSQL replica's Debezium consumer is down for 6 hours due to a deployment bug. PostgreSQL retains WAL segments in the replication slot to allow the consumer to resume. 100GB of WAL segments accumulate. If disk fills, PostgreSQL stops accepting writes entirely. Fix: monitor replication slot lag, set `max_slot_wal_keep_size` as a safety limit.

## Common Misconceptions

- **"LSM-trees are strictly better for writes"**: True for sequential writes. For mixed random read/write workloads, B-trees are more predictable. LSM-tree compaction can cause write stalls.
- **"Bloom filters can tell you if a key exists"**: Bloom filters only guarantee absence. A positive result is probabilistic.
- **"Sharding is easy to add later"**: Resharding an existing production database is one of the most painful operations in distributed systems. Design sharding strategy early.
- **"Read replicas eliminate replication lag problems"**: Read replicas reduce lag on average but don't eliminate the lag window. Applications that require read-your-writes must not use read replicas naively.
- **"The WAL is just for crash recovery"**: The WAL is the primary mechanism for replication, CDC, and point-in-time recovery. It is much more than a crash safety mechanism.

## Key Takeaways

- B-trees optimize for reads (in-place updates, predictable lookup depth). LSM-trees optimize for writes (sequential appends, memtable buffering).
- The WAL is the source of truth for crash recovery and is the foundation of streaming replication and CDC.
- Sharding distributes data; hash sharding distributes evenly; range sharding enables range queries; consistent hashing minimizes data movement during topology changes.
- Replication lag causes read-your-writes violations. Monitor it actively.
- Bloom filters eliminate unnecessary reads in LSM-tree databases by quickly confirming key absence.
- Column-oriented storage is 10–100x faster for analytical queries over wide tables because it reads only the columns needed and compresses them effectively.
- Compaction is the operational heartbeat of LSM-tree databases. Tuning it incorrectly causes write stalls or read amplification.

## References

- Kleppmann, M. — *Designing Data-Intensive Applications*, Chapters 3 and 5
- O'Neil, P. et al. (1996). "The Log-Structured Merge-Tree (LSM-Tree)." Acta Informatica.
- Chang, F. et al. (2006). "Bigtable: A Distributed Storage System for Structured Data." OSDI.
- Bloom, B.H. (1970). "Space/time trade-offs in hash coding with allowable errors." Communications of the ACM.
- RocksDB documentation — https://rocksdb.org/ — particularly the "Compaction" section
- Apache Cassandra documentation — Storage Engine section

# Databases

> Choosing the right database is one of the highest-leverage decisions in system design — the wrong choice creates performance ceilings, operational complexity, and migration work that compounds over years.

## Core Concept

There is no universally "best" database. Every database type optimizes for a specific set of access patterns. The decision framework is: **what are your reads, what are your writes, what are your queries, and what consistency do you need?** Database selection is a consequence of answering those questions honestly, not a preference for a technology.

The mistake engineers make is defaulting to what they know (or what the team uses for other services) rather than matching the data model to the problem.

## Relational Databases (PostgreSQL, MySQL)

### What They Are
Tables with rows and columns, strict schemas, foreign key relationships, and ACID transactions. SQL as the query language.

### When to Use
- **Structured data with well-defined relationships**: Orders, customers, products, payments.
- **Complex queries**: Joins across multiple tables, aggregations, window functions.
- **ACID transactions are required**: Anything involving money, inventory, or state that must be atomically consistent.
- **The access pattern is read-heavy with ad-hoc queries**: Analysts can write SQL; you don't need to predict every query upfront.

### Indexing
- **B-tree index** (default): Good for equality and range queries (`WHERE id = 5`, `WHERE created_at > '2024-01-01'`). The default choice.
- **Hash index**: O(1) equality lookups only. Cannot do range queries. Used internally by some engines.
- **GIN (Generalized Inverted Index)**: For array containment, full-text search, JSONB queries in PostgreSQL.
- **BRIN (Block Range Index)**: For naturally ordered large tables (e.g., time-series in PostgreSQL). Very small, works by storing min/max per block.
- **Partial index**: Index only rows matching a condition. `CREATE INDEX ON orders(id) WHERE status = 'pending'`. Dramatically smaller and faster for selective queries.
- **Composite index**: Column order matters. A composite index on `(user_id, created_at)` supports queries filtering by `user_id` alone, but not `created_at` alone (unless the optimizer uses it as a partial scan).

### ACID
- **Atomicity**: All operations in a transaction succeed or all fail. No partial writes.
- **Consistency**: Transaction brings the DB from one valid state to another. Constraints (FK, NOT NULL, UNIQUE) are enforced.
- **Isolation**: Concurrent transactions don't interfere. Isolation levels: READ UNCOMMITTED, READ COMMITTED (PostgreSQL default), REPEATABLE READ, SERIALIZABLE.
- **Durability**: Once committed, data survives crashes. Ensured by WAL (Write-Ahead Log).

### When PostgreSQL Shines
- Full-text search (GIN + tsvector) as a lightweight alternative to Elasticsearch for moderate scale.
- JSONB for semi-structured data without sacrificing ACID.
- PostGIS extension for geospatial queries.
- Partitioning for large tables with range-based access.

### Operational Concerns
- **Vacuum**: PostgreSQL's MVCC creates dead tuples. AUTOVACUUM reclaims space but can cause I/O spikes. Monitor `n_dead_tup`.
- **Connection limits**: PostgreSQL creates one process per connection. At 500+ concurrent connections, use PgBouncer connection pooling.
- **Table bloat**: Heavy UPDATE/DELETE workloads create bloat. `pg_bloat_check` or `pgstattuple` helps monitor.

## Wide-Column Databases (Cassandra, HBase, Bigtable)

### What They Are
Data is organized in tables, but each row can have a different set of columns. Rows are identified by a partition key. Columns within a row are sorted. Optimized for writing and reading by primary key.

### The Data Model
Think of it as a distributed sorted map:
```
(partition_key, clustering_key) → value
```

In Cassandra:
- **Partition key**: Determines which node holds the data. All rows with the same partition key are on the same node (and its replicas).
- **Clustering key**: Sorts rows within a partition. Enables efficient range scans within a partition.

### When to Use
- **Known access patterns**: You design the table for the query, not the other way around. In Cassandra, you often have one table per query pattern.
- **Time-series or append-heavy workloads**: IoT sensor data, event logs, metrics, time-ordered activity feeds.
- **High write throughput**: LSM-tree storage means writes are sequential. Cassandra can sustain millions of writes/second.
- **Massive scale with no single point of failure**: Cassandra is peer-to-peer with no leader. Scale by adding nodes.

### What They Cannot Do
- Ad-hoc queries (no joins, no complex aggregations).
- Cross-partition transactions (no multi-row ACID).
- Efficient queries that weren't anticipated in the schema design.

### Cassandra Example — Time-Series Access Pattern
```cql
CREATE TABLE sensor_readings (
    sensor_id UUID,
    reading_time TIMESTAMP,
    temperature FLOAT,
    PRIMARY KEY (sensor_id, reading_time)
) WITH CLUSTERING ORDER BY (reading_time DESC);
```
This design stores all readings for a sensor on the same partition, sorted by time. Reading the last N readings for a sensor is O(1) — a single partition lookup.

## Document Databases (MongoDB)

### What They Are
Data is stored as JSON-like documents (BSON in MongoDB). Schema is flexible — different documents in the same collection can have different fields.

### When to Use
- **Semi-structured or variable data**: Product catalogs with different attributes per product category, user preferences, CMS content.
- **Rapid iteration**: Schema can evolve without migrations. Good for early-stage products where the data model is changing.
- **Hierarchical data**: Nested documents avoid joins. An order document can embed its line items.
- **The data is naturally document-shaped**: A blog post with its author, tags, and comments all in one document.

### When It's the Wrong Choice
- Highly relational data with complex joins (use PostgreSQL).
- Financial transactions requiring multi-document ACID (MongoDB 4.0+ supports multi-document transactions, but they're expensive).
- Very high write throughput with known access patterns (use Cassandra).

### Common Mistake
Treating MongoDB's flexible schema as an excuse not to think about the data model. Without schema validation, collections degrade into inconsistent messes. Use MongoDB's schema validation or enforce it at the application layer.

## Key-Value Databases (Redis, DynamoDB)

### What They Are
The simplest database model: a key maps to a value. Operations are typically GET, SET, DELETE by key.

### Redis
- In-memory store with optional persistence.
- Rich data structures: strings, hashes, lists, sets, sorted sets, HyperLogLog, streams, pub/sub.
- Sub-millisecond latency.
- Use cases: caching, session storage, rate limiting (sliding window with sorted sets), leaderboards (sorted sets), real-time pub/sub, distributed locks (Redlock).
- **Not a primary database**: Redis is persistence-optional. AOF and RDB persistence exist, but Redis is designed for in-memory access. Treat it as a fast cache layer.

### DynamoDB
- Fully managed AWS key-value/document store.
- Scales to any throughput and storage automatically.
- Single-digit millisecond latency at any scale.
- Data model: partition key (required) + optional sort key. Items in a partition are sorted by sort key.
- Secondary indexes: LSI (same partition key, different sort key) and GSI (different partition key).
- Billing: on-demand or provisioned capacity units.
- Use cases: user sessions, shopping carts, gaming leaderboards (with sort key), IoT device state.

### When Key-Value Is Appropriate
- Lookups are always by a known key.
- No need for complex queries.
- Maximum speed and scale is the priority.
- The data has a natural key-value shape (session ID → session data).

## Time-Series Databases (InfluxDB, TimescaleDB, Prometheus)

### What They Are
Optimized for storing and querying time-stamped data: metrics, monitoring data, financial ticks, IoT sensor readings.

### Why Not Just Use PostgreSQL?
Time-series data has specific patterns that generic databases handle poorly:
- **Write-heavy**: Continuous ingestion of high-frequency samples.
- **Recent data is hot**: 99% of queries are for the last hour/day/week.
- **Compression opportunities**: Adjacent time-series values are often similar. Delta encoding achieves 10–100x compression.
- **Downsampling**: Old data can be summarized. No need for per-second data from 2 years ago.
- **Automatic expiry (retention policies)**: Data older than 30 days is automatically deleted.

### Prometheus
- Pull-based metrics collection. Prometheus scrapes your services.
- TSDB (time-series DB) with high compression.
- PromQL for queries and alerting.
- Not designed for long-term storage (TSDB is local disk). Use Thanos or Cortex for long-term remote storage.
- Very high cardinality is a known problem — each unique label combination is a separate time series.

### TimescaleDB
- PostgreSQL extension. Time-series tables are automatically partitioned by time (hypertables).
- Full SQL support. Compression and retention policies built in.
- Best of both worlds: familiar PostgreSQL interface with time-series optimizations.
- Good for applications that need time-series + relational data in the same database.

### InfluxDB
- Purpose-built TSDB. InfluxQL or Flux query language.
- Built-in retention policies and continuous queries for downsampling.
- Better raw performance than TimescaleDB for pure time-series workloads.

## Graph Databases (Neo4j, Neptune)

### What They Are
Data is stored as nodes and relationships (edges). Both nodes and edges have properties. Query language (Cypher for Neo4j) traverses the graph.

### When to Use
- **Relationships are the primary data**: Social networks (friends of friends), fraud detection (who transacted with whom), recommendation engines (users who bought X also bought Y), knowledge graphs.
- **Recursive or deep traversals**: "Find all people within 3 hops of User X." In SQL, this requires expensive recursive CTEs. In a graph DB, it's natural.
- **Highly connected data**: If your relational schema has many-to-many join tables everywhere, and your queries traverse those relationships, a graph DB may be more natural.

### When Not to Use
- Simple CRUD with no relational complexity.
- High write throughput of simple records.
- Graph databases typically don't scale horizontally as easily as wide-column stores.

## Search Databases (Elasticsearch)

### What It Is
Elasticsearch is built on Apache Lucene. It maintains an inverted index — for each word/token, it stores the list of documents containing that word. This enables full-text search, relevance scoring, and fast faceting.

### Inverted Index
```
"distributed" → [doc1, doc3, doc7]
"systems"     → [doc1, doc2, doc3]
"design"      → [doc2, doc5]
```
A query for "distributed systems" finds the intersection of these lists and scores by relevance (TF-IDF or BM25).

### When to Use
- Full-text search with relevance ranking.
- Log analytics (ELK stack: Elasticsearch, Logstash, Kibana).
- Complex faceted search (filter by category, price range, tags simultaneously).
- Auto-complete and typeahead.

### What Elasticsearch Is NOT
- **Not a primary database**: Elasticsearch is eventually consistent and does not support ACID transactions. It is excellent for search but should be populated from a primary store (PostgreSQL, MongoDB) via CDC or an ETL pipeline.
- **Not for low-latency key-value lookups**: Redis is dramatically faster for simple key lookups.
- **Not for complex relational queries**: No joins.

### Operational Concerns
- Mapping explosions: unbounded dynamic field mapping can create tens of thousands of fields, degrading cluster performance.
- Shard sizing: too many small shards or too few large shards both cause problems. Target 10–50GB per shard.
- JVM heap: typically set to 50% of RAM, max 32GB (to stay in compressed OOP range).

## Vector Databases (Pinecone, pgvector, Weaviate, Qdrant)

### What They Are
Store high-dimensional embedding vectors and support approximate nearest neighbor (ANN) search — finding the K most similar vectors to a query vector by cosine similarity, dot product, or Euclidean distance.

### The Use Case
Large language models convert text, images, or any data into dense vector embeddings. To implement semantic search, recommendation, or RAG (Retrieval-Augmented Generation), you need to find the K nearest vectors to a query embedding efficiently.

Brute force comparison of a query against 10 million 1536-dimension vectors is computationally infeasible at query time. ANN algorithms (HNSW, IVF-Flat, FAISS) sacrifice a small amount of recall accuracy for dramatically better performance.

### Options
- **pgvector**: PostgreSQL extension. Good for moderate scale (< 10M vectors) where you want ANN search alongside relational data. Uses HNSW or IVF indexing.
- **Pinecone**: Managed, cloud-native vector DB. Easy to use, scales automatically. No infrastructure management.
- **Weaviate**: Open-source, supports storing objects with vectors. Built-in modules for vectorization.
- **Qdrant**: High-performance open-source vector DB. Excellent for self-hosted deployments.

### When to Use
- Semantic search (find documents similar in meaning, not just matching keywords).
- RAG for LLM applications.
- Recommendation systems based on content similarity.
- Anomaly detection (find vectors far from their nearest neighbors).
- Face recognition, image search.

## NewSQL (CockroachDB, Google Spanner)

### What They Are
Distributed SQL databases that provide ACID transactions across multiple nodes globally. They combine the horizontal scalability of NoSQL with the full SQL semantics and strong consistency of relational databases.

### Google Spanner
- Google's globally distributed relational database.
- External consistency (stronger than serializable) across global replicas.
- TrueTime API for clock synchronization.
- Full SQL support.
- Used for Google's Ads, F1 (Ads backend), and many internal systems.
- Available as Cloud Spanner on GCP.

### CockroachDB
- Open-source distributed SQL database.
- PostgreSQL-compatible wire protocol.
- Automatic sharding and rebalancing.
- Serializable isolation.
- Multi-region deployments with configurable data domicile.
- Resilient to node and zone failures while maintaining ACID guarantees.

### When to Use
- You need horizontal scaling AND strong consistency AND SQL.
- Global applications that need low-latency reads near users but globally consistent writes.
- Multi-region financial systems.

### The Cost
- Higher write latency than single-node PostgreSQL (consensus overhead).
- More operational complexity.
- Not every PostgreSQL feature is supported.

## Decision Framework: How to Pick the Right Database

### Ask These Questions

1. **What are my primary access patterns?**
   - Always-by-key → Key-value (Redis, DynamoDB)
   - Time-ordered, append-heavy → Time-series or Cassandra
   - Full-text search → Elasticsearch
   - Complex relational queries → PostgreSQL
   - Relationship traversal → Graph DB

2. **What is my write volume?**
   - Very high (millions/s) → Cassandra, DynamoDB
   - Moderate → PostgreSQL, MongoDB
   - In-memory speed → Redis

3. **What consistency do I need?**
   - ACID transactions → PostgreSQL, CockroachDB, Spanner
   - Eventual consistency is fine → Cassandra, DynamoDB
   - Tunable → Cassandra (per-query consistency level)

4. **What is my data model?**
   - Tabular, relational → PostgreSQL
   - JSON documents, variable schema → MongoDB
   - Wide rows with known keys → Cassandra
   - Nested + hierarchical → Document DB

5. **What is my scale requirement?**
   - Single server → PostgreSQL (can get very far on a good server)
   - Multi-server, no complex queries → Cassandra, DynamoDB
   - Global with strong consistency → Spanner, CockroachDB

6. **Is this a primary store or a secondary store?**
   - Redis: secondary cache.
   - Elasticsearch: secondary search index.
   - Prometheus: secondary metrics store.
   - Everything else: can be primary.

### The "Use PostgreSQL Until You Can't" Rule
PostgreSQL is remarkably capable. A well-indexed PostgreSQL instance on good hardware can handle thousands of writes per second and millions of reads. Most applications never outgrow it. Start with PostgreSQL and migrate to a specialized system only when you have a concrete bottleneck that cannot be resolved with better indexes, partitioning, or read replicas.

## Common Mistakes

- **Using Redis as a primary data store**: Redis is a cache with optional persistence. If you rely on it as your database of record and it restarts without AOF, you lose all data.
- **Using MongoDB because "we need flexibility"**: Schema flexibility without discipline creates unmaintainable data. MongoDB is a good choice; choosing it for the wrong reason leads to pain.
- **Elasticsearch as primary storage**: Elasticsearch is not a source of truth. It is a search index populated from a primary store.
- **Cassandra for highly relational data**: Cassandra has no joins. Modeling relational data in Cassandra requires denormalization and pre-computed joins. If your queries are ad-hoc, Cassandra is the wrong tool.
- **Scaling the wrong axis**: Vertical scaling a database (bigger machine) is often cheaper and simpler than introducing a new distributed database. Don't introduce operational complexity until you need to.
- **Single database for all workloads**: OLTP (transactional) and OLAP (analytical) have opposite optimization profiles. Mixing them on the same database causes table scans to destroy OLTP latency. Use read replicas or a separate data warehouse for analytics.

## SRE Lens: What This Looks Like in Production

### Cassandra Hotspot from Poor Partition Key Choice
An SRE investigates uneven CPU and disk usage across a Cassandra ring. One node is at 90% load; others are at 20%. Root cause: the partition key is `user_country`, and 60% of users are in "US". All US user data lands on a small subset of nodes. Fix: use a high-cardinality partition key or add a random bucket suffix to distribute load.

### PostgreSQL Connection Exhaustion
An on-call alert fires: `FATAL: remaining connection slots are reserved for non-replication superuser connections`. The app is creating a new DB connection per API request instead of pooling. Under load, all 100 available PostgreSQL connections are consumed. Fix: add PgBouncer in transaction pooling mode, then tune max connections.

### Elasticsearch Mapping Explosion
An ops team enables dynamic mapping on a log index and starts indexing JSON logs with arbitrary field names (e.g., user-defined metric names). Over weeks, the index accumulates 50,000+ fields. Cluster performance degrades. Fielddata cache memory is exhausted. Fix: disable dynamic mapping, use `object` type with fixed schema, move arbitrary key-value to a `labels` field with `keyword` type.

## Key Takeaways

- No database is universally best. Match the data model and access pattern to the database type.
- PostgreSQL is more capable than most engineers assume. Scale it before migrating to a distributed database.
- Cassandra is for high write throughput with known, key-based access patterns. It is not a replacement for PostgreSQL.
- Redis is a cache, not a primary database. Treat it as such.
- Elasticsearch is a search index populated from a primary store. It is not a source of truth.
- OLTP and OLAP need different systems. Don't run analytics queries on your production transactional database.
- Vector databases are emerging as infrastructure for AI/ML workloads. pgvector is a good starting point for moderate scale.
- The decision framework: access patterns → consistency needs → write volume → data model → scale requirements.

## References

- Kleppmann, M. — *Designing Data-Intensive Applications* — the definitive reference for all of this
- Corbett, J. et al. (2012). "Spanner: Google's Globally-Distributed Database." OSDI.
- Chang, F. et al. (2006). "Bigtable: A Distributed Storage System for Structured Data." OSDI.
- DeCandia, G. et al. (2007). "Dynamo: Amazon's Highly Available Key-value Store." SOSP.
- PostgreSQL documentation — https://www.postgresql.org/docs/
- Apache Cassandra documentation — https://cassandra.apache.org/doc/

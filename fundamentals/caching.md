# Caching

> Caching is trading memory for latency — but the patterns you use, the eviction policies you choose, and the failure modes you plan for determine whether a cache makes your system faster and more resilient or more fragile and harder to debug.

## Core Concept

A cache is a fast, temporary storage layer that holds copies of data to reduce the cost (latency, compute, I/O) of fetching it from the original source. Caches work when data is accessed repeatedly before it changes — the "temporal locality" principle.

The benefits of caching are obvious. The risks are less visible: stale data, cache stampedes, cold starts, and subtle consistency bugs that only appear under load. A cache is not "just adding Redis in front of the DB." It is a stateful component with its own failure modes.

**Cache hit rate** is the primary metric: `hits / (hits + misses)`. A well-tuned cache for a hot workload should achieve 80–99% hit rate. Below 70%, the cache may be adding latency overhead without proportional benefit.

## Cache-Aside (Lazy Loading)

### How It Works
The application manages the cache directly. The cache is not involved in the write path.

**Read**:
1. Check the cache. If a hit, return the cached value.
2. If a miss, read from the database.
3. Store the result in the cache with a TTL.
4. Return the value.

**Write**:
1. Write directly to the database.
2. Invalidate (delete) or update the corresponding cache entry.

### Pros
- Cache only contains data that has actually been requested — no wasted space on unread data.
- If the cache is unavailable, reads fall through to the database (degraded performance, not failure).
- Works with any database without requiring special support.

### Cons
- **Initial cold-miss latency**: First read of any key hits the database. Under high traffic, a hot key causes many DB reads before it's cached.
- **Potential for stale data**: If a write updates the DB but fails to invalidate the cache, stale data persists until TTL expires.
- **Race condition**: Thread A reads a miss, fetches from DB. Thread B updates the DB and deletes the cache entry. Thread A stores the now-stale value in the cache. The stale value lives until TTL.

### When to Use
The most common caching pattern. Good for read-heavy workloads where some staleness is acceptable. Most Redis caching implementations use this pattern.

## Write-Through

### How It Works
Every write goes to the cache and the database simultaneously (or the cache always stays in sync with the DB).

**Write**:
1. Write to the cache.
2. Write to the database (synchronously).
3. Acknowledge to the client only after both succeed.

**Read**: Check cache first. High hit rate because cache is always populated on write.

### Pros
- Cache is always up to date. No stale reads.
- Subsequent reads after a write always hit cache.

### Cons
- **Write latency**: Every write incurs the overhead of writing to both cache and DB.
- **Unused data in cache**: Writes populate the cache even for data that might never be read again. Wastes cache memory.
- **Write amplification**: Every write touches two systems, increasing the failure surface.

### When to Use
Appropriate when read-after-write consistency is important and write latency overhead is acceptable. Often combined with cache-aside for reads.

## Write-Behind (Write-Back)

### How It Works
Writes go to the cache only. The cache asynchronously flushes writes to the database in batches or after a delay.

**Write**:
1. Write to the cache.
2. Acknowledge to the client immediately.
3. Cache asynchronously writes to the database (batched, delayed).

### Pros
- **Lowest write latency**: Client gets an ack as soon as the cache is written.
- **Batching benefit**: Multiple updates to the same key between flushes result in only one DB write (reduces write amplification to DB).

### Cons
- **Durability risk**: Data in the cache buffer that hasn't been flushed to DB is lost on cache failure. This is a serious risk if the cache is in-memory only.
- **Complexity**: The async flush logic must handle failures, ordering, and retries.
- **Not suitable for data that cannot be lost**: Financial transactions, critical state changes.

### When to Use
Analytics counters, high-throughput event logging, user activity tracking — workloads where throughput is critical and losing a few seconds of data is acceptable. Rarely appropriate for primary transactional data.

## Read-Through

### How It Works
The cache sits between the application and the database. On a cache miss, the cache itself fetches the data from the database, stores it, and returns it.

**Read**:
1. Application requests a key from the cache.
2. If a hit, the cache returns the value.
3. If a miss, the cache fetches from the database, stores the value, and returns it.

The application never directly queries the database.

### Pros
- Application code is simpler — it only talks to the cache, never the database directly.
- Cache miss logic is centralized.

### Cons
- First request for a key is always slow (cold miss).
- Requires a cache layer that understands how to fetch from your specific data source (e.g., a smart caching proxy like DAX for DynamoDB, or a custom cache implementation).

### When to Use
Managed cache layers: Amazon DAX (DynamoDB Accelerator) is a read-through cache. Some ORM-level caches. When you want to abstract database access entirely behind a cache interface.

## Eviction Policies

When a cache reaches capacity, it must evict entries to make room. The eviction policy determines which entries are removed.

### LRU (Least Recently Used)
Evicts the entry that was accessed least recently. Assumes that recently accessed data is more likely to be accessed again.

**Implementation**: Maintain a doubly-linked list and a hash map. The linked list orders entries by recency. O(1) for get and put.

**Best for**: General-purpose caching for workloads with temporal locality — web session caches, CDN content, API response caches.

**Weakness**: Not optimal for scan patterns. A full table scan reads every row once, filling the cache with data that won't be re-read, evicting actually hot data. This is the "cache pollution" problem. PostgreSQL uses a "clock sweep" variant to handle this.

### LFU (Least Frequently Used)
Evicts the entry that has been accessed the fewest number of times.

**Best for**: Workloads with a stable "hot set" — certain data is always popular, and you want to keep it regardless of recency. Good for content recommendation engines where popular articles stay popular.

**Weakness**: New data starts with a count of 1 and gets evicted quickly even if it will be heavily accessed. "Frequency aging" (decaying counts over time) mitigates this.

### FIFO (First In, First Out)
Evicts the entry that was added to the cache first, regardless of access pattern.

**Best for**: Simple, low-overhead caching where access patterns are unpredictable and the cache is being used primarily as a write buffer.

**Weakness**: May evict frequently accessed entries just because they've been in the cache a long time.

### TTL-Based Eviction
Entries expire after a configured time-to-live regardless of access frequency or recency.

**Best for**: Data with a known maximum acceptable staleness — session tokens, rate limit counters, API rate limit windows, weather data, stock prices.

**TTL selection**: Too short → frequent cache misses, high DB load. Too long → stale data. A good rule: set TTL to the maximum staleness the application can tolerate.

### Redis-Specific Policies
Redis allows configuring `maxmemory-policy`. Options include: `noeviction` (errors when full), `allkeys-lru`, `volatile-lru` (LRU only among keys with TTL), `allkeys-random`, `volatile-random`, `volatile-ttl` (evict keys closest to TTL expiration).

## Cache Stampede / Thundering Herd

### What Causes It
A cache stampede occurs when a heavily accessed cached key expires and many concurrent requests simultaneously attempt to regenerate it from the database.

**Scenario**: 10,000 requests per second are hitting a cache for the top products list. The entry's TTL expires. All 10,000 in-flight requests get a cache miss simultaneously. All 10,000 attempt to fetch the same data from the database. The database, which normally handles maybe 100 QPS for this query, receives 10,000 requests at once. It collapses.

### Solutions

**Mutex Lock (Cache Locking)**:
When a miss occurs, acquire a distributed lock before fetching from the database. Other threads that also get a miss wait for the lock. The first thread fetches, stores in cache, and releases the lock. All waiting threads then hit the cache.

Risk: If the DB fetch is slow, all threads wait behind the lock. Adds latency but prevents stampede.

```python
def get(key):
    value = cache.get(key)
    if value:
        return value
    lock_key = f"lock:{key}"
    if cache.set(lock_key, 1, nx=True, ex=5):  # nx=True: only set if not exists
        value = db.fetch(key)
        cache.set(key, value, ex=300)
        cache.delete(lock_key)
    else:
        time.sleep(0.05)
        value = cache.get(key)  # retry after lock holder populates
    return value
```

**Probabilistic Early Expiration (PER)**:
Before the TTL expires, probabilistically begin regenerating the value. The probability of early regeneration increases as the TTL approaches zero.

`if (current_time + delta) > expiry_time` — where `delta` is drawn from a distribution weighted by how long the fetch takes.

This spreads the regeneration across time, preventing the synchronous expiry. No locking required.

**Background Refresh**:
A background job periodically refreshes cache entries before they expire. The entry is never cold. Requires knowing which keys need proactive refresh.

**Serve Stale on Miss**:
When a cache miss occurs, serve the stale value while asynchronously triggering a refresh. The user gets slightly stale data but without waiting. ("Stale-while-revalidate" from HTTP Cache-Control.)

## Hot Key Problem

### What It Is
A single cache key that receives a disproportionate fraction of all requests. In a distributed cache like Redis Cluster, all requests for that key go to the same shard, creating a hot shard bottleneck.

**Example**: The profile of a celebrity with 100 million followers. Every user viewing any content from that celebrity requires reading their profile from the cache. A single Redis node receives millions of requests per second for one key.

### Solutions

**Local In-Process Cache (L1)**:
Cache the hot key in the application's own memory (JVM heap, Python dictionary). Requests never leave the application server. Each app server has its own copy. Invalidation is harder (need to broadcast invalidation across all app instances), and memory is per-server, but for extreme hot keys it's the most effective solution.

**Key Splitting (Replication)**:
Store N copies of the hot key: `hot_key:0`, `hot_key:1`, ..., `hot_key:N-1`. Reads choose a copy at random (or based on request ID). Writes must update all N copies. This spreads the load across N Redis shards at the cost of N writes instead of 1.

**Read Replicas**:
Use Redis read replicas (Redis Cluster replicas or Sentinel replicas) and route reads for hot keys to replicas.

## Cache Warming

### Why Cold Cache After Deploy Is Dangerous
When a new service is deployed or a cache is flushed, the cache is empty. All requests result in cache misses. The database receives the full production load, which it may not be sized to handle (it was sized assuming a cache hit rate of 95%).

This is called "cache stampede at deploy time" and can take down the database.

### Strategies

**Gradual Traffic Shift (Canary)**:
Deploy to a small percentage of traffic first. The cache warms up gradually before the full traffic shift.

**Pre-warm Before Deploy**:
Before deploying, run a warm-up job that reads the most popular keys from the database and populates the cache. This requires knowing which keys are hot (from logs or analytics).

**Copy Cache From Old Deployment**:
If using Redis with persistence, restore the old cache snapshot into the new deployment. This is particularly useful for planned maintenance.

**Accept Warm-Up Period**:
Design the system to tolerate a 5–10 minute warm-up period. Use circuit breakers to prevent the database from being overwhelmed during this window. Rate-limit cache misses.

## What NOT to Cache

Not everything benefits from caching. Some data should never be cached:

- **Highly personalized data**: Every user has unique data. Caching 1 million users' unique data provides no hit rate benefit but wastes memory.
- **Low-read, high-write data**: If a value changes every 30 seconds and is read once per minute, the cache provides no benefit (every read is a miss or immediately stale).
- **Data where staleness causes correctness bugs**: Real-time inventory during a flash sale — if you cache "5 items remaining" and 5 users buy simultaneously, you've sold 5 items 5 times.
- **Security-sensitive data**: Session tokens and auth decisions should not be aggressively cached if they can be revoked. Use short TTLs if you must cache them.
- **Large blobs**: Caching multi-MB objects consumes enormous cache memory for marginal benefit. Serve large blobs directly from object storage (S3) + CDN.

## Multi-Layer Caching (L1 / L2 / L3)

Real systems use multiple cache tiers:

| Layer | Technology | Latency | Scope | TTL |
|---|---|---|---|---|
| L1: Local | In-process memory (HashMap, Caffeine) | ~100ns | Single server | Seconds to minutes |
| L2: Distributed | Redis, Memcached | ~0.5ms | All servers | Minutes to hours |
| L3: CDN | CloudFront, Fastly | 1–50ms (edge PoP) | Global users | Minutes to days |

**L1 → L2 → L3 → Origin**: Each layer is faster but smaller and less consistent.

**Invalidation challenge**: A write to the origin must propagate invalidations to all three layers. CDN invalidation is asynchronous (takes 1–10 seconds). L1 caches in each app server need a broadcast mechanism (pub/sub, or just rely on TTL).

## CDN Caching: Cache-Control Headers, TTL, Invalidation

### Cache-Control Headers
The HTTP `Cache-Control` header instructs CDNs and browsers how to cache a response:

- `Cache-Control: public, max-age=86400` — cacheable by CDN and browser for 24 hours.
- `Cache-Control: private, max-age=0` — not cacheable by CDN (user-specific). Browser may cache.
- `Cache-Control: no-store` — don't cache anywhere.
- `Cache-Control: stale-while-revalidate=60` — serve stale for up to 60 seconds while revalidating in background.
- `Cache-Control: s-maxage=3600, max-age=0` — CDN caches for 1 hour; browsers don't cache. Useful for controlling CDN independently of browser.

### TTL Selection for CDN
- Static assets (images, JS, CSS): Use content-addressed filenames (`app.a1b2c3.js`). Set `max-age` to 1 year. When content changes, the filename changes — no cache invalidation needed.
- HTML pages: Short TTL (seconds to minutes) or `no-store` for dynamic content.
- API responses: Cache aggressively with a CDN for public, non-personalized data (product catalog, public profiles).

### CDN Cache Invalidation
CDN invalidation is not instant. CDN providers (CloudFront, Fastly) support "cache purge" APIs that invalidate specific URLs or patterns. Propagation takes 5–30 seconds.

Design strategies:
- **Immutable versioned URLs**: Never need invalidation. Best practice for static assets.
- **Surrogate-Key / Cache-Tag**: Tag responses with cache tags (e.g., `product:123`). Purge all CDN responses tagged with `product:123` when that product changes. Fastly and Varnish support this.
- **TTL + Stale-While-Revalidate**: Accept a short window of stale content, serve stale while refreshing. Avoids hard invalidation complexity.

## SRE Lens: What This Looks Like in Production

### Cache Stampede During Black Friday
At midnight on Black Friday, a scheduled job flushes the product catalog cache to load new sale prices. All 50,000 concurrent users hit cache misses simultaneously. The PostgreSQL primary receives 50,000 "SELECT * FROM products" queries at once. Query queue depth explodes. The database runs out of connections. The site goes down for 8 minutes.

**Post-mortem learnings**: Never flush the entire cache simultaneously. Use a "write-through with versioning" strategy — populate the new cache before switching. Apply jittered TTLs so entries don't expire simultaneously.

### Redis Memory Pressure Evicting Hot Keys
An SRE notices a degradation in API latency. Cache hit rate has dropped from 95% to 60%. Redis is running at 95% memory. The `allkeys-lru` policy is evicting keys, but it's evicting keys that are actually hot — recently accessed but not *most* recently accessed. The issue: the working set has grown beyond the cache size. Fix: increase Redis memory, use a smaller working set, or implement multi-tier caching with a local L1.

### Stale Cache After Multi-Datacenter Failover
A service fails over to a secondary datacenter. The secondary's Redis cache is cold (it was never receiving traffic). The database in the secondary datacenter, sized for 10% of load as a standby, receives 100% of traffic through the cold cache. Latency spikes. Fix: warm secondary caches proactively, or route reads to the primary datacenter cache during failover until the secondary warms up.

## Common Misconceptions

- **"Adding a cache always improves performance"**: A cache with a low hit rate adds latency (cache check + miss + DB fetch > DB fetch alone). Only cache data with high temporal locality.
- **"TTL expiry prevents stale data"**: TTL reduces the window of staleness, but during that window stale data is served. For correctness-critical operations, TTL-based invalidation is not enough — use explicit invalidation on write.
- **"Redis is durable"**: Default Redis is in-memory with no persistence. RDB snapshots are periodic (data loss between snapshots). AOF provides better durability but still has a flush interval risk window. Don't use Redis as your primary data store without understanding these tradeoffs.
- **"Cache invalidation is easy to get right"**: Phil Karlton: "There are only two hard things in Computer Science: cache invalidation and naming things." The race condition between writes and cache updates is a real source of bugs.
- **"More cache = better"**: Cache memory is finite. Caching the right data at the right TTL beats caching everything with long TTLs.

## Key Takeaways

- Cache-aside is the most common pattern: check cache, fall through to DB on miss, write DB first and then invalidate cache.
- Write-through keeps cache consistent at the cost of write latency. Write-behind is fast but risks durability.
- LRU is the default eviction policy. Use TTL for data with known freshness requirements. LFU for stable hot-set workloads.
- Cache stampede is caused by synchronous expiry of a hot key. Mitigate with probabilistic early expiration, mutex locks, or background refresh.
- Hot keys in distributed caches require local in-process caching (L1) or key sharding.
- Pre-warm caches before deploying. Cold caches at deploy time can take down the database.
- Multi-layer caching (in-process → Redis → CDN) is the production pattern for high-traffic systems.
- CDN cache-busting with content-addressed URLs (hash in filename) is the best practice for static assets.

## References

- Nishtala, R. et al. (2013). "Scaling Memcache at Facebook." NSDI. — the canonical large-scale caching paper
- Kleppmann, M. — *Designing Data-Intensive Applications*, Chapter 12
- Redis documentation — https://redis.io/docs/manual/eviction/
- HTTP Caching — MDN Web Docs — https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching
- Varnish Cache documentation on cache invalidation patterns
- "Cache stampede" — Wikipedia entry has good pseudocode for probabilistic early expiration

# System Design: Platform Architecture Learning Journal

> Goal: Become a Platform Architect by deeply understanding how the world's most
> scaled systems are designed, built, and operated.

## Background & Context

Coming from a Sr Staff SRE background means you already have:
- Deep ops knowledge: reliability, SLOs, incident response, infra
- Understanding of how systems *fail* and how to *keep them running*

The gap to Platform Architect is:
- **Why** systems are designed the way they are (not just how they run)
- Design decisions and trade-offs made *before* things go to production
- Data modeling, API contracts, service boundaries
- Business context that drives architectural choices
- End-to-end system reasoning: from user action to data at rest

This repo documents that learning. One company at a time, studied deeply.

---

## Learning Path

The path is organized in phases. Each phase builds on the last. Do not skip phases.

```
Phase 0: Foundations         — The vocabulary and concepts every architect uses
Phase 1: Building Blocks     — Small systems that appear inside every large one
Phase 2: Communication       — Messaging, feeds, real-time at scale
Phase 3: Marketplace/Matching — Geo, supply-demand, pricing
Phase 4: Media & Streaming   — Upload pipelines, CDN, recommendations
Phase 5: Community & Social  — Feed ranking, voting, moderation
Phase 6: Infrastructure      — Platforms that platforms are built on
Phase 7: Search & AI         — Indexing, ranking, inference at scale
Phase 8: Finance & Compliance — Correctness over availability trade-offs
```

---

## Phase 0: Foundations

> Do not skip this. Your SRE background gives you operational knowledge.
> Foundations give you the design vocabulary.

| Topic | File | Key Concepts |
|-------|------|-------------|
| CAP Theorem & PACELC | [fundamentals/cap-theorem.md](fundamentals/cap-theorem.md) | Consistency, Availability, Partition tolerance |
| Consistency Models | [fundamentals/consistency-models.md](fundamentals/consistency-models.md) | Strong, eventual, causal, linearizable |
| Database Types | [fundamentals/databases.md](fundamentals/databases.md) | SQL, NoSQL, NewSQL, time-series, graph, vector |
| Caching Strategies | [fundamentals/caching.md](fundamentals/caching.md) | Write-through, write-behind, read-through, cache-aside, TTL |
| Load Balancing | [fundamentals/load-balancing.md](fundamentals/load-balancing.md) | L4/L7, algorithms, health checks, sticky sessions |
| Message Queues & Streams | [fundamentals/message-queues.md](fundamentals/message-queues.md) | Queues vs logs, at-least-once, exactly-once, backpressure |
| Networking | [fundamentals/networking.md](fundamentals/networking.md) | TCP/UDP, HTTP2/3, WebSockets, gRPC, long polling |
| Storage Patterns | [fundamentals/storage-patterns.md](fundamentals/storage-patterns.md) | Sharding, replication, compaction, LSM trees, B-trees |
| API Design | [fundamentals/api-design.md](fundamentals/api-design.md) | REST, GraphQL, gRPC, pagination, versioning, idempotency |
| Distributed Systems Concepts | [fundamentals/distributed-systems.md](fundamentals/distributed-systems.md) | Clock drift, leader election, consensus, two-phase commit |
| Scalability Patterns | [fundamentals/scalability-patterns.md](fundamentals/scalability-patterns.md) | Horizontal vs vertical, CQRS, event sourcing, saga pattern |
| Observability (SRE lens) | [fundamentals/observability.md](fundamentals/observability.md) | Metrics, traces, logs, SLOs — now from a design perspective |

---

## Phase 1: Building Blocks

> These small systems are embedded in almost every company studied later.
> Learn them here so you recognize them when they appear inside larger systems.

| System | Folder | Core Problems |
|--------|--------|--------------|
| URL Shortener | [building-blocks/url-shortener/](building-blocks/url-shortener/) | Hashing, redirection, analytics, expiry |
| Rate Limiter | [building-blocks/rate-limiter/](building-blocks/rate-limiter/) | Token bucket, sliding window log, distributed state |
| Consistent Hashing | [building-blocks/consistent-hashing/](building-blocks/consistent-hashing/) | Virtual nodes, rebalancing, hotspots |
| Distributed Cache | [building-blocks/distributed-cache/](building-blocks/distributed-cache/) | Eviction, cache stampede, thundering herd, warming |
| Web Crawler | [building-blocks/web-crawler/](building-blocks/web-crawler/) | BFS/DFS, politeness, deduplication, distributed coordination |
| Notification System | [building-blocks/notification-system/](building-blocks/notification-system/) | Fan-out, delivery guarantees, multi-channel, opt-out |
| Search Autocomplete | [building-blocks/search-autocomplete/](building-blocks/search-autocomplete/) | Trie, top-k, prefix indexing, personalization |
| Distributed ID Generator | [building-blocks/id-generator/](building-blocks/id-generator/) | Snowflake IDs, UUID, sortability, monotonicity |

---

## Phase 2: Communication at Scale

| Company | Folder | Scale | Key Problems |
|---------|--------|-------|-------------|
| WhatsApp | [companies/whatsapp/](companies/whatsapp/) | 2B users, 100B msgs/day | Message delivery guarantees, E2EE, offline queues, group messaging |
| Slack | [companies/slack/](companies/slack/) | 20M DAU, 1B msgs/day | Channels, workspaces, search, real-time delivery |
| Twitter / X | [companies/twitter/](companies/twitter/) | 500M tweets/day, 350M users | Fanout, timeline assembly, @ mentions, trending |
| Bluesky | [companies/bluesky/](companies/bluesky/) | Decentralized, AT Protocol | Federation, DID, PDS, relay, AppView — how decentralization actually works |
| Discord | [companies/discord/](companies/discord/) | 19M DAU, 4B msg/day | Guilds, voice/video, real-time presence, massive read fanout |

---

## Phase 3: Marketplace & Matching

| Company | Folder | Scale | Key Problems |
|---------|--------|-------|-------------|
| Uber | [companies/uber/](companies/uber/) | 19M trips/day | Geospatial matching, dynamic pricing, maps, driver supply |
| Lyft | [companies/lyft/](companies/lyft/) | 2M rides/day | Similar to Uber — study differences in architecture choices |
| Tinder | [companies/tinder/](companies/tinder/) | 1.6B swipes/day | Proximity matching, Elo/Glicko ranking, deck assembly |
| Airbnb | [companies/airbnb/](companies/airbnb/) | 6M listings | Search, availability calendar, pricing, trust & fraud |
| DoorDash | [companies/doordash/](companies/doordash/) | 37M orders/month | Three-sided marketplace, real-time dispatch, ETA |

---

## Phase 4: Media & Streaming

| Company | Folder | Scale | Key Problems |
|---------|--------|-------|-------------|
| YouTube | [companies/youtube/](companies/youtube/) | 500 hrs uploaded/min, 1B hrs watched/day | Upload pipeline, transcoding, CDN, recommendations |
| Netflix | [companies/netflix/](companies/netflix/) | 270M subscribers, 15% global internet traffic | Streaming, encoding (per-title), CDN (Open Connect), chaos engineering |
| Spotify | [companies/spotify/](companies/spotify/) | 600M users, 100M tracks | Audio delivery, playlists, offline sync, Discover Weekly |

---

## Phase 5: Community & Social

| Company | Folder | Scale | Key Problems |
|---------|--------|-------|-------------|
| Reddit | [companies/reddit/](companies/reddit/) | 57M DAU, 130K communities | Voting, karma, feed ranking, moderation, NSFW content |
| Instagram | [companies/instagram/](companies/instagram/) | 2B users, 100M photos/day | Media processing, stories, reels, explore, follow graph |

---

## Phase 6: Infrastructure Platforms

| System | Folder | Scale | Key Problems |
|--------|--------|-------|-------------|
| Apache Kafka | [companies/kafka/](companies/kafka/) | Trillions of msgs/day at LinkedIn | Distributed log, partitioning, consumer groups, compaction, exactly-once |
| Google Bigtable | [companies/bigtable/](companies/bigtable/) | Exabytes at Google | Wide-column store, tablet serving, compaction, Chubby |
| Amazon S3 | [companies/amazon-s3/](companies/amazon-s3/) | 100T+ objects | Object storage, durability, consistency (strong since 2020) |
| Cloudflare CDN | [companies/cloudflare/](companies/cloudflare/) | 20% of web traffic | Anycast, edge caching, DDoS mitigation, Workers |

---

## Phase 7: Search & AI

| System | Folder | Scale | Key Problems |
|--------|--------|-------|-------------|
| Google Search | [companies/google-search/](companies/google-search/) | 8.5B searches/day | Crawling, indexing, PageRank, serving, freshness |
| ChatGPT / OpenAI | [companies/chatgpt/](companies/chatgpt/) | 100M users | Inference serving, GPU scheduling, context windows, streaming |
| Elasticsearch | [companies/elasticsearch/](companies/elasticsearch/) | Used by most companies | Inverted index, sharding, replication, relevance tuning |

---

## Phase 8: Finance & High Correctness

| System | Folder | Scale | Key Problems |
|--------|--------|-------|-------------|
| Stock Exchange | [companies/stock-exchange/](companies/stock-exchange/) | NASDAQ: 6B trades/day | Order book, matching engine, microsecond latency, correctness |
| Stripe | [companies/stripe/](companies/stripe/) | 250M API calls/day | Payment rails, idempotency, ledger, fraud |

---

## Per-Company Study Template

Every company folder follows the same structure. See [TEMPLATE.md](TEMPLATE.md).

```
companies/<name>/
├── README.md              — overview, scale, business context
├── requirements.md        — functional + non-functional requirements
├── architecture.md        — high-level design + component diagram
├── data-model.md          — schemas, storage choices, why
├── deep-dives/            — one file per subsystem (e.g., feed.md, search.md)
├── trade-offs.md          — key design decisions and alternatives considered
├── failure-modes.md       — what breaks, how, post-mortems where public
└── references.md          — eng blogs, papers, talks, source material
```

---

## SRE → Platform Architect: The Mental Shift

As a Sr Staff SRE you already think about systems from the outside in (how do I keep this running?). Platform Architect thinking is inside out:

| SRE Mindset | Architect Mindset |
|-------------|------------------|
| How do I detect when this breaks? | Why would this break by design? |
| What is the SLO? | What consistency model makes this SLO achievable? |
| How do I scale this service? | Should this be one service or three? |
| What's the runbook? | What should never need a runbook? |
| How do I monitor the database? | Which database fits this access pattern? |
| What's causing the latency spike? | What latency budget does each layer have? |
| How do I add capacity? | Where are the bottlenecks by design? |

Both modes of thinking are needed. Your SRE lens makes you a better architect because you've seen what actually breaks in production.

---

## Recommended Supplementary Reading

**Books**
- *Designing Data-Intensive Applications* — Martin Kleppmann (read this first)
- *System Design Interview Vol 1 & 2* — Alex Xu (breadth coverage)
- *Building Microservices* — Sam Newman
- *Software Architecture: The Hard Parts* — Ford, Richards, Sadalage, Dehghani
- *The Staff Engineer's Path* — Tanya Reilly (career context)

**Engineering Blogs to Follow**
- [eng.uber.com](https://eng.uber.com)
- [netflixtechblog.com](https://netflixtechblog.com)
- [engineering.atspotify.com](https://engineering.atspotify.com)
- [discord.com/blog/engineering](https://discord.com/blog/engineering)
- [slack.engineering](https://slack.engineering)
- [instagram-engineering.com](https://instagram-engineering.com)
- [blog.twitter.com/engineering](https://blog.twitter.com/engineering)
- [engineering.fb.com](https://engineering.fb.com)
- [highscalability.com](http://highscalability.com) (aggregates all of the above)

**Papers**
- Google MapReduce (2004)
- Google Bigtable (2006)
- Amazon Dynamo (2007)
- Facebook Cassandra (2010)
- LinkedIn Kafka (2011)
- Google Spanner (2012)
- Facebook TAO (2013)

---

## Progress Tracker

| Phase | System | Status | Notes |
|-------|--------|--------|-------|
| 0 | CAP Theorem | - | |
| 0 | Consistency Models | - | |
| 0 | Databases | - | |
| 0 | Caching | - | |
| 0 | Load Balancing | - | |
| 0 | Message Queues | - | |
| 0 | Networking | - | |
| 0 | Storage Patterns | - | |
| 0 | API Design | - | |
| 0 | Distributed Systems | - | |
| 0 | Scalability Patterns | - | |
| 1 | URL Shortener | - | |
| 1 | Rate Limiter | - | |
| 1 | Consistent Hashing | - | |
| 1 | Distributed Cache | - | |
| 1 | Web Crawler | - | |
| 1 | Notification System | - | |
| 1 | Search Autocomplete | - | |
| 1 | ID Generator | - | |
| 2 | WhatsApp | - | |
| 2 | Slack | - | |
| 2 | Twitter | - | |
| 2 | Bluesky | - | |
| 2 | Discord | - | |
| 3 | Uber | - | |
| 3 | Lyft | - | |
| 3 | Tinder | - | |
| 3 | Airbnb | - | |
| 3 | DoorDash | - | |
| 4 | YouTube | - | |
| 4 | Netflix | - | |
| 4 | Spotify | - | |
| 5 | Reddit | - | |
| 5 | Instagram | - | |
| 6 | Kafka | - | |
| 6 | Bigtable | - | |
| 6 | Amazon S3 | - | |
| 6 | Cloudflare | - | |
| 7 | Google Search | - | |
| 7 | ChatGPT | - | |
| 7 | Elasticsearch | - | |
| 8 | Stock Exchange | - | |
| 8 | Stripe | - | |

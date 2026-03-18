# Building Blocks

These are the small, reusable subsystems that appear inside almost every
large system you study. Learn them once, recognize them everywhere.

## Study Order

| # | System | Why First |
|---|--------|-----------|
| 1 | [id-generator/](id-generator/) | IDs appear everywhere. Understand Snowflake before studying Kafka, Twitter, etc. |
| 2 | [consistent-hashing/](consistent-hashing/) | Used in every distributed cache, DB shard ring. Foundation for scaling. |
| 3 | [url-shortener/](url-shortener/) | Simple end-to-end design. Good for practicing the full template. |
| 4 | [rate-limiter/](rate-limiter/) | Used at the API gateway of every company studied. |
| 5 | [distributed-cache/](distributed-cache/) | Used everywhere. Need to understand before studying feeds, search. |
| 6 | [web-crawler/](web-crawler/) | Foundation for Google Search, Elasticsearch indexing. |
| 7 | [notification-system/](notification-system/) | Cross-cutting: used by Uber, WhatsApp, Twitter, Airbnb. |
| 8 | [search-autocomplete/](search-autocomplete/) | Simplified search — good intro before Elasticsearch deep dive. |

## What These Are Not

These are not toy problems. When you study the URL Shortener, the goal is
not to "solve" an interview question. It is to:

- Understand the full storage, caching, and routing design
- Understand what breaks at 100M shortlinks vs 100B shortlinks
- Understand why Bitly, YouTube, and Twitter each made different choices

Apply the full [TEMPLATE.md](../TEMPLATE.md) to each building block.

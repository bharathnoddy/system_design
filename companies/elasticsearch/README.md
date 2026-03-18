# Elasticsearch

> Distributed search — Lucene at scale, used inside most other companies studied

## What to Study

- Inverted index internals — how Lucene builds and merges index segments on disk
- Sharding and routing — primary shards, replica shards, routing by document ID
- Relevance scoring — BM25 scoring function, field boosting, query-time vs index-time relevance
- Aggregations — bucket and metric aggregations, cardinality estimation (HyperLogLog)
- Cluster state management — master election, shard allocation, split-brain prevention

## Why This Matters for Platform Architecture

Elasticsearch is the search layer inside Slack, GitHub, Netflix, Uber, and nearly every other system in this repo. Understanding how it distributes Lucene across a cluster — and the trade-offs that introduces for consistency, reindex cost, and query latency — explains design decisions visible in all those systems.

## Study Checklist

Use [TEMPLATE.md](../../TEMPLATE.md) as your study guide.

- [ ] Business context & scale documented
- [ ] Requirements defined
- [ ] Architecture diagram created
- [ ] Data model analyzed
- [ ] Key trade-offs articulated
- [ ] At least 2 primary sources read
- [ ] Failure modes analyzed
- [ ] Can compare to a previously studied system

## Status

Not started

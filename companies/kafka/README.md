# Kafka

> Distributed log — the infrastructure platform other platforms are built on

## What to Study

- Log segment storage — how Kafka persists messages as an ordered, immutable log on disk
- Partitioning — the unit of parallelism, ordering guarantees, and partition assignment
- Consumer groups — cooperative vs eager rebalancing, offset management, at-least-once semantics
- Log compaction — retaining the latest value per key for changelog use cases
- Exactly-once semantics — idempotent producers, transactional API, and the cost of EOS

## Why This Matters for Platform Architecture

Kafka is the connective tissue in most large-scale platforms studied in this repo — DoorDash, LinkedIn, Uber, and others use it as their event backbone. Understanding its internals explains why so many architectural patterns (event sourcing, CQRS, change data capture) are built on top of it.

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

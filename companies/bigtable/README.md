# Bigtable

> Google's wide-column store — the paper that shaped NoSQL

## What to Study

- Tablet architecture — how the key space is range-partitioned into tablets served by tablet servers
- SSTable (Sorted String Table) — the on-disk format and how minor/major compactions work
- Chubby distributed lock service — how it provides metadata coordination and master election
- Compaction strategies — minor, merging, and major compaction and their I/O trade-offs
- Row key design — how key design determines read/write locality and hotspot risk

## Why This Matters for Platform Architecture

The 2006 Bigtable paper directly influenced HBase, Cassandra, and the broader NoSQL movement. Its tablet/SSTable model is the foundation for understanding how wide-column stores achieve horizontal scalability and how row key design determines real-world performance.

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

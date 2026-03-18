# Amazon S3

> Object storage — durability model, strong consistency, and the design behind 11 nines

## What to Study

- Erasure coding — how S3 achieves 11 nines durability without full replication across AZs
- Strong consistency evolution — how S3 moved from eventual to strong read-after-write consistency in 2020
- Multipart upload — chunked upload protocol for large objects and its failure recovery model
- Versioning — object version chains, delete markers, and lifecycle policy interactions

## Why This Matters for Platform Architecture

S3's durability model is the reference point for every "how do you store it forever" question in system design. The 2020 consistency upgrade is a case study in retrofitting stronger guarantees into an eventually consistent system at planetary scale without a flag day.

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

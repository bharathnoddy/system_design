# Stock Exchange

> Financial exchange — correctness over availability, microsecond matching

## What to Study

- Order book data structure — price level aggregation, best bid/ask, order priority queues
- Price-time priority matching — FIFO within a price level, pro-rata as an alternative
- FIFO queue guarantees — how exchanges prevent reordering and ensure fair sequencing
- Circuit breakers — halt conditions, market-wide breakers vs individual security halts
- Market data distribution — consolidated tape, SIP feeds, direct feeds and latency arbitrage

## Why This Matters for Platform Architecture

A stock exchange is the highest-stakes example of a system where correctness is non-negotiable and availability is secondary. Its order matching engine demonstrates that for some systems, strong consistency and deterministic ordering are worth significant architectural cost.

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

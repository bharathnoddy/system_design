# Uber

> Ride matching at 19M trips/day — geospatial, surge pricing, supply-demand

## What to Study

- Geohash and H3 hexagonal indexing for proximity queries and supply/demand mapping
- Dispatch system — matching riders to drivers under latency constraints
- HSOT (Heterogeneous Supply/demand Over Time) model for forecasting
- Dynamic surge pricing — real-time supply/demand imbalance signals and pricing feedback loops

## Why This Matters for Platform Architecture

Uber's core matching and geospatial systems are the reference architecture for marketplace dispatch. The tension between matching quality, latency, and geographic indexing precision applies to any system that must connect supply and demand in physical space.

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

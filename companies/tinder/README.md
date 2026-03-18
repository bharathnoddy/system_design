# Tinder

> Dating at 1.6B swipes/day — proximity matching, Elo ranking, deck assembly

## What to Study

- Geospatial queries for proximity-based candidate retrieval
- Recommendation stack — Elo-style ranking, collaborative filtering, real-time score updates
- Swipe mechanics — high write throughput, eventual consistency trade-offs
- Match notifications — low-latency delivery when mutual likes occur

## Why This Matters for Platform Architecture

Tinder's deck assembly problem (generating a ranked, personalized, location-filtered card stack in milliseconds) combines geospatial indexing, ML ranking, and high write throughput in a way that generalizes to any recommendation-driven, location-aware product.

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

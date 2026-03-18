# Cloudflare

> CDN + edge platform — 20% of web traffic, DDoS mitigation, Workers

## What to Study

- Anycast routing — how Cloudflare routes traffic to the nearest PoP using BGP anycast
- Edge caching — cache topology, cache key design, purge propagation across 300+ PoPs
- KV at the edge — Workers KV consistency model and the trade-offs of eventual consistency globally
- Workers serverless model — V8 isolates vs containers, cold start elimination, execution model
- DDoS mitigation design — volumetric vs application-layer attacks, mitigation at the edge without impacting legitimate traffic

## Why This Matters for Platform Architecture

Cloudflare's architecture demonstrates how pushing compute and data to the network edge changes the design space for latency, resilience, and cost. Its Workers platform is the reference implementation for isolate-based edge compute, and its DDoS mitigation is the most publicly documented at its scale.

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

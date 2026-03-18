# DoorDash

> Food delivery — three-sided marketplace, real-time dispatch, ETA

## What to Study

- Consumer/Dasher/Merchant matching — coordinating three independent agents in real time
- Routing and assignment optimization under delivery time constraints
- ETA prediction pipeline — restaurant prep time, Dasher travel, traffic signals
- Kafka usage for event streaming between marketplace components

## Why This Matters for Platform Architecture

DoorDash's three-sided marketplace is architecturally more complex than a two-sided one — every order requires simultaneously coordinating consumer expectations, merchant readiness, and Dasher availability. Its heavy use of Kafka as a connective tissue is a reference model for event-driven marketplace design.

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

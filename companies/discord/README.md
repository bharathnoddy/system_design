# Discord

> Gaming communication at 19M DAU — guilds, voice, massive read fanout

## What to Study

- WebRTC for voice/video — TURN/STUN, SFU vs MCU architecture
- Cassandra for message storage — time-series data model and the 2022 migration away from it
- Guild (server) architecture and how channel read state is tracked at scale
- Presence at scale — who is online, in what game, with what status

## Why This Matters for Platform Architecture

Discord's architecture illustrates the cost of massive read fanout in community platforms with large servers, and the real engineering pain of time-series data at scale. Its public postmortems on Cassandra and read state are required reading for anyone designing high-fan-out systems.

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

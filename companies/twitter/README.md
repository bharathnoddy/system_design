# Twitter

> Social feed at 500M tweets/day — the canonical fanout problem

## What to Study

- Push vs pull fanout trade-offs (fan-out-on-write vs fan-out-on-read) and the hybrid approach for celebrities
- Timeline storage — Redis for home timelines, Manhattan for persistent storage
- Trending topics — real-time counting, locality, and decay functions
- @ mentions, retweet chains, and notification fanout

## Why This Matters for Platform Architecture

Twitter's timeline system is the most studied example of the fanout problem in system design. The tension between write amplification (push) and read amplification (pull) appears in every social feed, and Twitter's hybrid solution defines the pattern.

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

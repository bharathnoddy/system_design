# Reddit

> Community platform — voting, karma, feed ranking, 130K subreddits

## What to Study

- Vote counting at scale — approximate counts, anti-fraud vote fuzzing, score decay
- Feed algorithm — Hot/Top/New/Rising ranking functions and their trade-offs
- Community (subreddit) isolation — moderation tooling, content policies per community
- Search — full-text search across posts and comments at petabyte scale

## Why This Matters for Platform Architecture

Reddit's voting and ranking system is one of the cleanest examples of approximate counting under adversarial conditions. Its community-as-namespace model is a pattern for any platform that hosts user-governed sub-communities with independent moderation.

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

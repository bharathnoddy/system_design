# Instagram

> Photo/video at 2B users — media processing, stories, explore

## What to Study

- Media upload and processing pipeline — image resizing, video transcoding, CDN distribution
- Follow graph storage and traversal — how the social graph is stored and queried
- Feed assembly — ranked feed generation from a follow graph of billions of edges
- Explore ranking — content discovery for users outside their follow graph

## Why This Matters for Platform Architecture

Instagram's evolution from a single Postgres instance to a globally distributed system is one of the most documented scaling stories in the industry. Its feed assembly and Explore systems illustrate the contrast between personalized ranked feeds and cold-start discovery.

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

# YouTube

> Video at scale — 500hrs uploaded/min, 1B hrs watched/day

## What to Study

- Upload pipeline — chunked upload, virus scanning, copyright fingerprinting (Content ID)
- Transcoding at scale — VP9/AV1 codecs, per-resolution ladder, distributed job scheduling
- CDN architecture — Google's global edge network and cache fill strategies
- Recommendations — the two-tower neural retrieval + ranking system
- Comment system at scale — ordering, spam filtering, notification fanout

## Why This Matters for Platform Architecture

YouTube's upload-to-playback pipeline is the canonical reference for media processing infrastructure. The transcoding bottleneck and CDN design decisions it pioneered are reproduced in every video platform, and its recommendation system is the most influential in the industry.

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

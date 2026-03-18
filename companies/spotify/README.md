# Spotify

> Music streaming — 600M users, offline sync, Discover Weekly

## What to Study

- Audio CDN — how tracks are stored, cached, and delivered with low latency
- Offline storage — download management, DRM, sync state across devices
- Playlist management — collaborative playlists, ordering, conflict resolution
- Recommendation ML pipeline — Discover Weekly, collaborative filtering + NLP on playlist context

## Why This Matters for Platform Architecture

Spotify's offline sync model is a reference for any system that must maintain consistency between a local cache and a remote source of truth across unreliable connections. Its recommendation pipeline, particularly the NLP-based audio analysis + collaborative filtering hybrid, is widely studied.

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

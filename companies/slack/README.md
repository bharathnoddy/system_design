# Slack

> Real-time team messaging — channels, search, workspaces, presence

## What to Study

- WebSockets for real-time delivery and connection management at scale
- Message threading model and how it affects storage layout
- Full-text search indexing across billions of messages (Elasticsearch under the hood)
- Workspaces as isolated tenants — multi-tenancy architecture and data isolation

## Why This Matters for Platform Architecture

Slack illustrates how to layer real-time pub/sub, persistent storage, and full-text search into a single coherent product. Its workspace-as-tenant model is a reference design for SaaS multi-tenancy isolation.

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

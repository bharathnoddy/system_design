# Architecture Patterns

These patterns appear repeatedly across the companies you study.
Document them here as you encounter them. Cross-reference to companies where you've seen them in practice.

## Patterns to Document

### Data Patterns
- [ ] CQRS (Command Query Responsibility Segregation)
- [ ] Event Sourcing
- [ ] Saga Pattern (distributed transactions)
- [ ] Outbox Pattern (reliable event publishing)
- [ ] Fan-out on Write vs Fan-out on Read
- [ ] Read Replicas + Eventual Consistency
- [ ] Denormalization for Read Performance

### Scalability Patterns
- [ ] Consistent Hashing
- [ ] Database Sharding Strategies
- [ ] Cell-Based Architecture (Uber, Slack)
- [ ] Bulkhead Pattern
- [ ] Circuit Breaker

### Communication Patterns
- [ ] Pub/Sub vs Message Queue
- [ ] Request/Reply vs Event-Driven
- [ ] Long Polling vs WebSocket vs SSE
- [ ] gRPC vs REST vs GraphQL — when to use which

### Reliability Patterns
- [ ] Idempotency Keys
- [ ] Retry with Exponential Backoff + Jitter
- [ ] Dead Letter Queue
- [ ] Two-Phase Commit vs Saga
- [ ] Distributed Locking

### Caching Patterns
- [ ] Cache-Aside
- [ ] Write-Through
- [ ] Write-Behind (Write-Back)
- [ ] Read-Through
- [ ] Thundering Herd / Cache Stampede Prevention

## How to Document a Pattern

For each pattern create a file `patterns/<pattern-name>.md` with:
1. What problem it solves
2. How it works
3. Trade-offs
4. When to use vs when NOT to use
5. Real companies where you observed it during study

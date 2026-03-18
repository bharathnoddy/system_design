# [Company Name] — System Design

> **One-line summary:** What this system does, at what scale, for whom.

---

## 1. Business Context & Scale

Answer these before touching any architecture:

- What problem does this solve for users?
- What is the scale? (users, requests/sec, data volume, geographical spread)
- What are the business model constraints that affect architecture?
  - e.g., Ads business = read-heavy feed. Payments = strong consistency required.
- When was it built? What were the constraints at that time?

**Key Metrics to Know:**
| Metric | Value |
|--------|-------|
| Daily Active Users | |
| Requests/sec (peak) | |
| Data written/day | |
| Storage footprint | |
| Geographical reach | |

---

## 2. Requirements

### Functional Requirements
What the system must *do* (features, from user perspective).

- [ ] Requirement 1
- [ ] Requirement 2

### Non-Functional Requirements
What the system must *be* (quality attributes).

| Attribute | Target | Rationale |
|-----------|--------|-----------|
| Availability | 99.99% | |
| Latency (p99) | <100ms | |
| Consistency | Eventual / Strong | |
| Durability | 99.999999999% | |
| Read/Write ratio | X:1 | |

### Out of Scope
What this design explicitly does *not* cover.

---

## 3. High-Level Architecture

> Draw this as a diagram. Text descriptions of architecture are insufficient.
> Use ASCII or link to a Mermaid/draw.io file in deep-dives/.

**Components:**
- Component A: responsibility
- Component B: responsibility

**Data flow (happy path):**
1. User does X
2. Request hits Y
3. Y calls Z
4. Response returns

---

## 4. Core Design Decisions

This is the most important section. For each major decision:

### Decision: [e.g., "Push vs Pull for feed delivery"]

**Context:** What made this a decision worth making?

**Options Considered:**
| Option | Pros | Cons |
|--------|------|------|
| Option A | | |
| Option B | | |

**Choice:** Option X

**Why:** The reasoning that made this the right trade-off *for this specific system*.

**Consequence:** What this choice makes harder or requires elsewhere.

---

## 5. Data Model

For each major entity:

```
Entity: [Name]
Purpose: [Why does this exist]

Fields:
  id           — type, constraints, generation strategy
  field_1      — type, indexed? why?
  field_2      — type, denormalized? why?
  created_at   — type

Storage: [Which database and why]
Access Patterns:
  - Read by: [key/index]
  - Write pattern: [single row / batch / stream]
  - Scale: [rows/sec, total volume]
```

**Storage Choices:**
| Data | Store | Why |
|------|-------|-----|
| User profiles | PostgreSQL | Relational, low write rate, ACID needed |
| Feed events | Cassandra | Time-series, high write rate, append-only |
| Session state | Redis | In-memory, TTL, fast lookup |
| Media files | S3 | Blob storage, durability, CDN-friendly |

---

## 6. Scalability Deep Dive

### Bottleneck Analysis

Where does this system break first as load increases?

| Component | Bottleneck Type | Mitigation |
|-----------|----------------|------------|
| | CPU / Memory / Network / IO | |

### Sharding Strategy
- What is the shard key?
- What are the hotspot risks?
- How is rebalancing handled?

### Caching Strategy
- What is cached?
- Cache invalidation strategy?
- What is the cache hit rate target?

### Read Scaling
- Read replicas?
- CDN for static assets?
- Denormalization to avoid joins?

### Write Scaling
- Async writes / event-driven?
- Write batching?
- CQRS?

---

## 7. Reliability & Failure Modes

> This is where your SRE background gives you an edge.
> Think about this from design time, not after deployment.

### Single Points of Failure
| Component | Failure Mode | Impact | Mitigation |
|-----------|-------------|--------|------------|

### Known Failure Scenarios
| Scenario | What Breaks | How System Degrades | Recovery |
|----------|-------------|---------------------|---------|

### Cascading Failure Risks
- What happens when [dependency X] is down?
- Are there retry storms or thundering herds?
- What are the circuit breaker boundaries?

### Data Consistency Risks
- What operations require cross-service consistency?
- Where is eventual consistency acceptable?
- What compensating transactions exist?

---

## 8. Key Trade-offs

A summary of the core trade-offs this architecture makes. Each is a deliberate choice.

| Trade-off | What Was Chosen | What Was Sacrificed |
|-----------|----------------|---------------------|
| Availability vs Consistency | | |
| Latency vs Durability | | |
| Storage vs Compute | | |
| Simplicity vs Flexibility | | |

---

## 9. What Would You Do Differently Today?

Based on what's publicly known from eng blogs and post-mortems:

- What early decisions became technical debt?
- What did they re-architect and why?
- What do you see as a remaining weakness?

This is where you practice architect-level synthesis: reading between the lines of what companies publish.

---

## 10. References

**Official Engineering Content:**
- [ ] Link to official eng blog posts
- [ ] Conference talks (QCon, Velocity, InfoQ, StrangeLoop)
- [ ] Published papers

**Third-party Analysis:**
- [ ] High Scalability write-ups
- [ ] YouTube breakdowns (Gaurav Sen, ByteByteGo, etc.)

**Post-mortems:**
- [ ] Any public incident reports

---

## Checklist: Before Marking This Complete

- [ ] Can you explain the full request lifecycle from user action to data storage?
- [ ] Can you explain every major storage choice and why that specific DB was chosen?
- [ ] Can you describe what breaks first at 10x current load?
- [ ] Can you articulate the 3 most important trade-offs this architecture makes?
- [ ] Have you read at least 2 primary sources (eng blog, talk, or paper)?
- [ ] Can you compare this system to a similar one studied earlier?

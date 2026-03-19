# Observability

> Observability is the degree to which you can understand the internal state of a system from its external outputs — and designing for observability from the start is fundamentally different from bolting it on after the system is in production.

## Core Concept

Observability is not just monitoring. Monitoring tells you that something is wrong. Observability allows you to understand *why* it's wrong without deploying new code or adding new instrumentation.

The traditional notion of monitoring was: define known failure modes, alert when metrics cross thresholds. This works for known unknowns — failures you anticipated. Observability addresses unknown unknowns — the novel failure modes you didn't anticipate at design time.

A system is observable if you can ask arbitrary questions about its behavior and get answers from existing telemetry data. The signal that your system is not observable: when something goes wrong and the first reaction is "we need to add more logging."

## The Three Pillars: Metrics, Traces, Logs

### Why All Three Are Needed
These are complementary, not interchangeable.

- **Metrics** tell you *what* is happening at a statistical level: request rate, error rate, latency percentiles, CPU utilization. Fast to query, cheap to store, but aggregated — you lose individual event detail.
- **Traces** tell you *where* time is spent for a specific request as it flows through multiple services. Show you which service call in a distributed chain is slow.
- **Logs** tell you *what happened* in detail for specific events. The narrative of individual requests or system events. Expensive to store and query at scale, but rich.

**Example**: An alert fires because p99 latency is 3000ms.
- **Metrics** show the spike started 10 minutes ago, and it's isolated to the `/checkout` endpoint.
- **Traces** show that the checkout request is spending 2800ms waiting for the payments service.
- **Logs** in the payments service show `ERROR: database connection pool exhausted` at the exact time the spike started.

Without all three, you might identify *what* is slow (metrics) but not *which service* (traces) and not *why* (logs). Each pillar answers a different question.

### The Fourth Pillar: Events
Some practitioners add structured events as a fourth pillar: discrete records of things that happened (a deploy, a config change, a feature flag flip). Useful for correlating system changes with performance anomalies.

## SLI, SLO, SLA

### SLI (Service Level Indicator)
A quantitative measurement of a specific aspect of service behavior. The metric you measure.

**Good SLIs are**:
- Meaningful to users (latency they experience, error rate they see).
- Measurable precisely (from server logs or client-side, not estimated).
- Representative (measures the service the user cares about, not a proxy).

**Examples of SLIs**:
- The fraction of requests that complete in less than 200ms (latency SLI).
- The fraction of requests that succeed (availability SLI: `(total - errors) / total`).
- The fraction of records processed within 5 minutes of ingestion (freshness SLI).
- The read throughput of the storage system (throughput SLI).

**Common mistake**: Measuring internal metrics (CPU, memory, queue depth) as SLIs. These are implementation details. Your users don't care about CPU. They care about whether their request succeeds and how fast.

### SLO (Service Level Objective)
A target for your SLI. The commitment you make internally about service behavior.

**Format**: "Over a 30-day rolling window, 99.9% of requests will succeed and 95% of requests will complete in less than 200ms."

**Good SLOs are**:
- Defined from the user's perspective.
- Realistic — based on what you can actually achieve, not aspirational.
- Specific — define the measurement window (rolling 30 days is common), the threshold, and the measurement method.
- Tight enough to be meaningful, loose enough to be achievable.

**Choosing the right number**: "Five nines" (99.999% = 5 minutes downtime per year) sounds impressive but is often overkill and very expensive to achieve. 99.9% (8.7 hours downtime per year) is achievable and appropriate for most services. Match the SLO to what users actually need.

### SLA (Service Level Agreement)
A contractual commitment to customers with financial consequences for violation. The SLA is the public, legally-binding version of the SLO.

**Key distinction**: Your SLO should be **tighter** than your SLA. If your SLA says 99.9%, your internal SLO should target 99.95%. This buffer ensures SLO violations (which trigger internal action) happen before SLA violations (which trigger financial penalties and customer escalations).

**SLA example**: AWS S3 SLA: 99.9% monthly uptime. If uptime drops below 99.9%, customers receive service credits. AWS internally targets much higher reliability than this.

## Error Budget

### What It Is
The error budget is the complement of the SLO: the amount of unreliability you are allowed to have over the measurement window.

**Formula**: `Error budget = 1 - SLO`

For an SLO of 99.9% over 30 days:
- Error budget = 0.1%
- 30 days = 43,200 minutes
- 0.1% of 43,200 = 43.2 minutes of acceptable downtime per month.

### How It Drives Decisions

The error budget is a **shared currency** between product/development (who want to move fast, deploy frequently) and SRE (who want to maintain reliability). Both teams agree to the SLO; both are accountable to the error budget.

**Error budget is healthy**: Development can deploy frequently, experiment, take risks. The budget absorbs mistakes.

**Error budget is near exhaustion**: Development slows down. Risky changes are deferred. Reliability work takes priority over feature work.

**Error budget is exhausted**: Development freezes non-critical deployments. SRE can enforce a "no new features" policy until reliability is restored. This is the SRE/dev contract.

**Why this works**: It prevents the permanent standoff between "SRE says no" and "product needs to ship." The error budget is an objective measure. Neither SRE nor product is being unreasonable — the math is.

### Error Budget Burn Rate Alerts
Rather than alerting when the SLO is violated (too late), alert when you're burning through the budget faster than expected.

**SRE Book alert model**:
- Fast burn (2% error budget consumed in 1 hour) → urgent page.
- Slow burn (5% consumed in 6 hours) → Slack alert.
- Very slow burn (10% in 3 days) → ticket.

Fast burn means a major outage. Slow burn means a subtle degradation. Both are caught before the budget is exhausted.

## Distributed Tracing

### What It Is
Distributed tracing tracks a single request as it flows through multiple services. It reconstructs the causal chain of calls that together constituted the user's request.

### Trace ID Propagation
When a request enters the system at the edge, a **trace ID** is generated (UUID or similar). This trace ID is propagated in request headers to every downstream service the request touches.

**Standard headers**: W3C Trace Context (`traceparent`, `tracestate`) is the interoperability standard. Zipkin (`X-B3-TraceId`, `X-B3-SpanId`), Jaeger, and AWS X-Ray each have their own formats; they are converging on W3C.

**Automatic propagation**: Service meshes (Istio, Linkerd) can automatically inject and propagate trace headers at the sidecar proxy level, so application code doesn't need to handle it. Without a service mesh, libraries (OpenTelemetry) provide instrumentation.

### Span
A **span** represents a single unit of work in the trace: one service handling one request, one database query, one external API call.

**A span contains**:
- Trace ID (links it to the overall trace).
- Span ID (unique within the trace).
- Parent Span ID (who called this span).
- Operation name (e.g., `GET /users/{id}`, `db.query users`, `redis.get session`).
- Start time and duration.
- Status (success/error).
- Attributes (HTTP method, URL, DB query, error message).
- Events (timestamped messages within the span: "cache miss at 12ms").

**Visualization**: Traces are displayed as a waterfall diagram showing spans in time order, with parent-child relationships. You can immediately see which service is contributing the most latency.

### Designing for Traceability
- **Propagate trace context at all boundaries**: HTTP, gRPC, message queues (add trace ID to message headers), scheduled jobs.
- **Instrument critical paths**: Don't trace everything (too expensive). Instrument all external calls, database queries, cache operations, and the slow parts of your hot path.
- **Sampling**: In high-throughput systems, tracing every request is too expensive. Sample: trace 1% of requests, 100% of error requests, 100% of requests slower than 1 second. Adaptive sampling adjusts rate based on traffic.
- **Add business context to spans**: Add `user_id`, `order_id`, `tenant_id` as span attributes. This enables "show me all traces for user 123" queries.

### Tools
- **Jaeger**: Open-source distributed tracing (CNCF project). Good for self-hosted.
- **Zipkin**: Original distributed tracing system from Twitter.
- **AWS X-Ray**: Managed tracing for AWS-native services.
- **Honeycomb**: Observability platform built around high-cardinality events; excellent for trace analysis.
- **Datadog APM**: Managed tracing integrated with Datadog's broader observability platform.

## Cardinality in Metrics

### Why High-Cardinality Labels Kill Prometheus

**Cardinality** in metrics refers to the number of unique time series. Each unique combination of label values creates a separate time series.

**Example**: Suppose you add a `user_id` label to a request counter metric. If you have 10 million users, Prometheus creates 10 million unique time series for that metric. Each time series uses memory in the TSDB. 10 million time series × ~1KB each = 10GB of RAM just for that one metric.

Prometheus is not designed to handle millions of time series for a single metric. High-cardinality labels cause:
- Out-of-memory errors.
- Slow queries (too many series to scan).
- Slow ingestion.
- Degraded cluster performance.

### What Makes a Label High Cardinality
- User IDs, order IDs, session IDs: unbounded cardinality. Never use as labels.
- IP addresses: technically bounded but practically very high. Avoid.
- Kubernetes pod names: medium-high cardinality. Can be acceptable.
- HTTP path with user-specific segments: `/users/123/orders` — if un-templated, every user creates a new series. Always normalize paths: `/users/{id}/orders`.
- Environment (prod/staging), region (us-east-1, eu-west-1), HTTP method (GET/POST), status code: low cardinality. Fine as labels.

### The Rule
High-cardinality data belongs in your tracing and logging systems (Jaeger, Elasticsearch, Honeycomb), not in your metrics system (Prometheus, Datadog metrics). Metrics should answer "what is the rate/latency across all requests?" Traces and logs answer "what happened for this specific user/request?"

### High Cardinality in Observability Platforms
Honeycomb, Lightstep, and similar "observability platforms" are designed from the ground up for high-cardinality data. They store events (not aggregated metrics) and allow you to group, filter, and aggregate by any field, including user ID. They are architecturally different from Prometheus and are appropriate for "why is this specific customer experiencing slow checkouts?"

## Structured Logging

### Why Unstructured Logs Are a Problem
```
[2024-01-15 10:23:45] ERROR: Payment failed for user 12345 with error "card declined" after 250ms
```

This is human-readable but machine-unfriendly. To find all payment failures for a user, you need a regex. To aggregate error rates, you need log parsing. To join with a trace, you need to manually correlate timestamps.

### Structured Logs
```json
{
  "timestamp": "2024-01-15T10:23:45.123Z",
  "level": "ERROR",
  "service": "payment-service",
  "trace_id": "7f8e9d0a1b2c3d4e",
  "span_id": "a1b2c3d4",
  "user_id": "12345",
  "order_id": "order-67890",
  "event": "payment_failed",
  "error": "card_declined",
  "duration_ms": 250,
  "payment_provider": "stripe"
}
```

This is machine-readable. You can:
- Filter by `user_id = "12345"` to find all events for that user.
- Group by `error` to find the most common payment failure reasons.
- Join with traces via `trace_id`.
- Create dashboards that count by `event = "payment_failed"`.
- Alert on high error rate by `payment_provider`.

### Implementation
- Use a structured logging library: `logrus`, `zap` (Go); `structlog` (Python); `slf4j` with JSON appender (Java).
- Standardize fields across services: always use the same field names (`trace_id`, `user_id`, `duration_ms`).
- Include the trace ID in every log line. This is the link between your logs and your traces.
- Log to stdout/stderr in JSON. Let the container runtime or log shipper (Fluentd, Filebeat) collect and forward to your log aggregation system (Elasticsearch, Loki, Splunk).

## Alerting Design: Symptom-Based vs Cause-Based

### Cause-Based Alerts (The Anti-Pattern)
Alert on the cause of the problem:
- CPU > 90%
- Database connections > 80% of max
- Disk > 85% full
- Queue depth > 10,000

**Problems**:
- CPU at 90% might not affect users at all (background batch job).
- A queue depth of 10,000 might be fine if consumers are keeping up.
- These alerts generate noise without answering "is the user affected?"
- Different root causes produce the same alert; you don't know what to fix.

### Symptom-Based Alerts (The Right Approach)
Alert on user-visible symptoms:
- Error rate > 1% (users are seeing errors).
- p99 latency > 2 seconds (users are experiencing slow responses).
- Availability < 99.9% (SLO burn rate alert).

**Advantages**:
- If users aren't affected, no alert fires.
- Directly maps to SLO burn rate.
- Fewer false positives.

**Cause-based metrics are still valuable** — but as diagnostic tools, not as alerts. When a symptom alert fires, you investigate causes (CPU, memory, DB connections, queue depth) to find the root cause.

### Alert Design Principles
1. **Every alert must be actionable**: If an alert fires and you can't do anything about it, remove it or lower it to a warning.
2. **Every alert needs a runbook**: Document what to do when this alert fires. Good runbooks include: what this alert means, what to check, how to mitigate, when to escalate.
3. **Alert on symptoms, diagnose with causes**.
4. **Use multi-window burn rate**: Alert on fast error budget burn rate, not static thresholds.
5. **Avoid alert fatigue**: An on-call engineer who receives 50 alerts per shift starts ignoring them. Tune signal-to-noise ruthlessly.

## Observability at Design Time: What to Instrument

Most observability debt comes from designing a system and adding instrumentation as an afterthought. Design for observability from the beginning:

### At Every Service Boundary
- Log incoming request details (method, path, user, trace ID, request ID).
- Log outgoing request details to dependencies (URL, parameters, latency, response status).
- Emit a latency metric per endpoint.
- Emit a success/error counter per endpoint.
- Propagate trace ID to all downstream calls.

### At Every External Dependency
- Database query duration (histogram), success/error count.
- Cache hit/miss rate.
- External API call latency and error rate.
- Queue publish/consume latency and error rate.

### Business Events
Emit structured log events or metrics for critical business operations:
- `payment_completed`, `payment_failed`, `order_created`, `user_registered`.
- These events enable both operational monitoring and business analytics from the same pipeline.

### Health and Readiness Endpoints
- `/health` (liveness): Is the process alive? Returns 200 if yes.
- `/ready` (readiness): Is the service ready to receive traffic? Returns 200 only if DB is connected, caches are warm, all dependencies are reachable.
- `/metrics` (Prometheus endpoint): Exposes all metrics for scraping.

### The Golden Signals (Google SRE Book)
Instrument these four metrics for every service:
1. **Latency**: Time to serve a request. Separate successful from failed requests.
2. **Traffic**: Request rate (requests per second).
3. **Errors**: Error rate (HTTP 5xx, exceptions, timeouts).
4. **Saturation**: How "full" the service is: CPU, memory, connection pool utilization, queue depth.

These four signals, plus a trace on every request, give you enough to debug virtually any production issue.

## On-Call and Runbook Design

### How Good Architecture Reduces Runbook Complexity
**Design for self-healing**: If a service can automatically reconnect to a database, retry a failed request with exponential backoff, or circuit-break a slow dependency — the runbook is shorter or unnecessary.

**Design for isolation**: Cell-based architecture, bulkhead isolation, and circuit breakers contain failures. When a failure is contained, the runbook says "cell X is degraded, affect is limited to Y users, here's how to accelerate recovery" — not "everything is on fire."

**Design for observability**: The runbook can only say "check the latency dashboard and look for which service is slow" if there is a latency dashboard. If the system is not instrumented, the runbook says "SSH into the box and grep the logs," which is slow and error-prone.

### Runbook Structure
A good runbook for an alert includes:
1. **Alert summary**: What this alert means in plain language.
2. **Impact**: What users are affected, to what degree.
3. **Diagnostics**: Step-by-step: "Check X dashboard. If Y, then Z." Link to specific dashboards.
4. **Mitigation**: Immediate action to reduce user impact (enable feature flag, restart service, scale up, switch to backup).
5. **Root cause investigation**: How to find the root cause after mitigation.
6. **Escalation**: When to escalate, who to call.
7. **Links**: Dashboards, traces, service dependencies, related runbooks.

## The SRE Lens on Observability

### Designing Systems That Are Observable by Default

The question to ask at every architectural decision: **"If this component fails in an unexpected way, will I be able to figure out why from existing telemetry?"**

If the answer is no, that's an observability gap. Add it now, not when you're debugging a 3am incident.

**Database layer**: Every query executed → latency histogram, slow query log, connection pool utilization.

**Cache layer**: Hit rate per cache key prefix, eviction rate, memory utilization.

**Message queue**: Consumer lag per consumer group, publish latency, DLQ depth.

**External dependencies**: Latency percentiles, error rate, circuit breaker state.

**Deployment events**: Every deploy emits an event (timestamp, version, who deployed). Visible as an annotation on dashboards. "Did the deploy cause this spike?" becomes instantly answerable.

### Observability as a Force Multiplier
Observability investment compounds over time. Each incident that is resolved faster because of good instrumentation saves more incident time than it cost to implement. Systems with good observability are:
- Easier to debug (root cause in minutes, not hours).
- Easier to operate (anomalies are caught early).
- Safer to change (you can see the effect of changes in real time).
- Less risky to hand over to new team members (the system explains itself).

## SRE Lens: What This Looks Like in Production

### P99 Latency Alert With No Trace Data
An alert fires: p99 latency for the checkout service is 4 seconds. The SRE opens the dashboard — the chart shows the spike started 20 minutes ago. But there are no traces instrumented in the checkout service. The SRE can't tell which internal step is slow. They must add logging, redeploy, and wait for the next occurrence. That's a 30-minute operational loss that proper tracing would have reduced to 2 minutes. This is observability debt being repaid under the worst possible conditions.

### Cardinality Explosion Killing Prometheus
A developer adds a `user_id` label to the `http_request_duration_seconds` metric. With 5 million active users, Prometheus ingestion rate spikes 1000x. Prometheus OOMs. All metric-based alerting goes dark. The SRE must identify the offending metric, drop it at the scrape level (metric relabeling to drop `user_id`), and restart Prometheus. The developer should have used the trace system for per-user analysis.

### Alert Fatigue Causing Missed Incident
An on-call SRE has been receiving 150 alerts per shift, mostly cause-based (CPU, disk, minor queue depth spikes). They become desensitized. A new alert fires — `user_error_rate > 2%` — but it's buried in noise and isn't caught for 25 minutes. 25 minutes of customer-facing errors during peak traffic. Fix: audit all alerts, delete any that aren't actionable within 5 minutes, convert cause-based to symptom-based, reduce alert volume by 80%.

## Common Misconceptions

- **"If it's logged, it's observable"**: Unstructured logs that are hard to query don't provide observability. Structured logs with traces and metrics constitute observability.
- **"More metrics = better observability"**: High-cardinality or irrelevant metrics add cost and noise. The right metrics are more important than many metrics.
- **"Distributed tracing is optional"**: In a microservices architecture, tracing is mandatory. Without it, a latency spike affecting the user experience is nearly impossible to attribute to a specific service in the call chain.
- **"SLOs are just marketing numbers"**: SLOs, implemented with error budgets, are a management tool that aligns development and operations priorities. When used correctly, they create a fact-based negotiation process for reliability vs feature velocity.
- **"Observability is an ops concern"**: Observability must be built in at design time by developers. Ops cannot add instrumentation to code they don't own. If developers don't instrument their code, SREs cannot observe it.

## Key Takeaways

- Metrics, traces, and logs are complementary pillars. Metrics tell you what, traces tell you where, logs tell you why. All three are needed.
- SLIs measure user-visible behavior. SLOs set targets. SLAs are external contracts. SLO must be tighter than SLA.
- Error budgets make reliability a quantitative, shared concern between product and SRE. When the budget is healthy, ship fast. When it's gone, slow down.
- Distributed tracing requires trace ID propagation at every service boundary, in headers and message metadata.
- Never use high-cardinality data (user IDs, request IDs) as metric labels. Route high-cardinality analysis to traces and logs.
- Structured logging with consistent fields enables machine-readable analysis, dashboard creation, and trace correlation.
- Alert on symptoms (user-visible impact), not causes (CPU, disk). Cause-based metrics are diagnostic tools.
- Instrument the four golden signals for every service: latency, traffic, errors, saturation.
- Design for observability at architecture time, not after you've already shipped. The cost of retroactive instrumentation is paid during incidents.

## References

- Beyer, B. et al. — *Site Reliability Engineering* (Google) — Chapters 4 (SLOs), 6 (Monitoring), 13 (Emergency Response)
- Beyer, B. et al. — *The Site Reliability Workbook* — practical implementations of SRE concepts
- Majors, C., Fong-Jones, L., Miranda, G. — *Observability Engineering* (O'Reilly) — modern observability with high cardinality
- OpenTelemetry documentation — https://opentelemetry.io/docs/ — standard instrumentation library
- Prometheus documentation — https://prometheus.io/docs/ — metrics best practices, cardinality
- Kleppmann, M. — *Designing Data-Intensive Applications*, Chapter 1 (reliability definitions)
- Distributed Tracing — W3C Trace Context spec — https://www.w3.org/TR/trace-context/

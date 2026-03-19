# API Design

> API design is the contract between services and their consumers — poor design decisions accumulate as technical debt that must be maintained forever, while good design enables independent evolution without breaking changes.

## Core Concept

An API is a contract. Every field, endpoint, method, and status code you expose is a commitment that downstream consumers will depend on. Breaking that commitment breaks your consumers. API design decisions made at v1 persist for years, sometimes decades.

The goal is to design APIs that are:
- **Intuitive**: Consumers can guess how to use them correctly.
- **Evolvable**: You can add new capabilities without breaking existing consumers.
- **Consistent**: Similar operations behave similarly.
- **Performant**: Consumers can retrieve what they need efficiently.

## REST: What It Actually Is vs What Most APIs Call REST

### Richardson Maturity Model
Most "REST APIs" are not RESTful in the academic sense. Roy Fielding defined REST with specific constraints. The Richardson Maturity Model describes levels:

- **Level 0**: HTTP as a transport for RPC. One URL, one method (POST). `POST /api` with `action: "getUser"` in the body. This is SOAP-style.
- **Level 1**: Resources. Different URLs for different resources. `GET /users/123`, `POST /orders`.
- **Level 2**: HTTP Verbs. Use GET, POST, PUT, PATCH, DELETE correctly. Use HTTP status codes meaningfully.
- **Level 3**: Hypermedia (HATEOAS). Responses include links to related resources and available actions. Clients navigate the API by following links. Rarely implemented in practice.

**What most people call "REST"**: Level 2. Resources + HTTP verbs + status codes. This is pragmatically valuable even if academically incomplete.

### REST Principles That Matter Practically

**Resources, not actions**:
- Good: `POST /orders` (create a resource)
- Bad: `POST /createOrder` (action-oriented)

**Correct HTTP method semantics**:
- `GET`: Retrieve, idempotent, safe (no side effects).
- `POST`: Create a new resource (non-idempotent).
- `PUT`: Replace a resource entirely (idempotent).
- `PATCH`: Partially update a resource (idempotent if designed correctly).
- `DELETE`: Remove a resource (idempotent).

**Correct HTTP status codes**:
- `200 OK`: Successful GET, PUT, PATCH, DELETE.
- `201 Created`: Successful POST that created a resource. Include `Location` header pointing to the new resource.
- `204 No Content`: Successful DELETE (or PUT/PATCH with no response body).
- `400 Bad Request`: Client sent invalid request (malformed JSON, missing required field).
- `401 Unauthorized`: Authentication required. (Confusingly named — it means "not authenticated.")
- `403 Forbidden`: Authenticated but not authorized.
- `404 Not Found`: Resource does not exist.
- `409 Conflict`: Conflict with current resource state (e.g., unique constraint violation, optimistic lock conflict).
- `422 Unprocessable Entity`: Request is syntactically valid but semantically invalid (validation errors).
- `429 Too Many Requests`: Rate limit exceeded.
- `500 Internal Server Error`: Server-side bug.
- `503 Service Unavailable`: Service temporarily down (overloaded, maintenance).

## REST Trade-offs: Over-fetching, Under-fetching, N+1

### Over-fetching
A REST endpoint returns more data than the client needs.

**Example**: `GET /users/123` returns a full user object with 40 fields. The client only needs `name` and `email`. 38 fields are serialized, transmitted, and parsed for nothing.

**Impact**: Bandwidth waste, slower serialization, mobile clients pay more in data costs and CPU.

### Under-fetching
A single REST endpoint doesn't return enough data, requiring multiple requests.

**Example**: Render a user profile page.
- `GET /users/123` — get user data.
- `GET /users/123/orders` — get orders.
- `GET /orders/456/items` — get items for the first order.

Three sequential requests, each dependent on the previous one. On a 100ms RTT, that's 300ms of network time before the page can render.

### N+1 Query Problem
A client fetches a list and then makes N additional requests for details of each item.

**Example**: 
- `GET /orders` returns 50 orders.
- Client loops and makes `GET /orders/{id}` for each of the 50 orders to get details.
- 51 requests total: 1 for the list, 50 for details.

**Server-side N+1**: Occurs when a REST handler fetches the list from the DB, then loops and makes N separate DB queries for related data (instead of a JOIN). ORM libraries (Django ORM, ActiveRecord) are notorious for causing this.

**Solutions**:
- Add query parameters for embedding related data: `GET /orders?include=items,shipping`.
- Design compound documents: the response includes related resources.
- Use GraphQL (addresses this architecturally).

## GraphQL

### What Problems It Solves
GraphQL was created at Facebook to address the over-fetching and under-fetching problems of REST for their mobile clients.

**How it works**: The client specifies exactly which fields it needs in the query. The server returns exactly those fields. One request can fetch deeply nested related data.

```graphql
query {
  user(id: "123") {
    name
    email
    orders(last: 5) {
      id
      total
      items {
        name
        quantity
      }
    }
  }
}
```

This fetches a user, their last 5 orders, and each order's items — in a single request. The client specifies the shape; the server provides it.

### When GraphQL Is the Right Choice
- Frontend-driven development where the client team needs flexibility to compose queries without backend changes.
- Mobile clients on constrained bandwidth where minimizing data transfer matters.
- A "graph" of interconnected data (users, orders, products, reviews) that clients navigate in varied ways.
- Multiple clients (web, iOS, Android) with different data needs from the same backend.

### When GraphQL Is the Wrong Choice
- Simple, well-defined APIs with few clients and stable access patterns. REST is simpler.
- Public APIs consumed by third-party developers — GraphQL's introspection exposes schema, which can be a security concern; caching is harder with POST-based queries.
- Small teams or early-stage products where the overhead of a GraphQL layer is not worth it.
- APIs that are primarily action-oriented (payments, webhooks, commands) rather than data-fetching.

### GraphQL Pitfalls
- **N+1 problem still exists server-side**: A GraphQL resolver for `user.orders` will fire N DB queries unless you use DataLoader (batching). DataLoader batches calls in a single event loop tick.
- **Complexity**: Schema design, resolver chains, authorization at the field level — all more complex than REST.
- **Caching**: HTTP caching doesn't work out of the box (queries are POST requests). Requires CDN-level query normalization or persisted queries.

## gRPC

### Strong Typing and Streaming
gRPC uses Protocol Buffers (protobuf) as the schema definition and serialization format. The `.proto` file is the contract. Client and server stubs are generated.

**Advantages**:
- Compile-time type safety.
- Efficient binary serialization (3–10x smaller than JSON).
- Native streaming: server streaming, client streaming, bidirectional streaming.
- HTTP/2 multiplexing.
- Auto-generated client libraries for Go, Java, Python, Ruby, etc.

### When to Use gRPC over REST
- Internal service-to-service communication where you control both sides.
- High-throughput services where JSON serialization is measurably expensive.
- Streaming use cases (log tailing, live data feeds, bidirectional protocols).
- Polyglot microservices that benefit from generated client stubs.

**Not ideal for**: Browser clients (requires gRPC-Web + proxy, adds complexity), external/public APIs, teams that don't own both producer and consumer.

## Pagination: Cursor-based vs Offset-based

### Offset-based Pagination
```
GET /orders?page=3&page_size=20
```
The server executes `SELECT ... LIMIT 20 OFFSET 40`.

**Advantages**: Simple to implement, supports random page access (jump to page 50).

**Problems**:
1. **Data drift**: If a new record is inserted before page 2 is fetched, all records shift. What was on page 2 is now on page 3. You'll see duplicates or skip records.
2. **Inefficiency at depth**: `OFFSET 100000` in PostgreSQL still scans 100,000 rows before returning 20. Deep offsets are slow.
3. **Not stable under concurrent writes**: Inserts and deletes between page requests shift results.

### Cursor-based Pagination
```
GET /orders?after=cursor_abc123&limit=20
```
The cursor encodes the position in the result set (typically the last seen ID or timestamp). The server queries `WHERE id > last_seen_id LIMIT 20`.

**Advantages**:
- Stable under concurrent writes: the cursor points to a specific position in the sorted order, not an index.
- Efficient: the query uses an indexed predicate, not OFFSET.
- Correct: no skips or duplicates regardless of inserts/deletes between pages.

**Disadvantages**:
- No random page access. You can only go forward (or backward with bi-directional cursors).
- The cursor must be opaque to the client (base64 or encrypted) to prevent clients from relying on its internal format.

**Standard implementation**: Return a cursor pointing to the last item in the current page. On the next request, pass that cursor. The server resolves the cursor to the appropriate WHERE clause.

**Recommendation**: Always use cursor-based pagination for production systems where data changes. Offset-based is acceptable for truly static data or admin tools with small datasets.

## Idempotency

### What It Means
An operation is idempotent if executing it multiple times produces the same result as executing it once. `f(f(x)) = f(x)`.

- `GET` is idempotent (and safe).
- `DELETE /orders/123` is idempotent: deleting a non-existent order returns 404, not a new deletion.
- `PUT /orders/123` with a full resource body is idempotent.
- `POST /orders` is NOT idempotent by default: two identical requests create two orders.

### Idempotency Keys
For non-idempotent operations (like payment submission), idempotency keys allow clients to safely retry.

**How it works**:
1. Client generates a unique key (UUID) for the operation.
2. Client sends the request with `Idempotency-Key: uuid-1234`.
3. Server stores the key + result in a idempotency store (Redis with 24-hour TTL).
4. If a duplicate request arrives with the same key, the server returns the stored result without re-executing.
5. Client can safely retry on network error — the second request returns the same result as the first.

**Example (Stripe's API)**: Stripe's payment API requires an `Idempotency-Key` header for all payment operations. If the client doesn't receive a response (network error), it retries with the same key. Stripe returns the original result without charging the customer again.

### Why It Matters
Without idempotency keys:
- Client sends payment request. Network error. Client doesn't know if the request was received.
- Client retries. Both requests succeed. Customer is charged twice.

With idempotency keys:
- Client retries with same key. Server detects duplicate. Returns original success. Customer is charged once.

**Design rule**: Any operation with side effects (writes, external API calls, payments) should support idempotency keys.

## Versioning Strategies

### URL Path Versioning
```
GET /api/v1/users/123
GET /api/v2/users/123
```

**Advantages**: Explicit, visible in logs and browser history. Easy to route different versions to different backends. Unambiguous which version a client is using.

**Disadvantages**: Proliferates multiple version paths in the codebase. Clients must explicitly migrate.

**When to use**: The most common approach for public APIs (Stripe, Twilio, GitHub all use this). Good when you anticipate major breaking changes and want to maintain multiple major versions simultaneously.

### Header Versioning
```
Accept: application/vnd.myapi.v2+json
```
or
```
API-Version: 2024-01-01
```
Stripe switched to date-based header versioning. The version is the date of the API schema when the client was built. They maintain every version that clients have used and don't break old clients when the schema evolves.

**Advantages**: Cleaner URLs. API version is a metadata concern, not a resource concern.

**Disadvantages**: Less visible. Harder to test in a browser. Requires middleware to route on headers.

### No Versioning (Additive-Only Changes)
Don't version the API. Instead, commit to never making breaking changes. You may only add new fields and new endpoints. Old fields and behavior are never removed.

**Advantages**: No migration burden on clients. One codebase, one endpoint.

**Requirements**: Strict additive-only discipline. Strict backward compatibility testing (consumer-driven contract testing with Pact).

**When feasible**: Internal APIs with a small, cooperative set of consumers who can update on a shared schedule.

## Rate Limiting at the API Layer

### Why It's Necessary
- Prevent abuse (DDoS, credential stuffing, scraping).
- Protect the service from runaway clients.
- Enforce tiered service plans (free: 100 req/s, paid: 10,000 req/s).
- Ensure fair usage across many clients.

### Algorithms

**Fixed window**: Count requests in a fixed time window (e.g., per minute). At the window boundary, reset the count. Problem: burst at the boundary — 1,000 requests in the last second of minute 1, 1,000 more in the first second of minute 2 = 2,000 requests in 2 seconds.

**Sliding window log**: Maintain a log of request timestamps. Count how many timestamps fall within the last N seconds. Expensive to store for high-traffic APIs.

**Sliding window counter**: Approximate sliding window using two fixed windows and interpolation. Most practical balance of accuracy and storage efficiency.

**Token bucket**: A bucket holds N tokens. Each request consumes one token. Tokens are added at a fixed rate. If the bucket is empty, the request is rejected. Allows bursting up to the bucket size.

**Leaky bucket**: Requests enter a queue. The queue processes at a fixed rate (leaks at a constant rate). Smooths out bursts. Good for rate-limiting egress.

### Response Headers
Always return rate limit information in response headers:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 450
X-RateLimit-Reset: 1706745600
Retry-After: 30
```

Return `429 Too Many Requests` when the limit is exceeded. Include `Retry-After` header.

## API Contracts and Backwards Compatibility

### Additive-Only Changes (Non-Breaking)
Safe to make without a version bump:
- Adding a new optional field to a response.
- Adding a new optional request parameter.
- Adding a new endpoint.
- Adding a new value to an enum (with caution — if clients switch on enum values, a new value may fall into an "unexpected" branch).

### Breaking Changes (Require Versioning or Coordination)
Never make without a version change or migration plan:
- Removing a field from a response.
- Renaming a field.
- Changing a field's type.
- Making a previously optional field required.
- Changing the semantics of an existing field.
- Removing an endpoint.

### Consumer-Driven Contract Testing
A technique where consumers define the API contract they depend on as a test. When the provider makes changes, it runs the consumer contracts as part of CI. If any consumer contract breaks, the build fails.

**Tool**: Pact (https://pact.io). Used at ThoughtWorks, Netflix, and others.

**Why it matters**: Without contract testing, a producer team may unknowingly break a consumer team's integration. Contract tests make this visible at the time of the change, not in production.

## Webhooks vs Polling

### Polling
The client periodically queries the API to check for new data.

```
while True:
    data = GET /orders?since=last_seen
    process(data)
    sleep(30)
```

**Advantages**: Simple client implementation. The client controls the pace.

**Disadvantages**: Latency between event and detection = polling interval. Polling generates load even when nothing has changed. Wasteful at scale (N clients polling continuously = N * polls/interval requests, most returning empty results).

### Webhooks
The server sends an HTTP POST to a client-provided URL when an event occurs. Event-driven push.

**How it works**:
1. Consumer registers a webhook URL: `POST /webhooks {"url": "https://consumer.example.com/events", "events": ["order.created", "order.shipped"]}`.
2. When the event occurs, the server sends a POST to the registered URL with the event payload.
3. Consumer returns 200 OK to acknowledge. If consumer returns non-200, the server retries.

**Advantages**: Low latency (near-real-time delivery). No polling overhead.

**Considerations**:
- The consumer endpoint must be publicly accessible (or use a relay like ngrok for development).
- Delivery is at-least-once. Consumer must handle duplicates.
- The server must retry on failed deliveries with exponential backoff.
- Signature verification: the server signs the webhook payload with an HMAC secret. Consumer verifies the signature before processing. Prevents spoofed webhooks.
- Webhook fan-out at scale: if you have 10,000 subscribers and an event fires 100 times per second, you're making 1,000,000 outgoing HTTP requests per second. Requires a dedicated webhook delivery infrastructure (queue-backed, retry logic, dead-letter handling).

### When to Use Each

| Scenario | Recommendation |
|---|---|
| Frequent events, many consumers | Webhooks |
| Simple integration, low event frequency | Polling |
| Real-time notifications | Webhooks |
| Client is a browser (can't receive inbound HTTP) | SSE or WebSocket |
| Consumer doesn't have a public URL | Polling or long-polling |

## SRE Lens: What This Looks Like in Production

### API Version Sunset Causing Client Failures
An API team deprecates v1 after 6 months' notice. On deprecation day, 3 internal services and 200 external customers are still on v1 (they didn't update). The v1 endpoints return 410 Gone. Incidents flood in. Fix: Proactively identify clients still using old versions (via API key + version headers in logs), notify them, offer migration help, track migration progress, and defer sunset until migration is complete.

### Missing Idempotency Key Causing Double Charges
A billing service sends a charge request to Stripe. Network timeout. The service retries. Both requests succeed — customer is charged twice. Root cause: the retry logic was not sending the `Idempotency-Key` header. Fix: always generate and send idempotency keys for any payment or write operation, add tests specifically for retry behavior.

### Offset Pagination Causing Missing Records in Report
A report pipeline paginates through 100,000 orders using `offset`. During the report run, 500 new orders are inserted. The offset shift causes 500 orders to be processed twice and 500 others to be skipped. The report is silently incorrect. Fix: switch to cursor-based pagination, or take a consistent snapshot at the start of the report.

## Common Misconceptions

- **"If it uses HTTP, it's REST"**: Using HTTP and HTTP verbs does not make an API RESTful. REST has specific architectural constraints. Most "REST APIs" are better described as HTTP APIs.
- **"GraphQL is always better than REST"**: GraphQL adds significant complexity. For simple, stable APIs with few clients, REST is simpler and performs better (GET requests are cacheable; GraphQL POST requests are not by default).
- **"PUT and PATCH are interchangeable"**: PUT replaces the entire resource. PATCH applies partial modifications. Sending a PATCH with only `{email: "new@example.com"}` should update only the email, not overwrite the entire user record.
- **"Adding a new field to a response is always safe"**: Adding fields can break clients that use strict deserialization (unknown fields cause errors). Clients should use lenient/tolerant reader pattern. But many older or rigid clients will break.
- **"Rate limiting only matters for public APIs"**: Internal services can abuse each other under load spikes. Rate limiting between internal services prevents cascading failures.

## Key Takeaways

- REST is defined by architectural constraints, not just "uses HTTP." Most "REST APIs" are level-2 HTTP APIs.
- Use correct HTTP verbs and status codes consistently. `400` for client errors, `5xx` for server errors, `404` for not found, `409` for conflict.
- GraphQL solves over-fetching and under-fetching for complex, graph-like data accessed by multiple clients with different needs.
- gRPC beats REST for internal service-to-service communication: binary serialization, strong typing, streaming.
- Use cursor-based pagination for production systems. Offset pagination is incorrect for changing data.
- Implement idempotency keys for all non-idempotent write operations, especially payments.
- API versioning should support additive-only changes without breaking existing clients.
- Webhooks over polling for event-driven integrations where the consumer can receive inbound HTTP.

## References

- Fielding, R. (2000). "Architectural Styles and the Design of Network-based Software Architectures." PhD Thesis, UC Irvine. — the original REST thesis
- Richardson, L. and Amundsen, M. — *RESTful Web APIs* (O'Reilly)
- Kleppmann, M. — *Designing Data-Intensive Applications*, Chapter 12
- Stripe API documentation — https://stripe.com/docs/api — example of excellent API design (idempotency, versioning)
- gRPC documentation — https://grpc.io/docs/
- Pact — consumer-driven contract testing — https://pact.io

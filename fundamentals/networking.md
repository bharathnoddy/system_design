# Networking

> Network protocols are not just transport details — the choice of TCP vs UDP, HTTP/1.1 vs HTTP/2, and WebSocket vs SSE directly determines your system's latency, scalability, and failure characteristics.

## Core Concept

Network communication is the seam between distributed system components. Protocol choices made at design time have lasting consequences: they affect tail latency, concurrent connection counts, server memory, and the complexity of failure handling. Understanding what each protocol trades off is prerequisite knowledge for any distributed system design.

## TCP vs UDP

This is not "TCP is reliable, UDP is fast." The full picture is more nuanced.

### TCP (Transmission Control Protocol)

**What it provides**:
- **Reliable delivery**: Every segment is acknowledged. Lost segments are retransmitted.
- **Ordered delivery**: Segments are reassembled in sequence even if they arrive out of order.
- **Flow control**: The receiver's buffer size limits how fast the sender transmits.
- **Congestion control**: TCP backs off when it detects network congestion (packet loss or ECN marks).

**Connection overhead**: TCP requires a 3-way handshake before any data can be sent (SYN, SYN-ACK, ACK). This is 1.5 RTT of overhead before the first byte of data. With TLS, add another 1–2 RTTs.

**Head-of-line blocking**: If one segment is lost, subsequent segments — even if already received — are held back until the lost segment is retransmitted. This is TCP's head-of-line blocking.

**When TCP is the right choice**:
- Any workload where data loss is unacceptable: HTTP APIs, database connections, file transfers.
- Ordered delivery matters: streaming media with synchronization, financial transactions.
- Congestion control is desired: TCP is a good citizen — it backs off when networks are congested.

### UDP (User Datagram Protocol)

**What it provides**:
- **No reliability**: Packets may be lost, duplicated, or delivered out of order. UDP does nothing about this.
- **No ordering**: Packets may arrive out of order.
- **No connection**: Each packet is independent. No handshake.
- **No congestion control**: UDP will saturate a link until the application throttles itself.

**Latency benefit**: No handshake, no retransmission waiting, no head-of-line blocking. The application receives whatever arrives immediately.

**When UDP is the right choice**:
- **Real-time media (audio/video)**: A 200ms-old video frame is worthless. Better to show a glitchy frame than to wait for retransmission. Audio/video codecs handle packet loss with interpolation.
- **DNS**: Every query is independent, stateless, and small. A lost query is just re-sent by the client. The overhead of TCP for every DNS lookup would be enormous.
- **QUIC / HTTP/3**: Implements its own reliability and congestion control in user space, on top of UDP, to avoid OS-level TCP head-of-line blocking.
- **Gaming**: Real-time position updates. An old position packet is irrelevant once a newer one arrives.
- **IoT sensor telemetry**: High-frequency, low-consequence data. A missed reading is acceptable.

**The real tradeoff**: UDP requires the application to implement whatever reliability it needs. If you need ordered, reliable delivery, you're re-implementing TCP. Use UDP only when you specifically need to deviate from TCP's model.

## HTTP/1.1 vs HTTP/2 vs HTTP/3

### HTTP/1.1

**Connection model**: One request at a time per connection (pipelining is theoretically possible but practically disabled due to head-of-line blocking issues). To send N requests in parallel, the browser opens N TCP connections (typically max 6 per origin).

**Head-of-line blocking**: If one request is slow, it blocks subsequent requests on that connection.

**Headers**: Plain text, repeated in full on every request. No compression.

**Performance implications**: 6 parallel connections per origin is a fundamental parallelism limit. HTTP/1.1 drove a generation of performance anti-patterns: domain sharding (distribute assets across multiple domains to get more than 6 parallel connections), concatenating JS/CSS bundles, CSS sprites to reduce request count.

### HTTP/2

**Connection model**: A single TCP connection multiplexes many logical streams (requests/responses). Each stream is independent. No need for multiple connections.

**Head-of-line blocking**: Eliminated at the HTTP layer — one slow stream does not block other streams. However, TCP-level head-of-line blocking still exists: a lost TCP segment blocks all HTTP/2 streams on that connection.

**Header compression**: HPACK compression reduces header overhead. Repeated headers (like `Cookie` or `Authorization`) are not retransmitted in full.

**Server push**: The server can proactively send resources the client will need before they are requested. In practice, this has been deprecated or disabled in most browsers due to complexity.

**Multiplexing benefit**: Assets that HTTP/1.1 required domain sharding or bundling to parallelize can now be requested individually. Smaller files, better caching granularity.

**Still has**: TCP head-of-line blocking at the transport layer. A single packet loss can pause all streams.

### HTTP/3 (over QUIC)

**QUIC (Quick UDP Internet Connections)**: A transport protocol built on UDP. Implements reliability, congestion control, and multiplexing in user space, eliminating TCP's head-of-line blocking.

**Key properties**:
- **0-RTT or 1-RTT connection establishment**: QUIC combines the transport handshake and TLS handshake. For known hosts, 0-RTT resumes in a single round trip.
- **Stream-level independence**: A lost packet only blocks the stream it belongs to, not all streams. This eliminates the TCP head-of-line blocking that HTTP/2 still suffers from.
- **Connection migration**: A QUIC connection is identified by a connection ID, not by IP+port. If a mobile client switches from WiFi to cellular (IP change), the QUIC connection survives the IP change. TCP connections break on IP change.

**Performance impact**: Primarily benefits high-latency, lossy networks (mobile, intercontinental). On low-latency LAN connections, the improvement is marginal.

**Adoption**: HTTP/3 is now supported by all major browsers and CDNs (Cloudflare, Fastly, Google). Backend support is growing but less universal.

## WebSockets

### What They Are
WebSocket provides a full-duplex bidirectional communication channel over a single TCP connection. The server can push data to the client at any time without the client polling.

**Protocol**: WebSocket begins as an HTTP/1.1 Upgrade request. Once the server responds with 101 Switching Protocols, the connection switches to the WebSocket protocol. TLS (WSS) works normally.

### When to Use
- Real-time bidirectional communication: live chat, collaborative editing (Google Docs), multiplayer games.
- Server needs to push updates to clients: live sports scores, financial market data, real-time notifications.
- Low-latency server-to-client messages: you need the server to initiate the push without waiting for a client poll.

### Connection Overhead and Scaling Challenges
Each WebSocket is a persistent TCP connection. A server with 100,000 concurrent WebSocket connections maintains 100,000 open file descriptors. Each connection consumes ~40–50KB of kernel memory for TCP buffers.

**Scaling**: Stateful WebSocket servers cannot be arbitrarily scaled horizontally. When a client is connected to server A, server A must be the one to receive pushes for that client. This requires:
- A pub/sub layer (Redis pub/sub, Kafka) for server-to-server message routing.
- Or a centralized WebSocket gateway (Pusher, Ably, managed WebSocket service).

**Example**: A chat application. User A is connected to server 1. User B is connected to server 2. User A sends a message. Server 1 publishes to Redis. Server 2 subscribed to the channel, receives the message, pushes to User B's WebSocket connection.

### Kubernetes/Container Environments
WebSocket connections and Kubernetes don't mix naturally. Pod autoscaling terminates pods that are running WebSocket connections. Use connection draining and graceful shutdown to allow existing connections to finish before termination.

## Server-Sent Events (SSE)

### What They Are
SSE is a one-directional push protocol: the server sends a stream of events to the client over a standard HTTP connection. The client cannot send data over the SSE connection.

**Protocol**: The client makes a standard HTTP GET request to an SSE endpoint. The server responds with `Content-Type: text/event-stream` and keeps the connection open, periodically flushing lines of text.

### When to Use
- One-directional server-to-client push: live dashboards, progress indicators, activity feeds, notification streams.
- Browser clients where WebSocket's bidirectional overhead is unnecessary.
- Simpler to implement and operate than WebSocket for read-only streams.

### Comparison with WebSockets

| Feature | WebSocket | SSE |
|---|---|---|
| Direction | Bidirectional | Server-to-client only |
| Protocol | Custom framing | HTTP/1.1 or HTTP/2 |
| Browser support | All modern browsers | All modern browsers |
| Reconnection | Manual | Automatic (built into spec) |
| Load balancer compatibility | Requires L7 WebSocket support | Standard HTTP, works everywhere |
| Complexity | Higher | Lower |

**SSE with HTTP/2**: SSE over HTTP/2 is very efficient — multiple SSE streams multiplex over a single TCP connection, each as an independent HTTP/2 stream. This is a significant advantage over WebSocket with HTTP/2.

## Long Polling

### What It Is
Long polling is a technique for server-to-client push using HTTP, before WebSocket was available.

**How it works**:
1. Client sends an HTTP request to the server asking for new data.
2. Instead of responding immediately, the server holds the connection open until it has new data to send (or until a timeout, e.g., 30 seconds).
3. When new data is available, the server responds.
4. The client immediately sends another request.

### Historical Context
Long polling predates WebSocket and SSE. It was the mechanism behind early real-time apps (Gmail chat, Facebook early notifications, Comet framework).

**Problems**: Increased server connection overhead, latency between messages (one round trip per batch of messages), HTTP overhead on every "cycle."

**Current relevance**: Mostly replaced by WebSocket and SSE. Still used in environments where WebSocket is blocked (some corporate firewalls block WebSocket upgrades). Some cloud services (like some SQS implementations) use long polling for their pull APIs.

## gRPC

### What It Is
gRPC is a high-performance RPC (Remote Procedure Call) framework by Google. Uses Protocol Buffers (protobuf) for serialization and HTTP/2 for transport.

### Why It Beats REST for Service-to-Service Communication

**Strong typing**: Protobuf schemas are the contract. Client and server code is generated from the `.proto` file. Type mismatches are caught at compile time.

**Binary serialization**: Protobuf is 3–10x more compact than JSON. Faster serialization and deserialization.

**HTTP/2**: Full multiplexing, header compression, bidirectional streaming.

**Streaming**: gRPC supports 4 communication patterns:
- **Unary**: Single request, single response (like REST).
- **Server streaming**: Single request, stream of responses.
- **Client streaming**: Stream of requests, single response.
- **Bidirectional streaming**: Both sides stream simultaneously.

### When gRPC Beats REST
- Internal service-to-service communication where you control both ends.
- High-throughput APIs where JSON parsing overhead is measurable.
- Streaming APIs: log streaming, real-time data feeds, bidirectional protocols.
- Polyglot microservices: protobuf code generation for Go, Java, Python, Rust, etc.

### When REST Is Still Preferable
- Public APIs consumed by external developers (JSON is more universally accessible).
- Browser clients without a gRPC-Web layer.
- Teams that don't own both producer and consumer (protobuf schema coordination overhead).

## DNS

### Resolution Chain
1. Browser checks local DNS cache.
2. OS checks `/etc/hosts` and OS DNS cache.
3. Query sent to recursive resolver (typically ISP or Google 8.8.8.8).
4. Recursive resolver queries root name servers, TLD name servers, then authoritative name servers.
5. Authoritative name server returns the record.
6. Response cached at each layer with TTL.

### TTLs and Propagation
DNS TTL controls how long a record is cached. A TTL of 300 seconds (5 minutes) means changes take up to 5 minutes to reach clients with stale caches.

**For failover operations**:
- Pre-reduce TTL to 60 seconds hours before a planned failover.
- Execute the failover change.
- After the old TTL has expired everywhere, you can increase TTL back.
- If you don't pre-reduce, your "instant failover" is actually slow because existing caches hold the old record for the full old TTL.

### DNS and Service Discovery
Internal service discovery (how services in a cluster find each other) is often DNS-based:
- Kubernetes DNS: Every service gets a DNS name (`service-name.namespace.svc.cluster.local`). The CoreDNS service resolves these to ClusterIPs.
- AWS Route 53 private zones for VPC-internal DNS.
- Consul DNS interface for service mesh-adjacent service discovery.

## TCP Connection Pooling

### Why It Matters for Latency
Every new TCP connection requires a 3-way handshake (1.5 RTT) before data can be sent. With TLS, add 1–2 more RTTs. For a service making 10,000 requests/second to a database, creating a new connection for each request would add 15–30ms of overhead per request and saturate the server with connection setup work.

**Connection pooling**: Maintain a pool of pre-established TCP connections. Requests reuse existing connections. The handshake cost is amortized over many requests.

**Pool sizing**: Too few connections → requests queue waiting for a connection. Too many connections → server is overwhelmed by open connections. Target: enough connections to keep all request threads busy, but not more.

**Pool parameters** (for database connection pools):
- `min_pool_size`: Connections maintained even when idle.
- `max_pool_size`: Maximum connections. PostgreSQL default max is 100.
- `connection_timeout`: How long to wait for an available connection before failing.
- `idle_timeout`: How long an idle connection is kept before being closed.

## TLS

### Handshake Cost
TLS 1.2: 2 RTTs to establish. TLS 1.3: 1 RTT (or 0-RTT for resumed sessions).

On a 50ms RTT connection (cross-region), TLS 1.2 adds 100ms to every new connection. This is why connection pooling is critical for TLS-protected services.

### Session Resumption
TLS 1.2 introduced session tickets and session IDs to resume sessions without a full handshake. TLS 1.3 supports 0-RTT resumption: the client can send application data in the first handshake message if it has a pre-shared key from a previous session.

**0-RTT security caveat**: 0-RTT data is not protected against replay attacks. Safe for idempotent reads; should not be used for non-idempotent writes without application-level replay protection.

### mTLS (Mutual TLS)
In standard TLS, only the server presents a certificate. In mTLS, both the client and server present certificates. The server verifies the client's identity.

**Use case**: Service-to-service authentication in a zero-trust network. Instead of relying on network-level controls ("if the request came from inside the VPC, trust it"), mTLS ensures every service is authenticated cryptographically.

**Implemented by**: Istio service mesh (automatic mTLS for all pod-to-pod traffic), AWS App Mesh, Linkerd.

**Operational overhead**: Each service needs a certificate. A PKI (Public Key Infrastructure) must issue, rotate, and revoke certificates. Service meshes like Istio automate this with short-lived certificates (24-hour TTL) rotated automatically.

## Latency Numbers Every Engineer Should Know

These numbers come from Jeff Dean's famous "Latency Numbers Every Programmer Should Know" (circa 2012, approximate modern values):

| Operation | Approximate Latency |
|---|---|
| L1 cache reference | ~1 ns |
| L2 cache reference | ~4 ns |
| L3 cache reference | ~10 ns |
| Main memory (DRAM) reference | ~100 ns |
| SSD random read (4KB) | ~100 µs |
| HDD seek | ~10 ms |
| Network RTT, same datacenter | ~0.5 ms |
| Network RTT, same cloud region (cross-AZ) | ~1–3 ms |
| Network RTT, cross-continent | ~50–150 ms |
| Network RTT, around the world | ~300 ms |
| Read 1MB from memory | ~250 µs |
| Read 1MB from SSD | ~1 ms |
| Read 1MB from HDD | ~20 ms |
| Read 1MB from network (1Gbps, intra-DC) | ~8 ms |

**Practical implications**:
- A database call (0.5ms+) is 5,000x more expensive than a memory operation. Caching is not premature optimization at scale.
- Cross-AZ latency (~2ms) is cheap enough to do synchronously on the hot path. Cross-region (~100ms) is not.
- Disk reads (SSD: 100µs, HDD: 10ms) are expensive. Avoid random HDD reads on the hot path.
- Memory bandwidth is abundant within a machine; network bandwidth is constrained.

## SRE Lens: What This Looks Like in Production

### Connection Pool Exhaustion
A service's database connection pool is set to `max_pool_size=10`. Under moderate load, requests start timing out with "could not acquire connection within 5 seconds." The service is handling 50 concurrent requests, each needing a DB connection. 40 are waiting for a pool connection. Fix: increase pool size, add connection timeout alerting, investigate why so many concurrent requests need simultaneous DB connections.

### TLS Handshake Bottleneck on New Connection Spike
After a deploy, thousands of connections are re-established simultaneously (old connections closed, new process instances start). TLS handshakes consume 100% CPU on load balancers for 30 seconds. Latency spikes until connection pools warm up. Fix: pre-warm connections during deploy (health check that exercises the connection pool before traffic is switched), use 0-RTT TLS resumption where appropriate.

### DNS TTL Too High During Failover
A primary Redis cluster fails. The failover involves updating a DNS record to point to the standby. The DNS TTL is 3600 seconds (1 hour). All services have cached the old address. They continue hitting the dead Redis for up to 1 hour. Fix: reduce Redis DNS TTL to 60 seconds, pre-reduce before planned maintenance, use DNS-aware connection pools that respect TTL changes.

### HTTP/2 Multiplexing Causing Unexpected Queue Depth
A service uses HTTP/2 to call a downstream service. The downstream service has high latency on some requests. Because HTTP/2 multiplexes over a single connection, many requests are queued behind slow ones. The actual head-of-line blocking is at the application layer inside the HTTP/2 stream processing. Fix: tune HTTP/2 max concurrent streams, implement circuit breaking at the HTTP/2 level, or open multiple HTTP/2 connections for parallel processing.

## Common Misconceptions

- **"UDP is always faster than TCP"**: UDP has lower overhead, but applications often need reliability that UDP lacks. Implementing reliability in user space may be slower than kernel-level TCP.
- **"HTTP/2 eliminates head-of-line blocking"**: HTTP/2 eliminates application-level HOL blocking. TCP-level HOL blocking remains. HTTP/3 addresses this with QUIC.
- **"WebSocket is the best choice for real-time"**: SSE is simpler and sufficient for server-to-client push. WebSocket is needed only for bidirectional communication.
- **"DNS changes propagate instantly"**: DNS changes propagate after cached records expire. TTL controls this. A 24-hour TTL means 24 hours for full propagation.
- **"mTLS is too complex for internal services"**: Service meshes like Istio provide automatic certificate rotation and mTLS with zero application code changes. The operational cost is in the mesh, not the application.

## Key Takeaways

- TCP provides reliability and ordering at the cost of handshake overhead and HOL blocking. UDP provides raw speed at the cost of reliability guarantees.
- HTTP/2 multiplexes over one connection, eliminating per-request connection overhead and application HOL blocking. TCP HOL blocking remains.
- HTTP/3 over QUIC eliminates TCP HOL blocking and enables 0-RTT and connection migration.
- WebSocket for bidirectional real-time; SSE for server-to-client push (simpler, standard HTTP).
- gRPC beats REST for internal service-to-service communication: binary serialization, strong typing, streaming.
- DNS TTL controls failover speed. Pre-reduce TTL before planned failovers. Low TTL for failover-critical DNS records.
- Connection pooling amortizes TLS handshake cost. Size pools appropriately.
- Know your latency numbers: memory (100ns), same-DC network (0.5ms), cross-AZ (2ms), cross-region (100ms).

## References

- Stevens, W.R. — *TCP/IP Illustrated, Volume 1* — authoritative TCP/IP reference
- HTTP/3 explained — https://http3-explained.haxx.se/ — by Daniel Stenberg (curl author)
- Dean, J. and Ghemawat, S. — "Latency Numbers Every Programmer Should Know" (Google)
- RFC 7540 — HTTP/2 specification
- RFC 9114 — HTTP/3 specification
- gRPC documentation — https://grpc.io/docs/
- Grigorik, I. — *High Performance Browser Networking* (O'Reilly) — free online

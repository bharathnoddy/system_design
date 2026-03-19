# Load Balancing

> Load balancing distributes incoming traffic across multiple servers, and the choice of algorithm, layer, and failure handling strategy determines whether your load balancer makes your system resilient or becomes its single point of failure.

## Core Concept

A load balancer sits between clients and a pool of servers, distributing requests to ensure no single server is overwhelmed, to maximize throughput, and to route around failed instances. Without a load balancer, your service can only scale as far as one machine, and that machine becomes a single point of failure.

A load balancer is itself infrastructure that must be:
- Highly available (redundant pair minimum)
- Fast enough to not become the bottleneck
- Intelligent enough to route around failures
- Transparent enough not to break application semantics

## L4 vs L7 Load Balancing

### L4 (Transport Layer) Load Balancing

Operates at TCP/UDP (OSI layer 4). The load balancer forwards packets based on IP address and port without reading packet content.

**How it works**:
1. Client establishes a TCP connection to the load balancer's VIP (Virtual IP).
2. The load balancer uses NAT or TCP proxying to forward the connection to a backend.
3. Subsequent packets in that TCP connection go to the same backend.
4. The load balancer does not parse HTTP, gRPC, or any application protocol.

**Characteristics**:
- Very fast and low latency — minimal processing per packet.
- Cannot make routing decisions based on application-level data (URL path, HTTP headers, gRPC service name).
- Connection-level granularity — cannot load-balance individual HTTP/2 streams at L4.

**When to use**: High-throughput, low-latency workloads where routing decisions do not need to inspect content. TCP services that are not HTTP/HTTPS.

**Examples**: AWS NLB (Network Load Balancer), HAProxy in TCP mode, hardware load balancers (F5 in TCP mode).

### L7 (Application Layer) Load Balancing

Operates at HTTP/HTTPS/gRPC (OSI layer 7). The load balancer terminates the connection, reads the request, and makes routing decisions based on content.

**How it works**:
1. Client establishes a TLS connection to the load balancer.
2. The load balancer decrypts, reads headers, path, host, cookies.
3. Routes to a backend based on routing rules.
4. Establishes a potentially pooled, long-lived connection to the backend.

**Characteristics**:
- Routes by URL path, HTTP headers, canary flags, A/B experiments.
- Terminates TLS, offloading crypto from backends.
- Handles HTTP/2 and gRPC request multiplexing correctly — each stream routed independently.
- Injects headers (X-Forwarded-For, X-Request-ID), rewrites URLs, modifies responses.
- Higher CPU overhead than L4 due to protocol parsing and TLS termination.

**Examples**: NGINX, HAProxy (HTTP mode), AWS ALB, Envoy, Traefik.

### L4 vs L7 Summary

| Feature | L4 | L7 |
|---|---|---|
| Protocol awareness | No (TCP/UDP only) | Yes (HTTP, gRPC, WebSocket) |
| Routing granularity | Per connection | Per request |
| TLS termination | No (passthrough) | Yes |
| Header manipulation | No | Yes |
| URL-based routing | No | Yes |
| Performance overhead | Very low | Moderate |
| HTTP/2 stream routing | No | Yes |

## Load Balancing Algorithms

### Round Robin
Distribute requests in a circular sequence: backend 1, backend 2, backend 3, backend 1, backend 2, ...

**Best for**: Identical backend capacity and uniform request cost. Simple, even distribution.

**Failure mode**: Variable-cost requests (1ms vs 10s) result in uneven actual load even with even distribution.

### Weighted Round Robin
Backends assigned a weight proportional to capacity. A backend with weight 3 receives 3x the requests of one with weight 1.

**Best for**: Heterogeneous backend pools, canary deployments (route 5% of traffic to new version).

### Least Connections
Route the next request to the backend with the fewest active connections.

**Best for**: Variable-cost requests. Prevents congestion buildup at slow backends.

**Variant — Least Outstanding Requests (LOR)**: Counts in-flight requests rather than open TCP connections. More accurate.

### IP Hash
Hash the client IP to determine the backend. Same IP always goes to the same backend.

**Problem**: Clients behind NAT share a single IP. Load distribution is by IP, not request volume. Unreliable stickiness.

### Random with Power of Two Choices
Pick two backends at random, send to the one with fewer active connections. Near-optimal load distribution with very low overhead. Better than pure round-robin, no global state needed.

## Consistent Hashing in Load Balancing

Consistent hashing routes requests for the same object (user ID, resource ID) to the same backend, minimizing remapping when backends are added or removed.

**How it works**: Backend servers and request keys map to positions on a hash ring. A key is served by the first server clockwise from the key's position. When a server is added, only keys between the new server and its predecessor are remapped.

**Use cases**:
- Memcached/Redis sharding: same cache key always hits the same node, maximizing hit rate.
- Stateful backends: same entity routed to same worker.
- CDN origin selection: cache misses for the same URL hit the same origin.

**Virtual nodes**: Each physical server gets multiple ring positions, improving load distribution for small backend pools.

## Health Checks: Active vs Passive

### Active Health Checks
The load balancer proactively sends health check requests to backends on a regular interval.

**Types**:
- **TCP connect**: Verifies the port is listening. Minimal overhead, does not verify application health.
- **HTTP health check**: Polls a dedicated endpoint (`/health`, `/ready`). Should verify DB connectivity and dependency availability, not just that the process is running.
- **gRPC health check**: Uses the gRPC Health Checking Protocol (`grpc.health.v1.Health/Check`).

**Parameters**: `interval` (5–10s typical), `healthy_threshold` (consecutive successes to mark healthy), `unhealthy_threshold` (consecutive failures to mark unhealthy), `timeout`.

### Passive Health Checks
The load balancer observes real traffic responses. If a backend consistently returns 5xx errors or times out, it is marked unhealthy.

**Advantages**: No extra health check traffic. Detects degraded backends on real request patterns.

**Disadvantages**: Real client requests fail before the backend is removed. Slow to detect failures for low-traffic backends.

### Best Practice: Use Both
Active health checks for baseline up/down detection. Passive health checks (circuit breaker) to detect backends that are technically alive but returning errors. Envoy, NGINX, and HAProxy support both.

## Sticky Sessions

### What They Are
Sticky sessions (session affinity) ensure requests from the same client always go to the same backend. Implemented via cookie (the load balancer sets `AWSALB`, `SERVERID`) or IP hash.

### Why They Are a Problem for Scale

1. **Uneven distribution**: An overloaded sticky backend cannot shed load to other backends.
2. **State is coupled to the server**: Sticky sessions exist because server-side state is not shared. This is the underlying anti-pattern.
3. **Deploy complications**: Restarting a backend invalidates all its sticky sessions.
4. **Scale-out doesn't help**: New backends only receive new sessions, not redistributed existing ones.

### The Right Solution
Move session state to a shared external store (Redis, DynamoDB). All backends are stateless and can serve any request. Sticky sessions become unnecessary.

**Sticky sessions are a symptom of stateful servers. Fix the statefulness, not the symptom.**

Legitimate uses: long-lived WebSocket connections, streaming sessions, deliberate per-client sharding. These are design choices, not workarounds.

## Global Load Balancing

### GeoDNS
DNS returns different IP addresses based on the geographic location of the DNS resolver. European users resolve to European servers; Asian users resolve to Asian servers.

**Limitations**:
- DNS TTLs make failover slow. Clients cache the old IP until TTL expires.
- Resolver IP is not always near the user (corporate DNS resolvers may be in a different region).
- No real-time backend health awareness.

### Anycast
The same IP address is announced from multiple geographic locations via BGP. Routers direct traffic to the nearest announcement point automatically.

**Advantages**: Transparent to clients. Failover handled by BGP routing convergence (faster than DNS TTL). Used by CDNs for edge node routing.

**Used in**: Cloudflare, Fastly, Google's global network, DNS root servers (all 13 root server addresses are anycast IPs served by hundreds of nodes globally).

### Global Server Load Balancer (GSLB)
Routes based on geographic proximity, backend health, load, and latency. Products: AWS Global Accelerator, Cloudflare Load Balancing, Akamai GSLB.

## Load Balancing vs Service Mesh

### Traditional Load Balancing (North-South Traffic)
Handles traffic entering the cluster from outside — from users or external systems. Centralized appliance or proxy.

### Service Mesh (East-West Traffic)
Handles traffic between services within the cluster. Every service gets a sidecar proxy (Envoy) that handles:
- Load balancing to downstream services.
- Circuit breaking.
- Retry with exponential backoff.
- mTLS for service-to-service authentication.
- Distributed tracing header injection.
- Traffic splitting for canary deployments.

**Key difference**: A load balancer is centralized. A service mesh distributes load balancing logic into every service instance. No central proxy to overload.

**When to use a service mesh**: Microservices with complex service-to-service communication, mTLS everywhere requirements, sophisticated canary/traffic-splitting needs, inter-service observability.

**Operational cost**: Envoy sidecar proxies add ~1ms latency and memory per pod. Istio control plane is operationally complex. Introduce it when you have concrete problems it solves, not as default infrastructure.

## Hardware LB vs Software LB

### Hardware Load Balancers (F5 BIG-IP, Citrix ADC)
Dedicated physical appliances with purpose-built ASICs for packet forwarding.

**Advantages**: Very high throughput (100Gbps+), low latency, hardware-accelerated SSL offload.

**Disadvantages**: Expensive ($50K–$500K+), slow configuration changes, vendor lock-in, fixed capacity, requires a redundant pair.

**When used**: Traditional enterprise data centers, financial services with compliance requirements, workloads needing 100Gbps+ throughput.

### Software Load Balancers

**NGINX**: Started as a web server, widely used as L7 reverse proxy and load balancer. Large ecosystem and community.

**HAProxy**: Purpose-built load balancer. Very high performance, low memory footprint, powerful ACL-based routing. The standard choice for high-performance L4/L7 load balancing.

**Envoy**: Modern, cloud-native, gRPC-first, HTTP/2 native. Configured dynamically via xDS APIs (Istio's data plane). Standard choice for service meshes.

**Cloud Load Balancers (AWS ALB/NLB, GCP LB)**: Managed, elastic, deeply integrated with cloud infrastructure. Default choice for cloud-native applications.

## Connection Draining During Deploys

When a backend is removed from the pool (for deploy, scale-down, or maintenance), in-flight requests should complete rather than being abruptly terminated.

### How It Works
1. Backend is marked "draining." No new requests are sent to it.
2. In-flight requests continue to be served until they complete.
3. After the drain timeout (typically 30–300 seconds), the backend is forcibly terminated.
4. Load balancer removes it from the pool.

### AWS Implementation
AWS ALB/NLB: "Deregistration delay" on the Target Group. Default is 300 seconds.

### Zero-Downtime Deploy Pattern
1. Autoscaler launches new instances with new code.
2. New instances pass health checks and join the pool.
3. Old instances are removed (draining starts).
4. After drain timeout, old instances terminate.
5. No disruption to in-flight requests.

## How This Appears in Real Systems

**HAProxy at GitHub/Facebook**: GitHub has used HAProxy as its primary load balancer for years. Facebook ran HAProxy at enormous scale before building custom solutions. HAProxy's transparent proxy mode and its rich stats socket interface make it highly instrumentable.

**AWS ALB weighted target groups**: AWS ALB supports weighted forwarding rules, enabling blue/green deployments and canary releases natively. Set 90% weight on old version, 10% on new, monitor metrics, shift weights.

**Envoy in Kubernetes**: When using Istio, every pod has an Envoy sidecar. All ingress and egress traffic is intercepted by Envoy. Routing decisions, retries, and circuit breaking happen at the sidecar level — no application code changes needed.

## SRE Lens: What This Looks Like in Production

### Cascading Failure After Backend Removal
An SRE removes 2 of 10 backends during a quick deploy. The remaining 8 receive 125% of normal load. One backend's connection pool fills and it starts returning 503s. The load balancer marks it unhealthy. Now 7 backends receive ~143% of normal load. Another backend fails. Cascade. Fix: Only remove backends when replacements are healthy and in pool. Use circuit breakers. Implement gradual traffic shifting.

### DNS Failover Slower Than Expected
Primary datacenter experiences a complete power failure. The SRE team updates GeoDNS to point all traffic to the secondary. Traffic does not shift for 5 minutes because DNS TTL is set to 300 seconds. Many clients are stuck resolving to the dead datacenter. Fix: Pre-reduce TTL before maintenance windows. Use low TTLs (30–60 seconds) for public-facing DNS when rapid failover is important. Consider Anycast for network-level failover.

### Sticky Sessions Preventing Scale-Out
A deployment team doubles the backend pool from 4 to 8 instances to handle an expected traffic spike. Only the original 4 instances are handling the load. Root cause: cookie-based sticky sessions. Existing users are all stuck to the original 4 backends. New backends only receive new sessions. Fix: Migrate session state to Redis, remove sticky sessions.

## Common Misconceptions

- **"L7 is always better than L4"**: L7 adds latency and CPU overhead. For high-throughput non-HTTP TCP workloads, L4 is the right choice.
- **"Round-robin is fair"**: It distributes connections equally, not load equally. Variable-cost requests cause uneven server loads with round-robin.
- **"Active health checks are enough"**: A backend can pass TCP health checks while returning application errors. Layer both active and passive checks.
- **"Sticky sessions are an acceptable long-term architecture"**: They are a workaround for stateful servers. Fix the statefulness.
- **"The load balancer is not a bottleneck"**: A single NGINX instance handles ~50K RPS before becoming a bottleneck. Use multiple load balancers behind Anycast or use managed cloud load balancers.

## Key Takeaways

- L4 is fast and simple: TCP/UDP routing by IP and port, no application-layer awareness.
- L7 is smart: HTTP/gRPC routing by path, header, method. Higher overhead, much more capability.
- Round-robin for uniform workloads; least-connections for variable-cost workloads; consistent hashing for cache affinity.
- Active health checks detect dead backends; passive health checks detect degraded ones. Use both.
- Sticky sessions are an anti-pattern for scalability. Move session state to a shared store.
- GeoDNS for geographic routing; Anycast for network-level failover without DNS TTL delays.
- Service meshes handle east-west traffic with distributed load balancing, mTLS, and observability as sidecars.
- Connection draining is essential for zero-downtime deploys. Configure it and test it.

## References

- NGINX documentation — Load Balancing — https://docs.nginx.com/nginx/admin-guide/load-balancer/
- HAProxy documentation — https://www.haproxy.org/
- Envoy documentation — https://www.envoyproxy.io/docs/envoy/latest/
- Karger, D. et al. (1997). "Consistent Hashing and Random Trees." STOC.
- AWS ALB documentation — Target Groups and Deregistration Delay
- Mitzenmacher, M. (2001). "The Power of Two Choices in Randomized Load Balancing." IEEE TPDS.

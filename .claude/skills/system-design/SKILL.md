---
name: system-design
description: "Comprehensive guidance for designing scalable, reliable, and maintainable distributed systems including architecture patterns, scalability strategies, database selection, and production-grade system design. Use when designing scalable distributed systems, selecting architecture patterns, estimating capacity, or making technology stack decisions. Keywords: design scalable system, architecture diagram, CAP theorem, load balancer, database sharding, microservices design."
allowed-tools: Read,Glob,Grep
user-invocable: true
---


<!-- generated-by: claude-global-library/tools/project_sync -->
<!-- source: skills/system-design/SKILL.md -- edit the library, then re-run sync_project.py -->

# System Design Skill Instructions

### Skill Name
system-design

### Description
Provides comprehensive guidance for designing scalable, reliable, and maintainable distributed systems. Covers architecture patterns, scalability strategies, database selection, caching, messaging, API design, reliability, observability, security, and common system design exercises used in real-world engineering and interviews.

---

## Mathematical Foundations

### M1: Amdahl's Law (Scalability Ceiling)
```
Speedup(p) = 1 / (f + (1-f)/p)

Where:
  f = serial fraction (cannot be parallelized)
  p = number of processors/instances
  As p → ∞: Speedup_max = 1/f
```
- f=0.05 (5% serial) → max speedup = 20x regardless of how many servers added
- f=0.20 (20% serial) → max speedup = 5x
- f=0.01 (1% serial) → max speedup = 100x

Application: If a system has a 10% serial bottleneck (global locks, single-writer DB), maximum throughput gain is 10x no matter how many servers are added. Identify the serial bottleneck before scaling horizontally.

### M2: Little's Law (Queue and Pool Sizing)
```
L = λ × W

Where:
  L = average number of requests in system (queue + in-process)
  λ = average arrival rate (requests/second)
  W = average time a request spends in the system (seconds)
```
- 1000 RPS with 50ms avg latency → L = 1000 × 0.05 = 50 concurrent requests in-flight
- Thread pool sizing: pool_size = target_RPS × avg_latency_seconds
- Queue depth: queue_length = burst_RPS × tolerable_queue_wait_seconds

Application: Size connection pools, thread pools, and queue depths before load testing.

### M3: Availability Math
```
Availability = Uptime / (Uptime + Downtime)
```
| SLA       | Downtime/Year  | Downtime/Month | Downtime/Week  |
|-----------|----------------|----------------|----------------|
| 99%       | 3.65 days      | 7.30 hours     | 1.68 hours     |
| 99.9%     | 8.76 hours     | 43.8 minutes   | 10.1 minutes   |
| 99.99%    | 52.6 minutes   | 4.38 minutes   | 1.01 minutes   |
| 99.999%   | 5.26 minutes   | 26.3 seconds   | 6.05 seconds   |

Serial dependencies multiply unavailability:
```
Availability_total = A₁ × A₂ × ... × Aₙ
```
- Two 99.9% services in series: 0.999 × 0.999 = 99.80% (doubles downtime)

Parallel redundancy reduces unavailability:
```
Unavailability_parallel = (1 - A₁) × (1 - A₂)
```
- Two 99.9% services in parallel: (0.001)(0.001) = 0.0001% unavailability → 99.9999%

### M4: Quorum Math (Replication Consistency)
```
Strong consistency requires: W + R > N

Where:
  N = total replicas
  W = minimum write acknowledgments required
  R = minimum replicas to read from
```
| N | W | R | Consistency  | Fault Tolerance |
|---|---|---|--------------|-----------------|
| 3 | 2 | 2 | Strong       | 1 node failure  |
| 3 | 1 | 1 | Eventual     | 2 node failures |
| 5 | 3 | 3 | Strong       | 2 node failures |
| 5 | 1 | 5 | Strong reads | 4 node failures |

Fault tolerance formula: `floor((N-1)/2)` failures tolerated while maintaining quorum.

### M5: Back-of-Envelope Estimation Reference
```
Storage:  1 char=1B, 1 KB=10³B, 1 MB=10⁶B, 1 GB=10⁹B, 1 TB=10¹²B, 1 PB=10¹⁵B

Latency reference:
  L1 cache: ~1 ns      | RAM: ~100 ns
  L2 cache: ~4 ns      | SSD random: ~100 µs
  L3 cache: ~40 ns     | HDD seek: ~10 ms
                       | Network RTT: ~150 ms (cross-region)

Throughput reference:
  RAM: ~50 GB/s   | SSD seq: ~500 MB/s
  Gigabit NIC: ~125 MB/s

QPS estimation:
  Daily_avg_QPS  = DAU × requests_per_user / 86,400
  Peak_QPS       = Daily_avg_QPS × peak_multiplier (2–10×)

Storage estimation:
  Daily_storage  = writes_per_day × avg_object_size_bytes
  Total_storage  = Daily_storage × retention_days × replication_factor
```

### M6: Cache Hit Rate and Zipfian Distribution
Web content access follows Zipfian (power law) distribution with exponent s ≈ 1:
```
P(rank k) ∝ 1/k   →   top 20% of content serves ~80% of requests (Pareto principle)
```
Effective latency with cache:
```
Effective_latency = (hit_rate × cache_latency) + ((1 - hit_rate) × db_latency)
```
- 90% hit rate, cache=1ms, DB=50ms: `0.9×1 + 0.1×50 = 5.9ms` vs 50ms without cache
- Cache ROI: caching 20% of dataset achieves ~80% hit rate for typical workloads

Cache miss storm (thundering herd): `P(stampede) = e^(-λ×TTL_jitter)` — add random ±10% jitter to TTL to spread expiry.

### M7: Consistent Hashing Ring Math
```
Ring size: 2³² positions (0 to 4,294,967,295)

Virtual nodes (v) per server reduces load imbalance:
  Load std deviation ≈ 1 / √(v × num_servers)

  v=1:   σ ≈ 50% load deviation (unacceptable)
  v=100: σ ≈ 7% load deviation (acceptable)
  v=150: σ ≈ 5% load deviation (production standard)

Key redistribution on server add/remove:
  Keys_moved = total_keys / (num_servers + 1)
```
Consistent hashing moves only `1/(n+1)` fraction of keys on topology change, vs ~50% for modular hashing.

### M8: Tail Latency Amplification
p99 latency in sequential microservice chains:
```
P(at least one service hits p99) = 1 - (1 - p99_rate)^n
```
- 5 services in chain, each p99=10ms: `1 - (0.99)^5 = 4.9%` of requests hit p99 → effective chain p99 = p95 of single service

Fan-out (parallel calls) worst-case:
```
p99_fanout ≈ p(1 - (1-p99_rate)^n)  ≈ p99.999 of single service for n=100 parallel calls
```
This is why fan-out architectures must optimize for the single slowest call, not the average.

### M9: Rate Limiting Algorithms
Token bucket state equation:
```
tokens(t) = min(capacity, tokens(t-Δt) + refill_rate × Δt)
Allow request if tokens(t) ≥ cost; deduct cost after allowing.
```
Sliding window log memory cost:
```
memory = request_rate_per_window × window_seconds × bytes_per_log_entry
```
Fixed window allows 2× burst at window boundary; sliding window eliminates it:
```
fixed_window_burst  = 2 × rate_limit   (at boundary)
sliding_window_max  = rate_limit        (always)
```

### 10. SLO Error Budget and Burn Rate
```
Error_budget_fraction = 1 - SLO_target
Error_budget_minutes  = (1 - SLO) × minutes_in_period

Example: 99.9% SLO over 30 days:
  Error_budget = 0.001 × 30 × 24 × 60 = 43.2 minutes/month

Burn rate = actual_error_rate / (1 - SLO)
```
Alerting thresholds (Google SRE model):
| Window  | Burn Rate Threshold | Budget Consumed | Action      |
|---------|---------------------|-----------------|-------------|
| 1 hour  | > 14.4×             | 2% in 1h        | Page now    |
| 6 hours | > 6×                | 5% in 6h        | Page        |
| 3 days  | > 1×                | 10% in 3d       | Ticket      |

---

## System Design Fundamentals

### 1. Client-Server Architecture
- The foundational model where clients send requests and servers process and return responses
- Thin clients delegate processing to servers; thick clients handle more logic locally
- Stateless servers simplify horizontal scaling; stateful servers require sticky sessions or shared state
- Consider latency, bandwidth, and connection management between client and server tiers

### 2. Monolith vs Microservices
- Monolith: single deployable unit, simpler development and debugging, shared database, tight coupling
- Microservices: independently deployable services, team autonomy, polyglot technology, operational complexity
- Start with a monolith and extract services when clear domain boundaries emerge (monolith-first approach)
- Microservices introduce challenges: distributed transactions, service discovery, network latency, data consistency
- Consider modular monolith as a middle ground with clear module boundaries but single deployment

### 3. Layered Architecture
- Presentation layer handles UI and user interaction
- Application/business logic layer enforces domain rules and workflows
- Data access layer abstracts persistence and external data sources
- Each layer depends only on the layer directly below it, promoting separation of concerns
- Anti-corruption layers prevent leaking of external system models into core domain

### 4. Hexagonal Architecture (Ports and Adapters)
- Core domain logic is isolated from external concerns (databases, APIs, UI)
- Ports define interfaces the domain exposes or requires
- Adapters implement ports for specific technologies (REST adapter, database adapter)
- Enables easy testing by swapping real adapters with test doubles
- Promotes technology-agnostic domain code that survives infrastructure changes

### 5. Event-Driven Architecture
- Components communicate through events rather than direct calls
- Producers emit events without knowledge of consumers (loose coupling)
- Event brokers (Kafka, RabbitMQ) mediate event delivery between producers and consumers
- Supports real-time processing, event replay, and temporal decoupling
- Challenges include eventual consistency, event ordering, and debugging event flows

### 6. CQRS (Command Query Responsibility Segregation)
- Separates read models (queries) from write models (commands)
- Write side optimized for consistency and validation; read side optimized for query performance
- Read models can be denormalized projections updated asynchronously from events
- Often paired with event sourcing for full audit trail and temporal queries
- Adds complexity; use only when read and write patterns differ significantly

### 7. Domain-Driven Design (DDD)
- Bounded contexts define clear boundaries around domain models with explicit contracts
- Aggregates enforce consistency boundaries and transactional integrity
- Entities have identity; value objects are immutable and compared by attributes
- Ubiquitous language ensures developers and domain experts share terminology
- Context mapping defines relationships between bounded contexts (shared kernel, customer-supplier, anticorruption layer)

---

## Scalability

### 1. Horizontal vs Vertical Scaling
- Vertical scaling (scale up): add more CPU, RAM, or storage to a single machine; limited by hardware ceiling
- Horizontal scaling (scale out): add more machines to distribute load; requires stateless or shared-state design
- Horizontal scaling provides near-linear capacity growth and better fault tolerance
- Vertical scaling is simpler but creates a single point of failure and has physical limits
- Prefer horizontal scaling for production systems; use vertical scaling for databases when partitioning is complex

### 2. Auto-Scaling Strategies
- Reactive auto-scaling: scale based on real-time metrics (CPU, memory, request count, queue depth)
- Predictive auto-scaling: use historical patterns to pre-provision capacity before demand spikes
- Scheduled auto-scaling: add capacity at known peak times (business hours, marketing campaigns)
- Cool-down periods prevent thrashing (rapid scale up/down cycles)
- Define minimum and maximum instance counts to control cost and ensure baseline availability

### 3. Stateless Design
- Store session state externally (Redis, database) so any server can handle any request
- Stateless services are trivially horizontally scalable and replaceable
- Use tokens (JWT) to carry authentication state without server-side sessions
- Externalize configuration via environment variables or config services
- Avoid local file storage; use object storage (S3) or distributed file systems

### 4. Sharding Strategies
- Range-based sharding: partition data by key ranges (A-M, N-Z); risk of hotspots if distribution is uneven
- Hash-based sharding: apply hash function to shard key for uniform distribution; range queries become expensive
- Directory-based sharding: lookup service maps keys to shards; flexible but adds lookup latency
- Geographic sharding: partition data by region for data locality and compliance
- Choose shard key carefully; resharding is expensive and disruptive

### 5. Partitioning
- Horizontal partitioning (sharding): split rows across multiple databases or tables
- Vertical partitioning: split columns into separate tables (frequently vs rarely accessed data)
- Functional partitioning: separate data by business function (orders database, users database)
- Partition pruning optimizes queries by scanning only relevant partitions
- Cross-partition queries and joins are expensive; design data access patterns around partition boundaries

### 6. Consistent Hashing
- Maps both servers and data keys onto a hash ring for minimal redistribution on topology changes
- Adding or removing a server only reassigns keys from adjacent ring positions
- Virtual nodes improve balance by mapping each physical server to multiple ring positions
- Used by DynamoDB, Cassandra, and many distributed caching systems
- Reduces data movement during scaling events compared to modular hashing

### 7. Load Distribution
- Distribute load across services, databases, and caches to avoid bottlenecks
- Use read replicas to offload read-heavy workloads from primary databases
- Separate compute-intensive tasks into dedicated worker pools
- Queue-based load leveling absorbs traffic spikes by buffering requests
- Geographic distribution (multi-region, CDN) reduces latency for global users

---

## Load Balancing

### 1. L4 vs L7 Load Balancers
- L4 (transport layer): routes based on IP and TCP/UDP port; fast, no payload inspection
- L7 (application layer): routes based on HTTP headers, URL path, cookies; more flexible but higher latency
- L4 suitable for TCP-level load balancing (databases, non-HTTP protocols)
- L7 enables content-based routing, SSL termination, header manipulation, and request rewriting
- Many production setups combine L4 (global traffic distribution) with L7 (application routing)

### 2. Algorithms
- Round-robin: distributes requests sequentially; simple but ignores server capacity differences
- Weighted round-robin: assigns proportional traffic based on server capacity weights
- Least connections: routes to the server with fewest active connections; good for long-lived requests
- IP hash: hashes client IP to consistently route to the same server; useful for session affinity
- Least response time: routes to the server with fastest response; requires active monitoring
- Random: simple probabilistic distribution; surprisingly effective with large server pools

### 3. Health Checks
- Active health checks: load balancer periodically probes backend servers (HTTP GET, TCP connect)
- Passive health checks: monitor real traffic for errors and timeouts to detect unhealthy servers
- Configure check intervals, timeout thresholds, and failure/recovery count before marking server status
- Separate liveness checks (is the process running) from readiness checks (can it serve traffic)
- Drain connections gracefully before removing unhealthy servers from rotation

### 4. Sticky Sessions
- Bind a client to the same backend server for the duration of a session
- Implemented via cookies, URL parameters, or source IP affinity
- Reduces session consistency issues but limits load distribution effectiveness
- Server failure causes session loss unless sessions are replicated or externalized
- Prefer stateless design with external session stores over sticky sessions

### 5. Global Server Load Balancing (GSLB)
- Distributes traffic across geographically distributed data centers
- Uses DNS-based routing, anycast, or global load balancer appliances
- Factors include geographic proximity, data center health, and current load
- Supports active-active (all regions serve traffic) and active-passive (failover) configurations
- Latency-based routing sends users to the nearest healthy data center

### 6. DNS-Based Routing
- Route53, Cloudflare DNS, or similar services return different IPs based on routing policies
- Geolocation routing: return IP of nearest data center based on client location
- Weighted routing: distribute traffic percentages across endpoints for canary deployments
- Failover routing: return backup IP when primary health check fails
- TTL affects how quickly DNS changes propagate; lower TTL for faster failover, higher for caching

### 7. HAProxy, Nginx, AWS ALB
- HAProxy: high-performance TCP/HTTP load balancer, advanced ACLs, connection management
- Nginx: web server + reverse proxy + load balancer, strong HTTP/2 support, caching
- AWS ALB: managed L7 load balancer, native integration with ECS/EKS, path-based routing, WAF integration
- AWS NLB: managed L4 load balancer, ultra-low latency, static IPs, TLS passthrough
- Choose managed services for reduced operational burden; self-managed for maximum control

---

## Caching Strategies

### 1. Cache-Aside (Lazy Loading)
- Application checks cache first; on miss, reads from database and populates cache
- Only requested data is cached, avoiding unnecessary cache population
- Cache misses incur extra latency (cache check + database read + cache write)
- Stale data possible if database is updated without invalidating cache
- Most common caching pattern; simple to implement and understand

### 2. Write-Through
- Application writes to cache and database simultaneously (cache is always up-to-date)
- Eliminates stale data since every write updates the cache
- Higher write latency because both cache and database must acknowledge
- Cache may contain data that is never read (wasted memory)
- Best when read-after-write consistency is critical

### 3. Write-Behind (Write-Back)
- Application writes to cache; cache asynchronously flushes to database in batches
- Reduces write latency and database load through batching
- Risk of data loss if cache fails before flushing to database
- Requires reliable cache infrastructure with persistence or replication
- Effective for write-heavy workloads where slight data loss is acceptable

### 4. Read-Through
- Cache sits between application and database; cache handles fetching on miss
- Application only interacts with cache, simplifying application code
- Cache provider must support configurable data loading logic
- Similar to cache-aside but the cache itself manages population
- Well-suited for ORM-level or framework-level caching

### 5. Cache Invalidation Strategies
- Time-based (TTL): cache entries expire after a set duration; simple but allows brief staleness
- Event-based: invalidate cache when underlying data changes (database triggers, application events)
- Version-based: include version number in cache key; new version creates new cache entry
- Write invalidation: delete cache entry on write; next read repopulates from source
- Purge/flush: manually or programmatically clear specific keys or entire cache

### 6. TTL (Time-to-Live)
- Set appropriate TTL based on data volatility: short TTL for frequently changing data, long for static
- Jitter (randomized TTL variation) prevents cache stampede when many entries expire simultaneously
- Sliding TTL resets expiry on each access, keeping hot data cached longer
- Zero TTL or negative TTL forces immediate expiry (useful for debugging)
- Monitor cache hit rates to tune TTL values for optimal performance

### 7. Distributed Caching
- Spread cache across multiple nodes for scalability and fault tolerance (Redis Cluster, Memcached)
- Consistent hashing determines which node holds each cache key
- Replication across cache nodes provides redundancy and read scaling
- Network latency between application and cache nodes is a factor; co-locate when possible
- Partition tolerance: decide behavior when cache nodes become unreachable

### 8. CDN Caching
- Cache static assets (images, CSS, JS) and sometimes dynamic content at edge locations
- Reduces origin server load and improves latency for geographically distributed users
- Cache-Control headers (max-age, s-maxage, no-cache, no-store) control CDN behavior
- Cache purging/invalidation APIs for updating stale CDN content
- Popular CDNs: CloudFront, Cloudflare, Akamai, Fastly

### 9. Multi-Level Caching
- L1 cache: in-process memory (fastest, limited size, per-instance)
- L2 cache: distributed cache like Redis (shared across instances, network hop)
- L3 cache: CDN edge cache (closest to user, limited control)
- Waterfall: check L1, then L2, then L3, then origin; populate lower levels on miss
- Balance memory usage, consistency, and latency across levels

### 10. Cache Stampede Prevention
- Stampede occurs when many requests simultaneously miss cache for the same key
- Locking: only one request fetches from source; others wait for cache population
- Probabilistic early recomputation: refresh cache before expiry based on probability
- Request coalescing: group identical concurrent requests into a single origin fetch
- Pre-warming: populate cache before anticipated high-traffic events

---

## Database Selection

### 1. SQL vs NoSQL Decision Matrix
- SQL when: strong consistency needed, complex joins and aggregations, well-defined schema, ACID transactions
- NoSQL when: flexible schema, horizontal scalability, high write throughput, denormalized data models
- Document stores (MongoDB, CouchDB): semi-structured data, nested objects, schema evolution
- Key-value stores (Redis, DynamoDB): simple lookups, session storage, caching
- Column-family stores (Cassandra, HBase): wide rows, time-series data, write-heavy workloads
- Graph databases (Neo4j, Neptune): highly connected data, relationship traversal, social networks

### 2. CAP Theorem
- Consistency: every read returns the most recent write or an error
- Availability: every request receives a non-error response (may not be the most recent data)
- Partition tolerance: system continues operating despite network partitions between nodes
- In a distributed system, you must choose two of three during a partition event
- CP systems (MongoDB, HBase): sacrifice availability for consistency during partitions
- AP systems (Cassandra, DynamoDB): sacrifice consistency for availability during partitions

### 3. ACID vs BASE
- ACID: Atomicity, Consistency, Isolation, Durability; traditional relational database guarantees
- BASE: Basically Available, Soft state, Eventually consistent; NoSQL and distributed system model
- ACID ensures correctness at the cost of scalability and availability
- BASE prioritizes availability and performance, accepting temporary inconsistency
- Many modern systems offer tunable consistency (e.g., DynamoDB strong vs eventual consistency)

### 4. Polyglot Persistence
- Use different database technologies for different data needs within the same system
- User profiles in a document store, financial transactions in a relational database, analytics in a columnar store
- Increases operational complexity (multiple database technologies to manage)
- Requires clear data ownership boundaries and synchronization strategies
- Align database choice with each service's access patterns and consistency requirements

### 5. Read Replicas
- Create read-only copies of the primary database to offload read queries
- Replication lag introduces eventual consistency between primary and replicas
- Route read-after-write queries to primary to ensure consistency when needed
- Scale read capacity horizontally by adding more replicas
- Replicas can also serve as failover targets for high availability

### 6. Write-Ahead Logging (WAL)
- Log every change before applying it to the database for crash recovery
- Enables point-in-time recovery by replaying WAL from a base backup
- WAL shipping to replicas provides replication mechanism (PostgreSQL streaming replication)
- Change Data Capture (CDC) systems read WAL to stream database changes to other systems
- WAL size and retention affect recovery time and storage requirements

### 7. Database per Service
- Each microservice owns its database; no direct database sharing between services
- Ensures loose coupling and independent deployability of services
- Cross-service queries require API calls or asynchronous data replication
- Distributed transactions become complex; use saga pattern or eventual consistency
- Simplifies schema evolution since each service controls its own schema

### 8. Shared Database
- Multiple services share a single database; simpler but tightly coupled
- Schema changes risk breaking multiple services simultaneously
- Joins across service data are easy but create hidden dependencies
- Acceptable for small systems or services that are always deployed together
- Anti-pattern for microservices; consider as a stepping stone during migration from monolith

---

## Message Queues & Event Streaming

### 1. Synchronous vs Asynchronous Communication
- Synchronous (HTTP, gRPC): caller blocks until response; simple, but creates temporal coupling
- Asynchronous (message queues, events): caller sends message and continues; decoupled, resilient
- Async improves resilience: if consumer is down, messages queue up for later processing
- Async adds complexity: message ordering, delivery guarantees, error handling
- Use sync for queries needing immediate response; async for commands and event notifications

### 2. Message Brokers
- Central component that receives, stores, and delivers messages between producers and consumers
- Kafka: distributed log, high throughput, persistent, ordered within partitions
- RabbitMQ: traditional message broker, flexible routing, priority queues, multiple protocols
- AWS SQS: managed queue service, no infrastructure management, automatic scaling
- Choose based on throughput needs, ordering requirements, and operational capacity

### 3. Pub/Sub Pattern
- Publishers broadcast messages to topics without knowledge of subscribers
- Subscribers receive all messages on subscribed topics (fan-out)
- Decouples producers from consumers; new consumers can be added without modifying producers
- Use for event notification, real-time updates, and broadcasting state changes
- Consider message filtering to reduce unnecessary message delivery to subscribers

### 4. Event Sourcing
- Store all state changes as an immutable sequence of events rather than current state
- Rebuild current state by replaying events from the beginning (or from a snapshot)
- Provides complete audit trail and temporal query capability
- Enables event replay for debugging, analytics, or rebuilding read models
- Requires snapshots for performance when event history grows large

### 5. Saga Pattern
- Manage distributed transactions across multiple services using a sequence of local transactions
- Orchestration: central coordinator directs each step and handles compensation on failure
- Choreography: each service listens for events and triggers the next step; no central coordinator
- Compensating transactions undo completed steps when a later step fails
- Design for idempotency so retries and compensations are safe to repeat

### 6. Outbox Pattern
- Write events to an outbox table in the same database transaction as the business data
- A separate process reads the outbox table and publishes events to the message broker
- Guarantees atomicity between data change and event publication (no dual-write problem)
- Outbox reader uses polling or CDC (Change Data Capture) to detect new events
- Ensures at-least-once delivery; consumers must handle duplicates

### 7. Idempotency
- An operation is idempotent if applying it multiple times produces the same result as applying it once
- Essential for message consumers that may receive duplicate messages
- Use idempotency keys (unique request IDs) to detect and skip duplicate processing
- Design database operations to be naturally idempotent (upserts, conditional updates)
- Store processed message IDs to prevent reprocessing

### 8. Exactly-Once Delivery
- True exactly-once delivery is extremely difficult in distributed systems
- Most systems achieve effectively-once by combining at-least-once delivery with idempotent consumers
- Kafka supports exactly-once semantics within its ecosystem using transactional producers and consumers
- Deduplication at the consumer level is the most practical approach
- Understand the tradeoffs: exactly-once adds latency and complexity

---

## API Design

### 1. REST Principles
- Resources identified by URIs; use nouns, not verbs (GET /users, not GET /getUsers)
- HTTP methods convey intent: GET (read), POST (create), PUT (replace), PATCH (update), DELETE (remove)
- Stateless: each request contains all information needed; no server-side session dependency
- HATEOAS: responses include links to related resources and available actions
- Content negotiation via Accept and Content-Type headers for format flexibility

### 2. GraphQL Considerations
- Single endpoint with flexible queries; clients request exactly the fields they need
- Eliminates over-fetching (too many fields) and under-fetching (too few fields requiring multiple calls)
- Schema-first design with strong typing provides self-documenting API
- N+1 query problem requires DataLoader or batching; monitor query complexity
- Best for complex, interconnected data; overkill for simple CRUD APIs

### 3. gRPC for Internal Services
- Binary protocol (Protocol Buffers) for high-performance service-to-service communication
- Strongly typed contracts with code generation for multiple languages
- Supports unary, server streaming, client streaming, and bidirectional streaming RPCs
- Built-in deadlines, cancellation, and flow control
- Not browser-friendly without gRPC-Web proxy; best for internal backend services

### 4. API Gateway Pattern
- Single entry point for all client requests; routes to appropriate backend services
- Handles cross-cutting concerns: authentication, rate limiting, logging, CORS
- Request aggregation: combine multiple backend calls into a single client response
- Protocol translation: expose REST to clients while using gRPC internally
- Popular options: Kong, AWS API Gateway, Apigee, custom Nginx/Envoy-based gateways

### 5. Rate Limiting
- Protect services from abuse and ensure fair usage across clients
- Token bucket: steady rate with burst capacity; most flexible algorithm
- Sliding window: smooth rate enforcement without burst spikes
- Return 429 Too Many Requests with Retry-After header when limit exceeded
- Apply rate limits per API key, per user, per IP, or per endpoint as appropriate

### 6. Circuit Breaker
- Prevent cascading failures by stopping calls to a failing downstream service
- States: closed (normal), open (failing, fast-fail all requests), half-open (testing recovery)
- Track failure rate and response time; trip circuit when thresholds exceeded
- Return fallback responses or graceful degradation when circuit is open
- Libraries: Resilience4j, Hystrix (maintenance mode), Polly (.NET)

### 7. Bulkhead Pattern
- Isolate resources for different service calls to prevent one failure from consuming all resources
- Thread pool isolation: dedicated thread pools per downstream service
- Semaphore isolation: limit concurrent calls to each downstream service
- Connection pool isolation: separate connection pools for different backends
- Prevents a slow or failing dependency from exhausting shared resources

### 8. API Versioning
- URL versioning (/v1/users, /v2/users): simple, explicit, widely used
- Header versioning (Accept: application/vnd.api+json;version=2): clean URLs but less discoverable
- Query parameter versioning (/users?version=2): easy to implement but pollutes URL
- Support old versions for a deprecation period with clear sunset dates
- Minimize breaking changes; prefer additive changes (new fields, new endpoints)

---

## Data Storage Patterns

### 1. Blob Storage
- Store unstructured binary data (images, videos, documents, backups) in object storage
- AWS S3, Azure Blob Storage, Google Cloud Storage provide virtually unlimited capacity
- Organize with key prefixes (virtual directories); not a real file system
- Use presigned URLs for secure, time-limited direct client access
- Lifecycle policies automate transition to cheaper storage tiers and deletion

### 2. Data Lakes
- Store raw data in original format (structured, semi-structured, unstructured) at scale
- Schema-on-read: apply schema when querying, not when storing
- Built on object storage (S3, ADLS) with query engines (Athena, Presto, Spark)
- Risk of becoming a data swamp without governance, cataloging, and quality controls
- Delta Lake, Iceberg, and Hudi add ACID transactions and schema evolution to data lakes

### 3. Data Warehouses
- Optimized for analytical queries on structured, cleaned, and transformed data
- Schema-on-write: data is validated and structured before loading
- Columnar storage (Redshift, BigQuery, Snowflake) for fast aggregation queries
- ETL/ELT pipelines load data from operational systems into the warehouse
- Star schema and snowflake schema organize fact and dimension tables for analytics

### 4. Time-Series Databases
- Optimized for time-stamped data: metrics, IoT sensor data, financial ticks, logs
- InfluxDB, TimescaleDB, Prometheus for metrics; QuestDB for high-performance ingestion
- Automatic data downsampling and retention policies manage storage growth
- Efficient range queries and aggregations over time windows
- Often combined with visualization tools (Grafana) for monitoring dashboards

### 5. Search Engines
- Full-text search with relevance ranking, faceted search, and autocomplete
- Elasticsearch/OpenSearch: distributed, REST API, powerful query DSL, near-real-time indexing
- Inverted index structure enables fast text search across large document collections
- Use for product search, log analysis, and any use case requiring flexible text queries
- Not a primary database; sync data from source of truth via CDC or periodic indexing

### 6. Graph Databases
- Model and query highly connected data with nodes, edges, and properties
- Neo4j (Cypher), Amazon Neptune (Gremlin, SPARQL), JanusGraph for different scale needs
- Efficient for traversals: social networks, recommendation engines, fraud detection, knowledge graphs
- Relationship-centric queries that would require complex joins in SQL are natural in graph databases
- Consider when the relationships between entities are as important as the entities themselves

### 7. Hot/Warm/Cold Storage Tiers
- Hot: frequently accessed data on fast storage (SSD, in-memory); highest cost per GB
- Warm: less frequently accessed data on standard storage; moderate cost and latency
- Cold: rarely accessed data on archival storage (S3 Glacier, tape); lowest cost, high retrieval latency
- Automate data movement between tiers based on access patterns and age
- Design retrieval workflows for cold data that account for minutes-to-hours access time

---

## Reliability & Fault Tolerance

### 1. Redundancy
- Eliminate single points of failure by duplicating critical components
- Active-active: all instances serve traffic simultaneously (higher utilization)
- Active-passive: standby instances take over when primary fails (simpler but wastes resources)
- Apply redundancy at every layer: load balancers, application servers, databases, network paths
- Multi-region redundancy protects against regional outages and natural disasters

### 2. Replication
- Synchronous replication: primary waits for replica acknowledgment before confirming write (strong consistency, higher latency)
- Asynchronous replication: primary confirms write immediately, replica updates later (lower latency, risk of data loss)
- Semi-synchronous: primary waits for at least one replica acknowledgment (balanced approach)
- Multi-master replication allows writes to any node but requires conflict resolution
- Replication factor determines how many copies of data exist (typically 3 for production)

### 3. Failover Strategies
- Automatic failover: monitoring detects failure and promotes standby without human intervention
- Manual failover: operators trigger failover after confirming the issue (safer but slower)
- DNS failover: update DNS records to point to healthy instances (limited by TTL propagation)
- Database failover: promote read replica to primary using automated tools (Patroni, RDS Multi-AZ)
- Test failover procedures regularly through drills and chaos engineering

### 4. Circuit Breaker Pattern
- Detect repeated failures to a dependency and stop calling it temporarily
- Prevents resource exhaustion (thread pools, connections) from waiting on failing services
- Open state returns error immediately without attempting the call
- Half-open state periodically allows test requests to check if service has recovered
- Configure failure thresholds, timeout durations, and recovery probe intervals

### 5. Retry with Exponential Backoff
- Retry failed operations with increasing delays: 1s, 2s, 4s, 8s, etc.
- Add jitter (random variation) to prevent thundering herd when many clients retry simultaneously
- Set maximum retry count and total timeout to avoid infinite retry loops
- Only retry on transient failures (network timeout, 503); do not retry on 400/401/403
- Combine with circuit breaker to stop retrying when a service is confirmed down

### 6. Bulkhead Isolation
- Partition system resources so failure in one area does not cascade to others
- Separate thread pools for different downstream dependencies
- Dedicate infrastructure (separate clusters, databases) for critical vs non-critical workloads
- Container resource limits (CPU, memory) act as bulkheads in containerized environments
- Prevent noisy neighbor problems in multi-tenant systems

### 7. Graceful Degradation
- Continue serving reduced functionality when components fail rather than total outage
- Serve cached or stale data when the database is unreachable
- Disable non-essential features (recommendations, analytics) to preserve core functionality
- Show informative error messages and reduced UI rather than error pages
- Feature flags enable rapid disabling of degraded features in production

### 8. Chaos Engineering
- Intentionally inject failures to test system resilience (network delays, server crashes, disk failures)
- Netflix Chaos Monkey pioneered the practice of randomly terminating production instances
- Start in non-production environments; graduate to production with safety controls
- Game days: scheduled events where teams practice responding to simulated incidents
- Tools: Chaos Monkey, Gremlin, Litmus, AWS Fault Injection Simulator

---

## Observability

### 1. Logging Strategy
- Structured logging (JSON format) enables machine parsing and querying
- Log levels: DEBUG, INFO, WARN, ERROR, FATAL; configure appropriately per environment
- Include correlation IDs to trace requests across multiple services
- Centralize logs with ELK Stack (Elasticsearch, Logstash, Kibana) or Loki + Grafana
- Avoid logging sensitive data (passwords, tokens, PII); use redaction

### 2. Metrics Collection
- Four golden signals: latency, traffic, errors, saturation (Google SRE)
- RED method for request-driven services: Rate, Errors, Duration
- USE method for resources: Utilization, Saturation, Errors
- Time-series metrics stored in Prometheus, Datadog, CloudWatch, or InfluxDB
- Custom business metrics (orders per minute, signup conversion) alongside technical metrics

### 3. Distributed Tracing
- Track requests across service boundaries with trace IDs and span hierarchy
- OpenTelemetry provides vendor-neutral instrumentation for traces, metrics, and logs
- Jaeger, Zipkin, AWS X-Ray, Datadog APM for trace visualization and analysis
- Sample traces (1-10%) in high-traffic systems to manage overhead and storage
- Identify latency bottlenecks, error sources, and dependency chains from traces

### 4. Alerting
- Alert on symptoms (high error rate, high latency) not causes (high CPU)
- Reduce alert fatigue: only alert on actionable conditions that require human intervention
- Multi-level severity: page for critical, ticket for warning, dashboard for informational
- Alert routing: direct alerts to the right team based on service ownership
- Include runbooks in alert notifications with steps to diagnose and resolve

### 5. SLIs, SLOs, and SLAs
- SLI (Service Level Indicator): measurable metric (e.g., 99.5% of requests < 200ms)
- SLO (Service Level Objective): target value for an SLI (e.g., 99.9% availability per month)
- SLA (Service Level Agreement): contractual commitment with consequences for missing SLOs
- Error budget: allowed downtime or error rate; when exhausted, prioritize reliability over features
- Track SLI compliance on dashboards and alert when error budget is being consumed too quickly

### 6. Dashboards
- Overview dashboards for system-wide health at a glance
- Service-specific dashboards with detailed metrics per component
- Business dashboards for non-technical stakeholders (revenue, user signups, feature usage)
- Use consistent layouts and color coding across dashboards
- Grafana, Datadog, Kibana, and CloudWatch are popular dashboard tools

### 7. Incident Response
- Define incident severity levels (SEV1 through SEV4) with clear criteria
- On-call rotation with escalation paths and primary/secondary responders
- War room or incident channel for real-time coordination during major incidents
- Document timeline, actions taken, and resolution during the incident
- Post-incident review within 48 hours while details are fresh

### 8. Postmortem Culture
- Blameless postmortems focus on system improvements, not individual fault
- Document: what happened, timeline, root cause, impact, remediation actions
- Share postmortems broadly to spread learnings across the organization
- Track action items to completion; assign owners and deadlines
- Patterns across postmortems reveal systemic issues needing architectural changes

---

## Security Architecture

### 1. Zero Trust
- Never trust, always verify; authenticate and authorize every request regardless of network location
- No implicit trust based on network perimeter (VPN, corporate network)
- Verify identity, device health, and context for every access request
- Micro-segmentation: fine-grained network policies between services
- Continuous verification: re-evaluate trust throughout the session, not just at login

### 2. Authentication/Authorization Separation
- Authentication (AuthN): verify identity (who are you?)
- Authorization (AuthZ): verify permissions (what can you do?)
- Keep authentication centralized (identity provider) and authorization distributed (per service)
- Use standard protocols: OAuth 2.0 for authorization, OIDC for authentication
- Token-based systems carry identity and permissions in JWT claims

### 3. Secrets Management
- Never hardcode secrets in source code, configuration files, or environment variables in plain text
- Use dedicated secrets managers: HashiCorp Vault, AWS Secrets Manager, Azure Key Vault
- Rotate secrets automatically on a regular schedule
- Audit secret access and use least-privilege access policies
- Inject secrets at runtime via sidecar, init container, or SDK

### 4. Encryption at Rest and in Transit
- TLS 1.2+ for all network communication (in transit); enforce HTTPS everywhere
- Encrypt sensitive data at rest using AES-256 or equivalent
- Database-level encryption (Transparent Data Encryption) and column-level encryption for sensitive fields
- Manage encryption keys separately from encrypted data (KMS)
- Certificate management: automate renewal with Let's Encrypt or ACM

### 5. Network Segmentation
- Isolate workloads in separate network segments (VPCs, subnets, security groups)
- DMZ for public-facing services; private subnets for databases and internal services
- Network policies (Kubernetes NetworkPolicy, security groups) restrict traffic flow
- Service mesh (Istio, Linkerd) provides mTLS and fine-grained traffic control
- Limit blast radius of a breach by containing it within a network segment

### 6. DDoS Protection
- Absorb volumetric attacks with CDN and edge network capacity (CloudFront, Cloudflare)
- Rate limiting at load balancer and application levels
- AWS Shield, Azure DDoS Protection for cloud-native mitigation
- Geo-blocking and IP reputation filtering for known malicious sources
- Auto-scaling helps absorb legitimate traffic spikes but is not a DDoS solution alone

### 7. WAF (Web Application Firewall)
- Inspect HTTP requests and block common attacks (SQL injection, XSS, CSRF)
- Rule sets: OWASP Core Rule Set provides baseline protection
- Custom rules for application-specific threats and business logic abuse
- Rate limiting and bot detection at the WAF layer
- Deploy WAF in front of API gateway or load balancer; AWS WAF, Cloudflare WAF, ModSecurity

---

## Common System Designs

### 1. URL Shortener
- Key components: URL mapping (hash function or counter), redirection service, analytics
- Base62 encoding of auto-increment ID or random hash for short codes
- 301 (permanent) vs 302 (temporary) redirects affect caching and analytics
- Cache popular mappings in Redis; store full mapping in database
- Consider: custom aliases, expiration, abuse prevention, rate limiting

### 2. Chat System
- WebSocket connections for real-time bidirectional communication
- Message storage: write-optimized database with conversation partitioning
- Presence service tracks online/offline status with heartbeats
- Push notifications for offline users via APNs/FCM
- Consider: group chats, read receipts, media attachments, message search, end-to-end encryption

### 3. Notification Service
- Multi-channel: push notifications, email, SMS, in-app notifications
- Priority queues: urgent notifications processed before digest/batch notifications
- User preferences: per-channel opt-in/opt-out, quiet hours, frequency caps
- Template engine for consistent formatting across channels
- Delivery tracking and retry logic per channel

### 4. Rate Limiter
- Centralized rate limiting service using Redis for distributed state
- Algorithms: token bucket (allows burst), sliding window (smooth), fixed window (simple)
- Return rate limit headers: X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Reset
- Separate limits per user, per API key, per endpoint, and global limits
- Consider: distributed rate limiting consistency, race conditions, graceful throttling

### 5. Web Crawler
- URL frontier: priority queue of URLs to crawl with politeness policies (robots.txt, crawl delay)
- DNS resolver cache and connection pooling for efficiency
- Deduplication: URL normalization and content fingerprinting to avoid re-crawling
- Distributed architecture with multiple crawler workers coordinated via message queue
- Consider: depth limits, domain-level rate limiting, content extraction, incremental re-crawling

### 6. News Feed
- Fan-out on write (push model): precompute feed for each user when content is created
- Fan-out on read (pull model): assemble feed at read time from followed users' posts
- Hybrid: push for users with few followers, pull for celebrities with millions of followers
- Ranking and relevance scoring to order feed items
- Consider: real-time updates, pagination, media content delivery, content moderation

### 7. Video Streaming
- Content ingestion: upload, transcode to multiple resolutions and formats (HLS, DASH)
- CDN distribution for low-latency delivery close to users
- Adaptive bitrate streaming adjusts quality based on network conditions
- Thumbnail generation, metadata extraction, and content indexing
- Consider: live streaming, DRM, subtitle support, recommendation engine, view analytics

### 8. File Storage (Cloud Drive)
- Block-level deduplication and chunked uploads for efficient storage and transfer
- Metadata service tracks file hierarchy, permissions, and version history
- Sync engine detects local changes and resolves conflicts
- Presigned URLs for direct upload/download to object storage
- Consider: sharing and permissions, real-time collaboration, offline access, quota management

### 9. Search Autocomplete
- Trie (prefix tree) data structure for fast prefix matching
- Precomputed top suggestions based on search frequency and recency
- Personalized suggestions from user search history
- Update suggestion data asynchronously from search logs
- Consider: typo tolerance, multi-language support, trending queries, offensive content filtering

### 10. Payment System
- Idempotency keys prevent duplicate charges from retries
- Payment state machine: pending, authorized, captured, settled, refunded, failed
- PCI DSS compliance: tokenize card data, never store raw card numbers
- Reconciliation service matches internal records with payment provider statements
- Consider: multiple payment methods, multi-currency, fraud detection, dispute handling, ledger system

---

## Anti-Patterns to Avoid

- Ignoring Amdahl's ceiling: adding more servers to cut latency dominated by a serial bottleneck (global lock, single-writer DB) — throughput plateaus at 1/f regardless of horizontal scale (see M1)
- Shared database "microservices": services stay independently deployable in name only while reading/writing the same tables — a schema change in one service silently breaks another (see Database Selection #7/#8)
- Sticky sessions over externalized state: binding a client to one server for session data reintroduces a single point of failure and blocks even load distribution, defeating the point of horizontal scaling (see Load Balancing #4, Scalability #3)
- Uniform TTL cache stampede: caching many keys with the same expiry causes them to miss simultaneously under load, sending a correlated burst at the database instead of a smooth trickle (see M6, Caching Strategies #10)
- Read-after-write on a lagging replica: routing a read to a replica immediately after the user's own write returns stale data because replication lag has not caught up yet (see Database Selection #5)
- Dual write without an outbox: writing to the database and publishing the corresponding event as two separate operations leaves them inconsistent if the process crashes in between (see Message Queues & Event Streaming #6)
- Unbounded fan-out: calling many downstream services in parallel for one request pushes effective latency toward the slowest service's p99.999, even when every individual service meets its own SLO (see M8)
- Quorum misconfigured below W+R>N: picking W and R that do not satisfy the quorum inequality (e.g., N=3, W=1, R=1) silently downgrades an assumed-strongly-consistent system to eventual consistency (see M4)
- Resharding after picking a hotspot-prone key: choosing a shard key by convenience (e.g., signup date) instead of access-pattern uniformity forces an expensive full resharding once one shard becomes a hotspot (see Scalability #4)

---

## Response Rules

- Always start with requirements clarification: functional and non-functional requirements
- Provide back-of-the-envelope estimation for storage, bandwidth, and QPS
- Design top-down: high-level architecture first, then deep dive into critical components
- Justify every technology choice with tradeoffs and alternatives considered
- Address scalability, reliability, and performance at every layer
- Include data flow diagrams and component interaction descriptions
- Discuss failure modes and how the system handles each

---

## What Not to Do

- Do not skip requirements gathering and jump straight to architecture
- Do not choose technologies without explaining tradeoffs
- Do not ignore non-functional requirements (latency, throughput, availability targets)
- Do not design without considering failure scenarios and recovery
- Do not overlook data consistency requirements across distributed components
- Do not present a single solution without discussing alternatives
- Do not forget about operational concerns (deployment, monitoring, debugging)

---

## Output Expectations

- Provide clear component diagrams described in text (listing components and their interactions)
- Include capacity estimation with calculations shown
- List technology choices with reasoning and alternatives
- Describe data models and access patterns for critical components
- Detail request flow for key operations step by step
- Address scalability bottlenecks and how to handle growth
- Include monitoring, alerting, and operational considerations

---

## Skill Scope

In scope:
- Distributed system architecture and design patterns
- Scalability strategies and capacity planning
- Database selection and data modeling for distributed systems
- Caching, messaging, and communication patterns
- API design principles and gateway patterns
- Reliability, fault tolerance, and disaster recovery
- Observability, monitoring, and incident response
- Security architecture for distributed systems
- Common system design exercises and walkthroughs

Out of scope:
- Implementation code for specific programming languages
- Specific cloud provider console walkthroughs
- Project management and team organization
- Cost optimization and billing analysis
- Detailed networking protocols (TCP/IP stack internals)
- Hardware architecture and data center physical design

---

## Version
2.0.1 — Added "Anti-Patterns to Avoid" section covering Amdahl ceiling, shared-database microservices, sticky sessions, cache stampede, stale replica reads, dual-write without outbox, unbounded fan-out, quorum misconfiguration, and hotspot-prone shard keys.

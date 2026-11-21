# 00 · Fundamentals & Cross-Cutting Concerns

Notes that apply to every design below. They come from `D.drawio`, `2026Q3Notes.drawio` and the shared notes in `All.drawio`, with extra items added to finish each list.

---

## 1. Interview Framework (order of attack)
1. **Clarify requirements**: functional, non-functional, out of scope.
2. **Estimate scale**: DAU, QPS (peak = 2–5× avg), R/W ratio, storage/yr, bandwidth.
3. **API design**: endpoints, request/response, idempotency, pagination.
4. **Data model**: entities, keys, access patterns → pick storage.
5. **High-level design**: client → LB → gateway → services → cache → DB → async.
6. **Deep dives**: the 2–3 hardest parts (consistency, hot keys, fan-out, ordering).
7. **Bottlenecks & failure modes**: SPOFs, retries, back-pressure, DR.
8. **Wrap-up**: monitoring, security, cost, future work.

## 2. Non-Functional Checklist
- Availability (SLA: 99.9% ≈ 8.7 h/yr, 99.99% ≈ 52 min/yr)
- Latency (p50 / p99 targets)
- Durability (replication factor, backups)
- Consistency (strong vs eventual, read-your-writes)
- Scalability (horizontal, stateless services)
- Security (AuthN/AuthZ, TLS, encryption at rest)
- Observability (logs, metrics, traces, alerts)
- Cost

## 3. SQL vs NoSQL
- **SQL (RDBMS)**
  1. Consistent, structured schema
  2. Relational: joins, foreign keys
  3. ACID transactions
  4. Scales vertically first; sharding is manual and painful
  5. Use for money, bookings, inventory, anything with strong invariants
- **NoSQL**
  1. Flexible or no schema (unstructured / semi-structured)
  2. Scales horizontally (partitioning is built in)
  3. Usually BASE / eventual consistency (tunable in Cassandra/Dynamo)
  4. Data is modeled per query and denormalized
  5. Types:
     - Key-Value: Redis, DynamoDB
     - Wide-column: Cassandra, HBase
     - Document: MongoDB
     - Graph: Neo4j, Neptune
     - Time-series: InfluxDB, TimescaleDB
     - Search: Elasticsearch / OpenSearch

## 4. REST vs GraphQL vs gRPC
- **REST**: resource-oriented, cacheable (HTTP caching), simple; prone to over- and under-fetching.
- **GraphQL**: more flexible because the client fetches exactly the fields it needs (less over- and under-fetching). Harder to cache, N+1 resolver risk.
- **gRPC**: binary (protobuf), HTTP/2, streaming, fast service-to-service calls; less browser-friendly.

## 5. API Design Checklist
- Resource naming: nouns and plural (`/v1/users/{id}/orders`)
- Versioning: `/v1/` in the path or a header
- Idempotency: `Idempotency-Key` header on POST that creates resources or payments
- Pagination: cursor-based (`?cursor=…&limit=50`) over offset for large or changing sets
- Status codes: 200/201/202/204, 400/401/403/404/409/429, 500/503
- Rate limit headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After`
- Auth: OAuth2 / JWT bearer tokens; scopes
- Long operations: return `202 Accepted` + status URL, or a webhook

## 6. Retry Strategy
1. **Retry** only transient errors (timeouts, 5xx, 429), never 4xx validation errors.
2. **Exponential backoff**: `delay = base * 2^attempt`, capped at a max.
3. **Jitter**: randomize the delay (full jitter: `rand(0, delay)`) so retries don't arrive in synchronized waves (thundering herd).
4. **Max attempts**, then a dead-letter queue or a surfaced error.
5. **Idempotency**: a retry must be safe; use an idempotency key / dedup table.
6. **Circuit breaker**: stop calling a failing dependency (closed → open → half-open).
7. **Retry budget**: limit retries to about 10% of traffic to avoid retry storms.

## 7. Common Building Blocks
- **Load Balancer**: L4 vs L7; round-robin, least-connections, consistent hashing
- **API Gateway**: request validation, AuthN, AuthZ, TLS termination, encryption, rate limiting, auditing, routing
- **CDN**: static and media content at the edge; push vs pull CDN
- **Cache**: local (Caffeine) vs distributed (Redis); see [03_Cache](03_Cache.md)
- **Message Queue**: decoupling, buffering, async; see [01_Notification_MessageQueue](01_Notification_MessageQueue.md)
- **Blob storage**: S3; pre-signed URLs for direct client upload and download
- **Service discovery / config**: ZooKeeper, etcd, Consul, K8s
- **Unique ID generation**: Snowflake, ticket server, UUID; see [05_TinyUrl_PasteBin](05_TinyUrl_PasteBin.md)

## 8. Consistency & Distributed Concepts
- CAP: under a partition, choose C or A. PACELC: else choose Latency or Consistency.
- Quorum: `R + W > N` gives strong reads
- Consensus: Raft / Paxos (leader election, replicated log)
- Distributed lock: Redis (SET NX PX / Redlock), ZooKeeper ephemeral nodes; always add a TTL and fencing tokens
- 2PC vs Saga; see [12_TicketBooking](12_TicketBooking.md)
- Delivery semantics: at-most-once, at-least-once, effectively-once (= retry + idempotent processing)
- Consistent hashing with virtual nodes

## 9. Security Checklist
1. TLS everywhere; mTLS service-to-service
2. AuthN (OAuth2/OIDC, MFA), AuthZ (RBAC/ABAC)
3. Encryption at rest (KMS-managed keys)
4. Firewall / security groups / WAF
5. Rate limiting and bot protection
6. Input validation; prevent injection
7. Audit logging
8. Secrets management (Vault, KMS)

## 10. Observability Checklist
- Metrics: RED (Rate, Errors, Duration) and USE (Utilization, Saturation, Errors)
- Structured logs with correlation IDs
- Distributed tracing (OpenTelemetry)
- Alerts on SLO burn rate
- Dashboards per service

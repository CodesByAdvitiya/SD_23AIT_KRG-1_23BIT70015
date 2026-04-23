# Scaling Strategy

## System: E-Commerce Platform

---

## 1. Load Balancing

### Strategy
- **Layer 7 Load Balancer** (NGINX or AWS ALB) sits in front of each microservice tier.
- Traffic is distributed using **round-robin with health checks** — instances that fail liveness probes are removed from the pool within 10 seconds.
- **No sticky sessions** — all services are stateless, so any instance can handle any request. Session data lives in Redis, not in the service process.

### Setup
- API Gateway (Kong) handles client-facing load balancing and routes to the correct downstream service.
- Each microservice (Order, Product, Payment, etc.) has its own internal load balancer.
- **Weighted routing** is used for canary deployments — e.g., 5% of traffic routed to a new service version for testing before full rollout.

### Health Checks
- Kubernetes **liveness probes** restart unresponsive pods automatically.
- Kubernetes **readiness probes** prevent traffic from being sent to pods that are still starting up.

---

## 2. Caching Strategy

### Redis Cluster Setup
- **6-node Redis Cluster** — 3 master shards + 3 replicas, using consistent hash slots.
- Automatic failover: if a master node goes down, its replica is promoted within seconds.

### Cache Layers and Patterns

| Cache | Key Pattern | TTL | Pattern |
|---|---|---|---|
| Product metadata | `product:{id}:meta` | 5 min | Cache-aside |
| Product variant stock | `product:variant:{id}:stock` | No TTL (managed) | Write-through |
| User cart | `cart:{user_id}` | 7 days | Cache-aside |
| Session / JWT | `session:{token_hash}` | 15 min | Write-through |
| Rate limit window | `ratelimit:{ip}:{minute}` | 60 sec | Sliding window |
| Idempotency key | `idem:{key}` | 24 hr | Write-through |
| Search autocomplete | `autocomplete:{prefix}` | 10 min | Cache-aside |

### Cache Invalidation Rules
- **Product update** (price, stock) → invalidate `product:{id}:meta` immediately.
- **Order placed** → stock decrement is performed directly in Redis (write-through); DB updated asynchronously.
- **Cart change** → cart cache is updated synchronously on every add/remove.

### CDN Caching
- Product images, thumbnails, and static assets are served from Cloudflare CDN with a 24-hour cache TTL.
- Product listing pages are partially cached at the CDN edge for anonymous users (TTL: 1 min) to absorb homepage traffic spikes.

---

## 3. Database Scaling

### Read Replicas
- Each PostgreSQL primary has **2 read replicas** in the same region.
- Read-heavy queries (GET /products, GET /orders, search filters) are routed to replicas via a read-write proxy (PgBouncer or AWS RDS Proxy).
- Write queries (INSERT order, DECR stock) always go to the primary.

### Connection Pooling
- **PgBouncer** runs in transaction-pooling mode in front of each PostgreSQL primary.
- Limits per-service DB connections to a fixed pool, preventing connection exhaustion under traffic spikes.
- Each service maintains a pool of 20–50 connections regardless of the number of running pods.

### Database Sharding (for Scale-Out)
- **Order DB** — sharded by `user_id` using consistent hashing. Orders for the same user always land on the same shard, making per-user queries fast.
- **Product DB** — sharded by `category_id` for seller-heavy categories.
- Sharding is managed via an application-level shard router; no cross-shard joins are required because each service owns only its data.

### Elasticsearch Scaling (Search Service)
- Elasticsearch index is partitioned into **5 primary shards** with **1 replica each**.
- Replicas serve read queries in parallel, allowing search throughput to scale horizontally.
- Product data is indexed asynchronously via a Kafka `product.updated` consumer — search index is eventually consistent with the product DB (typically < 2 seconds lag).

### ClickHouse (Analytics)
- All order, payment, and user activity events flow from Kafka into ClickHouse via a Kafka consumer.
- ClickHouse partitions data by `toYYYYMM(placed_at)` for time-range query efficiency.
- Used exclusively for read-heavy analytics dashboards — never queried in the critical order flow.

---

## 4. Horizontal Scaling (Kubernetes)

### Container Orchestration
- Each microservice is packaged as a Docker container and deployed on **Kubernetes (EKS / GKE)**.
- Services are stateless; all state lives in Redis or PostgreSQL.

### Auto-Scaling
- **Horizontal Pod Autoscaler (HPA)** scales pod replicas based on:
  - CPU utilization (target: 70%)
  - Custom metrics via Prometheus adapter (e.g., orders_per_second > 500 → scale up Order Service)
- **Cluster Autoscaler** provisions new EC2/GKE nodes when existing nodes are at capacity.

### Pre-Scaling for Sale Events
- Before a scheduled sale or flash event, **Kubernetes node pools are manually pre-scaled** 15–30 minutes in advance.
- Redis inventory stock is pre-loaded with accurate product quantities.
- Kafka consumer group lag is monitored; additional consumers are provisioned if lag grows.

---

## 5. Fault Tolerance and Resilience

### Circuit Breakers
- **Resilience4j** (Java) or **Hystrix** wraps all inter-service HTTP calls.
- If Order Service calls Payment Service and it fails 5 times in 10 seconds, the circuit opens — Order Service stops calling Payment Service and returns a graceful fallback (queue the order for retry) until the circuit resets.

### Retry with Backoff
- Kafka consumers retry failed event processing with **exponential backoff**: 1s → 2s → 4s → DLQ.
- HTTP calls between services retry up to 3 times with jitter before failing.

### Dead Letter Queue (DLQ)
- Failed Kafka events (after all retries) are routed to a DLQ topic.
- An on-call alert is triggered; engineers can inspect and replay DLQ messages after fixing the root cause.

### Multi-Region Deployment (Future)
- For global availability, the system can be deployed across 2 AWS regions (primary + warm standby).
- Route 53 latency-based routing directs users to the nearest healthy region.
- PostgreSQL uses **logical replication** to keep the standby region in sync (RPO < 5 min).

# Caching

A **cache** is a smaller, faster storage layer that keeps a copy of data whose original source is slower or more expensive to reach. It trades **memory (and some risk of staleness)** for **speed and reduced load** on the underlying system.

---

## 1. Purpose — Why Cache?

Caching addresses three fundamental problems:

| Problem | How caching helps |
|---|---|
| **Latency** | RAM access (~100 ns) is orders of magnitude faster than a disk read, a database query (~ms), or a remote API call (tens–hundreds of ms). |
| **Load** | If 10,000 users request the same product page, compute it once and serve 9,999 copies from cache — the database is protected from repeated identical work. |
| **Cost** | Fewer calls to paid APIs, less compute, smaller database instances. |

The principle that makes caching work is **locality**: recently or frequently accessed data is likely to be accessed again soon (*temporal locality*). A cache only pays off when the **hit ratio** (hits / total lookups) is high enough to justify the memory and complexity.

> **Rule of thumb:** cache data that is *read often, written rarely, and expensive to produce*.

---

## 2. Common Usages

- **Database query / object caching** — store the result of an expensive query (e.g., in Redis or Memcached) so repeated reads skip the database entirely. The classic pattern for read-heavy applications.
- **HTTP / web caching** — browsers, CDNs, and reverse proxies (Varnish, nginx) cache responses, controlled by headers such as `Cache-Control`, `ETag`, and `Expires`. Static assets should almost never hit the origin server.
- **Application-level memoization** — caching the result of a pure or expensive function in-process (Python `functools.lru_cache`, Java `ConcurrentHashMap`, Spring `@Cacheable`).
- **Session storage** — user sessions kept in a fast shared store (e.g., Redis) so any application server can handle any request.
- **DNS caching** — resolved domain names cached at the OS / resolver level with a TTL.
- **CPU caches (L1/L2/L3)** — the same concept in hardware; the reason iterating an array is faster in practice than a linked list.
- **Computed / derived data** — rendered HTML fragments, image thumbnails, ML inference results, dashboard aggregations.

---

## 3. Internal Architecture

Conceptually, a cache is a **key-value store with a bounded size**, plus policies that decide what happens at the boundaries.

### 3.1 Storage structure

The heart is usually a **hash table** for O(1) lookup by key.

The canonical **LRU cache** design pairs the hash map with a **doubly linked list**:

```
HashMap:  key ──▶ node in list

List (recency order):
HEAD (most recent) ◀──▶ ... ◀──▶ TAIL (least recent, evicted first)
```

- On **read/write**: move the node to the head — O(1).
- On **eviction**: remove the tail node — O(1).

### 3.2 Eviction policy — what to remove when full

| Policy | Idea | When it fits |
|---|---|---|
| **LRU** (Least Recently Used) | Evict what hasn't been touched longest | Sensible default for most workloads |
| **LFU** (Least Frequently Used) | Evict what is accessed least often | Stable popularity distributions; more bookkeeping |
| **FIFO** | Evict the oldest insert, regardless of use | Simplicity above all |
| **TTL-based** | Entries expire after a fixed lifetime | Bounding staleness, not just size |
| **W-TinyLFU** (Caffeine) | Hybrid recency + frequency with probabilistic counters | Best hit ratios in modern in-process caches |

### 3.3 Expiration / freshness

TTL (time-to-live) bounds **staleness**, independently of memory pressure. Expired entries are removed:

- **Lazily** — checked on read; expired entries are discarded when touched.
- **Actively** — a background sweeper samples and removes expired keys (scanning everything eagerly would be too costly).

Most systems (including Redis) combine both.

### 3.4 Write policies — keeping cache and source in sync

| Pattern | Flow | Trade-off |
|---|---|---|
| **Cache-aside** (lazy loading) | App checks cache → on miss, reads DB and populates cache | Most common; cache is passive; first request is slow (cold miss) |
| **Read-through / write-through** | The cache sits in front of the store and loads/writes on your behalf; writes hit cache + DB synchronously | Consistent, simpler app code, but slower writes |
| **Write-behind** (write-back) | Writes hit the cache; flushed to the DB asynchronously | Fast writes, but data loss possible on crash |
| **Invalidation on update** | On writes, *delete* the cached key (preferred) or update it | The famously hard part of caching |

> *"There are only two hard things in computer science: cache invalidation and naming things."* — Phil Karlton

**Recommended starting point:** cache-aside + TTL. It is simple and self-healing — worst case, stale data expires on its own.

### 3.5 Concurrency & distribution

- **In-process caches** need thread-safe structures (striped locks, lock-free maps).
- **Distributed caches** (Redis Cluster, Memcached) shard keys across nodes, typically via **consistent hashing**, so adding/removing a node reshuffles only a fraction of the keys.
- **Cache stampede (thundering herd):** when a hot key expires, thousands of concurrent requests may hit the database simultaneously. Mitigations:
  - **Request coalescing** — only one caller recomputes; others wait for the result.
  - **Probabilistic early refresh** — refresh slightly before actual expiry.
  - **Locking / single-flight** around the recompute.

### 3.6 Key metric

**Hit ratio = hits / (hits + misses).** A low hit ratio means you are paying memory and complexity for little benefit — sometimes the right answer is *not* to cache. Monitor it, along with eviction counts and memory usage.

---

## 4. Caching in AWS

AWS offers caching at every layer of a typical architecture — from the edge, through the API layer, down to the database.

```
User ──▶ CloudFront (edge/CDN) ──▶ API Gateway (API response cache)
              │                          │
              ▼                          ▼
         S3 (static)               Lambda / ECS / EC2
                                         │
                          ┌──────────────┼──────────────┐
                          ▼              ▼              ▼
                    ElastiCache        DAX          RDS/Aurora
                   (Redis/Valkey/   (DynamoDB       (internal
                    Memcached)       cache)         buffer pool)
```

### 4.1 Amazon ElastiCache — the general-purpose cache

Fully managed in-memory data store supporting **Redis/Valkey** and **Memcached**. The workhorse for application-level caching.

- **Typical use:** cache-aside in front of RDS/Aurora/DynamoDB, session storage, leaderboards, rate limiting, pub/sub.
- **Redis/Valkey engine:** rich data structures (hashes, sorted sets), replication, automatic failover, cluster-mode sharding, persistence options.
- **Memcached engine:** simpler, multi-threaded, pure cache semantics (no persistence/replication) — good for straightforward horizontal scaling of a plain cache.
- **ElastiCache Serverless:** no capacity planning; scales automatically and bills per usage — good default for spiky or unknown workloads.
- Related: **Amazon MemoryDB** — Redis-compatible but *durable* (multi-AZ transaction log); use it when the in-memory store is the primary database, not just a cache.

### 4.2 Amazon CloudFront — caching at the edge (CDN)

Caches HTTP responses at 600+ global edge locations, close to users.

- **Typical use:** static assets from S3, images/video, but also cacheable API responses and full pages.
- Controlled via **cache policies** (which headers/cookies/query strings form the cache key) and origin `Cache-Control` headers.
- Supports **invalidations** (purge by path) and **origin shield** (an extra caching layer that collapses requests to the origin).
- Biggest win: requests served at the edge never touch your infrastructure at all.

### 4.3 Amazon API Gateway response caching

A built-in, per-stage cache (0.5 GB – 237 GB) for REST APIs.

- Caches responses keyed by resource + method + configured parameters, with a TTL (default 300 s).
- **Typical use:** endpoints returning slowly changing data (catalogs, reference data) — reduces Lambda invocations and backend load without any code changes.

### 4.4 DynamoDB Accelerator (DAX)

A **read-through / write-through** cache purpose-built for DynamoDB.

- API-compatible with DynamoDB: point the SDK client at DAX and reads go from single-digit **milliseconds to microseconds**.
- Caches both individual items (`GetItem`) and query/scan results, in separate internal caches.
- **Typical use:** extremely read-heavy, latency-sensitive workloads (gaming, ad-tech, real-time bidding) where even DynamoDB's normal latency is too slow, or where hot-key reads would be expensive.
- Note: eventually consistent reads only benefit; strongly consistent reads bypass the cache.

### 4.5 Caching inside compute

- **Lambda:** the execution environment is reused between invocations — anything stored in variables *outside the handler* (DB connections, config, small datasets) survives warm invocations. This is free in-process caching; just treat it as best-effort.
- **EC2 / ECS / EKS:** classic in-process caches (Caffeine, `lru_cache`) or a sidecar/local cache in front of ElastiCache for a two-tier (L1 in-process + L2 Redis) setup.

### 4.6 Databases' own caching

- **RDS / Aurora** maintain internal buffer pools (cached pages in RAM) — one reason instance sizing matters for read performance.
- **Route 53 / resolvers** cache DNS answers per record TTL.

### 4.7 Choosing the right AWS cache

| Need | Service |
|---|---|
| Cache static/HTTP content close to users | **CloudFront** |
| Cache REST API responses with zero code changes | **API Gateway caching** |
| General app cache, sessions, shared state | **ElastiCache (Redis/Valkey)** |
| Microsecond reads on DynamoDB, no code rewrite | **DAX** |
| Redis as a durable primary datastore | **MemoryDB** |
| Cheap per-invocation reuse in serverless code | **Lambda warm-container caching** |

These layers **stack**: a real production request might be served by CloudFront (edge), fall through to API Gateway cache, then hit Lambda whose warm container has config cached, which reads from ElastiCache, which fronts Aurora. Each layer absorbs traffic so the next one sees less.

---

## 5. Pitfalls & Best Practices

- **Decide the invalidation story *before* adding the cache.** Stale-data bugs are subtle and painful.
- **Prefer deleting keys over updating them** on writes — updates race; deletes converge.
- **Always set a TTL**, even with explicit invalidation — it's the safety net.
- **Watch the hit ratio.** A cache with a 20% hit rate is usually complexity without benefit.
- **Plan for cold starts** — a freshly deployed cache absorbs nothing; consider warming critical keys.
- **Guard against stampedes** on hot keys (coalescing, jittered TTLs).
- **Don't cache what's cheap** — caching adds a consistency liability; only take it on when the read is genuinely expensive or frequent.

# System Design Foundations

> **Purpose:** Core concepts that underpin every system design interview.   
> **Rule:** Every design decision must be justified using one or more of these concepts.

---

## Table of Contents

1. [CAP Theorem](#1-cap-theorem)
2. [PACELC — The Extension of CAP](#2-pacelc--the-extension-of-cap)
3. [Consistency Models](#3-consistency-models)
4. [SQL vs NoSQL — When to Pick Which](#4-sql-vs-nosql--when-to-pick-which)
5. [Caching Strategies](#5-caching-strategies)
6. [Load Balancing](#6-load-balancing)
7. [Message Queues](#7-message-queues)
8. [CDN](#8-cdn)
9. [DNS Internals](#9-dns-internals)
10. [Database Sharding and Replication](#10-database-sharding-and-replication)
11. [The System Design Framework](#11-the-system-design-framework)

---

## 1. CAP Theorem

A distributed system can guarantee **at most two** of three properties simultaneously:

| Property | Meaning |
|---|---|
| **Consistency (C)** | Every read receives the most recent write or an error. All nodes see the same data at the same time. |
| **Availability (A)** | Every request receives a response (not necessarily the latest data). System is always operational. |
| **Partition Tolerance (P)** | System continues operating even when network messages between nodes are dropped or delayed. |

### The real-world constraint

**Network partitions are not optional** — they will happen in any distributed system. So the real choice is always:

- **CP** — Consistency + Partition Tolerance (sacrifice availability)
- **AP** — Availability + Partition Tolerance (sacrifice consistency)

CA systems only exist on a single machine.

### Scenario map

| Scenario | Choice | Reason |
|---|---|---|
| Bank transfer / payments | **CP** | Stale read = money in two accounts simultaneously |
| DNS resolution | **AP** | Slightly stale IP better than no answer |
| Shopping cart | **AP** | Merge conflicts at checkout; don't block shopping |
| Social media like counter | **AP** | Approximate count is fine |
| Domain availability check | **AP** (check) + **CP** (registration) | Fast check ok; actual write must be atomic |
| Distributed config (etcd) | **CP** | Half-fleet with old DB password = catastrophic |

### Interview sentence

> "Network partitions aren't optional — they will happen. So the real choice is always CP vs AP. I'd choose CP here because [reason] — a stale read in this context causes [specific harm]. I'd choose AP here because [reason] — eventual convergence is acceptable because [why]."

---

## 2. PACELC — The Extension of CAP

**If Partition** → choose between **A**vailability and **C**onsistency  
**Else (normal operation)** → choose between **L**atency and **C**onsistency

This matters because even without a partition, replication has a latency cost.

| Category | Example Systems | Trade-off |
|---|---|---|
| **PA/EL** — availability + low latency | DynamoDB, Cassandra, CouchDB | Sacrifice consistency for speed |
| **PC/EC** — strong consistency | HBase, Zookeeper, etcd, Spanner | Pay latency for correctness |

---

## 3. Consistency Models

From strongest to weakest:

### Linearizability (strongest)
All operations appear atomic at a single point in real time. Global ordering matches wall clock. Used by: etcd, Google Spanner.

### Strong consistency
After a write completes, every subsequent read returns that value. Achieved via single primary or quorum reads. Higher latency.  
**Used by:** PostgreSQL with sync replication, etcd, Zookeeper.  
**When:** financial transactions, inventory counts.

### Sequential consistency
All nodes see operations in the same order — but that order need not match wall-clock time.

### Causal consistency
Causally related operations are seen in the same order everywhere. Independent operations may appear in different orders.  
**Used by:** MongoDB causal sessions, collaborative editing systems.

### Read-your-writes (session consistency)
A client always sees its own writes immediately after making them.  
**Implementation:** route same session's reads to same replica, or use session token with timestamp.  
**Required by:** virtually all user-facing web apps.

### Eventual consistency (weakest)
All replicas will converge given no new writes. Reads may return stale data.  
**Used by:** DNS, CDN caches, DynamoDB default, Cassandra default.  
**Conflict resolution:** LWW (last-write-wins), CRDTs, application-level merge.

### Interview one-liner

> "Strong consistency: after a write, every read returns that value — costs latency. Eventual consistency: replicas converge over time but reads may be stale — much faster, acceptable when approximate values are fine (like counts, view counts, DNS TTLs)."

---

## 4. SQL vs NoSQL — When to Pick Which

### Decision framework

```
Need complex JOINs or ACID multi-table transactions?  → SQL (PostgreSQL, MySQL)
Simple key-value lookups, sessions, caching?           → Key-value (Redis, DynamoDB)
Flexible/nested schema, variable attributes?           → Document (MongoDB, Firestore)
Write-heavy, time-ordered, massive scale?              → Wide-column (Cassandra, HBase)
Relationship traversal is the core operation?          → Graph (Neo4j, Neptune)
Time-series metrics, IoT, monitoring?                  → TimescaleDB, InfluxDB
```

### SQL (PostgreSQL, MySQL)
**Strengths:** ACID, complex queries, rich indexing, mature tooling, strong consistency.  
**Weaknesses:** horizontal write scaling is hard, schema migrations required.  
**Scale pattern:** primary + read replicas → connection pooling (PgBouncer) → sharding last resort.  
**Use for:** users, orders, payments, anything requiring multi-table transactions.

### Key-value (Redis, DynamoDB, Memcached)
**Strengths:** O(1) get/set, extreme throughput, horizontal scaling.  
**Weaknesses:** no joins, no complex queries, must know key upfront.  
**Use for:** caching, sessions, rate limiting, distributed locks, feature flags, URL mappings.  
**Redis extras:** sorted sets (leaderboards/timelines), pub/sub, Lua scripting, streams.

### Document (MongoDB, Firestore)
**Strengths:** flexible schema (add fields without migrations), nested/hierarchical data.  
**Weaknesses:** no native JOINs, eventual consistency default, data duplication.  
**Use for:** user profiles, product catalogs, CMS, variable-schema data.

### Wide-column (Cassandra, HBase, BigTable)
**Strengths:** massive horizontal write scale, excellent for time-series, designed for write-heavy.  
**Weaknesses:** query patterns must be defined upfront, no JOINs, eventual consistency.  
**Design rule:** design tables per query, not per data model. Duplication is intentional.  
**Use for:** IoT sensor data, application logs, user activity feeds at Twitter/Instagram scale.

### Graph (Neo4j, Amazon Neptune)
**Strengths:** relationship traversal is O(relationship count) not O(table size).  
**Weaknesses:** not general purpose, poor for aggregation.  
**Use for:** social networks (friends-of-friends), recommendations, fraud detection, RBAC hierarchies.

### Interview warning

> "The common mistake: 'I'd use MongoDB because NoSQL scales better.' Scale depends on access patterns, not the database type. PostgreSQL with read replicas scales reads very effectively. I'd choose based on the specific access patterns and consistency requirements."

---

## 5. Caching Strategies

### Where caches live (mention all layers in any design)

```
Client-side (browser)  → HTTP cache headers, localStorage
CDN (edge)             → Static assets, video segments, API responses
API Gateway            → Rate-limited response cache
Application layer      → Redis/Memcached — hot data, computed results
Database               → Query result cache, buffer pool
```

### Cache-aside (lazy loading) — most common

```
get(key):
  cached = cache.get(key)
  if cached → return cached
  data = db.read(key)
  cache.set(key, data, ttl)
  return data
```

✓ Only caches what's requested. Resilient to cache failure (just slower).  
✗ Cache miss = 3 round trips. Cold start is slow. Stale on external DB updates.  
**Use for:** read-heavy workloads, URL redirects, domain lookups.

### Write-through — strong consistency

```
write(key, val):
  cache.set(key, val)
  db.write(key, val)   ← both written synchronously
  return OK
```

✓ Cache never stale. Reads always fast.  
✗ Write latency doubles. Cache fills with never-read data.  
**Use for:** payment state, user balances, anything that must always be fresh on read.

### Write-behind (write-back) — high write throughput

```
write(key, val):
  cache.set(key, val)
  return OK              ← immediate response
  // background: flush to DB asynchronously
```

✓ Lowest write latency. Absorbs write spikes.  
✗ Data loss risk if cache fails before flush. Not for ACID requirements.  
**Use for:** analytics counters, like counts, view counts.

### Eviction policies

| Policy | When to use |
|---|---|
| **LRU** (least recently used) | Default choice. Good when recent access predicts future access. |
| **LFU** (least frequently used) | Stable popularity patterns. Old trending items evicted. |
| **TTL** (time to live) | Always combine with other policies. Prevents stale data. |
| **FIFO / Random** | Simple, rarely optimal. Use when patterns are unknown. |

### Asymmetric TTL — a senior detail

Different states deserve different TTLs based on how quickly they change:

```
Domain taken    → TTL 1 hour    (rarely becomes available)
Domain available → TTL 5 min   (can become taken instantly)

Tweet content   → TTL 24 hours  (rarely changes)
User presence   → TTL 5 min    (changes frequently)
```

---

## 6. Load Balancing

### L4 vs L7

| | L4 (Transport layer) | L7 (Application layer) |
|---|---|---|
| **Routes on** | IP address + TCP/UDP port | HTTP headers, URL paths, cookies |
| **Speed** | Faster — no packet inspection | Slower — full packet inspection |
| **Features** | Raw TCP/UDP routing | SSL termination, path routing, request rewriting |
| **Use for** | Non-HTTP protocols, raw TCP | Microservices, HTTP APIs, A/B testing |

### Balancing algorithms

| Algorithm | How | Best for |
|---|---|---|
| **Round robin** | Rotate through servers in order | Identical servers, simple default |
| **Weighted round robin** | Higher-capacity servers get more | Heterogeneous fleets |
| **Least connections** | New request → server with fewest active | Variable-duration requests (WebSocket, streaming) |
| **IP hash / sticky** | Same client IP → same server | Session state not shared across servers |
| **Power of two choices** | Pick 2 random, send to less busy | Near-optimal, very simple |

### Health checks

LB continuously pings `/health` endpoint. Fail N consecutive checks → remove from rotation. Healthy again → re-add. This enables zero-downtime deploys: drain old instance, bring up new one.

---

## 7. Message Queues

### When to use

| Scenario | Why a queue |
|---|---|
| Async processing | Return immediately, process in background (video transcoding, email) |
| Load leveling | Absorb traffic spikes; workers drain at steady rate |
| Fan-out | One event triggers multiple independent consumers |
| Fault tolerance | Consumer down → messages queue up, process on recovery |
| Decoupling | Producer doesn't need to know about consumers |

### Kafka vs RabbitMQ

| | Kafka | RabbitMQ |
|---|---|---|
| **Model** | Immutable log | Traditional queue |
| **Message retention** | Days/weeks (configurable) | Deleted after ACK |
| **Consumer model** | Consumer groups, each reads independently | Competing consumers |
| **Throughput** | Millions/sec | Thousands/sec |
| **Use for** | Event sourcing, analytics pipelines, audit logs | Task queues, RPC-style jobs, notifications |

### Delivery guarantees

| Guarantee | Behaviour | Use for |
|---|---|---|
| **At-most-once** | 0 or 1 deliveries, may be lost | Metrics where losing a few is OK |
| **At-least-once** | 1+ deliveries, may duplicate | Most cases — consumer must be idempotent |
| **Exactly-once** | Delivered exactly once | Payments, financial transactions |

**Dead letter queue:** messages that fail after N retries → DLQ. Prevents poison messages blocking the queue. Always configure one.

---

## 8. CDN

### How it works

```
User → DNS resolves to nearest CDN edge node
  → Edge cache hit  → return content (~5-20ms)
  → Edge cache miss → fetch from origin, cache, return to user
                      (future users in region get cache hit)
```

### Pull CDN vs Push CDN

| | Pull CDN | Push CDN |
|---|---|---|
| **How** | Edge fetches from origin on first miss | You proactively push content to edges |
| **Config** | Zero setup | Must manage what gets pushed |
| **First request** | Slightly slower (origin fetch) | Instant — already cached |
| **Use for** | Unpredictable access patterns, most cases | Large files, predicted high traffic (viral videos) |

### Cache-Control headers

```
Cache-Control: max-age=31536000, immutable    ← static assets (versioned filenames)
Cache-Control: max-age=3600                   ← semi-static content
Cache-Control: no-cache                       ← must revalidate before serving
Cache-Control: no-store                       ← never cache (private, personalised)
```

### When CDN is the architecture (not an optimisation)

For YouTube at 46 Tbps, for Twitter media, for domain search suggestions — the CDN is not an add-on. It IS the read architecture. Origin only handles cache misses and new content.

---

## 9. DNS Internals

### Resolution chain

```
1. Browser cache         → TTL-based. Hit → done.
2. OS / hosts file       → Local resolver + /etc/hosts. Hit → done.
3. Recursive resolver    → Your ISP's DNS (or 8.8.8.8). Queries on your behalf.
4. Root nameservers      → 13 clusters worldwide. Knows .com → Verisign.
5. TLD nameserver        → .com registry (Verisign). Points to authoritative NS.
6. Authoritative NS      → GoDaddy's own DNS servers. Returns the A record.
```

### Record types

| Record | Maps | Example |
|---|---|---|
| **A** | hostname → IPv4 | `godaddy.com → 52.85.45.12` |
| **AAAA** | hostname → IPv6 | Same for IPv6 |
| **CNAME** | hostname → hostname (alias) | `www.godaddy.com → godaddy.com` |
| **MX** | domain → mail server | Routes email |
| **NS** | domain → authoritative nameserver | Which server is authoritative |
| **TXT** | arbitrary text | SPF, DKIM, domain verification |
| **TTL** | — | How long resolvers cache this record |

### TTL and propagation

Lower TTL = faster change propagation but more queries to authoritative server.

**GoDaddy-specific:** when a domain's DNS records are updated, the change propagates gradually as resolver TTLs expire. Can take up to 48 hours globally. Best practice: lower TTL 24h before a planned change, make the change, restore TTL after.

### CNAME at root domain — cannot do it

A CNAME at the root (naked domain) is illegal per DNS spec — the root must be an A record. This is why AWS Route 53 has ALIAS records (GoDaddy equivalent: CNAME flattening) — they resolve the CNAME server-side and return an A record.

---

## 10. Database Sharding and Replication

### Replication — scale reads

**Primary-replica:** One primary handles writes. Multiple replicas receive copies asynchronously. Reads distributed across replicas.  
✓ Scale read throughput linearly with replicas. Automatic failover.  
✗ Replication lag — reads may be slightly stale. Write bottleneck remains.

**Multi-primary:** Multiple primaries accept writes. Requires conflict resolution.  
✓ High write throughput across regions.  
✗ Conflict resolution complexity. Use only when single-primary write throughput is genuinely exhausted.

### Sharding — scale writes

Splits data across multiple DB instances. Each shard owns a partition of the data.

| Strategy | How | Pros | Cons |
|---|---|---|---|
| **Hash sharding** | shard = hash(key) % N | Uniform distribution | No range queries |
| **Range sharding** | A-M → shard 1, N-Z → shard 2 | Range queries work | Hot shards if distribution uneven |
| **Directory sharding** | Lookup table: key → shard | Most flexible | Extra hop for lookup |
| **Consistent hashing** | Virtual nodes on a ring | Adding/removing shards only remaps fraction of keys | More complex |

### Sharding trade-offs to always mention

- Cross-shard queries are expensive (scatter-gather)
- JOINs across shards are nearly impossible — must denormalise
- Re-sharding is painful — design shard key for growth
- Hotspot problem — popular user routes all traffic to one shard regardless of hash

### The senior recommendation

> "I'd start with a single primary + read replicas. Only add sharding when a single primary cannot handle write throughput — which is much later than most people think. Premature sharding is one of the most expensive architectural mistakes in distributed systems."

---

## 11. The System Design Framework

Apply this structure to every question. 45-60 minutes total.

### Step 1 — Requirements clarification (5 min)

Always ask before touching the whiteboard:

```
Functional requirements:   What features must the system have?
Non-functional:            Scale (users, QPS), latency SLAs, availability (99.9% vs 99.99%),
                           durability, consistency requirements
Out of scope:              What can I explicitly ignore?
```

### Step 2 — Capacity estimation (5 min)

Always show the math. Interviewers want to see you can reason about numbers.

```
QPS:       requests/day ÷ 86,400 = baseline QPS. Peak = 3-10× baseline.
Storage:   requests/day × avg_record_size × 365 = annual storage
Bandwidth: QPS × avg_response_size = outbound bandwidth
Cache:     storage × hot_data_fraction (usually 20%) = cache size needed
```

### Step 3 — High-level design (10 min)

Draw the boxes: clients → load balancer → services → databases → caches. Walk through the happy path end-to-end. Identify the primary read path and write path separately.

### Step 4 — Component deep-dive (15 min)

Pick 2-3 most complex components. For each: schema, API design, key algorithms, data structures. This is where LLD appears within HLD.

### Step 5 — Scale and reliability (10 min)

Identify bottlenecks. How does each component fail? Mitigation strategies:
- DB: replicas for reads, sharding for writes
- Cache: eviction strategy, cache stampede prevention
- Services: horizontal scaling, circuit breakers
- Network: CDN for static content, rate limiting for abuse

### Step 6 — Trade-offs (5 min)

Every major decision has a trade-off. State it explicitly:

> "I chose X over Y because [specific reason]. The trade-off is [what you lose], which is acceptable because [why it doesn't matter in this context]."

### The senior differentiators

- Ask clarifying questions before drawing anything
- State data structure choice before writing code
- Mention distributed/multi-instance implications proactively
- Use correct HTTP status codes (410 vs 404, 429 vs 503)
- Mention idempotency for write operations
- Eventual consistency is often the right answer — say so explicitly and justify it

---


# System Design — URL Shortener (TinyURL / Bitly)

> **Difficulty:** Entry-to-mid level HLD. Good warm-up question.  
> **Key insight:** Read path (120K redirects/sec) and write path (1.2K creates/sec) are completely asymmetric — design them separately.  
> **What interviewers test:** Short code algorithm, 301 vs 302 semantics, three-layer caching, analytics decoupling.

---

## Table of Contents

1. [Requirements](#1-requirements)
2. [Capacity Estimation](#2-capacity-estimation)
3. [High-Level Design](#3-high-level-design)
4. [Shortening Algorithm](#4-shortening-algorithm)
5. [Redirect Flow](#5-redirect-flow)
6. [Database Schema](#6-database-schema)
7. [Scale and Reliability](#7-scale-and-reliability)
8. [Trade-offs](#8-trade-offs)
9. [Interview Script](#9-interview-script)
10. [Follow-up Probes and Answers](#10-follow-up-probes-and-answers)

---

## 1. Requirements

### Functional
- User submits a long URL → receives a short URL (e.g. `short.ly/abc1234`)
- Visiting the short URL redirects to the original long URL
- Custom aliases: user can request `short.ly/mylink` instead of a generated code
- Optional expiration date per URL
- Basic analytics: click count per short URL (not real-time)

### Non-functional
- High availability — redirects are revenue-critical, must never be down
- Low latency — redirect in under 100ms globally
- Durability — short URLs never disappear until explicitly expired or deleted
- Scale: 100M URLs created per day, read:write ratio ~100:1

### Out of scope
- User authentication and account management (acknowledge but don't design)
- Real-time analytics dashboard
- Link preview / OG tag extraction

---

## 2. Capacity Estimation

```
Writes:
  100M URLs/day ÷ 86,400 = 1,157 writes/sec
  Peak (3×):               ~3,500 writes/sec

Reads (redirects):
  100:1 read:write ratio = 115,700 reads/sec
  Peak:                    ~350,000 reads/sec

Short code length:
  62^7 = 3.5 trillion combinations → 7 chars sufficient for decades

Storage per URL:    ~500 bytes (long URL + short code + metadata)
Annual storage:     100M × 365 × 500B = ~18 TB/year (trivial with object storage)

Cache sizing:
  Top 20% of URLs generate 80% of traffic (Pareto principle)
  100M active URLs × 20% × 500B = ~10 GB → fits in a single Redis cluster
```

---

## 3. High-Level Design

```
WRITE PATH                           READ PATH (100× more traffic)
─────────────────────────────────    ───────────────────────────────────────

Client                               User Browser
  │                                    │
  ▼                                    ▼
Load Balancer (L7, SSL termination)  CDN Edge (Cloudflare / Fastly)
  │                                    │  cache hit  → 302 in ~5ms
  ▼                                    │  cache miss ↓
URL Shortener Service                Redirect Service
  │                                    │
  ├─→ ID Generator (Snowflake)          ├─→ Redis cache     → 302 in ~15ms
  │                                    │     cache miss ↓
  ▼                                    ├─→ PostgreSQL read replica  ~30ms
PostgreSQL primary                   Analytics (Kafka → async)
  │                                    NOT in the redirect path
  └─→ Read replicas (3×)
```

**Write path:** validate → generate short code → persist to PostgreSQL → return to caller.  
**Read path:** CDN → Redis → PostgreSQL replica. Primary only handles writes.

---

## 4. Shortening Algorithm

Three approaches. Know all three and why the third is correct.

### Approach 1 — MD5 hashing (reject this)

```typescript
// MD5("https://example.com/very-long-path") → "a9b3c7d8e2f1..."
// Take first 7 chars → "a9b3c7d"
```

**Problems:**
- Different URLs can produce the same 7-char prefix (collision)
- Same URL always produces the same code — can't create two short codes for one URL
- Must check DB for collision on every generation — no uniqueness guarantee by construction

### Approach 2 — Random Base62 string (acceptable, not ideal)

```typescript
function generateCode(): string {
  const chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
  return Array.from({ length: 7 }, () =>
    chars[Math.floor(Math.random() * 62)]
  ).join('');
}
```

**Problems:**
- Must check DB for collision on every generation
- As the table fills, collision probability grows
- At 50% fill: ~1 in 2 generated codes already exist — retry loops become expensive

### Approach 3 — Auto-increment ID + Base62 encoding (recommended)

```typescript
// Step 1: get a globally unique integer (DB auto-increment or Snowflake ID)
// e.g. ID = 123456789

const CHARS = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';

function toBase62(id: number): string {
  let result = '';
  while (id > 0) {
    result = CHARS[id % 62] + result;
    id = Math.floor(id / 62);
  }
  return result.padStart(7, 'a'); // always 7 chars
}

// 123456789 → "BFXPFp" → padded → "aBFXPFp"
```

**Why this is best:**
- Zero collisions — unique integer = unique code, by construction
- No DB check needed — no retry logic
- Deterministic: O(log₆₂ n) encoding
- 62^7 = 3.5 trillion codes before exhaustion

**Caveat:** sequential IDs produce sequential, guessable codes.  
**Mitigation:** use Snowflake IDs (timestamp + machine ID + sequence) — looks random but is still globally unique. No predictable enumeration.

### Custom alias handling

```typescript
// User requests: POST /api/v1/urls { long_url: "...", custom_alias: "mylink" }

// 1. Validate: alphanumeric, 3-50 chars, no reserved words
// 2. Check if alias taken: SELECT WHERE short_code = 'mylink'
// 3. If available: insert with custom alias instead of generated code
// 4. Rate-limit custom aliases more strictly (abuse vector)
```

---

## 5. Redirect Flow

### 301 vs 302 — a critical decision

| | 301 Moved Permanently | 302 Found (Temporary) |
|---|---|---|
| **Browser caches** | Yes — permanently | No |
| **Analytics** | Lost — browser bypasses servers | Tracked — every visit hits servers |
| **Can update destination** | No — cached in browser forever | Yes |
| **Can expire URL** | No | Yes |
| **Server load** | Near-zero after first visit | Full traffic on every visit |

**Recommendation: 302 for any URL shortener that needs analytics or URL mutability.**  
Use 301 only if zero server load is the explicit goal and analytics are irrelevant.

### Full redirect flow — step by step

```
1. Browser: GET short.ly/abc1234
   DNS resolves to CDN edge node

2. CDN edge checks its own cache
   HIT  → return 302 immediately (~5ms)
   MISS → forward request to Redirect Service

3. Redirect Service checks Redis (cache-aside)
   HIT  → return 302, also populate CDN cache (~15ms)
   MISS → query PostgreSQL read replica

4. PostgreSQL lookup (B-tree index on short_code, O(log n))
   SELECT long_url, expires_at FROM urls WHERE short_code = 'abc1234'
   Populate Redis with TTL 24h
   Return 302 (~30ms total)

5. Async: publish click event to Kafka
   Analytics consumer aggregates counts separately
   The redirect NEVER waits on analytics — fire and forget
```

### 410 vs 404 for expired URLs — a senior HTTP detail

- **404 Not Found** — resource never existed or location unknown
- **410 Gone** — resource existed and is intentionally, permanently gone

Expired short URLs must return **410 Gone**.  
This tells search engines not to retry the URL. Using 404 for expired URLs is incorrect and commonly seen as a mistake in interviews.

---

## 6. Database Schema

```sql
-- Primary URL table
CREATE TABLE urls (
  id          BIGSERIAL       PRIMARY KEY,          -- auto-increment → drives Base62
  short_code  VARCHAR(16)     NOT NULL UNIQUE,       -- 7 chars generated OR custom alias
  long_url    TEXT            NOT NULL,              -- original URL, up to 2048 chars
  user_id     BIGINT,                                -- nullable (anonymous creation allowed)
  created_at  TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
  expires_at  TIMESTAMPTZ,                           -- NULL = never expires
  is_custom   BOOLEAN         DEFAULT FALSE          -- custom alias flag
);

-- Critical: redirect lookup is always by short_code
CREATE UNIQUE INDEX idx_short_code ON urls(short_code);  -- B-tree, O(log n)

-- For user dashboard: list my short URLs
CREATE INDEX idx_user_created ON urls(user_id, created_at DESC);

-- For expiry cleanup job (partial index — only rows with expiry set)
CREATE INDEX idx_expires ON urls(expires_at) WHERE expires_at IS NOT NULL;


-- Analytics table — SEPARATE from urls table
-- DO NOT co-locate with the hot redirect path
CREATE TABLE clicks (
  id          BIGSERIAL     PRIMARY KEY,
  short_code  VARCHAR(16)   NOT NULL,
  clicked_at  TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
  referrer    TEXT,
  user_agent  TEXT,
  country     CHAR(2)                             -- from IP geolocation
);

CREATE INDEX idx_clicks_code_time ON clicks(short_code, clicked_at DESC);
```

### Why analytics is a separate table

The clicks table grows at ~120K rows/sec. If co-located with the urls table, it thrashes the DB buffer pool and degrades every redirect. Write analytics asynchronously to Kafka, aggregate offline in ClickHouse or BigQuery. The redirect path must never block on analytics.

---

## 7. Scale and Reliability

### Three-layer caching for the read path

```
Layer 1 — CDN edge cache
  Cache the 302 response itself at the CDN
  Cache-Control: max-age=3600 for non-expiring URLs
  On URL deletion: purge CDN cache via API
  Covers ~60% of redirect traffic

Layer 2 — Redis (application cache)
  key:   short_code  →  value: { long_url, expires_at }
  TTL:   24 hours for non-expiring URLs
  Eviction: LRU
  Covers additional ~20%

Layer 3 — PostgreSQL read replicas (3 replicas)
  Handles remaining ~20% cold cache misses
  Primary only handles writes (~1,200/sec — trivial)
```

### ID generation options

| Option | Pros | Cons | When to use |
|---|---|---|---|
| **DB auto-increment (BIGSERIAL)** | Simple, zero extra infra | Single primary for ID generation; bottleneck ~50K/sec | Start here |
| **Snowflake IDs** | No central coordinator, 4M IDs/ms across 1024 machines | More complex setup | When write throughput demands it |

**Recommendation:** start with DB auto-increment. Switch to Snowflake when write throughput exceeds ~50K/sec (well past 100M URLs/day in this design).

### Cache stampede prevention for viral URLs

When a hot URL's Redis TTL expires, thousands of concurrent requests simultaneously miss cache and hit the DB — the "thundering herd."

```typescript
// Probabilistic early expiration — refresh before TTL expires
const ttlRemaining = await redis.ttl(shortCode);
const shouldEarlyRefresh = ttlRemaining < 300 && Math.random() < 0.1;
if (shouldEarlyRefresh) backgroundRefreshFromDb(shortCode);

// Alternative: distributed lock on cache miss
const lock = await redis.set(`lock:${shortCode}`, '1', 'NX', 'EX', 5);
if (lock) {
  const data = await db.findByShortCode(shortCode);
  await redis.setex(shortCode, 86400, JSON.stringify(data));
} else {
  await sleep(100); // another process is refreshing — wait and retry from cache
}
```

### URL expiry cleanup

```sql
-- Background job — hourly during off-peak
-- Batch with LIMIT to avoid table locks
DELETE FROM urls
WHERE expires_at < NOW()
  AND expires_at IS NOT NULL
LIMIT 1000;
-- Partial index on expires_at makes this fast
```

---

## 8. Trade-offs

### 301 vs 302

| Consideration | 301 | 302 |
|---|---|---|
| Server load after first visit | Near-zero (browser caches) | Full — every visit |
| Analytics capability | None (bypassed) | Full click tracking |
| URL mutability | Impossible — cached in browser | Yes — can update or expire |
| **When to use** | Static, permanent links, zero server load is the goal | Any production URL shortener with analytics or expiry |

### SQL vs NoSQL for URL storage

**Choose SQL (PostgreSQL):**
- Unique constraint on `short_code` is critical — SQL enforces this atomically
- Custom aliases need transactional uniqueness check
- Data model is simple and well-defined
- Operations familiarity — SQL is the safe default when scale doesn't demand otherwise

**DynamoDB would also work** — native key-value fit — but custom alias uniqueness becomes an application-level conditional put with a small race window under very high concurrency.

### Cache-aside vs write-through

**Choose cache-aside:**
- Most short URLs are long-tail (created once, accessed rarely or never)
- Write-through would fill cache with URLs that are never redirected — wasted memory
- Cache miss on first access is acceptable given the low write rate

### Base62 + counter vs random string

**Choose Base62 + counter (Snowflake):**
- Zero collisions — no retry logic needed
- Predictability concern mitigated by using Snowflake IDs (non-sequential, time-ordered)
- Simpler implementation and reasoning

---

## 9. Interview Script

### Opening (2 min) — requirements first, always

> "Before I design anything — let me clarify requirements. Functional: create short URLs, redirect on visit, custom aliases, optional expiry, click analytics. Non-functional: high availability for redirects, sub-100ms globally. I'm estimating 100M creates per day and a 100:1 read ratio — so about 120,000 redirects per second. That asymmetry completely shapes the architecture."

### Core insight (1 min) — state this early

> "The key insight: the write path and read path are completely asymmetric. Writes are ~1,200/sec and can tolerate 100ms. Redirects are 120,000/sec and need to be under 20ms globally. I'll design them as separate services with different scaling strategies."

### Algorithm (2 min)

> "For the short code — I won't use MD5 hashing. It has collision problems and ties the same long URL to the same short code always. I'll use auto-increment integer IDs encoded to Base62. Unique integer means unique code — zero collisions, no retry logic needed. I'll use Snowflake IDs rather than a plain counter to avoid predictable sequential enumeration. 62^7 gives us 3.5 trillion codes."

### 301 vs 302 (1 min) — this always comes up

> "For redirects: 302, not 301. A 301 is permanently cached in the browser — zero server load after first visit, but we lose analytics entirely and can't expire or update URLs. Those features require 302. One more detail: expired URLs return 410 Gone, not 404 — the resource existed and is intentionally removed. This tells search engines not to retry."

### Scaling (2 min)

> "120K reads/sec through three caching layers: CDN edge serves about 60% — the 302 response itself is cached. Redis serves another 20% — short_code to long_url mapping with 24-hour TTL. PostgreSQL read replicas handle the remaining 20% cold misses. The DB primary only sees writes — about 1,200/sec, which PostgreSQL handles trivially."

### Proactive trade-off (1 min)

> "Two trade-offs worth stating: cache-aside means the first redirect after creation hits the DB — I accept that. Most URLs are long-tail, write-through would waste cache space. And for viral URLs, I'd add probabilistic early expiration to prevent cache stampede — 10% chance of background refresh as TTL approaches zero."

---

## 10. Follow-up Probes and Answers

**"How would you scale to multiple regions?"**  
Active-active with regional read replicas. Snowflake IDs are globally unique regardless of which region generates them (machine ID bits differentiate regions). CDN already handles geographic read distribution. Writes route to the nearest regional primary, async-replicated globally. Read replicas in each region handle local cache misses.

**"How do you prevent abuse — spam links, malware URLs?"**  
Rate limiting per IP and per user (token bucket). Malware/phishing scanning via Google Safe Browsing API asynchronously — scan after creation, flag/remove if malicious. Require account creation for custom aliases (reduces anonymous abuse). Blacklist known spam domains at the validation layer.

**"How would you handle analytics at scale?"**  
Async Kafka pipeline — redirect service publishes click events (fire-and-forget). Analytics consumer aggregates into ClickHouse or BigQuery. Never block a redirect on an analytics write. Displayed counts are eventually consistent — "1.2K clicks" not "1,247 clicks" — intentional approximation, not a limitation.

**"Why not just use DynamoDB?"**  
DynamoDB works well — the key-value access pattern fits. Trade-off: the unique constraint on custom aliases becomes a DynamoDB conditional put, which has a small race condition window under very high concurrency. PostgreSQL's unique index handles this atomically. At 1,200 writes/sec, SQL is not the bottleneck, so there's no pressure to move to NoSQL here.

**"What happens when a URL is deleted?"**  
Soft delete: set `deleted = true` in DB. Purge from Redis immediately. Purge CDN cache via the CDN's purge API. Any subsequent request returns 410 Gone. Background job hard-deletes soft-deleted rows older than 30 days to reclaim storage.

**"How do you handle the same long URL being shortened multiple times?"**  
Two approaches: (1) allow duplicates — each call creates a new short code. Simple, no extra lookup. (2) deduplication — hash the long URL, check if it already exists, return existing short code. Deduplication trades the extra lookup for storage savings. For most use cases, allow duplicates — it's simpler and users sometimes want different short codes for the same destination for tracking purposes.

---

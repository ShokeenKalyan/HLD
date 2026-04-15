# System Design — Twitter / X (Social Media Feed)

> **Difficulty:** Hard HLD — the news feed / timeline problem is one of the most discussed distributed systems challenges.  
> **Key insight:** There is no clean solution to the fan-out problem. Pure fan-out on write explodes for celebrities (170M followers × 1 tweet = 170M writes). Pure fan-out on read collapses under read load (2,000 followees × 100K reads/sec = 200M DB queries/sec). The answer is always a hybrid.  
> **What interviewers test:** Fan-out strategies, celebrity problem, Redis sorted set for timelines, Snowflake IDs, cursor-based pagination, view count eventual consistency.

---

## Table of Contents

1. [Requirements](#1-requirements)
2. [Capacity Estimation](#2-capacity-estimation)
3. [The Core Problem — Fan-out](#3-the-core-problem--fan-out)
4. [High-Level Design](#4-high-level-design)
5. [Tweet Posting](#5-tweet-posting)
6. [Fan-out Pipeline](#6-fan-out-pipeline)
7. [Home Timeline Service](#7-home-timeline-service)
8. [The Celebrity Problem](#8-the-celebrity-problem)
9. [Search and Trending](#9-search-and-trending)
10. [Database Schema](#10-database-schema)
11. [Trade-offs](#11-trade-offs)
12. [Interview Script](#12-interview-script)
13. [Follow-up Probes and Answers](#13-follow-up-probes-and-answers)

---

## 1. Requirements

### Functional
- Post a tweet (280 chars, optional images/video)
- Follow / unfollow users
- Home timeline — ordered feed of tweets from people you follow
- User timeline — all tweets from a specific user
- Search tweets and users (full-text)
- Trending topics / hashtags
- Likes and retweets (acknowledge, don't deep-dive)

### Non-functional
- Scale: 400M users, 500M tweets per day, 100M DAU
- Timeline latency: home feed must load in under 200ms
- Write latency: tweet must appear in followers' feeds within 5 seconds
- Read-heavy: read:write ratio ~1000:1
- Availability: 99.99%

### Out of scope
- Direct messages (different system — see WhatsApp design)
- Twitter Spaces / live audio
- Ad serving and promoted content
- Algorithmic ranking / recommendation model internals

---

## 2. Capacity Estimation

```
Tweets/day:      500M → 5,800 tweets/sec
Peak (3×):       ~17,500 tweets/sec

Timeline reads:  100M DAU × 10 timeline reads/day = 1B reads/day
                 → 11,500 reads/sec avg, ~100K/sec peak

Fan-out cost:
  Average user:  200 followers
  5,800 tweets/sec × 200 = 1.16M fan-out writes/sec (manageable)

Celebrity cost:
  Elon Musk:     170M followers
  1 tweet        = 170M fan-out writes
  At 20 tweets/day = 3.4B writes from one account alone
  ← This is why pure fan-out on write fails

Storage:
  500M tweets × 280 chars ≈ 140 GB/day text (trivial)
  With media: 500M × 10% × 200KB = ~10 TB/day
```

---

## 3. The Core Problem — Fan-out

This is the central design question. State it clearly before drawing anything.

```
Option A — Pure fan-out on write (push model):
  When a tweet is posted, immediately push it to every
  follower's timeline cache.
  Reads are instant — just read your pre-built cache.

  Problem: Elon Musk tweets once.
  170M followers × 1 tweet = 170M Redis writes.
  At 100K writes/sec that takes 28 MINUTES to fan out.
  By then the tweet is ancient news.
  The fan-out queue falls permanently behind.

Option B — Pure fan-out on read (pull model):
  Timeline computed at read time.
  Fetch latest tweets from all followees, merge, sort, return.

  Problem: user follows 2,000 accounts.
  1 timeline request = 2,000 DB reads (one per followee).
  At 100K timeline reads/sec = 200M DB queries/sec.
  System collapses.

Option C — Hybrid (what Twitter / Instagram / Facebook use):
  Fan-out on write  for normal users (< 1M followers)
  Fan-out on read   for celebrities  (≥ 1M followers)
  Redis cache       for everything

  This is always the correct answer.
  The threshold is a tunable operational knob, not a hard constant.
```

---

## 4. High-Level Design

```
                          Client (web / mobile)
                                  │
                                  ▼
                     API Gateway + Load Balancer
                    ╱                │                ╲
                   ╱                 │                 ╲
                  ▼                  ▼                  ▼
         Tweet Service        Timeline Service      Search Service
         (write path)         (read path)           (read path)
                │                   │                    │
                ▼                   ▼                    ▼
         Kafka: tweet.created   Redis Timeline       Elasticsearch
                │                Cache (sorted        (inverted index
                │                sets per user)       full-text search)
                │
        ┌───────┴────────┐
        ▼                ▼
  Fan-out Worker    Search Indexer
  (normal users)    (async, ~2s lag)
        │
        ▼
  Redis sorted sets
  timeline:{user_id}
  [tweet_id, tweet_id, ...]
  (score = Snowflake ID = chronological order)

  Cassandra tweets table
  (permanent store — partitioned by user_id)

  Presence service        Social graph service
  (Redis + fan-out)       (Cassandra — two tables)
```

---

## 5. Tweet Posting

### Write path — step by step

```
Step 1 — Client posts
  POST /api/v1/tweets
  Body: { content: "Hello world", media_ids: [...] }
  JWT authentication required

Step 2 — Tweet service validates
  Content ≤ 280 chars
  Media IDs valid (media pre-uploaded separately, like YouTube)
  Rate limit check (2,400 tweets/day/account, 300/3hrs)
  Spam / abuse filter — async, doesn't block the response

Step 3 — Generate tweet ID
  Snowflake ID — 64-bit, time-ordered, globally unique
  Higher ID = newer tweet → free chronological sort
  No central coordinator needed — each server generates its own

Step 4 — Persist tweet
  Write to Cassandra (primary store)
  Also write to Redis (creator's own user timeline cache)
  Return 201 Created to client
  Creator sees their tweet immediately

Step 5 — Publish event
  Publish tweet_id to Kafka: tweet.created
  Response to client is NOT blocked on fan-out
  Fan-out starts after client receives 201
```

### Snowflake ID structure

```
 63                22 21          12 11           0
  ┌────────────────┬──────────────┬───────────────┐
  │   41 bits      │   10 bits    │   12 bits     │
  │   timestamp    │  machine ID  │   sequence    │
  └────────────────┴──────────────┴───────────────┘

41 bits timestamp = ~69 years from custom epoch (Twitter uses Nov 4, 2010)
10 bits machine   = 1,024 unique machines
12 bits sequence  = 4,096 IDs per millisecond per machine

Why Snowflake for tweets:
  1. Time-ordered — higher ID = newer tweet, no separate timestamp sort
  2. No central coordinator — each server generates its own IDs
  3. Decode the timestamp from the ID without a DB lookup
  4. Snowflake ID doubles as the pagination cursor
     "give me tweets with ID < {cursor}" = "give me tweets older than X"
```

### Rate limiting

```
Twitter's limits:
  2,400 tweets/day per account
  300 tweets per 3-hour window
  
Implementation: token bucket in Redis (same as rate limiter design)
  redis.get("rate:{userId}:tweets") → check bucket
  If within limit: redis.decr, allow tweet
  If over limit:   return 429 Too Many Requests

Rate limiting is also the first line of defence against fan-out storms.
A user who tweets 10,000 times per hour would cause 10,000 fan-out events.
Rate limiting prevents this before it reaches the fan-out queue.
```

---

## 6. Fan-out Pipeline

### Normal users (< 1M followers) — fan-out on write

```
Kafka consumer (fan-out worker) picks up tweet.created event
        │
        ▼
Fetch follower list from social graph service
  SELECT follower_id FROM followers WHERE followee_id = author_id
  (Cassandra — could be 200 rows for a normal user)
        │
        ▼
Filter inactive followers
  Skip users who haven't logged in for 30+ days
  Their cache was evicted — no point writing to it
  They'll get a cache rebuild on next login (fan-out on read at that point)
        │
        ▼
Batch write to Redis sorted sets (pipeline, not individual commands)
  For each active follower:
    ZADD timeline:{follower_id} {tweet_id} {tweet_id}
    ← score = tweet_id (Snowflake = chronological)
    ← value = tweet_id (store IDs, not full tweet content)
        │
        ▼
Cap timeline at 800 entries (remove oldest if over limit)
  ZREMRANGEBYRANK timeline:{follower_id} 0 -801
```

### Why store tweet IDs, not full tweet content

```
Store IDs (correct):
  Timeline is a sorted set of IDs → tiny (8 bytes per entry)
  Tweet content fetched separately at read time
  Edit or delete: update one record in Cassandra/Redis content cache
  No need to update potentially millions of timeline copies

Store full content (wrong):
  Tweet edited → must find and update in EVERY follower's timeline
  Elon Musk edits a tweet → 170M Redis updates
  Memory cost: 800 tweets × full content × 400M users = enormous
  Correctness nightmare
```

### Redis sorted set — the timeline data structure

```
Key:   timeline:{user_id}
Score: tweet_id (Snowflake — higher = newer = higher score)
Value: tweet_id

Operations:
  Fan-out write:   ZADD timeline:{uid} {tweet_id} {tweet_id}
  Cap at 800:      ZREMRANGEBYRANK timeline:{uid} 0 -801
  Read 20 newest:  ZREVRANGE timeline:{uid} 0 19
  Cursor paginate: ZREVRANGEBYSCORE timeline:{uid} ({cursor} -inf LIMIT 0 20

Memory:
  800 tweet IDs × 8 bytes × 400M users = 2.56 TB
  Redis cluster with 256GB per node = ~10 nodes
  Acceptable — Twitter actually runs something like this
```

---

## 7. Home Timeline Service

### Read flow — step by step

```
Step 1 — Client requests
  GET /api/v1/timeline/home?limit=20&cursor=
  Cursor = Snowflake ID of last seen tweet (empty = latest)

Step 2 — Check Redis
  ZREVRANGE timeline:{userId} 0 19         ← first page
  ZREVRANGEBYSCORE timeline:{userId} ({cursor} -inf LIMIT 0 20  ← subsequent
  Returns up to 20 tweet IDs in ~3ms

Step 3 — Hydrate tweet content
  Fetch full tweet objects for those 20 IDs
  Check tweet content cache (Redis) first
  Miss → Cassandra lookup
  Run all 20 fetches in parallel (Promise.all)
  ~10ms total

Step 4 — Merge celebrity tweets (hybrid model)
  For each celebrity the user follows:
    Fetch their latest 20 tweets from Cassandra directly
  Merge into the timeline, re-sort by tweet_id (chronological)
  ~20ms for typical user following a few celebrities

Step 5 — Return
  20 fully hydrated tweets
  next_cursor = lowest tweet_id in the response
  Total: ~50-80ms well within 200ms SLA
```

### Cursor-based pagination vs offset

```
Offset pagination (wrong for feeds):
  GET /timeline?page=5&limit=20
  SQL: SELECT ... LIMIT 20 OFFSET 100
  Problem 1: O(offset) — page 100 scans 2,000 rows to skip them
  Problem 2: new tweets arrive between page 1 and page 2 requests
             tweets shift — items appear twice or get skipped

Cursor-based pagination (correct):
  GET /timeline?cursor=1234567890&limit=20
  SQL: ZREVRANGEBYSCORE timeline:{uid} ({cursor} -inf LIMIT 0 20
  Stable: "give me tweets older than this ID" regardless of new arrivals
  O(log n): sorted set range query is logarithmic

Snowflake IDs as cursors:
  Decode the timestamp from the cursor without a DB lookup
  Know exactly "give me tweets before 2:34pm on April 12" from the ID alone
```

### Cache miss — user returning after inactivity

```
User timeline cache evicted after 30 days of inactivity.
On next login: Redis sorted set is empty.
Timeline service detects empty set → triggers cache rebuild:

  1. Fetch user's follow list (Cassandra following table)
  2. For each followee: fetch their last 20 tweets from Cassandra
  3. Merge, sort by tweet_id, insert top 800 into Redis sorted set
  4. Show loading state in UI during rebuild (2-5 seconds for 1,000 followees)

This is fan-out on read — but only for the cache rebuild.
Subsequent reads are served from the warm Redis cache.
```

### Timeline assembly for users following celebrities

```typescript
async function buildHomeFeed(userId: string): Promise<Tweet[]> {
  const followees = await socialGraph.getFollowing(userId);

  // Split: normal users vs celebrities
  const [normal, celebs] = partition(followees, f => f.followerCount >= 1_000_000);

  // Path 1: pre-computed timeline from Redis (normal users)
  const tweetIds = await redis.zrevrange(`timeline:${userId}`, 0, 99);
  const normalTweets = await hydrateTweets(tweetIds);

  // Path 2: fetch celebrities' tweets from Cassandra at read time
  const celebTweets = await Promise.all(
    celebs.map(celeb =>
      cassandra.query(
        'SELECT * FROM tweets WHERE user_id = ? ORDER BY id DESC LIMIT 20',
        [celeb.userId]
      )
    )
  );

  // Merge and re-sort by Snowflake ID (chronological)
  return [...normalTweets, ...celebTweets.flat()]
    .sort((a, b) => b.id - a.id)
    .slice(0, 20);
}
```

---

## 8. The Celebrity Problem

### Why celebrities break fan-out on write

```
Elon Musk: 170M followers
One tweet: 170M Redis ZADD operations
At 100K write ops/sec: 170M ÷ 100K = 1,700 seconds = 28 MINUTES to fan out
By the time fan-out completes, the tweet is cold

At busy times:
  MrBeast, Taylor Swift, Barack Obama all tweet within 5 minutes
  Fan-out queue depth grows faster than workers can drain it
  System is permanently behind — never catches up
```

### The hybrid solution

```
Threshold: users with ≥ 1M followers are "celebrities"
  Stored in a Redis set: SADD celebrities {userId}
  O(1) lookup: SISMEMBER celebrities {userId}
  Updated by background job when follower_count crosses threshold

Normal users (< 1M followers):
  Fan-out on write as described
  Tweet pushed to all active followers' Redis sorted sets immediately

Celebrity users (≥ 1M followers):
  NO fan-out on write
  Tweet stored in Cassandra under their user_id only
  At timeline read time: fetch their recent tweets directly from Cassandra
  Merge with the pre-built timeline from Redis

Effect:
  One Elon Musk tweet = 1 Cassandra write (not 170M Redis writes)
  Cost moves from write time to read time
  Read-time cost: 1 Cassandra query per celebrity the user follows
  Most users follow a handful of celebrities — bounded, acceptable cost
```

### Edge cases to mention proactively

```
User crosses 1M followers:
  Background job detects threshold crossing
  Add to celebrities Redis set
  Future tweets: no fan-out
  Existing followers' caches: kept until natural eviction (30 days inactivity)

Celebrity tweet latency:
  Celebrity tweets appear slightly slower in some timelines
  (fetched on-demand vs pre-pushed)
  Acceptable trade-off — the alternative is melting the fan-out workers

Celebrity who follows another celebrity:
  When requesting their own home timeline:
    Their pre-built timeline covers normal followees
    + On-demand Cassandra fetch for celebrity followees
  Bounded cost — even celebrities follow at most a few thousand people
```

---

## 9. Search and Trending

### Tweet search

```
Architecture: Elasticsearch with inverted index

Index contains per tweet:
  tweet_id, content (tokenised), author_id, created_at,
  hashtags, @mentions, URLs,
  engagement signals (like_count, retweet_count — periodically updated)

Ingestion: Kafka consumer indexes new tweets asynchronously
  ~1-5 second delay before searchable (eventually consistent)
  Real-time search is not required — 5 seconds is fine

Ranking: text relevance × engagement signals × recency
  Not computed in real time — pre-computed scoring factors
  Elasticsearch's BM25 for text relevance + custom scoring for engagement
```

### Trending topics

```
What "trending" means:
  NOT the hashtags with the most total usage ever
  BUT the hashtags with the highest VELOCITY of usage in the last hour
  #WorldCup trends because it's being used more NOW than 15 minutes ago

Implementation: sliding window aggregation (Kafka Streams or Flink)

  For each tweet consumed:
    Extract hashtags
    Increment counter in a 1-hour sliding window (5-minute granularity)

  Every 5 minutes:
    Compute top-50 hashtags by usage velocity
    Write to Redis: SET trending:{region}:{language} [list of hashtags]
    TTL: 5 minutes (refreshed by each computation)

Personalisation:
  Trending is partitioned by (region, language)
  Client sends locale headers
  Server returns the trending list for that shard
  A hashtag trending in Brazil may not trend in Japan
```

### Trending computation

```
stream
  .flatMap(tweet => extractHashtags(tweet.content))
  .groupByKey()
  .windowedBy(SlidingWindow.of(Duration.ofHours(1), Duration.ofMinutes(5)))
  .count()                              ← count per hashtag per window
  .toStream()
  .filter(count > TRENDING_THRESHOLD)   ← filter noise

Top-K selection: min-heap of size 50
  For each incoming (hashtag, count): if count > heap.min → replace
  Result: top-50 hashtags by velocity, O(log 50) per update

Write to Redis every 5 minutes:
  SETEX trending:{region}:{language} 300 [json array of top 50]
```

---

## 10. Database Schema

### Tweets (Cassandra)

```sql
-- Primary tweet store
-- Partition key: user_id — all tweets for one user in one partition
-- Clustering key: tweet_id DESC — newest tweets first on disk
-- Enables: SELECT ... WHERE user_id = ? ORDER BY id DESC LIMIT 20

CREATE TABLE tweets (
  user_id     UUID,
  tweet_id    BIGINT,         -- Snowflake ID
  content     TEXT,
  media_ids   LIST<UUID>,
  reply_to_id BIGINT,         -- null for top-level tweets
  retweet_of  BIGINT,         -- null for original tweets
  like_count  COUNTER,        -- Cassandra native counter type
  created_at  TIMESTAMP,
  PRIMARY KEY (user_id, tweet_id)
) WITH CLUSTERING ORDER BY (tweet_id DESC);
```

### Social graph (Cassandra — two tables)

```sql
-- "Who do I follow?" — used for: timeline rebuild, show following list
CREATE TABLE following (
  follower_id UUID,
  followee_id UUID,
  created_at  TIMESTAMP,
  PRIMARY KEY (follower_id, followee_id)
);

-- "Who follows me?" — used for: fan-out (push to all my followers)
CREATE TABLE followers (
  followee_id UUID,
  follower_id UUID,
  created_at  TIMESTAMP,
  PRIMARY KEY (followee_id, follower_id)
);
```

**Why two tables:** Cassandra reads are fastest when data is pre-arranged per query. Fan-out needs "who follows user A?" → `followers` table. Timeline rebuild needs "who does user A follow?" → `following` table. Denormalising into two tables is intentional — each table serves one query pattern with O(1) partition lookup.

### User metadata (PostgreSQL)

```sql
CREATE TABLE users (
  id              UUID        PRIMARY KEY,
  username        VARCHAR(50) UNIQUE NOT NULL,
  display_name    TEXT,
  follower_count  BIGINT      DEFAULT 0,
  following_count BIGINT      DEFAULT 0,
  is_celebrity    BOOLEAN     DEFAULT FALSE,  -- fan-out routing flag
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
-- is_celebrity set by background job when follower_count crosses 1M
-- follower_count updated via Kafka consumer on follow/unfollow events
```

### Like and view counts (Redis + async flush)

```
Same pattern as YouTube view counts:

On each like event:
  redis.incr("likes:{tweetId}")

Background job every 30 seconds:
  count = redis.getdel("likes:{tweetId}")
  db.execute("UPDATE tweets SET like_count = like_count + ? WHERE id = ?",
             count, tweetId)

Displayed as: "1.2K likes" not "1,247 likes"
Eventual consistency is correct here — it is a deliberate product decision
```

---

## 11. Trade-offs

### Fan-out strategies

```
Fan-out on write:
  ✓ Instant reads from pre-built cache
  ✓ Simple read path: just read the sorted set
  ✗ Write amplification for celebrities (170M writes per tweet)
  ✗ Fan-out queue permanently behind during celebrity tweet storms

Fan-out on read:
  ✓ O(1) writes — no amplification
  ✓ Celebrity tweets always fresh
  ✗ 2,000 DB reads per timeline request
  ✗ Does not scale under read load

Hybrid (correct):
  ✓ Write amplification bounded (max 1M writes per tweet for non-celebrities)
  ✓ Celebrity reads bounded (handful of Cassandra queries per timeline)
  ✗ Complex — two code paths, threshold is an operational knob
  ✗ Celebrity threshold needs tuning and monitoring

Always use hybrid. It is what every major social platform uses.
```

### Redis sorted set timeline cap — 800 entries

```
800 tweets:
  ✓ Bounded memory per user: 800 × 8B = 6.4KB per user
  ✓ 400M users × 6.4KB = 2.56TB — manageable Redis cluster
  ✗ Users who scroll past 800 tweets hit cache miss → Cassandra fallback

Larger cap:
  ✓ More history served from cache
  ✗ Memory scales linearly: 8,000 entries = 25.6TB — expensive

2,000+ tweets:
  Rarely worth it — users almost never scroll that far
  On cache miss: fall back to Cassandra with acceptable occasional latency
```

### Cassandra vs PostgreSQL for tweets

```
Cassandra for tweets:
  ✓ Write optimised LSM tree — handles 17,500 writes/sec trivially
  ✓ Perfect access pattern: always (user_id, tweet_id range)
  ✓ Linear horizontal scaling — add nodes for capacity
  ✗ No JOINs, no ad-hoc queries, eventual consistency

PostgreSQL for user metadata:
  ✓ ACID for profile updates, follow/unfollow
  ✓ Unique constraint on username
  ✓ Rich queries for admin and analytics
  ✓ Low write volume — user updates are rare

Never use PostgreSQL for tweets at Twitter's write volume.
Single PostgreSQL primary saturates at ~50K writes/sec.
17,500 peak tweet writes/sec × fan-out = tens of millions/sec — SQL cannot handle this.
```

### Cursor pagination vs offset

```
Offset: GET /timeline?page=5&limit=20
  ✗ O(offset): page 100 scans and skips 2,000 rows
  ✗ New tweets shift the window: item appears on page 1 AND page 2

Cursor (Snowflake ID): GET /timeline?cursor=1234567890&limit=20
  ✓ O(log n) sorted set range query
  ✓ Stable: new tweets at top don't affect older page positions
  ✓ Can decode timestamp from cursor without DB lookup
  Always use cursor-based pagination for feeds.
```

---

## 12. Interview Script

### Opening — state the problem before anything else

> "The central challenge in Twitter's design is the timeline fan-out problem, and there are two naive approaches — both fail. Pure fan-out on write: Elon Musk has 170 million followers. One tweet triggers 170 million Redis writes. At 100,000 writes per second that takes 28 minutes to fan out — the queue permanently falls behind. Pure fan-out on read: a user follows 2,000 accounts. One timeline request requires 2,000 DB reads. At 100,000 requests per second that's 200 million DB queries per second — the system collapses from the other direction. The correct answer is a hybrid — fan-out on write for normal users under 1 million followers, fan-out on read for celebrities above that threshold."

### The Redis sorted set

> "Each user has a Redis sorted set as their pre-built timeline. The score is the Snowflake tweet ID — because Snowflake IDs embed a timestamp in the high bits, a higher ID always means a newer tweet. We get chronological ordering for free without any sorting. We store tweet IDs, not full content — if a tweet is edited or deleted, we update one content cache record, not 170 million timeline copies. The sorted set is capped at 800 entries per user to bound memory."

### Snowflake IDs — always mention proactively

> "I'd use Snowflake IDs for tweets. 64-bit, timestamp in the high 41 bits, machine ID and sequence in the lower bits. This gives globally unique IDs without central coordination, natural chronological ordering — higher ID means newer tweet — and you can decode the timestamp from the ID without a DB lookup. Critically, the Snowflake ID doubles as my pagination cursor. Cursor-based pagination — 'give me tweets older than this ID' — is O(log n) and stable regardless of new tweets arriving. Offset pagination breaks on live feeds."

### Two social graph tables

> "For the social graph I need two Cassandra tables — not one. A following table partitioned by follower_id, for 'who do I follow?' which is needed when rebuilding a timeline. And a followers table partitioned by followee_id, for 'who follows me?' which is needed for fan-out. Cassandra is fastest when data is pre-arranged per query pattern. Having two tables is intentional denormalisation."

### Like and view counts

> "Like counts and view counts are never updated via direct SQL increment at this scale. Write contention on a single row with millions of concurrent updates would be catastrophic. I'd use Redis INCR per tweet, flushed to Cassandra in batches every 30 seconds. Displayed counts are eventually consistent — '1.2K likes' not '1,247 likes.' That approximation is a deliberate product decision, not a limitation."

---

## 13. Follow-up Probes and Answers

**"How do you handle a tweet going viral after it was posted?"**  
The tweet is already in Cassandra (single source of truth). Fan-out already ran for all followers at post time. New followers gained after the tweet was posted don't see it retroactively in their feed — they'd find it via search or trending. The system doesn't re-fan-out when follower count grows. For celebrities, the tweet was never fanned out at all — it's always fetched on-demand from Cassandra.

**"How do you handle tweet deletion?"**  
Soft delete in Cassandra — set `deleted = true`. Publish a delete event to Kafka. A delete consumer removes the tweet_id from all timeline sorted sets (expensive for popular tweets but bounded — only active users' caches matter). Tweet content cache TTL naturally expires. For celebrity tweets, no timeline cache to clean up — the deleted tweet stops appearing at read time since the Cassandra row is soft-deleted.

**"How do you design the social graph at scale?"**  
Two Cassandra tables: `following (follower_id → followee_id)` and `followers (followee_id → follower_id)`. Both required. For users with millions of followers, paginate the followers table during fan-out — don't load all 170M IDs at once. Process in batches of 1,000, dispatch Redis writes concurrently per batch.

**"How do you count likes accurately at scale?"**  
Redis INCR per tweet (O(1), no contention). Flush to Cassandra every 30 seconds via a background job. Displayed count is eventually consistent. Twitter shows approximate counts — this is product design, not a limitation. If exact counts are required (for billing, contests), use a distributed counter with quorum reads, but that is not the case for public-facing like counts.

**"What happens when the fan-out queue falls behind?"**  
Monitor queue depth and consumer lag as a key SLA metric. Auto-scale fan-out workers horizontally (Kafka consumer group — add consumers, partitions distribute). Shed load during extreme spikes: temporarily lower the celebrity threshold from 1M to 100K followers, moving more accounts to the read-time path. Resume fan-out for recently inactive users last — they're less time-sensitive.

**"How does the 'For You' algorithmic feed differ from the chronological home timeline?"**  
Chronological home timeline is what we designed above — tweets from people you follow in time order, served from Redis sorted set + Cassandra. Algorithmic 'For You' is a separate ML pipeline: candidate generation (collaborative filtering, content signals) + ranking model (predicted engagement score). The ranked list is pre-computed offline and served from a separate Redis key `algo_timeline:{userId}`. The two timelines coexist — the client switches between them.

---

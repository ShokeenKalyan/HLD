# LRU Cache — Complete Interview Reference

> **Type:** Coding + design question testing data structures, OOP, complexity analysis, and concurrency.  
> **Expectation:** Explain the data structure choice BEFORE coding. State complexity BEFORE being asked. Raise thread safety WITHOUT being prompted.  
> **Rule:** The interviewer is not testing whether you know LRU. They are testing whether you think like a senior engineer — trade-offs, edge cases, production concerns, without needing to be asked.

---

## Table of Contents

1. [Clarifying Questions — Ask These First](#1-clarifying-questions--ask-these-first)
2. [What is an LRU Cache](#2-what-is-an-lru-cache)
3. [Why HashMap and Doubly Linked List](#3-why-hashmap-and-doubly-linked-list)
4. [Data Structure — Visual Walkthrough](#4-data-structure--visual-walkthrough)
5. [Space and Time Complexity](#5-space-and-time-complexity)
6. [TypeScript Implementation — Single Threaded](#6-typescript-implementation--single-threaded)
7. [Thread Safety — The Problem](#7-thread-safety--the-problem)
8. [Thread Safety — Solution 1: Mutex Lock](#8-thread-safety--solution-1-mutex-lock)
9. [Thread Safety — Solution 2: Read-Write Lock](#9-thread-safety--solution-2-read-write-lock)
10. [Edge Cases](#10-edge-cases)
11. [TTL Support — Extension](#11-ttl-support--extension)
12. [Trade-offs and Alternatives](#12-trade-offs-and-alternatives)
13. [Interview Script](#13-interview-script)

---

## 1. Clarifying Questions — Ask These First

> **Say all of this before writing a single line of code.**

```
Scope:
  "Is this a basic LRU, or should it also support TTL — entries expiring
   after N seconds regardless of whether they were accessed?"

Return values:
  "What should get() return for a missing key — null, -1, or throw?"

Types:
  "Should key and value types be generic, or fixed to specific types?"

Concurrency — THE most important question:
  "Is this cache accessed by a single thread, or must it be thread-safe
   for concurrent reads and writes?"
  (The interviewer is waiting for you to ask this — raise it yourself)

Capacity edge case:
  "What should happen if capacity is 0 or negative — no-op, or throw?"

Answers for this design:
  Basic LRU (TTL mentioned as extension)
  Return null for missing key
  Generic <K, V> types
  Thread-safe required
  Throw on capacity <= 0
```

---

## 2. What is an LRU Cache

### The concept

```
LRU = Least Recently Used

A cache with a fixed capacity that evicts the LEAST RECENTLY USED
item when the cache is full and a new item needs to be inserted.

"Recently used" means: the most recently accessed OR inserted item.
Every get() and every put() counts as "using" the item.

Visual:

  Capacity = 3

  put(1, 'a')   ->  [1]
  put(2, 'b')   ->  [2, 1]       most recent on left
  put(3, 'c')   ->  [3, 2, 1]
  get(1)        ->  [1, 3, 2]    1 moved to front (just accessed)
  put(4, 'd')   ->  [4, 1, 3]    2 evicted (was least recently used)
  get(2)        ->  null          2 was evicted
```

### Where LRU caches are used in production

```
Browser cache:      Most recently visited pages stay in memory
DNS cache:          Most recently resolved hostnames
Database buffer:    Most recently accessed pages stay in RAM
CDN edge cache:     Most recently requested content at edge nodes
API response cache: Most recently requested expensive query results
Session store:      Active sessions stay warm, idle ones evicted
Our PV systems:     Could cache rule evaluation results per workflow state
```

### The requirement

```
LRUCache<K, V>(capacity: number)

  get(key: K): V | null
    Return value if key exists, null if not
    Mark the key as most recently used

  put(key: K, value: V): void
    Insert or update key-value pair
    Mark as most recently used
    If cache is at capacity and key is new: evict the least recently used key

Both operations MUST run in O(1) time.
```

---

## 3. Why HashMap and Doubly Linked List

### The key insight — explain this before coding

```
We need two things simultaneously:
  1. O(1) lookup:  given a key, find its value instantly
  2. O(1) ordering: track which item was used most/least recently,
                    and update that ordering on every access

No single data structure gives us both. We combine two:

HashMap alone:
  O(1) get and put — perfect for lookup
  BUT: no ordering information — cannot find the LRU item efficiently

Array or Linked List alone:
  Maintains order — can track most/least recently used
  BUT: O(n) lookup — must scan to find a key
       O(n) removal — must scan to find and remove a node

Doubly Linked List + HashMap together:
  HashMap:  key -> Node    O(1) lookup of any node
  DLL:      maintains order (head = MRU, tail = LRU)
            O(1) remove any node (because we have prev AND next pointers)
            O(1) add to head

  Combined: O(1) get (HashMap lookup + move node to head)
            O(1) put (create node at head + HashMap insert)
            O(1) evict (remove tail node + delete from HashMap)
```

### Why DOUBLY linked (not singly linked)?

```
To remove a node from a linked list in O(1), you need:
  node.prev.next = node.next   <- need the PREVIOUS node
  node.next.prev = node.prev   <- need the NEXT node

With a SINGLY linked list:
  You can traverse forward but cannot access node.prev
  To remove a node, you must traverse from the head to find the node before it
  That makes removal O(n) — not acceptable

With a DOUBLY linked list:
  Every node has both prev and next pointers
  Given any node, you can remove it in O(1) — no traversal needed
  This is why doubly linked list is the correct choice here

The trade-off:
  Extra memory: each node needs an additional prev pointer
  Worth it: the memory overhead is O(n) total, which we already pay for capacity
```

### Sentinel nodes — why we use dummy head and tail

```
Without sentinel nodes, every operation needs null checks:
  if (this.head === null) { ... }
  if (node.prev === null) { ... }
  if (node.next === null) { ... }

With sentinel (dummy) head and tail nodes:
  head <-> [real nodes] <-> tail

  head and tail always exist — they never hold real data
  Real nodes are always between head and tail
  NO null checks needed in add/remove operations
  Simpler, cleaner, less error-prone code

  head.next = first real node (or tail if empty)
  tail.prev = last real node (or head if empty)
  head.prev = null (never accessed)
  tail.next = null (never accessed)
```

---

## 4. Data Structure — Visual Walkthrough

```
Capacity = 3

Initial state (empty):
  HashMap: {}
  DLL:     head <-> tail

─────────────────────────────────────────────
put(1, 'a')
  Create node(1, 'a')
  Add to front (after head)
  Add to HashMap: {1: node1}

  HashMap: { 1: node1 }
  DLL:     head <-> [1,'a'] <-> tail
                    (MRU)  (LRU)

─────────────────────────────────────────────
put(2, 'b')
  Create node(2, 'b'), add to front
  HashMap: { 1: node1, 2: node2 }
  DLL:     head <-> [2,'b'] <-> [1,'a'] <-> tail
                    (MRU)               (LRU)

─────────────────────────────────────────────
put(3, 'c')
  Create node(3, 'c'), add to front
  HashMap: { 1: node1, 2: node2, 3: node3 }
  DLL:     head <-> [3,'c'] <-> [2,'b'] <-> [1,'a'] <-> tail
                    (MRU)                           (LRU)
  Cache is now FULL (size = capacity = 3)

─────────────────────────────────────────────
get(1)     <- access key 1
  Step 1: HashMap.get(1) -> node1     O(1)
  Step 2: Remove node1 from its current position (tail side)
          node1.prev.next = node1.next      O(1)
          node1.next.prev = node1.prev      O(1)
  Step 3: Add node1 to front (after head)    O(1)

  HashMap: { 1: node1, 2: node2, 3: node3 }
  DLL:     head <-> [1,'a'] <-> [3,'c'] <-> [2,'b'] <-> tail
                    (MRU)                           (LRU)
  Returns: 'a'

─────────────────────────────────────────────
put(4, 'd')   <- cache is full, must evict
  Step 1: Cache is full (size == capacity)
  Step 2: Evict LRU = tail.prev = node2 (key=2)
          Remove node2 from DLL: O(1)
          Delete key 2 from HashMap: O(1)
  Step 3: Create node(4, 'd'), add to front
          Add to HashMap: {4: node4}

  HashMap: { 1: node1, 3: node3, 4: node4 }
  DLL:     head <-> [4,'d'] <-> [1,'a'] <-> [3,'c'] <-> tail
                    (MRU)                           (LRU)

─────────────────────────────────────────────
get(2)
  HashMap.get(2) -> undefined (was evicted)
  Returns: null

─────────────────────────────────────────────
put(1, 'UPDATED')   <- key exists: update + move to front
  Step 1: HashMap.get(1) -> node1 (exists!)
  Step 2: Update node1.value = 'UPDATED'
  Step 3: Remove node1 from current position
  Step 4: Add node1 to front

  HashMap: { 1: node1, 3: node3, 4: node4 }
  DLL:     head <-> [1,'UPDATED'] <-> [4,'d'] <-> [3,'c'] <-> tail
```

---

## 5. Space and Time Complexity

### Time complexity

```
Operation   Complexity   Reason
──────────────────────────────────────────────────────────────────
get(key)    O(1)         HashMap lookup: O(1)
                         Move node to head: O(1) with doubly linked list
                         (remove from current position: O(1) via prev/next
                          add to front: O(1) via head sentinel)

put(key)    O(1)         HashMap lookup for existence check: O(1)
  new key:               Create node: O(1)
                         Add to head: O(1)
                         HashMap insert: O(1)
                         Evict LRU (if full): O(1) remove tail.prev
                                              O(1) HashMap delete

  existing key:          Update value: O(1)
                         Move to head: O(1)

eviction    O(1)         Remove tail.prev from DLL: O(1)
                         Delete from HashMap: O(1)

Why O(1) and not O(n):
  The HashMap gives us direct access to the node — no traversal
  The doubly linked list lets us remove any node in O(1) — no traversal
  Sentinel nodes eliminate null checks and boundary conditions
  Every operation is a fixed number of pointer assignments
```

### Space complexity

```
O(capacity) — exactly

  HashMap:  capacity entries, each storing a reference to a node
  DLL:      capacity nodes, each with key, value, prev, next

  Plus 2 sentinel nodes (head and tail) — O(1) constant overhead

  Total: O(capacity) for both the HashMap and the DLL
         Each entry is stored exactly once — no duplication

Memory per node:
  key:   reference (8 bytes on 64-bit)
  value: reference (8 bytes)
  prev:  reference (8 bytes)
  next:  reference (8 bytes)
  Total: ~32 bytes per node (plus object header overhead in V8)

  At capacity 1,000,000 nodes: ~32 MB — very manageable
```

---

## 6. TypeScript Implementation — Single Threaded

```typescript
// ─── NODE ─────────────────────────────────────────────────────────────────────

class LRUNode<K, V> {
  key:   K;
  value: V;
  prev:  LRUNode<K, V> | null = null;
  next:  LRUNode<K, V> | null = null;

  constructor(key: K, value: V) {
    this.key   = key;
    this.value = value;
  }
}

// ─── LRU CACHE ────────────────────────────────────────────────────────────────

class LRUCache<K, V> {
  readonly #capacity: number;
  #size:              number = 0;
  readonly #map:      Map<K, LRUNode<K, V>>;

  // Sentinel nodes — always exist, never hold real data
  // head.next = most recently used
  // tail.prev = least recently used
  readonly #head: LRUNode<K, V>;
  readonly #tail: LRUNode<K, V>;

  constructor(capacity: number) {
    if (!Number.isInteger(capacity) || capacity <= 0) {
      throw new Error(`Capacity must be a positive integer, got: ${capacity}`);
    }
    this.#capacity = capacity;
    this.#map      = new Map<K, LRUNode<K, V>>();

    // Initialise sentinel nodes and link them
    // Using null! casts because sentinels don't carry real keys/values
    this.#head = new LRUNode<K, V>(null!, null!);
    this.#tail = new LRUNode<K, V>(null!, null!);
    this.#head.next = this.#tail;
    this.#tail.prev = this.#head;
  }

  // ── GET ───────────────────────────────────────────────────────────────────
  // O(1) — HashMap lookup + move node to head
  get(key: K): V | null {
    const node = this.#map.get(key);
    if (!node) return null;

    // Move to front: this key was just accessed
    this.#moveToHead(node);
    return node.value;
  }

  // ── PUT ───────────────────────────────────────────────────────────────────
  // O(1) — update existing node or create new, evict tail if full
  put(key: K, value: V): void {
    const existing = this.#map.get(key);

    if (existing) {
      // Key exists: update value, move to front
      existing.value = value;
      this.#moveToHead(existing);
      return;
    }

    // New key: create node and add to front
    const node = new LRUNode<K, V>(key, value);
    this.#map.set(key, node);
    this.#addToHead(node);
    this.#size++;

    // If over capacity: evict the least recently used (tail.prev)
    if (this.#size > this.#capacity) {
      const evicted = this.#removeTail();
      this.#map.delete(evicted.key);
      this.#size--;
    }
  }

  // ── PUBLIC ACCESSORS ──────────────────────────────────────────────────────

  get size(): number     { return this.#size; }
  get capacity(): number { return this.#capacity; }

  has(key: K): boolean { return this.#map.has(key); }

  // Peek: return value WITHOUT marking as recently used
  // Useful for inspection without affecting eviction order
  peek(key: K): V | null {
    return this.#map.get(key)?.value ?? null;
  }

  delete(key: K): boolean {
    const node = this.#map.get(key);
    if (!node) return false;
    this.#removeNode(node);
    this.#map.delete(key);
    this.#size--;
    return true;
  }

  clear(): void {
    this.#map.clear();
    this.#head.next = this.#tail;
    this.#tail.prev = this.#head;
    this.#size = 0;
  }

  // Returns keys from most recently used to least recently used
  keys(): K[] {
    const result: K[] = [];
    let current = this.#head.next;
    while (current !== this.#tail) {
      result.push(current!.key);
      current = current!.next;
    }
    return result;
  }

  // ── PRIVATE DLL HELPERS ───────────────────────────────────────────────────
  // These are the O(1) building blocks — all pointer manipulation

  // Add a node directly after the head sentinel (= mark as most recently used)
  #addToHead(node: LRUNode<K, V>): void {
    node.prev       = this.#head;
    node.next       = this.#head.next;
    this.#head.next!.prev = node;
    this.#head.next       = node;
  }

  // Remove a node from its current position in the DLL
  #removeNode(node: LRUNode<K, V>): void {
    node.prev!.next = node.next;
    node.next!.prev = node.prev;
  }

  // Move an existing node to the head (= mark as most recently used)
  #moveToHead(node: LRUNode<K, V>): void {
    this.#removeNode(node);
    this.#addToHead(node);
  }

  // Remove and return the tail sentinel's predecessor (= least recently used)
  #removeTail(): LRUNode<K, V> {
    const lru = this.#tail.prev!;  // the actual LRU node (not the sentinel)
    this.#removeNode(lru);
    return lru;
  }
}
```

---

## 7. Thread Safety — The Problem

### Why single-threaded implementation breaks under concurrency

```
In Node.js, JavaScript is single-threaded — there is only one call stack.
Pure synchronous operations like the LRU cache above are naturally safe
because only one piece of code runs at a time.

BUT: in a real production system, the LRU cache operations are often
interleaved with async operations (DB calls, network calls), and in
environments like Java, Go, or even Node.js Worker Threads, multiple
threads genuinely run concurrently.

The question is specifically asking about concurrent access.
The interviewer wants to see that you think about this.
```

### The race condition — show this explicitly

```
Scenario: two threads (or async operations) access the cache simultaneously.

Thread A: get(key_X) -> node found in HashMap
Thread B: put(key_Y) -> cache full, evicts tail -> happens to evict key_X
Thread A: now tries to moveToHead(node_X) -> node_X is already removed!
          node.prev is null -> NULL POINTER ERROR or CORRUPTED DLL

Another scenario:

Thread A: put(key_new) -> HashMap.size = capacity -> decides to evict
Thread B: put(key_new2) -> HashMap.size = capacity -> also decides to evict
Both evict -> two nodes removed but only one needed
Cache size drops below capacity by 2 instead of 1 -> subtle state corruption

Another scenario:

Thread A: #addToHead(nodeA)
  head.next.prev = nodeA           <- step 1
  [Thread B runs here: also addToHead(nodeB), overwrites head.next]
Thread A continues:
  head.next = nodeA                <- step 2 (head.next is now nodeA)
  But nodeB was supposed to be inserted! It is now orphaned.
  The DLL is corrupted — a node is lost.

Result: data loss, stale reads, memory leaks, or crashes.
These bugs are intermittent and extremely hard to reproduce and debug.
```

---

## 8. Thread Safety — Solution 1: Mutex Lock

### The concept

```
A Mutex (Mutual Exclusion lock) ensures only ONE thread can execute
the protected code at a time. All other threads WAIT.

Simple, correct, easy to reason about.

get():
  Acquire lock
  Read from map + move node to head
  Release lock

put():
  Acquire lock
  Update/insert node + possibly evict
  Release lock

Pros:
  Simple — one lock for everything
  Correct — no race conditions possible
  Easy to verify

Cons:
  Readers block other readers (get() blocks other get() calls)
  At high read concurrency: all readers queue up behind one lock
  Under heavy read load: significant performance bottleneck
```

### Implementation in Node.js with Worker Threads

```typescript
// In Node.js, true parallelism requires Worker Threads + SharedArrayBuffer
// For the interview: explain the concept and show the pattern

// Async mutex — simulates lock behaviour for async Node.js operations
class AsyncMutex {
  #queue:  Array<() => void> = [];
  #locked: boolean           = false;

  async acquire(): Promise<() => void> {
    return new Promise<() => void>(resolve => {
      const tryAcquire = () => {
        if (!this.#locked) {
          this.#locked = true;
          // Return the release function to the caller
          resolve(() => {
            this.#locked = false;
            // Wake up the next waiter in the queue
            if (this.#queue.length > 0) {
              const next = this.#queue.shift()!;
              next();
            }
          });
        } else {
          // Lock is held — queue this attempt
          this.#queue.push(tryAcquire);
        }
      };
      tryAcquire();
    });
  }
}

// ─── THREAD-SAFE LRU CACHE WITH MUTEX ─────────────────────────────────────────

class ThreadSafeLRUCacheMutex<K, V> {
  readonly #cache: LRUCache<K, V>;
  readonly #mutex: AsyncMutex;

  constructor(capacity: number) {
    this.#cache = new LRUCache<K, V>(capacity);
    this.#mutex = new AsyncMutex();
  }

  async get(key: K): Promise<V | null> {
    const release = await this.#mutex.acquire();
    try {
      return this.#cache.get(key);
    } finally {
      release();  // ALWAYS release in finally — never skip on error
    }
  }

  async put(key: K, value: V): Promise<void> {
    const release = await this.#mutex.acquire();
    try {
      this.#cache.put(key, value);
    } finally {
      release();
    }
  }

  async delete(key: K): Promise<boolean> {
    const release = await this.#mutex.acquire();
    try {
      return this.#cache.delete(key);
    } finally {
      release();
    }
  }

  get size(): number     { return this.#cache.size; }
  get capacity(): number { return this.#cache.capacity; }
}
```

---

## 9. Thread Safety — Solution 2: Read-Write Lock

### The concept

```
An improvement over mutex: separate locks for readers and writers.

Rules:
  Multiple readers CAN read simultaneously (reads don't conflict)
  A writer needs EXCLUSIVE access (no readers or other writers)

When a write happens:
  Wait for all current readers to finish
  Block new readers from starting
  Perform the write
  Allow readers again

When a read happens:
  If no writer is active or waiting: proceed immediately
  If a writer is active: wait

Read-Write Lock is better when:
  Reads are much more frequent than writes
  (e.g., 95% reads, 5% writes)

For LRU cache specifically:
  PROBLEM: every get() is also a WRITE operation (moves node to head)
  A pure read lock for get() is incorrect — get() mutates the DLL
  So for strict LRU: read-write lock doesn't help as much
  (Every operation modifies the DLL)

  However: if you use a read-write lock and accept that get() uses
  a write lock, the benefit is only for peek() (true read-only access)

  Or: use a segmented/striped approach (see Section 12 trade-offs)

The honest interview answer:
  "Read-write lock theoretically improves concurrency for caches
   where reads don't mutate state. In a strict LRU cache, every get()
   updates the order, so it needs a write lock too.
   If we relax the LRU requirement slightly — e.g., only update order
   every N accesses — then get() can use a read lock, which
   dramatically improves throughput under read-heavy workloads."
```

### Implementation

```typescript
class ReadWriteLock {
  #readers:       number  = 0;  // count of active readers
  #writeQueue:    Array<() => void> = [];
  #readQueue:     Array<() => void> = [];
  #writing:       boolean = false;

  // Acquire read lock — blocks if a writer is active
  async acquireRead(): Promise<() => void> {
    return new Promise(resolve => {
      const tryRead = () => {
        if (!this.#writing && this.#writeQueue.length === 0) {
          this.#readers++;
          resolve(() => {
            this.#readers--;
            // If no more readers and a writer is waiting: wake it up
            if (this.#readers === 0 && this.#writeQueue.length > 0) {
              const next = this.#writeQueue.shift()!;
              next();
            }
          });
        } else {
          this.#readQueue.push(tryRead);
        }
      };
      tryRead();
    });
  }

  // Acquire write lock — blocks if any reader OR writer is active
  async acquireWrite(): Promise<() => void> {
    return new Promise(resolve => {
      const tryWrite = () => {
        if (!this.#writing && this.#readers === 0) {
          this.#writing = true;
          resolve(() => {
            this.#writing = false;
            // Wake up waiting writers first, then readers
            if (this.#writeQueue.length > 0) {
              const next = this.#writeQueue.shift()!;
              next();
            } else {
              // Release all waiting readers simultaneously
              const waiting = [...this.#readQueue];
              this.#readQueue = [];
              waiting.forEach(fn => fn());
            }
          });
        } else {
          this.#writeQueue.push(tryWrite);
        }
      };
      tryWrite();
    });
  }
}

// ─── THREAD-SAFE LRU WITH READ-WRITE LOCK ─────────────────────────────────────

class ThreadSafeLRUCacheRWLock<K, V> {
  readonly #cache:  LRUCache<K, V>;
  readonly #rwLock: ReadWriteLock;

  constructor(capacity: number) {
    this.#cache  = new LRUCache<K, V>(capacity);
    this.#rwLock = new ReadWriteLock();
  }

  // get() uses WRITE lock — LRU requires updating order on every access
  // If you relax to "approximate LRU", you can use read lock here
  async get(key: K): Promise<V | null> {
    const release = await this.#rwLock.acquireWrite();
    try {
      return this.#cache.get(key);
    } finally {
      release();
    }
  }

  // peek() uses READ lock — does NOT update order
  // Multiple threads can peek simultaneously
  async peek(key: K): Promise<V | null> {
    const release = await this.#rwLock.acquireRead();
    try {
      return this.#cache.peek(key);
    } finally {
      release();
    }
  }

  // put() uses WRITE lock — modifies structure
  async put(key: K, value: V): Promise<void> {
    const release = await this.#rwLock.acquireWrite();
    try {
      this.#cache.put(key, value);
    } finally {
      release();
    }
  }
}
```

### Mutex vs Read-Write Lock comparison

```
                   Mutex             Read-Write Lock
─────────────────────────────────────────────────────────────────
Concept:           One lock for all  Separate read/write locks
Implementation:    Simple            Complex
get() uses:        Write lock        Write lock (LRU mutates DLL)
peek() uses:       Write lock        Read lock (multiple concurrent)
put() uses:        Write lock        Write lock
Concurrent reads:  Blocked           Allowed (for peek() only)
Best for:          Equal R/W ratio   Read-heavy with peek pattern
Performance gain:  Baseline          Higher only if peek is common
Correctness risk:  Low               Medium (more state to manage)

Honest answer for LRU interview:
  For a STRICT LRU cache: mutex is correct and simpler.
  For an APPROXIMATE LRU or a cache where reads don't update order:
  read-write lock gives meaningful throughput improvement.
  At massive scale (millions of requests/sec): striped locks are better
  (segment the cache into N shards, each with its own mutex).
```

---

## 10. Edge Cases

### Every edge case you must handle

```typescript
// ── EDGE CASE 1: capacity = 0 or negative ────────────────────────────────────

const cache = new LRUCache<string, number>(0);
// throws: "Capacity must be a positive integer, got: 0"

// Why throw: a cache with zero capacity is semantically meaningless
// put() would immediately evict the just-inserted item
// Silently doing nothing would hide bugs in the caller

// ── EDGE CASE 2: get on missing key ──────────────────────────────────────────

const cache = new LRUCache<string, number>(3);
cache.get('nonexistent');
// returns: null
// does NOT throw — a miss is expected behaviour, not an error

// ── EDGE CASE 3: put when key already exists — update + move to head ─────────

cache.put('a', 1);
cache.put('b', 2);
cache.put('c', 3);
// DLL: c <-> b <-> a (a is LRU)

cache.put('a', 99);  // a already exists
// DLL: a <-> c <-> b (a is now MRU, b is now LRU)
// cache.size is still 3, NOT 4 — no eviction happens
cache.get('a');  // returns 99 (updated value)

// ── EDGE CASE 4: capacity = 1 ─────────────────────────────────────────────────

const tiny = new LRUCache<string, string>(1);
tiny.put('x', 'X');
tiny.put('y', 'Y');  // x evicted immediately
tiny.get('x');       // null — already evicted
tiny.get('y');       // 'Y'

// ── EDGE CASE 5: eviction order correctness ───────────────────────────────────

const c = new LRUCache<number, string>(3);
c.put(1, 'a');  // [1]
c.put(2, 'b');  // [2,1]
c.put(3, 'c');  // [3,2,1]
c.get(1);       // [1,3,2]  <- 1 moved to front
c.put(4, 'd'); // [4,1,3]  <- 2 evicted (was LRU), not 1
c.get(2);      // null — 2 was evicted
c.get(3);      // 'c' — 3 was not evicted

// ── EDGE CASE 6: put the same key repeatedly ─────────────────────────────────

c.put('key', 1);
c.put('key', 2);
c.put('key', 3);
// Only one entry for 'key', value = 3
// cache.size = 1 (not 3)

// ── EDGE CASE 7: get immediately after put ────────────────────────────────────

c.put('a', 1);
c.get('a');   // returns 1 — MRU after get
// DLL: [a] — nothing changes since a was already MRU

// ── EDGE CASE 8: delete a key manually ───────────────────────────────────────

c.put('x', 10);
c.delete('x');   // true
c.get('x');      // null
c.delete('x');   // false (already deleted)

// ── EDGE CASE 9: stress test — insert, access, verify eviction order ──────────

const stress = new LRUCache<number, number>(3);
stress.put(1, 1);
stress.put(2, 2);
stress.put(3, 3);
stress.put(1, 1);  // 1 moves to MRU
stress.put(4, 4);  // 2 evicted (3 was MRU before put(1,1))
                   // wait — let's trace this:
                   // After put(1,1): [1,3,2], LRU=2
                   // put(4,4): evict LRU=2, result: [4,1,3]
console.assert(stress.get(4) === 4);
console.assert(stress.get(1) === 1);
console.assert(stress.get(3) === 3);
console.assert(stress.get(2) === null);  // 2 was evicted

// ── EDGE CASE 10: clear and reuse ────────────────────────────────────────────

stress.clear();
console.assert(stress.size === 0);
stress.put(99, 'after clear');
console.assert(stress.get(99) === 'after clear');
```

---

## 11. TTL Support — Extension

```
The interviewer may ask: "How would you add TTL support?"

TTL = Time To Live: an entry is automatically evicted after N milliseconds,
regardless of whether it was recently used.

Combined policy: evict if (LRU AND capacity full) OR (TTL expired)
```

### Implementation approach

```typescript
interface CacheEntry<V> {
  value:     V;
  expiresAt: number;  // Date.now() + ttlMs at insertion
}

class LRUCacheWithTTL<K, V> {
  readonly #cache:       LRUCache<K, CacheEntry<V>>;
  readonly #defaultTTL?: number;  // optional default TTL in ms

  constructor(capacity: number, defaultTTLMs?: number) {
    this.#cache      = new LRUCache<K, CacheEntry<V>>(capacity);
    this.#defaultTTL = defaultTTLMs;
  }

  get(key: K): V | null {
    const entry = this.#cache.get(key);  // updates LRU order
    if (!entry) return null;

    // Check if TTL has expired
    if (Date.now() > entry.expiresAt) {
      this.#cache.delete(key);  // lazy eviction on read
      return null;
    }

    return entry.value;
  }

  put(key: K, value: V, ttlMs?: number): void {
    const ttl = ttlMs ?? this.#defaultTTL;
    if (ttl === undefined) throw new Error('TTL required — no default configured');

    this.#cache.put(key, {
      value,
      expiresAt: Date.now() + ttl,
    });
  }

  // Optional: background cleanup job to proactively remove expired entries
  // Without this, expired entries only removed on get() — they still occupy space
  startCleanup(intervalMs: number): NodeJS.Timeout {
    return setInterval(() => {
      const now = Date.now();
      for (const key of this.#cache.keys()) {
        const entry = this.#cache.peek(key);  // peek — don't update LRU order
        if (entry && now > entry.expiresAt) {
          this.#cache.delete(key);
        }
      }
    }, intervalMs);
  }
}
```

### Trade-offs for TTL

```
Lazy eviction (on get()):
  Pros: simple, no background thread needed
  Cons: expired entries still consume space until they're accessed
        In worst case: cache fills with expired entries, new inserts
        evict recently-used-but-not-expired entries first

Active eviction (background job):
  Pros: space is reclaimed promptly
  Cons: background job adds complexity, slight CPU overhead
        Race condition: job might try to delete a key that was
        just accessed by another thread (needs synchronisation)

Min-heap approach (most efficient):
  Maintain a min-heap ordered by expiresAt
  Background job only needs to check the heap's minimum
  If min.expiresAt <= now: pop and delete
  O(log n) per eviction but only processes truly expired entries
  Used by Redis for its TTL eviction
```

---

## 12. Trade-offs and Alternatives

### Design decisions in this implementation

```
Decision              Choice                Trade-off
──────────────────────────────────────────────────────────────────────────────
Primary data          HashMap +             Optimal O(1) for both get and put.
structure             Doubly Linked List    Memory overhead: two pointers per
                                            node (prev + next). Justified.

Sentinel nodes        Yes (dummy head/tail) Eliminates null checks in all DLL
                                            operations. Slight memory overhead
                                            (2 extra nodes). Always worth it.

Key type              Generic <K>           Flexible. Requires K to work as
                                            a Map key (objects compared by
                                            reference by default in JS).

Null for missing      Yes                   Simple API. Some prefer -1 (LeetCode
key return                                  convention) or throwing an error.
                                            Null is most Typescript-idiomatic.

Capacity = 0          Throw                 Fail fast. A 0-capacity cache is
                                            a programming error, not a runtime
                                            condition to silently handle.

Thread safety         Mutex (primary)       Simpler and correct. Read-write lock
                      RW Lock (extension)   only improves throughput if peek()
                                            (non-mutating reads) is common.
```

### Alternative implementations

```
JavaScript Map (insertion-ordered):
  Map in JS preserves insertion order
  Can use: delete + re-insert to move to "end" (most recent)
  LRU = Map.keys().next().value (first key = oldest insertion)

  const cache = new Map();
  function get(key) {
    if (!cache.has(key)) return null;
    const value = cache.get(key);
    cache.delete(key);
    cache.set(key, value);  // re-insert = moves to end (MRU)
    return value;
  }
  function put(key, value) {
    if (cache.has(key)) cache.delete(key);
    cache.set(key, value);
    if (cache.size > capacity) {
      cache.delete(cache.keys().next().value);  // delete first key (LRU)
    }
  }

  Pros: extremely simple (~10 lines), no custom DLL needed
  Cons: cache.keys().next() iterates the Map iterator — O(1) in V8
        but less explicit about the data structure
        Harder to extend (e.g., adding peek without updating order)
  When to use: in interviews where you want to show you know the JS API
               but the interviewer didn't ask to implement from scratch

Custom DLL (what we built):
  Pros: explicit, shows understanding of the data structure
        Easier to extend (TTL, approximate LRU, stats)
  Cons: more code — but this is what a senior engineer writes

Striped/Segmented Cache (at scale):
  Divide the cache into N independent segments (like ConcurrentHashMap in Java)
  Each segment has its own mutex
  get(key): hash(key) % N -> pick segment -> acquire that segment's lock
  Only 1/N of threads contend for any given lock
  N=16 segments: 16x reduction in lock contention
  Trade-off: each segment has capacity/N slots — slightly less efficient

  class StripedLRUCache<K, V> {
    readonly #segments: ThreadSafeLRUCacheMutex<K, V>[];
    readonly #n: number;

    constructor(capacity: number, stripes = 16) {
      this.#n        = stripes;
      this.#segments = Array.from(
        { length: stripes },
        () => new ThreadSafeLRUCacheMutex<K, V>(Math.ceil(capacity / stripes)),
      );
    }

    #getSegment(key: K): ThreadSafeLRUCacheMutex<K, V> {
      const hash = String(key).split('').reduce((acc, c) => acc + c.charCodeAt(0), 0);
      return this.#segments[hash % this.#n];
    }

    async get(key: K): Promise<V | null> { return this.#getSegment(key).get(key); }
    async put(key: K, value: V): Promise<void> { return this.#getSegment(key).put(key, value); }
  }
```

---

## 13. Interview Script

### Opening — clarifying questions first

> "Before I start coding, let me ask a few questions. Do we need TTL support, or just capacity-based eviction? What should get return for a missing key — null or throw? Should the key and value types be generic? And — the most important question — is this cache accessed by a single thread, or does it need to be thread-safe for concurrent access?"

### Explain data structure BEFORE coding

> "Given those requirements, let me explain the data structure choice before I write any code, because this decision drives everything else.

> We need two things: O(1) lookup by key, and O(1) ordering so we can always find and update the least recently used item. No single data structure gives us both. A HashMap alone gives us O(1) lookup but no ordering. A linked list gives us ordering but O(n) lookup. So we combine them: a HashMap maps each key directly to its node in a doubly linked list. The doubly linked list maintains the order — the head is the most recently used, the tail is the least recently used. When we get or put, we move the node to the head in O(1) because we have both prev and next pointers. When the cache is full, we evict the tail.prev node in O(1).

> I am using a doubly linked list specifically because to remove a node from a list in O(1), you need access to the node before it. A singly linked list would require O(n) traversal to find the predecessor. The doubly linked list has a prev pointer on every node, so removal is always O(1).

> I will also use two sentinel nodes — a dummy head and dummy tail — so I never have to write null checks inside the add and remove operations. This makes the code much cleaner."

### State complexity proactively

> "Time complexity: both get and put are O(1). The HashMap gives us O(1) node lookup. The doubly linked list gives us O(1) move-to-head and O(1) evict-from-tail. Space complexity: O(capacity) — we store exactly one node per cached entry."

### Raise thread safety without being asked

> "One thing I want to raise before finishing — if this cache is accessed by multiple threads concurrently, the implementation I just wrote is not thread-safe. Two concurrent operations can corrupt the doubly linked list — for example, if two threads both try to add to the head simultaneously, they can both overwrite head.next and lose one of the nodes. To fix this, I would wrap all get and put operations in a mutex lock. For read-heavy workloads, a read-write lock is better — but strictly speaking, in an LRU cache, even get is a write operation because it updates the ordering. So a simple mutex is the correct and simpler choice here."

### Edge case walkthrough

> "Edge cases I've handled: capacity zero or negative throws immediately in the constructor — a zero-capacity cache is a programming error. Get on a missing key returns null. Put on an existing key updates the value and moves it to the head — no eviction, no size change. The most subtle one is concurrent put of the same key — both threads must not insert two nodes for the same key. The mutex prevents this."

---

*Generic LRU Cache — Complete Interview Reference*
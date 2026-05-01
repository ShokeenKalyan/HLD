# System Design — Order Placement API (E-Commerce, Amazon Scale)

> **Type:** Mixed coding + system design.  
> **Scale:** Amazon-scale — millions of orders per day.  
> **Interview focus:** Clarifying questions first, API design, data model, edge cases, concurrency, TypeScript implementation.

---

## Table of Contents

1. [Clarifying Questions — Ask These First](#1-clarifying-questions--ask-these-first)
2. [Requirements](#2-requirements)
3. [Capacity Estimation](#3-capacity-estimation)
4. [High-Level Architecture](#4-high-level-architecture)
5. [Data Model](#5-data-model)
6. [API Signatures — Request and Response](#6-api-signatures--request-and-response)
7. [Order Placement Flow — Step by Step](#7-order-placement-flow--step-by-step)
8. [Inventory Reservation — Three Strategies](#8-inventory-reservation--three-strategies)
9. [Handling Duplicate Orders](#9-handling-duplicate-orders)
10. [Handling Payment Failures](#10-handling-payment-failures)
11. [Handling Out of Stock](#11-handling-out-of-stock)
12. [Concurrency — Two Solutions](#12-concurrency--two-solutions)
13. [TypeScript Class Implementation](#13-typescript-class-implementation)
14. [Trade-offs Summary](#14-trade-offs-summary)
15. [Follow-up Probes and Answers](#15-follow-up-probes-and-answers)
16. [Interview Script](#16-interview-script)

---

## 1. Clarifying Questions — Ask These First

> **Say this out loud before touching the keyboard or drawing anything.**

```
Scope:
  "Are we designing order placement only, or also cancellation, returns, tracking?"
  "Is this B2C only, or does it include a seller/merchant layer?"
  "Do we need to handle cart management, or does the caller provide the item list?"

Consistency:
  "How strongly consistent must inventory be?
   Is it acceptable to show 'In Stock' and then fail at order time?
   Or must we guarantee stock at the moment we show availability?"

Idempotency:
  "If the client submits the same order twice (network retry),
   should we create two orders or return the same order?"

Payment:
  "Are we integrating with an external payment gateway,
   or is there an internal payment service we call?"
  "If payment fails, should we retry automatically or surface it to the user?"

Scale:
  "What is the expected order volume? Millions per day?"
  "Are we designing for a single region or globally distributed?"

SLA:
  "What is the acceptable latency for order placement — less than 500ms?"
  "What is the acceptable failure rate?"
```

---

## 2. Requirements

### Functional
- Customer places an order with one or more items
- System checks inventory availability before confirming
- System reserves inventory so two customers cannot buy the same last item
- System processes payment through a payment service
- On success: order confirmed, inventory decremented, customer notified
- On payment failure: order cancelled, inventory released
- On out-of-stock: order rejected with a clear error
- Duplicate order submissions (retries) are idempotent — same result, no duplicate charges
- Order lifecycle: PENDING → CONFIRMED → SHIPPED → DELIVERED (or FAILED / CANCELLED)

### Non-functional
- Scale: 10M+ orders/day (~120/sec average, ~1,200/sec peak)
- Latency: order placement < 500ms p99
- Consistency: inventory must be strongly consistent — overselling is not acceptable
- Availability: 99.99%
- Durability: once confirmed, an order must never be lost

### Out of scope
- Cart management (caller provides item list)
- Order tracking and fulfilment
- Cancellation and returns

---

## 3. Capacity Estimation

```
Orders:
  10M orders/day / 86,400 = 116 orders/sec average
  Peak (10x):               ~1,200 orders/sec

Items per order:
  Average 3 items per order
  Inventory checks: 1,200 x 3 = 3,600 inventory reads/sec at peak

DB writes per order:
  1 order row + ~3 order_item rows + ~3 inventory updates + 1 payment record
  Total: ~8-10 DB writes per order

Storage:
  ~2KB per order record
  10M x 365 x 2KB = ~7 TB/year
```

---

## 4. High-Level Architecture

```
Customer (web / mobile)
        |
        v
   API Gateway (auth, rate limit, SSL)
        |
        v
  Order Service         <- the system we are designing
        |
        |---> Inventory Service  <- check + reserve stock
        |           |
        |           +--> PostgreSQL (inventory table)
        |                with SELECT FOR UPDATE or Redis distributed lock
        |
        |---> Payment Service   <- charge the customer
        |
        |---> PostgreSQL        <- orders, order_items, payments
        |
        |---> Kafka             <- publish order.created event (async)
        |           |
        |           |--> Notification Service (email/SMS)
        |           +--> Fulfilment Service (warehouse)
        |
        +--> Redis              <- idempotency keys, distributed locks

Order placement: synchronous, strongly consistent, < 500ms
Post-order events: async via Kafka — eventual consistency acceptable
```

---

## 5. Data Model

### Orders table

```sql
CREATE TABLE orders (
  id               UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id      UUID          NOT NULL,
  status           order_status  NOT NULL DEFAULT 'PENDING',
  idempotency_key  VARCHAR(128)  UNIQUE,
  total_amount     NUMERIC(12,2) NOT NULL,
  currency         CHAR(3)       NOT NULL DEFAULT 'USD',
  shipping_address JSONB         NOT NULL,
  billing_address  JSONB,
  created_at       TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
  updated_at       TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
  confirmed_at     TIMESTAMPTZ,
  failed_at        TIMESTAMPTZ,
  failure_reason   TEXT
);

CREATE TYPE order_status AS ENUM (
  'PENDING', 'CONFIRMED', 'PAYMENT_FAILED',
  'CANCELLED', 'SHIPPED', 'DELIVERED'
);

CREATE INDEX idx_orders_customer ON orders(customer_id, created_at DESC);
CREATE UNIQUE INDEX idx_orders_idempotency ON orders(idempotency_key)
  WHERE idempotency_key IS NOT NULL;
```

### Order items table

```sql
CREATE TABLE order_items (
  id           UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id     UUID          NOT NULL REFERENCES orders(id),
  product_id   UUID          NOT NULL,
  sku_id       UUID          NOT NULL,
  quantity     INTEGER       NOT NULL CHECK (quantity > 0),
  unit_price   NUMERIC(12,2) NOT NULL,
  total_price  NUMERIC(12,2) GENERATED ALWAYS AS (quantity * unit_price) STORED,
  product_name TEXT          NOT NULL,   -- price + name snapshot at order time
  product_meta JSONB
);

CREATE INDEX idx_order_items_order ON order_items(order_id);
```

### Inventory table

```sql
CREATE TABLE inventory (
  id              UUID          PRIMARY KEY DEFAULT gen_random_uuid(),
  sku_id          UUID          NOT NULL UNIQUE,
  total_stock     INTEGER       NOT NULL CHECK (total_stock >= 0),
  reserved_stock  INTEGER       NOT NULL DEFAULT 0 CHECK (reserved_stock >= 0),
  version         INTEGER       NOT NULL DEFAULT 1,  -- for optimistic locking
  updated_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
  CONSTRAINT chk_reserved_not_exceed_total CHECK (reserved_stock <= total_stock)
);

-- available_stock = total_stock - reserved_stock (computed in queries)
```

### Payments table

```sql
CREATE TABLE payments (
  id                UUID           PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id          UUID           NOT NULL REFERENCES orders(id),
  status            payment_status NOT NULL DEFAULT 'PENDING',
  amount            NUMERIC(12,2)  NOT NULL,
  currency          CHAR(3)        NOT NULL,
  payment_method_id UUID           NOT NULL,
  gateway_ref       VARCHAR(255),
  failure_code      VARCHAR(100),
  failure_message   TEXT,
  created_at        TIMESTAMPTZ    NOT NULL DEFAULT NOW()
);

CREATE TYPE payment_status AS ENUM (
  'PENDING', 'AUTHORISED', 'CAPTURED', 'FAILED', 'REFUNDED'
);
```

---

## 6. API Signatures — Request and Response

### Place Order — POST /api/v1/orders

```
POST /api/v1/orders
Authorization: Bearer {jwt_token}
Idempotency-Key: {client_generated_uuid}    <- REQUIRED
Content-Type: application/json
```

**Request body:**
```json
{
  "items": [
    { "sku_id": "sku-uuid-123", "quantity": 2 },
    { "sku_id": "sku-uuid-456", "quantity": 1 }
  ],
  "shipping_address": {
    "line1": "123 Main Street", "city": "Mumbai",
    "state": "Maharashtra", "postcode": "400001", "country": "IN"
  },
  "payment_method_id": "pm-uuid-789"
}
```

**Success — 201 Created:**
```json
{
  "data": {
    "order_id": "order-uuid-abc",
    "status": "CONFIRMED",
    "total_amount": 1499.98,
    "currency": "USD",
    "items": [
      { "sku_id": "sku-uuid-123", "product_name": "Blue Widget (M)",
        "quantity": 2, "unit_price": 599.99, "total_price": 1199.98 }
    ],
    "confirmed_at": "2024-04-12T14:32:05Z"
  }
}
```

**Error responses:**
```
400 Bad Request       { "error": "VALIDATION_ERROR", "details": {...} }
409 Conflict          { "error": "INSUFFICIENT_STOCK",
                        "details": { "out_of_stock_items": [...] } }
402 Payment Required  { "error": "PAYMENT_FAILED",
                        "details": { "failure_code": "insufficient_funds" } }
429 Too Many Requests { "error": "RATE_LIMITED", "retry_after": 60 }
503 Unavailable       { "error": "SERVICE_UNAVAILABLE", "retry_after": 30 }
```

---

## 7. Order Placement Flow — Step by Step

```
Step 1: VALIDATE INPUT
  Required fields present? Items non-empty? Quantities > 0?
  -> 400 if invalid

Step 2: CHECK IDEMPOTENCY KEY
  redis.get("idempotency:{key}")
  -> If found: return cached response (no re-processing)
  -> If not found: proceed

Step 3: CHECK INVENTORY AVAILABILITY (read — no lock)
  For each item: is available_stock >= requested_quantity?
  -> 409 if any item is out of stock
  (Fast pre-check — no lock. Race condition handled in Step 6)

Step 4: FETCH PRODUCT PRICES AND DETAILS
  Snapshot price, name, metadata at order time

Step 5: CREATE ORDER RECORD (status = PENDING)
  Insert into orders + order_items tables

Step 6: RESERVE INVENTORY (lock + decrement)
  Critical section — acquire lock, decrement reserved_stock
  -> If reservation fails (race — stock gone): rollback order, 409

Step 7: PROCESS PAYMENT
  Call payment service with orderId as idempotency key
  -> Hard failure (card declined): release inventory, 402
  -> Soft failure (gateway timeout): HOLD inventory, return 503

Step 8: CONFIRM ORDER
  Update status to CONFIRMED
  Move inventory from reserved to purchased

Step 9: PUBLISH EVENT (async, fire-and-forget)
  Kafka: order.confirmed
  Notification, fulfilment, analytics — do NOT await

Step 10: CACHE IDEMPOTENCY RESULT
  redis.setex("idempotency:result:{key}", 86400, response)

Return 201 Created
```

---

## 8. Inventory Reservation — Three Strategies

### Strategy 1 — Reserve at Cart time

```
Flow: Customer adds to cart -> immediately reserve inventory
      Hold for N minutes -> release on abandon or convert on checkout

Pros:
  + Customer never sees "in stock" then gets surprised at checkout
  + Eliminates race condition at checkout
  + "Reserved for you" is familiar UX

Cons:
  - Cart abandonment is ~70% -> popular items locked by non-buyers
  - Background job needed to release expired reservations
  - Complex: reservation state must sync with cart state

Best for: limited inventory, high-demand (concert tickets, limited editions)
```

### Strategy 2 — Reserve at Order time (RECOMMENDED)

```
Flow: Customer clicks "Place Order"
      Step A: Check availability (read, no lock)
      Step B: Reserve (write, acquire lock, decrement reserved_stock)
      Step C: Payment succeeds -> confirm reservation
      Step D: Payment fails -> release reservation

Pros:
  + No cart-abandonment lockup
  + Reservation held only for ~2-5 seconds
  + Clean two-phase: reserve -> confirm or release

Cons:
  - Race condition: two customers see "in stock", only one succeeds
  - Loser gets 409 at checkout (worse UX than cart-time)

Best for: most B2C e-commerce (Amazon, Flipkart)
This is the recommended strategy for this design.
```

### Strategy 3 — Reserve at Payment Confirmed time

```
Flow: Process payment first, THEN decrement inventory
      If out of stock after payment: refund and cancel

Pros:
  + Simplest flow — no two-phase reservation
  + No inventory lock during payment

Cons:
  - Customer can be charged for an out-of-stock item
  - Requires immediate refund -> poor UX, regulatory risk
  - Payment processor fees on refunds

Best for: digital goods only. AVOID for physical inventory.
```

### Comparison

```
                   Cart time    Order time    Payment time
Stock lockup:      HIGH         LOW           NONE
Oversell risk:     NONE         LOW           MEDIUM
Payment on OOS:    NONE         RARE          POSSIBLE
UX on OOS:         Cart stage   Checkout      Post-charge
Complexity:        HIGH         MEDIUM        LOW
```

---

## 9. Handling Duplicate Orders

### The problem

```
Customer clicks "Place Order" -> server creates order -> network drops
Client shows error -> customer clicks again -> second request hits server

Without idempotency: two orders created, customer charged twice
With idempotency:    second request returns the SAME result
```

### Solution — Idempotency Key

```
Client generates UUID before the request:
  const idempotencyKey = crypto.randomUUID();

Sends as header:
  Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

Server flow:
  1. Check Redis: GET idempotency:result:{key}
  2. If found -> return cached response immediately (no DB, no payment)
  3. If not found -> process normally
  4. After processing: cache result for 24 hours
     redis.setex("idempotency:result:{key}", 86400, JSON.stringify(response))
```

### Race condition — two requests with same key simultaneously

```typescript
// Acquire a lock BEFORE processing to prevent concurrent duplicate processing
const lockAcquired = await redis.set(
  `idempotency:lock:${key}`,
  'processing',
  'NX',     // only set if not exists
  'EX', 30  // expire after 30s
);

if (!lockAcquired) {
  // Another instance is processing this key — wait and check result
  await sleep(500);
  const cached = await redis.get(`idempotency:result:${key}`);
  if (cached) return JSON.parse(cached);
  return { status: 202, message: 'Order is being processed' };
}

try {
  const result = await createOrder(dto);
  await redis.setex(`idempotency:result:${key}`, 86400, JSON.stringify(result));
  return result;
} finally {
  await redis.del(`idempotency:lock:${key}`);
}
```

### Idempotency key scenarios

```
Same key + same payload + first succeeded:  return 201 with original order
Same key + same payload + first failed:     return 402 with original failure
Same key + DIFFERENT payload:               return 422 (conflict — generate new key)
No key provided:                            return 400 (enforce idempotency)
```


---

## 10. Handling Payment Failures

### Failure types and responses

```
Failure type          Code                  Action
─────────────────────────────────────────────────────────────────────
Insufficient funds    insufficient_funds    Release inventory, return 402
Card declined         card_declined         Release inventory, return 402
Card expired          expired_card          Release inventory, return 402
Fraud detected        fraud_detected        Release inventory, return 402
Gateway timeout       request_timeout       HOLD inventory, return 503
Network error         network_error         HOLD inventory, return 503
```

### Why the distinction matters

```
Hard failures (card declined):
  Payment will NOT succeed on retry
  Release inventory immediately
  Return 402 with clear message — user must fix their payment method

Soft failures (gateway timeout):
  Payment MAY have gone through on the gateway side
  DO NOT release inventory — could double-charge on retry
  DO NOT retry automatically without checking the original request
  Return 503 — user can try again; background job reconciles

The horror scenario without proper handling:
  Request times out -> server assumes failure -> releases inventory
  Payment actually succeeded on the gateway
  Customer charged but order cancelled -> support ticket, chargeback
```

### Payment with idempotency

```typescript
const result = await paymentService.charge({
  idempotencyKey:  orderId,  // use orderId as payment key
  // Same orderId on retry -> gateway returns same result -> no double-charge
  amount,
  currency:        'USD',
  paymentMethodId,
});

if (result.success) {
  return { success: true, gatewayRef: result.id };
}

const HARD_FAILURES = new Set([
  'insufficient_funds', 'card_declined', 'expired_card', 'fraud_detected',
]);

if (HARD_FAILURES.has(result.failureCode)) {
  return { success: false, shouldRelease: true, ...result };
}

// Soft failure: hold inventory, return 503
return { success: false, shouldRelease: false, failureCode: 'GATEWAY_ERROR' };
```

### Two-phase payment (mention as enhancement)

```
Better flow: Authorise -> Capture
  AUTHORISE: reserve money on card (no charge yet, valid 7 days)
  CAPTURE: actually charge after confirming item is available and shipped

Benefits:
  Customer not charged for OOS items
  Can cancel auth without a refund
  Better for inventory race conditions

For this design: use CAPTURED status after confirmation.
```

---

## 11. Handling Out of Stock

### Pre-check vs race condition

```
Pre-check at Step 3:
  SELECT (total_stock - reserved_stock) AS available FROM inventory WHERE sku_id = $1
  If available < requested -> return 409 immediately

Race condition between Step 3 and Step 6:
  Customer A pre-checks -> 1 in stock -> OK
  Customer B pre-checks -> 1 in stock -> OK
  Customer A reserves   -> success (stock now 0)
  Customer B reserves   -> FAILS (stock is now 0) -> 409

Customer B gets 409 at checkout even though pre-check showed stock.
This is acceptable — rare, handled gracefully.
The alternative (lock during pre-check) would kill throughput.
```

### Out-of-stock response

```json
HTTP 409 Conflict

{
  "error": "INSUFFICIENT_STOCK",
  "message": "Some items in your order are no longer available",
  "details": {
    "out_of_stock_items": [
      {
        "sku_id": "sku-uuid-123",
        "product_name": "Blue Widget (M)",
        "requested": 2,
        "available": 1
      }
    ],
    "suggestions": [
      { "sku_id": "sku-uuid-124", "product_name": "Blue Widget (L)", "available": 5 }
    ]
  }
}
```

---

## 12. Concurrency — Two Solutions

### The problem

```
10 customers simultaneously buy the last 1 unit.

Without control:
  All 10 read: available = 1 -> all see "in stock"
  All 10 execute: UPDATE SET reserved_stock = reserved_stock + 1
  Stock goes from 1 to -9 (oversold by 9 units)

We need exactly 1 to succeed and 9 to fail gracefully.
```

---

### Solution 1 — Database-Level Locking

#### Option A — Pessimistic locking (SELECT FOR UPDATE)

```sql
BEGIN;

-- Lock the rows — other transactions trying to lock same rows WAIT
SELECT id, sku_id, total_stock, reserved_stock,
       (total_stock - reserved_stock) AS available_stock
FROM inventory
WHERE sku_id = ANY($1::uuid[])
FOR UPDATE;

-- Check availability while holding the lock
-- If insufficient: ROLLBACK (lock released)

-- Decrement
UPDATE inventory
SET reserved_stock = reserved_stock + $quantity,
    updated_at = NOW()
WHERE sku_id = $sku_id
  AND (total_stock - reserved_stock) >= $quantity;

COMMIT;  -- lock released
```

```
How it works:
  Transaction A: SELECT FOR UPDATE -> acquires lock
  Transaction B: SELECT FOR UPDATE -> WAITS (blocked)
  Transaction A: COMMIT (decrements stock, lock released)
  Transaction B: reads updated stock (0), sees insufficient -> ROLLBACK -> 409

Pros:
  + Simple — database handles all locking
  + Atomic — no partial updates
  + No external dependency
  + Consistent — guaranteed by DB transaction guarantees

Cons:
  - Lock contention on hot items (popular products block each other)
  - Long transactions block other transactions
  - Does not scale horizontally across multiple DB nodes
  - App crash while holding lock: lock held until DB timeout (~30s)

Best for: low to medium concurrency per item (< 100 concurrent buyers/item)
```

#### Option B — Optimistic locking (version-based)

```sql
-- No lock on read
SELECT id, sku_id, total_stock, reserved_stock, version
FROM inventory WHERE sku_id = $1;

-- Application checks: available >= requested?

-- Update ONLY IF version matches (nobody else updated while we read)
UPDATE inventory
SET reserved_stock = reserved_stock + $quantity,
    version = version + 1,
    updated_at = NOW()
WHERE sku_id  = $1
  AND version = $current_version       -- the version we read
  AND (total_stock - reserved_stock) >= $quantity;

-- Check rowCount:
-- 0 -> version changed (concurrent update) -> retry or return 409
-- 1 -> success
```

```
How it works:
  Transaction A reads: { available: 1, version: 42 }
  Transaction B reads: { available: 1, version: 42 }
  Transaction A updates WHERE version = 42 -> SUCCESS, version -> 43
  Transaction B updates WHERE version = 42 -> 0 rows (version is 43)
  Transaction B retries, reads: { available: 0, version: 43 } -> 409

Pros:
  + No lock held on read -> much higher read throughput
  + Short transactions

Cons:
  - Retry logic required in application code
  - Under high contention (flash sale): retry storm -> DB overload
  - Need exponential backoff

Best for: low-to-medium concurrency with rare conflicts
```

---

### Solution 2 — Redis Distributed Lock

```
For high-concurrency (flash sales, limited drops, > 1,000/item/sec):
  Acquire a Redis lock per SKU before touching the DB
  Only one process holds the lock at a time
  Others fail fast or wait briefly
```

```typescript
async reserveWithDistributedLock(
  items:   Array<{ skuId: string; quantity: number }>,
  orderId: string,
): Promise<void> {
  // SORT items to prevent deadlocks
  // (always acquire locks in same order across all processes)
  const sorted = [...items].sort((a, b) => a.skuId.localeCompare(b.skuId));
  const acquiredLocks: string[] = [];

  try {
    // Acquire Redis lock per SKU
    for (const item of sorted) {
      const lockKey = `inventory:lock:${item.skuId}`;
      const acquired = await redis.set(
        lockKey,
        orderId,      // value = orderId for ownership verification
        'NX',         // only set if not exists
        'PX', 5000,   // expire after 5 seconds (crash safety)
      );

      if (!acquired) {
        throw new InsufficientStockError([{
          skuId: item.skuId, requested: item.quantity, available: 0,
        }]);
      }
      acquiredLocks.push(lockKey);
    }

    // All locks acquired -> safely update DB
    await db.query('BEGIN');
    try {
      for (const item of sorted) {
        const result = await db.query(
          `UPDATE inventory
           SET reserved_stock = reserved_stock + $1, updated_at = NOW()
           WHERE sku_id = $2
             AND (total_stock - reserved_stock) >= $1`,
          [item.quantity, item.skuId],
        );
        if (result.rowCount === 0) {
          throw new InsufficientStockError([{ skuId: item.skuId, requested: item.quantity, available: 0 }]);
        }
      }
      await db.query('COMMIT');
    } catch (err) {
      await db.query('ROLLBACK');
      throw err;
    }

  } finally {
    // ALWAYS release locks — even if something threw
    await Promise.all(acquiredLocks.map(key => releaseLock(key, orderId)));
  }
}

async function releaseLock(key: string, orderId: string): Promise<void> {
  // Lua script: atomic check-then-delete (only release if WE own the lock)
  const script = `
    if redis.call("get",KEYS[1])==ARGV[1] then
      return redis.call("del",KEYS[1])
    else
      return 0
    end
  `;
  await redis.eval(script, 1, key, orderId);
}
```

```
How it works:
  Process A: SET inventory:lock:sku-123 orderA NX PX 5000 -> acquired
  Process B: SET inventory:lock:sku-123 orderB NX PX 5000 -> nil (already held)
  Process B: returns 409 immediately
  Process A: updates DB, commits, releases lock

Deadlock prevention:
  Always sort SKU IDs before acquiring locks
  All processes use the same order -> no circular wait possible

Crash safety:
  PX 5000: lock expires after 5 seconds if process crashes
  Prevents dead locks from a crashed process holding the lock

Pros:
  + Works across multiple application servers (truly distributed)
  + Fine-grained: one lock per SKU, not per table
  + Fast: ~1ms round trip to Redis
  + No DB lock contention
  + Scales to flash-sale concurrency

Cons:
  - Extra infrastructure (Redis must be HA)
  - More complex code (lock acquisition, release, Lua script)
  - Lock TTL expiry risk: if DB operation takes longer than TTL,
    another process acquires lock -> potential double-reservation
    Mitigation: set TTL conservatively (2-3x expected operation time)
  - Redis failure: no orders can be placed
    Mitigation: Redis Sentinel or Cluster

Best for: > 1,000 concurrent buyers per item, flash sales, multi-server deploys
```

### Which solution to use when

```
Concurrency level       Recommendation
────────────────────────────────────────────────────────────
< 100/item/sec          DB pessimistic (SELECT FOR UPDATE)
                        Simple, no extra infra

100-1,000/item/sec      DB optimistic (version) + retry
                        More throughput, occasional retries

> 1,000/item/sec        Redis distributed lock
(flash sales)           Required. Extra infra justified.

Amazon-scale hybrid:    Redis for hot items (high view count)
                        DB lock for normal items
                        Auto-escalate based on contention metrics
```

---

## 13. TypeScript Class Implementation

```typescript
import { Pool, PoolClient } from 'pg';
import Redis from 'ioredis';

// ─── TYPES ────────────────────────────────────────────────────────────────────

interface OrderItem {
  skuId:       string;
  quantity:    number;
  unitPrice:   number;
  productName: string;
  productMeta: Record<string, unknown>;
}

interface Order {
  id:              string;
  customerId:      string;
  status:          OrderStatus;
  items:           OrderItem[];
  totalAmount:     number;
  currency:        string;
  shippingAddress: Address;
  createdAt:       Date;
  confirmedAt:     Date | null;
}

interface Address {
  line1:    string;
  line2?:   string;
  city:     string;
  state:    string;
  postcode: string;
  country:  string;
}

interface PlaceOrderDto {
  customerId:      string;
  items:           Array<{ skuId: string; quantity: number }>;
  shippingAddress: Address;
  paymentMethodId: string;
  idempotencyKey:  string;
}

interface PaymentResult {
  success:        boolean;
  gatewayRef?:    string;
  failureCode?:   string;
  failureMessage?: string;
  shouldRelease?: boolean;
}

enum OrderStatus {
  Pending        = 'PENDING',
  Confirmed      = 'CONFIRMED',
  PaymentFailed  = 'PAYMENT_FAILED',
  Cancelled      = 'CANCELLED',
}

// ─── CUSTOM ERRORS ────────────────────────────────────────────────────────────

class AppError extends Error {
  constructor(
    message: string,
    public readonly statusCode: number,
    public readonly code:       string,
    public readonly details?:   unknown,
  ) {
    super(message);
    this.name = this.constructor.name;
    Object.setPrototypeOf(this, new.target.prototype);
  }
}

class ValidationError extends AppError {
  constructor(message: string, details?: unknown) {
    super(message, 400, 'VALIDATION_ERROR', details);
  }
}

class InsufficientStockError extends AppError {
  constructor(outOfStockItems: Array<{ skuId: string; requested: number; available: number }>) {
    super('Not enough stock for some items', 409, 'INSUFFICIENT_STOCK', {
      out_of_stock_items: outOfStockItems,
    });
  }
}

class PaymentFailedError extends AppError {
  constructor(failureCode: string, failureMessage: string) {
    super('Payment could not be processed', 402, 'PAYMENT_FAILED', {
      failure_code: failureCode, failure_message: failureMessage,
    });
  }
}

class PaymentGatewayError extends AppError {
  constructor() {
    super('Payment service temporarily unavailable', 503, 'SERVICE_UNAVAILABLE');
  }
}

// ─── PAYMENT SERVICE INTERFACE ────────────────────────────────────────────────

interface PaymentService {
  charge(params: {
    idempotencyKey:  string;
    amount:          number;
    currency:        string;
    paymentMethodId: string;
  }): Promise<PaymentResult>;
}

interface EventBus {
  publish(event: string, data: unknown): Promise<void>;
}

// ─── ORDER SERVICE ────────────────────────────────────────────────────────────

class OrderService {
  static readonly #HARD_FAILURES = new Set([
    'insufficient_funds', 'card_declined', 'expired_card',
    'fraud_detected', 'do_not_honor', 'restricted_card',
  ]);

  constructor(
    private readonly db:             Pool,
    private readonly redis:          Redis,
    private readonly paymentService: PaymentService,
    private readonly eventBus:       EventBus,
  ) {}

  async placeOrder(dto: PlaceOrderDto): Promise<Order> {
    // Step 1: Validate input
    this.#validateDto(dto);

    // Step 2: Check idempotency (return cached result if duplicate request)
    const cached = await this.#getCachedIdempotencyResult(dto.idempotencyKey);
    if (cached) return cached;

    // Step 3: Pre-check inventory (optimistic read, no lock)
    await this.#preCheckInventory(dto.items);

    // Step 4: Snapshot product details and prices
    const enrichedItems = await this.#enrichItems(dto.items);
    const totalAmount   = enrichedItems.reduce(
      (sum, item) => sum + item.unitPrice * item.quantity, 0,
    );

    // Step 5: Create order record (status = PENDING)
    const orderId = crypto.randomUUID();
    await this.#createOrder({ orderId, dto, items: enrichedItems, totalAmount });

    // Step 6: Reserve inventory (acquire lock, decrement reserved_stock)
    try {
      await this.#reserveInventory(dto.items, orderId);
    } catch (err) {
      await this.#cancelOrder(orderId, 'Inventory reservation failed');
      throw err;
    }

    // Step 7: Process payment
    const paymentResult = await this.#processPayment(
      orderId, totalAmount, dto.paymentMethodId,
    );

    if (!paymentResult.success) {
      if (paymentResult.shouldRelease) {
        await this.#releaseInventory(dto.items);
        await this.#markPaymentFailed(orderId, paymentResult);
        throw new PaymentFailedError(
          paymentResult.failureCode!,
          paymentResult.failureMessage!,
        );
      }
      // Soft failure — hold inventory, return 503
      throw new PaymentGatewayError();
    }

    // Step 8: Confirm order
    const confirmed = await this.#confirmOrder(orderId, dto.items, paymentResult.gatewayRef!);

    // Step 9: Publish event (fire-and-forget)
    this.eventBus.publish('order.confirmed', {
      orderId, customerId: dto.customerId, totalAmount, items: enrichedItems,
    }).catch(err => console.error('Event publish failed', { orderId, err }));

    // Step 10: Cache idempotency result
    await this.redis.setex(
      `idempotency:result:${dto.idempotencyKey}`, 86400,
      JSON.stringify(confirmed),
    );

    return confirmed;
  }

  // ── PRIVATE METHODS ────────────────────────────────────────────────────────

  #validateDto(dto: PlaceOrderDto): void {
    if (!dto.items?.length) {
      throw new ValidationError('Order must contain at least one item');
    }
    if (dto.items.length > 50) {
      throw new ValidationError('Order cannot contain more than 50 items');
    }
    for (const item of dto.items) {
      if (!item.skuId) throw new ValidationError('Each item must have a valid sku_id');
      if (!Number.isInteger(item.quantity) || item.quantity < 1) {
        throw new ValidationError('Item quantity must be a positive integer');
      }
      if (item.quantity > 100) throw new ValidationError('Item quantity cannot exceed 100');
    }
    if (!dto.shippingAddress?.country) {
      throw new ValidationError('Shipping address with country is required');
    }
    if (!dto.paymentMethodId) throw new ValidationError('Payment method is required');
    if (!dto.idempotencyKey)  throw new ValidationError('Idempotency-Key header is required');
  }

  async #getCachedIdempotencyResult(key: string): Promise<Order | null> {
    const lockAcquired = await this.redis.set(
      `idempotency:lock:${key}`, 'processing', 'NX', 'EX', 30,
    );
    if (!lockAcquired) {
      await new Promise(resolve => setTimeout(resolve, 500));
      const cached = await this.redis.get(`idempotency:result:${key}`);
      return cached ? JSON.parse(cached) as Order : null;
    }
    const cached = await this.redis.get(`idempotency:result:${key}`);
    if (!cached) return null;
    await this.redis.del(`idempotency:lock:${key}`);
    return JSON.parse(cached) as Order;
  }

  async #preCheckInventory(
    items: Array<{ skuId: string; quantity: number }>,
  ): Promise<void> {
    const { rows } = await this.db.query<{ sku_id: string; available: number }>(
      `SELECT sku_id, (total_stock - reserved_stock) AS available
       FROM inventory
       WHERE sku_id = ANY($1::uuid[])`,
      [items.map(i => i.skuId)],
    );
    const stockMap = new Map(rows.map(r => [r.sku_id, r.available]));
    const oos = items
      .filter(item => (stockMap.get(item.skuId) ?? 0) < item.quantity)
      .map(item => ({
        skuId:     item.skuId,
        requested: item.quantity,
        available: stockMap.get(item.skuId) ?? 0,
      }));
    if (oos.length > 0) throw new InsufficientStockError(oos);
  }

  async #enrichItems(
    items: Array<{ skuId: string; quantity: number }>,
  ): Promise<OrderItem[]> {
    const { rows } = await this.db.query(
      `SELECT s.id AS sku_id, s.price AS unit_price,
              p.name AS product_name, s.attributes AS product_meta
       FROM skus s JOIN products p ON p.id = s.product_id
       WHERE s.id = ANY($1::uuid[])`,
      [items.map(i => i.skuId)],
    );
    const detailMap = new Map(rows.map(r => [r.sku_id, r]));
    return items.map(item => {
      const d = detailMap.get(item.skuId);
      if (!d) throw new ValidationError(`SKU ${item.skuId} not found`);
      return {
        skuId: item.skuId, quantity: item.quantity,
        unitPrice: d.unit_price, productName: d.product_name,
        productMeta: d.product_meta ?? {},
      };
    });
  }

  async #createOrder(params: {
    orderId: string; dto: PlaceOrderDto;
    items: OrderItem[]; totalAmount: number;
  }): Promise<void> {
    const client = await this.db.connect();
    try {
      await client.query('BEGIN');
      await client.query(
        `INSERT INTO orders (id, customer_id, status, total_amount, currency,
           shipping_address, idempotency_key)
         VALUES ($1, $2, 'PENDING', $3, 'USD', $4, $5)`,
        [params.orderId, params.dto.customerId, params.totalAmount,
         JSON.stringify(params.dto.shippingAddress), params.dto.idempotencyKey],
      );
      for (const item of params.items) {
        await client.query(
          `INSERT INTO order_items (order_id, sku_id, quantity, unit_price, product_name, product_meta)
           VALUES ($1, $2, $3, $4, $5, $6)`,
          [params.orderId, item.skuId, item.quantity, item.unitPrice,
           item.productName, JSON.stringify(item.productMeta)],
        );
      }
      await client.query('COMMIT');
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }

  async #reserveInventory(
    items:   Array<{ skuId: string; quantity: number }>,
    orderId: string,
  ): Promise<void> {
    const sorted = [...items].sort((a, b) => a.skuId.localeCompare(b.skuId));
    const acquiredLocks: string[] = [];
    try {
      for (const item of sorted) {
        const lockKey = `inventory:lock:${item.skuId}`;
        const acquired = await this.redis.set(lockKey, orderId, 'NX', 'PX', 5000);
        if (!acquired) {
          throw new InsufficientStockError([{
            skuId: item.skuId, requested: item.quantity, available: 0,
          }]);
        }
        acquiredLocks.push(lockKey);
      }
      await this.db.query('BEGIN');
      try {
        for (const item of sorted) {
          const result = await this.db.query(
            `UPDATE inventory
             SET reserved_stock = reserved_stock + $1, updated_at = NOW()
             WHERE sku_id = $2 AND (total_stock - reserved_stock) >= $1`,
            [item.quantity, item.skuId],
          );
          if (result.rowCount === 0) {
            throw new InsufficientStockError([{
              skuId: item.skuId, requested: item.quantity, available: 0,
            }]);
          }
        }
        await this.db.query('COMMIT');
      } catch (err) {
        await this.db.query('ROLLBACK');
        throw err;
      }
    } finally {
      await Promise.all(acquiredLocks.map(k => this.#releaseLock(k, orderId)));
    }
  }

  async #processPayment(
    orderId: string, amount: number, paymentMethodId: string,
  ): Promise<PaymentResult> {
    try {
      return await this.paymentService.charge({
        idempotencyKey: orderId,   // prevents double-charge on retry
        amount, currency: 'USD', paymentMethodId,
      });
    } catch (err: unknown) {
      // Gateway unreachable — soft failure, hold inventory
      if (err instanceof Error && err.message.includes('timeout')) {
        return { success: false, shouldRelease: false, failureCode: 'GATEWAY_TIMEOUT' };
      }
      throw err;
    }
  }

  async #releaseInventory(items: Array<{ skuId: string; quantity: number }>): Promise<void> {
    await this.db.query(
      `UPDATE inventory i
       SET reserved_stock = reserved_stock - v.qty, updated_at = NOW()
       FROM unnest($1::uuid[], $2::int[]) AS v(sku, qty)
       WHERE i.sku_id = v.sku`,
      [items.map(i => i.skuId), items.map(i => i.quantity)],
    );
  }

  async #confirmOrder(
    orderId: string,
    items:   Array<{ skuId: string; quantity: number }>,
    gatewayRef: string,
  ): Promise<Order> {
    const client = await this.db.connect();
    try {
      await client.query('BEGIN');

      // Update order status
      const { rows: [row] } = await client.query(
        `UPDATE orders SET status = 'CONFIRMED', confirmed_at = NOW(), updated_at = NOW()
         WHERE id = $1 RETURNING *`,
        [orderId],
      );

      // Finalise inventory: decrement both reserved AND total
      await client.query(
        `UPDATE inventory i
         SET total_stock    = total_stock    - oi.quantity,
             reserved_stock = reserved_stock - oi.quantity,
             updated_at     = NOW()
         FROM order_items oi
         WHERE oi.order_id = $1 AND i.sku_id = oi.sku_id`,
        [orderId],
      );

      // Record payment
      await client.query(
        `INSERT INTO payments (order_id, status, amount, currency, payment_method_id, gateway_ref)
         VALUES ($1, 'CAPTURED', $2, 'USD', $3, $4)`,
        [orderId, row.total_amount, row.payment_method_id, gatewayRef],
      );

      await client.query('COMMIT');

      const orderItems = await this.#fetchItems(orderId);
      return this.#mapRow(row, orderItems);
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }

  async #cancelOrder(orderId: string, reason: string): Promise<void> {
    await this.db.query(
      `UPDATE orders SET status = 'CANCELLED', failure_reason = $2, updated_at = NOW()
       WHERE id = $1`,
      [orderId, reason],
    );
  }

  async #markPaymentFailed(orderId: string, result: PaymentResult): Promise<void> {
    await this.db.query(
      `UPDATE orders
       SET status = 'PAYMENT_FAILED', failed_at = NOW(),
           failure_reason = $2, updated_at = NOW()
       WHERE id = $1`,
      [orderId, `${result.failureCode}: ${result.failureMessage}`],
    );
  }

  async #releaseLock(key: string, value: string): Promise<void> {
    const script = `
      if redis.call("get",KEYS[1])==ARGV[1] then
        return redis.call("del",KEYS[1])
      else return 0 end
    `;
    await this.redis.eval(script, 1, key, value);
  }

  async #fetchItems(orderId: string): Promise<OrderItem[]> {
    const { rows } = await this.db.query(
      `SELECT sku_id, quantity, unit_price, product_name, product_meta
       FROM order_items WHERE order_id = $1`,
      [orderId],
    );
    return rows.map(r => ({
      skuId: r.sku_id, quantity: r.quantity,
      unitPrice: r.unit_price, productName: r.product_name,
      productMeta: r.product_meta,
    }));
  }

  #mapRow(row: Record<string, unknown>, items: OrderItem[]): Order {
    return {
      id:              String(row['id']),
      customerId:      String(row['customer_id']),
      status:          row['status'] as OrderStatus,
      items,
      totalAmount:     Number(row['total_amount']),
      currency:        String(row['currency']),
      shippingAddress: row['shipping_address'] as Address,
      createdAt:       new Date(String(row['created_at'])),
      confirmedAt:     row['confirmed_at']
        ? new Date(String(row['confirmed_at'])) : null,
    };
  }
}
```

---

## 14. Trade-offs Summary

```
Decision                  Choice             Trade-off accepted
──────────────────────────────────────────────────────────────────────────────
Inventory reserve timing  Order time         Customer may see "in stock" then
                          (Strategy 2)       get 409 at checkout. Better than
                                             locking stock during 70% abandon.

Concurrency control       Redis lock         Extra infra. Justified at Amazon
                          (primary)          scale. Fall back to DB lock for
                          DB lock (fallback) low-traffic items.

Payment idempotency       orderId as key     Prevents double-charge on retry.
                                             Small risk if orderId changes
                                             between retries (does not in
                                             our design — ID assigned first).

Pre-check then lock       Two-phase          Race condition possible between
                                             check and lock. Handled by lock
                                             step failing gracefully with 409.
                                             Avoids holding lock during product
                                             price lookup (slow operation).

Gateway timeout           Hold inventory,    Risk: customer doesn't know if
                          return 503         order succeeded. Mitigated by
                                             background reconciliation job.

Idempotency key required  Enforce as header  Extra client complexity. Prevents
                                             silent double-orders. Worth the
                                             friction for financial operations.

Price snapshot            Yes, at order time Order total never changes after
                                             placement. Fair to customer.
```

---

## 15. Follow-up Probes and Answers

**"What if the Redis lock expires before the DB transaction commits?"**
If the lock TTL is 5 seconds and the DB operation takes 6 seconds under load, another process could acquire the lock while the first is still writing. This can cause double-reservation. Mitigation: set TTL conservatively — 2-3x the expected operation time. Also: after the DB COMMIT, verify reserved_stock did not exceed total_stock. A background reconciliation job can detect and correct anomalies.

**"How do you handle a flash sale with 100,000 concurrent buyers for 1 item?"**
The Redis lock serialises all attempts — only 1 proceeds at a time, 99,999 get 409. This is correct but creates a thundering herd on Redis. Better for extreme flash sales: a dedicated SQS FIFO queue. All buyers enqueue. A single worker drains in order. First buyer gets the item, rest get 409. Eliminates lock contention entirely. Redis pub/sub or WebSocket pushes the result back to each waiting buyer.

**"What if the payment service is down for 10 minutes?"**
All orders during that window get 503. Inventory reservations are held. Orders remain PENDING. A background reconciliation job runs every minute: finds orders PENDING for > 15 minutes, queries the payment gateway directly, resolves them. If gateway confirms payment: mark CONFIRMED. If confirmed failure: release inventory, mark PAYMENT_FAILED. If no record: release inventory, mark CANCELLED.

**"How do you prevent a customer placing 1,000 simultaneous orders?"**
Rate limiting at the API Gateway: 5 order placement requests per customer per minute, keyed on customerId. Also: reject if customer has > N open PENDING orders. Business rule: one active cart per customer.

**"How do you handle partial order failure — 3 items succeed, 1 is out of stock?"**
Two options: (1) all-or-nothing — reject the whole order if any item fails. Simple, customer removes OOS item and reorders. (2) partial fulfilment — place order for available items, back-order the rest. For most e-commerce: option 1. For Amazon specifically: option 2 with clear communication ("Item X ships separately when available").

**"What is your database choice and why?"**
PostgreSQL. Orders require ACID transactions — inventory decrement and order creation must be atomic. Strong consistency is non-negotiable. JSONB for flexible address and product metadata. For the read side (order history, search), a read replica and Redis cache handle the load. At extreme Amazon scale, DynamoDB for order lookups (customerId → [orderIds]) with PostgreSQL for financial records.

---

## 16. Interview Script

### Opening — clarifying questions first

> "Before I design anything, let me ask a few clarifying questions. Are we designing order placement only, or also cancellation and returns? Is this B2C only? How consistent does inventory need to be — is occasional overselling acceptable, or must we guarantee it? Should the same order submitted twice create two orders or return the same one? What is the expected scale — millions of orders per day? And what is the latency SLA on order placement?"

### Core insight — state this upfront

> "The central challenge in order placement is the inventory race condition — two customers buying the last item simultaneously. There are three separate concerns that look similar but are not: the concurrency problem for inventory, idempotency for duplicate requests, and payment failure handling where we must distinguish between a hard decline and a gateway timeout. These three need different solutions."

### Inventory reservation strategy

> "I would use Strategy 2: reserve inventory at order placement time, not at add-to-cart time. Cart abandonment is around 70% — locking stock during that window makes popular items appear unavailable to genuine buyers. At order placement we have a much shorter reservation window — just the two to three seconds of the payment flow."

### Concurrency solution

> "For the race condition I would use a Redis distributed lock per SKU. Before touching the database, acquire a lock with SET NX and a five-second TTL. Only one process holds the lock at a time. Others fail fast with 409. Locks are always acquired in alphabetical order across all processes to prevent deadlocks on multi-item orders. For lower-traffic items, SELECT FOR UPDATE in a Postgres transaction is simpler — I would use Redis only for hot items where DB lock contention becomes measurable."

### Idempotency

> "The client generates a UUID before every order request and sends it as an Idempotency-Key header. The server checks Redis before processing. If the key exists, return the cached response — no DB query, no payment charge. After processing, cache the result for 24 hours. To handle two requests with the same key arriving simultaneously, use Redis SET NX as a lock on the key itself."

### Payment failure handling

> "Two different failure types need different treatment. Hard failures — card declined, insufficient funds — mean the payment will never succeed. Release inventory immediately and return 402 with a clear error message. Soft failures — gateway timeout, network error — are ambiguous. The payment may have succeeded on the gateway side. Never release inventory or retry blindly in this case. Return 503, hold the reservation, and let a background reconciliation job resolve it by querying the gateway directly."


---------------------------------------------------------------------------------


## Cart Management

**What it is:** The cart is a temporary, user-owned collection of items a customer intends to buy. It exists before an order is created and is discarded once an order is placed or abandoned.

**Data model:**
```
carts
  id          UUID
  customer_id UUID
  created_at  TIMESTAMPTZ
  updated_at  TIMESTAMPTZ
  expires_at  TIMESTAMPTZ   ← TTL: abandon after 30 days of inactivity

cart_items
  id          UUID
  cart_id     UUID REFERENCES carts(id)
  sku_id      UUID
  quantity    INTEGER
  added_at    TIMESTAMPTZ
  saved_price NUMERIC(12,2) NULL   ← optional: snapshot price when added (for "price changed" notice)
```

**Where it lives:** Cart data is read-heavy and session-scoped. The right storage is Redis (for active carts) with a PostgreSQL fallback for persistence across sessions. When a customer adds an item, write to Redis with a TTL. When the customer logs out and logs back in, restore from PostgreSQL. This gives you sub-millisecond cart reads without hammering the DB.

**Key behaviours:**
- Adding an item: `UPSERT` — if the SKU already exists in the cart, increment quantity
- Removing an item: hard delete from cart_items
- Merging carts: if a guest user logs in and has an existing cart, merge the two (take max quantity per SKU, or add quantities — business decision)
- Cart does NOT reserve inventory — this is Strategy 2 from the order design. Stock is only reserved at order placement time

**What NOT to do:** Never store the cart in the client (local storage / cookies) beyond a session. Prices change, products go out of stock, and you lose the ability to notify the customer. Always keep the authoritative cart on the server.

**Follow-up probe answer:** *"Cart is a temporary data structure stored in Redis with PostgreSQL for persistence. It does not touch inventory. The reservation only happens at order placement time — this is an explicit trade-off between UX (customer might see 'in stock' and then get a 409 at checkout) and stock utilisation efficiency (70% of carts are abandoned — we don't want to lock up inventory for non-buyers)."*

---

## Product Catalogue and Search

**What it is:** The product catalogue is the authoritative database of every product available on the platform — its name, description, images, attributes (size, colour, weight), pricing, and available SKUs (variants). Search is the mechanism by which customers find products.

**Data model:**
```
products
  id           UUID
  name         TEXT
  description  TEXT
  brand_id     UUID
  category_id  UUID
  is_active    BOOLEAN
  created_at   TIMESTAMPTZ

skus                         ← a product variant (size + colour combination)
  id           UUID
  product_id   UUID REFERENCES products(id)
  sku_code     VARCHAR(50)   ← internal code: "WIDGET-BLUE-M"
  price        NUMERIC(12,2)
  attributes   JSONB         ← { "colour": "blue", "size": "M", "weight_kg": 0.5 }
  is_active    BOOLEAN

categories                   ← hierarchical: Electronics > Phones > Smartphones
  id           UUID
  name         TEXT
  parent_id    UUID NULL REFERENCES categories(id)
```

**Read pattern — why it needs caching:** Product pages are the most read-heavy part of any e-commerce platform. On Amazon, a product page gets thousands of reads per second for a popular item. The DB cannot handle this directly. The pattern is:

```
Request for product {id}
  → Check Redis: GET product:{id}
  → Cache hit:   return instantly (~1ms)
  → Cache miss:  query PostgreSQL → store in Redis (TTL 5 min) → return
```

Price and stock change relatively rarely. A 5-minute TTL is acceptable for product details. For stock level ("In Stock" vs "Only 3 left"), use a shorter TTL or a separate real-time inventory read.

**Search — why PostgreSQL full-text search is not enough at Amazon scale:** PostgreSQL FTS works well up to ~10 million products. Amazon has hundreds of millions. At that scale you need Elasticsearch (or OpenSearch). The search pipeline looks like this:

```
Product created/updated in PostgreSQL
  → change event published to Kafka (via CDC or application event)
  → Search indexer service consumes event
  → Indexes document into Elasticsearch
    { id, name, description, brand, category, attributes, price, rating }

Customer searches "blue running shoes size 10"
  → Elasticsearch query with:
    - full-text match on name + description
    - filter on attributes.colour = "blue"
    - filter on attributes.size = "10"
    - filter on category = "Running Shoes"
    - sort by relevance score OR price OR rating
  → Returns product IDs + relevance scores
  → Product service fetches full details for those IDs (from Redis/DB)
  → Return to customer
```

**Why Elasticsearch over PostgreSQL for search:**
```
PostgreSQL FTS:   good for < 10M products, no relevance tuning,
                  no faceted search, no typo tolerance
Elasticsearch:   relevance scoring (BM25), fuzzy matching ("shooes" → "shoes"),
                 faceted search (filter by brand/price/colour simultaneously),
                 autocomplete suggestions, synonym handling
```

**Follow-up probe answer:** *"Product catalogue is PostgreSQL for write operations and authoritative storage, Redis for read caching with short TTLs. Search is Elasticsearch, kept in sync via Kafka events published on every product create/update. PostgreSQL FTS is good for smaller catalogues — at Amazon scale you need Elasticsearch for relevance ranking, faceted filtering, and typo tolerance."*

---

## Notification Service

**What it is:** A service that delivers messages to customers about events related to their account and orders — order confirmation, payment failure, shipment dispatch, delivery, promotional messages, and low-stock/restock alerts.

**Trigger mechanism:**
```
Order confirmed
  → Order Service publishes order.confirmed to Kafka
  → Notification Service subscribes to order.confirmed
  → Determines: what channels? (email + SMS + push)
  → Determines: what template? (order_confirmation_v2)
  → Renders message with order data
  → Dispatches via channel-specific provider
```

This is deliberately async and decoupled from the order placement flow. The order API does not wait for the notification to be sent before returning 201 to the customer. A notification failure must never fail an order.

**Channel providers:**
```
Email:      AWS SES v2 — transactional emails (order confirm, shipping)
SMS:        Twilio / AWS SNS — time-sensitive alerts (OTP, delivery window)
Push:       Firebase FCM (Android) / APNs (iOS) — app notifications
In-app:     WebSocket connection — real-time alerts inside the app
WhatsApp:   Twilio WhatsApp API — used heavily in India/SEA markets
```

**Notification data model:**
```
notifications
  id              UUID
  customer_id     UUID
  type            VARCHAR(50)    ← 'ORDER_CONFIRMED', 'SHIPPED', 'DELIVERED'
  channel         VARCHAR(20)    ← 'EMAIL', 'SMS', 'PUSH'
  status          VARCHAR(20)    ← 'PENDING', 'SENT', 'FAILED', 'DELIVERED'
  template_id     UUID
  payload         JSONB          ← rendered message data
  provider_ref    VARCHAR(200)   ← SES message ID, Twilio SID
  sent_at         TIMESTAMPTZ
  delivered_at    TIMESTAMPTZ
  failure_reason  TEXT
```

**Key concerns:**

*Idempotency:* If the Kafka consumer crashes after sending the email but before committing the offset, Kafka will redeliver the message. The consumer must check if a notification for this order + channel + type has already been sent (check notifications table) before sending. Otherwise the customer gets two confirmation emails.

*Preference management:* Customers opt out of promotional emails but still want transactional ones (order confirmation is always sent regardless of preference). Store notification preferences per customer per channel per type.

*Rate limiting:* A promotional campaign blast should not overwhelm SES. Use a token bucket or leaky bucket rate limiter — max N emails per second through SES, queue the rest.

**Follow-up probe answer:** *"Notification service subscribes to Kafka events published by the order service. It is completely decoupled — a notification failure never affects order placement. The service checks before sending to ensure idempotency in case of Kafka message redelivery. Channel selection is driven by customer preferences and notification type — order confirmations always go via email regardless of marketing opt-outs."*

---

## Order Status and Tracking

**What it is:** After an order is placed, the customer needs visibility into where their order is — confirmed, being picked, shipped, out for delivery, delivered. At Amazon scale this involves multiple systems: the order service, the warehouse management system (WMS), the logistics provider, and the customer-facing tracking UI.

**Order lifecycle and state machine:**
```
PENDING
  → CONFIRMED          (payment captured, sent to warehouse)
  → PAYMENT_FAILED     (terminal — customer must retry)
  → CANCELLED          (by customer or system)

CONFIRMED
  → PROCESSING         (warehouse picked up the order)
  → SHIPPED            (carrier scanned the package)
  → OUT_FOR_DELIVERY   (last-mile delivery started)
  → DELIVERED          (customer signed or photo taken)
  → RETURN_REQUESTED   (customer initiated return)

State transitions are published as events to Kafka.
Each consumer (notification, analytics, customer UI) reacts independently.
```

**Data model additions:**
```
order_events                 ← immutable append-only log of every status change
  id              UUID
  order_id        UUID REFERENCES orders(id)
  status          order_status
  description     TEXT       ← "Package picked up by DHL at Mumbai hub"
  location        TEXT NULL  ← "Mumbai Sorting Facility"
  occurred_at     TIMESTAMPTZ
  source          VARCHAR(50) ← 'WAREHOUSE_SYSTEM', 'CARRIER_WEBHOOK', 'CUSTOMER'

shipments
  id              UUID
  order_id        UUID
  carrier         VARCHAR(50)  ← 'DHL', 'FedEx', 'Delhivery'
  tracking_number VARCHAR(100)
  tracking_url    TEXT
  shipped_at      TIMESTAMPTZ
  estimated_delivery TIMESTAMPTZ
  delivered_at    TIMESTAMPTZ
```

**Real-time tracking:**

Polling ("refresh the page to check status") is a poor UX. The production pattern is:
```
Customer opens order tracking page
  → Browser opens WebSocket connection to the Notification Service
  → Subscribed to channel: order:{orderId}:{customerId}

Carrier sends a webhook (DHL scanned the package):
  → Webhook receiver updates order_events table
  → Publishes order.status_updated to Kafka
  → Notification Service consumes event
  → Pushes update to WebSocket subscribers for this order
  → Customer sees "Package scanned at Mumbai hub" in real time
  → Simultaneously: push notification sent to mobile app
```

**Carrier integration:** Carriers (DHL, FedEx, Delhivery) send webhooks to your platform when the package status changes. You also need outbound API calls to the carrier to get tracking URLs and estimated delivery windows. This is handled by a dedicated Logistics Service that abstracts away the different carrier APIs.

**Follow-up probe answer:** *"Order status is tracked via an immutable order_events table — every state change is appended, never updated. The current status is always the latest event. Real-time updates are pushed to the customer via WebSocket, triggered by Kafka events published when a carrier webhook arrives. This decouples the carrier integration from the customer UI and the notification system."*

---

## Inventory Updates — Restocking

**What it is:** When new stock arrives at a warehouse, the inventory system needs to be updated so the platform can show items as available again, notify customers on the waitlist, and resume taking orders.

**The restocking flow:**
```
Physical goods arrive at warehouse
  → Warehouse operative scans items into Warehouse Management System (WMS)
  → WMS publishes inventory.restocked event to Kafka:
    { skuId: 'uuid', warehouseId: 'wh-1', quantityAdded: 500, arrivedAt: '...' }

Inventory Service consumes inventory.restocked:
  → UPDATE inventory
    SET total_stock = total_stock + 500,
        updated_at  = NOW()
    WHERE sku_id = 'uuid'
  → Publishes sku.back_in_stock to Kafka (if stock was previously 0)

Product Catalogue Service consumes sku.back_in_stock:
  → Invalidates the Redis cache for this product
  → Next product page request reads fresh stock status from DB
  → Updates Elasticsearch document (is_in_stock = true)

Notification Service consumes sku.back_in_stock:
  → Queries waitlist table:
    SELECT customer_id FROM waitlist
    WHERE sku_id = 'uuid' AND notified_at IS NULL
    ORDER BY created_at ASC
    LIMIT 500                    ← first 500 customers on waitlist
  → Sends each customer:
    "Good news — Blue Widget (M) is back in stock.
     Reserve yours for the next 2 hours."
  → Marks those waitlist rows as notified_at = NOW()
  → Creates a temporary reservation window (2 hours) for notified customers
```

**The reservation window for waitlisted customers:**
```
After notification:
  INSERT INTO inventory_reservations
    (customer_id, sku_id, quantity, type, expires_at)
  VALUES
    (cust_id, sku_id, 1, 'WAITLIST', NOW() + INTERVAL '2 hours')

This soft-reserves one unit per notified customer
If the customer places an order within 2 hours: confirmed
If the customer doesn't act within 2 hours: reservation expires (background job)
Stock released back to general availability
```

**Concurrency concern at restock:** When 500 customers are all notified simultaneously and many click immediately, you get a flash-sale-like concurrency spike. The Redis distributed lock from the order placement design handles this — the same mechanism applies.

**Multi-warehouse complexity:** At Amazon scale, the same SKU exists in multiple warehouses. The inventory table tracks per-warehouse stock. When a customer orders, the system routes to the nearest warehouse with available stock (factoring in customer location and warehouse stock levels). This is handled by a Fulfilment Routing Service that receives the order.confirmed Kafka event and decides which warehouse fulfils it.

```
inventory (extended)
  id           UUID
  sku_id       UUID
  warehouse_id UUID       ← added for multi-warehouse
  total_stock  INTEGER
  reserved_stock INTEGER
  UNIQUE(sku_id, warehouse_id)
```

**Follow-up probe answer:** *"Inventory restocking is event-driven. The warehouse management system publishes an inventory.restocked event to Kafka when new stock arrives. The inventory service increments total_stock. If the SKU was previously out of stock, a sku.back_in_stock event is published. The notification service consumes this, queries the waitlist in FIFO order, and sends 'back in stock' alerts to the first N customers. Each notified customer gets a time-limited soft reservation — typically 2 hours — before the stock is released to general availability. The same concurrency controls from order placement apply here since the restock notification can trigger a flash-sale-like spike."*



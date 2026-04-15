# Technologies and Tools Reference

> **Purpose:** Deep understanding of every technology used across the Report Distribution Module and Case Reprocessing Framework — definitions, capabilities, limits, use cases, and the specific rationale for choosing each in a Pharmacovigilance context.  
> **Use:** Read this before any interview where you may be asked "why did you use X?" or "what are the limits of X?"

---

## Table of Contents

1. [AWS Lambda](#1-aws-lambda)
2. [AWS SQS (Simple Queue Service)](#2-aws-sqs-simple-queue-service)
3. [AWS Step Functions](#3-aws-step-functions)
4. [PostgreSQL](#4-postgresql)
5. [AWS S3 (Simple Storage Service)](#5-aws-s3-simple-storage-service)
6. [AWS SES v2 (Simple Email Service)](#6-aws-ses-v2-simple-email-service)
7. [TypeScript on Lambda](#7-typescript-on-lambda)
8. [Technology Choice Comparison Matrix](#8-technology-choice-comparison-matrix)

---

## 1. AWS Lambda

### What it is

AWS Lambda is a **serverless compute** service. You upload code (a function) and Lambda runs it in response to a trigger — an SQS message, an HTTP request, an S3 event, a scheduled timer, etc. You never provision or manage servers. AWS handles scaling, patching, and availability.

### How it works internally

```
Trigger arrives (e.g. SQS message)
  │
  ▼
Lambda service checks for a warm execution environment
  │
  ├── Warm container available
  │     → reuse it (no cold start)
  │     → run handler function (~ms startup)
  │
  └── No warm container (cold start)
        → spin up new container (~100ms-2s depending on runtime)
        → initialise runtime (Node.js, Python, Java, etc.)
        → run initialisation code (DB connections, config loading)
        → run handler function

After invocation:
  Container is kept warm for ~15 minutes
  If another invocation arrives → reuse the warm container
  If no invocation for ~15 minutes → container is frozen/reclaimed
```

### Key specifications and limits

```
Execution limits:
  Timeout:              15 minutes maximum per invocation
  Memory:               128 MB to 10,240 MB (10 GB)
  CPU:                  proportional to memory (1 vCPU at ~1,769 MB)
  Ephemeral storage:    512 MB to 10,240 MB (/tmp directory)
  Payload size:         6 MB synchronous, 256 KB async (SQS trigger)

Concurrency:
  Default account limit:  1,000 concurrent executions (per region)
  Burst limit:            500-3,000 depending on region
  Reserved concurrency:   can allocate a fixed number of executions to one function
  Provisioned concurrency: pre-warm containers to eliminate cold starts

Scaling:
  SQS trigger:  Lambda scales automatically based on queue depth
                Up to 1,000 concurrent executions per queue (configurable)
                Scales up quickly: +60 instances/min initially, then faster

Pricing:
  Billed per: number of invocations + GB-seconds of execution time
  Free tier:  1M invocations/month + 400,000 GB-seconds/month
  Very cheap for sporadic, bursty workloads
```

### Cold starts — what they are and when they matter

```
Cold start = time to initialise a new Lambda container before running code

Components of cold start:
  Container spin-up:      ~100ms (Lambda managed)
  Runtime init (Node.js): ~50-200ms
  Your init code:         DB connection pool setup, config loading
                          (runs once per container, not per invocation)

Cold starts are problematic when:
  - Latency-sensitive (user is waiting for a response)
  - Bursty traffic (many simultaneous cold starts)

In our systems:
  Cold starts are NOT a concern — cases go through SQS queues
  Latency tolerance: seconds to minutes, not milliseconds
  SQS-triggered Lambdas can tolerate cold start overhead entirely
```

### Use cases (general)

```
✓ Event-driven processing (SQS, S3, EventBridge triggers)
✓ API backends (behind API Gateway)
✓ Scheduled jobs (EventBridge cron)
✓ Data transformation pipelines
✓ Webhook handlers
✓ Async background tasks

✗ Long-running processes (> 15 minutes)
✗ Persistent stateful workloads (use ECS/EKS instead)
✗ Very high sustained throughput (thousands of req/sec sustained — EC2 is cheaper)
✗ Workloads needing more than 10GB memory
```

### Why Lambda was chosen for these systems

```
Reason 1 — Operational simplicity
  No servers to patch, scale, or monitor
  Each state of the processing pipeline is one Lambda function
  Infrastructure scales automatically with case volume

Reason 2 — Per-invocation isolation
  Each case is processed in its own Lambda invocation
  A Lambda failure for one case does not affect any other case
  Natural blast-radius isolation

Reason 3 — Elastic scaling
  Case volume is bursty (end-of-month reporting surges)
  Lambda scales from 0 to hundreds of concurrent executions automatically
  No idle compute cost during quiet periods

Reason 4 — Cost model
  Low and variable case volumes (thousands/day currently)
  Lambda's per-invocation pricing is far cheaper than dedicated servers
  For ~50,000 cases/day × 7 Lambdas = 350,000 invocations/day
  At ~128MB memory, ~2s average duration:
  350,000 × 128/1024 GB × 2s = 87,500 GB-seconds/day
  Cost: ~$0.14/day — essentially free at current scale

Reason 5 — Timeout tolerance
  Case processing states (form parsing, Argus submission) can take 30s-5min
  Lambda's 15-minute timeout is more than sufficient
  For the report distribution module, Step Functions with WaitForTaskToken
  removes even this constraint — Lambda signals completion, not a timeout
```

### Idempotency — critical for SQS-triggered Lambdas

```
SQS delivers messages AT LEAST ONCE.
In rare cases, the same message may be delivered twice.
Lambda must be idempotent — running twice must produce the same result.

Pattern used in our systems:
  Before doing work:
    existing = db.query("SELECT id FROM generation_records
                         WHERE case_id=? AND profile_id=?")
    if existing: sendTaskSuccess with existing data; return early

  This prevents:
    - Duplicate report generation
    - Duplicate S3 uploads
    - Duplicate Argus submissions
    - Duplicate DB rows
```

---

## 2. AWS SQS (Simple Queue Service)

### What it is

AWS SQS is a **fully managed message queue service**. Producers send messages to a queue. Consumers (Lambda functions, EC2 instances, ECS tasks) poll the queue and process messages. SQS decouples the producer from the consumer — the producer does not need to know who processes the message or when.

### Two queue types

```
Standard Queue:
  Throughput:   Unlimited (effectively)
  Delivery:     At-least-once (rare duplicates possible)
  Ordering:     Best-effort (not guaranteed)
  Use for:      High-throughput decoupling where order doesn't matter
                (our systems use this)

FIFO Queue:
  Throughput:   300 messages/sec (3,000 with batching)
  Delivery:     Exactly-once (deduplication within 5-min window)
  Ordering:     Guaranteed (strict FIFO per message group)
  Use for:      Order-sensitive processing, financial transactions
```

### Key specifications and limits

```
Message limits:
  Size:                 256 KB maximum per message
                        (our systems send { caseId } = ~50 bytes — trivial)
  Retention period:     1 minute to 14 days (our systems: 1-2 days)
  In-flight messages:   120,000 Standard / 20,000 FIFO
  Delay:                0-15 minutes per message (optional delivery delay)

Visibility timeout:
  Default:              30 seconds (our systems: 30-45 minutes)
  Range:                0 seconds to 12 hours
  Purpose:              How long a message is invisible after being received
                        (prevents other consumers from picking it up while
                        the first consumer is working)
  On Lambda failure:    Lambda does not delete message → timeout expires
                        → message becomes visible again → Lambda retried

Polling:
  Long polling:         Wait up to 20 seconds for a message (reduces empty polls)
  Short polling:        Return immediately (may return empty responses)
  Lambda trigger uses long polling automatically

Throughput (Standard queue):
  Virtually unlimited — AWS handles partitioning internally
  Tested at millions of messages/second across AWS fleet
  Individual queue: practically unlimited at our scale
```

### The visibility timeout — the key retry mechanism

```
Timeline of a message being processed by Lambda:

t=0:    Message arrives in SQS queue
t=0:    Lambda trigger polls queue, receives message
t=0:    Message becomes INVISIBLE (visibility timeout starts: e.g. 45 min)
t=0→t?: Lambda processes the message (form parsing, Argus submission, etc.)

Scenario A — SUCCESS:
  Lambda completes, calls sqs.deleteMessage
  Message permanently deleted from queue
  No retry

Scenario B — LAMBDA CRASH / TIMEOUT (infra error):
  Lambda invocation ends without deleting message
  t=45min: visibility timeout expires
  Message becomes VISIBLE again
  Lambda trigger picks it up → new invocation begins
  Lambda re-fetches state from DB, retries from scratch

Scenario C — CONTROLLED ERROR (business logic):
  Lambda catch block: update DB with error, invoke rule-transition
  Lambda calls sqs.deleteMessage explicitly
  Message deleted — no SQS retry
  rule-transition lambda handles routing logic instead

This distinction is critical:
  Infra errors → let SQS retry (don't delete the message)
  Business errors → delete message + invoke rule-transition
```

### Dead Letter Queue (DLQ)

```
A DLQ is a separate SQS queue that receives messages which have
exceeded their maximum receive count (maxReceiveCount).

Configuration:
  Main queue → DLQ (configured as a redrive policy)
  maxReceiveCount: e.g. 5 (after 5 failed receives → goes to DLQ)

Without DLQ (current gap in our systems):
  Message retried until Message Retention Period expires (1-2 days)
  Then: silently dropped
  Case: stuck indefinitely, no alert

With DLQ (planned improvement):
  After maxReceiveCount failures → message moves to DLQ automatically
  DLQ triggers a Lambda → mark case as UNRESOLVED_ERROR → alert ops team
  Eliminates all silent drops

DLQ best practice:
  Every production SQS queue should have a DLQ
  Monitor DLQ depth as a critical operational metric
  DLQ depth > 0 should trigger an immediate alert
```

### Why SQS was chosen

```
Reason 1 — Decoupling
  Lambdas do not directly invoke each other
  The queue is the interface contract — producer/consumer are independent
  Adding a new state = new Lambda + new queue + DB config row
  No changes to existing Lambdas

Reason 2 — Automatic retry for infra failures
  Lambda crashes, times out, or runs out of memory?
  The message remains in the queue and is retried after visibility timeout
  No code needed for retry logic — SQS provides it inherently
  In a direct Lambda-to-Lambda call, a crash = permanent case loss

Reason 3 — Backpressure and load levelling
  If case volume spikes (end-of-month surge), messages queue up
  Lambda processes them at its concurrency limit
  The queue absorbs the spike — no case is dropped
  Without a queue, a spike would directly hit Lambda concurrency limits
  and trigger throttling (HTTP 429 errors)

Reason 4 — Normalised interface
  Every queue receives the same payload: { caseId }
  Adding a new data requirement for a Lambda = add DB column, not change message
  The message format is stable forever

Reason 5 — Operational visibility
  Queue depth = how many cases are waiting at each state
  ApproximateNumberOfMessages → CloudWatch metric → alert when depth too high
  Gives real-time visibility into processing backlog per state

Alternative considered — Amazon EventBridge:
  ✗ Fan-out model doesn't fit a sequential pipeline naturally
  ✗ Rule-based routing would require complex event patterns
  ✗ No native visibility timeout retry semantics
  ✓ Better for event-driven fan-out to multiple consumers
  → SQS was correct for sequential pipeline; EventBridge for broadcast
```

---

## 3. AWS Step Functions

### What it is

AWS Step Functions is a **serverless orchestration service** that coordinates multiple services (Lambda, SQS, DynamoDB, etc.) into a visual workflow called a **state machine**. Each step in the state machine can invoke an AWS service, wait for a response, branch conditionally, retry on failure, or run steps in parallel.

### How it works

```
State machine definition: JSON/YAML (Amazon States Language — ASL)
  Defines:
    - States (Task, Wait, Choice, Parallel, Map, Pass, Succeed, Fail)
    - Transitions between states
    - Retry and catch configuration per state
    - Input/output filtering

Execution:
  Start execution → provide input JSON
  Step Functions executes states in order
  Each state can: invoke Lambda, SQS, DynamoDB, ECS, etc.
  Execution history stored for 90 days (Standard) / no history (Express)
```

### Two types of Step Functions

```
Standard Workflow:
  Execution duration:   Up to 1 year
  Execution model:      At-most-once (no duplicate executions)
  History:              Full execution history stored (queryable in console)
  Pricing:              Per state transition (~$0.025 per 1,000 transitions)
  Use for:              Long-running, auditable workflows (our systems use this)

Express Workflow:
  Execution duration:   Up to 5 minutes
  Execution model:      At-least-once (duplicates possible)
  History:              CloudWatch Logs only (no console history)
  Pricing:              Per execution + duration (cheaper for high-volume, short)
  Use for:              High-volume, short-duration event processing
```

### Key state types used in our systems

```
Task state:
  Invokes an AWS service (Lambda, SQS, DynamoDB, etc.)
  Can be synchronous (wait for completion) or async (WaitForTaskToken)
  Supports retry (exponential backoff) and catch (on error, go to X state)

Map state:
  Iterates over an array of items
  Runs each iteration in parallel (or concurrently with MaxConcurrency)
  Waits for ALL iterations to complete before moving to next state
  Used in our systems for:
    ReportGenerator: iterate over formats (XML, PDF) — generate in parallel
    ReportSender: iterate over destinations — send in parallel

WaitForTaskToken pattern:
  Step Function pauses at the Task state
  Passes a unique task token to the invoked service
  Service performs async work (may take seconds to minutes)
  Service calls StepFunctions.SendTaskSuccess(taskToken, output)
         OR StepFunctions.SendTaskFailure(taskToken, error, cause)
  Step Function resumes

  This is the core mechanism for async Lambda coordination in our Report
  Distribution Module — lambda generates XML, uploads to S3, THEN signals
  the Step Function. The Step Function does not poll or time out waiting.
```

### WaitForTaskToken — deep dive

```
Why it matters for our system:

Problem without it:
  XML generation takes 30 seconds
  Argus submission takes 2 minutes
  If Step Function calls Lambda synchronously:
    → Lambda timeout is 15 minutes (could work for generation)
    → But Step Function has its OWN timeout per state (also configurable)
    → More importantly: no parallelism across formats — sequential only

With WaitForTaskToken + Map state:
  Map state spawns 2 iterations: [XML, PDF]
  Both iterations invoke task-invoker Lambda simultaneously
  task-invoker sends message to xml-generator queue + task token
  task-invoker sends message to pdf-generator queue + task token
  xml-generator does its work (30s), calls SendTaskSuccess(token)
  pdf-generator does its work (45s), calls SendTaskSuccess(token)
  Map state waits until BOTH tokens are returned
  → True parallel async execution
  → No polling
  → No Lambda sitting idle waiting for another Lambda

Task token characteristics:
  Unique per state transition per execution
  Used to match a callback to its waiting state
  No expiry (the Step Function just waits indefinitely)
  Must be passed through to the worker lambda and returned with output
```

### Key limits

```
Execution limits:
  Max execution duration:     1 year (Standard)
  Max state transitions:      No limit
  Max execution history:      25,000 events per execution
  Concurrent executions:      Default 2,500-5,000 (can be increased)

Payload limits:
  Max state input/output:     256 KB
  (our systems pass minimal data — S3 keys and IDs, not file content)

Pricing (Standard):
  ~$0.025 per 1,000 state transitions
  For our system: 1 execution ≈ ~15-20 state transitions
  At 50,000 cases/day: 50,000 × 20 = 1M transitions/day
  Cost: ~$25/day — very reasonable for a regulated compliance system
```

### Why Step Functions was chosen for Report Distribution

```
Reason 1 — Parallel execution across formats and destinations
  Map state + WaitForTaskToken = native parallel async execution
  Generating XML and PDF simultaneously instead of sequentially
  Sending to Argus and 4 email destinations simultaneously
  Reduces total distribution time from sum-of-steps to max-of-steps

Reason 2 — Visual execution history for audit and debugging
  Every execution is recorded with input/output at each state
  When a case fails in distribution, open the Step Functions console
  → see exactly which state failed, what the input was, what the error was
  Invaluable for a compliance system where every failure must be traceable

Reason 3 — Built-in retry and error handling
  Each state has configurable retry (exponential backoff, max attempts)
  Catch blocks: "if this state fails, go to this error state"
  No need to write retry logic in Lambda code

Reason 4 — WaitForTaskToken enables true async orchestration
  Lambdas signal completion on their own schedule
  Step Function waits without polling, without timeouts, without cost
  Critical for Argus submission which has variable response times

Reason 5 — Execution isolation
  Each case gets its own Step Function execution
  A failure in one execution has zero impact on other executions

Alternative considered — custom orchestration in Lambda + DB:
  Could have tracked "all formats generated?" in DB with a Lambda polling loop
  ✗ More code to write and maintain
  ✗ Polling adds latency and Lambda cost
  ✗ No visual execution history
  ✗ Race conditions in the "are we done?" check
  → Step Functions gives all of this for free
```

---

## 4. PostgreSQL

### What it is

PostgreSQL is an open-source **relational database management system (RDBMS)**. It stores data in tables with rows and columns, enforces relationships between tables (foreign keys), supports ACID transactions, and provides SQL for rich querying.

### Key features relevant to our systems

```
ACID transactions:
  Atomicity:   All operations in a transaction succeed or all fail
  Consistency: Database always moves from one valid state to another
  Isolation:   Concurrent transactions don't interfere with each other
  Durability:  Committed transactions survive crashes

Why ACID matters here:
  Case state updates must be atomic — can't have a case partially updated
  Rule evaluation depends on consistent state — can't read mid-update data
  Regulatory compliance requires that DB state is always correct

JSONB column type:
  Stores JSON as binary (faster querying than plain JSON)
  Supports GIN indexing on JSON fields
  Used for: case_json (canonical adverse event data),
            destination.config (connection details vary by type),
            transition_rules.data_point_conditions (flexible conditions)
  Allows schema flexibility within a structured relational context

Partial indexes:
  CREATE INDEX idx_expires ON urls(expires_at) WHERE expires_at IS NOT NULL
  Index only covers rows matching the WHERE condition
  Used for: case expiry cleanup, active-only rule queries
  Much smaller and faster than full column indexes for sparse data

ON CONFLICT (upsert):
  INSERT INTO ... ON CONFLICT (case_id, profile_id) DO UPDATE SET ...
  Critical for idempotency — if Lambda retried and record already exists,
  update it rather than failing with unique constraint violation
```

### Capacity and performance

```
Throughput:
  Single primary (m5.large, 8GB RAM):
    Reads:  ~10,000-50,000 simple reads/sec (index lookups)
    Writes: ~5,000-10,000 simple writes/sec
    Complex queries: hundreds/sec (depends on query plan)

  With read replicas:
    Read throughput scales linearly with replicas added
    Each replica handles the same read throughput as the primary

Connection limits:
  Default max connections: 100-300 depending on instance size
  PgBouncer (connection pooler): allows thousands of app connections
    to share a pool of actual DB connections
  Lambda + RDS: use RDS Proxy (AWS managed connection pooler)
    prevents Lambda scale-out from exhausting DB connections

Storage:
  Row size: no hard limit, practically up to 1 GB per row
  Table size: no hard limit (petabytes possible with partitioning)
  Index size: scales with table size

At our scale (50,000 cases/day × 10 DB writes/case = 500,000 writes/day):
  = ~6 writes/sec average
  = ~60 writes/sec at peak
  Single PostgreSQL primary handles this trivially
```

### RDS vs self-managed PostgreSQL

```
AWS RDS PostgreSQL (managed):
  ✓ Automated backups, point-in-time recovery
  ✓ Multi-AZ for high availability (automatic failover ~60 seconds)
  ✓ Automated minor version upgrades
  ✓ RDS Proxy for Lambda connection pooling
  ✓ CloudWatch metrics built-in
  ✗ Slightly less configurable than self-managed
  ✗ More expensive than self-managed on EC2

Our systems use RDS PostgreSQL — the operational simplicity justifies the cost
for a compliance-critical system where data durability is paramount.
```

### Why PostgreSQL was chosen

```
Reason 1 — Relational data with clear relationships
  Cases, profiles, destinations, rules, workflows — all have FK relationships
  JOIN queries are natural and performant
  E.g.: fetch all rules for a case's workflow state with one JOIN query

Reason 2 — ACID for compliance
  In a regulated PV system, partial writes are unacceptable
  Case state must be updated atomically with audit records
  PostgreSQL's transaction isolation prevents race conditions

Reason 3 — JSONB for flexibility where schema varies
  Case data varies by form type
  Destination configuration varies by destination type (email vs Argus)
  Transition rule conditions vary by rule complexity
  JSONB columns store these variable structures while the surrounding
  schema remains strictly relational and queryable

Reason 4 — SQL for operational queries
  "How many cases are stuck at Form Parsing in ERROR state for > 24 hours?"
  "Which rules have been triggered most in the last 7 days?"
  Rich ad-hoc SQL queries for operations team — no specialised tools needed

Reason 5 — Mature, well-understood, widely supported
  PV systems have strict software validation requirements
  PostgreSQL's stability, decade-long production track record, and extensive
  documentation satisfy validation requirements better than newer databases

Alternative considered — DynamoDB:
  ✓ Serverless, auto-scaling, no connection limits
  ✗ No JOINs — complex queries require multiple reads + application-level join
  ✗ Eventual consistency by default — risky for case state management
  ✗ Schema design upfront is harder for the complex relational data model
  → PostgreSQL's relational model was the correct fit
```

---

## 5. AWS S3 (Simple Storage Service)

### What it is

AWS S3 is an **object storage service** — a highly durable, infinitely scalable store for files (objects) of any size. Objects are stored in buckets and addressed by a key (a path-like string). S3 is not a filesystem — there are no folders, just keys with `/` in their names that look like paths.

### Key specifications

```
Object limits:
  Single object size:   5 TB maximum
  Single PUT:           5 GB maximum
  Multipart upload:     for objects > 100 MB (recommended), up to 5 TB
  Key length:           1,024 bytes maximum

Durability and availability:
  Durability:    99.999999999% (11 nines) — designed to lose 0.000000001% of
                 objects/year across the S3 fleet
                 Achieved by replicating objects across ≥ 3 AZs within a region
  Availability:  99.99% SLA for Standard storage class

Throughput:
  No defined per-bucket limit for reads or writes
  AWS scales S3 internally — thousands of requests/second per prefix
  For very high throughput: use random key prefixes to distribute load
  At our scale: trivial — hundreds of objects/day

Latency:
  First-byte latency: 100-200ms typical for small objects
  Not suitable for ultra-low-latency use cases
  Fine for async processing pipelines (our use case)

Storage classes (cost tiers):
  Standard:           Frequent access. ~$0.023/GB/month
  Infrequent Access:  Accessed less than once/month. ~$0.0125/GB/month
  Glacier:            Archival, retrieval in minutes-hours. ~$0.004/GB/month
  Glacier Deep:       Long-term archival, retrieval in 12 hours. ~$0.00099/GB/month
```

### Pre-signed URLs

```
A pre-signed URL is a time-limited URL that grants temporary access to a
specific S3 object without requiring AWS credentials.

Generated by:
  s3.getSignedUrl('getObject', { Bucket, Key, Expires: 3600 })
  → URL valid for 1 hour, can be used by anyone

Used in our systems for:
  Report Distribution: generate a pre-signed GET URL so the sender lambda
  can download the XML/PDF for transmission without needing AWS SDK or credentials
  in the sending component
  Also used in ingestion pipeline for allowing form attachments to be
  fetched by processing lambdas
```

### Why S3 was chosen for report and JSON storage

```
Reason 1 — File size
  E2B XML reports: 10KB-500KB
  CIOMS PDF reports: 100KB-5MB
  Raw case JSON: 50KB-2MB
  PostgreSQL BLOBs/text fields are technically possible but
  storing large files in a relational DB degrades query performance
  and backup/restore times significantly.
  S3 is purpose-built for blob/file storage.

Reason 2 — Regulatory retention requirement
  Adverse event reports must be retained indefinitely (decades)
  S3 Lifecycle policies can automatically transition:
    Standard → Infrequent Access after 90 days (~46% cost reduction)
    Infrequent Access → Glacier after 1 year (~68% further reduction)
  Retention managed as config, not application code

Reason 3 — Durability
  11 nines of durability — regulatory bodies require that submission
  records are never lost. S3's durability exceeds what any self-managed
  file system or DB BLOB storage could provide at comparable cost.

Reason 4 — Decoupling storage from compute
  XML generator Lambda writes to S3 → returns S3 key
  Safety system sender Lambda reads from S3 using that key
  The two Lambdas never need to coordinate directly or share memory
  S3 is the hand-off medium — works perfectly with async pipelines

Reason 5 — Versioning
  S3 versioning can be enabled per bucket
  If a report is regenerated (e.g., corrected XML), both versions are retained
  Useful for audit: "what exactly was submitted to Argus on date X?"

Reason 6 — Server-side encryption
  SSE-S3 or SSE-KMS encrypts all objects at rest automatically
  Required for PHI (Protected Health Information) under data protection regulations
  (Adverse event data contains patient demographics and medical information)
```

---

## 6. AWS SES v2 (Simple Email Service)

### What it is

AWS SES (Simple Email Service) v2 is a **cloud-based email sending service**. It handles the infrastructure for sending email at scale — SMTP, DNS records (SPF, DKIM, DMARC), bounce and complaint handling, dedicated IP management, and deliverability monitoring.

### Key specifications

```
Sending limits:
  Sandbox mode (new accounts): 200 emails/day, 1/sec
  Production mode (after review): starts at 50,000/day
  Can be increased on request to millions/day

Sending rate:
  Default: 14 emails/sec (can be increased)
  At our scale: 4-5 destinations × thousands of cases/day
  = tens of thousands of emails/day — well within default limits

Email size:
  Maximum: 40 MB (including attachments)
  Our PDF reports: 100KB-5MB — comfortably within limit

Features:
  Attachments:        Yes — MIME format, any file type
  HTML/plain text:    Both supported
  Bounce handling:    Automatic — suppression list for bouncing addresses
  Complaint handling: Automatic — unsubscribe on spam complaint
  DKIM signing:       Automatic — improves deliverability
  Dedicated IPs:      Optional — for high-volume senders needing reputation control
  Templates:          Email templates stored in SES, rendered server-side
  Configuration sets: Group sends for tracking/monitoring

SES v2 vs SES v1:
  v2 adds: bulk email support, contact list management, better analytics
  Our systems use v2 for the improved attachment handling and monitoring APIs
```

### Why SES v2 was chosen for email distribution

```
Reason 1 — Regulatory email reliability
  Pharmacovigilance email distributions go to regulatory teams, safety officers,
  and department heads. Deliverability is critical — a missed email can be a
  regulatory issue.
  SES handles SPF/DKIM/DMARC automatically → high deliverability
  Bounce and complaint handling → maintain sender reputation

Reason 2 — Attachment support
  Reports (PDF, XML) are sent as email attachments
  SES v2 handles MIME encoding and attachment delivery natively
  No need to host attachments separately and include links

Reason 3 — Integration with Lambda
  AWS SDK call from Lambda:
  ses.sendEmail({ From, To, Subject, Body, Attachments })
  Simple, native, no external dependencies or credentials to manage

Reason 4 — Cost
  $0.10 per 1,000 emails
  At 50,000 cases/day × 4 email destinations = 200,000 emails/day
  Cost: $20/day — negligible for a regulated enterprise system

Reason 5 — Audit capability
  SES v2 provides send events (sent, delivered, bounced, complained)
  These can feed into CloudWatch or EventBridge
  Failed delivery alerts can trigger case error-marking

Alternative considered — Internal SMTP server / Microsoft Exchange:
  ✓ Company may already have Exchange
  ✗ Lambda → Exchange requires network configuration (VPC, firewall)
  ✗ Exchange reliability, capacity, and monitoring are ops team's concern
  ✗ Bounce/complaint handling is manual
  → SES v2 is simpler, more reliable, and cheaper for this use case
```

---

## 7. TypeScript on Lambda

### Why TypeScript over plain JavaScript

```
JavaScript problems at scale:
  No type safety — function receives any shape of object, no compile-time check
  DB query returns untyped rows — easy to misuse fields
  Refactoring is risky — no IDE support for "find all usages of this field"
  Runtime type errors are discovered in production, not at compile time

TypeScript solves these:
  Compile-time type checking — catch errors before deployment
  IDE autocomplete and refactoring support
  Interfaces for DB row types, Lambda event types, Step Function payloads
  Strict null checks — forces handling of undefined/null explicitly
  Generic types — the task-invoker pattern, the next-state helper are
  more safely expressed with TypeScript generics
```

### TypeScript on Lambda — practical details

```
Build process:
  TypeScript → tsc / esbuild → JavaScript (Lambda runs Node.js, not TypeScript)
  esbuild is preferred: 100× faster than tsc, produces smaller bundles
  Bundle size matters: smaller = faster cold starts

  Typical build:
    esbuild --bundle src/handler.ts --outfile dist/index.js
    --platform node --target node18 --minify

Runtime: Node.js 18.x or 20.x on Lambda
  AWS-managed, automatically patched for security vulnerabilities

Type definitions used:
  @types/aws-lambda     — Lambda handler event types (SQSEvent, etc.)
  @types/pg             — PostgreSQL client types
  @aws-sdk/client-sqs   — AWS SDK v3 (tree-shakeable, smaller bundle)
  @aws-sdk/client-s3
  @aws-sdk/client-sfn   — Step Functions client
  @aws-sdk/client-sesv2
```

### Lambda cold start optimisation with TypeScript

```
Initialisation code (outside the handler function) runs ONCE per container:

// This runs once per container (cold start only)
const dbPool = new Pool({ connectionString: process.env.DB_URL });
const sqsClient = new SQSClient({ region: 'eu-west-1' });

export const handler = async (event: SQSEvent) => {
  // This runs on every invocation (warm or cold)
  const { caseId } = JSON.parse(event.Records[0].body);
  // dbPool and sqsClient are reused across invocations
};

Best practices:
  ✓ Initialise DB connections, SDK clients outside handler
  ✓ Use esbuild to minimise bundle size
  ✓ Use AWS SDK v3 (smaller, tree-shakeable vs v2)
  ✓ Avoid large dependencies (no moment.js, use date-fns)
  ✓ Use Lambda Layers for shared code (audit service client, DB helpers)
```

---

## 8. Technology Choice Comparison Matrix

### Why not alternatives?

```
COMPUTE
──────────────────────────────────────────────────────────────────────────────
Lambda (chosen)    vs    ECS / EC2              vs    Fargate
  ✓ No servers             ✓ Long-running           ✓ No servers
  ✓ Auto-scale to 0        ✓ > 15min execution      ✓ Container flexibility
  ✓ Per-ms billing         ✗ Idle cost when quiet   ✓ No 15-min limit
  ✗ 15-min max             ✗ Manual scaling         ✗ Slower cold start
  ✗ Cold starts            ✗ Patching required      ✗ Container management
Verdict: Lambda correct for event-driven, <15min, bursty workloads

MESSAGING
──────────────────────────────────────────────────────────────────────────────
SQS (chosen)       vs    SNS                    vs    EventBridge    vs  Kafka
  ✓ Queue semantics        ✓ Pub/sub fan-out         ✓ Event routing   ✓ Streaming
  ✓ Visibility timeout     ✗ No retry semantics      ✓ Complex rules   ✓ Replay
  ✓ DLQ support            ✗ No DLQ                  ✗ No queue depth  ✓ High throughput
  ✓ Per-message retry      ✗ No per-consumer retry   ✗ No visibility   ✗ Operational overhead
  ✗ Not pub/sub            ✗ Fire-and-forget         timeout           ✗ Overkill at this scale
Verdict: SQS for sequential pipeline retry; SNS/EventBridge for fan-out

ORCHESTRATION
──────────────────────────────────────────────────────────────────────────────
Step Functions     vs    Custom DB state machine  vs   Airflow / Temporal
(chosen)
  ✓ Visual history         ✓ Full control            ✓ Complex DAGs
  ✓ WaitForTaskToken       ✗ Must write all logic     ✓ Rich scheduling
  ✓ Map state parallelism  ✗ No visual history        ✗ Operational overhead
  ✓ Managed retries        ✗ Race conditions risk     ✗ Not serverless
  ✓ AWS native             ✗ More code to maintain    ✗ Separate infra to run
  ✗ Limited to AWS         ✗ Hard to debug failures   ✓ Vendor-neutral
Verdict: Step Functions correct for AWS-native, parallel async coordination

DATABASE
──────────────────────────────────────────────────────────────────────────────
PostgreSQL         vs    DynamoDB               vs    MySQL           vs  MongoDB
(chosen)
  ✓ ACID                   ✓ Serverless              ✓ ACID             ✓ Flexible schema
  ✓ Rich queries           ✓ Auto-scale              ✓ Familiar         ✓ Horizontal scale
  ✓ JOINs                  ✗ No JOINs                ✓ Good tooling     ✗ No ACID by default
  ✓ JSONB flexibility      ✗ Eventual consistency    ✗ Weaker JSONB     ✗ No JOINs
  ✓ Mature / validated     ✗ Query patterns upfront  ✓ Good community   ✓ Good for variable schema
  ✗ Manual scaling         ✓ No connection limits    ✗ Scaling same     ✗ Overkill here
Verdict: PostgreSQL correct for relational compliance data with complex queries

FILE STORAGE
──────────────────────────────────────────────────────────────────────────────
S3 (chosen)        vs    EFS (Elastic File System) vs  DB BLOBs
  ✓ 11 nines durability   ✓ POSIX filesystem            ✗ Degrades DB performance
  ✓ Infinite scale        ✓ Shared across instances     ✗ Backup/restore cost
  ✓ Lifecycle policies    ✗ Higher cost                 ✗ Not purpose-built
  ✓ Encryption at rest    ✗ NFS overhead                ✓ Single system
  ✓ Versioning            ✓ Good for shared access      ✗ Size limits
  ✓ Pre-signed URLs       ✗ Lambda mount overhead       ✗ Poor for large files
Verdict: S3 correct for durable, large-file, async pipeline storage

EMAIL
──────────────────────────────────────────────────────────────────────────────
SES v2 (chosen)    vs    SMTP / Exchange          vs   SendGrid / Mailgun
  ✓ AWS native            ✓ May already exist            ✓ Rich features
  ✓ Auto DKIM/SPF         ✗ Network config for Lambda   ✓ Good deliverability
  ✓ Bounce handling       ✗ Ops team dependency          ✗ External vendor
  ✓ Low cost              ✗ Manual scaling               ✗ Additional cost
  ✓ Lambda SDK native     ✗ Limited monitoring           ✓ Better analytics
Verdict: SES v2 correct for AWS-native, simple attachment delivery
```

---

## Quick Reference — Limits That Matter

```
Service         Limit                           Value             Impact if exceeded
──────────────────────────────────────────────────────────────────────────────────────
Lambda          Execution timeout               15 minutes        Infra failure → SQS retry
Lambda          Memory                          10 GB             OOM → Lambda crash → SQS retry
Lambda          Concurrent executions (acct)    1,000 default     Throttling → SQS queuing
Lambda          SQS payload size                256 KB            Not an issue ({ caseId } = 50B)
SQS             Message size                    256 KB            Not an issue ({ caseId } = 50B)
SQS             Message retention               14 days max       Messages expire if not processed
SQS             Visibility timeout              12 hours max      Our setting: 30-45 min
SQS             In-flight messages              120,000 Standard  Not an issue at our scale
Step Functions  Execution duration              1 year            Not an issue
Step Functions  State transition payload        256 KB            Use S3 keys not file contents
Step Functions  Concurrent executions           Default 5,000     Not an issue at our scale
PostgreSQL      Max connections (RDS m5.large)  ~600              Use RDS Proxy for Lambda
S3              Single object size              5 TB              Not an issue (reports < 10MB)
S3              PUT request rate                3,500/sec/prefix  Not an issue at our scale
SES v2          Send rate (default)             14 emails/sec     Not an issue at our scale
```

---

## One-liner Rationale for Each Technology

For use in interviews when asked "why X?":

```
AWS Lambda:
  "Event-driven, serverless compute that scales automatically with case volume,
  costs nothing when idle, and provides natural isolation — each case processed
  in its own invocation. The 15-minute timeout is sufficient for all our
  processing states."

AWS SQS:
  "Fully decouples our processing states — no Lambda knows about another.
  The visibility timeout provides automatic retry for infrastructure failures
  at zero additional code. Queue depth gives us natural backpressure and
  real-time backlog visibility."

AWS Step Functions:
  "Orchestrates parallel async execution of report generation and transmission
  using WaitForTaskToken and Map state. Visual execution history is invaluable
  for compliance — we can see exactly what happened in every distribution
  workflow. Managed retries and catch blocks reduce orchestration code."

PostgreSQL:
  "Relational data with clear FK relationships between cases, profiles,
  destinations, rules, and workflows. ACID transactions are non-negotiable
  for a regulated system where partial writes are a compliance risk.
  JSONB columns handle the variable parts of our schema without losing
  the benefits of relational structure."

AWS S3:
  "Purpose-built object storage with 11-nines durability — regulatory
  requirements mandate indefinite retention of submitted reports. S3 Lifecycle
  policies automate tiering to cheaper storage classes over time. Objects are
  naturally suited to async pipeline hand-offs between Lambdas."

AWS SES v2:
  "Managed email infrastructure with native Lambda integration, automatic
  DKIM/SPF for deliverability, and built-in bounce handling. At our volume,
  it costs less than $20/day while being far simpler than maintaining
  an SMTP relay inside our infrastructure."

TypeScript:
  "Type safety at compile time catches errors before they reach production
  in a compliance-critical system. Strict typing of DB row types, Step Function
  payloads, and Lambda events makes refactoring safe and the codebase
  self-documenting."
```

---

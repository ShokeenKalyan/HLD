# System Design — Case Reprocessing Framework (Pharmacovigilance)

> **Domain:** Pharmacovigilance (drug safety reporting)  
> **Role:** Senior Backend Lead  
> **Type:** Real system — production design with architectural evolution, design decisions, trade-offs, and improvement opportunities  
> **Core problem:** Decouple a multi-state case processing pipeline from tight lambda-to-lambda coupling, enable SQS-driven automatic retry on infra failures, and introduce a rule-based transition framework that allows cases to be re-routed to any workflow state without re-ingestion.

---

## Table of Contents

1. [Domain Context and Problem Statement](#1-domain-context-and-problem-statement)
2. [Requirements](#2-requirements)
3. [Scale and Capacity](#3-scale-and-capacity)
4. [System Context — The Case Lifecycle](#4-system-context--the-case-lifecycle)
5. [Architecture Evolution — Before and After](#5-architecture-evolution--before-and-after)
6. [High-Level Architecture](#6-high-level-architecture)
7. [Component Deep Dives](#7-component-deep-dives)
8. [Data Model](#8-data-model)
9. [Key Design Decisions and Rationale](#9-key-design-decisions-and-rationale)
10. [Rule-Based Transition Framework](#10-rule-based-transition-framework)
11. [Failure Handling and Reprocessing Scenarios](#11-failure-handling-and-reprocessing-scenarios)
12. [Trade-offs and Known Limitations](#12-trade-offs-and-known-limitations)
13. [Future Improvements](#13-future-improvements)
14. [Interview Script](#14-interview-script)
15. [Follow-up Probes and Answers](#15-follow-up-probes-and-answers)

---

## 1. Domain Context and Problem Statement

### What is a case in this system?

A **case** is an Individual Case Safety Report (ICSR) — a structured record of an adverse drug reaction. Cases arrive from multiple external sources in multiple formats. Each case must be ingested, parsed, cleaned, validated, and then distributed as a regulated report to safety authorities (Argus, EudraVigilance) within strict regulatory deadlines.

### Sources and form types

```
Sources:
  Email mailboxes       ← attachments forwarded by reporters
  FTP locations         ← bulk transfers from partners
  Web portals           ← HA-Canada, EV-Web (EudraVigilance Web)
  Cloud storage         ← Box, SharePoint integrations

Form types:
  CIOMS Form I          ← PDF, standardised paper form scanned/digital
  MedWatch 3500A        ← PDF, FDA form
  E2B R2 / R3 XML       ← structured XML, already machine-readable
  HAC Excel             ← Health Canada proprietary Excel format
  (others per client)
```

### The original problem

Before this design, the processing pipeline was a chain of direct Lambda-to-Lambda invocations:

```
email-reader → identification → form-processor → data-cleaner → case-saver → ...
```

This created three critical problems:

**Problem 1 — Tight coupling:** Each lambda was hardwired to invoke the next. Changing the processing sequence for a new form type or client required code changes and redeployment.

**Problem 2 — No fault tolerance:** An infrastructure failure (Lambda timeout, memory error, downstream service blip) at any state permanently interrupted the case. No automatic retry. No recovery path without re-ingesting the case from scratch.

**Problem 3 — No re-routing capability:** If a case errored at Form Parsing (e.g., unrecognised form layout), the only recovery was to re-ingest the original source document. There was no mechanism to route a case to a different state (e.g., Manual Review) based on what went wrong.

---

## 2. Requirements

### Functional Requirements

- Ingest cases from email, FTP, web portals, and cloud storage
- Process cases through a configurable sequence of states: Registration → Identification → Form Parsing → Data Cleaning → Case Saving → (Manual Review) → Report Distribution → Closure
- Support different workflow paths per tenant, source, and form type — E2B XML may skip Data Cleaning; a specific client may add an extra validation state
- Automatically retry a state if an infra or unhandled error occurs — without human intervention
- Route cases to alternative states (non-linear paths) based on configurable rules when a state errors — without re-ingesting the case
- Allow users to manually route a case to any valid state from the UI
- Allow users to edit case data at manual states before re-routing
- Support circular routing paths (e.g., Report Distribution → Error → Reprocess → Report Distribution again after fix)
- Maintain full visibility — current state, sub-state, error reason, history — for every case

### Non-functional Requirements

- **Decoupling:** no lambda should directly invoke another lambda
- **Fault tolerance:** infra failures must be automatically retried without case loss
- **Configurability:** workflow paths, state sequences, and transition rules must be configurable from UI without engineering involvement
- **Normalised interface:** all state-processing lambdas receive only `{ caseId }` — no routing logic in the message payload
- **Scalability:** must handle current thousands of cases/day scaling to hundreds of thousands

### Constraints

- No case should ever need to be re-ingested from the source due to a processing error
- Workflow configuration must be manageable by a Business Analyst or Admin via UI
- The framework must be generic — any new source, form, or state must fit within the same pattern without architectural changes

---

## 3. Scale and Capacity

```
Current:
  Active clients:      3-4
  Cases/day:           thousands (~1,000-5,000)
  States per case:     5-8 automated states + 1-2 manual states
  SQS messages/case:   5-8 (one per automated state traversal)
  SQS messages/day:    ~25,000-40,000

Near-term:
  Cases/day:           tens of thousands (~10,000-50,000)
  SQS messages/day:    ~250,000-400,000

Far future:
  Cases/day:           hundreds of thousands
  SQS messages/day:    ~millions

Error/reprocessing rate:
  Production (fully configured): < 1% of cases error
  Most errors: Argus connectivity, infra spikes at Report Distribution / Ack Pending
  Pre-extraction errors: very rare — occasional business logic edge cases

SQS message size:
  { caseId: "uuid" } = ~50 bytes
  Trivially small — no payload size concerns

Lambda concurrency:
  Each state lambda handles one case at a time per invocation
  Multiple invocations run in parallel for concurrent cases
  Concurrency limit: per-queue Lambda trigger concurrency setting
```

---

## 4. System Context — The Case Lifecycle

```
INGESTION
─────────
External Source
  (email / FTP / portal / cloud storage)
          │
          ▼
  Source Reader Lambda              ← email-reader, portal-reader, ftp-reader
  (triggered by polling or webhook)
          │
          ▼

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  CASE PROCESSING PIPELINE (this design)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  State 1: Case Registration
    → fetch source attachment, create case record in DB

  State 2: Form Identification
    → identify form type from pre-configured identifiers in DB

  State 3: Form Parsing
    → extract raw case data from PDF/XML/XLSX/DOC → raw JSON → S3

  State 4: Data Cleaning       ← OPTIONAL (skipped for E2B XML)
    → clean, normalise, map to canonical case JSON → S3

  State 5: Case Saving
    → fetch final JSON from S3, persist all data fields to DB

  State 6+: Manual Review (optional, configured per workflow)
    → user reviews and edits case data from UI

  State 7: Report Distribution
    → generate XML/PDF reports, transmit to Argus/email destinations
    (full design covered in System 1 — Report Distribution Module)

  State 8: Ack Pending
    → await acknowledgement from Argus / regulatory authority

  State 9: Case Closed
    → terminal state

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

NON-LINEAR PATHS (rule-driven)
───────────────────────────────
  Any errored state → rule-transition lambda → configured alternative state
  Examples:
    Form Identification error → Manual Review (user enters form details)
    Form Parsing error        → Manual Review (user uploads corrected form)
    Report Distribution error → Reprocess Case (parking state → re-route after fix)
    Ack Pending error         → Reprocess Case (parking state → re-route after fix)
```

---

## 5. Architecture Evolution — Before and After

### Before — tight lambda coupling

```
BEFORE (tightly coupled, hardcoded):

email-reader
    │
    └──direct invoke──▶ identification
                              │
                              └──direct invoke──▶ form-processor
                                                        │
                                                        └──direct invoke──▶ data-cleaner
                                                                                  │
                                                                                  └──▶ case-saver

Problems:
  ✗ Adding a new state = change multiple lambdas
  ✗ Skipping a state (E2B skips data-cleaner) = conditional code in lambdas
  ✗ Infra failure at any point = case stuck, manual re-ingestion required
  ✗ Different workflow per client = branching logic scattered across lambdas
  ✗ Message payload carried routing context = fragile, hard to evolve
```

### After — SQS-decoupled, DB-driven routing

```
AFTER (decoupled, configuration-driven):

email-reader
    │
    └──SQS(registration-queue)──▶ registration-lambda
                                          │
                                          │ update DB: case.current_state = 'IDENTIFICATION'
                                          │ lookup next queue from meta-schema
                                          └──SQS(identification-queue)──▶ identification-lambda
                                                                                  │
                                                                                  │ update DB
                                                                                  │ lookup next queue
                                                                                  └──SQS(...)──▶ ...

Benefits:
  ✓ Each lambda knows only its own queue — zero knowledge of neighbours
  ✓ Routing is DB-driven — change workflow = DB update, no code change
  ✓ Skip a state = simply don't include it in the workflow transitions table
  ✓ SQS visibility timeout = automatic retry on infra failure
  ✓ All lambdas share the same interface: receive { caseId }, fetch context from DB
  ✓ Rule-transition lambda can intercept any errored state and re-route
```

---

## 6. High-Level Architecture

```
                    ┌────────────────────────────────────────────────────────────┐
                    │              CASE PROCESSING FRAMEWORK                      │
                    │                                                             │
  External Source   │                                                             │
  (email/FTP/       │  ┌─────────────┐                                           │
   portal/cloud)    │  │source-reader│  email-reader / portal-reader / ftp-reader│
       │            │  │  Lambda     │◀── polling / webhook trigger               │
       └───────────▶│  └──────┬──────┘                                           │
                    │         │  SQS(registration-queue) ← { caseId }            │
                    │         ▼                                                   │
                    │  ┌──────────────────────────────────────────────────────┐  │
                    │  │              AUTOMATED STATE CHAIN                    │  │
                    │  │                                                       │  │
                    │  │  ┌──────────────┐   SQS → ┌──────────────────┐       │  │
                    │  │  │ registration │──────────▶ identification   │       │  │
                    │  │  │   Lambda     │           │    Lambda        │       │  │
                    │  │  └──────────────┘           └────────┬─────────┘      │  │
                    │  │                                      │ SQS            │  │
                    │  │                                      ▼                │  │
                    │  │                             ┌──────────────────┐      │  │
                    │  │                             │  form-processor  │      │  │
                    │  │                             │    Lambda        │      │  │
                    │  │                             └────────┬─────────┘      │  │
                    │  │                                      │ SQS            │  │
                    │  │                                      ▼                │  │
                    │  │                             ┌──────────────────┐      │  │
                    │  │                             │  data-cleaner    │      │  │
                    │  │                             │  Lambda (opt.)   │      │  │
                    │  │                             └────────┬─────────┘      │  │
                    │  │                                      │ SQS            │  │
                    │  │                                      ▼                │  │
                    │  │                             ┌──────────────────┐      │  │
                    │  │                             │   case-saver     │      │  │
                    │  │                             │    Lambda        │      │  │
                    │  │                             └────────┬─────────┘      │  │
                    │  │                                      │                │  │
                    │  └──────────────────────────────────────┘                │  │
                    │                                         │                 │  │
                    │         ┌───────────────────────────────┘                 │  │
                    │         │  (manual states, report distribution,           │  │
                    │         │   ack pending, closure)                         │  │
                    │         ▼                                                  │  │
                    │  ┌──────────────────────────────────────────────────────┐ │  │
                    │  │         RULE-TRANSITION FRAMEWORK                     │ │  │
                    │  │                                                       │ │  │
                    │  │  Any state lambda (on error in catch block)           │ │  │
                    │  │    │                                                  │ │  │
                    │  │    └─ SQS(rule-transition-queue) ← { caseId }        │ │  │
                    │  │                │                                      │ │  │
                    │  │                ▼                                      │ │  │
                    │  │      rule-transition Lambda                           │ │  │
                    │  │        1. fetch case state, sub-state, error details  │ │  │
                    │  │        2. fetch transition rules for this state        │ │  │
                    │  │        3. evaluate rules in configured priority order  │ │  │
                    │  │        4. first matching rule → target state           │ │  │
                    │  │        5. update case.current_state in DB              │ │  │
                    │  │        6. invoke target state's SQS queue              │ │  │
                    │  │           OR set to manual state (await user action)   │ │  │
                    │  └──────────────────────────────────────────────────────┘ │  │
                    │                                                             │  │
                    │  Supporting infrastructure:                                 │  │
                    │    PostgreSQL  ← case-schema, meta-schema, workflow config  │  │
                    │    S3          ← raw JSON, cleaned JSON, reports            │  │
                    │    UI          ← manual state editing, case routing buttons │  │
                    └────────────────────────────────────────────────────────────┘
```

---

## 7. Component Deep Dives

### 7.1 Meta-schema — the routing table

The meta-schema is a DB table that maps state names to SQS queue URLs. It is the single source of truth for "which queue handles which state."

```sql
-- meta_schema table (simplified)
CREATE TABLE meta_schema (
  state_name   VARCHAR(100) PRIMARY KEY,  -- 'IDENTIFICATION', 'FORM_PARSING', etc.
  queue_url    TEXT NOT NULL,             -- SQS queue URL for this state
  lambda_name  TEXT,                      -- informational reference
  is_active    BOOLEAN DEFAULT TRUE
);

Examples:
  state_name='CASE_REGISTRATION' → queue_url='https://sqs.../registration-queue'
  state_name='IDENTIFICATION'    → queue_url='https://sqs.../identification-queue'
  state_name='FORM_PARSING'      → queue_url='https://sqs.../form-processor-queue'
  state_name='DATA_CLEANING'     → queue_url='https://sqs.../data-cleaner-queue'
  state_name='CASE_SAVING'       → queue_url='https://sqs.../case-saver-queue'
  state_name='RULE_TRANSITION'   → queue_url='https://sqs.../rule-transition-queue'
  state_name='REPORT_DISTRIBUTION'→queue_url='https://sqs.../profile-assembler-queue'
```

### 7.2 Case-schema — workflow and transitions

The case-schema holds the workflow definition and transition sequence per form type.

```sql
-- workflow table: maps a form to a workflow definition
CREATE TABLE workflow (
  id           UUID PRIMARY KEY,
  form_id      UUID NOT NULL,           -- identified form type
  tenant_id    UUID,                    -- optional: tenant-specific workflow
  source_id    UUID,                    -- optional: source-specific workflow
  name         VARCHAR(100) NOT NULL,   -- 'E2B_R3_Standard', 'CIOMS_ClientA'
  is_active    BOOLEAN DEFAULT TRUE
);

-- transitions table: the ordered state sequence for a workflow
CREATE TABLE transitions (
  id              UUID PRIMARY KEY,
  workflow_id     UUID NOT NULL REFERENCES workflow(id),
  current_state   VARCHAR(100) NOT NULL,
  next_state      VARCHAR(100) NOT NULL,  -- name looked up in meta_schema
  sequence_order  INT NOT NULL,           -- for display and validation
  UNIQUE(workflow_id, current_state)
);

Example — E2B R3 workflow (skips data cleaning):
  workflow_id=W1, current_state='CASE_REGISTRATION' → next_state='IDENTIFICATION'
  workflow_id=W1, current_state='IDENTIFICATION'    → next_state='FORM_PARSING'
  workflow_id=W1, current_state='FORM_PARSING'      → next_state='CASE_SAVING'  ← skip cleaning
  workflow_id=W1, current_state='CASE_SAVING'       → next_state='REPORT_DISTRIBUTION'

Example — CIOMS PDF workflow:
  workflow_id=W2, current_state='CASE_REGISTRATION' → next_state='IDENTIFICATION'
  workflow_id=W2, current_state='IDENTIFICATION'    → next_state='FORM_PARSING'
  workflow_id=W2, current_state='FORM_PARSING'      → next_state='DATA_CLEANING'
  workflow_id=W2, current_state='DATA_CLEANING'     → next_state='CASE_SAVING'
  workflow_id=W2, current_state='CASE_SAVING'       → next_state='MANUAL_REVIEW'
  workflow_id=W2, current_state='MANUAL_REVIEW'     → next_state='REPORT_DISTRIBUTION'
```

### 7.3 The generic next-state helper function

Every state lambda uses the same shared helper to determine and invoke the next state. This is the central mechanism of the decoupled architecture.

```typescript
// Shared helper — imported by every state lambda
async function invokeNextState(caseId: string): Promise<void> {
  // 1. Fetch the case's current form and workflow from DB
  const caseRecord = await db.query(
    `SELECT form_id, workflow_id, current_state FROM cases WHERE id = $1`,
    [caseId]
  );

  // 2. Look up the next state from the transitions table
  const transition = await db.query(
    `SELECT next_state FROM transitions
     WHERE workflow_id = $1 AND current_state = $2`,
    [caseRecord.workflow_id, caseRecord.current_state]
  );
  const nextState = transition.next_state;

  // 3. Look up the queue URL from meta_schema
  const meta = await db.query(
    `SELECT queue_url FROM meta_schema WHERE state_name = $1`,
    [nextState]
  );

  // 4. Update case's current state in DB
  await db.query(
    `UPDATE cases SET current_state = $1, sub_state = 'IN_PROGRESS',
     updated_at = NOW() WHERE id = $2`,
    [nextState, caseId]
  );

  // 5. Send { caseId } to the next state's queue
  await sqs.sendMessage({
    QueueUrl: meta.queue_url,
    MessageBody: JSON.stringify({ caseId }),
  });
}
```

This function is the entire routing engine. No lambda needs any knowledge of what comes before or after it. Changing a workflow is purely a DB update to the transitions table.

### 7.4 State lambda pattern — every automated state

All state lambdas follow the same structure:

```typescript
// Generic state lambda pattern (e.g., form-processor)
export const handler = async (event: SQSEvent): Promise<void> => {
  const { caseId } = JSON.parse(event.Records[0].body);

  try {
    // 1. Fetch all needed context from DB using caseId
    const caseRecord = await db.getCaseWithContext(caseId);

    // 2. Do the actual work (state-specific logic)
    const rawJson = await parseForm(caseRecord.attachmentS3Key, caseRecord.formType);

    // 3. Persist results to S3 / DB
    const s3Key = await s3.upload(`cases/${caseId}/raw.json`, rawJson);
    await db.updateCase(caseId, {
      rawJsonS3Key: s3Key,
      sub_state: 'COMPLETED'
    });

    // 4. Invoke next state via helper
    await invokeNextState(caseId);

  } catch (error) {
    // 5. On error: update DB with error details
    await db.updateCase(caseId, {
      sub_state: 'ERROR',
      error_message: error.message,
      error_type: classifyError(error),
    });

    // 6. Invoke rule-transition lambda to handle routing
    await invokeRuleTransition(caseId);
    // invokeRuleTransition sends { caseId } to rule-transition-queue
  }
};
```

**The key discipline:** the catch block never re-throws to SQS. It handles the error explicitly by invoking rule-transition. This prevents the case from being retried via SQS visibility timeout for business logic errors (wrong data, unrecognised form) — only infra errors should cause SQS retries.

### 7.5 SQS retry for infra errors

```
Infra failure scenario (Lambda crashes mid-execution, memory error, timeout):
  The message remains invisible in SQS (visibility timeout: 30-45 min)
  Lambda invocation ends without completion
  No explicit error update to DB occurs
  Visibility timeout expires → message becomes visible again
  Lambda is reinvoked with the same { caseId }
  Lambda re-fetches all context from DB and retries from scratch

This handles:
  Lambda cold start failures
  Memory/CPU spikes
  Transient downstream service errors
  Lambda timeout (15-min limit)

Message Retention Period: 1-2 days
  A message retried every 30-45 minutes for 1-2 days
  = 32-96 retry attempts before the message expires

When message expires without success:
  Message is silently dropped (no DLQ currently)
  Case remains in current state indefinitely → GAP ⚠
```

### 7.6 rule-transition Lambda

**Trigger:** SQS message from any errored state lambda: `{ caseId }`

```typescript
export const handler = async (event: SQSEvent): Promise<void> => {
  const { caseId } = JSON.parse(event.Records[0].body);

  // 1. Fetch full case context from DB
  //    - current_state, sub_state, error_type, error_message
  //    - form details, source details, case data points
  const caseContext = await db.getFullCaseContext(caseId);

  // 2. Fetch all transition rules for this workflow state
  const rules = await db.query(
    `SELECT * FROM transition_rules
     WHERE workflow_state_id = $1
       AND is_active = true
     ORDER BY priority ASC`,          // first satisfied rule wins
    [caseContext.workflow_state_id]
  );

  // 3. Evaluate rules in priority order — first match wins
  let targetState: string | null = null;
  for (const rule of rules) {
    if (evaluateRule(rule, caseContext)) {
      targetState = rule.target_state;
      break;
    }
  }

  if (!targetState) {
    // No rule matched — mark case as ERROR, await manual intervention
    await db.updateCase(caseId, { sub_state: 'UNRESOLVED_ERROR' });
    return;
  }

  // 4. Update case state in DB
  await db.updateCase(caseId, {
    current_state: targetState,
    sub_state: isManualState(targetState) ? 'AWAITING_USER' : 'IN_PROGRESS',
  });

  // 5a. If target is an automated state: invoke its queue
  if (!isManualState(targetState)) {
    const meta = await db.getMetaSchema(targetState);
    await sqs.sendMessage({
      QueueUrl: meta.queue_url,
      MessageBody: JSON.stringify({ caseId }),
    });
  }
  // 5b. If target is a manual state: no queue — UI shows it to the user
};
```

### 7.7 Rule evaluation engine

```typescript
function evaluateRule(rule: TransitionRule, ctx: CaseContext): boolean {
  // Rules are AND-conditions across multiple criteria
  // All specified criteria must match for the rule to fire

  if (rule.required_state && ctx.current_state !== rule.required_state)
    return false;

  if (rule.required_sub_state && ctx.sub_state !== rule.required_sub_state)
    return false;

  if (rule.required_error_type && ctx.error_type !== rule.required_error_type)
    return false;

  if (rule.required_form_id && ctx.form_id !== rule.required_form_id)
    return false;

  if (rule.required_source_id && ctx.source_id !== rule.required_source_id)
    return false;

  // Additional data point conditions stored as JSONB and evaluated dynamically
  if (rule.data_point_conditions) {
    for (const condition of rule.data_point_conditions) {
      if (!evaluateDataPointCondition(condition, ctx.caseData))
        return false;
    }
  }

  return true; // all conditions matched
}
```

### 7.8 Manual state and user-driven routing

When a case lands in a manual state (Manual Review, Reprocess Case):

```
UI shows the case with:
  - Current state and sub-state
  - Error reason (if errored)
  - Case data editor (editable fields)
  - "Route to [X state]" button (configured target states for this manual state)

User flow:
  1. User reviews the case (and optionally edits data points)
  2. User clicks "Route to Report Distribution"
  3. UI calls: POST /api/v1/cases/{caseId}/route { targetState: 'REPORT_DISTRIBUTION' }
  4. API validates the target state is allowed from this manual state
  5. API updates case.current_state in DB
  6. API sends { caseId } to Report Distribution queue
  7. Case resumes automated processing
```

---

## 8. Data Model

```sql
-- Core case record
CREATE TABLE cases (
  id                  UUID          PRIMARY KEY,
  case_number         VARCHAR(50)   UNIQUE NOT NULL,
  tenant_id           UUID          NOT NULL,
  source_id           UUID          NOT NULL,
  form_id             UUID,                        -- populated after identification
  workflow_id         UUID,                        -- populated after identification
  current_state       VARCHAR(100)  NOT NULL,
  sub_state           VARCHAR(50)   NOT NULL,
  -- 'IN_PROGRESS' | 'COMPLETED' | 'ERROR' | 'AWAITING_USER' | 'UNRESOLVED_ERROR'
  error_type          VARCHAR(100),
  error_message       TEXT,
  attachment_s3_key   TEXT,                        -- original source attachment
  raw_json_s3_key     TEXT,                        -- after form parsing
  cleaned_json_s3_key TEXT,                        -- after data cleaning
  due_date            TIMESTAMPTZ,
  created_at          TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
  updated_at          TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

-- Workflow definition (per form/tenant/source combination)
CREATE TABLE workflow (
  id          UUID          PRIMARY KEY,
  form_id     UUID,
  tenant_id   UUID,
  source_id   UUID,
  name        VARCHAR(100)  NOT NULL,
  is_active   BOOLEAN       DEFAULT TRUE
);

-- Ordered state sequence for a workflow
CREATE TABLE transitions (
  id              UUID          PRIMARY KEY,
  workflow_id     UUID          NOT NULL REFERENCES workflow(id),
  current_state   VARCHAR(100)  NOT NULL,
  next_state      VARCHAR(100)  NOT NULL,
  sequence_order  INT           NOT NULL,
  UNIQUE(workflow_id, current_state)
);

-- State-to-queue routing table
CREATE TABLE meta_schema (
  state_name    VARCHAR(100)  PRIMARY KEY,
  queue_url     TEXT          NOT NULL,
  lambda_name   TEXT,
  is_active     BOOLEAN       DEFAULT TRUE
);

-- Transition rules (evaluated by rule-transition lambda on error)
CREATE TABLE transition_rules (
  id                    UUID          PRIMARY KEY,
  workflow_state_id     UUID          NOT NULL,    -- which state these rules apply to
  rule_name             VARCHAR(100)  NOT NULL,
  priority              INT           NOT NULL,    -- lower = evaluated first
  required_state        VARCHAR(100),
  required_sub_state    VARCHAR(50),
  required_error_type   VARCHAR(100),
  required_form_id      UUID,
  required_source_id    UUID,
  data_point_conditions JSONB,                     -- flexible additional conditions
  target_state          VARCHAR(100)  NOT NULL,    -- where to route the case
  is_active             BOOLEAN       DEFAULT TRUE,
  created_at            TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

-- State processing history (one row per state traversal per case)
CREATE TABLE case_state_history (
  id            UUID          PRIMARY KEY,
  case_id       UUID          NOT NULL REFERENCES cases(id),
  state         VARCHAR(100)  NOT NULL,
  sub_state     VARCHAR(50)   NOT NULL,
  entered_at    TIMESTAMPTZ   NOT NULL DEFAULT NOW(),
  exited_at     TIMESTAMPTZ,
  error_type    VARCHAR(100),
  error_message TEXT,
  triggered_by  VARCHAR(50)   NOT NULL  -- 'SYSTEM' | 'USER' | 'RULE_TRANSITION'
);

-- Indexes
CREATE INDEX idx_cases_state     ON cases(current_state, sub_state);
CREATE INDEX idx_cases_tenant    ON cases(tenant_id, current_state);
CREATE INDEX idx_history_case    ON case_state_history(case_id);
CREATE INDEX idx_rules_state     ON transition_rules(workflow_state_id, priority);
```

---

## 9. Key Design Decisions and Rationale

### Decision 1 — SQS as the coupling layer (not direct Lambda invocation)

**Problem:** Lambda-to-Lambda invocation creates a dependency chain where any failure propagates immediately and there is no retry.

**Options considered:**
```
Option A: Direct Lambda invocation (original design)
  Lambda A synchronously calls Lambda B
  ✗ Tight coupling — A must know B's ARN
  ✗ B fails → A fails → case stuck → re-ingestion required
  ✗ Sequence changes require code changes in A

Option B: EventBridge (event bus)
  Lambda publishes event, next lambda subscribes
  ✓ Decoupled
  ✗ Fan-out model doesn't fit sequential pipeline well
  ✗ More complex rule configuration

Option C: SQS per state (chosen)
  Each state has a dedicated queue
  Lambda writes to queue, never invokes another lambda
  ✓ Fully decoupled — lambdas are independent
  ✓ SQS visibility timeout = automatic retry on infra failure
  ✓ Queue depth is a natural backpressure mechanism
  ✓ Scales horizontally — more Lambda concurrency = more throughput
```

**Why SQS specifically:** visibility timeout provides free retry semantics for infra failures without any code. Message retention period gives a generous retry window (1-2 days). Queue depth gives natural observability into processing backlog.

---

### Decision 2 — caseId as the only message payload

**Problem:** Early designs passed routing context (next state, form type, S3 keys) in the SQS message body. This created a coupling between the message producer and consumer — the producer needed to know what data the consumer needed.

**Options:**
```
Option A: Fat payload (original design)
  { caseId, nextState, formType, s3Key, ... }
  ✗ Producer must know consumer's data requirements
  ✗ Payload evolves with consumer — tight coupling reintroduced
  ✗ Message payload becomes a hidden API contract

Option B: caseId only (chosen)
  { caseId }
  ✓ Zero coupling between producer and consumer
  ✓ Consumer fetches everything it needs from DB
  ✓ DB is the single source of truth for case state
  ✓ Adding a new data requirement = add DB column, not change message format
  ✓ All consumers use the same message schema forever
```

**The normalised payload principle:** the message is not data — it is a notification. The data lives in the DB.

---

### Decision 3 — DB-driven routing via meta-schema + transitions table

**Problem:** The sequence of states varies per form type and client. Hard-coding sequences in Lambda code means every workflow change is a deployment.

**Solution:** Two DB tables encode routing:
```
transitions table:  for workflow W, after state X, go to state Y
meta_schema table:  state Y maps to queue URL Z

invokeNextState() reads both tables at runtime → zero hardcoding
```

**Why this separation:** transitions encode the business logic (workflow sequence). meta_schema encodes the infrastructure (queue locations). They evolve independently — a new Lambda for a new state only requires a meta_schema row. A new workflow only requires transition rows.

---

### Decision 4 — rule-transition as a separate lambda, invoked from the catch block

**Problem:** When a state lambda errors, something must decide where the case goes next. Options:

```
Option A: Each state lambda contains its own error routing logic
  ✗ Routing rules duplicated across many lambdas
  ✗ Adding a new rule = touch multiple lambdas

Option B: Centralised rule-transition lambda (chosen)
  Error catch block → SQS → rule-transition lambda
  ✓ Single place for all error routing logic
  ✓ Rules are DB-configured — no code changes for new rules
  ✓ Rule changes are BA/Admin operations via UI
  ✓ Any new state lambda gets rule-transition for free
```

**Why invoked from the catch block (not a DLQ consumer):** business logic errors (wrong data, unrecognised form) should not be retried by SQS — they need to be handled immediately. Only infra errors should exhaust the SQS retry window. The catch block distinguishes these two categories and routes accordingly.

---

### Decision 5 — case state updated in DB before invoking next queue

**Principle:** the DB is always the authoritative state of a case. The SQS message is derived from the DB state, not the reverse.

```
Order of operations (every state lambda):
  1. Complete the state's work
  2. Update DB: cases.current_state = next_state
  3. Send SQS message to next queue

If step 3 fails after step 2:
  The next lambda is never invoked
  Case is in DB as 'IDENTIFICATION' state but no SQS message exists
  Detected by: monitoring job watching cases in IN_PROGRESS for > N hours
  Recovery: re-send the SQS message manually or via monitoring alert

If step 2 fails:
  DB not updated, next state never triggered
  Same detection and recovery path

Why not reverse the order (SQS first, then DB)?
  If SQS succeeds but DB update fails:
  Next lambda runs before DB is updated
  Next lambda fetches stale state from DB
  Data corruption possible
```

---

## 10. Rule-Based Transition Framework

### What it is

A configurable, priority-ordered rule evaluation system that determines where a case should be routed when it errors at any state. Rules are configured by BAs/Admins via UI and stored in the `transition_rules` table.

### Rule structure

```
A transition rule has:
  - Applied at: which workflow state this rule applies to
  - Priority: lower number evaluated first (first match wins)
  - Conditions (AND-combined):
      current_state       = exact state name
      sub_state           = sub-state (usually 'ERROR')
      error_type          = classified error type ('NO_FORM_FOUND', 'SCHEMA_INVALID', etc.)
      form_id             = optional form-specific condition
      source_id           = optional source-specific condition
      data_point_conditions = optional JSONB conditions on case data fields
  - Target state: where to route the case if all conditions match

Example rules for Form Identification state:
  Rule 1 (priority=1): state=IDENTIFICATION, sub_state=ERROR,
          error_type=NO_FORM_FOUND, source_id=EV_WEB
          → target=MANUAL_REVIEW
          (EV-Web cases with no form found → manual review)

  Rule 2 (priority=2): state=IDENTIFICATION, sub_state=ERROR,
          error_type=NO_FORM_FOUND
          → target=MANUAL_REVIEW
          (Any other case with no form found → manual review)

  Rule 3 (priority=3): state=IDENTIFICATION, sub_state=ERROR
          → target=ERROR_PARKING
          (Any other identification error → error parking)
```

### Rule evaluation flow

```
Errored state lambda
  │
  └─ catch block → update DB with error details → invoke rule-transition queue
                                                           │
                                                           ▼
                                              rule-transition Lambda
                                                           │
                                              fetch case context from DB
                                                           │
                                              fetch rules for this state
                                              (ordered by priority ASC)
                                                           │
                                              ┌────────────▼──────────────┐
                                              │  Rule 1 conditions met?   │
                                              │  YES → route to target    │
                                              │  NO  ↓                    │
                                              │  Rule 2 conditions met?   │
                                              │  YES → route to target    │
                                              │  NO  ↓                    │
                                              │  Rule N conditions met?   │
                                              │  YES → route to target    │
                                              │  NO  → UNRESOLVED_ERROR   │
                                              └───────────────────────────┘
```

### Routing patterns enabled by the framework

```
Pattern 1 — Linear re-routing (jump forward)
  Identification ERROR → skip remaining pre-extraction states
  → Manual Review (user enters data manually)
  → Case Saving → Report Distribution

Pattern 2 — Circular re-routing (retry after fix)
  Report Distribution ERROR
  → Reprocess Case (parking — user fixes data or waits for Argus)
  → User clicks "Route to Report Distribution"
  → Report Distribution (second attempt)
  Possible to loop multiple times until resolved

Pattern 3 — Source-specific routing
  Identification ERROR for EV-Web source only
  → EV-Web specific Manual Review state
  vs.
  Identification ERROR for Email source
  → Generic Manual Review state

Pattern 4 — Error-type specific routing
  Form Parsing ERROR, error_type=CORRUPTED_FILE → request re-upload state
  Form Parsing ERROR, error_type=UNSUPPORTED_FORMAT → manual review
  Form Parsing ERROR, error_type=INFRA_TIMEOUT → retry same state
```

### Circular path safeguard (current gap)

```
Current: no safeguard against infinite circular routing
  User can route: Report Distribution → Error → Reprocess → Report Distribution
  repeatedly without the error being resolved
  System will keep attempting and failing

Risk level: LOW in practice
  Circular paths are user-triggered (not automated)
  User must manually click "Route to X" each time
  In practice, users stop routing after 2-3 failed attempts and escalate

Future improvement:
  Track routing_attempt_count per state per case
  If count > configured_threshold (e.g., 3) → block routing, alert supervisor
```

---

## 11. Failure Handling and Reprocessing Scenarios

### Scenario matrix

```
Failure type         At state              SQS retry?   Rule transition?   Outcome
─────────────────────────────────────────────────────────────────────────────────────
Lambda crash         Any automated state   YES           NO (not reached)   Auto-retry
Lambda timeout       Any automated state   YES           NO (not reached)   Auto-retry
Memory error         Any automated state   YES           NO (not reached)   Auto-retry
No form found        Identification        NO             YES                Manual Review
Corrupted PDF        Form Parsing          NO             YES                Manual Review
Argus timeout        Report Distribution   NO             YES                Reprocess Case
Wrong XML field      Report Distribution   NO             YES                Reprocess Case
Argus processing err Ack Pending           NO             YES                Reprocess Case
Message TTL expires  Any state             NO (dropped)   NO                 Case stuck ⚠
```

### Key reprocessing flows

#### Flow 1 — Infra failure and automatic recovery

```
form-processor Lambda (processing CIOMS PDF)
  │
  ✗ Lambda times out (cold start + large PDF)
  │
  SQS: message remains invisible (30-45 min visibility timeout)
  │
  Visibility timeout expires
  │
  Lambda reinvoked with same { caseId }
  │
  Lambda re-fetches all context from DB:
    current_state='FORM_PARSING', attachment_s3_key=...
  │
  Lambda retries form parsing from scratch
  │
  Success → invokes case-saver queue
```

#### Flow 2 — Business error with rule-based rerouting

```
identification Lambda (E2B XML from FTP)
  │
  ✗ No matching form identifier found in DB
  │
  catch block:
    1. db.updateCase: sub_state='ERROR', error_type='NO_FORM_FOUND'
    2. sqs.sendMessage(rule-transition-queue, { caseId })
  │
  rule-transition Lambda:
    - fetches case: state=IDENTIFICATION, error=NO_FORM_FOUND, source=FTP
    - evaluates rules → Rule 2 matches (generic NO_FORM_FOUND rule)
    - target_state = MANUAL_REVIEW
    - db.updateCase: current_state=MANUAL_REVIEW, sub_state=AWAITING_USER
    - no SQS message sent (MANUAL_REVIEW is a manual state)
  │
  UI shows case to user in Manual Review queue
  User selects the correct form type and saves
  User clicks "Route to Form Parsing"
  │
  API: db.updateCase: current_state=FORM_PARSING
  API: sqs.sendMessage(form-processor-queue, { caseId })
  │
  Case resumes automated processing
```

#### Flow 3 — Report Distribution error with parking and re-routing

```
safety-system-sender Lambda (Argus submission)
  │
  ✗ Argus endpoint timeout
  │
  catch block → rule-transition Lambda invoked
  │
  rule-transition Lambda:
    - error_type = ARGUS_TIMEOUT at REPORT_DISTRIBUTION
    - matches rule → target = REPROCESS_CASE
    - db.updateCase: current_state=REPROCESS_CASE, sub_state=AWAITING_USER
  │
  Argus team notified (via existing operational alert)
  Argus recovers after 2 hours
  │
  User opens Reprocess Case state in UI
  Case shows: "Errored at Report Distribution — Argus timeout"
  User clicks "Route to Report Distribution"
  │
  Case re-runs full Report Distribution pipeline
  (XML already generated in S3 — generation step is idempotent)
  Reports transmitted to Argus successfully
```

---

## 12. Trade-offs and Known Limitations

### Trade-off 1 — No DLQ on SQS queues

**Current:** Messages that exhaust retries during the message retention period (1-2 days) are silently dropped. The case remains stuck in its current state indefinitely.

**Impact:** If a Lambda has a persistent bug (introduced via deployment), all cases hitting that state over 1-2 days will eventually be dropped silently.

**Better approach:** DLQ per queue. Messages landing in DLQ trigger an alert and automatically invoke rule-transition to mark the case as `UNRESOLVED_ERROR`. Zero silent drops.

---

### Trade-off 2 — Lambda re-executes from scratch on SQS retry

**Current:** When a Lambda is retried by SQS (after visibility timeout), it re-fetches all context from DB and redoes all computation from the start.

**Impact:** If form parsing a large PDF takes 90 seconds and fails at 85 seconds due to a transient error, the next retry redoes all 90 seconds. Inefficient for compute-heavy states.

**Better approach:** Checkpoint progress to S3 (e.g., partially parsed JSON). On retry, resume from checkpoint. Appropriate for expensive states like form parsing of large Excel files.

---

### Trade-off 3 — No safeguard against infinite circular routing

**Current:** A user can route a case circularly indefinitely. If Argus is broken, a user could send a case to Report Distribution 50 times.

**Impact:** Wasted Lambda invocations, cluttered case history, operator confusion.

**Better approach:** `routing_attempt_count` per state tracked in `case_state_history`. If count exceeds configured threshold, block further routing and escalate to supervisor.

---

### Trade-off 4 — Rule evaluation happens entirely in Lambda memory

**Current:** All transition rules for a state are fetched from DB and evaluated in the rule-transition Lambda on each invocation.

**Impact:** For states with many rules (e.g., 20+ rules), DB query and evaluation time adds latency. Not a concern at current scale.

**Better approach:** At scale, cache the rule set in ElastiCache/Redis with a short TTL (5-10 minutes). Rule changes propagate within the TTL window without Lambda redeployment.

---

### Trade-off 5 — Workflow configuration UI requires discipline

**Current:** BAs can create any workflow with any state sequence and any transition rules. No validation prevents a BA from creating a workflow that routes a case into a non-existent state or creates an unresolvable loop.

**Impact:** Misconfiguration can cause cases to get stuck or be incorrectly routed.

**Better approach:** Workflow validation on save — check that every state in the transitions table has a corresponding meta_schema entry. Check for terminal states (at least one path ends at Case Closed). Warn (not block) on detected circular paths.

---

## 13. Future Improvements

### Priority 1 — Add DLQ to all queues

```
Every SQS queue → DLQ
DLQ consumer Lambda:
  - receives the dropped message { caseId }
  - updates case: sub_state = 'UNRESOLVED_ERROR'
  - triggers alert (PagerDuty / SNS email)
  - logs to case_state_history
Eliminates all silent case drops
```

### Priority 2 — Routing attempt count and circuit breaker

```
case_state_history tracks attempt count per state per case
rule-transition Lambda checks: if attempts > threshold → block and alert
Prevents users from accidentally looping cases indefinitely
```

### Priority 3 — Checkpoint for expensive state lambdas

```
Form Parsing and Data Cleaning lambdas:
  Write intermediate results to S3 after each major step
  On retry: check S3 for checkpoint, resume from there
  Saves significant compute time for large attachments
```

### Priority 4 — Workflow validation in UI

```
On workflow save:
  Validate all next_state values exist in meta_schema
  Validate at least one path leads to CASE_CLOSED
  Detect and warn on circular paths
  Preview the workflow as a visual state diagram
```

### Priority 5 — Priority queue for urgent cases

```
Current: all cases share the same SQS queues (FIFO by arrival)
Enhancement:
  Add a second high-priority queue per state
  Cases with due_date < 24 hours → high-priority queue
  Lambda trigger processes high-priority queue first
  Ensures urgent regulatory submissions are not delayed by routine cases
```

### Priority 6 — Full observability

```
Current: basic dashboard showing case counts by state/sub-state
Enhancement:
  Average time per state (identify bottlenecks)
  Error rate by error type, form, source
  Cases approaching due date with current state
  SQS queue depth per state (backlog visibility)
  DLQ depth alert (Priority 1 prerequisite)
```

---

## 14. Interview Script

### Opening — frame the problem and the evolution

> "I want to walk you through a system I designed and led in my current role at a Pharmacovigilance company. The system handles the end-to-end ingestion and processing of adverse drug event cases from multiple external sources — email, FTP, web portals, and cloud storage. Cases can arrive as PDFs, XMLs, Excel files, and Word documents. Each one needs to go through a multi-state processing pipeline: case registration, form identification, form parsing, data cleaning, case saving, and ultimately report distribution to regulatory authorities."

> "The original system had all these processing states hardwired — each Lambda directly invoked the next one. That created three problems: tight coupling meant any workflow change needed a code deployment; an infra failure at any state killed the case permanently requiring full re-ingestion; and there was no mechanism to handle business errors gracefully — a case with an unrecognised form had nowhere to go except a full restart."

### The SQS decoupling solution

> "The first major change was replacing Lambda-to-Lambda invocation with SQS queues. Every state now has a dedicated SQS queue. A Lambda finishes its work, looks up the next state from a DB transitions table, looks up that state's queue from a meta-schema table, and sends the message. The Lambda never knows what comes next — it just reads a routing table. The message payload is just the case ID. Everything else is fetched from DB. This gives us two things for free: complete decoupling — you can change any workflow by updating a DB row — and automatic retry, because SQS visibility timeout means an infra failure just results in the message becoming visible again and the Lambda retrying."

### The rule-transition framework

> "The second major enhancement was the rule-based transition framework. When a state errors — say Form Identification can't find a matching form — the Lambda's catch block invokes a centralised rule-transition Lambda with just the case ID. That Lambda fetches the case context from DB, evaluates a priority-ordered set of transition rules configured by business analysts, and routes the case to the appropriate state. This might be Manual Review, a parking state, or even back to the same state to retry. The rules can condition on the error type, the form type, the source, even case data points. Adding a new routing rule is a DB operation from the admin UI — no code deployment."

### The reprocessing scenarios

> "This framework directly solves the main error scenarios we see in production. Argus connectivity issues at Report Distribution — instead of losing the case or requiring re-ingestion, the rule routes it to a Reprocess Case parking state. Once Argus recovers, the user clicks 'Route to Report Distribution' from the UI, the case resumes exactly where it left off, and the already-generated XML is re-transmitted. Wrong data point in an XML report — user edits it at the Reprocess state, re-routes, the xml-generator picks up the corrected case data and regenerates."

### Trade-offs to mention proactively

> "Two trade-offs I'd flag. First: we have no DLQ on our SQS queues. Messages that exhaust the retention period are silently dropped. That's our top infrastructure gap — we're adding DLQs with automatic case error-marking and alerting. Second: circular routing paths are possible and we have no guard against a user looping a case indefinitely. In practice it doesn't happen because circular routes are always user-triggered and operators stop after a few attempts. But a routing attempt counter with a threshold is on our roadmap."

---

## 15. Follow-up Probes and Answers

**"How does the system handle a case where the same Lambda is retried 50 times over 2 days and still fails?"**  
Currently: the message expires after the retention period and is silently dropped. The case stays in its current state indefinitely and operations discovers it via the dashboard. The fix — DLQ per queue with an automatic error-marking consumer — is our top priority improvement. With DLQ in place: message drops into DLQ, consumer Lambda marks the case as `UNRESOLVED_ERROR` and triggers a PagerDuty alert, operations team investigates.

**"What prevents two Lambda invocations from processing the same case simultaneously?"**  
SQS visibility timeout. Once a message is received by a Lambda, it becomes invisible to all other consumers for the duration of the visibility timeout (30-45 minutes). Only after the Lambda completes (message deleted) or the timeout expires (Lambda failed silently) does the message become available again. There is no scenario where two Lambdas simultaneously process the same case from the same queue. However, if a Lambda crashes after updating the DB but before deleting the SQS message, the next retry will re-fetch the DB state and may detect the partially completed work — each Lambda is therefore designed to be idempotent.

**"How does adding a completely new state to the pipeline work?"**  
Three steps. First, implement a new Lambda with the standard pattern: receive `{ caseId }`, fetch context from DB, do the work, invoke next state via the helper, or invoke rule-transition on error. Second, add a row to the meta_schema table mapping the new state name to the new Lambda's SQS queue URL. Third, update the transitions table for any workflows that should include this new state. Zero changes to any existing Lambda code. The BA can add the state to workflows via the admin UI the same day.

**"How does the system handle E2B XML differently from a CIOMS PDF?"**  
At the workflow level. After form identification, the case is associated with a form type and through that to a specific workflow. The E2B workflow in the transitions table has no DATA_CLEANING row — after FORM_PARSING, the next_state is CASE_SAVING. The CIOMS workflow includes DATA_CLEANING between parsing and saving. The `invokeNextState` helper reads the transitions table for whichever workflow the case is on. No conditional code in the Lambdas — the path difference is entirely DB-driven.

**"Could two different transition rules match simultaneously? Who wins?"**  
Rules are evaluated in priority order (ascending integer). The first rule whose conditions are all satisfied is applied — remaining rules are not evaluated. This is a deliberate design: the BA configures rules from most-specific (highest priority, most conditions) to least-specific (fallback rule, fewest conditions). Example: Rule 1 (priority=1) applies only to EV-Web source with NO_FORM_FOUND error. Rule 2 (priority=2) applies to any source with NO_FORM_FOUND. An EV-Web case matches both, but Rule 1 fires. An email case matches only Rule 2. This is analogous to CSS specificity or firewall rule ordering.

**"How does the system ensure a case's regulatory due date is not missed during reprocessing?"**  
The due date is stored on the case record and is surfaced in the operational dashboard. When a case is parked in a Reprocess state, the dashboard highlights it if the due date is within 24 hours. However, there is no automated priority escalation currently — that's a known gap. The Priority 5 improvement in our roadmap addresses this: cases with due dates within 24 hours would be routed to a high-priority SQS queue that the Lambda trigger processes preferentially.

**"How is the workflow configuration protected from bad BA input — e.g., a workflow that never reaches Case Closed?"**  
Honestly, there is limited validation today — a BA could configure a workflow that loops indefinitely or has no terminal state. This is mitigated in practice by the approval process: new workflow configurations are reviewed by a senior BA or engineer before being activated for a client. The technical safeguard — workflow graph validation on save — is a planned improvement. We'd validate that every workflow has at least one path that terminates at CASE_CLOSED, and warn (not block) on detected circular state sequences.

---

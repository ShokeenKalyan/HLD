# System Design — Report Distribution Module (Pharmacovigilance)

> **Domain:** Pharmacovigilance (drug safety reporting)  
> **Role:** Senior Backend Lead  
> **Type:** Real system — production design with architectural decisions, trade-offs, and improvement opportunities  
> **Core problem:** Generating regulated adverse event reports (E2B R2/R3 XML, PDF) from structured case data and distributing them to safety systems (Argus) and email destinations reliably, in parallel, with full audit trail.

---

## Table of Contents

1. [Domain Context](#1-domain-context)
2. [Requirements](#2-requirements)
3. [Scale and Capacity](#3-scale-and-capacity)
4. [System Context — Where This Module Lives](#4-system-context--where-this-module-lives)
5. [High-Level Architecture](#5-high-level-architecture)
6. [Component Deep Dives](#6-component-deep-dives)
7. [Data Model](#7-data-model)
8. [Key Design Decisions and Rationale](#8-key-design-decisions-and-rationale)
9. [Failure Handling and Resilience](#9-failure-handling-and-resilience)
10. [Audit and Compliance](#10-audit-and-compliance)
11. [Trade-offs and Known Limitations](#11-trade-offs-and-known-limitations)
12. [Future Improvements](#12-future-improvements)
13. [Interview Script](#13-interview-script)
14. [Follow-up Probes and Answers](#14-follow-up-probes-and-answers)

---

## 1. Domain Context

### What is Pharmacovigilance?

Pharmacovigilance (PV) is the science of monitoring, detecting, and preventing adverse effects of pharmaceutical products. Regulatory bodies — the EMA (Europe), FDA (USA), and others — require pharmaceutical companies to report adverse drug reactions via standardised electronic formats.

### What is an adverse event report?

An **Individual Case Safety Report (ICSR)** is a structured record of an adverse event associated with a drug. When a patient or healthcare professional reports a side effect, it becomes a case in the safety database.

Regulations mandate:
- **Expedited reports:** serious, unexpected reactions must be submitted within **7 or 15 calendar days** of awareness
- **Periodic reports:** routine submissions on a defined schedule
- **Format:** E2B R2 (legacy XML, HL7 v2) or E2B R3 (current XML, HL7 FHIR-compatible), per ICH and EMA guidelines
- **Destination:** national regulatory authority safety systems (e.g., Argus Safety, VigiBase, EudraVigilance)

### Why this module is critical

A missed or late submission is a **regulatory violation** — potentially resulting in fines, license suspension, or market withdrawal. The report distribution module is therefore a compliance-critical, audit-required system where correctness and traceability matter more than raw throughput.

---

## 2. Requirements

### Functional Requirements

- Generate adverse event reports in E2B R3 XML, E2B R2 XML, and PDF formats from a structured case JSON
- Transmit generated reports to configured destinations:
  - **Safety systems** (e.g., Argus) via direct integration
  - **Email mailboxes** (internal departments, regulatory teams) via SES
- Support configurable profile-destination rules per workflow state — which format goes to which destination
- Support parallel generation of multiple formats and parallel transmission to multiple destinations
- Record generation and transmission status per case
- Invoke the audit service after every significant event (generation, upload, transmission)
- On full success: advance the case to its next configured workflow state
- On any failure: mark the case as Error and halt the workflow cycle for that case
- Support future addition of new formats (DOCX, XLSX) and new destination types (web portals, additional safety systems) without re-architecting

### Non-functional Requirements

- **Reliability:** no report must be silently lost — every failure must be detectable and recoverable
- **Traceability:** every generation and transmission event must be logged with timestamps, status, and reference IDs (e.g., Argus acknowledgement number)
- **Compliance:** audit trail must be immutable and complete — sufficient for regulatory inspection
- **Latency SLA:** serious cases must complete full distribution within hours of ingestion (informal, but operationally enforced via due-date tracking)
- **Configurability:** adding a new profile or destination should require only DB configuration — no code deployment
- **Scalability:** must scale from current thousands of cases/day to tens of thousands without architectural change

### Regulatory Constraints (shape the design)

- XML must strictly conform to **ICH E2B R3** or **E2B R2** schema — malformed XML is a submission failure
- Transmission to safety systems must capture and store the **acknowledgement reference number** (e.g., Argus case number) in the audit log
- Reports are retained indefinitely in S3 — never deleted after transmission failure
- Due-date tracking is per case, calculated from the source/form against configured rules (e.g., "15 days from receipt for serious spontaneous reports")

---

## 3. Scale and Capacity

```
Current state:
  Active clients:     3-4
  Cases/day:          ~thousands (est. 1,000-5,000/day)
  Peak:               ~5 cases/min during high-activity windows

Near-term (1-2 years):
  Cases/day:          tens of thousands (~10,000-50,000)
  Peak:               ~50 cases/min

Far future:
  Cases/day:          hundreds of thousands

Per-case resource consumption:
  Reports generated:  typically 1 XML + 1 PDF = 2 Lambda invocations
  Destinations:       typically 1 Argus + 3-4 email = 4-5 Lambda invocations
  Total lambdas/case: ~6-7 Lambda invocations per case
  S3 objects:         2 per case (XML + PDF), retained permanently
  DB writes:          ~8-10 rows per case (generation status, transmission
                      status per destination, audit entries)

At 50,000 cases/day:
  Lambda invocations: ~350,000/day = ~4/sec average
  S3 objects/day:     ~100,000
  DB writes/day:      ~500,000
  → Well within Lambda and PostgreSQL capacity at current architecture
```

---

## 4. System Context — Where This Module Lives

```
UPSTREAM (ingestion pipeline — not part of this design)
─────────────────────────────────────────────────────────

  External Source (email, portal, partner system)
       │
       ▼
  Email Reader Lambda           ← fetches attachment from mailbox
       │
       ▼
  Form Parser Lambda            ← parses CIOMS, MedWatch, E2B forms
       │
       ▼
  Data Mapper Lambda            ← maps to canonical case JSON + stores in DB
       │
       ▼
  Workflow Engine               ← determines next state for the case
       │
       │  if next_state == 'ReportDistribution':
       │    SQS.send(profile-assembler-queue, { caseId, workflowStateId })
       ▼

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
     REPORT DISTRIBUTION MODULE (this design)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

       │
       ▼  (covered in detail in sections 5 and 6)
  [Full distribution pipeline]
       │
       ▼

DOWNSTREAM
──────────
  Case advances to next workflow state
  OR
  Case marked as Error → manual review queue
```

---

## 5. High-Level Architecture

### Overview

```
                    ┌─────────────────────────────────────────────────────────┐
                    │              REPORT DISTRIBUTION MODULE                  │
                    │                                                          │
  Workflow Engine   │                                                          │
  (previous state   │   ┌──────────────────┐                                  │
  complete)         │   │ profile-assembler │◀── SQS Queue (A)                │
       │            │   │    Lambda         │                                  │
       └─SQS(A)────▶│   │  - fetch profiles │                                 │
                    │   │  - build payload  │                                  │
                    │   └────────┬──────────┘                                  │
                    │            │  invoke                                      │
                    │            ▼                                              │
                    │   ┌─────────────────────────────────────────────────┐   │
                    │   │          ReportGenerator Step Function           │   │
                    │   │                                                  │   │
                    │   │  ┌──────────────┐    Map State (per format)     │   │
                    │   │  │ task-invoker │◀── SQS Queue (B)              │   │
                    │   │  │  Lambda      │                                │   │
                    │   │  └──────┬───────┘                               │   │
                    │   │         │  dispatch via SQS                     │   │
                    │   │    ┌────┴──────┬──────────┐                     │   │
                    │   │    ▼           ▼           ▼                    │   │
                    │   │  SQS(C)     SQS(D)      SQS(...)               │   │
                    │   │    │           │                                 │   │
                    │   │    ▼           ▼                                 │   │
                    │   │ xml-gen     pdf-gen    [future: docx-gen]        │   │
                    │   │ Lambda      Lambda     Lambda                    │   │
                    │   │    │           │                                 │   │
                    │   │    └─────┬─────┘                                │   │
                    │   │          │  upload to S3, update DB              │   │
                    │   │          │  send TaskToken back to Step Fn       │   │
                    │   │          ▼                                       │   │
                    │   │   Step Function waits for all tokens             │   │
                    │   │   (WaitForTaskToken per Map iteration)           │   │
                    │   └─────────────────────────────────────────────────┘   │
                    │                    │                                      │
                    │                    │  on completion                       │
                    │                    ▼                                      │
                    │   ┌──────────────────────┐                               │
                    │   │ destination-assembler │◀── invoked by Step Fn        │
                    │   │      Lambda           │                               │
                    │   │  - check gen status   │                               │
                    │   │  - fetch destinations │                               │
                    │   │  - invoke ReportSender│                               │
                    │   └──────────┬────────────┘                              │
                    │              │  invoke                                    │
                    │              ▼                                            │
                    │   ┌─────────────────────────────────────────────────┐   │
                    │   │           ReportSender Step Function             │   │
                    │   │                                                  │   │
                    │   │  ┌──────────────┐    Map State (per destination)│   │
                    │   │  │ task-invoker │◀── SQS Queue (B) [reused]     │   │
                    │   │  │  Lambda      │                                │   │
                    │   │  └──────┬───────┘                               │   │
                    │   │         │  dispatch via SQS                     │   │
                    │   │    ┌────┴──────┬──────────┐                     │   │
                    │   │    ▼           ▼           ▼                    │   │
                    │   │  SQS(E)     SQS(F)      SQS(...)               │   │
                    │   │    │           │                                 │   │
                    │   │    ▼           ▼                                 │   │
                    │   │ safety-sys  email-sender  [future: portal-sender]│   │
                    │   │  sender     Lambda        Lambda                 │   │
                    │   │  Lambda        │                                 │   │
                    │   │    │           │  update DB, invoke audit svc    │   │
                    │   │    └─────┬─────┘  send TaskToken back            │   │
                    │   │          │                                       │   │
                    │   │   Step Function waits for all tokens             │   │
                    │   └─────────────────────────────────────────────────┘   │
                    │                    │                                      │
                    │                    │  on completion                       │
                    │                    ▼                                      │
                    │   ┌──────────────────────┐                               │
                    │   │ distribution-resolver │◀── invoked by ReportSender   │
                    │   │      Lambda           │                               │
                    │   │  - evaluate all rules │                               │
                    │   │  - mark success/error │                               │
                    │   │  - advance workflow   │                               │
                    │   └──────────────────────┘                               │
                    └─────────────────────────────────────────────────────────┘

External:
  S3 Bucket    ← XML and PDF reports stored here permanently
  PostgreSQL   ← Case, Profile, Destination, Rule, Generation, Transmission tables
  Argus Safety ← Safety system receiving XML via direct integration
  AWS SES v2   ← Email delivery for PDF/XML to department mailboxes
  Audit Service← Separate microservice capturing all events immutably
```

---

## 6. Component Deep Dives

### 6.1 profile-assembler Lambda

**Trigger:** SQS message from the workflow engine containing `{ caseId, workflowStateId }`

**Responsibilities:**
```
1. Fetch the case JSON from the DB using caseId
2. Fetch the workflow state configuration for workflowStateId
3. Query the Rule table to find all rules attached to this state
   SELECT r.*, p.*, d.*
   FROM rules r
   JOIN profiles p ON r.profile_id = p.id
   JOIN destinations d ON r.destination_id = d.id
   WHERE r.workflow_state_id = ?

4. Derive unique formats to generate:
   [XML(E2B R3), PDF] ← from the profile side of the rules

5. Build the payload for ReportGenerator Step Function:
   {
     caseId, caseJson,
     formatsToGenerate: [
       { format: 'XML_E2BR3', profileId, queueUrl: xml-gen-queue },
       { format: 'PDF',       profileId, queueUrl: pdf-gen-queue }
     ]
   }

6. Invoke ReportGenerator Step Function with this payload
```

**Why a separate assembler lambda?**  
Separates the "what needs to be done" query from the "doing it" orchestration. The Step Function receives a clean, pre-built payload — it does not know about DB schemas or rule evaluation logic.

---

### 6.2 task-invoker Lambda (generic dispatcher)

**Trigger:** SQS message containing `{ taskToken, targetQueueUrl, payload }`

**Responsibilities:**
```
1. Receive the task token from the Step Function
   (Step Function passes this when using WaitForTaskToken integration)
2. Forward the payload + task token to the target queue URL
   SQS.sendMessage({
     QueueUrl: targetQueueUrl,
     MessageBody: JSON.stringify({ ...payload, taskToken })
   })
3. Return immediately — does not wait for the target lambda
```

**Why this indirection exists:**  
Step Functions cannot natively invoke arbitrary SQS queues in a Map state with WaitForTaskToken while passing dynamic queue URLs as parameters. The task-invoker acts as a generic bridge — the Step Function knows only one queue (task-invoker's queue), and task-invoker routes to the correct worker queue at runtime. This makes the Step Function definition static and reusable while keeping routing logic in code.

---

### 6.3 xml-generator Lambda

**Trigger:** SQS message from task-invoker

**Responsibilities:**
```
1. Receive: { caseId, caseJson, profileId, taskToken, format: 'XML_E2BR3' }
2. Fetch E2B R3 mapping configuration for this profile from DB
3. Transform caseJson → E2B R3 XML structure
   - Apply ICH/EMA field mappings
   - Validate against E2B R3 XSD schema
   - Reject and signal error if schema validation fails
4. Upload generated XML to S3:
   s3://reports-bucket/{caseId}/{reportId}/report.xml
5. Create GenerationRecord in DB:
   { caseId, profileId, format, s3Key, status: 'GENERATED', generatedAt }
6. Invoke Audit Service asynchronously:
   POST /audit { entityType: 'GENERATION', entityId: reportId,
                 action: 'CREATE', details: { s3Key, format } }
7. Send TaskToken back to Step Function:
   StepFunctions.sendTaskSuccess({
     taskToken,
     output: JSON.stringify({ reportId, s3Key, status: 'SUCCESS' })
   })
   OR on failure:
   StepFunctions.sendTaskFailure({
     taskToken,
     error: 'XML_GENERATION_FAILED',
     cause: errorMessage
   })
```

**E2B XML generation note:**  
E2B R3 is a highly structured XML format with ~1,000+ possible fields. The mapping from the internal case JSON to E2B fields is driven by a configuration table in the DB, not hard-coded. This allows different profiles to map slightly different field sets (e.g., one profile for EMA, one for a national authority with additional required fields).

---

### 6.4 pdf-generator Lambda

**Trigger:** SQS message from task-invoker

**Responsibilities:**
```
1. Receive: { caseId, caseJson, profileId, taskToken, format: 'PDF' }
2. Fetch PDF template configuration for this profile
3. Render PDF using a template engine (CIOMS form, MedWatch form, etc.)
4. Upload to S3: s3://reports-bucket/{caseId}/{reportId}/report.pdf
5. Create GenerationRecord in DB, invoke audit service
6. Send TaskToken back to Step Function (success or failure)
```

---

### 6.5 destination-assembler Lambda

**Trigger:** Invoked directly by ReportGenerator Step Function on completion

**Responsibilities:**
```
1. Receive the Step Function output containing all generation results
2. Check generation status for all formats from DB
   - If ANY generation failed: mark case as ERROR, stop pipeline
   - If ALL successful: proceed

3. Query the Rule table for all destinations:
   SELECT r.destination_id, d.type, d.config, r.profile_id, p.format
   FROM rules r
   JOIN destinations d ON r.destination_id = d.id
   JOIN profiles p ON r.profile_id = p.id
   WHERE r.workflow_state_id = ?

4. Cross-reference: for each rule, fetch the S3 key of the
   generated report for that profile (from GenerationRecords in DB)

5. Build payload for ReportSender Step Function:
   {
     caseId,
     destinationsToSend: [
       { destId, type: 'SAFETY_SYSTEM', config: {...argusConfig},
         s3Key: '...xml path...', reportId },
       { destId, type: 'EMAIL', config: {...emailConfig},
         s3Key: '...pdf path...', reportId },
       ... (one entry per rule)
     ]
   }

6. Invoke ReportSender Step Function with this payload
```

**The key insight here:** destination-assembler is the bridge between generation and sending. It resolves the profile-to-S3-key mapping so that each sender lambda receives a concrete S3 key to fetch, not an abstract profile reference. It also enforces the fail-fast rule: if generation failed, transmission never starts.

---

### 6.6 safety-system-sender Lambda (Argus)

**Trigger:** SQS message from task-invoker

**Responsibilities:**
```
1. Receive: { caseId, destId, s3Key, reportId, taskToken, config: argusConfig }
2. Download XML from S3
3. Submit XML to Argus via configured integration (SOAP/REST/file-based)
4. Parse Argus response for acknowledgement reference number
5. Update TransmissionRecord in DB:
   { caseId, destId, reportId, status: 'SENT', argusRefNo, sentAt }
6. Invoke Audit Service:
   POST /audit { entityType: 'TRANSMISSION', entityId: transmissionId,
                 action: 'UPDATE', details: { argusRefNo, sentAt } }
7. Send TaskToken to Step Function (success or failure)
```

**Argus integration note:**  
Argus Safety is Oracle's pharmacovigilance system. Integration is typically via an E2B XML gateway — the XML is submitted and Argus returns an ACK with its internal case number. This reference number is captured in the audit trail, which is a regulatory requirement for traceability.

---

### 6.7 email-sender Lambda

**Trigger:** SQS message from task-invoker

**Responsibilities:**
```
1. Receive: { caseId, destId, s3Key, reportId, taskToken, config: emailConfig }
2. Download report from S3 (PDF or XML depending on rule)
3. Send email via AWS SES v2:
   - To: emailConfig.recipients
   - Subject: emailConfig.subject (may include case ID, due date)
   - Attachment: the report file
4. Update TransmissionRecord in DB, invoke audit service
5. Send TaskToken to Step Function
```

---

### 6.8 distribution-resolver Lambda

**Trigger:** Invoked directly by ReportSender Step Function on completion

**Responsibilities:**
```
1. Fetch all GenerationRecords for this case from DB
2. Fetch all TransmissionRecords for this case from DB
3. Fetch the configured rules for this workflow state

4. Evaluate rules against records:
   For each rule (profileId, destinationId):
     - Is there a GenerationRecord with status=GENERATED for profileId?
     - Is there a TransmissionRecord with status=SENT for destinationId?
     If either missing or failed → mark case as ERROR

5. If ALL rules satisfied → mark case workflow state as COMPLETED
   → Invoke workflow engine to advance case to next state

6. If ANY rule violated → mark case workflow state as ERROR
   → Log detailed error reason in DB
   → Case goes to manual review queue

7. Invoke audit service for final status update
```

**This is the rule engine heart of the module.** The resolver doesn't need to know about formats or destinations in detail — it just checks DB records against the configured rule set. Adding a new rule (new profile-destination pair) requires only a DB insert — no code change.

---

## 7. Data Model

```sql
-- Case table (owned by upstream — referenced here)
CREATE TABLE cases (
  id              UUID          PRIMARY KEY,
  case_number     VARCHAR(50)   UNIQUE NOT NULL,
  status          VARCHAR(30)   NOT NULL,    -- 'IN_PROGRESS' | 'COMPLETED' | 'ERROR'
  due_date        TIMESTAMPTZ,               -- calculated from source/form rules
  case_json       JSONB         NOT NULL,    -- canonical adverse event data
  created_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

-- Profile: defines a report format and its generation configuration
CREATE TABLE profiles (
  id              UUID          PRIMARY KEY,
  name            VARCHAR(100)  NOT NULL,    -- 'E2B_R3_XML', 'CIOMS_PDF', 'E2B_R2_XML'
  format          VARCHAR(20)   NOT NULL,    -- 'XML' | 'PDF' | 'DOCX'
  standard        VARCHAR(20),               -- 'E2B_R3' | 'E2B_R2' | null
  config          JSONB,                     -- field mapping config, template ref
  created_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

-- Destination: defines where to send a report
CREATE TABLE destinations (
  id              UUID          PRIMARY KEY,
  name            VARCHAR(100)  NOT NULL,    -- 'Argus_EU', 'Pharmacovigilance_Email'
  type            VARCHAR(30)   NOT NULL,    -- 'SAFETY_SYSTEM' | 'EMAIL' | 'PORTAL'
  config          JSONB         NOT NULL,    -- connection details, recipients, etc.
  created_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

-- Rule: links a profile to a destination for a specific workflow state
-- Each rule = "generate THIS format and send it to THIS destination"
CREATE TABLE rules (
  id                  UUID      PRIMARY KEY,
  workflow_state_id   UUID      NOT NULL,    -- FK to workflow state config
  profile_id          UUID      NOT NULL REFERENCES profiles(id),
  destination_id      UUID      NOT NULL REFERENCES destinations(id),
  is_active           BOOLEAN   DEFAULT TRUE,
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE(workflow_state_id, profile_id, destination_id)
);

-- GenerationRecord: tracks each report generation attempt
CREATE TABLE generation_records (
  id              UUID          PRIMARY KEY,
  case_id         UUID          NOT NULL REFERENCES cases(id),
  profile_id      UUID          NOT NULL REFERENCES profiles(id),
  s3_key          TEXT,                      -- populated on success
  status          VARCHAR(20)   NOT NULL,    -- 'IN_PROGRESS' | 'GENERATED' | 'FAILED'
  error_message   TEXT,
  generated_at    TIMESTAMPTZ,
  created_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

-- TransmissionRecord: tracks each send attempt per destination
CREATE TABLE transmission_records (
  id              UUID          PRIMARY KEY,
  case_id         UUID          NOT NULL REFERENCES cases(id),
  destination_id  UUID          NOT NULL REFERENCES destinations(id),
  generation_id   UUID          NOT NULL REFERENCES generation_records(id),
  status          VARCHAR(20)   NOT NULL,    -- 'IN_PROGRESS' | 'SENT' | 'FAILED'
  external_ref    VARCHAR(200), -- Argus case number, SES message ID, etc.
  error_message   TEXT,
  sent_at         TIMESTAMPTZ,
  created_at      TIMESTAMPTZ   NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_gen_case    ON generation_records(case_id);
CREATE INDEX idx_trans_case  ON transmission_records(case_id);
CREATE INDEX idx_rules_state ON rules(workflow_state_id);
```

---

## 8. Key Design Decisions and Rationale

### Decision 1 — AWS Step Functions with WaitForTaskToken for async coordination

**Problem:** Generator and sender lambdas are async — they do real work (XML generation, Argus submission) that takes seconds to minutes. The orchestrator needs to wait for all of them before proceeding.

**Options considered:**
```
Option A: Polling loop
  Lambda checks DB every N seconds until all jobs complete
  Problem: wasteful, adds latency, risk of timeout

Option B: Direct Lambda invocation (synchronous)
  Each lambda called synchronously by the Step Function
  Problem: Lambda max timeout is 15 minutes; XML generation + Argus
  submission chain could exceed this; no parallelism

Option C: Step Functions WaitForTaskToken (chosen)
  Step Function passes a task token to each lambda
  Lambda does its work and calls SendTaskSuccess/Failure with the token
  Step Function resumes only when all tokens are returned
  Map state enables parallel execution across formats/destinations
```

**Why WaitForTaskToken:**
- True async — Step Function is paused at zero cost while lambdas work
- Parallelism via Map state — all formats generated simultaneously
- No polling loop — event-driven resumption
- Clean separation — lambdas don't need to know about each other

---

### Decision 2 — Two separate Step Functions (ReportGenerator + ReportSender)

**Problem:** Generation and sending are semantically different phases with a clear gate between them (destination-assembler checks generation success before proceeding).

**Options:**
```
Option A: Single monolithic Step Function
  All states in one machine
  Problem: harder to debug, harder to add new phases,
  error in generation bleeds into sender state definition

Option B: Two Step Functions with lambda bridge (chosen)
  destination-assembler acts as the bridge
  It evaluates generation results and decides whether to invoke ReportSender
  Problem: slightly more complexity — two state machines to maintain
```

**Why two:**
- Clear single responsibility per state machine
- destination-assembler can abort the pipeline cleanly without the Step Function needing conditional branching logic
- Each state machine can be versioned, tested, and deployed independently
- Mirrors the conceptual phases: "generate everything first, then send everything"

---

### Decision 3 — Generic task-invoker Lambda as a dispatcher

**Problem:** Step Functions Map state iterates over a list and invokes a task. The target lambda changes per iteration (xml-gen vs pdf-gen vs safety-sender vs email-sender). Step Functions cannot dynamically dispatch to different SQS queues in a Map state natively.

**Options:**
```
Option A: Separate Step Function state per format/destination
  Hard-code one state per generator/sender
  Problem: adding a new format requires changing the Step Function definition
  and re-deploying — violates the configurability requirement

Option B: task-invoker as generic router (chosen)
  Step Function knows only one queue: task-invoker
  task-invoker receives targetQueueUrl in its payload and forwards there
  New formats/destinations = new lambdas + DB config only
```

**Why task-invoker:**
- Step Function definition never needs to change when new formats are added
- The routing table lives in the payload assembled by profile-assembler — DB-driven
- Single lambda to maintain for the dispatching concern

---

### Decision 4 — Configuration-driven rules (not code-driven)

**Problem:** Different clients, different regulatory jurisdictions, different forms require different report formats sent to different destinations. Hard-coding these would mean a deployment per client.

**Solution:** Rule table linking profile → destination → workflow_state. Adding a new rule for a client requires only a DB insert — zero code deployment.

**Example configuration for a client:**
```
Rule 1: workflow_state=RD_001, profile=E2B_R3_XML, destination=Argus_EU
Rule 2: workflow_state=RD_001, profile=E2B_R3_XML, destination=Argus_US
Rule 3: workflow_state=RD_001, profile=CIOMS_PDF,   destination=PV_Team_Email
Rule 4: workflow_state=RD_001, profile=CIOMS_PDF,   destination=Regulatory_Email

Client with different requirements:
Rule 1: workflow_state=RD_002, profile=E2B_R2_XML, destination=Argus_Legacy
Rule 2: workflow_state=RD_002, profile=CIOMS_PDF,   destination=Safety_Email
```

The same code handles both clients — only the DB rules differ.

---

### Decision 5 — Status tracking in DB, not in Step Function output

**Problem:** Step Function execution output is ephemeral. If a lambda fails inside the Map state, the Step Function's error handling would need complex conditional logic to determine which lambdas succeeded and which failed.

**Solution:** Each lambda writes its status (`GENERATED`, `FAILED`, `SENT`) directly to the DB as part of its execution — before signalling the Step Function. The distribution-resolver then reads the DB to evaluate the final state, not the Step Function output.

**Why this matters:**
- DB is the single source of truth — auditable, queryable, persistent
- distribution-resolver has full history even if Step Function retries or re-runs
- Failure investigation is a DB query, not a Step Function execution history trawl

---

## 9. Failure Handling and Resilience

### Current failure modes and handling

```
Failure scenario          Current handling               Status
──────────────────────────────────────────────────────────────────
Lambda throws exception   SQS redrives (default retries) Handled
Lambda timeout            SQS redrives                   Handled
XML schema validation     TaskFailure sent to Step Fn    Handled
                          Case marked ERROR
Argus unreachable         TaskFailure sent to Step Fn    Handled
                          Case marked ERROR
Email delivery failure    TaskFailure sent to Step Fn    Handled
                          Case marked ERROR
SQS message lost          No DLQ — message silently lost GAP ⚠
Step Function timeout     Case remains in-progress       GAP ⚠
Partial success           Whole case marked as ERROR     By design
```

### Known gap — no Dead Letter Queue

Currently, if an SQS message exhausts its retry attempts (e.g., a lambda crashes repeatedly), the message is **silently dropped**. The case will remain in `IN_PROGRESS` indefinitely — never resolved.

```
Current:   SQS queue → Lambda → (failure × retries) → message discarded silently
Better:    SQS queue → Lambda → (failure × retries) → DLQ
                                                          │
                                                          ▼
                                                    Alert + manual review
                                                    Case marked ERROR
```

This is the most significant resilience gap in the current design.

### TaskToken failure flow

```
Lambda fails:
  └── sendTaskFailure(taskToken, error, cause)
        │
        ▼
  Step Function Map state records failure for that iteration
        │
        ▼
  All iterations complete (success or failure)
        │
        ▼
  destination-assembler invoked
        │
        ▼
  destination-assembler reads DB → finds FAILED GenerationRecord
        │
        ▼
  Does NOT invoke ReportSender
  Invokes distribution-resolver directly with error status
        │
        ▼
  distribution-resolver marks case as ERROR
```

### Due-date tracking and urgency

```
Serious cases (7-day or 15-day regulatory deadline):
  Due date stored in cases.due_date
  Calculated upstream based on source/form rules at ingestion time
  The distribution module does not calculate due dates — it respects them

Monitoring:
  A separate background job queries:
    SELECT * FROM cases
    WHERE status = 'IN_PROGRESS'
      AND due_date < NOW() + INTERVAL '24 hours'
  Alerts the operations team for cases approaching deadline
  Does NOT automatically expedite — human decision required
```

---

## 10. Audit and Compliance

### Audit service integration

The audit service is a separate microservice that maintains an **immutable event log** of all actions across the PV platform.

```
What gets audited in this module:
  ┌──────────────────────────────────────────────────────────────┐
  │ Event                │ Trigger Lambda      │ Key Fields      │
  ├──────────────────────┼─────────────────────┼─────────────────┤
  │ Report generated     │ xml-gen, pdf-gen    │ caseId,         │
  │                      │                     │ reportId, s3Key │
  │                      │                     │ format, time    │
  ├──────────────────────┼─────────────────────┼─────────────────┤
  │ Report uploaded to S3│ xml-gen, pdf-gen    │ s3Key, bucket   │
  ├──────────────────────┼─────────────────────┼─────────────────┤
  │ Transmission started │ sender lambdas      │ destId, caseId  │
  ├──────────────────────┼─────────────────────┼─────────────────┤
  │ Sent to Argus        │ safety-sys-sender   │ argusRefNo      │
  │                      │                     │ sentAt, caseId  │
  ├──────────────────────┼─────────────────────┼─────────────────┤
  │ Email sent           │ email-sender        │ sesMessageId    │
  │                      │                     │ recipients      │
  ├──────────────────────┼─────────────────────┼─────────────────┤
  │ Case completed       │ distribution-       │ completedAt     │
  │ Case errored         │ resolver            │ errorReason     │
  └──────────────────────┴─────────────────────┴─────────────────┘

Audit invocation pattern:
  Lambda completes its primary task (generate/send)
  Lambda writes status to its own DB table (synchronous — must succeed)
  Lambda invokes audit service asynchronously (fire-and-forget)
    → Audit failure does NOT fail the primary operation
    → A background reconciliation job catches missed audit entries
       by comparing DB records with audit log entries
```

### Why audit is invoked asynchronously

The primary operation (generation, transmission) must not be coupled to audit service availability. If the audit service is temporarily down, the case must still complete. The audit service has its own retry mechanism and the reconciliation job closes any gaps.

### Regulatory traceability requirements met

```
Requirement                                   How met
──────────────────────────────────────────────────────────────────
Every report submission must be logged        TransmissionRecord in DB +
                                              Audit service entry
Argus reference number must be captured       external_ref in TransmissionRecord
                                              + audit entry detail
Reports must be retained after transmission   S3 objects never deleted
Submission timestamps must be recorded        sent_at in TransmissionRecord
Failed submissions must be identifiable       status='FAILED' + error_message
                                              in TransmissionRecord
```

---

## 11. Trade-offs and Known Limitations

### Trade-off 1 — All-or-nothing failure model

**Decision:** If any generation or transmission fails, the entire case is marked as ERROR.  
**Benefit:** Simple, clear, unambiguous. A case is either fully distributed or not. Regulatory submissions require certainty — partial distribution is ambiguous.  
**Cost:** If PDF generation fails but XML was already sent to Argus successfully, both are marked as failed and the case requires manual review, even though Argus already has the case. In practice, this creates re-work.  
**Better approach:** A more granular status model — track success per rule, flag the case as partially complete, and allow re-running only the failed rules.

### Trade-off 2 — task-invoker adds an extra SQS hop

**Decision:** Step Function → SQS → task-invoker → SQS → worker lambda (two hops).  
**Benefit:** Enables dynamic routing without changing the Step Function definition.  
**Cost:** Extra latency (~50-100ms per hop), extra Lambda invocation cost, an additional point of failure.  
**Better approach:** As AWS Step Functions evolve, the SDK integration allows direct SQS sendMessage from a Map state with dynamic queue URLs — the task-invoker could be eliminated. Worth revisiting.

### Trade-off 3 — No DLQ on SQS queues

**Decision:** Shipped without DLQ to meet delivery timeline.  
**Cost:** Messages that exhaust retries (due to repeated Lambda failures) are silently dropped. Cases remain stuck in `IN_PROGRESS` indefinitely.  
**Better approach:** Add DLQ to every queue. Messages landing in DLQ trigger an alert and automatically mark the case as ERROR with the DLQ message details as the error reason.

### Trade-off 4 — E2B XML mapping is DB-configured but validated in code

**Decision:** The XSD schema validation for E2B R3 runs inside the xml-generator lambda.  
**Benefit:** Catches malformed XML before submission.  
**Cost:** XSD schemas change when regulations update. Schema updates require a Lambda code deployment, not just a DB config change.  
**Better approach:** Store the XSD schema in S3 alongside the profile configuration. The lambda downloads and validates against the version referenced in the profile config — schema updates become an S3 upload, not a code deployment.

### Trade-off 5 — Synchronous DB write before TaskToken signal

**Decision:** Each lambda writes its status to DB before calling `sendTaskSuccess`.  
**Benefit:** DB is authoritative — even if the TaskToken signal fails (rare AWS issue), the DB record is correct.  
**Cost:** If the DB write succeeds but `sendTaskSuccess` fails, the Step Function will retry the lambda. The lambda must be idempotent — writing the same DB record twice should be a no-op (use `ON CONFLICT DO UPDATE`).

---

## 12. Future Improvements

### Priority 1 — Add DLQ to all SQS queues

```
Current: message loss possible on repeated failure
Target:
  Every queue → DLQ
  DLQ event → Lambda → mark case ERROR + alert operations team
  Implementation: 1-2 days, high impact
```

### Priority 2 — Partial re-run capability

```
Current: full failure if any rule fails — must re-run entire distribution
Target:
  Track success per rule in DB
  distribution-resolver marks case 'PARTIAL_ERROR' with failed rule IDs
  UI allows re-running only failed rules without re-generating succeeded ones
  Implementation: medium complexity, high value for operations team
```

### Priority 3 — Eliminate task-invoker hop

```
Current:  Step Fn → SQS(task-invoker) → SQS(worker)   [2 hops]
Target:   Step Fn → SQS(worker)                         [1 hop]

Use Step Functions SDK integration (optimised integration):
  "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken"
  With dynamic QueueUrl from payload state
  Eliminates task-invoker Lambda entirely
  Saves ~100ms latency and one Lambda cost per iteration
```

### Priority 4 — S3 schema versioning for E2B XSD

```
Store E2B R3 XSD in S3:
  s3://config-bucket/schemas/e2b_r3_v1.xsd
  s3://config-bucket/schemas/e2b_r3_v2.xsd

Profile config references schema version:
  { "schemaVersion": "e2b_r3_v2", ... }

xml-generator downloads schema at runtime:
  const schema = await s3.getObject({ Key: profile.config.schemaVersion })
  validateXML(xml, schema)

Schema updates: S3 upload + DB profile update → no Lambda deployment
```

### Priority 5 — Observability dashboard

```
Current: case status queryable from DB; no real-time visibility
Target:
  CloudWatch dashboard:
    - Active cases in each pipeline stage
    - Average time per stage (generation, transmission)
    - Cases approaching due date
    - DLQ depth per queue
    - Error rate by error type
  Alerts:
    - Case in ERROR for > 1 hour → PagerDuty
    - Any DLQ message → immediate alert
    - Due-date breach → email to operations team
```

---

## 13. Interview Script

### Opening — set the domain context first

> "I want to walk you through a system I designed and led in my current role at a Pharmacovigilance company. The domain is drug safety reporting — when patients report adverse reactions to drugs, companies are legally required to submit structured electronic reports called ICSRs to regulatory authorities like the EMA within 7 or 15 days depending on severity. Late or missed submissions are regulatory violations. So the system I'm going to describe is compliance-critical — correctness, traceability, and audit completeness matter more than raw throughput."

### The problem being solved

> "The original system handled report generation as individual workflow states — one state for XML generation, one for PDF, one for sending to Argus, one for email. For every new client or new destination configuration, you had to wire up new workflow states. It was rigid and hard to configure without engineering involvement. We redesigned the entire report distribution into a single configurable module where you can attach any combination of profile-destination rules and the system handles the rest."

### The key architectural decision — async orchestration with Step Functions

> "The core challenge is parallelism and coordination. For a given case, you might need to generate an E2B R3 XML, a CIOMS PDF, then send the XML to Argus and two different email destinations, send the PDF to three other email destinations — all in parallel, all asynchronously, and then confirm every single one succeeded before marking the case complete. We used AWS Step Functions with WaitForTaskToken. The Step Function passes a task token to each worker lambda. The lambda does its work — generates the XML, submits to Argus — then signals the Step Function back with that token. The Map state handles parallelism across formats and destinations. The Step Function waits for all tokens before proceeding."

### The rule engine

> "The configurability comes from a three-table rule model in PostgreSQL — Profile, Destination, and Rule. A Profile defines what to generate: E2B R3 XML, CIOMS PDF. A Destination defines where to send it: Argus EU, PV team email. A Rule links a profile to a destination for a specific workflow state. Adding a new client configuration is purely a DB operation — no code deployment. The distribution-resolver lambda at the end evaluates every rule against DB records of what was actually generated and sent, and only marks the case complete if every rule is satisfied."

### Trade-offs to mention proactively

> "Two trade-offs I'd call out. First: we use an all-or-nothing failure model — if any generation or transmission fails, the whole case is marked as Error. That's intentional from a regulatory standpoint — ambiguous partial success is dangerous. But it does create re-work when, say, PDF fails but XML was already sent to Argus successfully. A partial re-run capability is on our roadmap. Second: we shipped without a DLQ on our SQS queues. That means a message that exhausts retries is silently dropped and the case stays stuck. That's a known gap we're addressing — adding DLQ with automatic case error-marking is the top priority improvement."

---

## 14. Follow-up Probes and Answers

**"How does the system handle a case with a 7-day regulatory deadline?"**  
The due date is calculated upstream at ingestion time based on configured rules against the source and form type — for example, "serious spontaneous report = due date of 7 calendar days from receipt." This due date is stored on the case. The distribution module respects it but doesn't enforce it at the pipeline level — a background monitoring job queries cases approaching their due date and alerts the operations team. The team then prioritises manually. An improvement would be a priority queue — serious cases with approaching due dates get a higher-priority SQS message and a dedicated set of Lambda concurrency.

**"What happens if Argus is down for 2 hours?"**  
Currently: the safety-system-sender lambda fails after its timeout, sends TaskFailure to the Step Function, the destination-assembler detects the failure, and the entire case is marked as ERROR. The case requires manual re-triggering after Argus recovers. A better approach would be a retry strategy with exponential backoff at the lambda level before signalling failure — attempt submission 3 times over 30 minutes before giving up. This would handle transient Argus downtime transparently.

**"Why did you choose Step Functions over a custom orchestration?"**  
Step Functions give us visual execution history, built-in retry logic, WaitForTaskToken for async coordination, and Map state for parallel execution — all without writing orchestration code. The alternative was a custom state machine in Lambda backed by DB polling, which would require significant engineering to achieve the same guarantees. The operational visibility alone — being able to open the Step Functions console and see exactly where a case is stuck — justified the choice.

**"How do you ensure the E2B XML is schema-compliant?"**  
The xml-generator lambda validates the generated XML against the E2B R3 XSD schema before uploading to S3. If validation fails, the lambda sends a TaskFailure with the schema error details, which ultimately marks the case as Error. Schema validation errors are surfaced in the error message on the case record. A known limitation: when the E2B schema is updated (which happens with ICH guideline revisions), we currently need to update the lambda code to use the new XSD — a future improvement is storing versioned XSDs in S3 and referencing them from the profile configuration, making schema updates a configuration change.

**"How would you add support for a new destination type — say, a web portal submission?"**  
Three steps. First, implement a new `portal-sender` Lambda that handles the portal's API. Second, add a new destination record in the DB with type `PORTAL` and the portal's connection config. Third, add rules in the DB linking the appropriate profile to this new destination. The task-invoker Lambda will route to the portal-sender's SQS queue because the queue URL is carried in the payload assembled by destination-assembler — no Step Function changes needed. This is the configurability the design was built for.

**"How do you handle idempotency — what if a lambda is retried after it already succeeded?"**  
Each lambda checks for an existing GenerationRecord or TransmissionRecord for that case and profile/destination combination before doing any work. If a record already exists with status `GENERATED` or `SENT`, the lambda skips the primary operation and directly sends TaskSuccess with the existing record's data. This makes every lambda idempotent — safe to retry. DB upserts use `ON CONFLICT DO UPDATE` to handle the race condition where two retried invocations arrive simultaneously.

---


# ADR 0001: Correlation ID process for the Python-based TWS adapter

**Status**: Proposed

**Date**: 2026-10-15 (revision 3; supersedes the 2026-10-13 draft)

**Authors**: Amit Thorat, Chandra Sekhar Reddy Buchha — Kroger Hybrid Workflow Orchestration Squad

**Contributors**: Tanmay Kachariya (listener), Akshata Dalvi (publisher)

**Reviewers**: Doug Heitkamp

**Decision forum**: Hybrid Workflow Orchestration checkpoint, 2026-10-09; status update, 2026-10-16

---

## Context

Hybrid workflow orchestration connects on-prem TWS (HCL Workload Automation) and Databricks through **data availability events**, not job-status coupling. Every event follows the shared data-available schema (Confluence: *Systems Integration Solutions → Schema*, Avro, backlog #16). Each event carries a `correlation_id` so one hybrid run can be traced end to end.

The TWS adapter (deck slide 6) is a Python component on AIX with two halves:

| Step | What happens | Component |
|---|---|---|
| 1 | Producers (Databricks jobs, Informatica jobs, file-transfer drops) publish a data availability event to DESP / Azure Event Hub | Producers |
| 2 | Consume, validate, map to a TWS custom event, handle retry and idempotency using durable state | **TWS Event Listener** |
| 3 | Submit the custom event with `sendevent` | Listener → TWS |
| 4 | Event Processing Server evaluates event rules and runs the action | TWS |
| 5 | **Option 1:** submit a job / job stream (event-driven run). **Option 2:** reply to a prompt that an *existing scheduled* run is waiting on | TWS |
| 6 | The run executes the Informatica PowerCenter workflow | Informatica |
| 7 | Collect run values, validate, build the event per the schema, publish to DESP, store a per-job audit record | **TWS Event Publisher** |

`sendevent` is **asynchronous**. A successful response only confirms the event was *submitted*. It does not say which rule evaluated it, which action fired, or which job run resulted. As a result:

- **C1 — Event-to-job traceability is missing**: we cannot say which event started a given job run.
- **C2 — Rule / action traceability is missing**: we cannot say which rule and action processed a given event.
- The publisher cannot tell which inbound `correlation_id` to carry forward, so the chain breaks at every TWS hop.

On 2026-10-09 Doug Heitkamp confirmed the design serves two purposes: put all diagnostic and audit data in one place, and establish the correlation ID so a job run can be traced to its triggering event.

**Question:** how does the adapter re-establish the link between an inbound event and the job run it caused, when `sendevent` does not return it?

## Decision

**The adapter owns the correlation process. It rebuilds the event → job run → published event chain from one source-controlled event config and three Delta audit tables. Run facts come from TWS's supported interfaces (DISC / REST API, job logs). The adapter never reads the TWS database.**

The 9 Oct design matched on "the latest successful event for the dataset". This revision replaces that with an explicit **claim**: each event is bound to one specific job run, or flagged.

### Components

| Component | Type | Written by | Purpose |
|---|---|---|---|
| `TWS_EVENT_CONFIG` | YAML, source-controlled | Onboarding (PR) | Per workflow: consumed / produced datasets, TWS rule and action, job / job stream / prompt, Informatica folder / workflow / session, static event attributes, claim settings |
| `TWS_EVENT_LISTENER_AUDIT` | Delta table | Listener | One row per inbound event: its identity, routing, `sendevent` result and processing state. Also the listener's durable state for idempotency |
| `TWS_JOB_AUDIT` | Delta table | Run resolver | One row per claimed TWS job run: event ID, correlation ID, job stream instance, job number, status and times. This is the deck's "Job Audit Table" |
| `TWS_EVENT_PUBLISHER_AUDIT` | Delta table | Publisher | One row per outbound event: new event ID, carried-forward correlation ID, source job run, payload summary, publish result |

Processes on AIX:

| Process | Runs as | Responsibility |
|---|---|---|
| **Listener** | Long-running service | Steps 2–3 |
| **Run resolver** | To decide: service, loop, or TWS job (Tanmay, query 1) | Claim runs, track them to a final state |
| **Publisher** | Same host and pattern as the resolver | Step 7 |

---### Sequence

```mermaid
sequenceDiagram
    autonumber
    participant EH as DESP / Event Hub
    participant L as Listener (Python)
    participant LA as TWS_EVENT_LISTENER_AUDIT
    participant TWS as TWS Event Processing
    participant R as Run resolver
    participant JA as TWS_JOB_AUDIT
    participant INF as Informatica
    participant P as Publisher
    participant PA as TWS_EVENT_PUBLISHER_AUDIT

    EH->>L: data-available event (event_id, correlation_id)
    L->>LA: dedupe on event_id; insert RECEIVED
    L->>L: validate schema, route via TWS_EVENT_CONFIG
    L->>TWS: sendevent (custom event)
    L->>LA: update SUBMITTED (or SUBMIT_FAILED)
    L->>EH: checkpoint (only after the audit row is committed)
    TWS->>INF: rule action → submit job stream / reply to prompt → workflow runs
    R->>LA: read SUBMITTED, unclaimed events
    R->>TWS: DISC API: runs of the configured job / job stream
    R->>JA: claim one run (event_id, correlation_id, instance id)
    R->>TWS: poll until SUCC / ABEND
    R->>JA: update status, start / end
    P->>JA: read runs in publish_on_status, not yet published
    P->>INF: read session log (via config mapping)
    P->>EH: publish new event (new event_id, same correlation_id)
    P->>PA: insert PUBLISHED; mark job audit row published
```
----
### Event state model

Each inbound event moves through one state in `TWS_EVENT_LISTENER_AUDIT.PROCESSING_STATE`:

```
RECEIVED → SUBMITTED → CLAIMED → RUN_COMPLETE → PUBLISHED
    │           │          │            │
    ├ DUPLICATE ├ SUBMIT_FAILED          ├ RUN_FAILED (ABEND)   └ PUBLISH_FAILED
    └ INVALID   └ UNMATCHED / AMBIGUOUS (no run, or more than one candidate, inside the window)
```

Terminal error states raise an alert (backlog #40). Failed events can be re-driven by the reprocess utility (#46).

### Listener design

1. **Consume** from Event Hub / DESP with a dedicated consumer group. Checkpoint per partition **only after** the audit row for the event is committed.
2. **Validate** against the data-available schema. Required fields and envelope copies must equal their payload fields (`eventHeader.datasetName` = `data.dataset_name`, and so on). Failures → `INVALID`.
3. **Deduplicate** on `eventHeader.id` against `TWS_EVENT_LISTENER_AUDIT`. An already-submitted ID → `DUPLICATE`, no second `sendevent` (backlog #10, #32).
4. **Correlation ID at the start of a chain.** The schema makes `data.correlation_id` optional, but `TWS_JOB_AUDIT.CORRELATION_ID` is not nullable. If the inbound event has none, the listener **mints one** (UUID) and records `CORRELATION_ORIGIN = MINTED`.
5. **Route** by `domain` + `dataset_name` to a config entry. No entry → `INVALID` (unrouted), alert.
6. **Build the custom event.** Carry `event_id`, `correlation_id`, `dataset_name`, `domain`, `availability_type` and `business_date` as custom event properties, so rules can filter on them. This also keeps Alternative C possible.
7. **`sendevent`** with bounded retry and back-off. Record return code, retry count and error.
8. **Audit** the outcome, then checkpoint.

### Run resolver and claim rule

The resolver works on `SUBMITTED` events that have no claim. It uses the config entry's `tws` block to query the DISC API **by job / job stream name**. The API requires name filters, so the resolver never scans the whole plan.

The rule depends on the action type (deck slide 6):

| Action | What counts as the run | Claim when |
|---|---|---|
| **Submit job / job stream** (Option 1) | A **new** job stream instance created after the event | Name matches config; instance **created after** `SUBMITTED_AT` and within `claim.window_minutes`; instance not already claimed |
| **Reply to prompt** (Option 2) | The **planned** instance (from Jnextday) that was waiting on the configured prompt | Instance depends on the configured prompt; prompt answered after `SUBMITTED_AT`; the run **started** after that; instance not already claimed |

- The two rules exist because an Option 2 run is a **planned run by design**. A blanket rule of "ignore plan-created runs" would never match prompt actions.
- No candidate inside the window → `UNMATCHED`. More than one → `AMBIGUOUS`. **The resolver never guesses.**
- Every claim records its basis (`MATCH_BASIS`, `MATCH_DELTA_SECONDS`) so a wrong match can be found later.
- After the claim, the resolver polls the run until a final state (SUCC / ABEND). It updates `JOB_STATUS`, `JOB_START_TIME` and `JOB_END_TIME`.

### Publisher design

1. Pick up `TWS_JOB_AUDIT` rows whose `JOB_STATUS` is in the entry's `publish_on_status` and that are not yet published.
2. Find the Informatica session log through the config's `informatica` block. This is the job → workflow / session mapping that is missing today.
3. Build the payload from three sources (publisher field mapping, Akshata):
   - **Session log:** row count from `CMN_1740` applied rows; completion time from `TM_6020`; session run ID; database / schema / table or file name.
   - **Config file:** domain, dataset name, availability type, resource type, system, host / path, watermark column.
   - **Python:** event ID (new UUID), event time, event type, schema version, envelope copies, watermark value.
4. Take `data.correlation_id` from the claimed `TWS_JOB_AUDIT` row, never from a "latest event" lookup.
5. Validate against the schema, publish to DESP, and write `TWS_EVENT_PUBLISHER_AUDIT`.
6. Publishing is **idempotent per job run**. A unique key on `JOB_AUDIT_ID` in the publisher audit stops a re-run from emitting a second event.

### Event config (illustrative, to be finalised with backlog #27 / #28)

```yaml
version: 1
workflows:
  - workflow_id: wf_sales_daily_load              # unique key
    consumes:
      - { domain: sales, dataset_name: pos_transactions_daily }
    produces:
      - domain: sales
        dataset_name: sales_daily_summary
        availability_type: BUSINESS_DATE
        resource:
          resource_type: EDW_TABLE
          system: EDW
          watermark_column: LOAD_TS
    tws:
      custom_event: DataAvailable                 # custom event type sent via sendevent
      rule: RL_SALES_DAILY_DA
      action: SUBMIT_JOBSTREAM                    # SUBMIT_JOB | SUBMIT_JOBSTREAM | REPLY_PROMPT
      workstation: <WS>
      job_stream: JS_SALES_DAILY
      job: J_SALES_DAILY_LOAD
      prompt: null                                # set when action = REPLY_PROMPT
    informatica:                                  # closes the TWS job → session log gap
      folder: <FOLDER>
      workflow: wf_sales_daily_load
      sessions: [s_m_sales_daily_load]
    claim:
      window_minutes: 30
    publish_on_status: [SUCC]
```

### Audit tables: proposed columns

`TWS_JOB_AUDIT` is Akshata's design and is shown as drafted. The other two tables are **proposals for Tanmay and Akshata to confirm**.

**`TWS_EVENT_LISTENER_AUDIT`**

| Column | Type | Notes |
|---|---|---|
| LISTENER_AUDIT_ID | BIGINT identity | PK |
| EVENT_ID | STRING | `eventHeader.id`; unique, used for dedup |
| CORRELATION_ID | STRING | Inbound, or minted |
| CORRELATION_ORIGIN | STRING | INBOUND / MINTED |
| EVENT_SOURCE | STRING | `eventHeader.source` |
| DOMAIN, DATASET_NAME | STRING | Routing keys |
| AVAILABILITY_TYPE, BUSINESS_DATE | STRING, DATE | From payload |
| EVENT_TIME | TIMESTAMP | `eventHeader.time` |
| PARTITION_ID, SEQUENCE_NUMBER | STRING, BIGINT | Event Hub position, for replay |
| WORKFLOW_ID | STRING | Config entry matched |
| TWS_RULE, TWS_ACTION, TWS_TARGET | STRING | From config (what *should* fire) |
| SENDEVENT_RC, RETRY_COUNT, ERROR_MESSAGE | INT, INT, STRING | Submission result |
| SUBMITTED_AT | TIMESTAMP | Start of the claim window |
| PROCESSING_STATE | STRING | State model above |
| INGESTION_TIMESTAMP, INGESTION_DATE | TIMESTAMP, DATE | Partitioning |

**`TWS_JOB_AUDIT`** (Akshata's design, plus three columns proposed here)

| Column | Type | Notes |
|---|---|---|
| JOB_AUDIT_ID | BIGINT identity | PK (`JOB_RUN_AUDIT_ID` in the draft) |
| EVENT_ID | STRING | The inbound event that started the run |
| CORRELATION_ID | STRING, not null | Carried to the published event |
| JOB_STREAM_NAME, JOB_STREAM_INSTANCE_ID | STRING | The instance is the claim key |
| JOB_NAME, JOB_NUMBER | STRING, BIGINT | From config / DISC API |
| JOB_STATUS | STRING | Publish only when in `publish_on_status` |
| JOB_START_TIME, JOB_END_TIME | TIMESTAMP | From DISC API / job log |
| *MATCH_BASIS* | STRING | *Proposed*: SUBMIT_AFTER_EVENT / PROMPT_RELEASE |
| *MATCH_DELTA_SECONDS* | INT | *Proposed*: event-to-run gap |
| *PUBLISHED_FLAG* | BOOLEAN | *Proposed*: idempotent publishing |
| INGESTION_TIMESTAMP, INGESTION_DATE | TIMESTAMP, DATE | |

**`TWS_EVENT_PUBLISHER_AUDIT`**

| Column | Type | Notes |
|---|---|---|
| PUBLISHER_AUDIT_ID | BIGINT identity | PK |
| EVENT_ID | STRING | New outbound `eventHeader.id` |
| CORRELATION_ID | STRING | Same as the inbound event |
| SOURCE_EVENT_ID | STRING | Inbound event |
| JOB_AUDIT_ID | BIGINT, unique | The run this event announces |
| DOMAIN, DATASET_NAME, RESOURCE_TYPE | STRING | |
| RECORD_COUNT, AVAILABLE_AT | BIGINT, TIMESTAMP | Payload summary |
| PUBLISH_STATUS, ERROR_MESSAGE | STRING | PUBLISHED / PUBLISH_FAILED |
| PUBLISHED_AT, INGESTION_TIMESTAMP, INGESTION_DATE | TIMESTAMP, TIMESTAMP, DATE | |

## Validation against sources

The 9 Oct deck (slides 3–4, 6), the publisher field mapping and Job Audit Table (Akshata), and the TWS Adapter queries (Tanmay, 14 Oct) were checked against this ADR.

| # | Finding | Resolution in this ADR |
|---|---|---|
| V1 | Deck step 9 matched on the latest successful event by dataset name and domain. That breaks with close events or manual runs. | Replaced by the claim rule |
| V2 | Deck slide 6, Option 2 is a reply to a prompt for an existing **scheduled** run, so that run comes from the plan | Separate claim rule for prompt actions |
| V3 | `data.correlation_id` is optional in the schema; `CORRELATION_ID` is not null in the Job Audit Table | Listener mints one at the start of a chain |
| V4 | `data.available_at`: the EDW mapping takes `TM_6020` from the session log; the Job Audit Table says `JOB_END_TIME` | **Open**: pick one. Proposed: `TM_6020` (data written) over job end (job finished) |
| V5 | Names differ: deck "Job Audit Table" vs. ADR "TWS_JOB_RUN_AUDIT"; "Route config" (Job Audit sheet) vs. `TWS_EVENT_CONFIG` (deck) vs. "Orchestration Config File" (#28) | Table renamed `TWS_JOB_AUDIT`. Config name **open**: agree one name across #27 / #28 |
| V6 | Deck slide 6: listener does retry and idempotency with durable state; the 13 Oct draft did not say so | Listener design steps 3 and 7 |
| V7 | No link from a TWS job to its Informatica session log (Tanmay) | `informatica` block in the config |
| V8 | Job-status source: logs vs. DISC API with name filters (Tanmay) | DISC API with names from config; polling interval **open** |
| V9 | `data.is_complete` defaults to true; the schema says it should not just mean task success unless agreed | **Open**: agree the completion rule per dataset |

## Consequences

**Positive**

- **Unbroken correlation chain** across TWS hops, with each link backed by a recorded claim.
- **One place for diagnostics.** Support can trace a run from the three tables without logging into TWS.
- **No dependency on the TWS database schema.** Only supported interfaces are used.
- **Same pattern on both platforms.** The Databricks adapter uses config, event tables and table-update triggers.
- **Config-driven onboarding.** A new workflow is a reviewed config entry, not a code change.
- **Failures are visible.** Unmatched, ambiguous and failed events are explicit states with alerts, not silent gaps.

**Negative**

- **Correlation is inferred, not reported by TWS.** The claim rule narrows the risk but cannot remove it. Remaining failure cases:
  - `sendevent` reports success but the event never reached the Event Processing Server. The event ends `UNMATCHED`.
  - One prompt gates several jobs. If one event releases a prompt shared by jobs in different config entries, the claim is ambiguous.
  - A manual resubmit of the same job stream inside the window can be claimed by the wrong event.
- **Polling cost.** Long-running jobs are polled until SUCC / ABEND. The load grows with in-flight runs.
- **Rule / action traceability (C2) is still indirect.** We record which rule *should* fire, not which one did.
- **Config drift.** The config duplicates rule metadata held in TWS. Mitigation: one onboarding PR updates both, plus a periodic config-vs-deployed-rules check.
- **New moving parts.** One config file, three tables and up to three AIX processes to own, secure, retain and monitor. The tables live in Azure, so the on-prem scripts take a new network dependency. This is recorded in the hybrid FMEA.

## Alternatives considered

### A. Rely on the `sendevent` acknowledgement only

**Rejected**: no event-to-job or event-to-rule traceability, and the publisher cannot carry the correlation ID forward.

### B. Read the TWS database / plan tables directly

**Rejected**: couples the adapter to an internal, version-specific schema, needs elevated database access, and gives no shared audit store for both platforms.

### C. Carry the correlation ID into the run through the rule action (to investigate)

If the event rule action can pass a custom event property into the submitted job stream (for example as an alias or variable), the run carries the correlation ID itself. Correlation becomes **observed, not inferred**, for Option 1 actions. This is **untested**. If it works, the claim rule stays only for prompt actions and as a fallback.

### D. Keep the 9 Oct "latest successful event" lookup

**Rejected**: two close events, or a manual or planned run, attach a wrong or stale correlation ID with nothing to flag it (V1).

## Open items

| # | Item | Owner |
|---|---|---|
| 1 | Agree the claim rule: window size, tie-break, handling of UNMATCHED / AMBIGUOUS | Amit / Akshata |
| 2 | Run-resolver and publisher hosting on AIX: service, loop, or TWS job | Tanmay |
| 3 | DISC API polling interval and acceptable load | Tanmay / Akshata |
| 4 | Fill the `informatica` mapping for the 16 Phase 1 use cases | Tanmay |
| 5 | Confirm how to detect prompt release in the DISC API / job log | Tanmay |
| 6 | `data.available_at` source: `TM_6020` vs. job end (V4) | Akshata |
| 7 | One config name and schema across #27 / #28 (V5) | Amit |
| 8 | Completion rule for `data.is_complete` (V9) | Team |
| 9 | Confirm proposed columns for listener and publisher audit tables | Tanmay / Akshata |
| 10 | Test Alternative C | Amit |
| 11 | Retention and access for the three tables | Chandra |

## References

- *Hybrid Workflow Orchestration – Task Updates – 09Oct2026* (deck), slides 3–4 (correlation strategy) and 6 (TWS adapter architecture).
- `Correlation.drawio`: correlation ID purpose and challenges C1 / C2.
- Meeting transcript, 2026-10-09: Doug Heitkamp asked for this ADR.
- Publisher field mapping and Job Audit Table (Akshata Dalvi, `Akshathas_Analysis.xlsx`).
- *TWS Adapter queries*, Tanmay Kachariya, 2026-10-14.
- Confluence: *Systems Integration Solutions → Schema*.
- Backlog #10 (event state for retry / idempotency), #16 (event schema), #18 / #19 (TWS / Informatica publisher), #27 / #28 (routing spec / config file), #32 (duplicate protection), #37 (orchestration audit), #40 (alerting), #46 (publish-failure handling).
- [ADR 0002](0002-tws-azure-service-bus-job-plugin-secondary-option.md): keeps the Python listener this ADR depends on.

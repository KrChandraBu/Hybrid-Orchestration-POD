# Hybrid Orchestration on KPS — Developer Build Guide



## 1. Purpose and how to use this guide

This guide is what Tanmay and Akshatha build from, and what Amit reviews. It covers the three KPS workloads (Listener, Correlation Reconciler, Publisher), the correlation ID rules, and the Delta tables they read and write. Production hosting is **KPS** (ADR-003, accepted 29 Oct 2026). The Linux VM and TWS plugin options are closed.

| Person | Role in this guide |
| --- | --- |
| Tanmay | Builds the **Listener** (section 4) and the shared `common` package |
| Akshatha | Builds the **Publisher** (section 6), publish rules and attribute mapping with data owners |
| Amit | Architecture owner ; Validates the correlation ID logic and table design; comments using the checklist in section 12 |
| Chandra | Architecture owner; answers design questions and closes open items |

- **Settled:** items in sections 2–8 unless marked *proposed*.
- **Proposed:** marked with an **S** number (orange in the diagram). These are architecture suggestions, not Kroger decisions yet. Comment if you disagree.
- **Open:** marked with a **Q** number (red in the diagram). Build around them with flags and mocks, don't wait (section 11).
- **How to comment:** select the text and add a comment. Tag the S or Q number when it applies.
- **Diagram:** `Hybrid_Orchestration_E2E_Flow.drawio` (page 1 detailed, page 2 happy path). Hover any S, Q or step badge for its full text.
- **Reconciler owner:** not assigned yet. Suggest Tanmay, since it shares the TWS client with the Listener.

## 2. Architecture overview
<img width="3162" height="2018" alt="Hybrid_Orchestration_E2E_Flow1-Detailed — flow, suggestions, open questions drawio" src="https://github.com/user-attachments/assets/c32bee3f-a298-418e-8ec5-8a32e5509cca" />

One Data Available event in, one TWS job stream run, one Data Available event out, all tied together by one `correlation_id`. All three workloads run in namespace `hybrid-orch-<env>` on KPS and only call **out**; TWS never calls into KPS.

&#91;embedded content: Hybrid Orchestration on KPS · components and calls\]

All calls go out from KPS. The Reconciler pulls run status from TWS; nothing on-prem calls into the cluster.

| Component | Runs as | Owner | Reads | Writes / calls |
| --- | --- | --- | --- | --- |
| Event Listener | KPS Deployment, 2 replicas in prod | Tanmay | DESP inbound topic, `tws_event_config.yaml` | Listener state + audit, dead letter; `sendevent` to TWS EPS |
| Correlation Reconciler | KPS CronJob, every 2 min | TBD (suggest Tanmay) | Listener state, TWS REST plan (:31116) | `TWS_EVENT_RUN_LINK` |
| Event Publisher | KPS CronJob, every 2 min | Akshatha | Run link, listener state, config, DISC API | Publisher state + audit, dead letter; DESP outbound topic |
| TWS (HCL Workload Automation) | On-prem | Sravanthi / John | Custom event `HO_DATA_AVAILABLE` | EPS rule submits job stream, alias `HO_<CorrShort>`, variable `CORR_ID` |
| Informatica PowerCenter | On-prem | Chandrakant | Generic KSH, `pmcmd -rin HO_<CorrShort>` | Workflow run; stats to DISC. Workflows are **not** changed |
| Databricks Unity Catalog | Azure | ETL Platform | — | `${catalog}.hybrid_orch` Delta tables (section 9) |

**Platform standards:** GitHub Actions → Snyk → JFrog → Harness NextGen → Helm on `kps-base-helmchart` · secrets from Akeyless via External Secrets Operator · JSON logs to stdout → Dynatrace · Kyverno policies (requests/limits, read-only root FS, non-root, no `latest` tag, CronJob TTL).

## 3. Correlation ID and other IDs

One `correlation_id` (UUID v4) per delivery cycle. It is set once by the Listener and **never re-minted**, not even on retry. Every table row and every log line carries it.

| ID | Format | Created by | When | Purpose |
| --- | --- | --- | --- | --- |
| `correlation_id` | UUID v4 | Listener | Reused from inbound `correlationid` if present, else minted | Ties inbound event → TWS run → Informatica run → outbound event |
| `corr_short` | First 8 hex chars of `correlation_id`, upper case | Listener | With `correlation_id` | Goes into the TWS alias `HO_<corr_short>` and `pmcmd -rin` |
| `event_id` | From inbound envelope `id` | Producer | — | Identifies the inbound message; becomes the outbound `causationid` |
| `idempotency_key` | SHA-256 hex of the config dedupe fields, joined with `\|` | Listener | On config match | Dedupe; primary key of listener state |
| `trigger_id` | UUID v4 | Listener | Per trigger attempt | Identifies one `sendevent` attempt |
| `jobstream_instance_id` | From TWS plan | TWS | When TWS submits | Links to the exact TWS run |
| `publish_id` | **UUID v5** of `correlation_id` + outbound dataset (S7) | Publisher | On claim | Outbound event `id`; the same on a resend, so duplicates are harmless |

**How the ID crosses into TWS:** `sendevent HO_DATA_AVAILABLE` carries `CorrId`, `CorrShort`, `DatasetId`, `BusinessDate`. The EPS rule submits the job stream with alias `HO_<CorrShort>` and variable `CORR_ID` = full UUID. The KSH passes `-rin HO_<CorrShort>` to `pmcmd` (Q12), so the same tag appears in DISC.

**How it comes back:** `sendevent` returns an ACK only, with no run ID. The Reconciler finds the run by job stream + alias, confirms `CORR_ID` matches the full UUID (S5), and writes the link to `TWS_EVENT_RUN_LINK`.

**Collision check:** two IDs can share the same 8-character `corr_short`. Always confirm the full `CORR_ID` before linking; if TWS can't return the variable, treat two alias matches as `AMBIGUOUS`.

## 4. Listener build spec 

The Listener decides whether an event should start a TWS job and, if so, starts it **exactly once** with the correlation ID attached. It writes state **before** it calls TWS and checkpoints the Event Hub offset **last**.

**Processing steps, per event**

1. **Receive.** Consume from the DESP inbound topic. Append `RECEIVED` to `TWS_EVENT_LISTENER_AUDIT`.
2. **Validate** the envelope against the JSON Schema (section 10). Fail → `INVALID` + dead letter, checkpoint, stop.
3. **Match config** on `domain` + `dataset_name` in `tws_event_config.yaml`. No match → `IGNORED` in audit, checkpoint, stop.
4. **Dedupe.** Build `idempotency_key` from the config's `dedupe_fields`. Conditional MERGE into `TWS_EVENT_LISTENER_STATE` (S2): 0 rows inserted → `DUPLICATE` in audit, checkpoint, stop.
5. **Assign the ID.** Reuse inbound `correlationid` (body or header, S4) or mint UUID v4. Derive `corr_short`.
6. **Record intent.** The inserted state row has status `TRIGGERING`, `trigger_id`, target job stream and workstation.
7. **Trigger TWS.** Run `sendevent HO_DATA_AVAILABLE` with `CorrId`, `CorrShort`, `DatasetId`, `BusinessDate`.
8. **Mark done.** Update state to `TRIGGERED`, append audit, **then** checkpoint the offset (S3).

**Retries**

- Timeout or 5xx from TWS: retry with exponential backoff (e.g. 3 attempts: 2 s, 8 s, 30 s), same `correlation_id` and `trigger_id`, `attempt_count` +1.
- Still failing, or a 4xx: `TRIGGER_FAILED`, write dead letter, alert, checkpoint.
- On restart, rows left in `TRIGGERING` older than 10 min are retried by the Listener (sweep at start-up and every 5 min). *Proposed — Amit to validate.*

**Concurrency (2 replicas in prod)**

- Event Hub partition key = `dataset_id` (S2, ask DESP Q9), so one dataset is handled by one consumer at a time.
- The conditional MERGE on `idempotency_key` is the second guard. Delta has no unique constraint, so check the rows-inserted count.

**Pseudocode**

```python
def handle(msg, cfg, store, tws):
    ctx = RunContext.from_message(msg)            # event_id, raw headers
    store.audit_listener(ctx, "RECEIVED")
    evt = validate_envelope(msg)                  # raises InvalidEvent
    rule = cfg.match(evt.domain, evt.dataset_name)
    if rule is None:
        return store.audit_listener(ctx, "IGNORED")
    key = ids.idempotency_key(evt, rule.dedupe_fields)
    ctx = ctx.with_correlation(ids.correlation_id(evt), key)
    if not store.claim_listener_state(ctx, rule, status="TRIGGERING"):
        return store.audit_listener(ctx, "DUPLICATE")
    with_retry(lambda: tws.sendevent("HO_DATA_AVAILABLE", ctx.tws_params()))
    store.update_listener_state(ctx, status="TRIGGERED")
    store.audit_listener(ctx, "TRIGGERED")
    # caller checkpoints the offset only after handle() returns
```

**Done when:** a test event on TWS-TEST produces one job stream with alias `HO_<corr_short>`, one `TRIGGERED` state row and audit rows for each step; sending the same event twice produces `DUPLICATE` and no second job stream.

## 5. Correlation Reconciler build spec

The Reconciler fixes the broken link: every 2 minutes it finds the TWS run each triggered event started and tracks it to a final status in `TWS_EVENT_RUN_LINK`. It only reads from TWS, so it is safe to run again at any time.

**Runtime:** KPS CronJob `*/2 * * * *`, `concurrencyPolicy: Forbid`, one pod per run, proposed 100m/500m CPU and 256Mi/512Mi memory. CronJob vs one always-on pod is still open (Q7).

**One cycle**

1. **Pick work** (look back 24 h):
   - listener state `TRIGGERED` with no run link row, and
   - run link rows in `MATCHED` or `RUNNING`.
2. **Query TWS REST** current plan for `target_job_stream` on `tws_workstation` with alias `HO_<corr_short>` (Q3).
3. **Match** (S5), best method first:
   - one instance with that alias and `CORR_ID` = full UUID → `MATCHED`, `match_method = ALIAS`
   - no alias support: one unlinked instance of that job stream submitted between `triggered_ts` and `triggered_ts + 15 min` → `MATCHED`, `match_method = TIME_WINDOW`
   - two or more candidates → `AMBIGUOUS`, alert, never published until someone resolves it
   - none yet → wait; after 30 min → `NOT_FOUND`, alert
4. **MERGE** the run link on `correlation_id`: instance ID, job numbers, `publish_on_job_status`, status.
5. **Follow** unfinished rows on later cycles: `MATCHED → RUNNING → SUCC / ABEND / CANCEL`. An operator rerun after an abend is the same instance, so the row simply moves back to `RUNNING`, then `SUCC`.

**Status is driven by the `publish_on` job**, not the whole job stream. The config names which job proves the data is really there.

**Pseudocode**

```python
def reconcile_cycle(store, tws, now):
    for item in store.pending_links(lookback_hours=24):
        ctx = RunContext.from_state(item)
        hits = tws.plan_query(item.target_job_stream, item.tws_workstation,
                              alias=f"HO_{item.corr_short}")
        hits = [h for h in hits if h.variables.get("CORR_ID") in (None, item.correlation_id)]
        if not hits and cfg.time_window_fallback:
            hits = tws.plan_query_window(item.target_job_stream, item.triggered_ts,
                                         item.triggered_ts + timedelta(minutes=15))
            hits = store.exclude_linked(hits)
        if len(hits) == 1:
            store.merge_run_link(ctx, hits[0], method="ALIAS" if hits[0].alias else "TIME_WINDOW")
        elif len(hits) > 1:
            store.merge_run_link(ctx, None, status="AMBIGUOUS"); alert(ctx)
        elif now - item.triggered_ts > timedelta(minutes=30):
            store.merge_run_link(ctx, None, status="NOT_FOUND"); alert(ctx)
```

**Done when:** for a test event, the run link row reaches `SUCC` within two cycles of the job finishing, and a deliberately abended then rerun job ends as `SUCC` on the same row.

The 15-minute window and 30-minute timeout are starting values; tune them with Sravanthi from real submit delays.

## 6. Publisher build spec 

The Publisher publishes **only** after the `publish_on` job is confirmed `SUCC` and the publish rules pass, and it publishes each delivery **once**. It never publishes on a `sendevent` ACK.

**Runtime:** KPS CronJob `*/2 * * * *`, `concurrencyPolicy: Forbid`, proposed 250m/1 CPU and 512Mi/1Gi memory.

**One cycle**

1. **Pick runs** from `TWS_EVENT_RUN_LINK` where `publish_on_job_status = 'SUCC'`, status is not `AMBIGUOUS`, and no `PUBLISHED` row exists in `TWS_EVENT_PUBLISHER_STATE`.
2. **Claim.** Conditional MERGE into publisher state with status `PUBLISHING` and the deterministic `publish_id` (S7). 0 rows inserted → skip.
3. **Gather:**
   - outbound section of `tws_event_config.yaml` (event type, publish rules, `attribute_map`)
   - listener state (inbound `event_id`, `dataset_id`, `business_date`, `correlation_id`)
   - run link (job stream instance, job numbers)
   - DISC API run stats by workflow + `HO_<corr_short>`, or by time window if `-rin` isn't possible (Q11, Q12)
4. **Check publish rules** (S8), e.g. `rows_written > 0`, `reject_pct < 1`, `workflow_status = Succeeded`.
   - pass → step 5
   - fail → `HELD`, alert, nothing published
   - DISC not ready → stay `PUBLISHING`, retry next cycle; after 60 min → `HELD`, alert
5. **Build** the outbound envelope (section 10): same `correlationid`, `causationid` = inbound `event_id`, `id` = `publish_id`, `data` from `attribute_map`.
6. **Publish** to the DESP outbound topic. Retry on timeout/5xx with the **same** `id`.
7. **Mark done.** Update state to `PUBLISHED` with `published_ts`; append to `TWS_EVENT_PUBLISHER_AUDIT`.

**Why duplicates are harmless:** Event Hubs gives at-least-once delivery. If the pod dies after DESP accepts the event but before `PUBLISHED` is written, the next cycle resends it with the same `id`. Consumers dedupe on `id` (ask DESP, Q10).

**Pseudocode**

```python
def publish_cycle(store, disc, desp, cfg):
    for link in store.ready_to_publish():
        ctx = RunContext.from_link(link)
        rule = cfg.outbound_for(link.dataset_id)
        pid = ids.publish_id(link.correlation_id, rule.outbound_dataset)   # UUID v5
        if not store.claim_publisher_state(ctx, pid):
            continue
        stats = disc.run_stats(rule.workflow, run_instance=f"HO_{link.corr_short}")
        if stats is None:
            store.touch_publisher_state(ctx)          # retry next cycle, HELD after 60 min
            continue
        failed = rules.evaluate(rule.publish_rules, stats)
        if failed:
            store.update_publisher_state(ctx, "HELD", reason=failed); alert(ctx); continue
        event = envelope.build_outbound(ctx, rule, stats, event_id=pid)
        with_retry(lambda: desp.publish(event, partition_key=link.dataset_id))
        store.update_publisher_state(ctx, "PUBLISHED")
        store.audit_publisher(ctx, "PUBLISHED", event_id=pid)
```

**Done when:** a successful test run produces exactly one outbound event whose `correlationid` matches the inbound one; a run with 0 rows ends `HELD` with no event; killing the pod mid-publish and rerunning sends the same `id` again, never a new one.

The 60-minute DISC wait is a starting value; confirm with the DISC team how long run stats take to appear.

## 7. Failure handling

Every failure ends as either an automatic retry or a status plus an alert; no event is lost and nothing is published unless the load really succeeded.

| What fails | What the code does | Status written | Who acts |
| --- | --- | --- | --- |
| Listener pod crashes mid-event | Offset not checkpointed → DESP redelivers; dedupe stops a second trigger | `DUPLICATE` on redelivery | Nobody |
| Event fails schema | Dead letter, checkpoint | `INVALID` | Producer team |
| `sendevent` times out / 5xx | Backoff retry, same IDs | `TRIGGERING` → `TRIGGERED` | Nobody |
| `sendevent` keeps failing / 4xx | Dead letter, alert | `TRIGGER_FAILED` | TWS team / us |
| Event rule doesn't fire | Reconciler times out after 30 min | `NOT_FOUND` | TWS team |
| Two candidate runs | No auto link, no publish | `AMBIGUOUS` | Us, resolve manually |
| TWS job abends | Run link tracks it; nothing published | `ABEND` | Ops rerun; row moves to `SUCC` by itself |
| Reconciler or Publisher pod fails | Next cycle picks up (all writes are MERGE) | unchanged | Nobody unless repeated |
| DISC down or stats late | Retry each cycle, `HELD` after 60 min | `PUBLISHING` → `HELD` | DISC team |
| Publish rules fail | No publish, alert | `HELD` | Data owner |
| DESP down on publish | Retry next cycle with the same `id` | `PUBLISHING` | DESP team |

**Retry rule (all workloads):** retry only on timeouts and 5xx, with exponential backoff; a 4xx goes straight to `*_FAILED` + dead letter. Never re-mint `correlation_id`.

**Alerts:** Dynatrace on log events (`level=ERROR` with `status` in `INVALID`, `TRIGGER_FAILED`, `NOT_FOUND`, `AMBIGUOUS`, `HELD`), routed to ServiceNow/xMatters, not email.

## 8. Coding rules and project structure

Core logic stays plain Python with **no KPS-specific code**, so the same packages run anywhere; only the thin `entrypoints` know they're on KPS.

**Rules**

- All IDs come from `common/ids.py` only. Never call `uuid4()` anywhere else.
- Pass a `RunContext` (`correlation_id`, `event_id`, `dataset_id`, `trigger_id` / `publish_id`) everywhere and put it on every log line.
- Pattern: **write state → call the external system → update state + append audit**. State tables use MERGE on key; audit tables are append-only.
- Retry on timeouts/5xx only, exponential backoff; 4xx → `*_FAILED` + dead letter.
- UTC everywhere (`datetime.now(timezone.utc)`), stored as `TIMESTAMP`.
- No hard-coded environment values. Endpoints come from `app_config_<env>.yaml`; secrets from Akeyless via ESO (env vars or mounted files, still to confirm with KPS). **No secrets in YAML or code.**
- Never log secrets, tokens or full payloads. Log IDs, statuses and counts.
- **Non-prod uses non-prod sources only:** TWS-TEST, non-prod DISC/DESP, no prod PowerCenter repo, no `/gpfsprod` (App Security Policy 4.8.2).
- Every external client (TWS, DISC, DESP, Delta store) sits behind an interface with a mock, so open questions never block the build.

**Suggested repo layout**

```text
hybrid-orchestration/
  src/hybrid_orch/
    common/      ids.py  context.py  logging.py  retry.py  config.py  schemas/
    clients/     tws.py  disc.py  desp.py  delta_store.py   (+ mocks/)
    listener/    handler.py
    reconciler/  cycle.py
    publisher/   cycle.py  rules.py  envelope.py
    entrypoints/ listener_main.py  reconciler_main.py  publisher_main.py
  config/        tws_event_config.yaml  app_config_dev.yaml ...
  ddl/           hybrid_orch_tables.sql
  tests/         unit/  contract/  e2e/
  helm/          values-dev.yaml ...
  Dockerfile     (one image, three entrypoints)
```

**Log line shape (JSON to stdout)**

```json
{"ts":"2026-11-02T07:41:03Z","level":"INFO","component":"listener","correlation_id":"b1e7d9c4-...","event_id":"8f3c2a1e-...","dataset_id":"sales_daily","status":"TRIGGERED","attempt":1,"msg":"sendevent accepted"}
```

**Tests expected:** unit tests for `ids`, config matching, dedupe and publish rules; contract tests against the JSON Schemas; one end-to-end test on mocks (event in → event out with the same `correlation_id`).

## 9. Reference: table structures

Six Delta tables in `${catalog}.hybrid_orch`; every row carries `correlation_id` (except dead-letter rows rejected before an ID exists). Names follow the `TWS_EVENT_*` set, **pending Chandra's final call (Q19)**. The two `*_STATE` tables are new (S1).

| Table | Write style | Key | Written by | Read by |
| --- | --- | --- | --- | --- |
| `tws_event_listener_state` | MERGE (current status) | `idempotency_key` | Listener | Reconciler, Publisher |
| `tws_event_listener_audit` | Append-only | `audit_id` | Listener | Support / tracing |
| `tws_event_run_link` | MERGE | `correlation_id` | Reconciler | Publisher |
| `tws_event_publisher_state` | MERGE | `correlation_id` + `outbound_dataset` | Publisher | Support / tracing |
| `tws_event_publisher_audit` | Append-only | `audit_id` | Publisher | Support / tracing |
| `tws_event_dead_letter` | Append-only | `dead_letter_id` | All three | Support / replay |

### DDL

```sql
-- 1. Listener state: one row per delivery (dedupe key)
CREATE TABLE IF NOT EXISTS ${catalog}.hybrid_orch.tws_event_listener_state (
  idempotency_key     STRING    NOT NULL COMMENT 'SHA-256 of config dedupe fields',
  correlation_id      STRING    NOT NULL,
  corr_short          STRING    NOT NULL COMMENT 'first 8 hex chars, upper case',
  event_id            STRING    NOT NULL COMMENT 'inbound envelope id',
  inbound_correlation STRING             COMMENT 'correlationid as received, if any',
  domain              STRING    NOT NULL,
  dataset_id          STRING    NOT NULL,
  business_date       DATE      NOT NULL,
  target_job_stream   STRING    NOT NULL,
  tws_workstation     STRING    NOT NULL,
  trigger_id          STRING,
  status              STRING    NOT NULL COMMENT 'TRIGGERING|TRIGGERED|TRIGGER_FAILED',
  attempt_count       INT       NOT NULL,
  last_error          STRING,
  received_ts         TIMESTAMP NOT NULL,
  triggered_ts        TIMESTAMP,
  created_ts          TIMESTAMP NOT NULL,
  updated_ts          TIMESTAMP NOT NULL
) USING DELTA;

-- 2. Listener audit: one row per step
CREATE TABLE IF NOT EXISTS ${catalog}.hybrid_orch.tws_event_listener_audit (
  audit_id         STRING    NOT NULL,
  correlation_id   STRING             COMMENT 'null for INVALID/IGNORED before ID assignment',
  event_id         STRING    NOT NULL,
  idempotency_key  STRING,
  dataset_id       STRING,
  business_date    DATE,
  step_status      STRING    NOT NULL COMMENT 'RECEIVED|INVALID|IGNORED|DUPLICATE|TRIGGERING|TRIGGERED|TRIGGER_FAILED',
  detail           STRING,
  attempt          INT,
  source_partition STRING,
  source_offset    BIGINT,
  event_ts         TIMESTAMP NOT NULL,
  event_date       DATE GENERATED ALWAYS AS (CAST(event_ts AS DATE))
) USING DELTA PARTITIONED BY (event_date);

-- 3. Run link: inbound event -> TWS run
CREATE TABLE IF NOT EXISTS ${catalog}.hybrid_orch.tws_event_run_link (
  correlation_id         STRING    NOT NULL,
  idempotency_key        STRING    NOT NULL,
  trigger_id             STRING    NOT NULL,
  dataset_id             STRING    NOT NULL,
  business_date          DATE      NOT NULL,
  job_stream_name        STRING    NOT NULL,
  tws_workstation        STRING    NOT NULL,
  alias                  STRING             COMMENT 'HO_<corr_short>',
  jobstream_instance_id  STRING,
  sched_time             TIMESTAMP,
  event_rule_name        STRING             COMMENT 'expected rule from config (C2)',
  publish_on_job         STRING    NOT NULL,
  publish_on_job_number  STRING,
  publish_on_job_status  STRING,
  job_statuses           STRING             COMMENT 'JSON: [{job, number, status}]',
  informatica_run_name   STRING             COMMENT 'pmcmd -rin value, if used',
  match_method           STRING             COMMENT 'ALIAS|TIME_WINDOW',
  status                 STRING    NOT NULL COMMENT 'MATCHED|RUNNING|SUCC|ABEND|CANCEL|AMBIGUOUS|NOT_FOUND',
  first_seen_ts          TIMESTAMP,
  last_checked_ts        TIMESTAMP NOT NULL,
  completed_ts           TIMESTAMP,
  created_ts             TIMESTAMP NOT NULL,
  updated_ts             TIMESTAMP NOT NULL
) USING DELTA;

-- 4. Publisher state: one row per outbound delivery
CREATE TABLE IF NOT EXISTS ${catalog}.hybrid_orch.tws_event_publisher_state (
  correlation_id         STRING    NOT NULL,
  outbound_dataset       STRING    NOT NULL,
  publish_id             STRING    NOT NULL COMMENT 'UUID v5, = outbound event id',
  causation_id           STRING    NOT NULL COMMENT 'inbound event_id',
  jobstream_instance_id  STRING,
  status                 STRING    NOT NULL COMMENT 'PUBLISHING|PUBLISHED|HELD|PUBLISH_FAILED',
  rows_read              BIGINT,
  rows_written           BIGINT,
  rows_rejected          BIGINT,
  rule_result            STRING             COMMENT 'JSON: failed rules, if any',
  attempt_count          INT       NOT NULL,
  last_error             STRING,
  claimed_ts             TIMESTAMP NOT NULL,
  published_ts           TIMESTAMP,
  created_ts             TIMESTAMP NOT NULL,
  updated_ts             TIMESTAMP NOT NULL
) USING DELTA;

-- 5. Publisher audit: one row per step
CREATE TABLE IF NOT EXISTS ${catalog}.hybrid_orch.tws_event_publisher_audit (
  audit_id          STRING    NOT NULL,
  correlation_id    STRING    NOT NULL,
  publish_id        STRING    NOT NULL,
  outbound_dataset  STRING    NOT NULL,
  step_status       STRING    NOT NULL COMMENT 'CLAIMED|STATS_FETCHED|HELD|PUBLISHED|PUBLISH_FAILED',
  detail            STRING,
  attempt           INT,
  event_ts          TIMESTAMP NOT NULL,
  event_date        DATE GENERATED ALWAYS AS (CAST(event_ts AS DATE))
) USING DELTA PARTITIONED BY (event_date);

-- 6. Dead letter: anything rejected or failed for good
CREATE TABLE IF NOT EXISTS ${catalog}.hybrid_orch.tws_event_dead_letter (
  dead_letter_id    STRING    NOT NULL,
  component         STRING    NOT NULL COMMENT 'listener|reconciler|publisher',
  correlation_id    STRING,
  event_id          STRING,
  dataset_id        STRING,
  reason_code       STRING    NOT NULL COMMENT 'INVALID|TRIGGER_FAILED|PUBLISH_FAILED|...',
  error_detail      STRING,
  source_partition  STRING,
  source_offset     BIGINT,
  payload_sha256    STRING,
  payload_json      STRING             COMMENT 'only if data owner confirms no PI (Q16)',
  resolved          BOOLEAN   NOT NULL,
  event_ts          TIMESTAMP NOT NULL
) USING DELTA;
```

### Key MERGE patterns

Claim a delivery (Listener step 4). If `num_inserted_rows` = 0, it's a duplicate.

```sql
MERGE INTO ${catalog}.hybrid_orch.tws_event_listener_state t
USING (SELECT :idempotency_key AS idempotency_key, ...) s
ON t.idempotency_key = s.idempotency_key
WHEN NOT MATCHED THEN INSERT *;
```

Update a run link (Reconciler step 4), never moving a finished row backwards except `ABEND → RUNNING` on a rerun:

```sql
MERGE INTO ${catalog}.hybrid_orch.tws_event_run_link t
USING (SELECT :correlation_id AS correlation_id, ...) s
ON t.correlation_id = s.correlation_id
WHEN MATCHED AND t.status NOT IN ('SUCC','CANCEL') THEN UPDATE SET
  status = s.status, publish_on_job_status = s.publish_on_job_status,
  job_statuses = s.job_statuses, last_checked_ts = s.last_checked_ts, updated_ts = s.updated_ts
WHEN NOT MATCHED THEN INSERT *;
```

### Status values

| Table | Status | Meaning | Final? |
| --- | --- | --- | --- |
| Listener state | `TRIGGERING` | Intent written, `sendevent` in progress | No |
| Listener state | `TRIGGERED` | TWS accepted the event (ACK only) | Yes (for the Listener) |
| Listener state | `TRIGGER_FAILED` | Gave up after retries or 4xx | Yes |
| Run link | `MATCHED` | Run found, not started | No |
| Run link | `RUNNING` | `publish_on` job running | No |
| Run link | `SUCC` | `publish_on` job succeeded | Yes |
| Run link | `ABEND` | Job failed; can return to `RUNNING` on rerun | No |
| Run link | `CANCEL` | Cancelled in TWS | Yes |
| Run link | `AMBIGUOUS` | More than one candidate run | Until resolved |
| Run link | `NOT_FOUND` | No run after 30 min | Until resolved |
| Publisher state | `PUBLISHING` | Claimed, gathering stats / publishing | No |
| Publisher state | `PUBLISHED` | Outbound event accepted by DESP | Yes |
| Publisher state | `HELD` | Rules failed or DISC timed out | Until released |
| Publisher state | `PUBLISH_FAILED` | 4xx from DESP | Yes |

### Trace one delivery ("where's my data?")

```sql
SELECT l.dataset_id, l.business_date, l.correlation_id,
       l.status AS listener_status,
       r.jobstream_instance_id, r.status AS tws_status, r.match_method,
       p.status AS publish_status, p.publish_id, p.published_ts
FROM   ${catalog}.hybrid_orch.tws_event_listener_state l
LEFT JOIN ${catalog}.hybrid_orch.tws_event_run_link r USING (correlation_id)
LEFT JOIN ${catalog}.hybrid_orch.tws_event_publisher_state p USING (correlation_id)
WHERE  l.dataset_id = :dataset_id AND l.business_date = :business_date;
```

The first status that isn't final shows where the delivery stopped. Retention: 400 days proposed, pending data owner (Q16).

## 10. Reference: config files and event formats

Two YAML files drive the behaviour, both validated against a JSON Schema in CI and at start-up. Field names below are the proposal; the DESP envelope is illustrative until DESP shares the real schema (Q8).

### `tws_event_config.yaml` (one entry per dataset)

```yaml
version: 1
datasets:
  - dataset_id: sales_daily
    enabled: true
    owner: akshatha                    # who maintains this entry
    inbound:
      event_type: kroger.data.available
      match: { domain: sales, dataset_name: sales_daily }
      dedupe_fields: [dataset_name, business_date]
      required_fields: [dataset_name, business_date, status]
    tws:
      event_name: HO_DATA_AVAILABLE
      workstation: <TWS workstation>
      job_stream: EDW_SALES_DAILY
      event_rule: HO_SALES_DAILY_RULE   # expected rule (C2)
      publish_on_job: WF_LOAD_SALES     # job whose SUCC means data is ready
      time_window_fallback: true
    informatica:
      folder: <PowerCenter folder>
      workflow: wf_load_sales
      run_instance_prefix: HO_          # pmcmd -rin HO_<corr_short>
    outbound:
      dataset: edw_sales_daily
      event_type: kroger.data.available
      publish_rules:
        - { metric: rows_written,    op: ">",  value: 0 }
        - { metric: reject_pct,      op: "<",  value: 1.0 }
        - { metric: workflow_status, op: "==", value: Succeeded }
      attribute_map:
        domain: { const: sales }
        dataset_name: { const: edw_sales_daily }
        business_date: { from: inbound.business_date }
        record_count: { from: disc.rows_written }
        location: { const: <catalog.schema.table> }
```

### `app_config_<env>.yaml` (endpoints and secret names only)

```yaml
env: dev
desp:
  protocol: amqp            # or kafka (Q9)
  namespace: <eventhub namespace>
  inbound_topic: <topic>
  outbound_topic: <topic>
  consumer_group: hybrid-orch-dev
tws:
  rest_base_url: https://<tws-test host>:31116
  eps_host: <host>
  eps_port: <port>          # Q1
  credentials_secret: tws-svc-account      # name only, value from Akeyless
disc:
  base_url: <url>
  auth_secret: disc-sp
delta:
  catalog: <non-prod catalog>
  schema: hybrid_orch
  sql_warehouse_id: <id>    # Q14
schedules:
  reconciler_lookback_hours: 24
  match_window_minutes: 15
  not_found_after_minutes: 30
  held_after_minutes: 60
```

### `sendevent` parameters (Listener → TWS)

| Parameter | Example | Used by the EPS rule for |
| --- | --- | --- |
| `CorrId` | `b1e7d9c4-2f3a-4e8b-a6c1-7d5e9f0a1b23` | Variable `CORR_ID` |
| `CorrShort` | `B1E7D9C4` | Alias `HO_B1E7D9C4` |
| `DatasetId` | `sales_daily` | Choosing the job stream |
| `BusinessDate` | `2026-11-01` | Job stream variable / date |

### Inbound Data Available event (illustrative)

```json
{
  "specversion": "1.0",
  "id": "8f3c2a1e-5b7d-4c9a-9e21-0d4f6a7b8c90",
  "type": "kroger.data.available",
  "source": "databricks://etl-general/jobs/123456",
  "time": "2026-11-02T06:14:22Z",
  "correlationid": "b1e7d9c4-2f3a-4e8b-a6c1-7d5e9f0a1b23",
  "data": {
    "domain": "sales",
    "dataset_name": "sales_daily",
    "business_date": "2026-11-01",
    "status": "SUCCESS",
    "record_count": 1843921
  }
}
```

### Outbound Data Available event (illustrative)

```json
{
  "specversion": "1.0",
  "id": "<publish_id, UUID v5>",
  "type": "kroger.data.available",
  "source": "hybrid-orch://EDW_SALES_DAILY",
  "time": "2026-11-02T07:41:03Z",
  "correlationid": "b1e7d9c4-2f3a-4e8b-a6c1-7d5e9f0a1b23",
  "causationid": "8f3c2a1e-5b7d-4c9a-9e21-0d4f6a7b8c90",
  "data": {
    "domain": "sales",
    "dataset_name": "edw_sales_daily",
    "business_date": "2026-11-01",
    "status": "SUCCESS",
    "record_count": 1843115,
    "producer": { "platform": "informatica", "workflow": "wf_load_sales", "run_instance": "HO_B1E7D9C4" }
  }
}
```

If DESP's schema has no `correlationid` / `causationid` fields, send them as message headers (S4).

## 11. Open questions, suggestions and what to mock

None of the open questions should stop coding: each one has a mock or a config flag to build against until the answer arrives. Numbers match the diagram.

### Open questions

| # | Question | Ask | Affects | Build against meanwhile |
| --- | --- | --- | --- | --- |
| Q1 | Can a KPS container run `sendevent` and reach the EPS port? Which port? | TWS | Listener | `TwsClient` interface + mock; real client behind a flag |
| Q2 | Can the event rule set alias `HO_<id>` and `CORR_ID`? Max alias length? | TWS | Reconciler | Flag `time_window_fallback` |
| Q3 | Can REST find a run by job stream + alias and return ID, jobs, status? | TWS | Reconciler | Mock plan responses: 0, 1, 2 hits |
| Q4 | Can the plan show which event rule submitted the job stream? | TWS | Run link `event_rule_name` | Fill from config |
| Q5 | Cluster, namespace per environment, onboarding lead time? | KPS | Deployment | Run locally with Podman/Docker |
| Q6 | Can pods reach on-prem TWS and DISC? Fixed outbound IP? | KPS | All | Mocks |
| Q7 | CronJob every 2 min, or one always-on pod? | KPS | Reconciler, Publisher | Keep `cycle()` separate from the scheduler |
| Q8 | DESP schema and samples; `correlationid` field or header? | DESP | Listener, Publisher | Illustrative envelope (section 10) + JSON Schema |
| Q9 | Kafka or AMQP; topic; consumer group? | DESP | Listener | `EventSource` interface; Event Hubs emulator |
| Q10 | Do consumers dedupe on event `id`? | DESP | Publisher | Deterministic `publish_id` anyway |
| Q11 | DISC API for run stats; lookup by run name or time window; SP access? | DISC | Publisher | `DiscClient` mock with sample stats |
| Q12 | Can the KSH pass `pmcmd -rin HO_<id>`? Non-prod repo? | Chandrakant | Publisher | Time-window DISC lookup |
| Q13 | Firewall process and lead time KPS → TWS, DISC? | NetSec | All | — |
| Q14 | SQL Warehouse + SP per environment; best way to write Delta from a pod? | Databricks / ETL Platform | All | `DeltaStore` interface; local Delta or SQLite mock |
| Q15 | Latency SLA, overnight support, year-end freeze dates? | Gus | Schedules, go-live | 2-min cycles |
| Q16 | Who approves publish rules; 400-day retention; PI in events? | Data owners | Publisher, dead letter | Sample rules; no `payload_json` stored |
| Q17 | Jobs that need several datasets ready (fan-in)? | Design | Listener, TWS rules | Out of scope for first datasets |
| Q18 | How to replay a past business date or resend an event safely? | Design | Listener, Publisher | Not built yet |
| Q19 | Final table names: `TWS_EVENT_*` or `orch_*`? | Chandra | DDL | `TWS_EVENT_*` (section 9) |

### Suggestions in this design (proposed, not yet decided)

| # | Suggestion | Where |
| --- | --- | --- |
| S1 | Split state (MERGE) and audit (append-only) tables | Section 9 |
| S2 | Partition key = `dataset_id` + conditional MERGE to stop double triggers with 2 replicas | Listener |
| S3 | Write intent before `sendevent`; checkpoint the offset last | Listener |
| S4 | Carry `correlationid` as a header if the schema has no field; hide Kafka/AMQP behind one interface | Listener, Publisher |
| S5 | Confirm full `CORR_ID`; time-window fallback; `AMBIGUOUS` and `NOT_FOUND` handling | Reconciler |
| S6 | DISC API as the only run-stats source (no /gpfs, no Oracle repo, no prod data in non-prod) | Publisher |
| S7 | Deterministic outbound `id` (UUID v5) | Publisher |
| S8 | Failed publish rules → `HELD`; never publish on ACK | Publisher |
| S9 | Fallback if `sendevent` can't run from a container: TWS REST submit (design change, needs Doug) | Listener |

## 12. Review checklist for Amit

Please comment directly on the section, and tick each item once you're satisfied or have left a comment on it.

**Correlation ID**

- [ ] `correlation_id` is set once and never re-minted on retry, restart or redelivery (sections 3, 4)
- [ ] Reusing the inbound `correlationid` is safe, and minting when it's missing is the right default
- [ ] The 8-character `corr_short` plus the full `CORR_ID` check is enough to avoid wrong links
- [ ] The time-window fallback and `AMBIGUOUS` handling are acceptable if TWS can't carry the alias

**Dedupe and exactly-once behaviour**

- [ ] The `idempotency_key` fields per dataset are right (Akshatha is confirming with data owners)
- [ ] The conditional MERGE + partition key prevents double triggers with 2 listener replicas
- [ ] Rows stuck in `TRIGGERING` are recovered safely (section 4, retries)
- [ ] The deterministic `publish_id` makes resends harmless

**Tables**

- [ ] State vs audit split (S1) and the keys in section 9
- [ ] Column list is complete for tracing and support; nothing stores secrets or full payloads
- [ ] Status values and which are final
- [ ] Partitioning and retention (400 days proposed)
- [ ] Small, frequent writes from pods won't hit Delta concurrency problems (Q14)

**Config and events**

- [ ] `tws_event_config.yaml` holds everything needed per dataset, and ownership of each entry is clear
- [ ] Envelope fields and the header fallback (S4)

**Overall**

- [ ] Anything in sections 4–7 that could lose an event or publish twice



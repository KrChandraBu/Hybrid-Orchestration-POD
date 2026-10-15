# ADR 0001: Correlation ID for the TWS adapter via event config and Delta audit tables

**Status**: Proposed

**Date**: 2026-10-13

**Authors**: Amit Thorat, Chandra Buchha — Kroger Hybrid Workflow Orchestration Squad

**Reviewers**: Doug Heitkamp

**Decision forum**: Hybrid Workflow Orchestration task update, 2026-10-09

---

## Context

Hybrid workflow orchestration connects on-prem TWS (HCL Workload Automation) and Databricks through **data availability events** instead of job-status coupling. Every data availability event carries a **correlation ID** so that a single hybrid orchestration run can be traced end to end across both systems.

The TWS adapter flow is:

1. A producer publishes a data availability event (with correlation ID) to the streaming platform — Azure Event Hub today, DESP or another platform in future.
2. The **TWS Event Listener** (Python) consumes the event, builds a TWS custom event, and submits it with the native `sendevent` command to the TWS Event Processing Server.
3. The Event Processing Server evaluates the configured **event rules** and runs the matching **action** — submit a job / job stream, or reply to a prompt.
4. The job runs an Informatica PowerCenter workflow.
5. The **TWS Event Publisher** (Python) publishes a new data availability event for the dataset that workflow produced.

The problem is step 2–3. `sendevent` is **asynchronous**. A successful response only confirms the event was *submitted*; it says nothing about which rule evaluated it, which action fired, or which job run resulted. Event submission and job execution happen at different times, so there is no durable link between them. As a result:

- **Event-to-job traceability** is missing — we cannot say which event started a given job run.
- **Event rule / action traceability** is missing — we cannot say which rule and action processed a given event.
- The Event Publisher in step 5 has no way to know which inbound correlation ID to carry forward, so the correlation chain breaks at every TWS hop.

The question is **how to re-establish the link between an inbound event and the job run it caused, without that information coming back from `sendevent`.**

## Decision

<img width="693" height="344" alt="image" src="https://github.com/user-attachments/assets/554a5cbc-6b6e-4382-ba70-dc96df073d89" />








**Establish correlation outside TWS, using a per-workflow event config file and Delta audit tables written by the listener and publisher. Do not read TWS system tables.**

Components:

| Component | Type | Purpose |
|---|---|---|
| `TWS_EVENT_CONFIG` | YAML config file | Per workflow: workflow ID, the dataset(s) it depends on, the dataset(s) it produces, the associated TWS event rule(s) and action, and any other identifying fields (e.g. domain). |
| `TWS_EVENT_LISTENER_AUDIT` | Delta table (Azure) | One row for every event the listener consumes and submits via `sendevent` — event ID, correlation ID, dataset name, domain, submit status, timestamps. |
| `TWS_EVENT_PUBLISHER_AUDIT` | Delta table (Azure) | One row for every data availability event the publisher emits — new event ID, carried-forward source correlation ID, workflow ID, run stats. |

Runtime flow:

1. **Listener** consumes the event, builds the custom event, calls `sendevent`, and records the event and submit result in `TWS_EVENT_LISTENER_AUDIT`.
2. Event Processing Server evaluates rules and runs the action (unchanged).
3. When the Informatica workflow completes, the **publisher** collects runtime stats for the workflow ID (and other unique fields) from the workflow / session logs or DISC API.
4. The publisher looks up the workflow in `TWS_EVENT_CONFIG` to get its required dataset(s) and event rule details.
5. The publisher queries `TWS_EVENT_LISTENER_AUDIT` for the **latest successfully submitted event** matching that dataset (and domain / other key fields), and takes its event ID and source correlation ID.
6. The publisher emits the new data availability event carrying that correlation ID and records it in `TWS_EVENT_PUBLISHER_AUDIT`.

The Databricks adapter follows the same pattern: look up the config, write the event to the event tables, and let the Databricks native **table-update trigger** start the job. Both sides therefore produce audit records in the same shape.

## Consequences

**Positive**

- **Correlation chain is unbroken** across TWS hops: the outbound event carries the correlation ID of the inbound event that (by config) caused the run.
- **One place for diagnostics and audit.** Listener and publisher audit tables hold every event in and out, so support teams can trace a hybrid run without logging into TWS.
- **No dependency on TWS internals.** Not reading TWS system tables avoids coupling to an undocumented, version-specific schema and avoids needing elevated access to the TWS database.
- **Same pattern on both platforms.** TWS and Databricks adapters share config-driven lookup and audit tables, which keeps operations and onboarding consistent.
- **Config-driven onboarding.** Adding a workflow means adding a config entry, not changing listener or publisher code.

**Negative**

- **Correlation is inferred, not observed.** The link between event and job run is reconstructed from config and "latest successful event for this dataset", not reported by TWS. If two events for the same dataset arrive close together, or a job is started manually / by a schedule, the publisher may attach the wrong (or a stale) correlation ID. Mitigation to be agreed: bound the lookup by time window and run start time, and record the match basis in the publisher audit row so a mismatch can be detected.
- **Rule / action traceability is indirect.** We know which rule *should* have handled the event (from config), not which rule actually fired. If confirmed traceability is required, a further source (e.g. TWS event logs) would be needed.
- **Config must stay in sync with TWS.** `TWS_EVENT_CONFIG` duplicates rule and dependency metadata that also lives in TWS event rules. Drift breaks correlation silently. Mitigation: one source of truth for onboarding, and a validation check between config and deployed rules.
- **New moving parts.** One config file and two audit tables to own, secure, retain and monitor. The audit tables live in Azure, so on-prem listener/publisher scripts take a new network dependency (to be captured in the hybrid FMEA).

## Alternatives considered

### A. Rely on the `sendevent` acknowledgement only

Treat a successful submit as sufficient and do not correlate further.

**Rejected because**: it gives no event-to-job or event-to-rule traceability, and the publisher cannot carry the correlation ID forward. The correlation chain breaks at every TWS hop, which defeats the purpose of the correlation ID.

### B. Read TWS system tables / plan data to link events to job runs

Query the TWS database or plan to find which job instance was triggered by which event.

**Rejected because**: the team chose not to depend on TWS system tables. It couples the adapter to TWS internal schemas and access, and does not give a clean place to hold audit data for both platforms. *(Authors to confirm the full rationale before acceptance.)*

## Open items

- **Data availability event source is not final.** Informatica workflow logs, session logs, and DISC API are all candidates; this ADR uses workflow/session logs as the working assumption. The choice affects the publisher's runtime-stats step and the config schema, and should be decided (possibly in its own ADR) before this ADR moves to Accepted.
- Final columns for `TWS_EVENT_CONFIG`, `TWS_EVENT_LISTENER_AUDIT` and `TWS_EVENT_PUBLISHER_AUDIT`, aligned with the Avro data availability event schema (backlog #16).
- Lookup rules for "latest successful event" (keys, time window, tie-breaking) — see Negative consequences.

## References

- [ADR 0002](0002-reject-tws-azure-service-bus-job-plugin-for-event-intake.md) — keeps the Python listener that this ADR depends on.

# ADR 0002: Keep the custom Python event listener; do not use the TWS Azure Service Bus job plugin for event intake

**Status**: Proposed

**Date**: 2026-10-13

**Authors**: Chandra Buchha — Kroger Hybrid Workflow Orchestration Squad

**Reviewers**: Doug Heitkamp

**Decision forum**: Hybrid Workflow Orchestration task update, 2026-10-09

---

## Context

The TWS adapter receives data availability events from the streaming platform and turns them into TWS custom events. The proposed design uses a **TWS Event Listener** (Python) that consumes each event, builds the custom event, calls the native `sendevent` command, and then completes the message — giving one control point for **receive → process → `sendevent` → acknowledge / retry**. The listener also writes the audit record that correlation depends on (see [ADR 0001](0001-tws-adapter-correlation-id-via-config-and-delta-audit.md)).

In an earlier session, Doug Heitkamp pointed to HCL Workload Automation's **Azure Service Bus job integration** plugin as a possible native replacement for the custom listener. The question is **whether that plugin can replace the custom listener, or simplify it enough to be worth adopting.**

What the plugin is, per HCL documentation:

- A **TWS job type**. Service Bus operations run as part of a TWS job's lifecycle, not as a standing service.
- **SEND** to a queue or topic — JSON, XML or plain text; message body or file.
- **RECEIVE** from a queue or a topic subscription — number of messages and maximum wait are configurable; the documented receive mode is **`RECEIVEANDDELETE`**.
- It is **not documented as an always-on Service Bus listener**.

## Decision

**Do not adopt the TWS Azure Service Bus job plugin for event intake. Keep the custom Python TWS Event Listener as designed.**
<img width="692" height="356" alt="image" src="https://github.com/user-attachments/assets/8713551f-147b-4871-baca-a81dea2e4b1e" />

Mapping the plugin onto the adapter flow shows it would only replace the "receive" step:

```
Azure Service Bus
  → TWS Azure Service Bus RECEIVER job      (plugin: consume only)
  → TWS listener job (custom, still needed)  (parse, build custom event, sendevent)
  → TWS Event Processor → rules & actions → submit job / reply to prompt
```

Most of the logic — parsing the message, building the custom event syntax, calling `sendevent`, handling failure, writing audit — would still sit in a custom successor job. In Doug's words from the review: *too much business logic is still needed between the incoming event and `sendevent`, which defeats the simplicity the plugin was supposed to bring.*

## Consequences

**Positive**

- **Single control point** for receive → process → `sendevent` → acknowledge / retry, so a message is only acknowledged (completed on Service Bus, checkpointed on Event Hub) after `sendevent` succeeds.
- **No message loss on downstream failure.** A failed `sendevent` leaves the message unacknowledged, so it can be retried or redelivered.
- **Correlation and audit stay in one component.** The listener writes `TWS_EVENT_LISTENER_AUDIT` in the same step that submits the event (ADR 0001).
- **Broker-agnostic.** The listener already targets Azure Event Hub and can move to DESP or another platform; the plugin only covers Azure Service Bus.
- **Event-driven, not polled.** An always-on listener reacts as events arrive instead of on a job schedule.

**Negative**

- **We own and run custom code** (Python listener) instead of a vendor-supported job type: packaging, hosting, monitoring, patching and HA are on us.
- **We forgo a native TWS integration** that TWS operators may find more familiar to schedule and monitor.

## Alternatives considered

### A. Plugin RECEIVER job + custom successor listener job

A scheduled TWS Azure Service Bus RECEIVER job pulls messages; a successor TWS job parses each message and calls `sendevent`.

**Rejected because**:

- **Polling, not listening.** The plugin runs as a job that has to be scheduled or submitted periodically. It is not documented as an always-on listener, which adds latency and job-scheduling overhead.
- **No `sendevent` capability.** The receiver job cannot parse the message or call `sendevent`; a separate custom job is still required, so custom code is not removed.
- **`RECEIVEANDDELETE` semantics.** The message is deleted from Service Bus as soon as it is received — before downstream `sendevent` processing succeeds.
- **No redelivery on downstream failure.** If `sendevent` fails after receive, retrying the downstream job does not bring the original message back. Events can be lost.
- **No single acknowledge / retry point.** Splitting receive and process across two jobs removes the receive → process → `sendevent` → acknowledge control that the listener design relies on.

### B. Plugin RECEIVER job alone

**Rejected because**: not feasible — the plugin cannot transform the message into a TWS custom event or invoke `sendevent`, so it cannot trigger event rules on its own.

## Open items

- Undocumented behaviour (e.g. a peek-lock receive mode, or an always-on mode) was not tested. If HCL confirms either, this decision can be revisited; the conclusion above is based on documented capabilities only.

## References

- *Hybrid Workflow Orchestration – Task Updates – 09Oct2026* (deck), slides 5–7: Azure Service Bus job integration, proposed TWS adapter architecture, findings.
- HCL Automation Hub — Azure Service Bus integration: https://www.yourautomationhub.io/detail/azureservicebus
- Meeting recording / transcript, 2026-10-09 — Doug Heitkamp asked for this evaluation to be captured as an ADR.
- [ADR 0001](0001-tws-adapter-correlation-id-via-config-and-delta-audit.md) — correlation ID design that depends on the listener.

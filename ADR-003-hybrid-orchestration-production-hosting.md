# ADR-003: Production hosting for Hybrid Orchestration — KPS vs. TWS plugin

| Field | Value |
| --- | --- |
| **Status** | **Accepted — Option 1 (KPS)** |
| **Date** | 29 Oct 2026 |
| **Decided by** | Doug |
| **Application** | Hybrid Orchestration (APM0008112) |
| **Supersedes** | Linux VM option (dropped) |

## 1. Context

Hybrid Orchestration links data-availability events to the batch jobs that act on them. A source system publishes a Data Available event to DESP. A **listener** picks it up and triggers TWS, which runs the Informatica workflow. When the load succeeds, a **publisher** uses run stats from DISC to publish a new Data Available event. A correlation ID ties every step together, and all audit data is kept in Databricks.

The Linux VM option was dropped because of governance concerns. Two production options were evaluated for the **8 Jan 2027** go-live.

## 2. Decision drivers

- Can be approved by Cloud Governance.
- Meet the 8 Jan 2027 go-live, just after the year-end freeze.
- Full end-to-end traceability through one correlation ID.
- Reliable processing: no lost events and no duplicate publishing.
- Reasonable run cost and support model.
- A good long-term home as more datasets are onboarded.

## 3. Components common to both options

| Component | Role |
| --- | --- |
| **DESP (Azure Event Hubs)** | Carries inbound and outbound Data Available events |
| **TWS (HCL Workload Automation)** | Schedules and runs the job streams. EPS event rules turn a custom event into a job stream submission |
| **Informatica PowerCenter** | Runs the workflow that loads the data (no changes to workflows) |
| **DISC API** | Source of workflow and session run statistics for the outbound event |
| **Databricks (Unity Catalog)** | Delta tables for listener audit, run link, publisher audit and dead letter |
| **Orchestration config** | Per-dataset rules: inbound match, dedupe keys, TWS job stream, publish rules, attribute mapping |
| **Correlation ID** | One ID per delivery cycle, carried through DESP, TWS, Informatica and the audit tables |

## 4. Options

### Option 1 — KPS (Kroger's shared Kubernetes)

**Concept.** The orchestration logic runs as small Python services on KPS. All calls go **out from KPS** to DESP, TWS, DISC and Databricks. TWS never calls into KPS. KPS **pulls** status from TWS and decides when to publish.

**Components**

| Component | Runs as | What it does |
| --- | --- | --- |
| Event Listener | KPS Deployment (always on) | Consumes events from DESP, matches config, dedupes, assigns the correlation ID, and triggers TWS with `sendevent`, passing the ID |
| Correlation Reconciler | KPS CronJob (every 2 min) | Queries the TWS REST API to find the job stream run started by each event, and records its status in the run link table |
| Event Publisher | KPS CronJob (every 2 min) | Picks runs that finished successfully, gets run stats from DISC, builds and publishes the outbound event to DESP |
| Platform services | KPS | Helm on kps-base-helmchart, Harness for deployment, Akeyless with External Secrets for secrets, Dynatrace for logs and monitoring, Kyverno policies |

**Flow (conceptual)**

```
DESP ──► Listener ──sendevent──► TWS EPS ──► Job stream ──► Informatica ──► DISC
              │                                   ▲
              ▼                                   │ (REST status query)
          Databricks ◄──── Reconciler ────────────┘
          audit tables
              ▲
              └──── Publisher ──► DISC (run stats) ──► DESP (outbound event)
```

**Pros**
- Strategic, governance-endorsed home for custom services.
- No OS patching; the platform team manages the nodes.
- Standard CI/CD, security policies and monitoring.
- Strongest traceability and self-healing status checks.
- Outbound-only networking, so no inbound path from on-prem.

**Cons**
- Longer lead time: governance review, KPS onboarding and network approvals.
- Triggering TWS from a container still needs to be proven.
- New skills needed for the team (Helm, Harness, Kyverno).
- Tight against the 8 Jan 2027 date.

### Option 2 — TWS plugin (platform-native)

**Concept.** The orchestration logic runs inside platforms that already exist. The listener and publisher are Databricks jobs. TWS **pushes** completion to Databricks through its plugin, so no new hosting platform is needed.

**Components**

| Component | Runs as | What it does |
| --- | --- | --- |
| Listener job | Databricks job (streaming or scheduled) | Consumes events from DESP, matches config, dedupes, assigns the correlation ID, and calls the TWS REST API to submit a generic job |
| Generic sendevent job | TWS job | Sends the custom event to EPS, which submits the target job stream |
| EDWA event rule | TWS | Detects that the Informatica job succeeded |
| TWS Databricks / RESTful plugin | TWS agent | Starts the Databricks publisher job and passes TWS runtime values (job, job stream, instance ID, correlation ID) |
| Publisher job | Databricks job | Gets run stats from DISC, builds and publishes the outbound event to DESP, and writes the audit |

**Flow (conceptual)**

```
DESP ──► Databricks listener ──REST──► TWS generic job ──sendevent──► EPS ──► Job stream ──► Informatica
                                                                                  │
                         DESP ◄── Databricks publisher ◄──plugin── EDWA rule ◄────┘ (on success)
```

**Pros**
- No new hosting platform.
- Uses existing TWS and Databricks capabilities.
- Near-real-time publishing.

**Cons**
- **Not expected to be approved by Cloud Governance.**
- Feasibility not proven: REST submission, parameter passing, and plugin installation.
- More orchestration logic sits in TWS configuration.
- An always-on listener adds Databricks compute cost.
- Push-only completion needs a safety-net check for missed calls.

## 5. Comparison

| Criterion | Option 1 — KPS | Option 2 — TWS plugin |
| --- | --- | --- |
| Governance approval | Expected (KPS is the endorsed platform) | **Not expected** |
| Fit with go-live date | Tight, but achievable with early onboarding | Depended on unproven tests |
| Traceability | Strong (reconciler confirms each run) | Depended on parameter passing |
| Reliability | Self-healing (pull-based) | Needed a safety-net check |
| Run cost | Low (small namespace quota) | Higher (always-on job) |
| Long-term fit | Strategic | Limited to batch orchestration |
| Publish latency | A few minutes | Near real time |

## 6. Decision

**Option 1 — KPS is selected.**

**Reason:** Cloud Governance will not approve the TWS plugin option. KPS is the governance-endorsed platform for custom services, and it gives the strongest traceability and reliability.

## 7. Next steps

1. **KPS assessment:** complete the assessment of what KPS requires (onboarding, policies, CI/CD, secrets, network) and close the open technical items, including triggering TWS from a container.
2. **Dev onboarding:** submit the KPS resource requirement and set up the dev namespace, CI/CD pipeline, secrets and network access to DESP, TWS, DISC and Databricks.
3. **Governance review:** prepare the Cloud Governance submission for the review planned on **10 Nov 2026**.

## 8. Consequences

- Go-live depends on KPS onboarding, network approvals and the 10 Nov governance outcome.
- The team builds Helm, Harness and Kyverno skills and supports the KPS workloads.
- The TWS plugin option is closed. The core logic stays hosting-independent.
- Non-prod uses only non-prod data sources, and the 8 Jan 2027 date is checked against the year-end freeze.

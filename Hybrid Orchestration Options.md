# Hybrid Orchestration — Implementation Options

**Audience** Gus**Date** Thu 29 Oct 2026**Status** Draft for discussion

The orchestration loop between TWS, Informatica PowerCenter, Databricks and Azure Event Hub (DESP). The **listener** consumes the data-availability event and triggers TWS. The **publisher** builds and publishes the data-availability event after the Informatica job finishes. Both write audit and state tables.

## At a glance

| # | Option | What it proves / delivers | Governance effort | Time to first result | Production path? |
| --- | --- | --- | --- | --- | --- |
| 1 | Local end-to-end prototype | The logic works (events, correlation, audit) | None | Days | No, proof only |
| 2 | Containerized prototype (Podman / Docker) | Same as 1, plus a portable image | None to low | Days | No, but the image carries forward to option 3 |
| 3 | KPS (Kubernetes) | Production-grade hosting on Kroger's shared AKS | High (Cloud Governance review, KPS onboarding) | Weeks to months | Yes |
| 4 | Platform-native orchestration (TWS EDWA + plugins + Databricks jobs) | Production design on platforms that are already approved | Low (no new compute) | Weeks | Yes, if TWS capabilities are confirmed |

## Options

Sections

Option 1Local end-to-end prototypePython scripts on a laptop; external systems mocked where needed

### What it is

The listener and publisher run as Python scripts on a developer laptop. External systems are either reached directly where network access allows, or simulated where it doesn't.

### Steps

1. **Set up the environment.** Install a pinned Python version in a virtual environment and keep a `requirements.txt` of the libraries.
2. **Stub or connect the external systems:**
   - **Event Hub:** use a non-prod namespace, or a local emulator or mock.
   - **TWS:** a mock REST endpoint that records calls. Use the real non-prod TWS only if it is reachable from the laptop.
   - **Informatica run stats / DISC API:** sample JSON responses.
   - **Audit and state tables:** local files or SQLite standing in for Delta.
3. **Build the listener:** consume the event, dedupe it against the state table, call TWS (mock), and write the listener audit.
4. **Build the publisher:** read `TWS_EVENT_CONFIG`, gather run stats, look up the correlation ID from the listener audit, build the event to Doug's structure, publish it, and write the audit.
5. **Run test scenarios:** happy path, duplicate event, Informatica failure, two events close together, and a workflow missing from the config.
6. **Demo** the flow and walk through the audit trail.

### Trade-offs

| Pros | Cons |
| --- | --- |
| Fastest and cheapest route; no approvals needed | Not deployable; runs only on one laptop |
| Validates event schema, correlation and dedupe logic early | Real TWS, Informatica and Event Hub are mostly mocked, so integration risk remains |
| Easy to iterate on Doug's event structure | Laptop network may block on-prem or Azure endpoints |
|  | Must not use production data or credentials locally |

### Reasoning

Use this to de-risk the **design logic**, not the platform. It answers "does the correlation and event model work?" before anyone commits to hosting.

Option 2Containerized prototype on local machine (Podman / Docker)Option 1 packaged as container images, built to KPS rules

### What it is

The same prototype as option 1, packaged as container images and run locally with Podman or Docker, with one container for the listener and one for the publisher.

### Steps

1. **Write a Dockerfile.** Use a pinned Python base image and install libraries at build time. Use the JFrog/Artifactory mirror if it can be reached.
2. **Externalize configuration.** Pass config through environment variables or a mounted `TWS_EVENT_CONFIG` file, and pass secrets through a local `.env` file. Never bake config or secrets into the image.
3. **Compose the stack** with `podman-compose` or `docker compose`: listener, publisher, mock TWS endpoint, and mock DISC API.
4. **Apply KPS-style rules now:** non-root user, read-only root filesystem, logs to stdout, graceful SIGTERM handling, no `latest` tag.
5. **Run the same test scenarios** as option 1.
6. **Scan the image** with Snyk, if available locally.

### Trade-offs

| Pros | Cons |
| --- | --- |
| Image and config pattern carry straight into KPS (option 3) | Still a local prototype, not production |
| Behaves the same on every developer machine | Container runtime needs setup; Docker Desktop has licensing considerations, Podman doesn't |
| Surfaces KPS policy issues early (read-only FS, non-root, resource limits) | Same network and mock limitations as option 1 |
|  | Slightly more effort than option 1 |

### Reasoning

This is a better version of option 1 **if KPS remains a candidate**, because the work isn't thrown away. If option 4 is chosen, the extra container effort adds little.

Option 3KPS (Kroger shared Kubernetes / AKS)Listener and publisher as containers in a KPS namespace

### What it is

The listener (a Deployment) and the publisher (a Job or CronJob) run as containers in a namespace on KPS, Kroger's shared multi-tenant AKS platform. KPS SRE owns the nodes; the ETL team owns the image.

### Steps

1. **Cloud Governance review.** This is a KPS prerequisite.
2. **KPS onboarding:** CET Foundations request, resource requirement document, capacity and quota review, and namespaces for dev, stage and prod.
3. **Build and deploy.** Build the image with GitHub Actions, scan it with Snyk, and push it to Artifactory. Package it as a Helm chart on `kps-base-helmchart` and deploy it with Harness NextGen.
4. **Secrets.** Store them in Akeyless and sync them into the namespace with External Secrets Operator.
5. **Resolve the two hard design items:**
   - **/gpfs access:** NFS volume, Azure storage, or on-prem file edge.
   - **TWS hand-off:** Event Hub, an agent on a utility server, or a containerized agent.
6. **Network.** Private endpoints to Event Hub, Databricks and Key Vault. On-prem allow-lists through Azure Interconnect for TWS, Oracle and GPFS (NetSec).
7. **Observability.** Dynatrace and Prometheus/Grafana are provided by the platform.
8. **Comply with Kyverno policies:** requests and limits set, read-only root filesystem, no `latest` tag, CronJob TTL.
9. **Test end to end in non-prod, then promote** through stage to prod.

### Trade-offs

| Pros | Cons |
| --- | --- |
| Strategic, governance-endorsed hosting model | Longest lead time: Governance review, onboarding, NetSec allow-lists |
| No OS patching or Midrange support (KPS SRE owns nodes) | /gpfs and TWS hand-off have no direct equivalent on Kubernetes; both need redesign |
| Scales, self-heals, standard CI/CD and observability | New skills needed (Helm, Harness, Kyverno) |
| Reuses the image from option 2 | Risky against the 8 Jan 2027 go-live and the year-end change freeze |

### Reasoning

This is the right long-term home if custom Python services are needed. The **governance and onboarding path is the bottleneck**, not the code.

Option 4Platform-native orchestration Validate firstTWS event rules + TWS plugins + Databricks jobs, on platforms already approved

### What it is

Orchestration is built from capabilities **already running and approved** at Kroger:

- **TWS:** REST API, Event Processing Server and EDWA event rules.
- **The TWS Databricks plugin or RESTful job plugin.**
- **Databricks jobs** in the existing `etl-general` workspace, which host the listener and publisher logic.

**Why this option exists:** to avoid the governance bottleneck of provisioning a new VM or getting KPS access. There is no new compute to stand up and no Python libraries to deploy onto a new server or into a container platform. The code runs inside the existing, already-governed Databricks workspace, and TWS uses its own built-in features.

### How the flow works (diagram step numbers)

**Inbound — listener, steps 2 → 8**

1. A Databricks listener job consumes the event from DESP.
2. It checks applicability against the config and dedupes against the listener state.
3. It calls the TWS REST API to submit a generic job.
4. The generic job's shell script runs `sendevent` to EPS.
5. The event rules either submit the target job or reply YES to a prompt, and TWS runs the Informatica job.

**Outbound — publisher, steps 9 → 11**

1. An EDWA rule detects that the Informatica job succeeded.
2. The rule invokes a generic Databricks publisher job through the Databricks plugin or a REST job, passing TWS runtime values: job name, job stream name and instance ID.
3. The publisher collects static attributes from the config, dynamic attributes from the DISC API, and the correlation ID from the listener state.
4. It publishes to DESP and writes the publisher state and job audit.

### Steps

1. **Confirm TWS capabilities with Sravanthi.** These are the gating items:
   - Can REST submissions pass **per-call dynamic arguments** (variable table or variable list values)? Are they safe when submissions run concurrently?
   - Can the **Databricks plugin or REST job** pass runtime values (job name, job stream, instance ID) as Databricks job parameters?
   - Which built-in variables are available inside an EDWA rule action?
   - Are the plugins licensed and installed on the target agent?
2. **Build the TWS side** (TWS team): a generic `sendevent` job, a custom event type, event rules (submit / reply-to-prompt), and an EDWA rule on Informatica job success.
3. **Build the Databricks side** (ETL team): a listener job (streaming consumer of DESP), a generic publisher job with job parameters, Delta tables for listener state, publisher state and job audit, deployed through the existing Databricks CI/CD (OIDC deployer service principal).
4. **Connectivity and access:** service account and credentials for Databricks → TWS REST, credentials for TWS plugin → Databricks, the firewall path between them, and certificates for TLS.
5. **Test end to end in non-prod:** correlation ID present in every audit table, duplicate events ignored, failure path behaves as designed, concurrent events keep their own parameters.
6. **Confirm with Cloud Governance** that hosting the logic in the existing Databricks workspace doesn't need a new intake, or only needs a light one.

### Trade-offs

| Pros | Cons |
| --- | --- |
| **Avoids the VM / KPS governance bottleneck:** no new compute, no Python libs on new infrastructure | Feasibility depends on TWS answers (dynamic REST arguments, plugin parameter passing) that are still open |
| Uses approved platforms (TWS, Databricks, Event Hub); fastest realistic route to 8 Jan 2027 | Listener as a continuously running Databricks job adds an always-on compute cost |
| Removes the /gpfs mount and host-installed TWS workstation problems | More logic sits in TWS configuration (rules, generic jobs), so the TWS team carries more ownership |
| TWS stays the system of record for scheduling; the native plugin reports status back | Still depends on the DISC API for run stats |
| No OS patching, Midrange support, or container platform to run | Variable-table concurrency needs a careful design |
|  | Governance still needs to agree that no new intake is required |

### Reasoning

This is the **fastest path to production that stays inside approved platforms**, which is why it is worth validating first. The risk is concentrated in a few TWS capability questions, and each can be answered with a small proof test.

## Suggested sequencing

1. **Now:** run option 1 (or option 2 if KPS stays in play) to prove the event model and correlation logic.
2. **In parallel:** get Sravanthi's answers on the option 4 gating questions and run the two small TWS proof tests.
3. **Decide:** if the option 4 answers are positive, take it as the target for the 8 Jan 2027 go-live. If they are negative, fall back to option 3 (KPS) and start Governance and onboarding immediately.

## Open decisions (apply to every option)

- Event Hub vs DESP naming and namespace
- Final event schema (Doug's structure) and audit schemas
- `TWS_EVENT_CONFIG` ownership
- Behaviour on failure (no event vs failure event)
- Dedupe strategy
- Non-prod sources: no production data in non-prod
- Year-end change freeze vs the 8 Jan 2027 go-live
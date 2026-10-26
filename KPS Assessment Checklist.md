APM0008112 · Hybrid Orchestration · KPS option · 26 Oct 2026

# KPS assessment checklist for the Python listener and publisher

Only the checks that exist because we are moving to KPS. Platform-independent design decisions (event schemas, Event Hub vs DESP, TWS rule design, non-prod sources and similar) are tracked separately.

**Design baseline: the correlation strategy diagram.**

The listener consumes the data-availability event and calls `sendevent` to the TWS event processing server. The publisher reads TWS_EVENT_CONFIG, workflow run stats and the listener audit, then publishes the next event. Both write Delta audit tables.

## KPS blockers

Each can stop the KPS option

B1Can a KPS pod run `sendevent`?

Today it only works from TWS servers and agents, and only after the SSL port was fixed (issue tracker #20). A pod needs the TWS CLI and config, a certificate, a licence position and a path to the event processing server. Otherwise the listener needs a REST call or a relay on an agent host.

B2On-prem access from a shared egress range

Pods must reach the TWS event processor, the PowerCenter Oracle repository and the session logs. KPS clusters share an egress range with other tenants; NetSec has to accept on-prem rules for it.

B3No /gpfs mount

KPS advises against NFS volumes, so the publisher can't read session logs or the YAML from $GPFSROOT/hybrid_orch the way a VM would.

B4Onboarding lead time vs 8 Jan 2027

KPS needs a Cloud Governance review, ServiceNow access, an Artifactory repo, a GitHub team, a BIA rating, a resource requirement doc, and a stage performance test before prod sizing.

B5Delta audit writes from a pod

Steps 3, 9 and 10 read and write Delta from outside Databricks: a SQL warehouse over a private endpoint, with its latency and cost on the hot path.

## KPS risks we are not yet covering

R1Pod restarts and replicas raise duplicate risk

Node patching moves pods mid-event and two listener replicas share partitions. A replay means a second `sendevent` and a second job stream unless the audit check runs first.

R2Read-only root filesystem

Enforced in KPS prod since 6 Oct 2026. pyarrow, the Databricks connector, schema-registry caches and `HOME` writes all need checking.

R3Two runtimes for one codebase

If the `sendevent` relay or any piece stays on AIX (Python 3.9, packages waiting on Midrange), the shared library forks.

R4Support split with KPS SRE

KPS SRE owns nodes; we own the image and the pods. Paging, patch windows and the manual fallback when TWS jobs wait need to be agreed.

R5Year-end freeze vs KPS prod onboarding

Prod quota is sized only after a stage performance test, and the freeze sits right before go-live.

## Design decisions that change between KPS and VM

Answer differently depending on the platform

Two of these are hinges: the TWS hand-off and file access. Settle them first, because the listener's and publisher's designs follow from them.

| Decision | On the VM | On KPS |  |
| --- | --- | --- | --- |
| How the listener submits to TWS | Native: a TWS workstation on the VM runs `sendevent` | `sendevent` in the image, TWS REST, or a relay on an agent host | hinge · B1 |
| Where run stats come from | Read session logs from the /gpfs mount | PowerCenter repository, IDMC or DISC API, or a file edge | hinge · B3 |
| Where TWS_EVENT_CONFIG lives | $GPFSROOT/hybrid_orch, edited in place | Image or ConfigMap, changed through a release |  |
| Publisher trigger and state | TWS job or cron on the VM; state on local disk | CronJob or Deployment with leases; state outside the pod |  |
| How strict dedupe must be | Single process; replays mostly from restarts | Replicas and pod moves; audit check before every `sendevent` is mandatory |  |
| Audit write path | SQL connector from a fixed host | Same connector, through a private endpoint from a shared range; restart-safe writes |  |
| Identity | System-assigned managed identity | SP with federated credentials or workload identity |  |
| Secrets | Key Vault | Akeyless through ESO (or Key Vault if KPS allows) |  |
| On-prem network | Rules for one VM IP | Rules for a shared egress range |  |
| Logging and alerts | OneAgent, log files | JSON on stdout, Grafana alerts as k8s resources |  |
| Runtime and patching | Midrange patches the OS; uv on the host | We patch the image; KPS SRE patches nodes |  |

The same on either platform: event and audit schemas, Event Hub vs DESP, correlation into TWS, TWS event rule design, non-prod source wiring, DISC API scope.

## Tanmay · Listener

sendevent from a pod (with Sravanthi)B1

- [ ] Find out exactly what `sendevent` needs to run: binary, libraries, config files, certificates, user, SSL port.
- [ ] Try it from a plain Linux host that is not a TWS agent, as the closest stand-in for a container.If it only works on registered agents, a pod won't work either.
- [ ] Prototype the alternative: an HTTPS call to the TWS REST interface, if Sravanthi confirms one exists for event submission.
- [ ] Define timeouts, retries and how the listener knows TWS accepted the event.

Replicas and restartsR1

- [ ] With two replicas, confirm partition ownership so the same event is never handled twice in parallel.
- [ ] Handle SIGTERM: finish or abandon the in-flight event without checkpointing it.
- [ ] Add readiness and liveness probes tied to the receive loop.
- [ ] Check the KEDA scaler fits the chosen consumer. [KRCHOP → KPS Migration](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/1461420471)

Delta audit writes from a podB5

- [ ] Choose the write path from a pod: SQL warehouse via the SQL connector, Databricks SDK, or events materialized by a Databricks job.
- [ ] Measure write latency and warehouse start-up on the hot path; note the cost of keeping it warm.
- [ ] Plan compaction for many small appends.

Container fitR2

- [ ] Pick the target Python version for the image and note any 3.9-only code.
- [ ] Run the listener in a local container as non-root with a read-only root filesystem and /tmp as the only writable path. [Read-only Root File System](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/88998207)
- [ ] Decide where TWS_EVENT_CONFIG lives for the listener (ConfigMap or image).

Identity, logging, sizing

- [ ] Test federated identity for Event Hub and Databricks instead of SP secrets. [Configuring SP Federated Identity](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/88998112) · [Workload Identity – DBx Access](https://kroger.atlassian.net/wiki/spaces/apiprogram/pages/1808368207)
- [ ] Switch logging to JSON on stdout with the correlation ID on every line (Dynatrace picks it up on KPS).
- [ ] Propose Grafana alerts: consumer lag, failed `sendevent`, dead letters. [Setting Up Alerts](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/89000411) · [Platform Level App Alerts](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/1573126983)
- [ ] Give Chandra CPU, memory and replica numbers from the POC for the resource requirement doc.

## Akshata · Publisher

Run stats without /gpfsB3

- [ ] List which session log fields the publisher uses and whether the PowerCenter repository, IDMC or the DISC API can replace each.
- [ ] For each source, record how it's reached from KPS and the auth it needs.

Running on Kubernetes

- [ ] Choose CronJob or Deployment; if a Deployment with two replicas, use a lease so only one polls.
- [ ] Keep high-water marks and processed-run state outside the pod.
- [ ] Decide how TWS_EVENT_CONFIG ships: in the image, a ConfigMap, or a repo synced at deploy.

Container, identity and sizingR2

- [ ] Run in a local container as non-root with a read-only root filesystem.
- [ ] List credentials (Oracle, Event Hub, Databricks, DISC) and where each comes from on KPS. [Akeyless Initial Setup](https://kroger.atlassian.net/wiki/spaces/ISAE/pages/58004769)
- [ ] Move the shared library from the Databricks volume (tracker #14) into a package both images install at build time.
- [ ] Give Chandra CPU, memory and schedule numbers for the resource requirement doc.

## Sravanthi · TWS

Submitting events from outside TWSB1

- [ ] Document the full set of prerequisites for `sendevent`, and whether it must run on a registered agent.
- [ ] Confirm whether our TWS version exposes a REST API for submitting events, and its auth model.
- [ ] Ask IBM (as with case TS023100260) whether a containerized client or agent is supported, and what licensing applies.
- [ ] Give the event processing server host, port and certificate details per environment for the firewall request.
- [ ] If a pod can't submit events, propose the lightest relay on an agent host.

## Chandrakant · KPS liaison

KPS team introduction and onboarding factsB4

- [ ] Book KPS office hours (Tuesdays and Fridays, 2–3 pm ET) and introduce the team to KPS SRE. [KRCHOP → KPS Migration](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/1461420471)
- [ ] Ask which shared cluster APM0008112's LOB and cost center map to, and the default dev quota. [Quota Management](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/88998134)
- [ ] Confirm the prerequisites and who has them: ServiceNow access, Artifactory repo (Tool Chain), GitHub team, BIA rating, SPM. [Azure Kubernetes Platform](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/88998039)
- [ ] Ask for realistic onboarding lead time, including the stage performance test before prod sizing.
- [ ] Ask how the DISC team handled KPS (secrets, identity, Harness) and borrow their setup. [DISC Modernization CI/CD Plan](https://kroger.atlassian.net/wiki/spaces/ETL/pages/1734378638)

Network, secrets and identity questions for KPSB2

- [ ] Get the cluster egress range and NetSec's position on on-prem rules for it (TWS, Oracle, session logs).
- [ ] Ask how private endpoints to Event Hub, Databricks and storage are set up for tenants. [Private Endpoints – Subnet Overrides](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/265029183)
- [ ] Ask whether Key Vault via ESO is allowed or Akeyless is required.
- [ ] Ask whether workload identity or federated credentials are supported for tenants now.

## Chandra · Architecture

Hinge decisions

- [ ] TWS hand-off on KPS: `sendevent` in a pod, REST, or relay on an agent host (from B1).
- [ ] Run-stats source without /gpfs (from B3).
- [ ] Audit write path from pods (from B5).

Network and security on KPS

- [ ] Build the KPS flow matrix: source, destination, port, protocol, encryption, approver, for the shared egress range.
- [ ] Identity and secrets design for both pods; certificate source for TLS. [Certificate Management](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/88998137)

KPS paperwork and governance

- [ ] Draft the resource requirement doc from Tanmay's and Akshata's numbers. [Template](https://kroger.atlassian.net/wiki/display/DCI/Resource+Requirements) · [ETL example (EPRN RAG)](https://kroger.atlassian.net/wiki/spaces/DCI/pages/1883242502)
- [ ] Walk the production readiness checklist and Kyverno policies for gaps. [Production readiness](https://kroger.atlassian.net/wiki/display/DCI/KPS+Production+Readiness+Requirements) · [Policy Enforcement](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/88998261)
- [ ] Decide the HA pattern for both pods. [HA best practices](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/88998185)
- [ ] Prepare the KPS-based Cloud Governance submission (a KPS onboarding prerequisite).

Go / no-goR4R5

- [ ] Set the criteria for VM-first versus KPS-first for 8 Jan 2027, with the fallback.
- [ ] Agree the support split with KPS SRE: paging, patch windows, manual fallback.
- [ ] Consolidate findings into the KPS design and book the review.

## References

- Project
- [Hybrid Orchestration – Issue Tracker & Status](https://kroger.atlassian.net/wiki/spaces/ETL/pages/1795850988)
- [Hybrid Orchestration Overview](https://kroger.atlassian.net/wiki/spaces/ETL/pages/1917616236)
- [DISC Modernization CI/CD Plan – Non Prod](https://kroger.atlassian.net/wiki/spaces/ETL/pages/1734378638)
- KPS onboarding
- [Azure Kubernetes Platform (prerequisites)](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/88998039)
- [KPS for Azure Platform On-boarding](https://kroger.atlassian.net/wiki/spaces/DCI/pages/155812309)
- [Resource Requirements template](https://kroger.atlassian.net/wiki/display/DCI/Resource+Requirements)
- [KPS Production Readiness Requirements](https://kroger.atlassian.net/wiki/display/DCI/KPS+Production+Readiness+Requirements)
- [KRCHOP → KPS Migration (platform differences)](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/1461420471)
- [Quota Management](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/88998134)
- [Rancher Access Request via SPM](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/1424262322)
- [Harness NextGen First Deployment](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/899058469)
- [Pull Images from Artifactory SaaS](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/88997977)
- Platform rules
- [Policy Enforcement (Kyverno)](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/88998261)
- [Read-only Root File System](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/88998207)
- [Best practices for high availability](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/88998185)
- [Right-sizing requests vs usage](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/88998048)
- [Certificate Management](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/88998137)
- [Setting Up Alerts](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/89000411) · [Platform Level App Alerts](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/1573126983)
- Identity, secrets, network
- [Configuring SP Federated Identity](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/88998112)
- [Workload Identity – DBx Access](https://kroger.atlassian.net/wiki/spaces/apiprogram/pages/1808368207)
- [Akeyless Initial Setup](https://kroger.atlassian.net/wiki/spaces/ISAE/pages/58004769)
- [Private Endpoints – Subnet Overrides](https://kroger.atlassian.net/wiki/spaces/KPSD/pages/265029183)
- Example
- [EPRN ETL Migration Assistant – KPS Resource Requirement](https://kroger.atlassian.net/wiki/spaces/DCI/pages/1883242502)
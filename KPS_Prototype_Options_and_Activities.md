# Hybrid Orchestration on KPS: prototype options and activities

APM0008112 · Hybrid Orchestration · 26 Oct 2026

## Walk-through for the call

1. **Direction.** Kubernetes (KPS) is the only hosting path. The application architecture stays the same: a Python listener and a Python publisher, TWS_EVENT_CONFIG, Delta audit tables and Event Hub. Only the hosting and a few integration points change.
2. **Where we prototype.** There are three places we can run the listener and publisher. We use all three in sequence, overlapping, so nothing waits on approvals.
3. **How we split the work.** Path A builds the prototype on local machines (Tanmay, Akshata). Path B gets KPS, TWS and the network ready (Chandrakant, Sravanthi, Chandra). The two meet when the dev KPS namespace is ready.
4. **What we need from this call.** Agreement on the split, confirmation that Podman or Docker is allowed on the offshore laptops, and a go-ahead to request the dev namespace and book KPS office hours this week.

## The three options

| | Local machine (plain Python) | Local container (Podman or Docker) | KPS dev namespace |
|---|---|---|---|
| **What it is** | Run the scripts directly with Python on the laptop | Package each service as an image and run it on the laptop | Deploy the same images to our namespace on the shared KPS cluster |
| **Setup needed** | Python and the packages | Podman or Docker installed and allowed; Artifactory access for the base image | Namespace, Rancher AD group, Harness project, secrets store |
| **Can start** | Today | Once tooling is approved (days) | After KPS onboarding (weeks) |
| **Proves app logic** | Yes | Yes | Yes |
| **Proves KPS runtime rules** (non-root, read-only filesystem) | No | Yes, simulated | Yes, enforced by Kyverno |
| **Proves restart safety** | Partly | Yes | Yes, with real pod moves and two replicas |
| **Reaches Event Hub and Databricks** | Likely | Likely | Yes, through private endpoints |
| **Reaches TWS-TEST and the Oracle repository** | Only if the offshore network allows it | Same as plain local | Yes, once NetSec approves |
| **Tests the TWS hand-off** | Only if TWS-TEST is reachable | Same; also shows whether `sendevent` works in a container | Yes, the real path |
| **Tests secrets and identity** | No (POC credentials) | No (POC credentials) | Yes (Akeyless, workload identity) |
| **Gives sizing for the resource doc** | No | Rough numbers | Real numbers |
| **Main limitation** | Says nothing about KPS | Can't prove network, policy or identity | Slowest to get |
| **Best used for** | Quick logic changes and unit tests | The prototype: KPS-ready images | Final validation and closing the prototype |

**Recommendation.** Keep iterating on the logic locally now. Move to local containers as soon as the tooling is approved, because that produces the images KPS will run. Deploy those same images to the dev namespace when onboarding completes, and close the prototype there.

If Podman or Docker isn't allowed on the laptops, the local container option drops out. Every container test then moves to the KPS namespace, and the onboarding timeline becomes the critical path.

## Activities

### Path A: local prototype (Tanmay, Akshata)

Goal: prove the application works as containers under KPS-style rules, so the images are ready when the namespace is.

| Owner | Activity | What we're trying to do |
|---|---|---|
| Both | Confirm Podman or Docker is allowed on their laptops | Everything container-based depends on it |
| Both | Check which systems their machines can reach (TWS-TEST, Oracle repository, Event Hub, Databricks) | Know which tests can run locally and which must wait for the namespace |
| Both | Get Artifactory access for the approved Python base image | Needed to build KPS-compliant images |
| Tanmay | Containerize the listener: non-root, read-only filesystem, /tmp only, JSON logs on stdout | Meet KPS rules so the pod deploys cleanly the first time |
| Tanmay | Add the duplicate check before `sendevent` or `run-now`, and test kill-and-replay | A replayed event must never submit the same TWS job twice |
| Tanmay | Time Delta audit writes from a container through the SQL warehouse | Know the latency and cost of writing an audit row per event |
| Akshata | Map every session-log field to a PowerCenter repository column or IDMC API field | Replace reading /gpfs, which pods can't reach |
| Akshata | Move TWS_EVENT_CONFIG to a ConfigMap with schema validation | Config changes go through a controlled release |
| Akshata | Package the shared library for both images; update DEART-58071 | Both services run identical, tested code with proper credentials |

### Path B: KPS platform readiness (Chandrakant, Sravanthi, Chandra)

Goal: get the namespace, network, access and TWS hand-off ready, so the images have somewhere to run and something to talk to.

| Owner | Activity | What we're trying to do |
|---|---|---|
| Chandrakant | Book KPS office hours (Tue or Fri, 2–3 pm ET) and get a named KPS SRE contact | A direct line to the team that sets our cluster, quota and timeline |
| Chandrakant | Request the dev namespace; track ServiceNow, Artifactory, GitHub team, BIA rating and SPM | KPS won't onboard us until these are in place |
| Chandrakant | Ask KPS SRE: cluster and quota, egress range, Akeyless or Key Vault, workload identity, lead time | Answers that shape the security design and the timeline |
| Chandrakant | Get non-prod PowerCenter repository access (host, port, service name, read-only account) | The publisher's replacement source for run stats |
| Chandrakant | Raise the NetSec request for on-prem access from the KPS egress range | Confirm pods can reach the TWS event server and the Oracle repository |
| Chandrakant | Add Tanmay, Akshata and Chandra to the Rancher Developer AD group | Access ready by the time the namespace is |
| Sravanthi | Test `sendevent` from a non-agent Linux container; check for a TWS REST API | Prove the listener can trigger TWS from a pod |
| Sravanthi | Find out what the TWS Kubernetes plug-in needs to reach our cluster | An alternative where TWS starts our pod instead |
| Sravanthi | Prove event-rule variables carry the job name and job stream; get the TWS-side POC scheduled | End-to-end correlation without changing Informatica jobs |
| Chandra | Update Jira to the KPS direction (DEART-58661, DEART-58015, DEART-58560, EDSP-6377; hold DEART-57854) | Tickets reflect the new direction |
| Chandra | Draft the network flow matrix and the KPS resource requirement doc | Inputs for NetSec, KPS and Governance |
| Chandra | Decide the three design hinges as findings come in: TWS hand-off, run-stats source, audit write path | A stable design to build against |

### Where the paths meet

When the dev namespace is ready:

1. Tanmay and Akshata deploy the same images to KPS through Harness.
2. They run the tests local machines couldn't: the TWS hand-off, Oracle queries through the egress range, two replicas and pod kills under Kyverno, and logs in Dynatrace.
3. Chandra records the results against the design decisions and closes the prototype on KPS.

### Dependencies between the paths

- **Path A needs from Path B:** Artifactory access and the read-only Oracle account (Chandrakant), and the `sendevent` findings (Sravanthi).
- **Path B needs from Path A:** CPU, memory and replica numbers for the resource requirement doc (Tanmay, Akshata).

## Next after the prototype

The governance readiness list (KPS_Governance_Readiness.xlsx) tracks the 21 assessments to complete before returning to Cloud Governance. The prototype results feed items 2, 3, 6, 7 and 12 directly.

# NiFi 1 → NiFi 2 (CDP) AI-Assisted Migration Tool — Discovery

_This document is the working artifact for the pre-build discovery phase of the NiFi AI Migration Tool project. It contains: the customer questionnaire, the proposed architecture the questionnaire is validating, and the v1 success criteria. Comments and edits welcome — this is meant to evolve as answers come in._

---

## Context

The project migrates customer NiFi 1.x flows to NiFi 2.x running on **CDP (Cloudera Flow Management / Cloudera DataFlow)**. The initial focus is customers running self-supported vanilla NiFi 1.x clusters, but the design is not tied to that source.

The envisioned tool:
- Docker-packaged, runs on a customer engineer's laptop.
- Reaches the **source NiFi 1 cluster** (mandatory).
- Pulls flow definitions + runtime metrics from the source into a local repo.
- Retrieves ontologies / migration rules / breaking-change catalog from a **vector DB** (RAG).
- Uses an **agentic LLM workflow** (LangGraph-style) to propose updated NiFi 2 flow definitions.
- Presents proposals through a **human-in-the-loop UI** — approve, reject, or "improve this."
- **v1 scope: emit approved NiFi 2 flow definition JSON files.** Customer imports them into CDP manually. Automated deployment to CDP DataFlow and shadow-run validation are follow-on features, not v1.
- **LLM assumption for v1:** the Docker container has network access to a **Cloudera AI Inference (CAI)** endpoint serving the model of choice. A provider-abstract client lets us swap to Bedrock / Anthropic direct / on-prem vLLM without rewriting the agent.

---

## 1. Note on runtime metrics: is there a "Spark History Server" analog in NiFi?

**Yes — the NiFi REST API + Provenance Repository together are the closest analog, and they're richer than the SHS for this use case.** No special service needs to be running; every NiFi 1.x cluster exposes this by default.

Key endpoints under `/nifi-api/` on the source cluster (auth required — usually mTLS, OIDC, LDAP, or Kerberos SPNEGO):

| Endpoint | What you get |
|---|---|
| `GET /flow/process-groups/root` | Root PG tree — recurse for the full hierarchy |
| `GET /process-groups/{id}/download` | **Full flow definition as JSON** (this is the key export for the migration) |
| `GET /flow/process-groups/{id}/status?recursive=true` | Real-time counters: bytes in/out, tasks, queued flowfiles, backpressure |
| `GET /flow/process-groups/{id}/status/history` | Time-series metrics (5-min buckets, retention configurable) |
| `GET /system-diagnostics` | JVM heap, GC, thread pools, disk usage per repo |
| `GET /flow/reporting-tasks` | Existing reporting tasks (Prometheus, Ambari, Site-to-Site provenance, etc.) |
| `POST /provenance` + `GET /provenance/{id}` | Lineage / event history queries (paged) — the closest thing to SHS event log |
| `GET /controller-services` | All controller services (JDBC pools, SSL contexts, credential providers) |
| `GET /flow/cluster/summary` | Cluster node health |
| `GET /flow/parameter-contexts` (1.10+) | Parameter contexts (variables in older versions) |

If the customer has a **Prometheus Reporting Task** configured, that is by far the cleanest metrics pull — TSDB-shaped, already aggregated. We ask about this in discovery.

**NiFi Registry**, if in use, is the other important source: `GET /nifi-registry-api/buckets/{id}/flows/{flowId}/versions/{v}` gives a versioned flow snapshot, which is more reliable than pulling from a running canvas.

### Quick primer: NiFi Registry vs live canvas

- **Live canvas** = the running NiFi 1 cluster's current flow state — what an engineer sees in the browser UI. Exportable via `GET /nifi-api/process-groups/{id}/download`. No version history — it is "whatever happens to be deployed right now."
- **NiFi Registry** = a *separate* service (installed alongside NiFi) that acts as **Git for NiFi flows**: versioned process groups organized into buckets, with commit history and a REST API at `/nifi-registry-api/`. Not every customer runs it; mature ops shops usually do.

For our tool, if Registry exists we prefer it (clean versioned artifacts). If not, we fall back to the live-canvas API. The Ingestor supports both, config-driven per flow.

---

## 2. Customer discovery questionnaire

Grouped so it can be trimmed or expanded per meeting. Anything marked ⭐ is a hard blocker if the answer is bad.

### A. Source NiFi environment inventory
1. ⭐ Exact NiFi version (e.g., 1.15.3, 1.23.2, 1.28.1). Minor version matters a lot for the 2.x diff.
2. Distribution: Apache vanilla, Hortonworks HDF, Cloudera CFM, or custom build?
3. Java version and OS (RHEL 7/8/9, Ubuntu, Windows)?
4. Deployment topology: single-node, clustered (how many nodes), ZooKeeper embedded or external?
5. Hosting: bare metal, on-prem VM, private cloud (which), public cloud (which), Kubernetes?
6. Is **NiFi Registry** in use? Version? Are all flows versioned there, or only some?
7. Are there custom NARs? How many, written in Java, Groovy, Jython, JS?
8. Node count, sizing (CPU/RAM), disk layout for content/flowfile/provenance repos.

### B. Flow inventory & complexity
9. ⭐ Total number of flows (root-level process groups) in scope.
10. Total processors, controller services, connections (approximate is fine — API can confirm).
11. Depth of process-group nesting (a rough max depth).
12. Use of **Variables** vs **Parameter Contexts**? (Variables are removed in NiFi 2 — this is a mechanical migration.)
13. Use of deprecated / removed-in-2.x processors? (We can probe with the API; ask if they know of any hand-coded exceptions.)
14. Site-to-Site (S2S) connections to other NiFi clusters or MiNiFi agents?
15. Sensitive properties: how is `nifi.sensitive.props.key` managed? Rotation history?
16. Reporting tasks in use (Prometheus, Ambari, Site-to-Site Provenance, custom)?

### C. Runtime characteristics (the "Spark History Server" pull)
17. ⭐ Is the REST API reachable, and with what auth (mTLS certs available? LDAP creds? OIDC token? Kerberos)?
18. Is a **Prometheus reporting task** or equivalent metrics scrape endpoint enabled?
19. Is the **provenance repository** enabled and queryable via the API? What retention?
20. Rough throughput per top flow: records/s, MB/s. Peak vs steady-state.
21. Known backpressure hotspots or SLA-critical flows.
22. Latency SLAs on any flow (end-to-end)?

### D. Data & security constraints
23. ⭐ **Can flow definitions (JSON) leave the customer environment** — even to the engineer's laptop? Even to a hosted LLM API?
24. Do flow properties contain secrets, PII, or PHI in clear text (a common anti-pattern)? If so, we need redaction before LLM calls.
25. Is the environment **air-gapped**, DMZ, or general egress-permitted?
26. Compliance regime (SOX, HIPAA, PCI, FedRAMP, ITAR)?
27. Secrets management integration (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, CyberArk)?

### E. Network reachability from the Docker tool (laptop-hosted)
28. ⭐ Is the source NiFi API reachable from an engineer laptop directly, via VPN, via bastion / jump host, or only from inside the datacenter?
29. Any HTTP(S) proxy required for outbound traffic (LLM API calls)?
30. Custom CA / trust store required for the source NiFi's TLS cert?
31. Is the **target CDP NiFi 2** cluster reachable from the same laptop, or only from a landing-zone workstation?

### F. LLM & AI infrastructure
_(v1 assumption: customer stands up a **Cloudera AI Inference (CAI)** endpoint reachable from the Docker container. These questions confirm that and cover fallbacks.)_

32. ⭐ Is a **Cloudera AI Inference (CAI)** deployment available or planned in this CDP environment? Which model(s) can we serve on it (Claude via partner endpoint, Llama 3.x, Mistral, other)?
33. If CAI is not available in time, what is the fallback? Options to probe:
    - **Anthropic Claude via AWS Bedrock** (common enterprise pattern; Cloudera-partner-friendly)
    - **Anthropic Claude direct API**
    - **On-prem open models** (Llama 3.x, Mistral, Qwen) via vLLM / TGI
34. Is the Docker container's outbound path to the CAI endpoint (or fallback LLM) network-reachable from the engineer's laptop, or must the tool run inside the customer's network?
35. Is there an existing enterprise LLM gateway (LiteLLM, Portkey, custom) in front of these endpoints?
36. Hard token/cost caps or rate limits per user / per day?
37. Data residency requirements for LLM traffic (region pinning)?
38. Approved embeddings model + where it is served (CAI, provider-native, on-prem)?

### G. Target CDP environment
_(v1 emits flow JSON files for manual import — questions here are lighter, but we still need to know what "correct output" looks like.)_

39. ⭐ CDP form-factor: **Public Cloud** (with **Cloudera DataFlow / CDF**), **Private Cloud Base + CFM**, or **Private Cloud Data Services**?
40. Target NiFi 2 version and CFM version (this pins the schema our output must match)?
41. Will flows eventually be deployed as **DataFlow Deployments** (managed) or into a customer-managed NiFi 2 cluster? (Informs v1.5 planning.)
42. Object storage for staging assets (S3, ADLS Gen2, GCS, Ozone)?
43. Environment / workload identity model (Machine Users, Workload Passwords, FreeIPA)?

### H. Ontology / migration-guideline source of truth
44. Does the customer have an **internal NiFi style guide** (naming, PG structure, error-handling patterns, secrets patterns)?
45. Is there an **approved processor allowlist / denylist**?
46. Any known internal "gotchas" from past NiFi upgrades?
47. Are they open to us seeding the vector DB with the **Cloudera-published NiFi 1→2 migration guide** + Apache NiFi release notes + our curated breaking-change catalog?

### I. Human-in-the-loop workflow & governance
48. Who approves migration proposals (individual SMEs, a review board, a ticket queue)?
49. Does the tool UI need SSO (SAML, OIDC)? (v1: local auth is likely fine on a laptop.)
50. Audit trail requirements — who approved what, when, with what rationale?
51. Integration with change-management tooling (ServiceNow, Jira Service Management)?
52. Is a signed-off audit log per flow a deliverable of the tool?

### J. Success criteria & pilot scope
53. ⭐ How many flows in the pilot? Which ones (start with medium-complexity, non-critical)?
54. Overall migration timeline & wave plan?
55. **v1 definition of "migrated": approved NiFi 2 flow JSON file, imported cleanly on the target and starting without validation errors.** Shadow-run & cutover are out of scope for v1 — confirm the customer agrees.
56. Test data available for later shadow-running (informs v1.5, not v1)?
57. Cutover strategy: parallel run + diff, big-bang, blue/green? (Informs v2.)

---

## 3. Proposed architecture (to validate against discovery answers)

### 3.1 High-level flow (v1)

```
[Source NiFi 1 REST API] ──► [Ingestor]                (v1)
[NiFi Registry (if used)] ─►    │
                                 ▼
                        [Local flow repo]  ◄── [Ontology / rules
                             (SQLite +          + breaking-change
                              flow JSONs)        catalog seeds]
                                 │                      │
                                 ▼                      ▼
                          [Chunker/Analyzer] ──► [Vector DB (Chroma)]
                                 │                      ▲
                                 ▼                      │
                        [LangGraph Agent] ──── RAG ─────┘
                        (plan → propose →
                         critique → refine)
                                 │
                              (LLM calls → CAI endpoint, via
                               provider-abstract client)
                                 │
                                 ▼
                        [HITL UI: FastAPI + React]
                                 │  approve / reject / "improve"
                                 ▼
                        [NiFi 2 flow JSON export]      ◄── v1 stops here
                                 │
                                 ▼
                (v1.5) [CDP DataFlow deploy API]
                                 │
                                 ▼
                (v2) [Shadow-run + diff harness]
```

**Flow-definition source selection (Registry vs live canvas):**
The Ingestor supports both paths, config-driven per flow. Discovery Q6 tells us which the customer actually uses:
- Registry in use and authoritative → prefer `nifi-registry-api/buckets/{id}/flows/{flowId}/versions/{v}`. Stable, versioned artifact with commit history.
- Registry absent or half-adopted → pull the live canvas via `/nifi-api/process-groups/{id}/download`. Snapshot-in-time; document the timestamp.
- Registry exists but not every flow is committed → per-flow selection; the ingest report flags "unversioned" flows.

### 3.2 Component picks (defaults; each has an alternative)

| Component | Recommended | Alternative | Reason |
|---|---|---|---|
| Agent framework | **LangGraph** | Vanilla LangChain, custom | HITL state machines are its sweet spot |
| LLM serving | **Cloudera AI Inference (CAI) endpoint**, called via a provider-abstract client (OpenAI-compatible or Anthropic-compatible interface) | Bedrock Claude, Anthropic direct, on-prem vLLM | Aligns with CDP posture; keep the client thin so we can swap |
| Embeddings | Served alongside on CAI if possible | Voyage / BGE-large / provider-native | Must sit on the same egress path as the LLM |
| Vector DB | **Chroma** (embedded in container) | Qdrant, Milvus, PGVector | Zero-ops for a laptop-scale MVP; swap later |
| Structured store | SQLite | Postgres in a sidecar | Laptop-scale, single file, easy to ship |
| UI | **FastAPI + React** | Streamlit for MVP | HITL diff-review needs rich UI; Streamlit is fine for week-1 |
| Packaging | **docker-compose** (agent, UI, vector DB) | Single monolithic image | Cleaner separation; still one `docker compose up` for the customer |
| Auth to source | mTLS / OIDC / LDAP / Kerberos — config-driven | — | Must match customer's NiFi auth |
| Secrets in tool | Docker secrets + `.env` | HashiCorp Vault client | Laptop MVP keeps it simple |

### 3.3 Key design decisions that discovery must lock

- **CAI reachability**: is the CAI endpoint reachable from the Docker container on an engineer's laptop, or must the tool itself run inside the customer network? Answer to Q32/Q34 decides packaging.
- **Air-gapped mode**: if yes, we need an on-prem model path and offline embeddings. Fork in the road, not a small config flag.
- **v1 scope is definition conversion only** — no deploy, no shadow-run. v1.5 adds the CDP DataFlow deploy API; v2 adds shadow-run.
- **Redaction pipeline**: before any flow JSON hits the LLM (even one served on CAI), run a pass that replaces sensitive property values and known credential patterns with tokens. Non-negotiable if answer to Q24 is "yes."
- **Ontology seed content**: pre-load Apache NiFi 2.0 release notes, Cloudera CFM migration guide, and a curated breaking-change table. Customer-specific style-guide docs get layered on top.

### 3.4 NiFi 1 → NiFi 2 breaking-change categories the agent must handle

These are patterns to encode as ontology rules (not exhaustive — verify against current Apache release notes as part of ontology seeding):
- **Variables → Parameter Contexts** (Variables removed).
- **Java 8/11 → Java 21** (all custom NARs must be recompiled).
- **Removed / repackaged processors** (several JMS, HBase, older Elasticsearch, some deprecated HTTP context handlers, etc.).
- **Sensitive property algorithm default changes** (`nifi.sensitive.props.algorithm`).
- **Flow definition schema changes** (`flow.json.gz` layout evolved).
- **Registry compatibility** (some 1.x Registry buckets need re-import).
- **Removed Expression Language functions** and behavioral changes on a few (`getStateValue`, some date functions).
- **Site-to-Site protocol changes** where present.
- **Reporting task API changes** for custom reporting tasks.

---

## 4. v1 success criteria

For each pilot flow (v1 scope: **definition conversion only**):
1. **Ingest**: tool fetches definition (Registry-preferred, live-canvas fallback) + last-30-days runtime status from source API; artifact stored locally.
2. **Analyze**: agent produces a diff report — what changes, why, referencing ontology rules.
3. **HITL**: engineer approves, rejects, or requests improvement; every decision + rationale is logged.
4. **Export**: approved flow exports as a NiFi 2-compatible flow definition JSON.
5. **Round-trip test** (manual by customer in v1): imported into a target CDP DataFlow / CFM 2.x cluster; starts without validation errors.
6. **Audit**: full trail exportable as PDF/JSON for the customer's change-management process.

_Deferred to v1.5+: automated deploy via CDP API, shadow-run, cutover tooling._

**Definition of done for v1:** 3–5 representative flows successfully converted, HITL-approved, manually imported into the target cluster by the customer without validation errors, with an audit trail the customer accepts.

---

## 5. Working assumptions (locked at time of writing)

- **v1 scope**: definition conversion only. Deploy, shadow-run, cutover are follow-on releases.
- **LLM**: assume the Docker container reaches a **Cloudera AI Inference (CAI)** endpoint; provider-abstract client keeps Bedrock / Anthropic direct / on-prem vLLM as trivial swaps if CAI isn't in place in time.
- **Flow source**: support both **NiFi Registry** and **live canvas** ingestion paths, config-driven per flow; discovery Q6 tells us which is primary at this customer.

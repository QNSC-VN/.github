# QNSC Platform Architecture

> **Status:** Target architecture (v3, revised 2026-07-01). Adopted incrementally — see [Phasing](#13-phasing--migration).
> **Audience:** Engineering, platform/DevOps, security, leadership.
> **Owner:** Platform team.
> **Related:** [TECH_STACK.md](./TECH_STACK.md) — tool choices and decision records. · [REPOSITORY_STRUCTURE.md](./REPOSITORY_STRUCTURE.md) — repo topology and migration.
> **Changes in v3:** Compute platform **re-sequenced** — ECS Fargate is the **deliberate starting platform** for Zone B (right for QNSC's current stage), and EKS is a **trigger-driven destination**, not a committed near-term migration. Added explicit EKS-migration triggers (§13.2), Zone C as the natural forcing function, and serverless (Lambda) placement.
> **Changes in v2:** Multi-account landing zone hardened to the AWS standard core-account model; Zone B sub-tiered (external SaaS vs internal); first-class security/compliance guardrails; data tenancy, DR, residency; zone-aware ArgoCD; phasing re-sequenced around the **signed IC/ITAR contract**.
> **Decision (2026-07-01):** **ECS Fargate is the starting platform for Zone B.** It is the correct choice for QNSC's current stage (small team, few live products) — no control-plane fee, no cluster to run. **EKS is the strategic destination, adopted only when a concrete trigger fires** (a staffed platform team, or Zone C's GPU/HIPAA needs, or real scale) — see [§8](#8-compute-platform-decision--ecs-fargate-now-eks-when-triggered) and [§13](#13-phasing--migration). **Full serverless (Lambda-only) is not the path** — Lambda is for event-driven glue, not the primary app runtime.

---

## 1. Purpose

QNSC builds a portfolio of products with **very different regulatory and trust requirements** — from commercial SaaS to military chip design to healthcare AI. This document defines how those products are deployed and isolated on AWS, so that:

- Each product lives in the **right trust zone** for its data sensitivity.
- We operate the **fewest platforms possible** without compromising compliance.
- Adding a new product (large or small) is **cheap and standardized**.
- Compliance evidence (SOC 2, ISO 27001, HIPAA, ITAR) falls out of the structure, not bolted on after.

The single organizing principle:

> **Products are grouped by the regulatory sensitivity of their data — not by size, and not by internal-vs-external.**

---

## 2. Project portfolio

| Project | Type | Users | Data sensitivity | Status |
|---|---|---|---|---|
| **IC** | 28nm chip design (EDA) | Internal chip engineers | **Military / ITAR-controlled** | **Contract signed — active** |
| **opshub** | Internal ops (employees, assets, requests) | Staff | Ordinary corporate | Live (ECS) |
| **Rally** | Multi-tenant SaaS | External customers | Standard commercial | Live (ECS) |
| **Learning** | Udemy/Coursera-style SaaS | External customers | Standard commercial | Roadmap |
| **Knowledge base** | SaaS | External customers | Standard commercial | Roadmap |
| **Hospital Camera AI** | AI + SaaS | Hospitals | **PHI / HIPAA-class** | Roadmap (future) |
| _Future small projects_ | Tools, sites, APIs, jobs | Varies | Usually ordinary commercial | Ongoing |

**Note:** `opshub` is *internal* but holds *ordinary* data, so it lives in the commercial zone. "Internal vs external" is **not** the dividing line — **data sensitivity is.**

---

## 3. The three trust zones

```
QNSC
│
├── ZONE A — IC (military chip design) ───────────── NOT Kubernetes · SIGNED, ACTIVE
│     EDA/HPC workloads. ITAR/EAR controlled. Separate AWS GovCloud org / on-prem.
│     ZERO shared infrastructure, identity, or pipeline with B/C.
│
├── ZONE B — Commercial platform ──────────────────  ECS Fargate now → EKS when triggered (§13.2)
│   ├── B1  external customer-facing SaaS:  rally · learning · knowledge-base
│   └── B2  internal:                       opshub · small tools/jobs
│     Hard isolation between B1 and B2 (separate services + SGs on ECS; node pools on EKS).
│
└── ZONE C — Hospital Camera AI (PHI) ─────────────  Isolated EKS + GPU (future — the EKS forcing function)
      Own accounts, HIPAA guardrails, GPU node pools, AWS BAA.
```

### Decision rule for any new project

```
What data does it handle?
├── Military / defense / ITAR ............ Zone A (separate GovCloud/on-prem org, non-K8s)
├── Patient health data (PHI) ............ Zone C (isolated medical EKS)
└── Everything else (commercial/internal)  Zone B
        ├── faces external customers? ..... B1 node pool (external tier)
        └── internal only? ................ B2 node pool (internal tier)
```

Size never decides the zone. Size only decides the **resource quota** within a zone.

---

## 4. Foundation: AWS Organization & accounts

A hardened multi-account landing zone (AWS Control Tower-style). The management/root account is kept **near-empty** — it holds governance only, never workloads. Shared tooling lives in its own account so a tooling compromise never sits in the org root.

```
AWS Organization (qnsc) — commercial + medical
│
├── CORE (governance & shared services)
│   ├── management        Organizations root · SCPs · IAM Identity Center (SSO). NO workloads.
│   ├── log-archive        Immutable central CloudTrail + Config + access logs (write-once).
│   ├── security/audit     GuardDuty, Security Hub, Config aggregator, IAM Access Analyzer (read-only org view).
│   └── shared-services    Shared ECR · ArgoCD hub (commercial) · Terraform/OpenTofu state · CI runners.
│
├── ZONE B — commercial:   qnsc-dev · qnsc-staging · qnsc-prod
├── ZONE C — medical:      qnsc-med-dev · qnsc-med-staging · qnsc-med-prod   (PHI-isolated)
└── ZONE A — IC:           SEPARATE GovCloud org / on-prem — NOT in this tree, NOT federated here.
```

**Why this shape:**
- **Account-per-environment** gives hard blast-radius and IAM isolation between dev and prod — a bad dev change cannot reach prod.
- **Empty management account** is the AWS-recommended posture; workloads (incl. ECR, ArgoCD, state) in the org root is a standard audit finding and a single point of total compromise.
- **Dedicated log-archive** account gives tamper-evident, write-once logs that auditors and incident response require.
- **Dedicated security/audit** account centralizes detection (GuardDuty/Security Hub) with read-only reach across the org.

> Zone A's accounts are intentionally absent from this organization. ITAR personnel- and infrastructure-access controls require it to be a fully separate boundary. See [§5](#5-zone-a--ic-military-chip-design--signed-active).

---

## 5. Zone A — IC (military chip design) — SIGNED, ACTIVE

**Workload:** EDA tools (Cadence, Synopsys, Mentor) — large licensed applications on HPC/VMs, **not containers**.

**Why fully separate:**
- ITAR/EAR export control forbids sharing infrastructure with commercial/internet-facing systems and forbids access by non-authorized persons.
- EDA is HPC, not a Kubernetes workload.

**Deployment model:**
- **Separate AWS GovCloud (US) Organization** or **on-premises**, possibly **air-gapped**.
- Dedicated EC2/HPC compute + EDA license servers.
- High-performance shared storage (FSx for Lustre / NetApp).
- NICE DCV remote workstations for authorized engineers.

**Hard boundary (non-negotiable):** No shared ECR, no shared ArgoCD, no shared CI/CD, **no IAM Identity Center federation**, no VPC peering, no shared Terraform state with Zones B/C. Identity and access for Zone A are administered entirely within Zone A's own boundary.

> **Status:** Contract is **signed**. This is the **highest-priority near-term track**, but it is **architecturally independent** of the EKS platform below — building Zone A does not build or validate Zone B/C. Run it in parallel.
>
> **Action:** Treat as a **separate engagement**. Engage an ITAR/defense-cloud specialist for compliance, authorization-to-operate, and the export-control program (technology control plan, access screening). **Out of scope** for the EKS platform work; tracked separately.

---

## 6. Zone B — Commercial platform

The primary platform. Hosts all ordinary-data products, large and small. **Current runtime is ECS Fargate** — the right choice for QNSC's stage (see [§8](#8-compute-platform-decision--ecs-fargate-now-eks-when-triggered)). Rally and opshub run on ECS Fargate; new products (Learning, KB) are born on ECS Fargate too. **EKS is the destination when a trigger fires** (§13.2) — not a committed near-term migration.

### 6.1 Sub-tiering: external vs internal

Even within "commercial," external customer-facing workloads and internal tooling are **not** allowed to share an execution host. A container escape or noisy-neighbor must not cross from internal tooling into a customer-data plane (or vice-versa).

```
Zone B (per env)
├── B1 — external customer-facing:  rally · learning · knowledge-base
│        Dedicated node pool / capacity (tainted). Internet-facing via WAF + ALB.
├── B2 — internal:                  opshub · small tools/jobs
│        Dedicated node pool / capacity (tainted). Not internet-exposed by default.
└── platform addons (shared control plane):
        ArgoCD · Karpenter · External Secrets · AWS LB Controller · policy engine
```

- **On ECS (now):** B1 and B2 run as separate services with separate Fargate capacity; **security groups** enforce that B2 cannot reach B1 data stores and neither reaches the other's secrets.
- **On EKS (later):** B1 and B2 are separate **Karpenter NodePools** (taints + labels) so pods never co-locate on the same EC2 instance, plus per-product `Namespace` + `RBAC` + `ResourceQuota` + **default-deny** `NetworkPolicy`.

### 6.2 Isolation between products

`Namespace` (logical) + `RBAC` (who can deploy) + `ResourceQuota` (CPU/mem caps) + `NetworkPolicy` (who talks to whom, default-deny). Node-level separation only between **tiers** (B1/B2), not between every product — products within a tier may bin-pack.

### 6.3 Multi-tenant SaaS — tenant isolation (Rally, Learning, KB)

Product isolation (above) is **not** customer isolation. Each SaaS product must declare its tenant model; this drives the data design and the SOC 2 story.

| Model | Isolation | Cost | When |
|---|---|---|---|
| **Pooled** — shared app + DB, row-level (`tenant_id` / Postgres RLS) | Logical | Lowest | Default for many small/medium tenants |
| **Bridge** — shared app, schema- or DB-per-tenant | Stronger data isolation | Medium | Tenants needing data separation |
| **Siloed** — dedicated stack per tenant | Physical | Highest | Enterprise/regulated tenants on contract |

**Default for Rally:** pooled with enforced RLS and a `tenant_id` on every row and every query path, with the option to promote a large/regulated tenant to bridge or siloed. Document the chosen model in `rally-infra`.

### 6.4 Small / spiky projects

Live as namespaces (EKS) or lightweight services (ECS) with a small quota. For idle-most-of-the-time workloads (cron jobs, webhooks, low-traffic APIs), use **KEDA scale-to-zero** (EKS) or scheduled/min-zero tasks (ECS) so they cost nothing while idle.

---

## 7. Zone C — Hospital Camera AI (PHI) — future

Healthcare data demands isolation comparable to (though distinct from) defense data. This zone is **future**, triggered when the hospital AI contract is real. It is the **first production EKS + GPU workload** and therefore the natural pilot for the EKS platform module.

```
Own account-per-environment (isolated from Zone B):
  qnsc-med-dev  →  qnsc-med-staging  →  qnsc-med-prod

Each runs an isolated EKS cluster:
  EKS cluster (per env)
  ├── namespace: camera-ai        (inference services)
  ├── GPU NodePool (Karpenter)    (AI compute, e.g. G/P-family)
  ├── ArgoCD (in-zone instance)   (NOT the commercial hub — see §9)
  └── HIPAA guardrails:
        encryption everywhere (KMS) · audit logging · signed AWS BAA ·
        VPC isolation · stricter SCPs/NetworkPolicies · PHI never leaves the zone
```

**Why its own zone (not a Zone B namespace):**
- PHI compliance (HIPAA-class) requires a separated data plane.
- Hospital contracts commonly demand data isolation.
- GPU-heavy AI benefits from a cluster tuned for accelerated compute.

**Key point:** Zone C uses the **same EKS platform module as Zone B**, deployed into isolated medical accounts with HIPAA guardrails and GPU node pools. *Build the module once, deploy it (eventually) into both zones.* Because Zone C is the first place EKS is genuinely required, **build and harden the module here first**, then backport to Zone B.

---

## 8. Compute platform decision — ECS Fargate now, EKS when triggered

**Decision: Zone B starts and runs on ECS Fargate. EKS is the strategic destination, adopted only when a concrete trigger fires (see [§13.2](#132-eks-migration-triggers-adopt-when-any-fires)) — not on a fixed date.** New products are born on ECS Fargate too, until the platform crosses to EKS. **Full serverless (Lambda-only) is explicitly not the path** — see *Serverless* below.

**The honest trade — same table, read at QNSC's current stage.** EKS wins most axes *at scale and with a platform team*; ECS wins the one axis that dominates *today* (few products, ~8 staff, no dedicated SRE):

| Axis | ECS Fargate | EKS (+ Karpenter) | Wins at scale | Wins now |
|---|---|---|---|---|
| Cost at scale | Per-task billing, no bin-packing — pay for requested vCPU/mem even when idle | Karpenter bin-packs + consolidation + broad Spot → ~30–50% lower compute at steady scale | **EKS** | ~tie (dev-only ≈ $0 both) |
| Performance | Micro-VM/task, ~30–60s cold start, cap 16 vCPU / 120 GB, no GPU | Warm nodes, any family incl. GPU/Graviton, no size cap | **EKS** | ECS enough |
| Scaling | Task-level target-tracking autoscale | HPA + VPA + Karpenter + KEDA (event-driven + scale-to-zero) | **EKS** | ECS enough |
| Portability | AWS-proprietary | Standard K8s — transfers to GovCloud/on-prem, multi-region | **EKS** | not needed yet |
| Ecosystem | Smaller, tight AWS integration | ArgoCD, Helm, Kyverno, KEDA, mesh, large community | **EKS** | — |
| **Ops simplicity / no team** | **AWS runs the data plane; no control-plane fee, no upgrades, no node patching** | You own upgrades (~3×/yr), add-ons, CVEs, RBAC, a node fleet | ECS | **ECS** |

At QNSC's current stage the decisive axis is the last one. With no dedicated platform engineer, an under-resourced EKS converts its theoretical cost/scaling wins into **real operational incidents.** ECS Fargate has **no $73/mo/cluster control-plane fee, no cluster upgrades, no node fleet to patch** — AWS runs the data plane. Running Zone B on ECS Fargate now is **not technical debt; it is the correct choice for the stage.**

**Cost reality (ap-southeast-1):**
- ECS control plane **$0** vs EKS **~$73/mo per cluster** — a fixed tax ECS never charges (×N if clusters split per env).
- At today's dev-only scale both are **~$50–150/mo**; cost is **not** the deciding factor now.
- At steady multi-product scale EKS + Karpenter + Spot + bin-packing runs **~30–50% cheaper on compute** — but that saving is **dwarfed by the platform-engineer salary** EKS requires (one SRE ≫ the entire monthly AWS bill). The **people cost, not the compute rate, is the real EKS cost.**

**The forcing function is Zone C, not the calendar.** Zone C (hospital AI) needs **GPU scheduling + HIPAA isolation** — the first workload ECS genuinely cannot serve well, and the first place EKS *must* exist. Build and harden the EKS platform module **there** (§7); Zone B then migrates **opportunistically** because the module and the team already exist — not because ECS failed.

**Serverless (Lambda) — a placement, not the platform.** Lambda is for **event-driven glue**: cron jobs, the dev-scheduler, webhooks, light async/S3-event processing. Those stay Lambda even after EKS arrives. QNSC does **not** adopt full serverless (Lambda + API Gateway as the primary app runtime) because (a) it is a *different* architecture, not a step toward EKS — going there **abandons** the K8s destination and the ECS→EKS carryover; (b) cold starts, the 15-minute cap, and VPC-attach latency hurt always-on SaaS APIs. **Rule: containers on Fargate for services; Lambda for glue.**

**The hard prerequisite still holds:** migrating **live products** (Rally, opshub) to EKS does not begin until a **staffed platform team** (2–3 engineers with real production EKS day-2 experience) is in place. Authoring modules can start earlier; live-product cutover cannot.

---

## 9. Shared platform model (3 layers)

The 3-layer model is **platform-agnostic** and carries from ECS to EKS unchanged:

| Layer | Repos | ECS role (now) | EKS role (later) |
|---|---|---|---|
| **Shared platform** | `qnsc-infra`, `qnsc-tf-modules` | Landing zone + ECS/network/data modules | + EKS platform module |
| **Shared delivery** | `qnsc-ci` (now) → `qnsc-gitops` (later) | `qnsc-ci`: reusable GHA — build, scan, sign, **push-deploy** to ECS | `qnsc-gitops`: ArgoCD ApplicationSets + shared Helm base — **pull-based** |
| **Per-product** | `rally`, `opshub`, … (monorepos) | `deploy/ecs/` task-def + service (via `infra/`) | Per-product Helm `chart/` + namespace + ArgoCD Application |

> **Delivery repo by phase:** In the **ECS phase (now)**, reusable CI/CD lives in **`qnsc-ci`** and deploys are push-based (`update-service`). **`qnsc-gitops` is created only when the EKS trigger fires** — it is the pull-based ArgoCD config hub, not a CI actions library. (See [REPOSITORY_STRUCTURE.md §4](./REPOSITORY_STRUCTURE.md#4-qnsc-gitops-layout-argocd-config-hub--eks-phase).)

**Modules to build:**
1. **Landing-zone module** — Organizations accounts, the core-account layout (§4), SCPs, IAM Identity Center (SSO).
2. **EKS platform module** — cluster + Karpenter + ArgoCD + External Secrets Operator + AWS Load Balancer Controller + policy engine. Built/hardened in Zone C first, then reused in Zone B.

**ArgoCD topology (zone-aware):**
- **Commercial hub** (in `shared-services`) manages Zone B clusters only.
- **Zone C runs its own in-zone ArgoCD instance.** The commercial hub must **not** reach into the PHI data plane — a hub that spans zones pierces the isolation the zones exist to provide.
- **Zone A has no ArgoCD reach at all.**

**Deploy flow:**
- **ECS (now):** push to `main` → GitHub Actions builds, pushes to ECR, deploys to ECS, verifies, runs migrations.
- **EKS (later):** push to `main` → ArgoCD syncs into the product's namespace in the dev cluster; tag a release → promote staging → prod. Declarative, auditable, self-healing.

---

## 10. Security & compliance guardrails

First-class, not implied. These apply org-wide (Zones B/C); Zone A runs an equivalent program inside its own boundary.

**Preventive (SCPs at the org/OU level):**
- Deny leaving approved regions (data residency).
- Deny disabling CloudTrail, Config, GuardDuty, or Security Hub.
- Deny root-user actions; deny creating IAM users (use SSO + roles).
- Require IMDSv2; deny public S3 / public AMI sharing.
- Medical OU: tighter SCPs (deny any service not in the HIPAA-eligible set).

**Detective:**
- **CloudTrail** (org trail) → write-once `log-archive`.
- **GuardDuty** + **Security Hub** + **AWS Config** conformance packs, aggregated in `security/audit`.
- **IAM Access Analyzer** for unintended external access.

**Workload identity & secrets:**
- **IRSA / EKS Pod Identity** for pod-to-AWS IAM (no static keys).
- **External Secrets Operator** syncing from AWS Secrets Manager / Parameter Store.
- **KMS** encryption at rest everywhere; TLS in transit everywhere.

**Supply chain & runtime:**
- **Image signing + attestation** (the `attest-image` action already exists — keep and enforce verification at deploy/admission).
- ECR image scanning; block deploy on critical CVEs.
- **Policy engine** (Kyverno or OPA Gatekeeper) on EKS: enforce non-root, no-privileged, required labels, default-deny NetworkPolicy, signed-images-only.

**Compliance mapping:**
- Zone B → SOC 2 / ISO 27001.
- Zone C → HIPAA (signed **AWS BAA**, PHI isolation, audit logging, encryption).
- Zone A → ITAR/EAR (separate boundary, GovCloud, technology control plan).

---

## 11. Data: tenancy, DR, residency

**Tenancy:** see [§6.3](#63-multi-tenant-saas--tenant-isolation-rally-learning-kb) for the SaaS tenant model.

**Availability & DR (state targets per zone):**

| Zone | Baseline | Prod DR target (RTO / RPO) |
|---|---|---|
| B (commercial) | Multi-AZ in one region | RTO ≤ 4h / RPO ≤ 15m via Multi-AZ RDS + automated + cross-region backups |
| C (medical) | Multi-AZ, isolated region | RTO ≤ 2h / RPO ≤ 5m; cross-region encrypted backups; tighter because PHI |
| A (IC) | Per ITAR engagement | Defined in the separate Zone A program |

- **Single region, multi-AZ** is the baseline. **Active-passive multi-region** only where contracts/RTO demand it (revisit per product, not globally).
- **Backups:** automated RDS snapshots + cross-region copy for prod and medical; periodic restore tests (a backup that has never been restored is not a backup).
- **State/control-plane during a region outage:** Terraform state and ArgoCD config are backed up and re-creatable; document the recovery runbook.

**Residency:** data stays in approved regions, enforced by SCP. PHI never leaves Zone C. If EU/sovereignty customers arrive, add a region within the same module — do not bolt on later.

---

## 12. Networking & cross-zone boundaries

- **Default-deny** between products (NetworkPolicy on EKS; security groups on ECS).
- **No cross-zone data path.** Zone B ↔ Zone C only via explicit, audited, least-privilege API calls if ever required — never shared data stores. Zone A is air-gapped from both.
- **Ingress:** WAF + ALB for B1 (external) and Zone C public surfaces; B2 internal services are not internet-exposed by default.
- **Egress:** controlled via NAT + egress allow-lists for regulated zones.

---

## 13. Phasing & migration

A **destination reached by triggers, not by calendar.** Zone B runs on **ECS Fargate as its production platform today**; EKS is adopted when a concrete condition fires (§13.2) — most likely Zone C. Because images are already containers and data never moves, the eventual migration is low-risk and incremental (§13.3). Zone A (signed IC/ITAR) runs as an independent parallel track throughout.

### 13.1 Phases

| Phase | Work | Trigger / status |
|---|---|---|
| **0 — now (ECS Fargate is the platform)** | (a) **Kick off Zone A ITAR engagement** (GovCloud/on-prem, specialist) — top priority, parallel track. (b) **Harden the landing zone**: core accounts (§4), SCPs, GuardDuty/Security Hub/CloudTrail→log-archive. (c) Build/run **Zone B on ECS Fargate**: shared ALB + host/path routing, capacity providers (Fargate + Fargate Spot), CI push-deploy, Secrets Manager injection, CloudWatch/ADOT observability. (d) New products (Learning, KB) **born on ECS Fargate**. | Current platform. IC signed |
| **1 — well-architected ECS** | Mature the ECS platform: **blue/green** deploys (CodeDeploy) + instant rollback, service auto-scaling (target tracking), **B1/B2** service + security-group separation, cost tags + Budgets, dev-scheduler off-hours, **supply-chain gates in CI** (sign + scan, block unsigned/critical-CVE before `update-service`). | Ongoing on ECS |
| **2 — EKS trigger fires** | When any trigger in §13.2 fires (most likely **Zone C** or **platform team hired**): author + harden the **EKS platform module** (cluster + Karpenter + ArgoCD + External Secrets + ALB Controller + Kyverno). If the trigger is Zone C, **build it in Zone C first** (§7), then reuse for Zone B. | Trigger-gated (§13.2) |
| **3 — migrate Zone B (opportunistic)** | Once the EKS module + platform team exist: migrate **opshub first** (internal, lower blast radius), then **Rally** (external) — service-by-service, workers before user-facing APIs, parallel-run + traffic-shift + instant rollback (§13.3). New products may start on EKS from here. | After module proven + team staffed |
| **4 — retire ECS** | Decommission ECS services and push-based deploy actions once everything is green on EKS. Rally tenancy model enforced. | After all services migrated |
| **5 — steady state** | One EKS platform across Zone B + Zone C (isolated, own accounts + in-zone ArgoCD + GPU). ECS gone. | End state |

### 13.2 EKS migration triggers (adopt when ANY fires)

EKS is not scheduled — it is **triggered.** Adopt it when any one of these becomes true; until then, ECS Fargate is the correct platform.

| Trigger | Threshold | Why EKS becomes the right call |
|---|---|---|
| **Platform team staffed** | 2–3 engineers with real production K8s day-2 experience hired | Removes the one axis ECS wins (ops-without-a-team). Hard prerequisite for live-product cutover regardless of other triggers |
| **Zone C goes real** | Hospital AI contract signed | **GPU scheduling + HIPAA isolation** — first workload ECS can't serve well; EKS *must* exist. The most likely forcing function |
| **Service sprawl** | > ~15–20 services across the portfolio | ECS service management + lack of bin-packing gets costly and unwieldy at this count |
| **Compute cost + low utilization** | Fargate bill material **and** average utilization < ~40% | Karpenter EC2 bin-packing saves ~30–50% — enough to fund the platform work |
| **Advanced platform needs** | mTLS service mesh, hard multi-tenant isolation, event-driven scale-to-zero (KEDA), GPU | ECS lacks these; EKS is purpose-built for them |
| **Portability** | Must run the same workloads on GovCloud / on-prem | Standard K8s transfers; ECS is AWS-proprietary |

**Most likely path:** Zone C forces EKS into existence → build/harden the module there → Zone B migrates opportunistically because the module + team now exist — **not** because ECS failed. If none of these fire, **staying on ECS Fargate indefinitely is a valid steady state.**

### 13.3 Migration mechanics (ECS → EKS)

Low-risk because **only the compute tier moves — not the data**, and the images are already containers. What carries over vs. what is rebuilt:

| Carries over 1:1 (no change) | Rebuilt at migration |
|---|---|
| Docker **images** (same artifact runs as task or pod) | ECS **task definition** → K8s **Deployment + HPA** (Helm chart) |
| **OpenTofu** + all VPC / RDS / ElastiCache / IAM / ECR modules | ECS **service** → K8s **Service + Deployment** |
| **ECR** registry · **GitHub Actions** build/test/scan/sign | ALB target-group wiring → **AWS LB Controller + Ingress** |
| **Secrets Manager** · **Cloudflare** edge · **data layer** (untouched) | Push CD (`update-service`) → **pull CD (ArgoCD)** |
| Observability destinations (**AMP/AMG/X-Ray**) | Add: Karpenter · ESO · Kyverno admission · KEDA · NetworkPolicy |

- **Data stores stay put.** RDS, ElastiCache, S3, SQS/SNS untouched; EKS pods hit the same endpoints. No data migration.
- **Secrets** move from ECS task injection to **External Secrets Operator** (same Secrets Manager source).
- **Per service:** write Helm chart → deploy to EKS **alongside** the running ECS service → **shift traffic gradually** (weighted target group / DNS) → observe SLOs → roll forward or **instant rollback** to ECS. Never a flag-day.
- **Order:** opshub before Rally; within a product, background workers/cron before the user-facing API.

**Why this sequencing:** running products keep serving on ECS with zero rewrite throughout. Zone A (signed, urgent) does not wait on any of this. Live-product cutover begins only after the platform team is in place and the EKS module is proven in production.

---

## 14. Why this is the right enterprise approach

- **Stage-appropriate compute:** ECS Fargate now — no control-plane fee, no cluster to run, no platform-team salary before there is scale to justify it. EKS is adopted on **triggers** (§13.2), not on hype.
- **Compliance-aligned:** zones map to what SOC 2 / ISO 27001 / HIPAA / ITAR assessors expect — isolation by data sensitivity, hardened landing zone, central tamper-evident logging. This holds identically on ECS and EKS.
- **Cheap optionality:** because images are containers, data never moves, and IaC/CI/edge/secrets all carry over (§13.3), the ECS→EKS move is a low-risk, incremental option — bought now, exercised only when a trigger fires.
- **Right-sequenced:** the signed IC work leads (separate track); Zone B ships on ECS Fargate today; EKS is built where it's first genuinely required (Zone C) and reused for Zone B, not stood up speculatively.
- **Cost-honest:** at current scale ECS ≈ EKS on the bill; at large scale EKS compute is cheaper but the platform-team salary dominates. TCO is judged on **compute *and* people** — which is exactly why EKS waits for a trigger.
- **Isolation that holds:** external vs internal separation (services + SGs now, node pools later), zone-aware ArgoCD, default-deny networking, and an empty management account keep a single compromise contained.
- **Right tool per workload:** Fargate for always-on services, Lambda for event-glue, HPC/GovCloud for IC, EKS+GPU for medical AI. Not dogmatic.

### Caveats (read these)

1. **ECS Fargate is a deliberate choice, not a stopgap.** If no trigger in §13.2 ever fires, staying on ECS indefinitely is valid. Do not migrate to EKS for its own sake.
2. **EKS is only as good as the team running it.** The platform-team hire (§8) is the hard prerequisite for any live-product cutover — under-resourced EKS is the main failure mode. Zone A compliance is its own specialist engagement on top.
3. **The design assumes the stated roadmap** (6 products, AI/GPU). If it shrinks materially, the EKS triggers simply never fire — and that's fine.
4. **Migration, when it happens, is incremental, never flag-day.** ECS keeps serving each product until that product is green on EKS.

---

## 15. Quick reference

| Question | Answer |
|---|---|
| Where does a new commercial product go? | Zone B — external → B1, internal → B2 |
| Where does a small tool / site / job go? | Zone B, small quota (KEDA/min-zero if spiky) |
| Where does anything touching patient data go? | Zone C — isolated medical EKS |
| Where does defense/chip work go? | Zone A — separate GovCloud/on-prem, non-K8s (signed, active) |
| One cluster for everything? | No. One ECS cluster **per env** now; one EKS cluster **per env per zone** later; products are services/namespaces; external/internal split at service+SG (ECS) or node level (EKS) |
| ECS or EKS? | **ECS Fargate now** — the right platform for QNSC's stage. **EKS is trigger-driven** (§13.2), most likely forced by Zone C. Not a scheduled migration |
| What triggers EKS? | Any of: platform team hired · Zone C (GPU/HIPAA) real · >~15–20 services · Fargate cost high + util low · mesh/GPU/KEDA needs · GovCloud portability (§13.2) |
| Serverless (Lambda)? | **Glue only** — cron, dev-scheduler, webhooks, light async. **Not** the primary app runtime; full-serverless would abandon the EKS path |
| What gates live-product migration? | A staffed platform team (2–3 with real EKS day-2 experience). Cutover waits for it |
| Does size decide the zone? | No. **Data sensitivity** decides the zone; size decides the quota |
| What's in the management account? | Governance only (Org, SCPs, SSO). No workloads, no ECR, no ArgoCD |
| Does the commercial ArgoCD manage the medical cluster? | No. Zone C has its own in-zone ArgoCD. Zone A has none |

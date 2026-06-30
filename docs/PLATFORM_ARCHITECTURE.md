# QNSC Platform Architecture

> **Status:** Target architecture (decided 2026-06-30). Adopted incrementally — see [Phasing](#7-phasing--migration).
> **Audience:** Engineering, platform/DevOps, security, leadership.
> **Owner:** Platform team.

---

## 1. Purpose

QNSC builds a portfolio of products with **very different regulatory and trust requirements** — from commercial SaaS to military chip design to healthcare AI. This document defines how those products are deployed and isolated on AWS, so that:

- Each product lives in the **right trust zone** for its data sensitivity.
- We operate the **fewest platforms possible** without compromising compliance.
- Adding a new product (large or small) is **cheap and standardized**.

The single organizing principle:

> **Products are grouped by the regulatory sensitivity of their data — not by size, and not by internal-vs-external.**

---

## 2. Project portfolio

| Project | Type | Users | Data sensitivity |
|---|---|---|---|
| **IC** | 28nm chip design (EDA) | Internal chip engineers | **Military / ITAR-controlled** |
| **opshub** | Internal ops (employees, assets, requests) | Staff | Ordinary corporate |
| **Rally** | Multi-tenant SaaS | External customers | Standard commercial |
| **Learning** | Udemy/Coursera-style SaaS | External customers | Standard commercial |
| **Knowledge base** | SaaS | External customers | Standard commercial |
| **Hospital Camera AI** | AI + SaaS | Hospitals | **PHI / HIPAA-class** |
| _Future small projects_ | Tools, sites, APIs, jobs | Varies | Usually ordinary commercial |

**Note:** `opshub` is *internal* but holds *ordinary* data, so it lives in the commercial zone. "Internal vs external" is **not** the dividing line — **data sensitivity is.**

---

## 3. The three trust zones

```
QNSC
│
├── ZONE A — IC (military chip design) ───────────── NOT Kubernetes
│     EDA/HPC workloads. ITAR/EAR controlled. Fully separate.
│
├── ZONE B — Commercial SaaS platform ───────────── Shared EKS
│     rally · learning · knowledge-base · opshub · (small projects)
│     Each product = a namespace. One cluster per environment.
│
└── ZONE C — Hospital Camera AI (PHI) ─────────────  Isolated EKS + GPU
      Own accounts, HIPAA guardrails, GPU node pools.
```

### Decision rule for any new project

```
What data does it handle?
├── Military / defense / ITAR ............ Zone A (separate org, non-K8s)
├── Patient health data (PHI) ............ Zone C (isolated medical EKS)
└── Everything else (commercial/internal)  Zone B (shared EKS namespace)
```

Size never decides the zone. Size only decides the **resource quota** within a zone.

---

## 4. Zone A — IC (military chip design)

**Workload:** EDA tools (Cadence, Synopsys, Mentor) — large licensed applications on HPC/VMs, **not containers**.

**Why fully separate:**
- ITAR/EAR export control forbids sharing infrastructure with commercial/internet-facing systems.
- EDA is HPC, not a Kubernetes workload.

**Deployment model:**
- **Separate AWS Organization** (likely **AWS GovCloud**) or **on-premises**, possibly **air-gapped**.
- Dedicated EC2/HPC compute + license servers.
- High-performance shared storage (FSx for Lustre / NetApp).
- NICE DCV remote workstations for engineers.
- Zero shared infrastructure with Zones B/C.

> **Action:** Treat as a **separate engagement**. Engage an ITAR/defense-cloud specialist for compliance. This zone is **out of scope** for the EKS platform work below.

---

## 5. Zone B — Commercial SaaS platform

The primary platform. Hosts all ordinary-data products, large and small.

```
Account-per-environment (separate AWS accounts):
  qnsc-dev  →  qnsc-staging  →  qnsc-prod

Each account runs ONE EKS cluster. Products are namespaces:

  EKS cluster (per env)
  ├── namespace: rally           (multi-tenant SaaS)
  ├── namespace: learning        (SaaS)
  ├── namespace: knowledge-base  (SaaS)
  ├── namespace: opshub          (internal ops)
  ├── namespace: <small-project> (small quota)
  └── platform addons (shared):
        ArgoCD · Karpenter · External Secrets · AWS LB Controller
```

**Isolation between products:** Kubernetes `Namespace` + `RBAC` (who can deploy) + `ResourceQuota` (CPU/mem caps) + `NetworkPolicy` (who talks to whom).

**Why shared, not a cluster per product:**
- The cluster's fixed cost (~$73/mo control plane + ops time) is paid once, amortized across all products.
- Adding product #7, #8, #20 is "add a namespace" — near-zero marginal cost.
- One platform team operates a handful of clusters, not dozens.

**Small / spiky projects:** live as namespaces with a small `ResourceQuota`. For idle-most-of-the-time workloads (cron jobs, webhooks, low-traffic APIs), use **KEDA scale-to-zero** so they cost nothing while idle.

---

## 6. Zone C — Hospital Camera AI (PHI)

Healthcare data demands isolation comparable to (though distinct from) defense data.

```
Own account-per-environment (isolated from Zone B):
  qnsc-med-dev  →  qnsc-med-staging  →  qnsc-med-prod

Each runs an isolated EKS cluster:
  EKS cluster (per env)
  ├── namespace: camera-ai        (inference services)
  ├── GPU NodePool (Karpenter)    (AI compute)
  └── HIPAA guardrails:
        encryption everywhere · audit logging · AWS BAA ·
        VPC isolation · stricter SCPs/NetworkPolicies
```

**Why its own zone (not a Zone B namespace):**
- PHI compliance (HIPAA-class) requires a separated data plane.
- Hospital contracts commonly demand data isolation.
- GPU-heavy AI benefits from a cluster tuned for accelerated compute.

**Key point:** Zone C uses the **same EKS platform module as Zone B**, deployed into isolated medical accounts with HIPAA guardrails and GPU node pools. *Build the module once, deploy it twice.*

---

## 7. Foundation: AWS Organization & accounts

```
AWS Organization (qnsc)
│
├── qnsc-shared (management account)
│     • AWS Organizations root · IAM Identity Center (SSO)
│     • Terraform/OpenTofu remote state
│     • Shared ECR · ArgoCD hub · central logging
│
├── Zone B — commercial:   qnsc-dev · qnsc-staging · qnsc-prod
├── Zone C — medical:      qnsc-med-dev · qnsc-med-staging · qnsc-med-prod
└── Zone A — IC:           SEPARATE org / GovCloud (not in this tree)
```

**Account-per-environment** gives hard blast-radius and IAM isolation between dev and prod — the AWS-recommended landing-zone model. A bad dev change cannot reach prod.

---

## 8. Shared platform model (3 layers)

The existing 3-layer model is **platform-agnostic** and carries over from ECS to EKS unchanged:

| Layer | Repos | EKS role |
|---|---|---|
| **Shared platform** | `qnsc-infra`, `qnsc-tf-modules` | Landing-zone + EKS platform modules |
| **Shared gitops** | `qnsc-gitops` | ArgoCD ApplicationSets + shared Helm/Kustomize bases |
| **Per-product** | `rally-infra`, `opshub-infra`, … | Per-product Helm chart + namespace + ArgoCD Application |

**Modules to build:**
1. **Landing-zone module** — AWS Organizations accounts, SCPs, IAM Identity Center (SSO).
2. **EKS platform module** — cluster + Karpenter + ArgoCD + External Secrets Operator + AWS Load Balancer Controller. Instantiated once per cluster (Zone B and Zone C).

**Deploy flow (GitOps):** push to `main` → ArgoCD syncs into the product's namespace in the dev cluster. Tag a release → promotes to staging → prod. Declarative, auditable, self-healing — replaces the push-based GitHub Actions `deploy.yml` model used on ECS.

---

## 9. Phasing & migration

This is a **destination**, adopted incrementally. Do not big-bang it.

| Phase | Work | Trigger |
|---|---|---|
| **0 — now** | Finish Rally + opshub on **ECS Fargate**; keep running | In progress |
| **1 — platform** | Landing zone + Zone B EKS cluster (dev first) + ArgoCD + Karpenter; reference Helm chart + ApplicationSet as the **golden path** | Before product #3 / first AI workload |
| **2 — migrate** | Move Rally + opshub from ECS to Zone B namespaces, one service at a time | After Zone B proven |
| **3 — medical** | Stand up Zone C (medical accounts, GPU, HIPAA) for hospital AI | When hospital AI is real |
| **4 — IC** | Separate GovCloud/on-prem HPC engagement | Independent track |

**Why phase, not rush:** ECS products keep running with zero rewrite during the transition. EKS gets built deliberately rather than under deadline pressure.

---

## 10. Why this is the right enterprise approach

- **Compliance-aligned:** zones map to what SOC 2 / ISO 27001 / HIPAA / ITAR assessors expect — isolation by data sensitivity.
- **Cost-efficient:** shared cluster amortizes fixed costs; Karpenter + Spot is materially cheaper than per-service Fargate at scale; new products are near-free to add.
- **Operable:** a small platform team runs a handful of clusters via standardized GitOps, not dozens of bespoke setups.
- **Avoids the "ECS trap":** ECS becomes costly and limiting past ~15–20 services and cannot do GPU; this design moves to the right platform *before* that wall.
- **Right tool per workload:** Kubernetes where it fits (SaaS, AI); HPC/GovCloud where it fits (IC). Not dogmatic.

### Caveats (read these)

1. **This is 6–18 months of platform engineering**, contingent on the K8s expertise being hired. The clean diagrams hide real build effort.
2. **The design assumes the stated roadmap** (6 products, AI/GPU, global). If the roadmap shrinks materially, revisit — parts may become premature.

---

## 11. Quick reference

| Question | Answer |
|---|---|
| Where does a new commercial product go? | Zone B — a namespace in the shared per-env EKS cluster |
| Where does a small tool / site / job go? | Zone B namespace, small quota (KEDA scale-to-zero if spiky) |
| Where does anything touching patient data go? | Zone C — isolated medical EKS |
| Where does defense/chip work go? | Zone A — separate GovCloud/on-prem, non-K8s |
| One cluster for everything? | No. One cluster **per environment per zone**; products are namespaces |
| Does size decide the zone? | No. **Data sensitivity** decides the zone; size decides the quota |

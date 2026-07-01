# QNSC Platform — Tech Stack & Decisions

> **Status:** Decided 2026-06-30; compute re-sequenced 2026-07-01. Companion to [PLATFORM_ARCHITECTURE.md](./PLATFORM_ARCHITECTURE.md).
> **Audience:** Engineering, platform/DevOps, security.
> **Scope:** The commercial platform (Zone B) and the medical platform (Zone C). Zone A (IC/ITAR) runs a separate stack — see [§7](#7-zone-a-ic--separate-stack).
> **Two phases (read first):** Zone B runs on **ECS Fargate today**; **EKS is trigger-driven** (architecture §8/§13.2). This document lists the **full target stack**; the [§2.1 phase map](#21-phase-map--what-runs-now-vs-at-eks) marks what is live on ECS now vs. what is added at the EKS phase. Everything outside the orchestration/CD layer (IaC, data, messaging, identity, edge, observability destinations, supply-chain) is **identical in both phases**.

---

## 1. Guiding principles

1. **Single-cloud platform, multi-vendor only at the edge.** Compute, Kubernetes control, data, and secrets stay on AWS. The only place QNSC deliberately leaves AWS is the **edge layer (CDN/DNS/WAF/DDoS) and heavy-egress object storage**, where AWS is materially overpriced. See [§5](#5-aws-vs-cloudflare--the-edge-boundary).
2. **Provisioning boundary is stable across phases.** OpenTofu owns durable infrastructure in both phases. **ECS phase (now):** CI push-deploys the app (renders task def, `ecs update-service`) — OpenTofu owns the cluster/ALB/IAM/data shell. **EKS phase (later):** OpenTofu stops at the cluster door and ArgoCD owns everything inside (pull-based GitOps). Same principle — durable infra in OpenTofu, the running app in the deploy tool. See [§3](#3-the-provisioning-boundary-opentofu--argocd) and [§4](#4-deploy-flow).
3. **Managed over self-hosted while the team is small.** Prefer AWS-managed services (AMP, AMG, RDS) to reduce day-2 load; revisit self-hosting only when cost at scale justifies the operational burden.
4. **YAGNI.** No service mesh, no cert-manager, no second IaC tool until a concrete need exists. See [§8](#8-deferred-yagni).
5. **Build on what exists.** OpenTofu, GitHub Actions, ECR, and the `qnsc-tf-modules` library are already in use — extend them, don't replace.

---

## 2. The stack (by layer)

| Layer | Choice | Rejected alternative(s) | Why |
|---|---|---|---|
| **IaC / provisioning** | **OpenTofu** | Terraform (licensing), Pulumi, CDK | Already in use; open-source; existing module library + state backend. |
| **Landing zone** | **AWS Control Tower** baseline + OpenTofu resources | Pure-OpenTofu landing zone | CT provides log-archive/audit accounts + guardrails out of the box (saves months); OpenTofu vends workload accounts and all resources. |
| **Container orchestration** | **ECS Fargate now → Amazon EKS when triggered** | self-managed k8s; full-serverless | ECS Fargate is right for the current stage (no control-plane fee/ops); EKS is trigger-driven. See architecture §8 & §13.2. |
| **EKS provisioning** | OpenTofu + **terraform-aws-eks** module | Hand-rolled cluster, eksctl | Community standard; do not hand-roll. |
| **GitOps / CD** | **ArgoCD** (app-of-apps + ApplicationSets) | Flux | Stronger multi-tenancy + UI for many products/teams. |
| **App packaging** | **Helm** — one reusable `qnsc-service` base chart | Raw manifests, Kustomize-only | Many similar SaaS services → one parametrized golden chart. Kustomize overlays only if needed. |
| **CI (build/test/scan/sign/push)** | **GitHub Actions** | GitLab CI, Jenkins | Already in use (`qnsc-gitops` actions). Clean split: GHA = CI, ArgoCD = CD. |
| **Image registry** | **Amazon ECR** | GHCR, Cloudflare | In-VPC, IRSA-integrated, no egress on in-region pulls. |
| **Node autoscaling** | **Karpenter** | Cluster Autoscaler | Faster provisioning, better bin-packing, broad Spot — the core compute cost win. |
| **Pod autoscaling** | **HPA** + **KEDA** | HPA-only | KEDA adds event-driven scaling + scale-to-zero for spiky jobs / small projects. |
| **Secrets** | **External Secrets Operator** ← AWS Secrets Manager | Sealed Secrets, SOPS | No static keys; KMS-encrypted; syncs from the AWS source of truth. |
| **Ingress (origin)** | **AWS Load Balancer Controller** + **ACM** certs | NGINX Ingress + cert-manager | ALB-native TLS via ACM skips cert-manager for public endpoints. |
| **DNS** | **Cloudflare DNS** via **external-dns** | Route53 | Declarative DNS from manifests; consolidates with the Cloudflare edge (§5). |
| **CNI / NetworkPolicy** | **VPC CNI** + its NetworkPolicy → **Cilium** later | Calico | Start AWS-native/simple; move to Cilium for eBPF + Hubble observability + richer policy when needed. |
| **Admission policy** | **Kyverno** | OPA / Gatekeeper | YAML-native, easier for a team newer to K8s. Enforces non-root, signed-images-only, default-deny, required labels. |
| **Supply chain** | **Cosign / Sigstore** (existing `attest-image`) + **Trivy** | Notary v1 | Sign + scan in CI; verify signatures at admission via Kyverno. |
| **Relational data** | **PostgreSQL** — Aurora (SaaS) / RDS (internal) | MySQL, self-hosted in K8s | RLS for multi-tenancy, JSONB, pgvector. See §10. Never run DBs in the cluster. |
| **Cache + in-app job state** | **Amazon ElastiCache (Redis)** | Self-hosted Redis in K8s | Cache + backing store for BullMQ job queues (persistence + HA enabled). |
| **Messaging / events** | **SQS + EventBridge + SNS** | RabbitMQ, self-run Kafka | Managed, zero-ops backbone. See §11. |
| **CDN / WAF / DDoS** | **Cloudflare** (Zone B) | CloudFront + Shield + AWS WAF | Flat/near-free bandwidth vs AWS egress; better DDoS. See §5. |
| **Bulk object storage** | **S3** default; **Cloudflare R2** for heavy-egress assets | S3-only | R2 has zero egress fees — decisive for media (e.g. Learning video). See §5. |
| **Metrics** | **AWS Managed Prometheus (AMP)** + **Managed Grafana (AMG)** | Self-hosted kube-prometheus-stack | Managed = less day-2 load; revisit self-host at cost scale. |
| **Logs** | **CloudWatch** (Fluent Bit / Container Insights) | Loki | Simplest to start; move to **Loki** if ingestion cost grows. |
| **Tracing** | **OpenTelemetry** → AWS X-Ray (or Tempo) | Vendor-specific SDKs | Vendor-neutral instrumentation. |
| **State backend** | OpenTofu state in **S3 + DynamoDB lock** | Terraform Cloud, local | Existing `state-backend` module; lives in the shared-services account. |

---

## 2.1 Phase map — what runs now vs. at EKS

The §2 table is the **full target**. Only the orchestration/CD layer differs by phase; everything else is identical.

| Concern | **ECS Fargate phase (now)** | **EKS phase (trigger-driven)** |
|---|---|---|
| Compute | **ECS Fargate** + **Fargate Spot** (dev/staging, stateless workers) | EKS + **Karpenter** (EC2 Spot/Graviton bin-packing) |
| Event-driven glue | **AWS Lambda** (cron, dev-scheduler, webhooks) — *stays Lambda after EKS too* | same |
| App packaging | ECS **task definition** (templated JSON) | **Helm** `qnsc-service` base chart |
| CD | **GitHub Actions push** → `ecs update-service` / **CodeDeploy blue-green** | **ArgoCD** app-of-apps + ApplicationSets (pull) |
| Node autoscaling | none (Fargate is nodeless) | **Karpenter** |
| Pod/service autoscaling | **ECS Service Auto Scaling** (target tracking) | **HPA** + **KEDA** (event-driven + scale-to-zero) |
| Secrets | **Secrets Manager → ECS task `secrets`** injection | **External Secrets Operator** ← Secrets Manager |
| Ingress | **ALB** + target groups (per service, host/path) | **AWS LB Controller** + Ingress |
| Admission policy | enforced **in the CI pipeline** (block unsigned / critical-CVE before deploy) | **Kyverno** `verifyImages` at admission |
| Network isolation | **security groups** (B1/B2 separate services + SGs) | **NetworkPolicy** (default-deny) + Karpenter NodePool taints |
| DNS | **Cloudflare** (managed in the product's `infra/`) | Cloudflare via **external-dns** |
| **Identical both phases** | OpenTofu · ECR · GitHub Actions build/test/scan/sign (Cosign/Trivy) · Cloudflare edge + R2 · PostgreSQL (Aurora/RDS) · pgvector · DynamoDB · SQS/EventBridge/SNS · IAM Identity Center · WorkOS · AMP/AMG/CloudWatch/X-Ray destinations · AWS Backup · Vanta/Drata | ← same |

**Workload identity by phase:** ECS uses **task IAM roles** (`taskRoleArn`); EKS uses **IRSA / Pod Identity**. Both are keyless; both scope least-privilege per service.

---

## 3. The provisioning boundary (OpenTofu ↔ ArgoCD)

> **OpenTofu provisions infrastructure + IRSA roles + bootstraps ArgoCD only. ArgoCD owns everything inside the cluster** — Karpenter, External Secrets, the AWS Load Balancer Controller, Kyverno, observability agents, and all application workloads.

*(This is the **EKS-phase** boundary. On **ECS today** there is no ArgoCD: OpenTofu owns the durable shell — cluster, ALB, target groups, task **IAM roles**, capacity providers, RDS, secrets — and **CI owns the task-def revision + `update-service`** as the fast deploy path. Same split: durable infra in OpenTofu, the running app in the deploy tool. See [§4](#4-deploy-flow).)*

**Why:** managing in-cluster Helm releases from OpenTofu's Helm provider creates two reconciliation loops fighting over the same Kubernetes objects, producing constant drift and brittle applies. OpenTofu stops at the cluster door; ArgoCD takes over inside. This keeps a single source of truth per object and makes the platform self-healing.

```
OpenTofu  ──>  AWS Org · accounts · VPC · EKS cluster · RDS · ElastiCache · ECR
          ──>  IRSA / Pod Identity roles
          ──>  install ArgoCD   ── (handoff) ──>  ArgoCD owns:
                                                    Karpenter · ESO · ALB controller ·
                                                    Kyverno · external-dns · observability ·
                                                    all product workloads (Helm + ApplicationSets)
```

---

## 4. Deploy flow

**ECS phase (now) — push-based:**
- **CI (GitHub Actions):** build image → test → Trivy scan → **Cosign sign** → **gate: block unsigned / critical-CVE** → push to ECR → render task-def revision → `ecs update-service` (rolling) or **CodeDeploy blue/green** → verify health → run DB migrations (one-off task).
- **Rollback:** re-point the service to the previous task-def revision (blue/green makes this instant).
- The supply-chain **verify** step lives in the pipeline here (there is no admission controller without K8s).

**EKS phase (trigger-driven) — pull-based:**
- **CI (GitHub Actions):** build → test → scan → sign → push to ECR → **bump image tag in the GitOps config repo** (Git).
- **CD (ArgoCD):** detects the Git change → syncs into the product namespace → tag a release → promote staging → prod. Declarative, auditable, self-healing.
- Verify moves to **admission** (Kyverno `verifyImages`), in addition to the CI gate.

The **build half is identical** across phases; only the deploy half changes. That is what makes the migration cheap (architecture §13.3).

---

## 5. AWS vs Cloudflare — the edge boundary

**Principle:** AWS egress/bandwidth (~$0.05–0.09/GB out) is the one materially overpriced layer. QNSC leaves AWS **only** at the edge and for heavy-egress object storage; everything else stays AWS.

| Component | Decision | Notes |
|---|---|---|
| DNS, CDN, WAF, DDoS (Zone B / B1 external) | **Cloudflare** | `external-dns` Cloudflare provider keeps DNS in GitOps. |
| Heavy-egress bulk assets (e.g. Learning video) | **Cloudflare R2** | Zero egress fees; decisive for media-heavy SaaS. |
| General object storage in compliance/VPC scope | **S3** | Stays AWS. |
| Compute, EKS, data, secrets, registry | **AWS** | Single-platform; do not fragment. |

**Cost rationale:** at ~50 TB/mo egress, AWS S3/CloudFront ≈ **$4,250/mo** vs ≈ **$0** through Cloudflare/R2. Even modest SaaS traffic saves meaningfully. This is the single largest external cost lever.

### Compliance guardrails on the edge

- **Zone B (commercial):** Cloudflare permitted. ✅
- **Zone C (medical / PHI):** Cloudflare permitted **only** with a signed Cloudflare **BAA (Enterprise)**; it adds a vendor to PHI compliance scope. **Default: keep Zone C AWS-native** (CloudFront + Shield + S3). Revisit only if egress cost there is large.
- **Zone A (IC / ITAR):** **No third-party services. Ever.** GovCloud/on-prem only.

### Security gotcha (mandatory if Cloudflare fronts an ALB)

Lock the AWS origin so it accepts **only Cloudflare** — allow-list Cloudflare IP ranges in the security group, use **Cloudflare Tunnel**, or enable **Authenticated Origin Pulls (mTLS)**. Otherwise attackers reach the ALB directly and bypass the WAF/DDoS protection. Pick **one DNS source of truth** (Cloudflare) and point `external-dns` there.

---

## 6. Ansible — placement

**Ansible is NOT part of the Kubernetes platform.** EKS runs immutable containers; pods are not configuration-managed.

- **Zone B / Zone C (EKS):** OpenTofu + Helm + ArgoCD cover everything. **No Ansible.**
- **Zone A (IC / EDA / HPC):** mutable EC2/VM license servers + NICE DCV workstations — this is where **Ansible** belongs (config management, package install, OS hardening). Provision with OpenTofu, configure with Ansible. Separate GovCloud track.

---

## 7. Zone A (IC) — separate stack

Zone A is ITAR-controlled, non-Kubernetes, and fully isolated. It does **not** share this stack, the AWS Organization, identity, registry, or pipelines.

| Layer | Zone A choice |
|---|---|
| Cloud | AWS GovCloud (US) or on-prem / air-gapped |
| Provisioning | OpenTofu (separate state, separate boundary) |
| Config management | **Ansible** |
| Compute | EC2/HPC + EDA license servers (Cadence/Synopsys/Mentor) |
| Storage | FSx for Lustre / NetApp |
| Access | NICE DCV workstations; in-boundary IAM only |
| Edge / third-party | **None** |

---

## 8. Deferred (YAGNI)

Adopt only when a concrete need appears:

- **Service mesh** (Istio / Linkerd) — until mTLS-everywhere or fine-grained traffic-splitting is genuinely required. NetworkPolicy + ALB cover current needs.
- **cert-manager** — ACM via ALB handles public TLS. Add only for in-cluster mTLS.
- **Self-hosted Prometheus / Loki** — start managed (AMP/AMG/CloudWatch); self-host only when cost at scale justifies the ops.
- **Cilium** — start with VPC CNI; adopt for eBPF/Hubble observability + advanced policy when needed.
- **Second IaC or GitOps tool** (Crossplane, Pulumi, Flux) — OpenTofu + ArgoCD are chosen; do not split the stack.

---

## 9. Decision record (summary)

| # | Decision | Rationale | Status |
|---|---|---|---|
| ADR-1 | **ECS Fargate is the starting platform**; EKS is the trigger-driven destination | Right for current stage (no control-plane fee/ops, no platform-team salary before scale). EKS wins only at scale + with a team (architecture §8) | **Revised 2026-07-01** |
| ADR-2 | OpenTofu for IaC | Existing use, open-source, module library | Accepted |
| ADR-3 | ArgoCD for CD; GitHub Actions for CI | Clean CI/CD split; ArgoCD multi-tenancy | Accepted |
| ADR-4 | OpenTofu provisions infra + bootstraps ArgoCD; ArgoCD owns in-cluster | Avoid dual reconciliation loops / drift | Accepted |
| ADR-5 | Helm base chart + ApplicationSets | One golden path for many similar services | Accepted |
| ADR-6 | Karpenter + HPA + KEDA | Cost-efficient, event-driven, scale-to-zero | Accepted |
| ADR-7 | External Secrets Operator + Secrets Manager + KMS | No static keys, single source of truth | Accepted |
| ADR-8 | Kyverno for admission policy | Simpler than OPA for the team | Accepted |
| ADR-9 | Cloudflare for Zone B edge (DNS/CDN/WAF/DDoS) + R2 for heavy egress | AWS egress is overpriced; biggest external cost lever | Accepted |
| ADR-10 | Zone C edge stays AWS-native by default; Zone A no third-party | Limit PHI compliance scope; ITAR isolation | Accepted |
| ADR-11 | Ansible only in Zone A (mutable VMs/HPC), not on EKS | Right tool for mutable vs immutable infra | Accepted |
| ADR-12 | Managed observability (AMP/AMG/CloudWatch) to start | Reduce day-2 load on a small team | Accepted, revisit at cost scale |
| ADR-13 | PostgreSQL everywhere — Aurora for SaaS, RDS for internal, pgvector for AI | RLS multi-tenancy, JSONB, vectors; no separate vector DB yet | Accepted |
| ADR-14 | S3 is the default object store; R2 only for heavy-egress public assets | R2 is a CDN-origin egress play, not a durable AWS-integrated store | Accepted |
| ADR-15 | SQS + EventBridge + SNS backbone; BullMQ/Celery for in-app jobs; no RabbitMQ; defer Kafka | Managed zero-ops backbone; streaming only when real | Accepted |
| ADR-16 | Workforce identity: IAM Identity Center federated to corporate IdP (Entra ID/Okta); no IAM users; MFA mandatory | One identity → AWS + EKS RBAC + ArgoCD + Grafana | Accepted |
| ADR-17 | Customer identity: WorkOS (B2B SSO/SCIM) preferred; Cognito if cost-first | Enterprise/hospital customers require SAML/OIDC + SCIM | Accepted, vendor TBD |
| ADR-18 | CI supply chain to SLSA L3: sign + verify-at-admission, SAST/IaC/container scan, SBOM, SHA-pinned actions, prod approval gates | Close the supply-chain loop; least-privilege CI | Accepted |
| ADR-19 | Observability: AMP + AMG + Fluent Bit→CloudWatch (→Loki) + OTel→X-Ray; SLOs; PagerDuty | Managed-first; control cardinality + log volume | Accepted |
| ADR-20 | FinOps: enforced cost tags + Kubecost showback; Spot/Graviton/Savings Plans; dev-scheduler off-hours | Per-product unit economics; cut waste | Accepted |
| ADR-21 | Backup/DR: AWS Backup + RDS cross-region + Velero + restore tests | Tested recovery, not just backups | Accepted |
| ADR-22 | Compliance automation (Vanta/Drata + Audit Manager); VPC endpoints/PrivateLink for AWS API/ECR/S3 | Audit evidence; keep internal traffic off the internet | Accepted |
| ADR-23 | EKS adoption is **trigger-gated** (team hired · Zone C real · >~15–20 services · Fargate cost high + util low · mesh/GPU/KEDA · GovCloud portability), not calendar-scheduled | Avoid paying the EKS ops/people cost before scale justifies it; Zone C is the likely forcing function | Accepted |
| ADR-24 | **Lambda for event-driven glue only** (cron, dev-scheduler, webhooks, light async); no full-serverless primary runtime | Full-serverless is a different architecture, not a step toward EKS; cold-start/15-min/VPC-latency hurt always-on SaaS | Accepted |
| ADR-25 | ECS deploy: CI push (`update-service` / CodeDeploy blue-green); supply-chain verify enforced **in-pipeline** (no admission controller pre-K8s) | Fast rollback, zero-downtime; keeps the sign→verify loop closed without ArgoCD/Kyverno | Accepted |

---

## 10. Data layer

**Relational (primary OLTP): PostgreSQL, two tiers.**

| Use | Engine | Notes |
|---|---|---|
| Customer SaaS (Rally, Learning, KB) | **Aurora PostgreSQL** (Serverless v2 where spiky) | 15 read replicas, ~30s failover, storage autoscaling. Worth the ~20% premium for revenue-facing workloads. |
| Internal / small (opshub, tools) | **RDS PostgreSQL** | Cheaper and simpler; does not need Aurora scaling. |
| AI vector search (KB, hospital AI) | **PostgreSQL + pgvector** | Embeddings beside the data; no dedicated vector DB (Pinecone/Weaviate) until scale forces it. |
| KV at massive scale / serverless (sessions, flags, high-write logs) | **DynamoDB** — only for these patterns | Not the default. Postgres first. |

- **Why Postgres over MySQL:** row-level security (the multi-tenant isolation model in architecture §6.3), JSONB, pgvector, rich extensions.
- **One module, two sizings** — matches "size decides the quota, not the zone."
- **Databases never run in Kubernetes.** Managed, in-VPC, in compliance scope.
- **Tenant data model:** pooled (shared Aurora + enforced RLS) by default; promote large/regulated tenants to schema- or DB-per-tenant. See architecture §6.3.

**Object storage:** S3 is the default (app uploads, backups, OpenTofu state, ALB/audit logs, data lake, AWS-service integration, all compliance-scope data). **Cloudflare R2** is the targeted exception for heavy-egress *public* assets (e.g. Learning video). See §5.

**Cache:** ElastiCache (Redis) — caching plus the backing store for BullMQ job queues (with persistence + HA enabled, since it then holds job state).

---

## 11. Messaging & events

"Message queue" spans three distinct needs; each gets the right tool.

| Need | Choice | Notes |
|---|---|---|
| Service-to-service decoupling / durable queues | **SQS** (+ dead-letter queues) | Managed, infinite scale, zero ops. |
| Pub/sub, event routing, fan-out | **EventBridge** (domain events) + **SNS** (simple fan-out) | Schema/rules routing via EventBridge; SNS→SQS fan-out where needed. |
| In-app job queues (delayed / retry / scheduled / rate-limited) | **BullMQ on Redis** (Node) · **Celery** (Python) | Worker frameworks, not infra. Celery can use an **SQS broker** → Celery DX on managed SQS. |
| Event streaming / replay / high throughput | **Deferred** — **Kinesis** (light) before **MSK/Kafka** (heavy) | Only for genuine streaming (hospital camera event streams, analytics/CDC). Likely a Zone C concern. |

**Backbone = SQS + EventBridge.** Managed, zero-ops, DLQ built in — covers ~90% of queue/event needs and keeps brokers off the platform team's plate.

**Rejected / deferred:**
- **RabbitMQ — skip.** SQS + EventBridge cover its role; it is just another stateful broker to run.
- **Kafka — defer.** Adopt only when streaming is real; try Kinesis first.

**Rule:** managed AWS backbone (SQS + EventBridge + SNS) for cross-service; app-framework queues (BullMQ/Celery) for in-app job ergonomics; no self-run brokers; Kafka only when streaming is genuinely required.

---

## 12. Identity & access

Three distinct layers — do not conflate them.

### 12.1 Workforce identity (engineers → AWS, K8s, tooling)

| Decision | Choice |
|---|---|
| Hub | **AWS IAM Identity Center (SSO)**, federated to the corporate IdP (**Entra ID / Okta**) |
| Propagation | SSO groups → AWS permission sets **+** EKS access entries / RBAC (group → ClusterRole) **+** ArgoCD RBAC (OIDC) **+** Grafana (OIDC) |
| Rules | MFA mandatory · **no IAM users** · root locked with hardware MFA · break-glass role audited |

One identity grants every surface; SSO group membership is the single source of authorization.

### 12.2 Customer / end-user identity (SaaS login)

QNSC sells B2B to enterprises and hospitals, which require **SAML/OIDC SSO + SCIM** provisioning — that requirement drives the choice.

| Option | Best when | Trade-off |
|---|---|---|
| **WorkOS** *(preferred)* | B2B enterprise SSO/SCIM is the core need | Purpose-built, predictable pricing, fast to ship |
| **Auth0** | Full CIAM (social, MFA, flows) + B2B | Best DX; gets expensive at scale |
| **Cognito** | Cost-first, willing to build enterprise SSO | AWS-native, cheap, weaker B2B-SSO ergonomics |
| Keycloak | Full control, have ops capacity | Self-host burden |

Default to **WorkOS**; choose **Cognito** if cost dominates. Per-tenant IdP connections map to the tenancy model (architecture §6.3). Vendor selection is the one open item here.

### 12.3 Workload identity

**ECS now:** per-service **task IAM roles** (`taskRoleArn`) — keyless, least-privilege per service. **EKS later:** **IRSA / Pod Identity** for pod → AWS. Both avoid static keys. Service-to-service auth uses app-level tokens/OIDC now; mTLS arrives with a service mesh (deferred, §8).

---

## 13. CI / supply-chain security

Builds on the existing OIDC, `attest-image`, `scan-secrets`, Dependabot, and CODEOWNERS. **Target: SLSA build level 3.**

| Control | Tool | Status |
|---|---|---|
| CI → AWS auth | **GitHub OIDC**, least-privilege role per repo/env | ✅ have |
| Secret scanning | gitleaks / `scan-secrets` | ✅ have |
| Dependencies (SCA) | Dependabot | ✅ have |
| SAST | **CodeQL** or Semgrep | add |
| IaC scan | **Trivy / tfsec / checkov** on OpenTofu | add |
| Container scan | **Trivy** in CI + ECR scan | add |
| SBOM | **Syft** | add |
| Sign + attest | **Cosign** (`attest-image`) | ✅ have |
| Admission verify | **Kyverno verifyImages** — block unsigned images in-cluster | add |
| Pipeline hardening | **pin actions to commit SHA** (not tags), allow-list third-party actions, ephemeral runners | add |
| Prod gate | GitHub **environment protection** — manual approval for prod | add |

Signing in CI plus verification at admission closes the supply-chain loop: only built-by-us, signed, scanned images run in the cluster.

---

## 14. Observability

Managed-first to keep day-2 load low; revisit self-hosting at cost scale.

| Signal | Pipeline |
|---|---|
| Metrics | app `/metrics` + kube-state-metrics + node-exporter → **ADOT / Prometheus agent** → **AMP** → **AMG** (dashboards as code, in Git) |
| Logs | **Fluent Bit** → CloudWatch (start) → **Loki** if ingestion cost grows. Structured JSON + correlation IDs |
| Traces | **OpenTelemetry (ADOT)** → X-Ray (or Tempo) |
| SLOs | SLI/SLO per service + error budgets |
| Alerting | Alertmanager / CloudWatch alarms → **PagerDuty / Opsgenie** + Slack |
| Uptime | CloudWatch Synthetics **or** Cloudflare health checks |
| Status page | Cloudflare / Statuspage |

**Cost discipline:** control Prometheus **cardinality** and CloudWatch **log volume** — both are silent cost bombs. Dashboards and alert rules live in Git.

---

## 15. FinOps / cost management

| Lever | Action |
|---|---|
| Visibility | Mandatory cost-allocation tags (product / env / zone / team) enforced by **tag policy + SCP**. **ECS now:** Cost Explorer + **ECS Split Cost Allocation** for per-service cost. **EKS later:** **Kubecost / OpenCost** per-namespace showback |
| Compute | **ECS now:** Fargate **Spot** (dev/staging + stateless workers), **Compute Savings Plans** for steady baseline, right-size task CPU/mem. **EKS later:** Karpenter Spot + consolidation + **Graviton** (ARM) |
| Data | **Aurora Serverless v2** scale-down, **RDS reserved** for steady load, S3 **Intelligent-Tiering** + lifecycle |
| Egress | Cloudflare / R2 (the largest external lever — §5) |
| Dev waste | **dev-scheduler** module (existing) — auto-shutdown off-hours; KEDA scale-to-zero for spiky workloads |
| Guardrails | Cost Explorer + **Budgets** + anomaly detection + alerts |

Per-product cost (via Kubecost) is the number that matters for SaaS unit economics — not just the total AWS bill.

---

## 16. Operational foundations

Decide before production, not after.

| Area | Choice |
|---|---|
| **Backup / DR** | **AWS Backup** (centralized) + RDS cross-region snapshots + **Velero** (K8s cluster state) + scheduled **restore tests**. Targets per architecture §11 |
| **Compliance automation** | **Vanta / Drata** (SOC 2 / HIPAA evidence) + AWS Audit Manager + Config conformance packs |
| **Network privacy & cost** | **VPC endpoints (PrivateLink)** for AWS API / ECR / S3 — keeps internal traffic off the internet (security + NAT cost) |
| **Feature flags** | OpenFeature + **Flagsmith / LaunchDarkly** |
| **Developer portal** | **Backstage** — *deferred* until product/team count makes it pay (§8) |

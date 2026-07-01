# QNSC Repository Structure

> **Status:** Decided 2026-06-30; deploy path made phase-aware 2026-07-01. Companion to [PLATFORM_ARCHITECTURE.md](./PLATFORM_ARCHITECTURE.md) and [TECH_STACK.md](./TECH_STACK.md).
> **Audience:** Engineering, platform/DevOps.
> **Owner:** Platform team.
> **Context:** Greenfield restructure. The current repos (`qnsc-*`, `rally-*`, `opshub-*`, …) are replaced by the topology below via a **safe, additive migration** — see [§5](#5-fresh-start-migration-archive-never-delete-first).
> **Phase note:** The repo topology is the **same in both compute phases**. Only the deploy layer differs: **ECS phase (now)** deploys via `qnsc-ci` + each product's `deploy/ecs/`; **EKS phase (later)** adds `chart/` per product and the `qnsc-gitops` ArgoCD hub. See [§3](#3-per-product-monorepo-layout) and [§4](#4-qnsc-gitops-layout-argocd-config-hub--eks-phase).

---

## 1. Principles

1. **Separate config from code.** Application repos build and push images; a dedicated GitOps config repo holds *what is deployed*. CI bumps an image tag in the config repo → ArgoCD syncs. Application CI never retriggers on a deploy.
2. **Repo boundaries follow ownership boundaries** (Conway's law). The platform team owns shared repos; each product team owns its product repo.
3. **One golden path.** New products are scaffolded from a template and wired to shared modules and reusable CI — adding product #7 is near-zero effort.
4. **Archive, never delete** repos that held production or regulated infrastructure — audit trail is a compliance requirement.

---

## 2. Topology (3 tiers)

### Tier 1 — Shared platform (platform team)

| Repo | Purpose |
|---|---|
| `qnsc-infra` | Live OpenTofu: AWS Organization, accounts, landing zone, per-env **ECS clusters** now (+ **EKS clusters** at the EKS phase), shared networking |
| `qnsc-tf-modules` | Reusable OpenTofu modules (**ecs-service**, network, rds/aurora, cache, secrets now; **eks** added at the EKS phase) |
| `qnsc-ci` *(or org `.github`)* | Reusable CI workflows + composite actions (build/test/scan/sign/push + **ECS push-deploy** now). **The active CD home in the ECS phase.** |
| `qnsc-gitops` | **ArgoCD config hub** *(EKS phase — created at migration, §4)* — app-of-apps, ApplicationSets, platform add-ons, cluster/env registry, shared `qnsc-service` Helm base chart, per-product env values + live image tags |
| `qnsc-packages` *(only if needed)* | Shared libraries published to CodeArtifact — **only** for code shared *across products*; code shared *within* a product lives in that product's `packages/` |

> **No dedicated template repo.** Scaffolding for new products uses GitHub's native **template repository** feature on the first clean product repo, or a `templates/` cookiecutter in `qnsc-ci`. Revisit a dedicated scaffolder/CLI only at product #3+. (Deferred — ADR-R7.)

### Tier 2 — Per product (product team) — one monorepo each

| Repo | Replaces |
|---|---|
| `rally` | `rally-api` + `rally-web` + `rally-infra` |
| `opshub` | `opshub-api` + `opshub-web` + `opshub-infra` |
| `learning`, `knowledge-base`, … | born new on this structure |

### Tier 3 — Zone A (IC / ITAR) — isolated

`ic-infra` + Ansible playbooks, in the **separate GovCloud/on-prem organization**. Not in the commercial org, not federated, not in this tree.

**Net count:** ~5–6 shared + 1 per product = **~11–12 commercial repos** (vs ~22 under the old 3-per-product pattern), plus the isolated Zone A repos. **In the ECS phase today the shared tier is only 3** (`qnsc-gitops` and `qnsc-packages` defer) — see §2.1.

---

## 2.1 Day-1 repo set (ECS phase) — create now vs. defer

The topology above is the **full target**. It is stable across the ECS→EKS migration — **you never restructure**; you only add `chart/` to products and stand up `qnsc-gitops` when the EKS trigger fires (architecture §13.2). So set up the lean ECS-phase set now and let it grow.

**The rule: repo boundaries follow ownership + compliance, not the runtime.** That is why the same topology serves ECS today and EKS later.

| Repo | Create now (ECS phase) | Why / trigger to add |
|---|---|---|
| `qnsc-infra` | ✅ **now** | Landing zone + per-env VPC / ECS cluster / RDS |
| `qnsc-tf-modules` | ✅ **now** | **Infra reuse engine** — `ecs-service`/`rds`/`network`/`cache`/`secrets`; every product's `infra/` calls these |
| `qnsc-ci` *(= existing org `.github`)* | ✅ **now** | **Pipeline reuse engine** — one reusable `build→scan→sign→ecs-deploy` workflow; every product calls it |
| `rally`, `opshub` (monorepos) | ✅ **now** | Consolidate old `*-api`/`*-web`/`*-infra`; `apps/`+`packages/`+`deploy/ecs/`+`infra/` |
| `learning`, `knowledge-base` | when the product starts | Born on this structure (ECS Fargate) |
| `qnsc-packages` | ❌ defer | Only when real code is shared **across** products (needs CodeArtifact publishing) |
| `qnsc-gitops` | ❌ defer | Created **when the EKS trigger fires** — it is ArgoCD config, useless pre-K8s |
| `ic-infra` (Zone A) | separate track | Isolated GovCloud/on-prem org — never in this tree |

**Lean-but-right day-1 set: `qnsc-infra` + `qnsc-tf-modules` + `qnsc-ci`(existing `.github`) + one monorepo per live product.**

### How it expands without restructuring

The whole point — each future step is **additive**, never a reshuffle:

| Future need | Action | Repo change |
|---|---|---|
| Add product #3+ | Clone the template monorepo | +1 product repo; wire 2 module calls + 1 workflow call |
| A service outgrows the monolith | Add `apps/<service>/` (§3.1) | none — new image/service, same repo |
| Real cross-product shared lib appears | Create `qnsc-packages` + CodeArtifact | +1 shared repo |
| A service earns a dedicated team | Graduate it to its own repo (§3.1) | +1 repo, on Conway's-law boundary |
| **EKS trigger fires** | Add `chart/` per product; create `qnsc-gitops` | +1 shared repo; **no product restructuring** |
| Zone C (hospital AI) becomes real | New isolated med accounts + repos | separate zone, reuses `qnsc-tf-modules` |

Every arrow is "+1", never "reorganize." That is what makes this the scale-friendly enterprise setup: **start with 3 shared repos, grow one repo at a time, never migrate the layout.**

---

## 3. Per-product monorepo layout

```
rally/
├── apps/                    # deployable services — each builds its OWN image,
│   │                        #   runs as its OWN ECS service now / Deployment + HPA on EKS later
│   ├── api/                 # today: modular monolith
│   ├── web/                 # frontend
│   └── worker/              # background jobs
│        ↓ as it decomposes, just add more (no restructuring):
│   ├── auth/  billing/  catalog/  notifications/  search/ ...
├── packages/                # shared libs ACROSS this product's services
│                            #   (types, auth, db models) — imported directly, not published
├── deploy/                  # deploy descriptors
│   └── ecs/                 #   ECS PHASE (now): task-def templates, service params
│       (chart/ added here at the EKS phase: Helm chart, versioned WITH the code)
├── infra/                   # OpenTofu: product-OWNED resources (Aurora, queues, buckets, ECS service shell)
│                            #   uses modules from qnsc-tf-modules
├── .github/workflows/       # thin — calls reusable workflows from qnsc-ci
└── README.md
```

- **Path-filtered CI:** a change under `apps/api/` builds only the api image, etc.
- **Atomic full-stack PRs:** an API contract change and its web consumer move in one PR.
- **Product-owned cloud resources** (its database, queues, buckets, ECS service shell) live in `infra/` and are applied by CI with a least-privilege, per-product role. Shared/landing-zone infrastructure stays in `qnsc-infra`.
- **Deploy descriptors are phase-dependent** (architecture §8/§13):
  - **ECS phase (now):** `deploy/ecs/` holds the **task-def template**; CI renders it, bumps the image tag, and `update-service` (or CodeDeploy blue/green). No separate GitOps repo needed.
  - **EKS phase (later):** add `chart/` (Helm templates, with the code); **env values + live image tag** move to `qnsc-gitops` and ArgoCD syncs — preserving config/code separation.

### 3.1 Scaling a product monorepo to microservices

**Monorepo describes where code lives, not how it runs.** A product monorepo runs as many independent microservices on EKS — they are not one process.

- A modular monolith starts as `apps/api`. As it decomposes, **add `apps/<service>/`** — each becomes its own image, Deployment, and HPA, scaling independently. **No repo migration.**
- Each new service gets one entry under `qnsc-gitops/products/<product>/` (its own values from the shared base chart) and ArgoCD deploys it.
- **Shared code stays trivial** via `packages/` (imported directly). Separate repos would force `qnsc-packages` + CodeArtifact publishing — the monorepo defers that overhead until it is actually needed.
- **Graduate a service out to its own repo only when it earns a dedicated team** / independent release cadence / different language (Conway's law). Do not pre-split. The structure supports the exit when the time comes.

---

## 4. `qnsc-gitops` layout (ArgoCD config hub) — EKS phase

> **Phase note:** This layout is the **EKS-phase** ArgoCD config hub. It is created **when the EKS trigger fires** (architecture §13.2) — not on day one. In the **ECS phase (now)**, GitOps/ArgoCD does not exist: reusable CI/CD lives in **`qnsc-ci`** (§2 Tier 1) and each product deploys via CI push (`deploy/ecs/` task-def → `update-service`). Stand `qnsc-gitops` up at the migration, then move per-product env values + image tags into it.

```
qnsc-gitops/
├── bootstrap/                  # app-of-apps root Application per cluster
├── clusters/                   # cluster/env registry
│   ├── qnsc-dev/
│   ├── qnsc-staging/
│   └── qnsc-prod/
├── addons/                     # platform add-ons (managed by ArgoCD, not OpenTofu)
│   ├── karpenter/  external-secrets/  aws-lb-controller/  kyverno/  observability/
├── charts/
│   └── qnsc-service/           # shared Helm base chart (the golden path)
├── applicationsets/            # generators that fan out products × envs → Applications
└── products/
    ├── rally/
    │   ├── base/               # references rally/chart + qnsc-service base
    │   └── envs/{dev,staging,prod}/values.yaml   # env values + image tag (CI bumps these)
    ├── opshub/
    └── learning/
```

**Flow:** product CI builds + pushes image to ECR → opens a PR (or commits) to `qnsc-gitops` bumping the env image tag → ArgoCD detects the change → syncs the product namespace. Promotion dev → staging → prod is a values change in `qnsc-gitops`.

**Access control:** one central repo with **CODEOWNERS per `products/<name>/` directory** — each team approves its own deploys; the platform team owns `addons/`, `charts/`, `clusters/`. Split into per-product config repos *only later* if a team needs full autonomy.

---

## 5. Fresh-start migration (archive, never delete-first)

Rally and opshub run in production on ECS today; their infra repos manage live state. The restructure is **additive**, then cuts over, then archives.

| Step | Action | Safety |
|---|---|---|
| **R0** | Create the new shared platform repos (`qnsc-infra`, `qnsc-tf-modules`, `qnsc-gitops`, `qnsc-ci`, `qnsc-infra-template`). | Additive — zero risk to prod |
| **R1** | Create product monorepos (`rally`, `opshub`) by consolidating existing `*-api`/`*-web`/`*-infra` (preserve git history via subtree merges where useful). Build new products (`learning`, `knowledge-base`) here directly. | Old repos still live and serving |
| **R2** | Migrate runtime ECS → EKS per architecture §13 (compute moves, data stays). `rally-infra`/`opshub-infra` remain **active** until that product is fully on EKS. | Never delete live-prod infra |
| **R3** | Once a product is fully on EKS and stable, **archive** (read-only) the old `*-api`/`*-web`/`*-infra` repos. | History + audit preserved |
| **R4** | Delete archived repos only much later, if ever — archived is free and keeps the audit trail. | Compliance-safe |

**Rule:** a repo that managed production or regulated infrastructure is **archived, not deleted** — preserving audit trail is a SOC 2 / HIPAA requirement.

---

## 6. Adding a new product (the test of the setup)

1. **GitHub "Use this template"** (or the `qnsc-ci` cookiecutter) scaffolds the new `product` monorepo — `apps/`, `packages/`, `chart/`, `infra/` wired to shared modules, `.github/workflows/` wired to `qnsc-ci`.
2. Add one ApplicationSet entry + `products/<name>/envs/*` values in `qnsc-gitops`.
3. ArgoCD creates the namespace and syncs.

Near-zero marginal effort — the platform's core promise.

---

## 7. Decision record

| # | Decision | Rationale | Status |
|---|---|---|---|
| ADR-R1 | Product monorepo (api+web+worker+chart+infra per product) | Atomic full-stack PRs, one owner, ~6 repos not ~18 | Accepted |
| ADR-R2 | Central `qnsc-gitops` config hub (CODEOWNERS per product dir) | One pane of glass, curated standards; split later if needed | Accepted |
| ADR-R3 | Separate config from code (chart with code; image tags/values in gitops) | GitOps best practice; clean deploy audit | Accepted |
| ADR-R4 | Move reusable CI out of `qnsc-gitops` into `qnsc-ci`/`.github` | `qnsc-gitops` becomes true GitOps config | Accepted |
| ADR-R5 | Reject full monorepo and reject 3-repos-per-product | Compliance blast-radius / tooling weight; vs repo sprawl / no atomicity | Accepted |
| ADR-R6 | Archive (never delete-first) repos that managed prod/regulated infra | Audit trail = compliance requirement | Accepted |
| ADR-R7 | No dedicated template repo — use GitHub template feature / `qnsc-ci` cookiecutter; revisit a scaffolder at product #3+ | Avoid premature repo; YAGNI | Accepted |
| ADR-R8 | Product monorepo scales to microservices via `apps/<service>/` + `packages/`; split to own repo only on team-split | Monorepo = code layout, not runtime; defers package-publishing overhead | Accepted |
| ADR-R9 | Deploy layer is **phase-aware**: ECS phase uses `qnsc-ci` + `deploy/ecs/`; EKS phase adds `chart/` + `qnsc-gitops` at migration. Same repo topology both phases | Matches ECS-now/EKS-when-triggered (architecture §8/§13); `qnsc-gitops` not created until the EKS trigger fires | Accepted 2026-07-01 |

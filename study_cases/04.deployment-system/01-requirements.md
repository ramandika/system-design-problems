# 01 — Requirements

## Scope decision

One product, two jobs that are usually two products: **CI** (build + test every commit) and
**CD** (move artifacts into environments safely). We design both because CD without CI's
artifact provenance is unsound, and CI without CD stops exactly where the money is. The
platform is multi-tenant: thousands of organizations, each with many repos, one shared
runner fleet and one shared artifact store.

## Functional requirements

**Pipeline definition & execution**

- Pipeline-as-code in the repo (a YAML manifest). The manifest resolves per event: a pull
  request event and a push event select different DAGs from the same file.
- The manifest declares a **DAG of jobs**, not a flat list: edges express "build → test
  (sharded ×8) → package → deploy-staging". Fan-out/fan-in is first-class.
- Trigger model: webhooks (push, PR, tag, schedule/cron, manual dispatch) and upstream
  events (a successful pipeline can trigger a downstream pipeline in another repo).
- **Matrix builds**: one job definition × a cartesian expansion of parameters (run the test
  shard matrix over 3 OSes × 2 versions = 6 concrete jobs).
- Job types are pluggable: shell steps, container-based steps, and a small set of platform
  services (artifact publish, cache restore/save, deploy gate).

**Artifacts & caching**

- Every run produces a stable identity: `(repo, commit SHA, pipeline)` → the run may publish
  named, immutable artifacts (containers, binaries, reports).
- A **content-addressed cache** for dependency layers and build outputs, keyed on declared
  inputs. Cache is per-repo-visible and never shared across trust boundaries by default
  (see 04, dive 2).
- Artifacts are immutable and content-addressed; promotion is by *address*, never by
  re-build ("the binary you tested is the binary you deploy" is a hard requirement).

**Deployment & rollout**

- Environments (`staging`, `us-prod`, `eu-prod`, …) are first-class objects with declared
  **concurrency policy** (how many simultaneous deploys), **required gates**, and
  approvers.
- Built-in rollout strategies: rolling, **blue/green**, **canary** (with metric-gated
  steps). Strategy is declarative: `canary: 1% → 5% → 25% → 50% → 100%, each gated`.
- **Health gates**: for each gate step, health is computed from metrics (error rate, p99
  latency, saturation) over a window, compared against a baseline (the pre-deploy version
  or a SLO target).
- **Automated rollback**: any gate failure reverts to the previous good artifact address.
  Rollback must work when the deploy controller's newer versions are partially live (mixed
  fleet).
- Full audit trail: who deployed what address to which environment, when, decided by which
  metric comparison. Auditable by people outside the platform team.

**Platform-level**

- Secrets: repos declare secret references; the platform mints **short-lived, scoped
  credentials** to jobs at start (no long-lived secret ever lands on a runner).
- Rich run UX: live log streaming, per-job status, artifacts, and re-run (a re-run reuses
  cached upstream results).

## Non-functional requirements

| Property | Target (rough, design-point) | Notes |
|---|---|---|
| Pipeline concurrency | ~100k steady, 250k burst ceiling | working-hours design point |
| New-run intake | ~500 runs/s peak start rate | scheduler must admit at this rate |
| Job pickup latency | p90 < 10 s from "queue" to "running" | users stare at this number |
| Run metadata durability | 0 loss of run records | status is transactional |
| Log/event loss tolerance | best-effort, detectable | logs are not the source of truth |
| Control-plane availability | rolling deploys of the platform itself, zero intake stoppage | the platform is always deploying itself |
| Artifact durability | ≥ 11 nines on unique content | same as cloud object storage (public target, "roughly") |
| Tenant isolation | no tenant can observe, starve, or spend another's quota | fairness and secrets both |
| Runner cold start | < 30 s VM, < 5 s warm-container pools | drives pool vs per-job VM trade-off |
| Metric gate freshness | gate decisions on data ≤ 60 s old | bounds the race in 04, dive 4 |

## Explicitly OUT of scope

- **Source control itself** (repos, PRs, code review) — the platform consumes webhooks.
- **Test frameworks** — the platform runs whatever command the manifest gives it; it does
  not understand tests beyond exit codes, parsed reports, and sharding.
- **The application metrics pipeline** — gates *query* an existing TSDB; we do not design
  the TSDB ingestion here.
- **Infrastructure provisioning** (Terraform-style IaC) — CD deploys *artifacts* to
  pre-existing environments; environment creation is another product.
- **On-prem hybrid agent management at depth** — we design a small "managed runners in the
  customer VPC" mode as a scope note only.
- **Per-job pricing/billing machinery** — quotas and fairness (04, dive 1) are in scope;
  invoicing is not.
- **Workflow marketplace/registry of third-party actions** — noted as a trust surface but
  not designed.

# 04 — Deep dives

## 1. Scheduling ~100k concurrent pipelines fairly (THE dive)

Every run becomes a DAG of jobs; every job needs a runner slot on the right SKU. At the
design point (02): ~200k live jobs, ~500 runs/s admission bursts, runner pools heterogeneous
(`linux-small`, `linux-large`, `gpu`, `windows`, `macos`). The scheduler is a queueing
problem stapled to a fairness problem.

**Option A — one global FIFO queue per pool.** Simple, perfectly fair per pool.
- wins: trivially correct, easy to reason about.
- loses: head-of-line blocking (a 4-hour GPU job queued behind nothing is fine, but a burst
  of one tenant's 10k-job matrix sits ahead of everyone); FIFO is *unfair* under
  concentration — whoever bursts first wins the whole window; no preemption.

**Option B — priority queue by tenant SLA.** Paying tiers sort earlier.
- wins: lets the business express intent directly.
- loses: starvation of the bottom tier is guaranteed at saturation; priority becomes a
  standing DoS on free tier p90; and priorities fight autoscaling (you never know whether
  to add runners or let the queue sort it out).

**Option C — partitioned queues with per-tenant fair dequeue (drained weighted round-robin
on top of per-pool queues).**

- Each runner pool has its own durable queue (claim-requests, lease-based). Dequeue iterates
  tenants in weighted round-robin by **dominant usage**: a tenant's weight decays with what
  it's *currently running* (jobs in flight, weighted by SKU cost), not what it has waited.
  A tenant that already holds 60% of `linux-small` capacity lands at the back of the round
  until others catch up — even if it just enqueued 10k more.
- Admission control kicks in above the burst ceiling (250k): tenants over their soft quota
  get queue-positions, others get rejections — backpressure must be visible and *fair*, not
  silent.
- Runner autoscaling reads queue depth *per SKU, per fairness group*: if the queue is deep
  but the deep part belongs to one tenant maxing its quota, adding runners is a mistake.

**Chosen: C.** A wins X (simplicity) but loses Y (fairness exactly when it matters —
saturation and bursts). B wins revenue alignment but loses universal starvation guarantees.
We pick C because the platform's core promise is "every tenant makes progress", and C gives
that with one dequeue loop; weighted priority can be a *tiebreaker inside* C later.

**Queue implementation**: durable per-pool queues in a sharded store (a claim is
`INSERT claim / SELECT next-waiting-per-tenant-fair` under row locks, or an in-memory
deque replica fed by the event bus with DB as authority). Claim = lease issuance with
**fencing token** (monotonic per job): a runner reporting with token 4 after the job was
re-claimed as token 5 is rejected. At-least-once claim delivery + client-side (runner-side)
fencing = the same "exactly-once as-felt" trick WhatsApp uses for message delivery —
see [WhatsApp dive 1](../03.whatsapp/04-deep-dives.md).

**Failure modes**:
- *Recovery stampede*: after a scheduler incident, 500k claim-requests flood back. Mitigate:
  fairness loop paces re-admission (replayed requests re-enter at round-robin weight, not
  head), runner heartbeat storms are rate-limited at intake.
- *Lease-expires-mid-job*: job heartbeat lags 5 s over TTL → re-claimed → two runners run
  the same job. Fencing makes the loser's writes (artifact publish, status, deploy
  proposal) inert; cost is duplicated compute, tolerated at ~0.1% rate.
- *Fairness metric gaming*: tenants split across many org accounts to double quota — quota
  must bind at the org-tree root (billing entity), not the repo.

## 2. Artifact store & build caching

Artifacts are the contract between test and deploy; the cache is the difference between a
50-minute and a 5-minute CI loop. Both want the same trick: **content addressing**.

**Store design**: chunks (e.g. 1 MB) hashed SHA-256, immutable, deduplicated globally per
region; a build's artifact set is a **signed immutable manifest** naming chunk hashes +
total size. `(repo, sha, manifest-hash)` is the only thing anyone ever references; GC scans
manifests' live-ref set (retention policies), not the chunk namespace — chunks whose refs
are gone become collectable in a second sweep-then-pause-detection pass. The
immutable-chunked-store shape echoes [YouTube's fixed 6 s video segments](../01.youtube/04-deep-dives.md),
but YouTube keys segments by `(video, rendition, index)` and does not content-dedupe them —
content addressing with cross-version dedup is a build-system idea (CAS-shaped, Bazel/Nix-style),
not a YouTube one; in 02's own numbers, a new version of a 350 MB app image changing ~10 MB
reuses ~97% of the old version's bytes.

**Why chunked dedup, not a whole-object versioned store**: the alternative — one immutable
object per artifact set, nothing shared between runs — wins operational simplicity (no chunk
namespace, no GC sweep) and is the right call for a small deployment. It loses on storage:
shared base layers, dependency archives, and re-run bytes are ~65% of the platform's volume
(02's 180 TB/day raw vs 63 TB/day unique), so at 90-day retention the whole-object store
holds ~16 PB where the chunked store holds ~6 PB. We pick chunked dedup because the artifact
store is the platform's largest store and a ~3× footprint cut also cuts the read amplification
(cache restores re-fetch the same shared chunks). The chunk size is the same trade: smaller
chunks dedup finer but blow up manifest and metadata size — ~1 MB is the rough knee where a
10 MB image change touches ~3% of a 350 MB image's chunks (~10 of ~350). Two-generation GC
(vs immediate refcount-to-zero collection) wins self-healing refcount races, loses ~48 h of
disk holding dead chunks — cheap against the deploy-time "artifact not found" failure it
prevents.

**Why signed manifests matter**: promotion by address (03) is only sound if the address's
bytes can't silently change. Chunks are content-addressed so they *can't*; manifests are
signed so a compromised store can't re-point a blessed tag at new bytes.

**Build cache — the key is the hard part.** Cache hits are only sound if the key captures
everything the output depends on:

- **Option A — time/name-keyed cache** ("latest cache for repo/branch"): trivial, hits
  often, and *wrong* — a dependency bump that doesn't invalidate the key yields stale
  builds whose tests pass on old code. Unsound by construction; rejected.
- **Option B — full input hashing**: hash every declared input (lockfiles, source globs,
  tool versions, base image digests). Key = H(inputs) → output. Sound, but misses:
  any input not declared (a transitive dep, a config file the build script read) breaks it.
- **Chosen: declared-input hashing (B) + verification at restore.** Restored caches carry a
  manifest of input hashes; the job verifies lockfile digests match before trusting the
  restore; mismatch → full rebuild. Trade-off made explicit: we accept ~2–5% false "misses"
  (rebuilds from undeclared-input conservatism) in exchange for never running tests on
  stale dependencies — an incorrect green checkmark is the platform's worst possible output.

Cache scoping is also a trust decision: **PRs from untrusted forks may read shared caches
only if the manifest verifies, and may *never write* to caches that base-branch runs read**
(cache poisoning: a malicious PR plants a trojaned dependency archive; the next trusted
main build restores it). Cache write admission = job's trust level (fork PR < first-party
branch < protected-branch release job). Failure mode if scoping is wrong: a supply-chain
compromise that executes *in your most privileged runners* — worse than a compromised app,
because it holds the secrets broker's clients.

**Scale math check (from 02)**: at 80% hit rate the registry/package egress avoided is ~8
PB/day; the CAS's own artifact read load (~3 GB/s average) lives fine behind a hot-object
cache for blessed/recent manifests. GC failure mode: a refcount bug collecting chunks still
referenced by a *just-published* manifest → deploy-time "artifact not found" of new content.
Mitigation: two-generation GC (collect only chunks unreferenced through two full GC cycles,
≥48 h), which makes any refcounting race self-healing.

## 3. Environment isolation & secrets

Two trust problems in one platform: runners execute **arbitrary untrusted code** (any PR),
and the same platform **mints production credentials**. The design must assume a runner is
hostile from the moment untrusted code lands on it.

**Compute isolation**:

- **Option A — long-lived shared runner hosts**, containers per job: cheap, fast dispatch,
  and a standing blast radius — kernel/container escapes, cross-job cache residue on disk
  (`~/.aws`, `.npmrc`), side-channel between tenants. Rejected as a default.
- **Option B — one ephemeral VM per job**: cleanest possible boundary (fresh kernel, disk
  dies with the job), costs ~30–60 s cold start and real money at 20M jobs/day (02's job
  volume; ~200k live at the design point).
- **Chosen: B as default, two mitigations for its cost**: (1) warm container pool *only* for
  jobs the platform itself trusts (platform-owned repos) where the blast radius is
  acceptable; (2) per-job VM boot from a pre-warmed image + snapshot (firecracker-style
  microVM startup is ~100s of ms) so the 30 s target holds. Trade-off accepted: slower
  pickup for untrusted PRs; never shared hosts across tenants.

**Secrets**: the rule is **runners never hold long-lived secrets — they hold leases.**

- A job's identity is a platform-issued token naming `(org_id, repo_id, run_id, job_id,
  ref, trust_level)`.
- The secrets broker federates that identity to each vendor (cloud STS-style OIDC
  exchange) and returns credentials scoped to *that job's declared needs* (a deploy job to
  `staging` cannot obtain `prod` DB credentials — environment scope is part of the token's
  claims, not the repo's secret list) with TTL ≤ 15 min.
- Long-lived secrets live in the vault; repos reference them by name. Rotation is the
  vault's problem, not 500k repos' problem.

Trade-off: minting adds a broker hop and a hard dependency (broker down = CI down) — we
accept availability coupling for breach-window minimization: a stolen runner credential is
worth at most 15 minutes of one job's scope, versus a year of a static key in `.npmrc`.

**Failure modes**:
- *PR secret leak by metadata*: a fork PR job that can read even *repo-level* env (e.g.
  `AWS_ROLE_ARN` strings in logs) — mitigation: fork jobs get a redacted token set and
  log scrubbing of `arn:`/key-shaped strings.
- *Workflow-trigger confusion* (the `pull_request_target` class of bug): a job that checks
  out *untrusted* code while holding *trusted* credentials. Mitigation: the broker's trust
  claim is derived from what was checked out, not what triggered — platform-enforced, not
  manifest-enforced, because manifests are written by the people most likely to get it wrong.
- *Broker as single point of exfiltration*: compromise of broker = all tenants' secrets.
  Broker runs in its own isolation domain, per-region, human access via quorum approval —
  it is treated with the trust level of a certificate authority, because functionally it is one.

## 4. Progressive rollout with automated rollback: the race between metrics and promotion

The deploy loop is a control system with two components that want to move at different
speeds: a **promote controller** (eager, event-driven, moves traffic) and a **health
evaluator** (passive, consults a TSDB whose data is 30–60 s stale by the time it's
queryable). Every bug in this subsystem is a race.

**Ownership first**: the evaluator is *advisory*; the controller is the *single writer* to
routing. One leader per rollout (lease + fencing token). Two rollouts of consecutive
versions to one environment are serialized by the environment's concurrency policy —
newer version's rollout waits or the older is cancelled first (declared policy picks).

**The core race — "healthy" computed on pre-promotion data**:

1. Controller promotes 1% → 5% at T=0, immediately asks the evaluator to judge 5%.
2. The TSDB's last 60 s of data is *mostly pre-promotion traffic* (the 5% slice is thin,
   and ingestion lags). The evaluator returns "healthy" having effectively measured the old
   version.
3. Controller promotes 25% on that verdict. If v2 was bad, you just amplified a mistake.

**Option A — time-bounded gates** ("each step holds ≥10 min, then decide"): simple, and
exactly what people naively build; but holding time ≠ fresh data — a 2-min ingestion stall
means a 10-min hold judged on minute-old data anyway.

**Option B — data-freshness bounding** (chosen): the evaluator refuses to return a verdict
unless the slice's data window satisfies: (a) `window_end ≥ promotion_time + Δ` where Δ
covers worst-case ingestion lag (measured from the TSDB's own ingestion watermark, not
assumed), and (b) the candidate slice has ≥ N requests so the delta isn't noise (a 1%
canary of a low-traffic service has no signal — the gate must say *insufficient data*,
and the controller holds, rather than promoting). Trade-off made explicit: Option B wins
correctness under lag and low traffic, loses rollout latency (holds wait for *provable*
fresh data rather than a wall clock — typically 12–15 min per step vs 10). We pay the extra
minutes; a fast wrong promote costs an outage, a slow right promote costs nothing.

**Rollback races** — four that must be handled explicitly:

- *Metric says rollback while controller promotes*: verdicts carry the fence token and the
  traffic-percent they were computed for; a rollback verdict for fence 4 percent-25 is
  ignored by fence 5 (which is at 50%) — instead it forces a fresh evaluation. Verdicts
  never jump the fence.
- *Two candidate versions live at once* (v2 at 5%, v3 canary starts): metrics must slice by
  version tag, not environment-wide; unsliced metrics blame v2 for v3's errors (and vice
  versa) — the evaluator's baseline diff is always version-tagged.
- *Flapping*: v2 fails → rollback to v1 → v1 shows pre-existing elevated errors (the
  baseline was polluted by v2 traffic) → "rollback failed" loop. Mitigation: baselines are
  computed from a clean pre-deploy window (fixed lookback before T=0), not rolling "current
  minus previous", so the rollback target is judged against its own steady state; a
  rollback that *also* trips gates escalates to humans (state `needs_human`, auto-retry
  disabled) rather than flapping forever.
- *Controller crash mid-promote*: DB row is the state machine (per 03); recovery resumes at
  the recorded step, re-issuing the fence at recovery — a re-issued fence with a new epoch
  invalidates any in-flight verdict from the crashed one.

**Failure modes beyond races**: metric gate silently stops firing because the version slice
label disappeared from a renamed pod label (gate reports *insufficient data* → hold → human,
never *healthy* — the evaluator must fail toward "no verdict", the controller fails toward
"hold", the whole chain fails toward "older version"); TSDB outage → all gates hold, deploys
stall — acceptable and visible, better than gate-less deploys.

## 5. Multi-region fleet updates (deploying the deployer)

The platform runs its own control plane across regions; each update is itself a deploy that
the mechanism in dive 4 executes. Two things make it harder than a customer deploy: the
platform is the recovery path for its own failures, and regions disagree about reality
during partitions.

**Wave structure**: dogfood tenant → internal staging → one production region (smallest
blast radius) → remaining regions in cohorts with **cooldown + platform-health gates between
waves**. Each wave's gate reads the same evaluator machinery as a customer canary (error
rate on intake, job pickup p90, scheduler depth, rollback-signal volume). A **global pause
switch** stops *scheduling new platform waves* and (if severe) intake itself; it is
consulted synchronously by intake.

**Option A — independent regions upgrade on their own schedule**: maximal autonomy, and the
worst case: a protocol bug in event serialization means region A (v5) writes transition
events region B (v4) rejects → cross-region run state diverges precisely during the rollout.
**Option B — globally serialized waves with a central coordinator**: one version of the
platform's *contract* (event formats, manifest schema, fence semantics) at a time; slow to
complete on a global fleet (hours), and the coordinator is a critical dependency.
**Chosen: B, with autonomy inside a wave** — each wave upgrades all its region's shards in
parallel, and the coordinator runs on a 3-node quorum so a coordinator-node loss pauses
rather than corrupts. We take hours of rollout latency to make "the whole world runs one
wire format" an invariant; contract-mismatch bugs during mixed versions are how fleet
rollouts actually fail.

**Partition behavior**: a region that loses the coordinator keeps running its *current*
version (leases expire; no new wave), which is the correct degraded mode — the dangerous
alternative is "continue the rollout on a stale plan". Regions rejoin by consulting the
coordinator's last-committed wave record, not local memory, so a region that was mid-wave at
partition either completes or reverts by the record, never by guesswork.

**Self-hosting discipline worth naming**: the platform's rollback target is the previous
blessed artifact *address* of the platform itself, pre-staged in every region (dive 2's
immutability is what makes pre-staging sound). Failure mode: fleet rollout of a version
whose *own* intake is broken deploys to region 1, gates pass on stale data (dive 4's race),
region 1 intake dies — mitigated by: global pause trips automatically when intake error rate
in *any* region exceeds the gate threshold, so the blast radius is one region plus whatever
intake backlog queues behind the pause. The queue (dive 1) absorbing the backlog is what
makes "pause intake, fix, resume" a non-event instead of an outage.

# 02 — Scale estimates (back-of-envelope)

All numbers are **rough estimates** built from stated assumptions, not measurements. Loose
public anchors: GitHub has publicly described Actions usage in the millions of executions
per day and enforces per-org concurrency caps by tier; Google SRE-style progressive delivery
is the norm at large shops. We size a mid-sized multi-tenant platform: GitHub-Actions-shaped.

## Assumptions

- **10,000 customer organizations**; ~500k active repos performing CI.
- **100,000 engineers (DAU).** A developer pushes by hand ~1.5 times on an active day; but
  every PR also lands as a merge push to main, CI bots push branches, dependency bots add
  commits, and re-runs re-push — across the org that multiplies to ~15 push events per
  engineer-day (est. 1.5 hand pushes × ~10 platform/bot/merge events) → say
  **1.5M pushes/day** (weekend decay ignored, averaged).
- Each push triggers an **average of 4 pipeline runs** (PR CI on the branch, post-merge CI
  on main, a deploy pipeline, plus retries/re-runs): 1.5M × 4 ≈ **6M push-triggered
  runs/day**.
- **Scheduled pipelines** (nightly tests, dependency rebuilds, cron deploys): ~4M/day by
  assumption (heavy customers run thousands of cron pipelines). 
- **Total ≈ 10M pipeline runs/day.** Each run averages **2 jobs** → **20M jobs/day**.

## Intake rate & concurrency

- Average intake: 10M / 86,400 s ≈ **115 runs/s**.
- Arrivals concentrate in working hours: assume **70% of runs fall in 10 working hours** →
  7M / 36,000 s ≈ **194 runs/s** working-hours average; bursts (batch merges, re-run storms
  after an infra incident) reach **~500 runs/s** — the scheduler admission design point.
- Mean run duration **W ≈ 10 min (600 s)** including queueing we do not promise to bound.
- Little's law, uniform assumption: L = λ × W = 115 × 600 ≈ **69k concurrent runs**.
- With working-hour concentration (λ ≈ 194/s): **~117k concurrent runs** during business
  hours; the remaining 30% spread over 14 off-hours → 3M / 50,400 s ≈ 60/s → **~36k
  overnight**. **Design point: ~100k concurrent (steady), 250k burst ceiling**
  (admission control kicks in above it).
- Job-level concurrency: ~2× run concurrency → **~200k live jobs** on the runner fleet;
  at any instant ~10–20% are in the CPU-heavy phase, so effective fleet planning is on
  memory/IO, not CPU-peak.

## Control-plane QPS

- State transitions: each run has ~5 status updates, each job ~4 → 10M × 5 + 20M × 4 ≈
  **130M row writes/day ≈ 1,500 writes/s average** (130M / 86,400 s ≈ 1,505/s), ~2,500/s
  working-hours average (70% of runs in 10 hours: 0.7 × 130M / 36,000 s ≈ 2,530/s; bursts
  higher). Sharded SQL handles this
  trivially; the design pressure is *consistency with the event stream*, not volume.
- API calls (UI polling, log tailing, status webhooks back to the SCM/git-host origin):
  ~30 calls per run → 300M calls/day ≈ 3,500/s average; reads are ~90% → **~10–20k QPS**
  design point with cache-friendly hot reads (recent runs for an org).
- Event stream (run/job lifecycle events to Kafka-shaped bus): 130M events/day plus log
  chunk appends. Log streams: each running job appends a **1/s batched chunk** for its
  duration → ~200k live jobs × 1/s ≈ **200k small appends/s** peak; per-stream rate averages
  ~1.7 KB/s (2 MB compressed per run ÷ 600 s ÷ 2 jobs/run), so 200k live streams produce
  ~340 MB/s during working hours — consistent with the 70%-concentrated ingest (~390 MB/s;
  0.7 × 20 TB/day ÷ 36,000 s) and the ~230 MB/s daily average under **Logs** below; producers
  coalesce into ~3,000 streams/s at the bus producer layer.
- Secret minting: ~1.5 credentials per job → 30M/day ≈ **350/s average** (347/s). Each is
  an asymmetric sign operation: ~350 signs/s is nothing for a small crypto fleet.

## Storage

**Artifacts** (content-addressed store):

- ~60% of runs publish artifacts; mean artifact set **30 MB** (container layers dominate).
  10M × 0.6 × 30 MB = **180 TB/day raw**.
- Content-addressing dedupes hard: base container layers, dependency archives, and re-runs
  repeat. A 350 MB app image changes by ~10 MB per commit; assume **~35% unique bytes** →
  180 TB × 0.35 ≈ **63 TB/day unique new content**.
- Retention 90 days (default) → steady-state unique store ≈ 63 TB × 90 ≈ **5.7 PB ≈ 6 PB**.
- Read volume: each artifact is fetched ~1.5× on average (deploy consumers, downstream
  jobs, human downloads) → 180 TB × 1.5 = 270 TB/day ≈ **3.1 GB/s average egress**,
  ~10 GB/s internal peak — internal-only, served from the CAS with a hot-object cache.
  (Same chunked-object physics as [YouTube's storage tier](../01.youtube/04-deep-dives.md);
  YouTube stores ~20 PB/day of video, the CAS takes in ~180 TB/day of artifact bytes —
  roughly 1% of it.)

**Logs** (build output, the big low-value store):

- Mean 2 MB compressed per run → 10M × 2 MB = **20 TB/day**.
- Retention: 30 days hot (search UI), then 60 days cold archive → hot tier ~600 TB.
- 20 TB/day ÷ 86,400 s ≈ 230 MB/s average ingest — a mid-sized log bus, no exotic design;
  the UX requirement (live tail) forces streaming, not batch upload.

**Run metadata** (durable, small):

- Runs: 10M/day × ~2 KB row ≈ 20 GB/day; jobs 20M × 1 KB ≈ 20 GB/day; with rollout records
  and gate evaluations call it **~50 GB/day ≈ 1.5 TB/month** — a sharded relational DB,
  partitioned by time, is the right tool and no one should reach for anything fancier.

**Metric-slice data for gates**: gate queries read *existing* application TSDB data over a
5–15 min window per gate: per gate ~20 series × 900 s ≈ small. Gate load is query-side only
(~500 gates/hour platform-wide ≈ negligible query QPS). Out of scope to store.

## Bandwidth

- Runner pulls: jobs pull inputs (repo tarball ~50 MB unpacked, caches) and push artifacts.
  Cache restore saves the big one: without cache, dependency install would pull ~500 MB/job
  on average; assume **80% cache hit rate** → 20M jobs × 500 MB × 0.2 = 2 PB/day of
  registry/package egress — that egress is against public registries, and is the strongest
  argument for the cache (dive 2).
- Internal artifact traffic ~3 GB/s average (above) and log streaming ~230 MB/s.
- Aggregate is dominated by *artifact and cache traffic*, an order of magnitude above
  control-plane traffic — sizing insight: the fleet's network is commodity, the CAS's is not.

## Consistency table (what needs what)

| Data | Needs | Why |
|---|---|---|
| Run/job status | strong (transactional rows) | duplicate/lost transitions corrupt UX and gates |
| Lifecycle events | at-least-once + consumer dedup | bus is for fanout, not authority |
| Artifact addresses | immutable, content-addressed | correctness = right bytes |
| Cache entries | best-effort, verifiable | wrong cache ≠ wrong build if keyed+verified |
| Gate metric reads | read-latest-≤60s | freshness bound, not linearizability |
| Secrets | never-at-rest-on-runner, short TTL | breach-window minimization |

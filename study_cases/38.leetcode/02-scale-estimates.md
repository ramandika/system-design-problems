# 02 — Scale estimates (back-of-envelope)

All numbers are **rough estimates** from stated assumptions, not measurements. Loose
public anchors: LeetCode's public pages have reported millions of registered users and
weekly contests with hundreds of thousands of participants; Codeforces and HackerRank run
events of comparable size. We size a LeetCode-shaped platform at the same order.

## Assumptions

- **10M registered users**, **1M DAU** (est.: registered-vs-active ratio ~10:1 for
  practice sites — most registrations churn without going active).
- An active user solves ~2 problems per active day; each problem costs on average **3
  submissions** (first attempt, one WA/TLE iteration, accepted) → 2 × 3 = **6
  submissions per DAU**.
- Submissions/day: 1M × 6 ≈ **6M practice submissions/day**.
- Add re-runs against test-set updates, contest replays, and API-driven traffic: +20% →
  6M × 1.2 = **7.2M ≈ 7M submissions/day** total.
- Each submission triggers on average **~12 test-case executions** (practice rounds run
  fewer; contest and full-suite rounds run ~50; compile failures run 0) plus one compile
  for compiled languages.

## Arrival rate & burst

- Average intake: 7M / 86,400 s ≈ **81 submissions/s**.
- Submissions concentrate in evenings/weekends: assume 60% fall in 8 peak hours →
  0.6 × 7M / 28,800 s ≈ **146/s peak-window average**.
- **Contest burst** is the design point and dwarfs this. Weekly contest:
  ~150k participants (publicly reported large weekly contests draw tens of thousands,
  with the biggest events exceeding 100k — rough anchor only; 150k is our assumption,
  est.), each ~70% submit at least once in the first hour;
  average 3 submissions/hour/person → 150k × 0.7 × 3 ≈ 315k submissions in the hour ≈ 88/s
  average — but arrivals are synchronised: the first 2 minutes after start carry ~10% of
  the hour's volume (everyone submits problem A immediately) → 31.5k / 120 s ≈ **263/s**
  sustained over the 2-minute window; we size for ~3× intra-window spikes, taking
  **~800 submissions/s as the sustained design point** (conservative headroom) and
  **~2k/s absolute spike** (assumed worst-case headroom), for the first run of each
  problem's test set.
- Judge wall time (estimate): a practice round (compile + a few eager test cases, per
  [01](01-requirements.md)) is ~5–6 s; the blended mean across all rounds is **~10 s**
  (compile 2 s + ~12 case executions × ~0.5 s + overhead); contest rounds run the full
  suite (~50 cases), ≈ **20 s** (2 s compile + ~35 *effective* case runs × ~0.5 s — a
  wrong answer terminates at the first failing case, so the effective mean for a 50-case
  suite is well below 50). Little's
  law at burst: L = λ × W = 800 × 20 ≈
  **16,000 concurrent judge slots needed** at the contest-start design point (contest
  arrivals are full-suite rounds, so use W ≈ 20 s, not the 10 s blended
  practice+contest mean); at the
  146/s practice peak, ≈ 1,500 slots. Baseline 81/s → ~800 slots.
- Fleet sizing: one sandbox ≈ 1 vCPU and ~512 MB–1 GB cap. A 64-core host fits ~50
  sandboxes (leaving headroom for the worker supervisor, I/O and per-sandbox slacks).
  ~16,000 slots ÷ 50 ≈ **320 judge hosts at contest peak**, vs ~1,500 slots ≈ **30 hosts**
  at the everyday practice peak — the contest burst multiplies the fleet ~10×, so
  pre-scaled warm capacity is a contest
  operational requirement, not an optimization.

## Queuing & languages

- **Per-language queues**: 20 languages, but the load is not uniform — Python and C++
  dominate (est. ~60% of submissions across the two, leaving ~15–20% for Java/JS and
  the rest long-tail). Per-language image pull/cold start:
  a cached ~500 MB image pulls in seconds from a local registry, but cold *sandbox* start
  is the real cost: fresh gVisor/microVM boot ≈ 0.3–1 s (namespace container ≈ 100–300 ms;
  microVM ≈ 0.5–2 s). Warm pools of pre-booted sandboxes per language absorb this; pool
  refill rate is the constraint at burst (800/s arrivals ÷ reuse of 5–20 runs per
  sandbox ⇒ refill churn ~40–160 sandbox-creations/s fleet-wide, plus warm-up headroom).

## Storage

- Submissions: ~7M/day × ~8 KB (source + metadata; small because source code only) ≈
  **56 GB/day** of submission records — trivial; keep hot (90 days) in SQL/row store and
  archive to blob storage.
- Test results (per-case stdout/stderr excerpts): cap retained output per run at ~64 KB
  (an Output-Limit-Exceeded attack run must not mint storage) → worst case 7M × 64 KB ≈
  450 GB/day, but typical runs store a few KB — **~50–100 GB/day realistic**.
- Verdict log (event stream): 7M verdicts/day + ~84M per-case runs with timing rows ≈
  91M rows/day, dominated by the per-case rows ≈ **~1,000 writes/s average**, ~1,800/s at
  the 146/s peak-window rate
  (146/s × 12 case rows), ~3,000/s with
  intra-window spikes (estimate) — one partitioned SQL or a
  log-shaped consumer ([Case 13](../13.kafka/README.md)) handles both; the contest burst is
  ~800 verdicts/s plus up to ~40k per-case rows/s (800/s × ~50 case rows), which is why
  the verdict log is an append-only stream rather than synchronous SQL writes
  (see [03](03-architecture.md)).
- Leaderboard state: per live contest, ~150k participants × per-problem scores in memory ≈
  150k × ~200 B ≈ **30 MB per contest** — leaderboard state is a cache-resident workload,
  not a storage workload.

## Delivery load (result fan-out)

- Each submission's owner polls-or-pushes: with verdict latency ~10 s, naive 500 ms
  polling costs 20 wasted polls per verdict (800 × 20 ≈ 16k QPS) — and if all ~150k
  standing participants poll every 500 ms regardless, that is 150k × 2/s ≈ **300k QPS**,
  absurd → polling must be exponential/pushed (see [03](03-architecture.md)/[04](04-deep-dives.md)). With SSE,
  delivery is ~1 message/verdict ≈ **800 msg/s during the burst**, plus leaderboard pushes:
  per participant a top-100 tick per 10 s plus an own-rank update per 30 s (the
  [01](01-requirements.md) freshness targets) ≈ 0.13 msgs/s → 150k × 0.13/s ≈ **20k msg/s
  outbound** at the contest peak across long-lived SSE/WebSocket connections — the real
  scale lever is standing connection count (~150k), not throughput.
- Standing connections: 150k concurrent SSE ≈ manageable on a handful of gateways if each
  gateway holds 10–20k connections; contest-peak connection set-up (150k in ~5 min) is a
  connection-handshake burst of ~500/s — fine, but it argues for contest clients opening
  sockets before T-0.

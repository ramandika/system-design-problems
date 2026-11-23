# 03 — Architecture

One decision pipeline at the gateway, with the *decision point tiered by which guarantee
the rule needs*: local token buckets for burst shaping, a sharded counter cluster for
hard cumulative quotas, no per-request round trip anywhere near the p99 budget.

## High level

```
                            CONFIG PLANE                              CLIENTS
                            ────────────                                 │
   ┌──────────────┐  push full    ┌──────────────┐    API request         ▼
   │ rule/service │  revisioned   │ snapshot blob │  (key+IP+route) ┌─────────────┐
   │ config store │──snapshots──▶ │ + revision no │                 │ edge / PoP  │
   │ (tiers, per- │  (static,     │ (every node   │                 │ LB / WAF    │
   │ key over-    │   CDN-able)   │  holds full   │                 └──────┬──────┘
   │ rides)       │               └──────────────┘                        │
   └──────────────┘                    ▲                                  ▼
        │ rule writes          revision broadcast                     ┌──────────────────┐
        ▼                      (each node pins last-known-good)       │ GATEWAY node (×~100)│
  admin API / dashboard                                              │ ┌──────────────┐ │
                                                                     │ │ local limiter │ │
        DECISION DATA PLANE                                          │ │ lease buckets │ │
        ────────────────────                                         │ │ (token buckets│ │
   API request ──▶ AUTH ──▶ rule resolution (from snapshot) ──▶      │ │  + rule cache)│ │
   ──▶ L1: LOCAL token bucket (burst-shaping rules; per-key lease    │ └──────┬───────┘ │
        refreshed in background)  ── admit/reject ──▶ headers ──▶    └────────┼─────────┘
                                                  │       │                   │
                                     lease refresh│       │ 429 + reason +    │ escalate only
                                     (async, batch│       │ Retry-After       │ (unknown IP,
                                     deltas, ~0.5s)       ▼      hard quota, ⌄
                                     ▼                            anomaly)   │
                       ┌─────────────────────────────┐                      ▼
                       │ quota authority (Redis-class│        ┌──────────────────────────┐
                       │ cluster, sharded by key hash│        │ counter store (Redis-    │
                       │ - per-shard lease manager   │◀──────▶│ class, ~10 shards)       │
                       │  for the hot-fat-key case)  │        │ - Lua-class atomic       │
                       └──────────────┬──────────────┘        │   decide-and-update      │
                                      │                       │ - chunked counters for   │
                                      │                       │   fat keys (deep dive 4) │
                                      ▼                       └────────────┬─────────────┘
                       monthly/hourly hard quotas,                       │
                       per-IP abuse verdicts                             │ decisions (sampled
                                      │                                  │ on backpressure)
                                      ▼                                  ▼
                            analytics / usage dashboards        Kafka-class decision log
                            billing rollups (async, replayable) ──▶ rollup jobs
```

## Components

- **Edge/PoP + gateway nodes (~100)**: terminate TLS, authenticate, then run the check
  pipeline. Each node holds (1) a *snapshot* of all limit rules in memory (02-scale-
  estimates: ~200 MB, affordable; deep dive 6), and (2) a *lease-bucket table*: token
  buckets for the keys this node has recently served, sized to its leased share of each
  key's allowance. In-process checks are ~µs — this tier carries ~95% of all checks
  (02-scale-estimates arithmetic) with zero network cost.
- **Quota authority (Redis-class cluster)**: the coordination plane for local buckets —
  grant/refresh leases for per-key throughput rules in the background, batched as deltas
  every ~500 ms per node, not per request. Also runs atomic decide-and-update scripts
  (deep dive 3) for requests that must be decided centrally.
- **Counter store (Redis-class cluster, ~10 shards)**: authoritative counters for the
  ~5% centralized tier — hourly/monthly hard quotas, per-IP abuse verdicts for
  unknown/suspicious IPs, anything a lease cannot express. Decisions are single atomic
  scripts (increment + compare + expire in one step). Hot keys get chunked counters
  (deep dive 4); the shards are the same tech family as
  [10.distributed-cache](../10.distributed-cache/README.md) but with write-heavy,
  no-read-mostly semantics.
- **Rule/config store**: tier defaults + ~1M per-key overrides as data. Publishes
  immutable numbered revisions as static snapshot blobs; nodes poll/broadcast for the
  current revision and pin last-known-good on fetch failure (deep dive 6). The
  coordinator/sequencing flavor is the same family as
  [11.zookeeper](../11.zookeeper/04-deep-dives.md).
- **Decision log (Kafka-class)**: every admission/429 emits one event,
  fire-and-forget, sampled or dropped under backpressure — feeds usage dashboards,
  billing rollups, and the abuse engine's feedback (IPs that keep hitting 429 get
  greylisted, which arrives as *new rules*, closing the loop through config).
  Never on the enforcement path.

## Data flows

**Admitted request (common case, local tier)**: request → auth → look up rules from
in-memory snapshot → decrement lease bucket in-process → append `RateLimit-*` headers
from the same bucket state → forward. No synchronous store access; the lease bucket
tops itself up from the quota authority asynchronously roughly every 500 ms.

**Centralized-tier request**: request → auth → rules → atomic script at the counter
store → decision + counter updated atomically → headers. One ~0.3–0.8 ms same-DC
round trip (02-scale-estimates), reserved for low-frequency high-stakes checks (monthly quota at 0:00, weird IP on a
suspicious endpoint).

**Rejection**: identical path up to the decision; the 429 carries the rule that fired
and a `Retry-After` derived from that rule's window/refill rate. Same cost as an
admission — checks are paid by attackers too (scope decision 3 in 01-requirements).

**Lease refresh (background)**: gateway node sums its local consumption per key since
last refresh, asks the quota authority for a lease renewal sized to its honest share
(consumption-proportional, capped); authority updates its ledger. Authority unreachable
→ node keeps the current lease with a decay factor (deep dive 5), which bounds
over-admission without making the counter store load-bearing.

**Config**: rule change → new revision published → nodes pull/are notified →
two-phase activation (prepare, then switch at a coordinated deadline) so the fleet does
not flap between old and new limits mid-deploy (deep dive 6).

## What is deliberately NOT here

- **No synchronous Redis look-and-decide on the burst tier.** The whole point of the
  lease design is that the counter store is not on any hot request's critical path.
- **No per-request rule lookup from a store.** Rules come from the in-memory snapshot;
  the on-path lookup is a hash, not a network call.
- **No separate "display" counters for headers.** `RateLimit-Remaining` reads the state
  that made the decision (scope decision 4 in 01-requirements), approximate by
  published design rather than approximate by accident.
- **No queueing/throttling-by-delay in v1.** Reject-or-admit only; admission control is
  a different (and dangerous) mechanism.

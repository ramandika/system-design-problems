# Safe deployment across the case catalog

Where canary / blue-green / rolling deploys, feature flags, shadow and dark launches, staged
rollouts, rollback, config-as-release, migration-without-downtime, and chaos-as-release practice
appear in the 50 study cases, what role each plays, and how deep each case goes. Companion to
[CONCEPTS.md](CONCEPTS.md) — unlike caching and the event backbone, this family was *not* kept as
a union there, so the counts below are the first cross-case tally of it.

Headline: **rollback speed is bought with immutability plus warmth, not with cleverness.** Every
case that can actually undo a bad change keeps (a) a content- or revision-addressed artifact with
the last-known-good still retained, and (b) a warm copy of it already running — so rollback is a
pointer move, a config activation, or a shard swap, never a rebuild. The one case that studies the
deploy machinery itself (04) spends its entire rollout deep dive not on the ladder but on the race
between metrics and promotion: the ladder is the easy part; a gate judged on stale data is the bug.

- Cases touching safe-deployment patterns: **26 / 50** (mention-based keyword scan, see "Filtering")
- A rollout/rollback/shadow mechanism is designed as part of the architecture: **7** (incl. 04)
- Brief to moderate single-pattern uses: **17**
- Analogy or named-and-rejected use ("this is not a feature flag"): **2**
- Explicitly rejected option (vs. rejected pattern): each *option* rejection is a ladder choice —
  big-bang and full-shadow are rejected in [08](08.chatgpt/README.md), full-shadow rollback and
  per-service isolation in [14](14.ml-production-system/README.md); the pattern itself is never
  rejected anywhere in the catalog.

## Summary table

| Case | Role | Depth |
|---|---|---|
| [04.deployment-system](04.deployment-system/README.md) | The case itself: gate/promote control system with automated rollback; deploys itself in waves | Full case |
| [08.chatgpt](08.chatgpt/README.md) | Model registry + canary router; shadow-before-canary ladder for model rollouts | Deep (designed) |
| [14.ml-production-system](14.ml-production-system/README.md) | Champion/challenger/shadow pointer table; rollback = repoint | Deep (designed) |
| [27.ev-charging-network](27.ev-charging-network/README.md) | Fleet firmware OTA: health-gated waves, A/B auto-revert, per-vendor canaries | Deep (designed) |
| [06.apple-app-store](06.apple-app-store/README.md) | Phased rollout as a post-approval release state (1%→…→100%, auto-pause) | Deep (designed) |
| [15.ml-ranking-system](15.ml-ranking-system/README.md) | Shadow/canary eval + blue-green shard warmup before cutover | Designed component |
| [31.netflix](31.netflix/README.md) | Chaos experiments in production, blast-radius budget, QoE-guardrail auto-rollback | Deep (designed) |
| [47.rate-limiter](47.rate-limiter/README.md) | Config as releases: immutable revisions, canary fleet slice, rollback = activate N−1 | Moderate |
| [30.facebook-messenger](30.facebook-messenger/README.md) | E2EE rollout: rollout flags per thread, two regimes coexisting for years | Brief–moderate |
| [24.mobile-chat-app](24.mobile-chat-app/README.md) | Local migration chain as the release mechanism; per-release backup undo path | Brief–moderate |
| [48.google-search](48.google-search/README.md) | Ranker releases: canary rollout + shadow scoring, minutes-class revert path | Brief |
| [10.distributed-cache](10.distributed-cache/README.md) | Post-restart staged warmup; slot migration copy-then-flip | Brief |
| [20.frontend-newsfeed](20.frontend-newsfeed/README.md) | Flag assignment at the edge, embedded in SSR payload (flicker-free A/B) | Brief |
| [39.e-commerce](39.e-commerce/README.md) | Kill switch on sale-page cache object; degradation ladder "scheduled like a deploy" | Brief |
| [49.monitoring-metrics](49.monitoring-metrics/README.md) | Rolling deploys as *load* on the metric system (churn, label-set recreation) | Brief |
| [38.leetcode](38.leetcode/README.md) | Per-language canary self-check job before a broken image reaches contests | Brief |
| [41.web-crawler](41.web-crawler/README.md) | URL-canonicalization rules as versioned config with canary/audit process | Brief |
| [16.dalle](16.dalle/README.md) | Shadow-evaluate new safety classifiers at deploy time | Brief |
| [26.google-maps](26.google-maps/README.md) | Shadow-scored model rollouts; staged key rollover per region | Brief |
| [45.typeahead](45.typeahead/README.md) | Double-buffer index swap: bad build is a rollback, not an outage | Brief |
| [03.whatsapp](03.whatsapp/README.md) | Deploy canary + connection drain as part of the protocol design | Brief |
| [13.kafka](13.kafka/README.md) | Dual-write both old and new partitions during partition-count migration | Brief (analogy) |
| [18.sora](18.sora/README.md) | Mid-job fleet rollout must not mix versions into one clip | Brief |
| [50.stock-exchange](50.stock-exchange/README.md) | Continuous standby verification doubles as a divergence canary | Brief (analogy) |
| [33.instagram](33.instagram/README.md) | "Ephemeral-at-scale … not a feature flag on the feed" — pattern named to deny it | Analogy / rejected |
| [25.mobile-newsfeed](25.mobile-newsfeed/README.md) | "The user's data plan, not a feature flag" — structural, not toggleable | Analogy / rejected |

## Per-case notes

### The case itself

**[04.deployment-system](04.deployment-system/README.md)** — a multi-tenant CD platform; the
release mechanism is a per-rollout state machine with single-writer fencing
([03-architecture.md](04.deployment-system/03-architecture.md)): declarative strategies
(`canary: 1% → 5% → 25% → 50% → 100%, each gated`), a promote controller racing a health
evaluator whose TSDB data is 30–60 s stale ([README.md](04.deployment-system/README.md)).
[Dive 4](04.deployment-system/04-deep-dives.md) is the sharpest rollout analysis in the catalog: the core race — a
"healthy" verdict computed on mostly pre-promotion traffic — is fixed by *data-freshness
bounding*, not hold timers (gate needs `window_end ≥ promotion_time + Δ` and ≥ N requests;
cost: 12–15 min/step vs 10, paid because "a slow right promote costs nothing, a fast wrong
promote costs an outage"). Four rollback races handled with fence-carrying verdicts,
version-tagged metric slices, baselines from a clean pre-deploy window (anti-flapping), and
a fail-toward-*hold*-toward-*older-version* posture for label-rename and TSDB-outage failures.
Fleet updates (dive 5) reuse customer-canary gates and choose globally serialized waves with
autonomy inside a wave — mixed wire-format versions are how fleet rollouts actually fail.

### Designed components

- **[08.chatgpt](08.chatgpt/README.md)** — model rollout as a pure routing problem
  ([04-deep-dives.md](08.chatgpt/04-deep-dives.md), dive 5): registry + canary router, version
  pinned per request at admission. Options weighed honestly: A big-bang (loses: all traffic hits
  a bad version at once), B full shadow (zeros user risk, doubles inference for days — "not a
  rounding error, it's a budget line"), C percentage canary with layered evals (0% → 1% → 5% →
  25% → 100%, guards include *safety-refusal-rate* as a first-class metric). Chosen C plus a
  <0.1% always-on shadow for hotfix validation; accepted cost: rollouts measured in days and the
  registry becomes a critical dependency (fails safe to last-known-good). Boundary: canary
  metrics lag product regressions — humans finish the ladder.
- **[14.ml-production-system](14.ml-production-system/README.md)** — the serving side of the
  same problem at platform scale ([04-deep-dives.md](14.ml-production-system/04-deep-dives.md),
  dive 4): champion/challenger/shadow splits as a cached pointer table over a content-addressed
  artifact store. Shadow = mirrored requests, scores logged not served; rollback = repoint
  `champion`, servers observe in seconds. Trade-offs stated: shadow wins risk-free evidence,
  loses ~2× serving cost during the window; pointer rollback wins seconds-level recovery, loses
  "one current artifact" simplicity (last-K versions stay warm or fast rollback dies). Named
  meta-failure: shadow that silently errors produces an *empty evidence set* that reads as
  "no problems" — shadow must report coverage, low coverage blocks promotion.
- **[27.ev-charging-network](27.ev-charging-network/README.md)** — the deployment target is
  firmware on 100K devices that cannot be reached again if bricked
  ([04-deep-dives.md](27.ev-charging-network/04-deep-dives.md)): staged waves gated on real fleet
  health (uptime, session success, CDR completion) sized to ~10 active rollout days inside a
  two-week budget with halt-and-resume slack; flash-slot auto-revert; per-vendor canaries as the
  cheapest catch for a 5-vendor test matrix. Sharpest lines in the catalog: the bootloader sits
  *outside* the A/B safety net (the thing that executes the revert cannot itself be reverted —
  slowest wave, signed on-flash fallback, per-hardware-gen canary), and a tariff change gets
  "the cheapest possible canary" — a shadow replay against the previous day of sessions before
  activation. Config rollout and firmware rollout land in one fleet plan.
- **[06.apple-app-store](06.apple-app-store/README.md)** — the ladder is a *state-machine
  state*, not a flag ([04-deep-dives.md](06.apple-app-store/04-deep-dives.md), dive 3):
  `RELEASED (phased)` advances 1%→2%→5%→10%→…→100% on 24 h windows with automatic pause on a
  crash-rate regression, and doubles as the throughput-shaping tool that caps the launch download
  wave. The same discipline applied to *platform* changes: ranking changes go out to holdback
  cohorts with metric guards and rollback is config-only
  ([04-deep-dives.md](06.apple-app-store/04-deep-dives.md), dive 2 failure modes); binaries keep
  last N≈4 versions for delta-base selection *and* rollback
  ([02-scale-estimates.md](06.apple-app-store/02-scale-estimates.md)). This is also where
  client rollout and server
  cutover meet: CDN pre-warm runs ahead of the rollout percentages because phased rollout caps
  the burst it creates.
- **[15.ml-ranking-system](15.ml-ranking-system/README.md)** — release discipline stated as a
  requirement and an architecture shape ([01-requirements.md](15.ml-ranking-system/01-requirements.md)):
  shadow evaluation and canary for model updates, serving never stops to rebuild the index —
  new shards warm up behind the old ones ([03-architecture.md](15.ml-ranking-system/03-architecture.md),
  blue-green shard swap). The ANN choice (rejecting HNSW) is justified *retroactively* by
  release safety: re-derivable-from-raw has a known freshness clock and cheap rollback;
  drift-rotted graph topologies fail silently ([04-deep-dives.md](15.ml-ranking-system/04-deep-dives.md),
  dive 2). "Quietly stale is worse than 2× more RAM" is a deployment-safety argument made inside
  an index-structure decision.
- **[31.netflix](31.netflix/README.md)** — release practice as *adversarial validation*
  ([04-deep-dives.md](31.netflix/04-deep-dives.md), dive 4): chaos experiments scheduled as
  routine, sized by a blast-radius budget (bounded concurrent experiments, % of fleet affected,
  never during peak block), automatic rollback on QoE guardrail breach. Trade-off: production
  chaos wins proof that fallbacks fire, loses short-term risk and chaos-first engineering cost
  (timeouts, fallbacks, idempotency); staging-only wins safety, loses — in its own words — any
  environment where 50M-peak session dynamics exist. The meta-failure modes are named: experiments
  that teach what you already suspected, and chaos decaying into ritual once remediation stops.

### Moderate: config-as-release and migration-as-release

- **[47.rate-limiter](47.rate-limiter/README.md)** — the purest config-rollout case
  ([04-deep-dives.md](47.rate-limiter/04-deep-dives.md)): rules ship as immutable numbered
  revisions; "rollback is a revision, not an investigation" (activate N−1 in seconds); new
  revisions canaried on a fleet slice (which sees the fat clients first, thanks to traffic
  skew). Every failure mode fails toward last-known-good, never toward no-config; tier defaults
  are compiled in as revision 0. 04's gate/promote machinery, specialized to stateless edge nodes.
- **[24.mobile-chat-app](24.mobile-chat-app/README.md)** — on the client, the *migration chain is
  the release mechanism* ([04-deep-dives.md](24.mobile-chat-app/04-deep-dives.md), dive 3):
  monotone versioned steps, copy-of-table instead of ALTER-in-place, per-release backup before
  big copies, preceding binary kept readable for one release — an expand-contract ladder at
  SQLite scale. "Migrations must not pile up at launch" is the client echo of 04's rollout
  pacing. [03-architecture.md](24.mobile-chat-app/03-architecture.md) lists feature flags as
  part of the local store design.
- **[30.facebook-messenger](30.facebook-messenger/README.md)** — an E2EE migration that turns
  progressive delivery *per entity*: two protocol regimes coexist for years by design
  ([04-deep-dives.md](30.facebook-messenger/04-deep-dives.md), dive 5), and the prescriptive line
  is "design rollout flags per thread, not per fleet" — the state machine of features per
  (thread, protocol version) is where incidents live. The publicly reported Messenger E2EE
  rollout (begun ~December 2023) is the anchor.

### Brief uses

- **[48.google-search](48.google-search/README.md)** — ranker releases get canary rollout +
  shadow scoring (score with new model, serve old, diff) plus an A/B revert path that *must* be
  minutes-class (NFR7), which forces coarse and heavy rankers to be independently deployable
  ([04-deep-dives.md](48.google-search/04-deep-dives.md)).
- **[10.distributed-cache](10.distributed-cache/README.md)** — two deployment-adjacent
  mechanics: staged warmup after restart (a restarted cache sharding that skips warmup causes
  the origin-pretection cliff of its own sizing) and slot migration as bulk copy then flip,
  where the migration's hit-rate cost is the design problem ([04-deep-dives.md]
  10.distributed-cache/04-deep-dives.md).
- **[20.frontend-newsfeed](20.frontend-newsfeed/README.md)** — flag delivery as an edge
  problem: assignment hashes (user ID + experiment salt) at the edge and writes variant flags
  into the SSR payload before HTML exists, so flag assignment can't flicker or invalidate CDN
  caching ([04-deep-dives.md](20.frontend-newsfeed/04-deep-dives.md), dive 6).
- **[39.e-commerce](39.e-commerce/README.md)** — kill switch: the sale PDP is a cache object
  with a kill switch, and the degradation ladder is "scheduled like a deploy", rehearsed before
  the season ([04-deep-dives.md](39.e-commerce/04-deep-dives.md)). It also names the cross-case
  lever: 06's phased rollout caps a download burst, this caps an order burst — same admission
  ladder, different expensive path.
- **[49.monitoring-metrics](49.monitoring-metrics/README.md)** — the *victim* of deploys: a
  500-pod rolling deploy is estimated to transiently churn ~10% of active series and re-create
  label sets, which the TSDB must absorb ([02-scale-estimates.md](49.monitoring-metrics/02-scale-estimates.md),
  [04-deep-dives.md](49.monitoring-metrics/04-deep-dives.md)). Rolling deploys as a load event
  on production systems, not as a practice.
- **[38.leetcode](38.leetcode/README.md)** — canary specialized to judges: one self-check job
  per language runs a known-good reference solution on deploy, so a broken compiler image is
  caught in minutes ([04-deep-dives.md](38.leetcode/04-deep-dives.md), explicitly by-reference
  to [04](04.deployment-system/04-deep-dives.md)).
- **[41.web-crawler](41.web-crawler/README.md)** — canonicalization rules treated as versioned
  configuration with a canary/audit process (sampled merges, human review) because a silent bad
  merge has no error signal ([04-deep-dives.md](41.web-crawler/04-deep-dives.md)).
- **[16.dalle](16.dalle/README.md)** — new safety classifiers are shadow-evaluated against
  production traffic at deploy; regression at deploy → shadow before serving
  ([04-deep-dives.md](16.dalle/04-deep-dives.md), [03-architecture.md](16.dalle/03-architecture.md)).
- **[26.google-maps](26.google-maps/README.md)** — serves under the 14 discipline: versioned
  features, shadow-scored model rollouts, historical fallback when the model service degrades;
  key rollover staged per region ([04-deep-dives.md](26.google-maps/04-deep-dives.md)).
- **[45.typeahead](45.typeahead/README.md)** — double-buffer swap: a bad index build is "a
  rollback, not an outage" ([04-deep-dives.md](45.typeahead/04-deep-dives.md)).
- **[03.whatsapp](03.whatsapp/README.md)** — one line with big consequences: "deploy canary +
  drain is part of the protocol design, not an ops afterthought" ([04-deep-dives.md]
  03.whatsapp/04-deep-dives.md) — rolling deploys of connection-heavy servers need protocol
  support (drain) built in.
- **[13.kafka](13.kafka/README.md)** — dual-write as data-plane expand-contract: changing
  partition count is a migration, not an ops command, so *double-write both old and new
  partitions during the transition* or keep an explicit key→partition mapping service
  ([04-deep-dives.md](13.kafka/04-deep-dives.md)). Same shape as config/config-version
  coexistence, applied to ordered streams.
- **[18.sora](18.sora/README.md)** — version pinning as the boundary of mixed-fleet rollouts: a
  fleet-wide rollout mid-job must not silently mix versions into one clip; resume only against
  the pinned version ([04-deep-dives.md](18.sora/04-deep-dives.md)). The request-pinning rule
  from 08/14 reappears where the artifact is a video.
- **[50.stock-exchange](50.stock-exchange/README.md)** — canary as a *state* concept, not a
  release one: the standby replicating and provably verifying continuously doubles as a live
  divergence canary alarming long before failover is needed
  ([04-deep-dives.md](50.stock-exchange/04-deep-dives.md), dive 4).

### Analogy / named and rejected

- **[33.instagram](33.instagram/README.md)** — "ephemeral-at-scale is its own storage and
  consistency problem, not a feature flag on the feed": the pattern is invoked *to deny* that
  the problem can be toggled into existence ([README.md](33.instagram/README.md)).
- **[25.mobile-newsfeed](25.mobile-newsfeed/README.md)** — the same move against image
  download: the cost belongs to "the user's data plan, not a feature flag"
  ([README.md](25.mobile-newsfeed/README.md)). Both mean: progressive delivery controls *exposure
  of built things*; it is not a substitute for building and sizing.

## The portable pattern

Every substantive use reduces to one contract, with [04.deployment-system](04.deployment-system/README.md)
as the reference machinery:

1. **Rollback is pointer arithmetic on an immutable artifact.** Content- or revision-addressed
   bytes, last-known-good still retained (04's signed manifests + pre-staged addresses, 06's
   N≈4 binaries, 47's immutable revisions, 14's retained versions, 31's documented regressed
   ladder). Nothing rebuilds during a rollback, or you don't have one.
2. **Keep the old thing warm.** 14 keeps last-K versions loaded, 15 warms new shards behind old
   ones, 47 pins last-known-good in the node, 06's full-download fallback ships from kept delta
   bases. Warmth is the price of seconds-to-minutes revert.
3. **Gate on slice-tagged, freshness-bounded metrics — and fail toward "no verdict" → "hold" →
   "older version".** 04's data-freshness bounding is the formal version; 47's fail-toward-
   last-known-good and 06's automatic pause are its specializations. A gate that can answer
   "healthy" on stale or empty data is the deploy-loop's only real bug.
4. **Percent ladders shape cost as well as risk.** 04: 1→5→25→50→100; 08: 0→1→5→25→100 over days;
   06: 1→2→5→10→…→100 over 24 h windows; 27: waves sized ~10 K devices/day inside a two-week
   budget. The same ladder doubles as burst admission (06's download cap; 39's version of it).
5. **Shadow and canary cost ~2× of the evaluated path and are bought only for the most expensive
   failure class.** 08 refuses full-fleet shadow on money; 14 accepts shadow on evidence because
   a bad artifact is the platform's highest-cost failure; 27 gets canary at near-zero cost by
   replaying yesterday's sessions.
6. **Boundaries.** The bootloader problem (27): whatever executes the revert cannot be protected
   by the revert — it needs its own slower, more conservative path. Mixed versions in one
   *request* are never allowed (08, 14, 18), while mixed versions across *entities* may coexist
   for years when deliberately per-entity (30); chaos (31) is release practice whose evidence is
   destruction, gated on the same guardrail machinery as a canary. Nothing in the catalog rejects
   progressive delivery as a practice — the rejected options are big-bang (08, 14) and
   hold-timer-only gates (04).

## Filtering

26 case directories above come from a keyword scan (`canary`, `feature flag`, `shadow traffic`,
`mirror traffic`, `blue-green`, `rolling deploy`, `rollback/dual-write/expand-contract`,
`staged rollout`, `kill switch`, `dark launch/mode`, A/B hits). The following matched tokens but
were excluded as false positives, verified in context:

- `gating` used for **domain policy, not releases**: [34.reddit](34.reddit/README.md) (karma
  thresholds), [32.spotify](32.spotify/README.md) (region gating = catalog filter), [36.airbnb]
  (36.airbnb/README.md) (review gating on verified stays).
- `staged` used for **in-request pipeline stages or timelines**: [42.linkedin](42.linkedin/README.md)
  (staged ranking funnel), [29.uber-eats](29.uber-eats/README.md) (courier arrival timeline),
  [37.ticketmaster](37.ticketmaster/README.md) (pre-staged requests), [42.linkedin]
  42.linkedin/03-architecture.md ("mirroring 15" is a citation, not mirror traffic).
- `mirrors` used as prose ("mirrors Case 19's", "mirrored client-side"), and 09.cdn's mirror
  *tier* of edge caches: [09.cdn](09.cdn/README.md), [19.frontend-chat-app](19.frontend-chat-app/README.md),
  [25.mobile-newsfeed](25.mobile-newsfeed/README.md) at line 234.
- `rollback` in domain state machines and migration prose: [22.frontend-uber](22.frontend-uber/README.md)
  (defined state-machine rollback events), [30.facebook-messenger](30.facebook-messenger/README.md)
  "scrollback" hits, [07.blob-store](07.blob-store/README.md) on tombstone retention — this last
  one is a judgment call: soft-delete-as-rollback-underpinning is a *deployment adjacency* and is
  covered indirectly via rule 1 above.

Mention-based counts (26/50) are not CONCEPTS.md unions and are not comparable to the 23/43
event-backbone tally; this family was not built as a union there.

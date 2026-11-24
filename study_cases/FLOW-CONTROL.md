# Overload & flow control across the case catalog

Where backpressure, admission control, load shedding, circuit breakers, and graceful degradation
appear in the 50 study cases, what mechanism each case actually uses, and how deep it goes.
Companion to [CONCEPTS.md](CONCEPTS.md) — its nearest row is "Rate limiting / admission control"
(12/43, Tier 2, technique), and there is **no backpressure/overload row at all**, so the counts
below are the first cross-case pass on that family. They are mention-based keyword scans and are
NOT comparable to CONCEPTS.md union counts.

Headline: **Overload response in this catalog is priority dropping, decided in advance — almost
never failure detection at runtime.** Every case that sheds declares *what it drops and in what
order* before the incident, and the order is nearly constant: telemetry, batch work, and derived
planes first; the transactional record last or never. The textbook circuit breaker (trip on an
error signal, fail over to cached behavior, self-heal) is named only three times — 27 (partner
token pulls), 38 (judge fleet), 10 (cache-stampede walls) — and even 27, the closest to
classic form, trips on an upstream outage with a cached fallback, not on a request-error-rate
proxy. Nobody trips a breaker on 5xx percentage; the load instrument everyone reads is
**queue depth**, at the boundary of the expensive plane.

- Case dirs matching the keyword net (backpressure, circuit breaker, load shed, admission
  control, queue depth, throttle, degrade, 429/503): **48 / 50** — `degrade` alone is nearly
  useless (48 hits; see Filtering). After reading every hit in context, **30 / 50** have a
  genuine flow-control mechanism.
- A declared drop order exists in **~10**: 08, 09 (edge tier), 15, 26, 28, 31, 39, 45, 50, 43.
- Queue depth is the load instrument in **8**: 04, 05, 06, 12, 16, 17, 38, 49.
- Visible-queue admission as a designed product surface: **5** (37, 39, 08, 16, 18); refused:
  **2** (40 drops instead of queueing, 47 "no queueing/throttle-by-delay in v1").
- Named "circuit breaker": **3** (10, 27, 38). Bulkhead by name: **0** — the equivalent is built
  from statelessness and partitioning and called cell / pool / per-partner adapter.

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [37.ticketmaster](37.ticketmaster/README.md) | The case itself: the waiting room *is* the load-shedding interface; admission order is the hard problem | Full case |
| [08.chatgpt](08.chatgpt/README.md) | Four-rung overload ladder + GPU admission; queues convert overload into TTFT, not errors | Designed |
| [16.dalle](16.dalle/README.md) | Queue buffer above fleet capacity; "queue depth becomes the product surface" | Designed |
| [18.sora](18.sora/README.md) | Cost-based admission (GPU-minutes as credits); publication backpressure, generation never | Designed |
| [39.e-commerce](39.e-commerce/README.md) | Five-layer flash-sale ladder, cheapest rejection first; admission at inventory-drain rate | Designed |
| [04.deployment-system](04.deployment-system/README.md) | Admission above the burst ceiling; queue-depth autoscaling per fairness group | Core component |
| [05.discord](05.discord/README.md) | Per-socket backpressure: drop live pushes, never queue a slow socket | Core component |
| [10.distributed-cache](10.distributed-cache/README.md) | Stampede circuit breakers: admit-window on SETs, self-throttling warmup, soft TTL | Core component |
| [12.bigtable](12.bigtable/README.md) | Write backpressure past 2 frozen MemTables; master throttles tablet reassignment | Core component |
| [27.ev-charging-network](27.ev-charging-network/README.md) | Storm admission shaping (token-bucketed replay intake); per-partner backpressure + breaker | Core component |
| [28.uber](28.uber/README.md) | Edge gateway drops a position sample, never a lifecycle event | Core component |
| [07.blob-store](07.blob-store/README.md) | Throttle new ingest ahead of durability rebuild; per-rack repair caps | Moderate |
| [15.ml-ranking-system](15.ml-ranking-system/README.md) | Ranker overload: hard candidate caps, shed to the pre-ranker path | Moderate |
| [17.elevenlabs](17.elevenlabs/README.md) | Send-side backpressure bound, not buffering; interactive/batch pool boundary | Moderate |
| [26.google-maps](26.google-maps/README.md) | Hot-region overload: shed ETA decoration, never the route | Moderate |
| [31.netflix](31.netflix/README.md) | Regional load shedding with a degradation ladder of row quality | Moderate |
| [33.instagram](33.instagram/README.md) | Ranker-down fallback to reverse-chron is a required product path | Moderate |
| [38.leetcode](38.leetcode/README.md) | Upstream queue admission per user; global circuit breaker; fail open to local bucket | Moderate |
| [41.web-crawler](41.web-crawler/README.md) | The client side of flow control: honor 429/503 + Retry-After as a contract | Moderate |
| [45.typeahead](45.typeahead/README.md) | Per-tier timeout-and-degrade; return empty fast; per-node load-shed | Moderate |
| [49.monitoring-metrics](49.monitoring-metrics/README.md) | Remote-write backpressure with an explicit, monitored drop policy; push retry storms | Moderate |
| [50.stock-exchange](50.stock-exchange/README.md) | Journaled, disclosed throttle events; throttle depth, never trades | Moderate |
| [02.twitter](02.twitter/README.md) | Fanout hybrid exists to bound the worst-case write fanout | Brief |
| [03.whatsapp](03.whatsapp/README.md) | Offline queue bounded by TTL, not by admission | Brief |
| [06.apple-app-store](06.apple-app-store/README.md) | Review-queue depth alert + expedited-quota shrink; phased download admission | Brief |
| [34.reddit](34.reddit/README.md) | Vote ingestion shaped so backpressure hits staleness, never content writes | Brief |
| [44.payment-system](44.payment-system/README.md) | Idempotency-store outage → 503 reject-conservatively, never process unkeyed | Brief |
| [09.cdn](09.cdn/README.md) | Edge-tier overflow; POP overload → BGP sub-portion withdrawal; log drops by class | Boundary |
| [40.tinyurl](40.tinyurl/README.md) | Drop-on-backpressure click telemetry (KAFKA.md contract) | Boundary |
| [47.rate-limiter](47.rate-limiter/README.md) | The limiter itself; refuses queueing as a flow-control design | Boundary |

## Per-case notes

### The case itself

**[37.ticketmaster](37.ticketmaster/README.md)** — the catalog's most explicit thesis about
admission control: "most visitors are *not* shown inventory … the waiting room is the
load-shedding interface" ([01-requirements.md](37.ticketmaster/01-requirements.md)), which
"inverts the usual web instinct". Arithmetic that forces it: ~2M arrivals at the on-sale instant
against an inventory tier absorbing ~15–20K seat ops/s; 2M users polling position at 1/3 s ≈
650K req/s. Three admission orders weighed (FCFS loses to bots; lottery feels unfair; chosen:
FCFS within randomized burst cohorts), plus the counter-case giving depth: "admission control is
better than degradation here" — degrading the map tier under 2M users melts it. Then signed TTL
tokens, coarse position counters, and the named admit-rate vs hold-conversion mismatch.

### Designed overload machinery: the queue as product, admission as budgeter

The GPU/flash-load family: all four price admission in the scarce resource, and all four state
the trade (queue latency vs unbounded cost).

- **[08.chatgpt](08.chatgpt/README.md)** — tiered ladder
  ([04-deep-dives.md](08.chatgpt/04-deep-dives.md)): (1) queue with position/ETA, since
  "queues convert overload into TTFT, not errors"; (2) shed free tier *before* GPU work;
  (3) downgrade model ("a *quality* trade used deliberately as a pressure valve");
  (4) priority preemption — never a paid stream mid-answer. Admission at ~17K req/s under 500K
  in-flight vs p99 TTFT ≤ 2 s; stated trade: approximate fairness (eventually-consistent
  counters, lagging refunds) over a consistent ledger that would put global coordination in
  front of every prompt.
- **[16.dalle](16.dalle/README.md)** — reserved/spot/queue split where the queue "absorbs
  everything above fleet capacity. Queue depth becomes the product surface: show position/ETA,
  don't hide it." Free tier fills leftover batch slots; the named risk is priority inversion
  (a free job preempted while paid jobs queue), bounded by tiered routing.
- **[18.sora](18.sora/README.md)** — Option A (generate everything) rejected with an exact
  trade-off: "a viral moment with a 'generate everything' meme and a bot farm is a
  budget-destroying event; no honest mechanism to shed load." Chosen: every submission priced in
  expected GPU-minutes, credits as budget, "the queue *is* the product"; and flow control has a
  direction — review-queue backpressure deliberately slows *publication*, not generation.
- **[39.e-commerce](39.e-commerce/README.md)** — five-layer ladder
  ([04-deep-dives.md](39.e-commerce/04-deep-dives.md)), "ordered from cheapest rejection to most
  expensive": kill-switched static edge page → waiting room on flagged SKUs (signed queue
  tokens, no centralized counter; admission rate = inventory drain rate) → per-SKU gateway token
  budget with 429+retry-after upstream of queueing → read-plane degradation rungs, "each rung
  pre-enacted config". Stated trade: the queue "trades perceived latency for predictable success"
  and honest visible waiting measurably reduces refresh storms.

### Core component: shed the derived plane, keep the record

The most portable shape: each case carries a guaranteed class (the write path, the message, the
event) and a droppable class, enforced mechanically at the gateway/buffer — not retrospectively.

- **[05.discord](05.discord/README.md)** — per-socket backpressure beyond a buffer watermark:
  "for a slow socket the gateway *drops live pushes* rather than queuing, because the store has
  the message and a reconnect/catch-up will re-sync. Losing a live push temporarily is
  self-healing; queueing to a slow socket is how fanout storms become OOM cascades." Hot-guild
  handler: channel-level rate cap + "channel is busy, fetch history" client state; fanout
  backlog detected via queue depth, re-published idempotently.
- **[28.uber](28.uber/README.md)** — the edge gateway's "one design job: apply *backpressure to
  telemetry* (drop a position sample, never a lifecycle event) so a burst cannot melt the ingest
  path." One sentence, whole doctrine.
- **[10.distributed-cache](10.distributed-cache/README.md)** — the stampede case. The eviction
  avalanche is spelled out as a feedback loop (pressure → misses → misses add SET writes → more
  pressure → 100% miss), with named "circuit breakers": admit-window on new SETs under pressure
  ("don't admit what you can't keep") and origin-side rate limiting as the final wall;
  plus self-throttling warmup (a restarted service ramps its pass-through 1% → 100%), soft TTL,
  and negative caching.
- **[12.bigtable](12.bigtable/README.md)** — backpressure on write rate past 2 frozen MemTables
  (the one place a Bigtable write is not flat); and the master *throttles reassignment* after a
  server death so replaying ~50 tablets' logs does not stampede the store. Overload control
  inside the storage engine, not at a gateway.
- **[27.ev-charging-network](27.ev-charging-network/README.md)** — storm handling as admission
  shaping, not connection throttling: reconnect storms admit connections immediately (a charger
  is a paying customer mid-trip) but shape *replayed message intake* per charger with a token
  bucket, because the replay burst (~0.6–2.5× the ~5–7K peak budget by 02's arithmetic) would hit
  the bus's per-key headroom. Also per-partner fanout backpressure ("a partner outage slows only
  their push") and a genuine textbook breaker on the roaming-token pull path: cached-token
  fallback with explicit "offline refuse" rather than hang. (Verified: none of 27's "circuit"
  mentions are electrical — all four are these fanout/breaker lines.)

### Moderate: declared ladders and single-mechanism throttles

- **[15.ml-ranking-system](15.ml-ranking-system/README.md)** — ranker overload (a campaign
  multiplies candidates per request): "hard candidate cap per source, load shedding by dropping
  to the pre-ranker path". Stages time out independently and degrade individually.
- **[26.google-maps](26.google-maps/README.md)** — the cleanest priority statement: hot-region
  overload → "load-shed ETA decoration first (routing still answers with historical speeds),
  never shed the route itself"; on the traffic pipeline, backpressure drops *sampling coverage*
  before dropping *freshness* on hot segments.
- **[31.netflix](31.netflix/README.md)** — "regional load shedding with a *degradation ladder*
  — drop personalization depth → drop row diversity/similarity freshness → generic-but-non-empty
  rows → cached catalog with a 'something went wrong' header." Framed as chaos-drill material
  (see DEPLOYMENT-SAFETY boundary in the pattern below).
- **[33.instagram](33.instagram/README.md)** — "worst case the feed API must degrade to
  reverse-chron over the pool … that fallback is now a required product path, not a
  nice-to-have" — degradation moved from ops handbook to product surface.
- **[45.typeahead](45.typeahead/README.md)** — timeout-and-degrade per tier (edge 10 ms, cache
  3 ms, index 25 ms); "an empty box costs the user a keystroke; a 400 ms suggestion list costs
  the whole interaction"; per-node load-shed as a GC-pause countermeasure.
- **[38.leetcode](38.leetcode/README.md)** — under 15 s latency budgets, "backpressure has to be
  *upstream* (queue admission per user) rather than downstream (long queue wait)"; per-user caps
  plus "a global circuit breaker that degrades to 'verdicts delayed, queue position visible'
  instead of dropping submissions"; limiter store down fails *open to the local per-instance
  bucket* — briefly under-limiting beats blocking submissions.
- **[17.elevenlabs](17.elevenlabs/README.md)** — real-time audio: "backpressure matters more
  than buffering on the send side — pause text intake rather than buffer unbounded audio; a
  stream whose output buffer exceeds a few seconds of media is errored and re-established."
  Batch vs interactive pools hard-partitioned because a shared queue couples "interactive p95
  to batch queue depth — exactly the coupling a latency SLO can't contain"; runaway batch
  watched via queue depth + interactive RTF, with automatic pool shrink.
- **[49.monitoring-metrics](49.monitoring-metrics/README.md)** — remote-write backpressure:
  block upload queue fills → drop policy "must be explicit and monitored, because 'which 2 h
  window silently vanished' is a question on-call will eventually ask". Push-vs-pull: a sick
  fleet with push "pushes harder (retry storms) exactly when the monitor should throttle".
- **[41.web-crawler](41.web-crawler/README.md)** — flow control from the other side of the TCP
  connection: honor `Retry-After` on 503, "treat 429 rate-limit responses as a host-level signal
  to back off (even though 429 is not robots, it is a contract)"; on robots fetch failure,
  last-known-good for a bounded grace window, then a conservative crawl-delay default — "never
  an implicit allow all".
- **[50.stock-exchange](50.stock-exchange/README.md)** — a hot symbol over capacity → "the venue
  throttles *new orders* for that symbol via an explicit, journaled throttle event (a designed,
  disclosed action — not silent queueing, which turns a fairness problem into an
  unbounded-latency problem)". Fanout split: trades never dropped or coalesced; depth updates
  coalesce latest-write-wins ("throttle depth, never trades"); post-flap replay storms are
  rate-limited and staggered.
- **[07.blob-store](07.blob-store/README.md)** — recovery-time flow control: "orchestration
  should throttle *new* ingest ahead of durability rebuild (write-pause: users wait hours for
  PUT, durability doesn't wait)"; repair priority-weighted within a ~10–20% I/O budget with
  hard per-rack caps so the post-rack-recovery storm is self-throttling.

### Brief

- **[02.twitter](02.twitter/README.md)** — the fanout hybrid is itself an overload decision
  ("bound the worst-case write fanout" of 100M inserts for a celebrity post), and celebrity
  drip-through-a-queue is load leveling; per-account limits are abuse (→ 47).
- **[03.whatsapp](03.whatsapp/README.md)** — the offline queue is bounded by TTL (30 days, then
  drop or client re-pull) rather than by admission; "infinite queueing = unbounded storage" is
  the named trap.
- **[34.reddit](34.reddit/README.md)** — event-log ingestion chosen exactly so "backpressure hits
  vote *staleness*, never content writes"; the guarantee class is defined by what is allowed to
  lag, and votes (seconds of lag OK) sit deliberately outside it.
- **[06.apple-app-store](06.apple-app-store/README.md)** — review-queue SLA run: "queue depth
  alert + category routing rebalance; expedited quota shrinks to protect the SLA"; delivery
  bursts shaped by phased download admission (same lever as [39](39.e-commerce/README.md)'s,
  applied to bytes instead of money).
- **[44.payment-system](44.payment-system/README.md)** — idempotency-store outage → "reject
  writes conservatively (503), never process unkeyed": fail-closed integrity, not capacity
  shedding, kept because it is the catalog's cleanest money-path refusal.

### Boundaries (owned elsewhere — linked, not duplicated)

- **[47.rate-limiter](47.rate-limiter/README.md)** — owns the limiter: token buckets, quotas,
  429 semantics, and its design refusal of queueing as flow control. The limiter is admission
  for *abuse/quota*; this doc is admission for *capacity* — same counters, different failure.
- **[40.tinyurl](40.tinyurl/README.md)** — drop-on-backpressure click telemetry is owned by
  [KAFKA.md](KAFKA.md)'s "fire-and-forget producers" contract (lagging rollups, drops accepted).
- **[09.cdn](09.cdn/README.md)** — both hits are edge-tier: POP overload sheds "the most distant
  traffic first" via BGP sub-portion withdrawal; the log pipeline drops analytics before billing.
  Edge-layer overflow belongs to [CDN.md](CDN.md), which does not yet tally these.
- **[43.zoom](43.zoom/README.md)** — not a row: its waiting room is *moderation* admission (a
  permission gate), and "drop video layers before audio" is bandwidth congestion control owned
  by [REALTIME-TRANSPORT.md](REALTIME-TRANSPORT.md) (already a Full-case row there).

**Adjacent graceful degradation, not counted** — component-failure fallbacks with no overload
mechanism and no declared drop order: [01.youtube](01.youtube/README.md), 29 (stale menus),
[32.spotify](32.spotify/README.md) (per-track), [36.airbnb](36.airbnb/README.md),
[42.linkedin](42.linkedin/README.md) (feed variety, badge staleness), 13's ISR degradation.
Availability design, not flow control.

## Filtering — false-positive dictionary

Mention-based scans hit 48/50 dirs; these matched the net but were excluded after reading each hit in context:

- `degrade` as component-failure fallback or performance decay: all the "adjacent" cases above;
  compaction/repair degradation in 05 and [10.distributed-cache](10.distributed-cache/README.md);
  eviction-efficiency decay in [PROBABILISTIC-STRUCTURES.md](PROBABILISTIC-STRUCTURES.md).
- `watermark`: content watermarking (18, 17), purge-version watermark in
  [09.cdn](09.cdn/README.md), replication high-watermarks in 13/consensus docs. Queue depth as a
  load signal survives only in the 8 cases counted above.
- `throttle` as battery/browser co-tenancy: [22.frontend-uber](22.frontend-uber/README.md)
  (battery-aware updates), 23 (background-tab timer clamping — the *browser* throttling us),
  19's typing-indicator debounce. Client UX, not service overload.
- Notification `throttling` in 42 and [46.google-docs](46.google-docs/README.md):
  per-recipient annoyance/spam discipline.
- `rate limit` as abuse enforcement outside 47: 02's per-account limits, 39's token-forgery
  trip wires, 35.tiktok's "invisible throttling" (shadow-landing of enforcement).
- `admit` as moderation or prose: 43.zoom waiting room, 15's "HNSW admitted at the edge cases",
  44's "the concurrency these numbers still admit".
- `circuit` as electrical: **zero hits** — the 27 check came back clean; every 27 "circuit"
  mention is request-flow backpressure or the token-pull breaker.
- `bulkhead`: **zero hits catalog-wide**. The concept exists but is named cell (31), separate
  pools (17), per-partner adapter (27), stateless shard (05, 28). Grep for those instead. The
  only real `429`/`503` strays were 44 (counted, fail-closed) and 41 (counted, client-side
  contract); the rest were math coincidences.

## The portable pattern

1. **Declare the drop order at design time; never pick it under load.** 39's L1–L4 ladder and
   31's member ladder are pre-enacted config; 26's "shed ETA decoration first, never the route";
   28's "a position sample, never a lifecycle event"; 50's "throttle depth, never trades." The
   order always maps onto (derived / re-derivable / lossy) → (transactional record).
2. **Queue depth is the load instrument** — read at the boundary of the plane being protected:
   05's fanout backlog, 04 and 12's queues, 16's product surface, 17's interactive RTF coupling,
   49's upload-queue fill, 38's blow-past-SLO trigger, 06's alert. Error rates drive nobody's
   trip logic in this catalog.
3. **Visibility is part of the mechanism.** 08's TTFT-not-errors, 16's "show position/ETA, don't
   hide it", 39's honest position + "do not refresh", 04's "backpressure must be visible and
   *fair*, not silent".
4. **Bound the queue or don't have one.** Where the protected plane is derivable, drop instead
   of queueing (05, 28, 40, 47, 09's analytics-before-billing); where it is not, the queue gets
   a TTL and an admission rate (03; 37's admit-rate tracking; 39's drain rate = inventory drain
   rate). 47 refuses queueing outright; 50 rejects *silent* queueing for fairness.
5. **Breakers guard derived planes, with cached or local fallbacks** — 27's cached tokens,
   38's fail-open local bucket, 10's admit-window + soft TTL, 41's last-known-good robots. The
   catalog's breaker is "stop calling the thing that is down, use the warm approximation" —
   never "measure caller-side error rate and stop the feature."
6. **Recovery storms are throttled like entry storms, on the healthy side:** 12 throttles
   reassignment so replay does not stampede the store; 07 write-pauses ingest ahead of rebuild;
   49 is pull, so the sick fleet's retry storm has nothing to hit.
7. **Boundaries.** [47.rate-limiter](47.rate-limiter/README.md) owns quota/abuse admission;
   [KAFKA.md](KAFKA.md) owns the drop-on-backpressure telemetry contract (40, 47); [CDN.md](CDN.md)
   owns edge-tier overflow (09); [REALTIME-TRANSPORT.md](REALTIME-TRANSPORT.md) owns
   congestion-driven media dropping (43); [DEPLOYMENT-SAFETY.md](DEPLOYMENT-SAFETY.md) owns
   blast radius and chaos framing — its ladders are *scheduled* releases of risk, this doc's
   are *runtime* defenses that fire without a human.

Where the pattern is the wrong tool:

- **[37.ticketmaster](37.ticketmaster/README.md)** states it outright: under 2M arrivals,
  "admission control is better than degradation here" — degrading the map tier melts it
  under everyone at once; the correct tool is metering entry.
- **[50.stock-exchange](50.stock-exchange/README.md)**: fairness cannot hide behind a queue or
  a silent shed — the overload response must be a journaled, disclosed event. "Fail conservative"
  there means *halt*, not *degrade*.
- **[44.payment-system](44.payment-system/README.md)**: the money path has no droppable class,
  so the defense is idempotency plus fail-closed 503, not a ladder.
- **[18.sora](18.sora/README.md)**: when one plane's congestion would corrupt another plane's
  promise ("serving must NEVER degrade because generation is congested"), the answer is plane
  separation at admission time, not a shared runtime ladder.
- **[08.chatgpt](08.chatgpt/README.md)**: a perfectly consistent admission gate is itself the
  wrong tool at ~17K admissions/s — "a global coordination step in front of every prompt — the
  cure kills the patient." Flow control can be the bottleneck it exists to prevent.

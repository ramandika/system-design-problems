# 04 — Deep dives

## Dive 1 — The For-You feed as per-request compute (no stored timeline)

**The problem.** Twitter fans out on write into per-user timelines because the follow
graph is the ground truth of who sees what — and reads only modestly outnumber writes
(~1B reads/day vs ~600M tweets/day, Case 02's own model), so
precomputing pays ([Case 02](../02.twitter/04-deep-dives.md)). Here there is no ground
truth: the audience for any video is *whoever the model decides*, and the decision
depends on state that changes by the second (the user's last ten swipes included).

**Decision.** *Option A: precompute per-user feeds on a batch cycle (minutes), cache,
serve from cache.* Option A wins rock-bottom read latency and predictable server cost,
loses relevance-to-the-moment (a burst of engagement in the last 60 s is invisible) and
catalog freshness (new videos appear only after the next batch), and makes experimentation
cruel — every model change requires a full recomputation day. *Option B: compute the
funnel per request.* Option B wins freshness at both ends (signals ≤ 30 s old, videos
published 30 s ago), cheap experimentation, and exploration that can react to early
engagement velocity; it loses a large standing compute fleet and turns every request into
a latency-critical distributed computation. **We pick B** — the product's core promise
("keep showing me what I want right now") *is* the freshness, and the cost is bounded by
making the funnel a strict budget (02's arithmetic: ~2B coarse scores/s mean across the
fleet, estimates). A hybrid exists and we use it: the client's session cache of next-up
candidates plus a "second page" of fine-ranked results lets one funnel computation cover
several swipes, amortizing the cost without freezing anything server-side.

Funnel ordering matters more than funnel depth. Strict rule: **degrade by slicing off
late stages, never by skipping the recall/filters stage** — a fallback that serves
random or recency-only content destroys the loop (users feel irrelevant content
immediately; see failure modes).

**Failure modes.**

- *Stale recall index*: a published video isn't in the ANN index yet → invisible. Mitigation: the publish hook upserts synchronously with a small hard-coded exploration
  injection (dive 5), and new-video TTL'd entries bypass the main index's staleness.
- *Latency budget blowout*: fine rank times out → fall back to coarse-only ordering,
  counted and alerted; the fallback is a *worse but sane* ordering, not a coin flip.
- *Hot-user feature stampede*: a viral event spikes feature-store reads for one
  population shard. Mitigation: user features are computed once per request (not per
  candidate) and cached per session window.
- *Funnel feedback poisoning*: a bad model deploy skews early engagement, which
  retrains the next model on its own garbage. Mitigation: champion/challenger with
  traffic slices and offline replay before full rollout (the same discipline as
  [Case 15](../15.ml-ranking-system/04-deep-dives.md)).

## Dive 2 — Transcode ladder optimized for mobile egress

**The problem.** 600 PB/day of egress (02) is the single biggest cost line, and every
byte is a product decision: the ladder defines both quality and the data bill. Short
videos are unlike long ones: the median watch is 25% of the video, so **first-frame
latency and first-segment size dominate perceived quality**, not steady-state bitrate.

**Decision.** *Option A: a fixed one-size ladder (say 360p/720p/1080p, one codec).*
Option A wins a trivial pipeline (encode everything once, cache forever) and easy CDN
segment reuse; it loses 20–40% of achievable bitrate efficiency (rough figure in line
with publicly documented per-title results from streaming operators), wastes bytes on
simple phone-shot footage and starves complex footage. *Option B: per-title ladders
(ladder computed from source complexity) plus a second codec tier (HEVC-class, AV1
where device coverage allows).* Option B wins the egress economy (N4) — at TikTok's
scale, tens of percent on 600 PB/day — and better quality for complex sources; it loses
transcode CPU (~26K cores sustained, 02, estimate), origin-cache complexity (the
ladder is per-video so edge caching keys on more variants), and adds device-compat
constraints. **We pick B**, with the per-title analysis cheap and conservative
(ladder decisions cached per video at transcode time — no per-request logic).

Design points that make B work for *short* video:

- Keep the ladder **shallow**: ~4–6 rungs between 240p and 1080p, no 4K — the phone
  screen and the swipe loop cap the utility of more rungs.
- **Separate audio track**: silent-swiping is common; audio can also be attached per
  segment cheaply rather than muxed into every rung.
- **First-segment primacy**: transcode and CDN-pin segment 1 of the top 1–2 rungs
  first; the client prefetches the next video's rung-1 segment during the current
  play ([Case 25](../25.mobile-newsfeed/04-deep-dives.md) owns the prefetch policy).
- **Rendition selection is negotiated**: server hints a rung per fetch (from network
  class and device tier), client may downgrade — never upgrade — under pressure.

**Failure modes.**

- *Ladder bug serves high rungs to weak devices* → an egress bill multiplier. Mitigation: rendition selection is server-hinted with hard caps at ingest; alerting on
  per-rung delivered-bytes mixes.
- *Transcode backlog* → publish delay, which starves the exploration burst (dive 5).
  Mitigation: priority queue by predicted audience, degrade gracefully by publishing
  low rungs first (dive 3), and shed to a reduced ladder under saturation rather than
  queueing indefinitely.
- *Codec compat gap* (AV1 coverage) → fallback ordering per device class must be
  explicit, not discovered per-request.
- *A/V desync* from independently encoded tracks — caught by pipeline-level probes,
  because a desync complaint arrives as "the app is buggy", not as a media bug report.

## Dive 3 — Creator upload pipeline (publish before transcode completes)

**The problem.** A post should reach its first viewers in seconds-to-a-minute (N3) —
the first hour of engagement velocity decides its fate (the algorithmic reason publish
latency is a product metric). But a full per-title ladder takes minutes of compute, and
moderation full review takes longer.

**Decision.** *Option A: publish only after complete ladder + moderation pass.* Option
A wins a single consistent quality level and no takedown-race awkwardness; it loses
minutes-to-hours before the first impression (on a 30M-upload/day estate (02), any
backlog multiplies that), and couples the product's most latency-sensitive event to the
slowest pipeline stage. *Option B: staged publish — moderation gate on a cheap automated
check, publish video metadata + lowest acceptable rung immediately, enrich renditions
as they land; the full ladder and deep moderation complete async, with takedown recall
if flagged.* Option B wins time-to-first-viewer and decouples publish from transcode
congestion; it loses a window where early viewers see a lower-quality rendition, and it
creates real races: metadata/counters may change while a video is live, and takedown
must beat the exploration burst. **We pick B** — the exploration burst is worthless if
it fires against a video that hasn't published, and "first 300 impressions were low-res"
is a trivially small loss against "never reached the burst".

Pipeline requirements that follow from B:

- **Chunked, resumable, idempotent upload sessions** — mobile networks drop mid-transfer;
  retries must not duplicate objects or double-publish.
- **Priority transcode queue** ordered by predicted audience (creator tier, source
  signals) so exploration-bound videos always have their burst rung ready.
- **Takedown recall as a first-class operation**: flag → strip from indexes/meta →
  CDN eviction → client-side eviction on next fetch; measured end-to-end to minutes (N7).

**Failure modes.**

- *Moderation race*: video live, flagged after the burst spent budget on it. Accepted
  with bounded exposure; the burst budget (dive 5) doubles as an exposure cap.
- *Encode farm saturation* (peak hour pile-up): shed to reduced ladders, never queue
  silently — a silent queue turns into "my video isn't published" tickets hours later.
- *Duplicate uploads from retries* → idempotency keys per session, dedupe at ingest.
- *Partial ladder confusion* (a rung exists, another 404s): rendition manifest is
  versioned and atomically swapped; clients only see complete manifests.

## Dive 4 — Engagement signal logging at extreme write scale

**The problem.** ~2.1T events/day, ~24M events/s mean, ~100M/s peak (02, estimates),
with two conflicting requirements: N2 freshness (≤ 5–30 s into the online feature
store) and N5-at-scale cheapness. Every downstream consumer needs *approximately the
same stream* with different guarantees: the feature updater wants fresh, the counters
want correct-enough, training wants exact and replayable.

**Decision 1 — delivery semantics.** *Option A: end-to-end exactly-once.*
Option A wins peace of mind for the training lake; it loses throughput and adds
coordination cost at exactly the tier where ~100M events/s peak leaves no headroom, and
exactly-once at the *client* boundary is unsolvable anyway (phones retry POSTs).
*Option B: at-least-once everywhere, with idempotency keys (event UUID produced
client-side) and dedup at consumers.* Option B wins order-of-magnitude cheaper ingestion
and graceful client-retry behavior; it loses exactness at the counters (acceptable —
like-counts are approximate by design) and requires dedup discipline in the feature
updater (a double-counted watch signal *does* skew a model). **We pick B**, with one
hard rule: the *feature-store* consumer dedups by event UUID, and the training lake
stores raw events so replay can rebuild anything downstream.

**Decision 2 — freshness mechanism.** *Option A: micro-batch every few minutes*
(closer to a warehouse ETL). Option A wins batch efficiency; loses the N2 freshness the
feed loop actually responds to. *Option B: streaming consumer writes feature updates
within seconds.* Option B wins the closed loop (N2, and the reason "watched 5 cat
videos recently" changes the next fetch); it loses at-least-twice machinery at the
feature-store tier and a write-amplification budget that can fight the ranker's reads.
**We pick B** with the freshness budget enforced as a measured SLO (event → feature
store), because a stale loop is invisible in dashboards — users just leave.

Topology: client batches ~50 events/POST → edge collectors acknowledge durably and
fast → Kafka-class log partitioned by user id ([Case 13](../13.kafka/README.md)) →
consumer families: (a) online feature updater → feature store, (b) streaming
aggregator → sharded approximate counters, (c) raw-event lake for training/replay.

**Failure modes.**

- *Hot partition*: a viral creator or event concentrates writes (their video's events
  plus fan-in from shared users). Mitigation: partition by user id (spreads natural
  load) plus salting for known-hot entities; counters sharded per item with
  approximate aggregation.
- *Feature-store write amplification* degrading ranker read latency: update batching
  with an explicit staleness-vs-latency knob per feature class.
- *Schema drift at 24M events/s*: consumers reject on unknown fields instead of
  crashing; the lake is schema-on-read so evolution never blocks ingest.
- *Double counting*: inflates exploration decisions (dive 5) — hence dedup at the one
  consumer that feeds a control loop, not just at the analytics tier.

## Dive 5 — Cold-start for new creators and new videos

**The problem.** A new video has zero engagement; a new creator has zero history and
their *prior* videos' data is thin or absent. Engagement-maximizing ranking is
risk-averse by construction — unshown videos can't accumulate the evidence that would
justify showing them. Left alone, the catalog ossifies: the same top creators, and the
newcomer pipeline that keeps the product alive dies.

**Decision.** *Option A: fixed time-boxed exploration burst — every new video is
guaranteed ~300 impressions in its first day (02: ≈ 3% of impressions, ~9B/day),
then competes on evidence alone.* Option A wins a simple, *explainable-to-creators*
promise ("every video gets seen"); it loses efficiency (many bursts are spent on
content the model already believes is weak) and it freezes decisions at 24 h —
late bloomers die. *Option B: purely adaptive exploration (bandit-style per video,
[Case 15](../15.ml-ranking-system/04-deep-dives.md) has the machinery).* Option B wins
impression efficiency; loses the floor — pure bandits starve videos whose prior is
bad, which is exactly the cold-start failure we are trying to prevent, and starves
*new creators* hardest since their priors come from empty history. **We pick A as the
floor, B as the extension**: every video gets the guaranteed burst; after it, adaptive
allocation with a *non-zero floor* for videos whose early signals beat their
content-feature prior, so a breakout video is detected in hours, not days.

What the early decision uses, in order of availability:

1. **Content priors, not engagement**: a video embedding (computed in the transcode
   pipeline — reuse dive 2's analysis) against user interest clusters; author-level
   history features; audio/text metadata. This is why a brand-new creator's first
   video still finds *some* audience immediately, and is the structural difference
   from follow-graph systems ([Case 02](../02.twitter/README.md)).
2. **Early velocity within the burst**: completion rate, rewatch, share at ~300
   impressions — small-sample but real; treated with explicit priors (a bandit over
   low counts, not an average of 30 data points).
3. **Creator-level evidence** once the account has history.

Contrast with [Case 18](../18.sora/README.md): there the expensive problem is
*creating* the video; here compute creates nothing — the budget spent on cold-start is
*impressions*, and spending it deliberately is the whole mechanism by which the
platform stays open to newcomers.

**Failure modes.**

- *Feedback death spiral*: poor early noise → low score → never re-shown → no
  evidence ever arrives. Mitigation: the guaranteed floor plus per-video min-exposure
  budgets over the first days, not just hour one.
- *Gaming the burst*: uploads engineered to pass the gate and farm initial impressions
  (engagement pods, loop-inducing gimmicks). Mitigation: velocity features weighted
  against per-account behavior features; burst budgets per account, not only per video.
- *Opaque suppression churn*: creators read invisible throttling as "shadowbanning" —
  a *product* failure of the exploration policy. Mitigation: publish transparent
  creator-facing signals (views sources) even if policy internals stay opaque.
- *Exploration cannibalizing exploitation*: 3% of impressions sounds cheap but at
  peak it displaces real engagement if applied uniformly. Mitigation: exploration
  injection is quota-managed in the slate layer (dive 1, step 5) and skews toward
  low-cost moments in the session, not uniformly.
- *Counter races at publish* (dive 3): a video's early counters are approximate and
  lagged; cold-start scoring that naively trusts them double-punishes new videos.
  Mitigation: velocity vs. baseline priors, never raw counts.

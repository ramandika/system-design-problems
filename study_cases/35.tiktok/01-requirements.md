# 01 — Requirements

## Functional

- **Publish**: a creator uploads a short video (typical 10 s–3 min, phone-recorded);
  the platform transcodes, moderates, then makes it available in feeds.
- **For-You feed**: an infinite single-item-at-a-time feed for any signed-in user,
  computed from implicit signals (watch time, rewatch, share, like, follow) — with no
  explicit subscription as a requirement. Users expect zero configuration: the feed
  starts showing relevant content within the first session of a brand-new account.
- **Secondary surfaces**: following feed, profile pages, search, sound/hashtag pages —
  in scope only where they share the same substrate (video meta service, CDN, logging).
- **Engagement feedback**: like, comment, share, save, follow, report, "not interested",
  rewatch — all must reach the recommendation stack.
- **Creator analytics**: aggregate views/likes/followers per video, updated with a delay
  (minutes is acceptable) — but visible *initial* views matter to creators immediately.

## Non-functional (the ones that shape the design)

- **N1 — Feed latency**: feed fetch under ~500 ms server-side p99 at peak to keep the
  swipe loop seamless; the first frame of the next video must render in under ~1 s on a
  mid-tier phone on 4G (estimates; the swipe loop tolerates almost nothing).
- **N2 — Freshness of signals**: an engagement event must be reflected in the online
  feature store within ~5–30 s (design target, an estimate of the point below which
  freshness stops mattering) — the feed is a closed loop and a stale loop shows repeat
  content.
- **N3 — Publish latency**: a new video must be *serveable* within ~30–60 s of upload
  completing (design target) and in exploration circulation within minutes, because
  early engagement velocity drives its distribution.
- **N4 — Egress economy**: mean mobile data consumed per DAU on the order of ~1 GB/day
  (estimates: public measurements of similar apps put usage in the ~0.7–1 GB/hour class;
  02's arithmetic lands at ~1.2 GB/DAU/day under its assumptions) → the ladder must be
  per-title and rendition selection client+server negotiated.
- **N5 — Write scale for telemetry**: ingest ~2.4×10⁷ events/s mean, ~10⁸/s peak
  (estimates, see 02) with at-least-once semantics and duplicate tolerance.
- **N6 — Availability, split by surface**: the For-You feed degrades *degraded*, never
  down — a fallback ordering (coarse-rank only, then recency-trending) is required;
  upload can be unavailable for minutes without user-visible failure.
- **N7 — Trust and safety**: automated moderation gates *serving* (nothing is
  distributed before a pre-publish check), while full human review is asynchronous;
  takedown must propagate to all surfaces within minutes.

## Scope decisions

- **We do not design the client feed mechanics** (prefetch, decode, scroll) — that is
  [Case 25](../25.mobile-newsfeed/README.md). We design the server pipeline and the
  artifact (ladder, metadata) the client consumes.
- **We do not design the ranker internals** — the multi-stage funnel ends at "scored
  candidates"; [Case 15](../15.ml-ranking-system/README.md) owns scoring, feature
  stores, and offline/online skew. We own what feeds it and what reorders its output
  (quota, diversity, exploration).
- **Storage and CDN are treated as given substrates**
  ([Case 07](../07.blob-store/README.md), [Case 09](../09.cdn/README.md)); the deep
  dives assume their properties (eventual consistency, edge hit ratios ≥ 90%, cold
  tiers).
- **Original/creator quality tooling** (in-app editor, effects, sounds) is out of scope;
  the upload pipeline receives a final MP4 and metadata.
- Live-streaming is out of scope: the ladder and feed math here are for VOD-style short
  clips, and live shifts the whole problem to origin fan-out.

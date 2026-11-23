# 02 — Scale estimates

Assumptions (adjust and redo arithmetic if they change). Everything here is an estimate for
arithmetic's sake; where a figure is public we say so.

- ~2B MAU, ~600M DAU (MAU publicly reported ~2B for Instagram; DAU estimated).
- Feed posts: ~200M photos/day + ~30M short videos/day (estimate; Instagram publicly reported
  ~40M photos/day for ~100M users back in 2012, so ~200M/day at ~2B users is a conservative
  extrapolation, not a figure).
- Stories: ~500M accounts post-or-view stories daily (Instagram publicly reported ~500M daily
  stories users in 2018; assume more now). Assume ~1B story items/day — posters are the posting
  subset of that population, call it ~300–500M posters averaging 2–3 items ≈ ~1B (estimate).
- Media GETs dominate: average DAU session loads a feed ~8×/day, each screen ~8 media items,
  and the tray/detail/explore pages add ~2× more. Estimate below.

## Upload path

**Media uploads**: 230M posts + 1B stories ≈ **1.23B uploads/day ≈ 14K QPS average**, peak
(weekday evenings, weekends) ~4× → **~60K QPS**. Each upload carries an original that averages
~1.5MB for photos (modern phone cameras produce 3–8MB; the client compresses before sending —
this matters for the pipeline in 04 #1) and ~10MB for short videos.

- Upload ingress bandwidth: 200M photos × ~1.5MB + 30M videos × ~10MB + 1B stories × ~1MB
  (stories are short and more heavily client-compressed — estimate) ≈ 0.3PB + 0.3PB + 1PB ≈
  **1.6PB/day ≈ ~18GB/s average**, ~75GB/s at peak. Peak must survive; uplink bandwidth is a
  real capacity line-item, unlike Twitter's text.

## Renditions (write-time precomputation)

Every photo becomes ~4–6 renditions (thumbnail, small, medium, full; WebP/HEIC variants), every
video an ABR ladder of ~4–5 encodings plus a thumbnail. Rendition bytes total ~1.5–2× the original
for photos, ~1× for video (video originals are heavily compressed already).

- Photo renditions: 200M × ~2.5MB total ≈ 500TB/day.
- Video renditions: 30M × ~10MB ≈ 300TB/day.
- Story renditions: 1B × ~1.5MB (≈1.5× the ~1MB story original) ≈ 1.5PB/day — but see the TTL band below; most of this is
  transient, not accrued.
- **Accruing** storage (permanent posts only): ~800TB/day ≈ ~290PB/year. Story storage held live
  at any moment ≈ 1.5PB/day × 1 day ≈ **~1.5PB of churn band** that turns over completely each day.
- Transcoding compute: ~6B renditions/day (~5 renditions per upload: 4–6 photo renditions,
  4–5 ABR encodings + thumbnail) ≈ ~70K renditions/sec average. Video dominates CPU
  (~seconds to tens of seconds of CPU per ladder), photos are cheap (~100ms). Budget O(50K–100K)
  vCPU-equivalents of a transcode farm — this is why it must be async (04 #1).

## Feed reads

- Feed opens: 600M DAU × ~8 opens/day ≈ 4.8B/day ≈ **56K QPS average**, peak ~3× → ~170K QPS.
- Media GETs: 4.8B feed opens × ~8 items × ~3 (tray, explore, detail, retries) ≈ 100B+/day ≈
  **~1.2M QPS average**. The overwhelming share must be answered by CDN edge caches
  (../09.cdn); an origin hit rate of even 5% is ~60K QPS against blob storage — that number is
  the reason renditions and URLs are designed for cacheability (immutable content-addressed URLs).

## Fanout (hybrid, ranked)

- Average account follows ~200–400 accounts (skewed like Twitter: median tiny, top ~200M+).
- If fanout-on-write materialized every post into every follower's feed pool: 230M posts × ~300
  followers ≈ **70B inserts/day**, plus 1B story items × ~150 viewers ≈ another 150B/day. Stories
  *do* use a viewer-list model (see 04 #2), so the story fanout is real but reads differently.
- Feed inserts at ~1ms each → a celebrity with 100M followers would take ~28h serialized; same
  wall as ../02.twitter/02-scale-estimates.md, but with an extra twist: because the feed is ranked,
  the cache stores candidate *pools* (~1–2K items per user, deduplicated), which caps pool storage
  at ~100B of ID + score + packed features per item ≈ ~0.1–0.2MB per pool ≈ ~100–200TB across
  ~1B users (estimate) — small because the *media bytes* are never in the timeline, only IDs +
  URLs; for calibration, ~200TB is under two weeks of one day's story churn (~1.5PB).
- Engagement writes: likes/comments/saves ~10–20 per post across its life → ~4B events/day ≈ 45K
  QPS — embarrassingly parallel, eventually consistent.

## Stories reads

- Story views: 500M viewers/day × ~15 stories watched ≈ 7.5B view-opens/day ≈ ~87K QPS average.
- Seen-state writes: ~7.5B/day of per-viewer marks (04 #2's whole problem), plus view-list reads
  for authors (500M authors × a few checks/day).

## Sanity checks

- Media GET:upload ratio ≈ 100B / 1.23B ≈ **~80:1** (Twitter's read:tweet was ~2:1) — the read
  amplification of a *file*-based social graph is dramatically higher, hence CDN-first design.
- Fanout insert:feed read ≈ 70B : 4.8B ≈ 15:1 (Twitter's was ~180:1 because posts are rarer than
  tweets relative to readers; either way fanout-on-write for everyone is dead on arrival).
- The single largest daily cost is CDN egress + origin bandwidth (~PB/day), not storage (PB/year
  accrues slowly); the second largest is transcode CPU.

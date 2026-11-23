# 04 — Deep dives

## 1. Transcoding as a parallel DAG

The naive design encodes each rendition linearly — a 2 h video takes hours to publish.
Instead: **split the video into N-second segments** and transcode segments independently,
then mux. 3600 s video at 6 s chunks = 600 parallel work units per rendition, ×5 renditions.

- **Trade-off**: segmented encoding slightly hurts compression quality vs encoding the whole
  file in one pass (no lookahead across segment boundaries; you can mitigate by passing
  hints/keyframe decisions and letting workers communicate via a few-second look-behind).
  We accept a small quality loss for near-linear publish speed — publishing latency is the
  user-visible win.
- **Chunk boundaries must land on keyframes** — so the ingestion stage runs a fast analysis
  pass (scene detection + keyframe map) that becomes the coordination artifact for the DAG.
- **Failure model**: each (video, rendition, segment) is a retryable idempotent unit; the
  catalog flips to "playable" only when every segment of every rendition in the base ladder
  is done. Stragglers (1 slow segment) need speculative re-execution, not queue-block the video.
- **Validated real-pipeline pattern** (vbench, ASPLOS '18, 5 Google co-authors): every
  upload is first transcoded to a *universal intermediate representation*, renditions are
  produced from that, and videos observed to be popular are transcoded **a second time at
  higher encoder effort** to shrink storage/network costs — popularity follows a power
  law, so spending extra software-encoder compute on the long tail never amortizes. This
  gives the DAG a second, popularity-triggered phase that watches view telemetry.

## 2. Adaptive bitrate & the encoding ladder

Problem: one file must play on a 2 Mbps cellular link and a 100 Mbps TV, with quality
changing mid-playback as bandwidth fluctuates.

Design: **rendition ladder** (e.g., 1080p@6Mbps, 720p@3Mbps, 480p@1.5Mbps, 240p@0.4Mbps),
each chopped into 2–10 s segments with its own index/playlist. The player measures
throughput, renders from the cheapest segment quality that fits current bandwidth, and
switches up/down segment by segment.

- **Trade-off**: more renditions = better "just right" quality = more storage + transcode
  CPU + CDN origin diversity (harder cache hits). Ladders converge to ~5–8 rungs.
- **Why segments and not byte ranges**: seeking needs to start a new stream from clean
  keyframes; byte ranges of a monolithic file force reading from the front to re-sync.
  Segmenting is what makes instant seek + mid-stream quality switch cheap. Costs:
  request overhead (each segment = round trip) — mitigated by short CDN-to-client latency
  and client-side prefetch.

## 3. CDN + cache hierarchy (thundering herd on the same bytes)

One viral video = millions of users requesting identical segments within hours. Design:
**client → regional CDN edge POP → (miss) → mid-tier "shield" cache per region → origin**.
The shield collapses many edge misses into one origin request.

- **Trade-off**: more cache layers → higher hit ratio but higher storage cost at every layer
  and slightly more indirection on misses. With extreme skew (top ~1% of videos = most watch
  hours), a tiny hot set served from everywhere absorbs most traffic — this is exactly the
  shape CDNs are built for.
- **Negative caching / negative lookups** matter too: "this segment doesn't exist" (deleted
 /private) results should be cached so abuse doesn't hammer origins.
- **Cold-start for new videos**: a video published 5 minutes ago may explode; pre-warm CDN
  on first-hit instead of full pre-warm of everything (wasteful, most videos stay small).
- **Real-world shape of this hierarchy (validated, Nov 2026 check)**: YouTube's edge
  clusters are racks of homogeneously configured cache servers, each holding video chunks
  in a DRAM → SSD → HDD tier stack with a remote origin as last resort, in 200+ countries
  (NSDI '23 "HALP" paper, written with YouTube CDN engineers). The outermost tier —
  Google Global Cache nodes — sits *inside ISP networks* and caches only locally-popular
  content temporarily; long-tail videos are served from higher tiers rather than replicated
  to every edge. Note how this inverts the usual "cache everything closest to the user"
  instinct: the tier closest to users stores the *least*, because it has the least room.

## 4. Storage tiering

From estimates: ~20 PB/day stored (raw ingest is ~6.5 PB/day), tail-heavy popularity. Design: **popularity-based
lifecycle** — new/hot renditions on fast storage, demote to dense/cheap cold storage as a
video ages without views, re-encode from kept originals if demand resurfaces.

- **Trade-off**: re-materializing from cold adds latency for the rare "old video suddenly
  pops" case (acceptable; minutes), vs paying SSD/HDD rates for data watched 4×/year
  (unaffordable). Key mechanism: cheap access-frequency signals (watch counts, CDN cache
  presence) drive tier transitions; delete only when retention policy says so, not when the
  code loses interest.

## 5. View counts & metadata at billions-scale

Viewing telemetry is firehose-scale (every second of playback emits events) but the query
shape is narrow: counts-per-video, updated frequently with ~10 min staleness tolerance.

Design: no 1-row-per-view table. **Event pipeline streams playback events → local
pre-aggregation → periodic grouped updates into a counter store** (e.g., Redis for the hot
recent window + DB for authoritative totals, periodic reconciliation).

- **Trade-off**: dedup (bots/refresh) is approximate at the edge and precise in an offline
  audit — eventual consistency buys us the stream-able design.
- Generalizes: view counts, likes, "watch later", comment counts — all eventually-consistent
  aggregates; only identity (video exists, is playable, permissions) is strongly consistent.
- **Real stores of record (validated, Nov 2026 check)**: YouTube's dimensional metadata
  warehouse (video/channel/playlist entities behind analytics and reporting) is built on
  Bigtable — 1:1 mapping of entity type to table, one protobuf-encoded row per entity
  (Google Cloud blog, Aug 2023, by the system's own engineers) — and its core transactional
  metadata lineage is **Vitess**, YouTube's sharded-MySQL layer (open-sourced). Two
  distinct planes: an operational store serves the request-time watch page, the Bigtable
  warehouse serves reporting/analytics; don't design one store to do both.
- **Recommendation serving (era-dated)**: the canonical published architecture is two-stage
  (deep candidate generation → deep ranking, Covington et al., RecSys 2016) with
  request-time scoring in hundreds of ms. Treat it as the starting architecture — modern
  pipelines add retrieval evolutions (e.g., two-tower models) and re-ranking/diversity
  stages on top; the two-stage shape itself is the part worth designing.

## 6. Cheap bitrate = the hidden lever

At ~1.4 EB/day, cutting average delivery bitrate 10% saves more than any other single
engineering initiative combined. Real system levers: better encoders (H.264 → VP9 → AV1
saves ~20–40% at equal quality, at 3–10× transcode cost — only applied to popular videos),
per-video per-scene bitrate optimization (content-adaptive encoding), and not over-encoding
the tail videos that nobody watches in 1080p anyway.

- **Trade-off**: AV1 on everything would save bandwidth but blow the transcode budget
  (~1.1M machine-hours/day from estimates) — so encode popular videos in the modern codec
  first, keep old codecs for the tail and old devices.
- **Hardware reframes this math** (see deep dive 7): with fixed-function transcoders the
  "3–10× transcode cost" tax of modern codecs collapses — the extra compute becomes
  silicon area, not billable CPU hours — which is exactly why YouTube builds ASICs, and
  also why Argos v1 shipped *without* hardware AV1 encode: the ladder's codec set is
  frozen at ASIC design time, so AV1 renditions stayed on CPU farms.

## 7. Transcoding compute economics: hardware beats hand-tuning (Argos VCU)

The estimates say ~1.1M machine-hours/day of CPU transcoding; VP9/AV1-class software
codecs need ~3–4× the H.264 compute per rendition for the bitrate savings that make the
read path affordable. CPUs can't absorb both factors across a ladder of renditions
(~5–8 rungs per codec family in our estimates; a dozen-plus once 4K, frame rates and codec
variants are added) — the answer isn't more Xeons, it's changing the cost constant itself.
YouTube's answer: **Argos VCU**, a custom transcoder ASIC in production since ~2018
(Google's Hot Chips 2021 talk).

- One VCU matches a two-socket Skylake server for H.264 and is ~5× a server for VP9;
  a 20-VCU server replaces **multiple racks** of CPU transcoding machines. Google reported
  "up to 20–33× compute-efficiency" vs its previous optimized software stack — a vendor
  figure, best-case from the VP9 columns, so treat as an upper bound; the direction
  (order-of-magnitude) is the load-bearing part.
- **Design consequence**: keep the transcode DAG codec-agnostic, and size the transcoding
  fleet in two currencies — software capacity (correctness, exotic codecs, AV1, anything
  the silicon can't do) + fixed-function capacity for the standard ladder rungs.
- The same two-pass economics from deep dive 1 applies on hardware: popular videos get the
  second, higher-effort pass precisely *because* the extra effort amortizes across
  playbacks — power-law popularity is what makes ASIC spend rational.

# Object storage across the case catalog

Where S3-class object/blob storage appears in the 50 study cases, what role it plays, and how
deep each case goes. Companion to [CONCEPTS.md](CONCEPTS.md) (storage union "Blob / object store
(incl. content-addressed)", Tier 2, 20/50 cases) and to the grep-derived count below, which also
counts brief archival mentions and boundary notes.

Headline: **object storage is the only storage class here that almost no case designs and almost
every case needs.** Only one case studies the store itself ([07.blob-store](07.blob-store/README.md));
everywhere else it enters as an assumed floor — a durable, immutable origin that bytes are written
to once and read many times. The three roles recur verbatim: the origin behind a CDN for media and
binaries, the durable lake that ML/analytics replay from, and the raw-event/corpus dump that makes
reprocessing cheaper than re-fetching.

- Cases mentioning object/blob storage (incl. archival one-liners and boundary notes): **31 / 50**
- Designed architectural component (sized or diagrammed): **21**
- Brief mention / "archive it to blob storage" shorthand only: **4** (27, 38, 46, 50)
- By-reference or implied origin (bytes must live somewhere; another case named): **3** (02, 09, 24)
- Boundary or explicitly rejected as a component for a hot path: **2** (11 = "no blob storage here",
  13 = rejected for tiered log reads), plus 08 rejecting one of its two object-storage uses

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [01.youtube](01.youtube/README.md) | Ingest store for originals + hot/cold rendition tiers behind CDN | Core component |
| [02.twitter](02.twitter/README.md) | Implied media origin; ~5% cache-miss fetches, pattern deferred to YouTube | By-reference |
| [03.whatsapp](03.whatsapp/README.md) | Content-hashed, E2E-encrypted media blobs served via CDN; bytes bypass chat path | Component |
| [04.deployment-system](04.deployment-system/README.md) | Artifact store: SHA-256 chunked, content-addressed, dedup-first | Component |
| [05.discord](05.discord/README.md) | Attachment object storage (~150 TB/day), decoupled from the row store | Component |
| [06.apple-app-store](06.apple-app-store/README.md) | Approved Binary Blob Store: the only source of client-facing bytes | Core component |
| [07.blob-store](07.blob-store/README.md) | The case itself: PUT/GET/DELETE/LIST, erasure coding, 11-nines durability | Full case |
| [08.chatgpt](08.chatgpt/README.md) | Conversation store ("KV/object store: durable text, cheap") + one rejected use | Component, one rejection |
| [11.zookeeper](11.zookeeper/README.md) | Boundary: "No blob storage, no media" — the coordinated systems' job | Line only |
| [13.kafka](13.kafka/README.md) | Rejected for tiered log reads: retention-cost win, replay-latency loss | Rejected |
| [14.ml-production-system](14.ml-production-system/README.md) | Durable lake: ~15 PB compressed / 30-day window in object storage + parquet | Core component |
| [16.dalle](16.dalle/README.md) | Result blob store (immutable, TTL'd) + proactive CDN push of derived sizes | Core component |
| [17.elevenlabs](17.elevenlabs/README.md) | Object store for batch WAV/MP3 deliverables; 30-day hot then archive tier | Component |
| [18.sora](18.sora/README.md) | Checkpoint & artifact store for intermediates (TTL'd) vs published clips | Core component |
| [24.mobile-chat-app](24.mobile-chat-app/README.md) | Blob store treated as given (case 07); client tolerates unreachable/stale | By-reference |
| [27.ev-charging-network](27.ev-charging-network/README.md) | Firmware artifacts (~100 GB) from object storage/CDN with HTTP range/resume | Brief |
| [30.facebook-messenger](30.facebook-messenger/README.md) | Media path: content-hashed blob store → CDN → receivers | Component |
| [31.netflix](31.netflix/README.md) | Origin storage: ~6 PB durability-critical masters kept for re-encode | Component |
| [32.spotify](32.spotify/README.md) | Origin blob store sees cache misses and offline backfills only | Core component |
| [33.instagram](33.instagram/README.md) | Originals + content-addressed renditions; 5% origin miss rate budgeted | Core component |
| [34.reddit](34.reddit/README.md) | Object store + CDN for media; posts store only URLs | Component |
| [35.tiktok](35.tiktok/README.md) | ~540 TB/day of originals + ABR ladder; ~70% expire from hot tiers | Core component |
| [36.airbnb](36.airbnb/README.md) | ~3 PB photos, content-addressed; enables perceptual-hash fraud detection | Component |
| [38.leetcode](38.leetcode/README.md) | Submission records archived to blob storage | Brief |
| [39.e-commerce](39.e-commerce/README.md) | ~600 TB image corpus, immutable, CDN-served | Component |
| [41.web-crawler](41.web-crawler/README.md) | Raw-HTML corpus (~150 TB/day blobs); blob retained as the recovery point | Core component |
| [43.zoom](43.zoom/README.md) | Recordings ~100 TB/day appended to object storage, retention per policy | Component |
| [46.google-docs](46.google-docs/README.md) | Cold docs' revision history (~1.5 PB) pages to object storage | Brief |
| [49.monitoring-metrics](49.monitoring-metrics/README.md) | Immutable 2 h TSDB blocks uploaded to object storage (Thanos pattern) | Component |
| [50.stock-exchange](50.stock-exchange/README.md) | Audit log ~36 TB/year to archive tier ("audit, not operational storage") | Brief |
| [09.cdn](09.cdn/README.md) | Origin types listed as "S3/blob/app" | By-reference |

## Per-case notes

### The case itself

**[07.blob-store](07.blob-store/README.md)** — the store is the product: HTTP PUT/GET/DELETE/LIST
over buckets and keys, 1 B–5 TB objects ([01-requirements.md](07.blob-store/01-requirements.md)
adopts S3's published limits as targets, labeled as such). Key decisions in
[04-deep-dives.md](07.blob-store/04-deep-dives.md): two-phase PUT (bytes plane then index plane,
plus orphan GC) — accepted cross-plane complexity in exchange for planes that scale and fail
independently; read-consistent GET with bounded list lag, framed against S3's documented
eventual-l listings; durability engineered to the published 11-nines target via redundancy +
scrub cadence + repair budget + DC layout + soft-delete, with the math shown, not asserted.

### Media serving + CDN handoff

The dominant role: the object store is the *origin* a CDN fronts. The recurring contract is
immutable, content-addressed keys serving renditions; the store sees only cache misses, and every
case sizes the miss budget explicitly. Trade-off in every one: object storage is cheap and durable
but slow-ish at scale, so latency and egress are bought from the CDN, not the origin.

- **[01.youtube](01.youtube/README.md)** — the fullest media pipeline: ingest store keeps the
  original for re-encodes (3× replicated or erasure-coded), rendition tiers move hot (SSD/HDD, by
  popularity skew) → cold via lifecycle rules — the trade is paying retention for agility
  (re-encode later) against cheap archive ([03-architecture.md](01.youtube/03-architecture.md)).
- **[31.netflix](31.netflix/README.md)** — the licensed-content flip of UGC economics: masters
  ~6 PB are durability-critical and kept for regeneration; total catalog ~10 PB × 2–3 replication
  ≈ 20–30 PB, yet "storage trivial : egress dominant" — origin bytes barely matter next to serving
  ([02-scale-estimates.md](31.netflix/02-scale-estimates.md)).
- **[32.spotify](32.spotify/README.md)** — origin blob store (07 cross-referenced) is designed to
  see *only* misses and offline-sync backfills; [04-deep-dives.md](32.spotify/04-deep-dives.md)
  weighs POP-ownership vs CDN rental and picks the mix — request volume is enormous but byte
  volume modest, and misses hit a regionally sharded origin serving renditions, not raw masters.
- **[33.instagram](33.instagram/README.md)** — puts a number on the miss budget: ~1.2M media QPS
  means even a 5% origin hit rate is ~60K QPS against blob storage — the reason URLs/renditions
  are designed for cacheability (immutable content-addressed keys, metadata row flipped to
  `SERVING` only after renditions land).
- **[35.tiktok](35.tiktok/README.md)** — the storage-economics case: ~18 MB/video → ~540 TB/day
  written ([02-scale-estimates.md](35.tiktok/02-scale-estimates.md)); the engagement long tail
  makes ~70% of uploads near-zero-view, so hot tier is capped at 7–30 days and unengaged videos
  lapse. Trade: cheaper retention vs losing long-tail replays cheaply retrievable.
- **[16.dalle](16.dalle/README.md)** — read-many, hot-right-after-creation: origin object store
  with TTL/lifecycle tiers plus derived sizes *proactively pushed* to CDN on job completion,
  because creation is the known access event — push beats first-miss pull.
- **[18.sora](18.sora/README.md)** — two object-store classes with opposite lifecycles: published
  clips (immutable, tiny, long-lived → object store + CDN) vs intermediates (latents, large,
  TTL ~7 days, separate prefix, deleted not archived). "The 225 TB/day is the *cost of
  editability*"; checkpoints go to a regional store, published output to the global CDN, so
  machine-to-machine checkpoint I/O never competes with playback.
- **[06.apple-app-store](06.apple-app-store/README.md)** — binaries as immutable versioned objects
  (~4 PB erasure coded), the only place client-facing bytes come from; deleting a version is
  replaced by tombstone-and-hydrate because the bytes are load-bearing for rollback.
- **[17.elevenlabs](17.elevenlabs/README.md)** — batch deliverables (2.4 TB/day) kept 30 days hot
  ≈ 72 TB, then downshifted to an S3-class archive tier; interactive audio is ephemeral (~1 day),
  a deliberate durability asymmetry between product surfaces.
- **[43.zoom](43.zoom/README.md)** — recordings are latency-tolerant by construction (compose +
  encode offline, not in the interactive path) then appended to object storage (~100 TB/day,
  estimate), retention per customer policy.
- **[03.whatsapp](03.whatsapp/README.md) / [30.facebook-messenger](30.facebook-messenger/README.md)
  / [34.reddit](34.reddit/README.md)** — the minimal version of the role: content-hashed blobs
  (E2E-encrypted in 03), media path bypassing the chat/row path entirely, messages storing only
  keys/URLs. 24's mobile client builds on 03's store and treats it as (a) reachable when the OS
  allows, (b) at-least-once, (c) possibly stale.
- **[36.airbnb](36.airbnb/README.md)** and **[39.e-commerce](39.e-commerce/README.md)** — media
  corpus sizing: ~3 PB photos and ~600 TB product images, both immutable and CDN-served; 36 adds
  the leverage point — content-addressed blob storage is what makes a perceptual-hash index over
  all listing photos (the best fake-listing signal) cheap to compute.
- **[05.discord](05.discord/README.md)** — attachment media decoupled from the message row store:
  ~150 TB/day into object storage, CDN-fronted, "trivially horizontal" next to the sharded
  metadata problem.
- **[27.ev-charging-network](27.ev-charging-network/README.md)** — the serving contract on a hostile
  network: firmware blobs (~100 GB total) from object storage/CDN with HTTP range/resume, and
  devices download *before* rebooting — byte storage is trivial here; the fetch discipline is the
  design.
- **By-reference**: [02.twitter](02.twitter/README.md) sizes media egress (~1.5 PB/day) and origin
  fetches at ~5% but defers the origin design to YouTube's pattern;
  [09.cdn](09.cdn/README.md) lists origins as "S3/blob/app" — for the CDN case, object storage is
  just a customer.

### Durable lake for ML & analytics

Second role: object storage is where long-window training and query data lives, because the
window is months and replicated-DB or head-store money cannot cover PBs. The pattern: land raw
once, compact into partitioned columnar files, read from the lake.

- **[14.ml-production-system](14.ml-production-system/README.md)** — the training window is "a
  first-class storage citizen": ~15 PB compressed over 30 days (~180 PB/year) in object storage,
  fed by a large Kafka + parquet-compaction pipeline. The named failure mode is the lake's
  discipline, not its size: retention shorter than the training window kills reproducibility.
- **[49.monitoring-metrics](49.monitoring-metrics/README.md)** — the analytics analogue: compacted
  immutable 2 h TSDB blocks are made immutable *precisely so* they upload to object storage as-is;
  13 months costs ~7–8 TB of "object-storage money instead of replicated-DB money", with a store
  gateway streaming chunks so old blocks stay queryable without living in RAM.
- **[08.chatgpt](08.chatgpt/README.md)** — uses object storage as the conversation record store
  ("durable text, cheap") and *rejects* its second use, persisting KV caches between turns:
  ~300 PB at max context (even mean-context ~50 PB), and loading tens of GB eats the TTFT budget
  — the store wins on cost and loses on transfer time against an interactive budget.

### Raw-event / corpus storage

Third role: keep the raw bytes so *replaying beats re-fetching*. The store is a recovery substrate,
not a serving layer.

- **[41.web-crawler](41.web-crawler/README.md)** — ~150 TB/day of fetched blobs; the store
  deliberately retains raw HTML for the corpus TTL rather than just parsed output, so a lost index
  recovers by re-extraction, not re-fetch — "the blob is the recovery point"
  ([04-deep-dives.md](41.web-crawler/04-deep-dives.md)); write-once append shape called out as
  same-as-07. Trade: ~2× storage over parsed-only, bought for cheap reprocessing.

### Archive-tier one-liners

- **[38.leetcode](38.leetcode/README.md)** — submission records (56 GB/day, trivial) hot in SQL,
  archived to blob storage.
- **[46.google-docs](46.google-docs/README.md)** — ~1.5 PB of revision history; cold docs' op
  history pages to object storage, hot ops stay in the fast store.
- **[50.stock-exchange](50.stock-exchange/README.md)** — audit trail ~36 TB/year to an archive
  tier, explicitly "audit, not operational storage".

### Boundaries and rejected uses

- **[11.zookeeper](11.zookeeper/README.md)** — the negative-space note: znodes are KB-scale, "no
  blob storage, no media — that is the job of the systems ZooKeeper coordinates". Same
  metadata-vs-data hard boundary the Bigtable case draws.
- **[13.kafka](13.kafka/README.md)** — tiered log storage to object storage is scoped out with an
  explicit verdict: reading from object storage abandons the page cache, so it is fine for
  retention *cost* and bad for *replay latency*. The boundary where the blob store loses: hot
  ordered-log reads.

## The portable pattern

Every substantive use reduces to one contract, cross-linked to
[07.blob-store](07.blob-store/README.md) as the reference implementation:

1. **Objects are immutable keys, mostly content-addressed.** Renditions, checkpoints, artifacts
   and crawled pages are written under a hash and never edited in place (01, 16, 18, 32, 33, 36,
   41). Mutability is faked with new keys + a metadata flip or a version pointer (33's `SERVING`
   row, 06's versioning) — which is also why deletion needs discipline (06 replaces delete with
   tombstone-and-hydrate).
2. **The prefix plan *is* the partitioning.** Lifecycle tiers, TTLs, retention and regional
   placement are all expressed by writing to the right prefix (18 separates TTL'd intermediates
   from published clips; 35 caps its hot tier by age; 49's block layout sets retention boundaries).
   Choosing keys first is choosing the store's operating model.
3. **Durability over latency; buy latency from the cache in front.** The store optimizes for
   never-losing and cheap-at-rest; every serving case puts a CDN or POP tier between users and
   origin and treats origin traffic as the miss remainder, sized to a budget (01, 02, 32, 33, 34) —
   and the rejected uses are exactly the hot-latency cases (08's KV cache, 13's tiered reads).
4. **Cheap replay substrate.** Landing raw bytes once makes reprocessing cheaper than refetching:
   crawl HTML re-extracts, log windows re-train the model, metric blocks re-query, audit re-serves
   (14, 41, 46, 49, 50). The store's immutability is what makes replay a pure function of prefix +
   time range.
5. **The trade everywhere is retention cost vs recompute ability.** Cases either expire
   aggressively on an engagement skew (35), push everything to cold tiers (01, 17, 46), or retain
   the raw layer exactly because the derived layers are rebuildable (06, 14, 41) — the choice is
   always stated as money vs recoverability, never left implicit.

# 02 — Scale estimates

Assumptions (all rough estimates for a hypothetical YouTube/Twitter-scale product; adjust
and redo arithmetic if they change):

- ~500M DAU, average session ~30 min, ~2 sessions/day
- One feed page shows ~50 items; the user scrolls ~4 pages per session, ~2 sessions/day →
  **~400 impressions and 8 ranking requests per user/day** → 500M × 8 requests/day =
  **4B ranking requests/day**
- Item corpus: ~500M items; ~2M new items/day published
- Ranking funnel per request: corpus 500M → retrieval returns ~2,000 candidates →
  heavy ranker scores all ~2,000 → final page ~50
- Each scored item uses ~200 features; feature vectors ~1 KB per candidate scored
- Events logged: an impression + ~1.5 interaction events (watch/skip/like) → **~2.5 events
  per item shown**

## Read path (the one that matters)

**Ranking requests**: 4B/day ÷ 86,400 s ≈ **46K QPS average**, peak ~4× → **~200K QPS peak.**
Every request is a full funnel run. Arithmetic per peak second:

- Retrieval: 200K requests/s. ANN lookup over 500M × 128-dim embeddings returning ~2K
  neighbors: ~10 ms per request sits comfortably inside the ~100 ms per-request budget, so
  this is the cheap stage;
  roughly 200K × 10 ms = 2,000 core-seconds/s → needs a few thousand cores of ANN fleet
  (estimate; ANN servers are mostly memory-bound).
- Heavy ranker scoring: 200K req/s × 2,000 candidates = **400M candidate scores/s at peak.**
  A ~10 ms DNN forward pass per candidate batch of 256 on one scoring node ≈ 25K
  scores/s/node (estimate) → ~16K scoring nodes if unmitigated. This is the load that
  forces cheap pre-rankers, embedding caches and candidate caps — see deep dive 1.
- Feature fetch: 200K requests × (user features once + 2,000 item feature vectors) 
  ≈ 200K × 2K = **400M feature lookups/s** nominally — only survivable as batched
  multi-gets (~64–256 keys each) plus an embedding cache; ~400M/128 ≈ 3M multi-gets/s.
  Item-side values are mostly cacheable (hot items dominate impressions); user-side is
  per-request.

## Write path

- **Event volume**: 4B requests × 50 impressions = 200B impressions/day ≈ 2.3M impression
  events/s; ~1.5 interaction events per impression → ~2.5 events per impression ≈
  2.3M × 2.5 ≈ **~5.8M events/s on average**; at the ~4× peak factor used for QPS that is
  ≈ 4 × 5.8M ≈ **~23M events/s at peak** (estimate). Event log is
  append-only, partitioned by user_id for label joining: impressions are the fat events
  (200B × ~150 B ≈ **30 TB/day**), interaction events are thin (~300B × ~50 B ≈ **~15 TB/day**)
  → **~45 TB/day of raw events** (estimate; before dedup/compaction — training pipelines keep
  ~90-day windows).
- **Realtime feature updates**: streaming counters per item (views last 5 min / 1 h,
  recent-watch-rate) updated per event: ~6M events/s on average, ~23M/s at peak into a
  counter store — small writes,
  hot keys on viral items (a viral item's counter becomes a single hot key; needs local
  aggregation or per-shard sub-counters).

## Storage

- **Corpus sidecar**: 500M items × ~2 KB metadata ≈ **1 TB**; embeddings 500M × 128 dim
  × 2 B (fp16) = **~128 GB** — the whole embedding table fits in RAM across the ANN fleet
  (a tens-of-nodes fleet with 64+ GB each); this is what makes exhaustive-scan hybrid
  design viable (deep dive 2).
- **Feature store**: online features (user profiles ~100 KB × 500M users ≈ 50 TB; item
  statistics ~5 KB × 500M ≈ 2.5 TB + per-item embedding cached) — user features are the
  big tier, item features the hot small tier.
- **Corpus growth**: 2M new items/day × ~3.5 KB (metadata + embedding) ≈ **~7 GB/day** —
  negligible; the corpus grows by bytes, the event log grows by tens of TB.

## Consistency check

- 4B requests/day ÷ 500M DAU = 8 requests/user/day ≈ 4 pages/session × 2 sessions — matches.
- 200B impressions/day ÷ 4B requests = 50 impressions/request — matches the page size.
- Scoring load: 400M scores/s at peak only works because the heavy ranker scores ~2K, not
  500M — the ratio 500M → 2K (250,000×) is the single most load-bearing number in the case;
  it is why a funnel exists at all (deep dive 1).
- Storage asymmetry: model+feature serving state (~54 TB: 50 TB user features + 2.5 TB
  item stats + ~1 TB corpus sidecar, single replica; embeddings ~128 GB are negligible)
  vs event log (~45 TB/day) —
  the training loop is the storage-heavy side; serving is memory-heavy. Budget them apart.

## Calibration against public figures

- The two-stage shape (candidate generation "from the millions down to the hundreds, then
  ranking") and a deep ranking network are publicly described for YouTube recommendations —
  Covington, Adams & Sargin, "Deep Neural Networks for YouTube Recommendations" (RecSys
  2016). Our 500M → 2,000 → 50 funnel is the same architecture one tier wider; the paper
  does not publish serving QPS, so per-request arithmetic above is our estimate.
- The funnel also mirrors search engines (retriever + re-ranker) and Twitter's Earlybird
  search-index serving of the timeline (see [Twitter deep dives](../02.twitter/04-deep-dives.md)):
  in both, the expensive stage runs on a tiny candidate set, and the cheap stage exists to
  guarantee its candidates are findable (recall), not good (precision).

# 02 — Scale estimates

Assumptions (order-of-magnitude, constructed to be internally consistent — adjust and redo
the arithmetic if any change; these are **estimates**, not measurements):

- Consumer product: ~100M DAU. The platform serves several model surfaces; the ranking model
  dominates the load, so we estimate that one ([15.ml-ranking-system](../15.ml-ranking-system/README.md)
  describes the consumer).
- ~15 feed refreshes / user / day → **1.5B predict requests/day**.
- Retrieval narrows the corpus to ~500 candidates per refresh; one **batched** model call
  scores all candidates per request.
- ~500 features per scored row (user-side cached per user; item-side fetched per candidate
  per table).

## Prediction throughput

- 1.5B requests/day ÷ 86,400 s ≈ **17.4K QPS average**; diurnal peak ~3× → **~52K QPS**.
- Scored rows: 1.5B × 500 = **750B rows/day** ≈ 8.7M rows/s average, ~26M rows/s peak.
  With batched inference this is the number that budgets hardware: at ~1 ms/row per core
  (rough estimate for a mid-size DNN row), peak needs ~26K cores-equivalent of scoring —
  hundreds of hosts: real, but ordinary fleet sizing for a scoring tier. Note also the
  *consumer-side* coupling: the same request pays retrieval, network and page render
  (a ranking case's problem, not ours).
- Feature store reads: per request 1 user multiget (~300 sub-keys) + the item side, fetched
  per candidate per table: ~25 item-feature tables × 500 candidates, batched as one multiget
  per table ≈ 25 × 500 + 300 = **~12.8K rows/request** → 17.4K × ~12.8K ≈ **~220M KV rows/s
  average, ~660M/s peak**. Counting "lookups" instead of rows only works if every table
  fetch is a ~500-row batched multiget — the row count is what sizes the KV fleet. This,
  not model FLOPs, is the serving tier's real load — the same shape lesson as Twitter
  serving timelines from a precomputed cache instead of computing on read (see
  [02.twitter deep dives](../02.twitter/04-deep-dives.md)).

## Logging (the firehose that feeds everything else)

Every prediction logs: model version, request context, the feature values actually fetched,
and scores. Per scored row, ~500 features; columnar (dict/RLE) encoding puts the average
feature cell at ~2 bytes for repeated categoricals + raw numerics — call it **~2 KB/row
encoded (estimate)**.

- 750B rows/day × 2 KB = **~1,500 TB/day (~1.5 PB/day) logged** raw; compressed (zstd ≈ 3:1
  on this data) → ~500 TB/day landed. Over 30 days: ~15 PB compressed — the training window
  is a first-class storage citizen (PBs in object storage), not an afterthought; the year's
  log is ~**180 PB**.
- Logging write throughput: 1,500 TB/day ÷ 86,400 s ≈ **17.4 GB/s average raw writes**
  (~6 GB/s compressed landed) — a serious ingest tier (large Kafka + parquet-compaction
  pipeline), not a sidecar. The hard part is not volume; it is **schema discipline**
  (deep dive 1) and compaction latency versus retrain start time.

## Training pipeline (petabytes through Spark)

- Full retrain of the hot ranker: 30-day window ≈ 750B rows/day × 30 = 22.5T rows × ~2 KB
  ≈ **~45 PB raw → ~15 PB compressed read per training cycle**. At ~3 GB/s aggregate read
  (a ~500-node pool's rough planning number) that is 15 PB ÷ 3 GB/s ≈ **~1,400 h (~58 days — ~2 months)**,
  so full rebuilds burst to a ~10× pool (~30 GB/s aggregate, still a rough planning number):
  15 PB ÷ 30 GB/s ≈ **~6 days**; the nightly **incremental** build over 1 day of data
  (~500 TB compressed) also runs at the burst rate — 500 TB ÷ 30 GB/s ≈ **~5 h** (at the
  standing pool's 3 GB/s it would take ~2 days, too slow for a daily retrain cadence) — and
  full point-in-time rebuilds are quarterly audits, not a routine.
- Label window: events stream in, labels (click/conversion) finalize over ~24 h; the nightly
  job builds "day-2 complete" partitions. This is why training always lags by ≥1 day —
  trading one day of model staleness for never training on half-labeled data.
- Backfills: a feature transformation changes → recompute *that feature* over its full
  history (~1 year of impressions ≈ 180 PB compressed; one feature's columnar slice of it
  is ~0.5–1 PB, an estimate — ~274T rows × 2–4 B/cell) as a one-off Spark job, not the
  whole training set — a targeted columnar backfill is ~100× cheaper than rebuilding every
  training set.

## Feature store

- Online store: ~100M user entities + ~1B item entities; ~500 features each, ~5–20 KB per
  object after packing → **~5–20 TB hot data**. Too big for one box, trivial for a
  consistent-hash KV cluster (few hundred GB/box); candidate-side features live as
  precomputed per-table blobs — compute-on-write for items, compute-on-read (cheap, cached)
  for the one user vector per request. Same hybrid Twitter picked for timelines.
- Offline store: 7 days of raw events + snapshots ≈ 7 × 1.5 PB ≈ ~10 PB raw / ~3.5 PB
  compressed; snapshots of the "current" feature tables are rebuilt after each batch cycle.
- Writes: streaming feature updates ~100M DAU × ~50 updates/day ≈ 5B events/day ≈ 58K/s —
  small events (~100 B) → ~6 MB/s streaming ingest, again modest; the cost is keeping
  stream and batch aggregates *consistent* (deep dive 2).

## Registry, serving fleet, monitoring

- ~200 model families across the company; top ~10 retrain daily → **~10–20 new registry
  versions/day**; ~50 model versions resident on the serving fleet.
- Artifacts: hot ranker ~0.5–1 GB (embedding tables dominate); ~1,000 retained versions ≈
  1 TB in artifact storage — storage is a non-problem; **distribution speed** to hundreds of
  servers is the constraint (pull + verify + load, deep dive 4).
- Monitoring: PSI per feature per model per hour: ~500 features × ~20 live models ≈ 10K
  distribution comparisons/hour, computed from the same logged data — monitoring rides the
  logging firehose; no separate capture path.

## Consistency check

- 1.5B requests/day ÷ 100M DAU = 15 refreshes/user/day — plausible for a heavy feed product.
- 1.5 PB/day logged vs ~500 TB/day retained ≈ consistent with columnar training sets of 2 KB/row.
- Feature-store reads (~12.8K rows/request vs 1 scored request) ≈ ~12,800:1 amplification —
  the estimate deliberately makes feature I/O the dominant serving cost; any redesign that
  doubles features per model doubles this line, which is why candidate features are
  compute-on-write and never recomputed per request.

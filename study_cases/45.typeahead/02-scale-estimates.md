# 02 — Scale estimates

Assumptions (adjust and redo arithmetic if they change):

- 100M DAU, ~8 searches/user/day → **800M search sessions/day**.
- A session fires ~3 debounced prefix requests (first letters often hit the client cache)
  → **~2.4B typeahead requests/day** ≈ 28K QPS average, ~3× peak ≈ **~85K QPS**.
- Each request returns ~8 candidates; response ~1 KB gzipped → egress ≈ 2.4B × 1 KB ≈
  **~2.4 TB/day** — noise, edge-cacheable for the hottest prefixes.
- p50 budget 30 ms / p99 100 ms (est. 01) — sub-100 ms is the design bar for in-typing aid.
- Query log: 800M logged queries/day; distinct normalized strings/day ~40–60M (est.);
  rolling 90-day window → candidate corpus of distinct strings with usable counts ~
  100–200M; all-time distinct grows to ~1B but contributes almost nothing to serving
  (long tail never ranks top-8).

## Read:write asymmetry (the number that shapes the design)

- Index "writes": the top-k materialized index is rebuilt batch-side hourly; the hot layer
  is updated per-event. Online index reads: **2.4B/day**.
- Ratio: ~2.4B reads/day vs index writes the online path never performs — the stable index
  rebuilds ~hourly, so ~100M reads ride each rebuild event (~10^8 reads per rebuild) and its
  live top-k mutations number ~a few million/day → ≥ ~10^3:1 by any count — every
  microsecond of query-time computation multiplies by 28K QPS; build-side work
  is amortized over minutes. The index is therefore *materialized per prefix*, and the online
  path never ranks (deep dives §1). Same asymmetry family as
  [40.tinyurl](../40.tinyurl/README.md) (~100:1) and [31.netflix](../31.netflix/README.md)
  catalog reads, but at the extreme where even a single cache-bypassing scan is a design bug.

## Index size (what fits where)

- Materialized index: prefixes with cumulative count ≥ ~100/90-day-window (est.) ≈
  **~30M live prefixes**. Per prefix: top-8 entries × (term-id 4 B + weight 4 B) = 64 B →
  30M × 64 B ≈ **~2 GB** of core entries.
- String pool for hydration: ~200M terms × ~30 B ≈ ~6 GB (+ dedup overhead) —
- Structure overhead (trie/FST pointers, offsets, per-shard maps) ≈ 5–10× the core
  (~10–20 GB) + the ~6 GB string pool ≈ 16–26 GB → order **~10–30 GB total per full
  replica**. A modern node holds this in RAM trivially:
  the index is *RAM-resident, sharded by prefix (e.g., hash of the normalized first 1–2
  characters) across ~20–50 service nodes, hot shards replicated more*.

## Traffic shape / Zipf

- Prefix popularity is heavily skewed: the top ~1% of by-traffic prefixes (2-letter and
  popular words) plausibly carry the bulk of request volume (Zipf-tail est.) → per-shard
  load: 85K peak QPS over ~30 shards averages ~3K/shard, but the shard owning "a"/"ne"
  (the most common openers) can see 5–10% of fleet traffic → **hot-shard + hot-key problem is
  the serving design constraint**, not average QPS (deep dives §3): replicate hot shards,
  coalesce same-prefix lookups in flight, serve stale-on-error.
- Query-time CPU: one point lookup + hydration; target < ~1 ms server-side compute at p50 —
  the rest of the 30 ms budget is network/queueing.

## Ingest/events

- Query events: 800M/day ≈ **~9K events/s average** fed to counters + batch build —
  comparable to [13.kafka](../13.kafka/README.md) topics of modest size; the streaming layer
  is small (est.), the correctness question is *freshness windows*, not throughput.

## Consistency check

- Sessions sanity: 800M sessions ÷ 100M DAU = 8/day ✓ (assumption used, trivially closed).
- Requests sanity: 3 requests/session × 800M = 2.4B ✓; ÷ 86,400 ≈ 28K QPS ✓; the ×3 peak
  matches a workday/evening search concentration, consistent with
  [02.twitter](../02.twitter/README.md)-style diurnal factors.
- Index-size sanity: 30M prefixes × 8 entries = 240M top-k slots — far below the ~200M-term
  string pool × ~8 prefixes a ~7-char term could realize in (~1.6B possible (prefix, term)
  pairs); consistent. RAM budget ~10–30 GB for a full
  replica (< ~1–2 GB per shard node, split ~20–50 ways) is
  why nobody serves typeahead from a general search engine when a KV-in-RAM will do.

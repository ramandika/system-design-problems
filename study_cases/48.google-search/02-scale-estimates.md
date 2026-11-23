# 02 — Scale estimates

All numbers are rough estimates, derived arithmetic unless sourced. Public anchors kept
in mind: Google has publicly stated its index covers "hundreds of billions of pages"
(its 2018 "How Search works" description); publicly reported search volumes are in the
"billions per day"; Barroso et al.'s "The Datacenter as a Computer" and "Taming the Long
Tail" describe index queries fanning out to thousands of shards. Everything else here is
derived and labeled as an estimate.

Corpus figures deliberately stay consistent with [Case 41](../41.web-crawler/02-scale-estimates.md):
its crawl target is ~100 B pages over a 30-day sweep, ~4 B pages/day. This case indexes
what that pipeline feeds it.

## 1. Corpus

- **Base index (the deep tier)**: ~100 B documents (consistent with the low end of
  "hundreds of billions"; matches 41's maintainable-corpus assumption ≈ the full sweep).
- **Documents "known but thin"** (URLs seen, some fetched-and-discarded as dup/spam:
  the 40 B/day discovery in [41 §4](../41.web-crawler/02-scale-estimates.md) accumulates
  here): ~1 trillion URL rows, but they are URL-table state, not indexed documents.
- **Fresh tier**: ~1–2 B documents = the 1% high-churn tier from
  [41 deep dive 6](../41.web-crawler/04-deep-dives.md), refreshed continuously.

```
base 100 B + fresh 1–2 B  → ~100–102 B indexed documents total
```

## 2. Documents and terms

```
mean tokens per extracted document ≈ 1,000     (rough: mean extracted main-text ~8 KB,
                                                ~1 token per 6–8 B → ~1,000–1,300 tokens)
total postings (term, doc)        = 100e9 × 1e3 = 1e14  ≈ 100 trillion postings
```

Postings count every (term, document) pair, so repeated terms in one document produce
multiple postings with positional data (needed for phrase queries).

```
vocabulary size: ~500 M unique terms          (rough: web text including misspellings,
                                               numbers, product codes, non-English;
                                               public IR studies of web crawls are in the
                                               100 Ms; estimate)
```

## 3. Index storage

Per-posting cost after the compression scheme in deep dive 1 (gap encoding + variable-
byte + block-min: doc-ID gaps are small for frequent terms, so the amortized average is
a few bytes):

```
postings   ≈ 1e14 postings × 3–4 B  ≈ 300–400 TB
positions  (positional postings, needed for phrases — only kept for terms with df below a
            cutoff; skip for stopwords):  assume ~30% of postings carry positions at +2 B
            ≈ 0.3 × 1e14 × 2 B      ≈ 60 TB
term dictionary ≈ 500 M terms × ~100 B (term, df, pointer, per-block stats)
                                    ≈ 50 GB whole-corpus → ~25 MB per shard's own terms
                                      (kept in RAM on the shard node)
document-level side data   (static quality score, term vectors for snippets, page
                            features): ~100 B docs × ~1 KB ≈ 100 TB
page/snippet store         (extracted text, [41 §8](../41.web-crawler/02-scale-estimates.md)
                            gives ~8 KB extracted mean): 100 B × 8 KB ≈ 800 TB
fresh tier index           (1–2 B docs × 1,000 postings × ~5 B uncompressed
                            + overhead ≈ 5–10 TB — uncompressed because rebuild is cheap)
```

Total serving-state ≈ **1.5 PB** (300–400 TB postings + 60 TB positions + 100 TB side data
+ 800 TB page store + change). This is small relative to what the crawler *stores*
(~55–73 PB/yr raw blobs in [41 §3](../41.web-crawler/02-scale-estimates.md)) — the index
is the derived, compressed, heavily-pruned view; the blob corpus is the recovery point.

## 4. Sharding

Document-partitioned (deep dive 2), so every query touches every shard:

```
shards = 2,000           → 100 B / 2,000 = 50 M docs per shard
index per shard  ≈ 400 TB postings / 2,000 ≈ 200 GB postings
                   + ~50 GB positions/side data + a slice of the page store
                   ≈ 600–700 GB serving state per shard node (SSD-based; hot tier in RAM)
```

2,000 is itself a *cost* choice: it is the same fanout number publicly described for
web-scale systems ("a query fans out to thousands of shards", Barroso et al.). Fewer
shards = larger per-shard state; more shards = higher per-query fanout cost and worse
tail latency (deep dive 4).

## 5. Query volume

```
queries/day ≈ 2.5 B          (rough estimate; publicly reported figures are "billions
                              per day" — we take the low end for a leading product)
queries/s   = 2.5e9 / 86,400 ≈ 29,000 ≈ 30 k QPS sustained average
diurnal peak ≈ 2× → 60 k QPS
hot-query burst: single queries can spike 10–100× (breaking news). Un-cached, that is
             60 k–600 k QPS against shards; cached, it collapses (§8).
```

## 6. Cost per query (the binding constraint)

Every query broadcasts to 2,000 shards. Per-shard index-read cost depends entirely on
whether the retrieval stage prunes:

```
Naive scoring (read every posting of every query term):
   postings read per query ≈ (df of the rarest term × a few terms) — for a rare term,
   df ~ 1e6–1e8 on 100 B docs; scoring even 1e6 candidates × ~50 ns ≈ 50 ms CPU/shard
   → 2,000 shards × 50 ms = 100 s CPU per query  → 30k QPS × 100 s = 3,000,000 cores
   (clearly impossible — this is why pruning is not optional)

Early-terminated retrieval (WAND + static score ordering, deep dive 4):
   assume pruning examines ~1–5% of the rare term's postings and skips the rest
   → 0.5–5 ms CPU per shard, call it ~2 ms avg
   per-query CPU = 2 ms × 2,000 = ~4 s CPU aggregate
   fleet for 30 k QPS = 30,000 × 4 s ≈ 120,000 cores of retrieval, plus:
     coarse ranker  (~0.1 s CPU/query on the top ~5k candidates)
     heavy ranker   (~0.5–2 s CPU/query on the top ~200 candidates, neural)
   → ~2–3 s more per query → total ~6–7 s CPU/query
   fleet ≈ 30,000 × 7 ≈ 200–250 k core-equivalents sustained (peak ~2× that)
```

The naive-vs-pruned factor is ~15–25× of CPU (100 s → 4–7 s per query; 3 M cores →
200–250 k), or ~50× on postings read (pruning examines ~1–5%). Estimates; the point is
the ratios, not the exact figures.

## 7. Latency breakdown (p50 target ~200 ms)

```
query parse + spelling + fan-out dispatch:            ~2–5 ms   (data-center RTT only)
retrieval on shards (top-K per shard, parallel, p99):  ~10–50 ms (the fanout tail
                                                       dominates: p99 per shard ≈ many
                                                       × median; deep dive 4)
top-K merge + coarse ranker (frontend service):        ~10–30 ms
heavy ranker (neural, batched over ~200 docs):         ~20–50 ms (model bound)
snippet fetch (page store, hot subset only):           ~5–15 ms
SERP assembly + serialization + network to client:     ~20–50 ms residual
p50 total ≈ 70–180 ms — leaves headroom to a 200 ms target
p99  is dominated by stragglers in the retrieval fanout (deep dive 4), not by medians
```

## 8. Caching

```
query distribution is Zipfian; assume top-100k queries ≈ 20–30% of volume,
top-1M ≈ 35–45%  (rough IR-industry folklore numbers — the exact curve is private)
results-cache hit rate target ≈ 20–30% at a short TTL (minutes, freshness-capped)
effective fanout QPS to shards = 30 k × (1 − 0.25) ≈ 22–24 k
cache also flattens burst events: a 10× spike on one query costs ~10 cached reads +
   ~1 fanout (the first), not 10 fanouts   → hot-query bursts stop at the cache layer
snippet cache: per-(query, doc) snippet → hit rate similar, ~20%
```

## 9. Ingest (index-building) load

```
pages/day into the index from [41](../41.web-crawler/02-scale-estimates.md) §1:
   4 B/day fetches; after dedup, dead pages and thin pages ~2–3 B survive to be indexed
   ≈ 25–35 k docs/s steady-state into the indexing pipeline
indexer throughput budget: parse → tokenize → (term,doc) postings, delta + compress,
   write to fresh tier segment: ~1–2 s CPU per 1k docs (rough)
   → ~50–100 CPUs for ingest  (tiny relative to serving — indexing is cheap compared
     to *serving* 30 k QPS; the expensive index work is compaction/merging, deep dive 3)
```

## 10. Summary table

| Quantity | Estimate | Derived from |
|---|---|---|
| Indexed docs | ~100 B base + 1–2 B fresh | consistent with 41's 100 B corpus; public "hundreds of billions" |
| Postings | ~1e14 (100 T) | 100 B docs × 1,000 tokens mean |
| Vocabulary | ~500 M terms | estimate, web-corpus IR studies |
| Postings storage | 300–400 TB | 3–4 B compressed per posting |
| Full serving state | ~1.5 PB | postings + positions + side data + page store |
| Shards | 2,000 × 50 M docs | broadcast-query cost vs per-shard state |
| Query QPS | ~30 k avg, ~60 k peak | 2.5 B/day ÷ 86,400, ×2 diurnal |
| Naive CPU/query | ~100 s (≈ 3 M cores at 30 k QPS) | unpruned score of full posting lists |
| Pruned CPU/query | ~7 s (≈ 200–250 k cores) | WAND + staged ranking §6 |
| p50 latency | 70–180 ms | §7 breakdown |
| Results-cache hit | ~20–30% | Zipfian query distribution §8 |
| Ingest docs/s | ~25–35 k/s | 4 B/day × 0.5–0.75 survival ÷ 86,400 |

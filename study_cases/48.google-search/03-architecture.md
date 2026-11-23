# 03 — Architecture

## 1. The one-paragraph picture

Three planes. The **build plane** consumes the crawler's handoff stream
([41 §3.6](../41.web-crawler/03-architecture.md)): it tokenizes, assigns doc-IDs, and
writes immutable index *segments* into a small fresh tier that merges over time into a
large, mostly-immutable base index. The **serving plane** fans each query out to all
~2,000 base-index shards plus the fresh-tier replicas, runs pruned top-K retrieval on
each, merges, ranks in stages, and composes the SERP with snippets from the page store.
The **cross-cutting plane** is caches and the page/snippet stores. Nothing in the query
path coordinates globally; the only "global" things are the query cache (keyed by query,
short TTL) and the doc-ID space assigned at indexing time.

## 2. Diagram

```
  ┌──────────────────────────── BUILD PLANE ────────────────────────────┐
  │  handoff stream  (content records, links, freshness signals)        │
  │  from [41](../41.web-crawler/03-architecture.md) §3.6               │
  │        │                                                            │
  │        ▼                                                            │
  │  ┌──────────────┐   ┌──────────────┐    ┌──────────────────────┐    │
  │  │ INDEXER      │──▶│ FRESH TIER   │───▶│ BASE INDEX           │    │
  │  │ docid assign │   │ (small immut.│  seg-│ (imm. segments,      │    │
  │  │ tokenize, pg │   │  segments,   │ merge│  ~2,000 shards,      │    │
  │  │ postings, cmp│   │  minutes-old)│      │  ~50 M docs each)    │    │
  │  └──────────────┘   └──────┬───────┘    └──────────┬───────────┘    │
  └────────────────────────────┼───────────────────────┼────────────────┘
                               │ queried in parallel   │ queried in parallel
                               ▼                       ▼
  ┌──────────────────────────── SERVING PLANE ──────────────────────────┐
  │                    ┌────────────────────┐                           │
  │ user query ───────▶│ FRONT-END (FE svc) │◀─── query cache (results, │
  │                    │ parse, spell,      │      short TTL, ~25% hit) │
  │                    │ plan → fanout      │                           │
  │                    └──┬───────┬─────────┘                           │
  │        ┌──────────────┘       │   ┌──────────────┐                 │
  │        ▼ broadcast            │   ▼ broadcast   ▼                 │
  │  ┌───────────────┐    ┌───────────────┐   ┌────────────────────┐    │
  │  │ BASE SHARD    │    │ BASE SHARD ...│   │ FRESH-TIER REPLICA │    │
  │  │ leaf: top-K   │    │ (2,000 total) │   │ + vertical leaves  │    │
  │  │ WAND retrieval│    │               │   │ (news/imgs/video)  │    │
  │  └──────┬────────┘    └──────┬────────┘   └─────────┬──────────┘    │
  │         │   per-shard top-k  │                      │               │
  │         └────────────┬───────┴──────────────────────┘               │
  │                      ▼                                              │
  │  ┌──────────────────────────┐     ┌──────────────────────┐          │
  │  │ MERGE + COARSE RANKER    │────▶│ HEAVY RANKER         │          │
  │  │ top ~5k → order          │ top~200 (neural model,          │          │
  │  └──────────────────────────┘     │  feature fetch)      │          │
  │                                   └──────────┬───────────┘          │
  │                                              ▼                      │
  │  ┌──────────────────────────┐     ┌──────────────────────┐          │
  │  │ PAGE STORE / SNIPPETS    │◀────│ SERP COMPOSER        │          │
  │  │ (extracted text 800 TB,  │     │ (snippets highlighted│          │
  │  │  per-doc, query-aware    │     │  + verticals + cache │          │
  │  │  cache [§8 02])          │     │  write-back)         │          │
  │  └──────────────────────────┘     └──────────┬───────────┘          │
  └──────────────────────────────────────────────┼──────────────────────┘
                                                 ▼
                                              response
```

## 3. Components

### 3.1 Indexer

Consumes the crawler stream (at-least-once; idempotent by doc canonical-URL hash — a
re-delivered document rewrites the same doc-ID). Pipeline: parse extracted record →
tokenize (language-specific, lowercase, stopword handling deferred: stopwords stay in the
postings for positional lookup but the deep dive 1 compression exploits them) → assign
doc-ID (sequential per shard; *not* global — IDs need only be unique within a shard, which
keeps gap-encoding small) → emit (term, doc, score, position) postings → apply deep dive 1
compression → write an immutable fresh-tier segment. Ingest ≈ 25–35 k docs/s
([02 §9](02-scale-estimates.md)) — cheap CPU, the discipline of immutability is what
matters, not throughput.

### 3.2 Fresh tier

Small, per-shard-set tier holding minutes-to-hours-old documents. Each shard group has
fresh-tier replicas queried alongside its base shard. Segments are tiny (minutes of
ingest each), unmerged, with modest compression. Documents here are *either not in the
base index yet* (new) or *about to leave* (updated versions shadow the base copy —
dedup rule: fresh-tier doc-ID wins on doc-key tie). A background merger periodically
promotes fresh-tier content into the base index (deep dive 3 covers merge policy and
failure modes).

### 3.3 Base index (sharded inverted index)

Document-partitioned: each shard owns ~50 M documents and the *complete* posting lists
for its docs. Per shard: term dictionary (~50 GB whole-corpus, ~25 MB per shard — kept in
RAM), postings on SSD with a hot cache, per-chunk skip pointers, pre-staged static
quality scores in the doc-ID gap structure so high-quality docs are encountered first in
posting-list order (impact ordering — deep dive 1). 2,000 shard machines (+2× replicas)
run entirely read-only during query time.

### 3.4 Front-end query service (FE)

Stateless, horizontally scaled. Does: parse, tokenize, spelling correction (FR7), decide
what to broadcast (base shards + fresh tier + eligible verticals), dispatch in parallel,
enforce the latency budget with per-stage deadlines, merge, and write the result into the
query cache. Its most important data structure is the *deadline clock*: each stage gets
a budget; missing one means degrade (deep dive 4), not wait.

### 3.5 Ranking stages

- **Merge + coarse ranker** (on FE or a nearby service): combine ≤2,000 per-shard top-Ks
  into ~5 k candidates, score with a cheap model on already-indexed features (match
  quality, static score, term-level signals). Same "millions → thousands" step as the
  [Ranking System](../15.ml-ranking-system/README.md) funnel.
- **Heavy ranker**: batch a re-scored top ~200 and run the expensive neural model
  (cross-encoder: query and document jointly). Feature fetch pulls page features from the
  page store; model is a versioned artifact (NFR7); serves in tens of ms.

### 3.6 Page store / snippets

Extracted text keyed by doc-ID ([41 §8](../41.web-crawler/02-scale-estimates.md)'s 8 KB
mean gives ~800 TB). Snippets are generated *at query time* for the ~10–30 docs that
surface (read a few KB per doc, pick the best-matching sentences) and are cached
query-aware. Keeping snippets query-dependent is why the snippet is a query-time job —
a stale pre-computed snippet cannot match a query invented after the crawl.

### 3.7 Caches

- **Results cache**: full per-query result set, keyed on normalized query (not including
  personalization — personalization is a cheap post-rank reorder on top of the cached
  base result). TTL short (minutes), freshness-capped, cluster-local.
- **Snippet cache** and **page-store block cache**: bounded LRU, hot-document-weighted.
- **Misses**: only the ~70–80% miss rate reaches shards ([02 §8](02-scale-estimates.md)),
  and burst events hit the cache first — this is what keeps hot-query spikes from
  becoming hot-shard storms.

## 4. Data flows

1. **Ingest**: crawler stream → indexer → fresh segment → (merge) base segment. Doc
   versions shadow old versions until promoted; deletes are tombstones applied at
   promotion, not eager index surgery.
2. **Query (base path)**: query → FE parse/spell → cache check (hit ≈ 25%: serve) →
   broadcast to 2,000 base shards + fresh tier + verticals → each shard returns ≤K
   candidates with their retrieval scores → merge to ~5 k → coarse ranker → top ~200 →
   heavy ranker → top 10 → snippet fetch → compose → cache-write → respond.
3. **Query (fresh path)**: fresh-tier replicas are queried in the *same* broadcast;
   their candidates join the same merge pool and are ranked by the same pipeline —
   freshness is a feature in the ranker (deep dive 5), not a separate result lane.
4. **Degradation flow**: any stage overrun → FE uses what it has: fewer candidates →
   coarser ranking; vertical timeout → drop the vertical; a shard's partial results →
   answer with them rather than wait past deadline (deep dive 4).

## 5. Failure handling at a glance

- **Index shard dies**: serving routes around it; response is built from
  1,999 shards' partial results + replica promotion. The freshness consequence (a slice
  of docs unreachable for minutes) is bounded by replica count.
- **Fresh-tier replica dies**: fresh docs go temporarily unfindable. Base index is
  unaffected; freshness SLA slips, not correctness. Rebuild from the stream
  ([13.kafka](../13.kafka/04-deep-dives.md) replay).
- **Heavy ranker degraded**: fall back to coarse ranker ordering (fewer features, worse
  ordering, same ten links). This is a *relevance* degradation, not an outage (NFR2).
- **Crawler stops feeding**: everything keeps serving stale (NFR4 is violated, NFR2
  isn't), which is the correct failure direction for a read system.
- **Cache cluster dies**: shards see full QPS again — miss-side load goes from
  ~22.5 k to the full 30 k queries/s (25% hit rate, [02 §8](02-scale-estimates.md)
  arithmetic), a ~1.3× jump that spikes to ~2× at peak. Shards must be sized to survive
  a ~2× QPS spike, not just the average; a cache-flush storm is the classic outage
  scenario for a read-heavy system.

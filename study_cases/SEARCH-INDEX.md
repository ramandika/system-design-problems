# Search index across the case catalog

Where keyword/full-text search-index technology — inverted indexes, posting lists, prefix
indexes, Elasticsearch/Lucene-class engines — appears in the 50 study cases, what role it
plays, and how deep each case goes. Companion to [CONCEPTS.md](CONCEPTS.md) (Tier 3 union:
"Search / inverted index", 8/43 cases). Semantic / embedding search is a sibling doc,
[VECTOR-SEARCH.md](VECTOR-SEARCH.md) (being written in parallel; same doc family, different
technology).

Headline: **in this catalog the search index is the canonical *read-optimized projection*
— and every case that designs one pays the same three bills: the fresh-vs-base tier trade,
the update-freshness vs serve-cost trade, and the partition-axis trade.** Only two cases
study the indexing engine itself; everywhere else it is a derived structure built off an
event spine or an authoritative store, updated asynchronously, and honest about lag.

- Cases that mention search-index technology (keyword/full-text): **16 / 50** —
  *mention-based count, including one-line refs; distinct from CONCEPTS.md's designed-component
  union of 8/43*
- A search index is a designed architectural component: **9**
- Studied as the case itself (the indexing system *is* the product): **2** (48, 45)
- Brief mention / indexing as an async side-path only: **4** (05, 08, 11, 33)
- Explicit out-of-scope boundary: **1** (41 — "the search index, ranking, and query path
  themselves" handed off to 48)
- Vector-only cases excluded from this doc: **1** (15 → [VECTOR-SEARCH.md](VECTOR-SEARCH.md))
- False positives filtered out: **3** (07, 28, part of 41 — see boundary note)

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [48.google-search](48.google-search/README.md) | The case itself: the 100 B-doc inverted index, built and sharded | Full case |
| [45.typeahead](45.typeahead/README.md) | The case itself: prefix index with materialized top-k per prefix | Full case |
| [39.e-commerce](39.e-commerce/README.md) | Search index as a projection off the event spine; faceted postings | Deep component |
| [36.airbnb](36.airbnb/README.md) | Search engine as the read plane: geo-cell + date-bitmap postings, delta-updated | Deep component |
| [02.twitter](02.twitter/README.md) | Earlybird search index reused as the celebrity timeline source (real system) | Component (by reference, deep) |
| [32.spotify](32.spotify/README.md) | Two indexes split by vocabulary: typeahead prefix index + ranked search index | Component |
| [26.google-maps](26.google-maps/README.md) | FST/trie POI index: prefix completion + fuzzy via Levenshtein automata | Component |
| [06.apple-app-store](06.apple-app-store/README.md) | Tens-of-GB in-memory inverted index + per-storefront bitset deltas | Component |
| [30.facebook-messenger](30.facebook-messenger/README.md) | Async derived search index over mailbox; E2EE pushes it client-side | Component (moderate) |
| [42.linkedin](42.linkedin/README.md) | Member-directory search service, Galene-style Lucene backend | Component (moderate) |
| [49.monitoring-metrics](49.monitoring-metrics/README.md) | *Custom* inverted index over label values, not text — postings shape reused | Component (engineered) |
| [41.web-crawler](41.web-crawler/README.md) | Indexer explicitly out of scope; the crawler ends at the handoff | Boundary |
| [05.discord](05.discord/README.md) | "Search indexing" as one async fanout worker hook | Brief |
| [08.chatgpt](08.chatgpt/README.md) | "search indexes" as a storage-tally line item | Brief |
| [11.zookeeper](11.zookeeper/README.md) | Solr named only in the list of systems ZooKeeper coordinated | Brief (analogy) |
| [33.instagram](33.instagram/README.md) | Author recent-posts index cited via Twitter's Earlybird pattern | Brief (by-reference) |

## Per-case notes

### The case itself

**[48.google-search](48.google-search/README.md)** — the catalog's reference implementation
of an inverted index at web scale: ~100 B docs, ~1e14 postings ([02 §3](48.google-search/02-scale-estimates.md)).
Key decisions in [04-deep-dives.md](48.google-search/04-deep-dives.md):

- **Posting-list layout (dive 1)**: sorted doc-ID gaps + variable-byte compression with
  chunked blocks, skip pointers, and block-max impact metadata — uncompressed postings would
  be 8 B × 1e14 ≈ 800 TB, ~2× the compressed 300–400 TB, on SSD; skip + block-max pruning
  is the ~50× less posting-list work the cost model stands on. Doc-IDs are assigned
  sequentially per shard in importance order (small gaps, high-quality docs at list heads),
  at the cost of shard-local, rebuild-scoped identity.
- **Sharding axis (dive 2)**: document-partitioned, 2,000 × 50 M docs. Term-partitioning
  (Option A) wins single-term locality but turns every real multi-term query into a
  distributed join and concentrates hot terms; document partitioning keeps all intersection
  local and pays a **broadcast tax** — cost per query ∝ shard count. Explicit handoff:
  term/prefix partitioning *is* the right axis for autocomplete-class workloads
  ([45.typeahead](45.typeahead/README.md)), which is precisely why that case sharded by prefix.
- **Fresh/base tiering (dive 3)**: fresh tier holds minutes-to-hours of ~25–35 K docs/s of
  arrivals as small mutable-ish segments priced at 5–10 TB (5–10% of base), shadowing stale
  base copies until background promotion merges it in; deletes apply at promotion, not
  eagerly. Explicitly compared to [Kafka segment compaction](13.kafka/03-architecture.md) /
  [KAFKA.md](KAFKA.md) and to the update-hates-read-optimized-index trade of LSM compaction
  ([12.bigtable](12.bigtable/03-architecture.md), [BIGTABLE.md](BIGTABLE.md)).
- **Query-side pruning (dive 4)**: WAND over block-max metadata, ~1–5% of postings
  examined, per-stage deadlines + hedged requests against a 2,000-wide broadcast tail.
- **Ranking (dive 5)**: retrieval (the index's job) → coarse ranker on *indexed* features →
  heavy ranker on *fetched* features — the retrieval-then-rank funnel of
  [15.ml-ranking-system](15.ml-ranking-system/README.md) / [CONCEPTS.md](CONCEPTS.md), with
  index-time feature freshness tied to the promotion cadence.

**[45.typeahead](45.typeahead/README.md)** — the other half of the search-index family: the
*prefix* index. The corpus is small (30 M live prefixes materialized; ~10–30 GB per full
replica per [02](45.typeahead/02-scale-estimates.md)) so the entire difficulty is latency
discipline and freshness. Key decisions in
[04-deep-dives.md](45.typeahead/04-deep-dives.md):

- **Materialized top-k per prefix vs query-time trie walk (the dive)**: Option A (walk a
  trie/FST per request, Lucene-style prefix queries) is always fresh but pays per-request
  CPU proportional to subtree size at 28 K avg / 85 K peak QPS. Chosen: **B** — offline
  build materializes top-8 × ~64 B per qualifying prefix, serving is a hash + memcpy (hundreds
  of µs), with a bounded fuzzy fallback for the long-tail prefixes that never earn a slot.
  Sub-decision: sorted segment files, *not* a live trie — "a trie is the right shape only
  when you refuse to materialize". Note the arithmetic: 30 M prefixes × 8 entries ≈ 240 M
  slots, far below the ~1.6 B possible (prefix, term) pairs — materialization is what keeps
  the index small by dropping the tail.
- **Two-speed freshness (dive 2)**: stable index (hours old) + streaming trending overlay
  (minutes) with guardrail caps, merged at query time; two cache TTL regimes. The same
  fresh/base split as 48's tiers, one order of magnitude smaller and latency-driven.
- Serving tiers (dive 3): client debounce → edge snapshots → Redis per-prefix cache with
  coalescing → RAM index shards replicated by observed skew — the Redis hot-key absorber of
  [REDIS.md](REDIS.md) under extreme Zipf.

### Designed components

**[39.e-commerce](39.e-commerce/README.md)** — the cleanest "index as projection" in the
catalog. The order/price/ATP core stays relational ([deep dive 1](39.e-commerce/04-deep-dives.md));
**the search index is just another consumer of the ~15 B events/day spine
([02](39.e-commerce/02-scale-estimates.md))** — ~1–2 TB of postings over 400 M docs with
~2 KB indexable text each (400 M × 2.5–5 KB/doc ≈ 1–2 TB), plus 30 M × 20 per-region
availability bitsets in the tens of GB.
The deep dive in [04](39.e-commerce/04-deep-dives.md) makes the facet trade explicit:
facets are **pre-aggregated at indexing time into per-field posting lists with result-count
metadata**, never computed by querying buckets at request time — counts may be sampled
estimates on long-tail combinations, acceptable because a facet count's job is navigation,
not bookkeeping. Availability and price bands live *inside* the index so an out-of-stock
item is never ranked, not filtered after ranking (post-filtering breaks facet counts and
pagination). One named failure: staleness of the index vs truth is bounded to minutes and
rebuild tooling is mandatory.

**[36.airbnb](36.airbnb/README.md)** — a search engine that carries *structural* filters,
not just text. Sized in
[02](36.airbnb/02-scale-estimates.md): 8 M listings × ~10 KB indexed (text, geo cells,
filter postings, per-month day bitmaps) ≈ **~80 GB × 5–10 replicas** — small enough that
the entire searchable corpus is RAM-resident, so **the index is sharded only by query load
and geography, never by capacity** ([02](36.airbnb/02-scale-estimates.md)). Decisions in
[04 dive 1](36.airbnb/04-deep-dives.md):

- **Geo as posting lists inside the engine**, not a separate geo database: H3-like cell
  postings with the parent-inclusion trick at viewport edges; post-verification beats
  expanding 50 child cells.
- **Date availability as per-listing bitmaps in the scorer, not postings**: intersecting
  seven "day available" posting lists per 7-night query drags the corpus through seven
  giant lists; a ~31-bit monthly bitmap per candidate is O(nights) bit ops instead.
- **Index update path (dive 2)**: delta push on booking/hold events (fresh in seconds) +
  nightly full rebuild *as an audit* — drift found is an incident, not the repair mechanism;
  per-listing last-published epoch makes staleness observable. Same contract as 39's spine.
- Failure mode: hot metro cells make one posting list the hottest object — replica-fan by
  shard-per-city, the skew playbook of [09.cdn](09.cdn/04-deep-dives.md).

**[02.twitter](02.twitter/README.md)** — the deepest *real-system* evidence in the catalog.
The hybrid fanout design in [04 dive 1](02.twitter/04-deep-dives.md) cites 2012 Twitter:
celebrity tweets (≈30 K followers+ users were 9.6% of QPS' weight in the case's own
framing) skipped fanout-on-write and were served on read by query expansion against the
**Earlybird** search index — home timelines became search queries like
`home_timeline:raffi OR user_timeline:celebrity` at >30 K QPS alongside 300 K+ QPS of
timeline reads (Earlybird ICDE 2012, QCon 2012, sources linked in-file). The reusable
lesson: **an inverted index can stand in for a bespoke recent-items store when the query
pattern is "recent posts by this author"** — also referenced as the fallback pattern in
[33.instagram](33.instagram/04-deep-dives.md)'s celebrity merge. In the case's own
architecture ([03](02.twitter/03-architecture.md)) the search/trend pipeline is a separate
DAG off the ingest stream that is allowed to lag.

**[32.spotify](32.spotify/README.md)** — one search surface, two index types, split by
vocabulary per [04 dive 5](32.spotify/04-deep-dives.md): a **typeahead/prefix index** over
the compacted term dictionary (FST/trie-style, RAM-resident on edge nodes — the same
structure family as 45 and 26) answering ~5–10 as-you-type sub-requests per query
([02](32.spotify/02-scale-estimates.md): ~1.7 K/s avg, ~5 K/s peak, each fanning out), and
a **ranked search index** for submitted queries scoring over a big candidate set. Two more
index placement decisions worth keeping: **region gating is a per-document rights bitmap
consulted during retrieval, not a post-filter and not per-market shards** — sharding
indexes per region would grow the fleet 3–4× to save nothing; and **typeahead is never
personalized at the cache layer** (personal signals are cache-hostile; region bitmaps are
cache-safe). Requirement driver: autocomplete < 100 ms p99 ([01](32.spotify/01-requirements.md)).

**[26.google-maps](26.google-maps/README.md)** — the FST/trie case for a *place* corpus.
[04 dive 2](26.google-maps/04-deep-dives.md): over ~100 M POIs (text ≈ 200 GB; FST + postings
≈ 2–3× that ≈ 500 GB per [02](26.google-maps/02-scale-estimates.md)), at ~230 K search QPS
with completions p99 < 200 ms ([01](26.google-maps/01-requirements.md)).
Flat inverted index + query-time edit distance (Option A) cannot afford on-the-fly
edit-distance over 100 M terms in the typing path; chosen: **FST/trie with a materialized
prefix structure** — prefix completion walks the automaton, and fuzzy matching compiles a
Levenshtein automaton against the FST so edit-distance-1/2 lookups cost a bounded walk, not
a scan. Trade-off made explicit: the shared automaton is awkward to edit incrementally, so
shards rebuild in batch (minutes of lag) and RAM per shard is a real budget line. Sharded
by region (S2-cell slices) so a location-carrying query fans out to 1–2 shards —
**term/prefix locality again, like 45, unlike 48**.

**[06.apple-app-store](06.apple-app-store/README.md)** — the small-index end of the scale
line: 2 M app documents, ~50 tokens/doc across locales → an in-memory inverted index in
**tens of GB** ([02](06.apple-app-store/02-scale-estimates.md)), the whole active index on
one big-memory fleet; ~4.6 K avg / 25 K peak search QPS. The design moves
([04 dive 2](06.apple-app-store/04-deep-dives.md)): global immutable doc corpus + **tiny
per-storefront delta postings (app_id → storefront bitset/mask)** updated on catalog events,
because filtering after ranking breaks result counts and pagination. The stated freshness
stance is the catalog's shortest version of the contract: *"an index is allowed to be stale
by minutes, never wrong by structure."*

**[30.facebook-messenger](30.facebook-messenger/README.md)** — search index as a derived
async projection of the mailbox store ([03](30.facebook-messenger/03-architecture.md): built
off mailbox rows, "derived"). The interesting part is the E2EE fork in
[04 dive 5](30.facebook-messenger/04-deep-dives.md): server-side search is a plaintext-era
feature, so under E2EE it moves to a per-device client-side index (feature-preserving but
slow, and it breaks cross-device search). A search index is shown following the *threat and
trust model*, not just the data model — the boundary where a server index is not merely a
worse option but an unavailable one.

**[42.linkedin](42.linkedin/README.md)** — member-directory search as one read surface
among projections: "inverted index; Galene-style Lucene backend" (the real system's index
service)
([03](42.linkedin/03-architecture.md)). Kept deliberately thin in the case — no index
internals — but two boundaries matter to this doc: degree annotation on search results
comes from the graph service's precomputed aggregates, never from the index itself
([04](42.linkedin/04-deep-dives.md)), and profile visibility is enforced once at event
capture, not re-enforced per consumer ([04](42.linkedin/04-deep-dives.md): consumer-side
filters fail by drift; the index is a consumer, never the authority).

**[49.monitoring-metrics](49.monitoring-metrics/README.md)** — an inverted index *built
from scratch for a non-corpus*: label-tag postings, not text. As stated in
[04 dive 3](49.monitoring-metrics/04-deep-dives.md): "the same inverted index every search
engine uses, applied per label" — each label value → sorted series-ID posting lists;
selectors intersect/union postings (`=` binary search, `=~` union, negative matchers =
all-postings minus matched). The case proves the shape transfers to any "find the members
of a set" query path while the *term* semantics do not: no relevance ranking, no phrase or
scoring problem, purely set algebra over IDs. Same posting-list technique, different
product ([CONCEPTS.md](CONCEPTS.md) buckets it as technique + storage).

### Brief mentions

- **[41.web-crawler](41.web-crawler/README.md)** — the boundary case: "The search index,
  ranking, and the query path themselves [are out of scope]. The crawler hands off … to the
  indexing/search stack" ([01](41.web-crawler/01-requirements.md)); 48 picks up exactly at
  that handoff ([48 README](48.google-search/README.md)). Its one "trie" hit
  ([04](41.web-crawler/04-deep-dives.md)) is SimHash Hamming-distance lookup organization
  for near-dup detection — a different index problem, noted in the false-positive list.
- **[05.discord](05.discord/README.md)** — "search indexing" appears once as an async
  fanout-worker side-path ([03](05.discord/03-architecture.md)), and
  [01](05.discord/01-requirements.md) declares full-text search out of scope (a separate
  inverted-index subsystem, a "search service" box only).
- **[08.chatgpt](08.chatgpt/README.md)** — "search indexes" as one storage-tally line
  ([02](08.chatgpt/02-scale-estimates.md)); no index design.
- **[11.zookeeper](11.zookeeper/README.md)** — Solr named only in the list of systems
  ZooKeeper/Chubby coordinated ([README](11.zookeeper/README.md)); a lineage reference,
  not a component (see [ZOOKEEPER.md](ZOOKEEPER.md) for what search engines actually used
  coordination for).
- **[33.instagram](33.instagram/README.md)** — the celebrity-author recent-posts index is
  cited via Twitter's Earlybird pattern ([04](33.instagram/04-deep-dives.md)); no index of
  its own is designed.

### Boundary and false positives

- **[07.blob-store](07.blob-store/README.md)** — "index shard" here, and its
  small-delta/base-snapshot merge design ([04](07.blob-store/04-deep-dives.md)), is a K/V
  *metadata* index for LIST pagination, not an inverted index. Structurally related (delta
  + immutable base ≈ fresh/base tiering) but excluded from the counts.
- **[28.uber](28.uber/README.md)** — the "geo index shard" that owns a driver's cell and
  its CAS ([04](28.uber/04-deep-dives.md)) is a spatial matching structure, not text search.
  Geo *postings inside a text engine* are 36/26's topic; pure geo-matching is not this doc's.
- **[15.ml-ranking-system](15.ml-ranking-system/README.md)** — its index work is vector
  search (ANN, EF-search) and its "search index" refs are by-reference to Earlybird and the
  retriever-ranker funnel. Out of scope here; covered by
  [VECTOR-SEARCH.md](VECTOR-SEARCH.md). The retrieval-then-rank funnel it shares with 48 is
  the one pattern belonging to both docs.

## The portable pattern

Every designed search index in the catalog reduces to one contract, with
[48.google-search](48.google-search/README.md) as the reference implementation and
[45.typeahead](45.typeahead/README.md) as its small-prefix sibling:

1. **The index is a projection; some other store is the truth.** The corpus or core is
   relational/authoritative and the index is a derived read structure fed asynchronously
   (39's event spine, 36's booking DB + update stream, 30's mailbox, 02's ingest DAG, 42,
   06). Rebuildability from that source is a stated requirement, not an accident — the same
   read-model shape as [KAFKA.md](KAFKA.md)'s consumer projections.
2. **Immutable base + small fresh tier; refresh latency is the design knob.** 48's
   minutes-to-hours fresh tier with background promotion; 45's stable index + hours-vs-minutes
   overlay; 26 accepting batch-rebuild lag in exchange for FST economics; 36 delta-push +
   nightly rebuild-as-audit; 06's "stale by minutes, never wrong by structure". The tiering
   is everywhere the same storage move as Kafka segment compaction / LSM merge
   ([KAFKA.md](KAFKA.md), [BIGTABLE.md](BIGTABLE.md)).
3. **Partition along the query's skew, not the corpus's size.** Multi-term broadcast
   queries → document partitioning, pay-per-broadcast (48). Single-term/prefix locality
   (typeahead, place completion, per-region fanout) → term/prefix/region partitioning into
   1–2 shard hits (45, 26, 36). Same axis decision as [CONCEPTS.md](CONCEPTS.md)'s
   partitioning pattern, made from the query shape.
4. **Buy latency by not touching postings you don't have to.** Block-max + WAND (48),
   materialized top-k per prefix (45), Levenshtein automata over an FST (26), bitset
   availability/rights filters evaluated during retrieval rather than post-filters (39, 36,
   06, 32), pre-aggregated facet counts with count metadata (39, 26, 45 top-8). Post-filters
   after ranking are the recognized anti-pattern (39, 06).
5. **Freshness must be bounded *and visible*.** Last-build-age as an SLO (45),
   per-listing snapshot epoch + age alarms (36), tombstone-lag audits (48), stale-on-error
   serving with a previous segment (45). The failure every case fears is the same: a silent
   staleness that looks like correctness.
6. **Cache per layer, keep personalization out of the shared key.** The two-speed TTL story
   (45's seconds-hot vs minutes-stable layers), 32's never-personalized typeahead, 48's
   short-TTL cluster-local results cache with fresh-tier invalidation. See
   [REDIS.md](REDIS.md) for the cache mechanics.

**Boundaries — where a search index is the wrong tool:**

- **The query is really a relational join or a correctness-critical lookup.** The catalog
  keeps truth out of the index deliberately: 39's order/price core is relational because
  transactional edits against denormalized documents produce silently half-updated SKUs
  ([39/04](39.e-commerce/04-deep-dives.md); see [POSTGRES.md](POSTGRES.md)); 42's degree
  annotation comes from the graph service, because an inverted index over members cannot
  answer "path through whom".
- **The query is a hot scalar or K/V range scan.** Object-store LIST (07) and TSDB point
  lookups are index problems, but their "index" is a sorted-map/K-V structure, not postings;
  only 49-style label selectors are genuinely an inverted-index-shaped query, and even there
  there is no relevance problem.
- **The answer's freshness or privacy deadline beats the index's update path.** 30's E2EE
  removes server plaintext entirely, so the index moves to the client — an index over data
  the server must not see is not an index the server can build.
- **The similarity is semantic, not lexical.** Vector databases / ANN indexes solve that;
  this catalog keeps them separate — [VECTOR-SEARCH.md](VECTOR-SEARCH.md).
- **Coordination.** Nowhere in the catalog does a search index coordinate anything — that
  is [ZOOKEEPER.md](ZOOKEEPER.md)'s job (11; Solr appears there only as a coordinated
  client).

# 04 — Deep dives

## 1. Inverted index layout: posting lists, compression, skip pointers

The inverted index maps each term to the list of documents containing it, plus — for
phrase queries — the word positions. At ~1e14 postings ([02 §3](02-scale-estimates.md))
the layout question is not "can we store it" but "can a query read the slice it needs
fast enough and cheaply enough". Posting lists are sorted by doc-ID, which is the whole
trick: sorted gaps are what make both compression and intersection cheap.

**Option A — uncompressed arrays**:

- wins: trivial to implement, O(1) random access into a posting, simplest skip logic.
- loses: 8 B per posting (doc-ID + score + position slots) ≈ 800 TB of postings —
  ~2× the compressed postings line in [02 §3](02-scale-estimates.md) (300–400 TB), on
  SSD, so every query is also I/O-bound at ~2× the bandwidth. Unaffordable.

**Option B — delta (gap) encoding + variable-byte compression**:

- wins: doc-IDs are sorted, so the *gap* between consecutive doc-IDs is small for
  frequent terms (a term in 1% of 50 M docs has a mean gap of ~100 → 1–2 B) and postings
  shrink to ~3–4 B amortized ([02 §3](02-scale-estimates.md)); decompression is a branchy
  scalar loop that runs at GB/s. Intersection cost also collapses: a gap-encoded term
  with doc-frequency 1e6 costs ~1 MB, not 8 MB.
- loses: random access inside a list is gone — you must decompress from a chunk boundary;
  long-tail terms (df=3) don't compress well and pay header overhead per block.

**Option C — B, plus chunking with skip pointers + block-max metadata (chosen)**:

- postings are written in fixed blocks (e.g. 128 postings), each block
  compressed independently, with a **skip pointer** (pointer to block N's last doc-ID)
  at the head of the list. Intersection and WAND traversal skip whole blocks instead of
  decompressing one posting at a time.
- each block also stores the block's **max impact score** (max static-quality-scored
  contribution in the block) so deep dive 4's early termination can discard an entire
  block from a single comparison.
- wins: B's compression, plus sublinear seeks in long lists, plus O(1) block skips — the
  difference between reading 1e6 postings and reading ~2% of them (~50× less posting-list
  work; [02 §6](02-scale-estimates.md)'s cost arithmetic starts here).
- loses: more encoding complexity, per-block metadata overhead, and a harder failure
  mode — a bug in one encoder version silently corrupts results for a term set, which is
  why index format versions are rolled out like code (NFR7 discipline).

**Chosen: C.** A is unaffordable at web scale; B without block-max metadata cannot feed
early termination, and [02 §6](02-scale-estimates.md) shows early termination is what the
whole system's cost model stands on.

**Sub-choice — doc-ID assignment**: sequential per shard, assigned at indexing time in
crawl/importance order, *not* global and *not* hash-ordered. Wins: small gaps everywhere,
and ordering by importance means high-quality docs cluster at list heads where the
traversal sees them first (impact ordering). Loses: doc-IDs are not stable across shard
rebuilds — every doc-ID consumers see is shard-local and rebuild-scoped; the URL table
([41](../41.web-crawler/03-architecture.md)) remains the stable identity.

**Failure modes**:
- Block-max metadata wrong (bug or partial rewrite) → early termination skips blocks
  that contained top results → query quality regression with *no error signal* — the
  results are plausible, just worse. Mitigation: golden-query regression suites run per
  index-format release; the index is immutable per release.
- One hot term (breaking-news word) gets a posting list appending fast in the fresh
  tier while base has a stale dense list — merge timing becomes the hot term's
  performance bound (deep dive 3's failure modes).
- Dictionary misses: vocabulary ~500 M terms — a shard's own term dictionary fits RAM
  (~25 MB of it, [02 §4](02-scale-estimates.md)), but per-term pointers point into SSD
  postings; a mis-sized block cache turns p50 into p99 under cache-flush storms
  (NFR2 sizing rule from [03 §5](03-architecture.md)).

## 2. Index sharding: document-partitioned vs term-partitioned

The 100 B-doc index must be spread over machines. Two axes exist, and they behave
oppositely at query time.

**Option A — term-partitioned** (each shard owns a disjoint set of terms, with their
full global posting lists):

- wins: a single-term query touches one shard — a huge fanout win for that query shape,
  and per-term working sets stay local.
- loses: every multi-term query needs a *distributed join* across the shards holding
  each term — you must ship intersecting doc-ID sets across the network per term, and
  the query is as slow as the slowest shard holding a common term (a term like "the"
  has a global posting list that no single machine holds cheaply — its block-max
  compression per shard was free in Option B; globally it's a shard of its own).
  Also: hot terms concentrate on few shards; load skews with query distribution, and
  rebalancing moves posting lists, not documents.

**Option B — document-partitioned** (each shard owns disjoint documents, with *complete*
posting lists for its docs):

- wins: any query — one term, ten terms, phrases — is answered *within* a shard by
  local intersection, then a cheap top-K merge across shards. No distributed join. Shards
  are identical and interchangeable (operationally simple at NFR3 scale: 2,000 nodes,
  replicas, no skew). The full posting list of "the" is fragmented into 2,000
  shard-local pieces, each small and block-compressible.
- loses: every query is a **broadcast** to all 2,000 shards ([02 §6](02-scale-estimates.md)'s
  arithmetic) — cost per query ∝ shard count, and tail latency is bounded by the
  slowest *of* 2,000 responses (deep dive 4's whole problem statement).

**Chosen: B.** The deciding factor is not cost, it is correctness-under-terms-mix: real
queries are multi-term, and A's distributed join turns *every* such query into the
network-heavy case, while B keeps all join work local and only the top-K merge
distributed. A is the better choice for specialized single-term workloads (autocomplete
prefix indexes, [45.typeahead](../45.typeahead/README.md)'s per-prefix serving) — which
is why that case's system is term/prefix-partitioned and this one is not.

**Chosen shard count**: 2,000 × 50 M docs ([02 §4](02-scale-estimates.md)) — a middle
point between per-shard state growth (fewer shards → SSD-bound reads per query) and
fanout tax (more shards → more CPU per query, worse tail). It is a cost knob first and a
latency knob second; it is not a "capacity" knob: adding shards grows cost per query
linearly rather than amortizing it.

**Failure modes**:
- Broadcast load is flat but not free: every query pays 2,000×; a 5% per-shard CPU
  regression shows up as a 5% fleet-wide cost regression — shard-level performance
  monitoring is where the fleet's money is.
- Replicas are for *availability*, not fanout reduction: a query does not split across
  replicas (it needs each shard's slice of documents), so a replica set is sized so
  2,000 shard replicas can absorb a cache-flush ([03 §5](03-architecture.md)), not for
  2× QPS at steady state.
- Hot shards: no shard should be hotter than another if documents are randomly
  assigned; if the assignment ever keys on topic/language, query skew follows —
  keep the doc→shard map hash-uniform.

## 3. Tiered index for freshness: small mutable fresh tier vs merged base

New and updated documents arrive at ~25–35 k docs/s ([02 §9](02-scale-estimates.md)) but
the base index is ~100 B docs whose segments are immutable (deep dive 1's compression
assumes written-once data). Where does a document live during the minutes-to-days before
it is part of the base? This is [Case 41's deep dive 6](../41.web-crawler/04-deep-dives.md)
revisited from the *serve* side: 41 decides *which* pages are re-fetched; this section
decides *when the index can be queried with* what was fetched.

**Option A — update base index in place** (delete old posting, insert new):

- wins: one copy of everything, no tier logic, freshness immediate.
- loses: base index immutability exists *for* deep dive 1's compressed block layout and
  merge economics; in-place updates on 2,000 shards mean per-shard RAM-based pending
  update structures, write contention on hot lists, and update amplification — one
  document changed touches every term it contains. This is the "updates are the enemy of
  the read-optimized index" trade, the same shape as LSM compaction in
  [12.bigtable](../12.bigtable/03-architecture.md).

**Option B — tiny always-merged fresh tier, no base promotion**:

- wins: simplicity — a small in-RAM tier queried beside base forever; documents never
  need to "move".
- loses: the fresh tier grows without bound (a month's 2–3 B docs at 1 KB/doc
  uncompressed ≈ 2–3 TB of poor-compression content serving 20% of the corpus's traffic
  value), and stale documents shadow base copies forever — the base index never gets
  its update, so the *newest ranking features* live in the tier with the *worst*
  ranking infrastructure. Long-term, the freshness SLA (NFR4) silently rots.

**Option C — fresh tier + background promotion to base (chosen)**:

- fresh tier holds the last minutes-to-hours of indexed documents as small immutable
  segments with light compression; queried *in parallel* with base shards
  ([03 diagram](03-architecture.md)), shadowing any stale base copy of the same
  document (doc-key dedup at merge time).
- a background promotion merges fresh segments together, then into base's
  newest-generation segments, then tombstones (deletes) apply — *at promotion*, not
  eagerly (NFR6's eventual-consistency window is exactly this interval).
- wins: immediate-ish freshness in the tier (Caffeine-style, publicly described for
  Google's 2010 index re-architecture), deep dive 1 economics preserved in base, and the
  tier is small enough that its poor compression doesn't matter ([02 §3](02-scale-estimates.md)
  prices it at 5–10 TB).
- loses: two lookup paths per query; merge storms degrade p99 on specific shards for a
  bounded window; dedup between "same doc in both tiers" adds a correctness surface
  (if the freshest version loses the tie, results are *newer-ordered but wrong*).

**Chosen: C.** A ignores the immutable-index economics B relies on; B loses freshness
under growth. C buys tiering cost (duplicate doc copies in-transit) in exchange for
keeping the fast-changing part small — the same storage-tiering move as
[Kafka's segment compaction](../13.kafka/03-architecture.md) and the hot/warm layering
reasoned through in [07.blob-store](../07.blob-store/02-scale-estimates.md).

**Failure modes**:
- Merge storm: a big promotion lands on a shard that's already hot → that shard's p99
  triples for 10–15 minutes → deep dive 4's partial-result path absorbs it if the
  deadline system works, *and* merge scheduling is admission-controlled (a promotion
  waits for shard headroom rather than preempting query traffic).
- Fresh tier over-promotion failure (a promotion crash): tier content is in segments —
  replay from the crawler stream rebuilds it ([13.kafka](../13.kafka/04-deep-dives.md));
  no data loss, only an SLA slip.
- Shadowing bug (fresh copy of a doc is older than the base copy): the doc-key vs
  content-hash check at merge time is what prevents "fresh wins on tie *regardless of
  content*"; the check must compare content version, not arrival order (a re-crawl of
  an unchanged page must not shadow the base entry with a worse-ranked copy).
- Delete lag: a page deleted upstream stays queryable until both tiers process the
  tombstone — bounded by promotion cadence; users see it in the freshness window
  (NFR4), and it must not silently become a permanent zombie (a weekly audit passes a
  random doc sample against the crawler stream's tombstone log).

## 4. Query fanout and tail latency: WAND, hedging, partial results

p50 is a per-shard property; p99 is a *fanout* property. A query waits on 2,000 shards:
if a per-shard p99 is 30 ms, the aggregate p99 of the *max* of 2,000 responses is much
worse than 30 ms — the tail compounds over broadcast width
(Barroso et al.'s "The Tail at Scale" describes exactly this). Two mechanisms fight it:
*spend less CPU per query* (pruning) and *absorb stragglers* (deadline + redundancy).

**Option A — full scoring with no early termination**:

- wins: exact top-K everywhere; no quality loss from pruning; no block-max metadata to
  maintain in deep dive 1.
- loses: [02 §6](02-scale-estimates.md)'s arithmetic — ~100 s CPU per query, 3 M cores
  at fleet scale. Unaffordable by orders of magnitude. The interesting part is *why* it's
  unaffordable: most of the work scores documents that would never reach the top 10
  because their static quality is low — wasted effort by construction.

**Option B — static-score-first posting order, no pruning** (best-first traversal):

- wins: the first documents visited are the highest-quality ones, so a truncated scan is
  *approximately* right even without termination checks; per-shard state stays simple.
- loses: without a score bound you still traverse to exhaustion; "approximately right"
  is unmeasurable without a termination criterion, so cost remains unbounded.

**Option C — WAND-style pruning over block-max metadata (chosen)**:

- as the traversal walks best-first, maintain a running *upper bound* on the max
  achievable score of the remaining candidates (from deep dive 1's block-max scores);
  once the current top-K heap's minimum exceeds that bound, the rest of the list can
  provably contain nothing better — stop.
- per-terms: frequent terms (stopwords, "the") contribute a known, cheap max; the rare
  term is the driver of the heap, so the traversal effectively ignores the common term's
  bulk.
- wins: [02 §6](02-scale-estimates.md) targets ~1–5% of postings examined (~50× less
  posting-list work, ~15–25× less query CPU fleet-wide), with a *provable*
  no-worse-than-K guarantee on the returned top-K (the
  pruning is lossless for top-K, though the *ranked order below K* is approximate).
- loses: the guarantee is per (term, block) — the pruning is only as good as block-max
  metadata, and the runtime cost grows with the *number of terms* in a query; per-shard
  latency becomes input-dependent (a pathological 30-term query is slower than p50 by
  far, and the FE deadline system must catch it).

**Chosen: C.** The lossless-top-K property is what makes it safe to base the fleet's
cost model on; B's "approximately right" has no stopping criterion and no cost bound.

**Handling the 2,000-wide tail — the second half of the dive:**

- **Per-stage deadlines** ([03 §3.4](03-architecture.md)): every fanout has a budget;
  the FE does not wait past it. If 1,995 of 2,000 shards responded, the merge proceeds
  with partial results (NFR2) rather than waiting for the tail. Quality degrades
  measurably but *bounded* — a few docs' shard was missed, not the query.
- **Hedged requests**: after a short delay (e.g. p95 of per-shard latency), the FE sends
  a duplicate request for the same shard slice to a replica. Costs ~5–10% extra QPS
  (only the slow minority is hedged), cuts p99 substantially — publicly described in
  "The Tail at Scale" as one of the highest-yield mitigations. Trade: the duplicate work
  is pure cost, and it amplifies load exactly when a shard is struggling — hedge only
  when the shard's health metric says it's a straggler, not a meltdown.
- **Result caching as a tail defense**: a cached query skips the fanout entirely, so the
  heavy tail of *cost per query* (not just latency) drops out for ~25% of traffic
  ([02 §8](02-scale-estimates.md)).

**Chosen: deadlines + hedging on top of C.** Each part is load-bearing: C bounds the
median, deadlines bound the tail, hedges convert "slow shard" into "extra request"
instead of "slow user".

**Failure modes**:
- Stragglers from *one shard's* GC pause or merge storm (deep dive 3) → the hedged
  request lands on a replica; if the *replica set* is the problem, only partial results
  protect the user. Both paths must be exercised in tests, not just the first.
- Over-hedging: hedged QPS doubles during a partial outage; the hedge budget must be
  capped fleet-wide (a few % of QPS) or mitigation turns into overload.
- Deadline too tight: shards legitimately slow because of a *real* load spike get
  dropped from every query → their documents systematically missing from results while
  the system "looks" healthy (p99 is fine!). Mitigation: shard-inclusion rate per
  shard is the health metric, not p99 alone — the same trap as a load balancer that
  health-checks after the pool has already drained.

## 5. Ranking pipeline: retrieval scores vs ML ranking under a latency budget

Retrieval (deep dive 4) returns a handful of top-K candidates per shard — ~5 k in
aggregate across the 2,000 shards ([03 §4](03-architecture.md)) —
each with cheap index-time scores; the user gets ten. Ranking is the funnel in between,
and every stage is a decision about *what the expensive model will not see*.

**Option A — score all candidates with the heavy model directly**:

- wins: best possible model everywhere; one stage; no coarse ranker to keep in sync.
- loses: the heavy model is ~0.5–2 s CPU per query ([02 §6](02-scale-estimates.md))
  *at 200 candidates*; at 5,000 candidates it is ~25× that (12–50 s/query), which
  inflates per-query CPU 3–8× over the ~6–7 s baseline, and even that is only the *feature fetch* cost at 5 k docs —
  page features are not indexed, they are read from the page store, and reading
  features for 5 k docs per query is an I/O problem before it's an inference problem.

**Option B — static scores only (no ML ranker)**:

- wins: free; fully deterministic; trivially debuggable.
- loses: static quality is query-independent — a great page about a different subject
  beats a perfect match on a weak page; text-frequency signals (BM25-class) are
  gameable by spam, and they cannot rank by intent at all (a navigational query and a
  how-to query get the same ordering logic). Quality plateaus; the product is
  "a keyword matcher".

**Option C — staged funnel (chosen)**: retrieval top-K → coarse ranker (cheap model on
*indexed* features: match quality, static score, term-proximity, freshness delta) keeps
~5 k → 200; heavy ranker (neural, query-doc cross-attention, on *fetched* features)
orders the 200; a light post-rank reorders (personalization, dedup of near-identical
results on the same page-1).

- wins: A's accuracy where it matters (top 200) at ~B's cost for the long tail; the
  funnel is the same shape as the [Ranking System case](../15.ml-ranking-system/README.md)
  and for the same reason.
- loses: the two-stage structure has a training-skew risk — the heavy model only sees
  documents the coarse model *selected*, so if the coarse model has a blind spot, no
  amount of heavy-model training fixes it ([14.ml-production-system](../14.ml-production-system/README.md)
  covers serving/training skew); and the pipeline's *stages* have a versioned contract
  (NFR7) that has to be maintained across re-trainings.

**Chosen: C.** The trade is explicit: we give up best-possible ranking on rank-400+
documents (they are invisible anyway) and accept coarse-model bias in what the heavy
model sees, in exchange for a fleet cost the system can actually afford.

**Feature freshness — a sub-problem worth its own paragraphs.** The ranker's features
describe documents, and documents change ([41's whole premise](../41.web-crawler/README.md));
a feature vector computed for the base-index copy is stale the moment the fresh tier
carries a new version. Options: (a) compute features at index time only — wins cheap
query time, loses: fresh docs have *no* features (unrankable against established pages,
so newest never wins, killing NFR4's purpose); (b) fetch all features at query time from
the page store — wins always-fresh signals, loses: an I/O tax on every query, the thing
deep dive 4 fights; (c) **index-time features for base, query-time for fresh-tier and
top-200 docs only (chosen)** — the coarse ranker uses cheap indexed features which are
recomputed at promotion (deep dive 3 cadence), and the heavy ranker fetches the ~200
docs' page features at query time anyway ([02 §7](02-scale-estimates.md) prices it at
~20–50 ms), so feature freshness is guaranteed exactly where the ranking decision is
actually made. Loses: some coarse-ranker signals are one promotion-cycle stale — a
bounded, accepted staleness.

**Failure modes**:
- Coarse-ranker blind spot (a class of documents it systematically under-scores): the
  heavy model never sees them; detected only by *offline* evaluation on held-out
  queries, not by online metrics — hence the golden-query regression suites
  (deep dive 1's discipline) plus periodic full-corpus re-scoring audits on samples.
- Feature skew between training and serving (feature computed one way offline, another
  online) — the standard
  [ML production failure](../14.ml-production-system/04-deep-dives.md); rankers get a
  canary rollout + shadow scoring (score with new model, serve old, diff).
- Fresh-doc cold start: a brand-new document has no interaction features at all — it
  competes on content signals only, briefly; deliberate, and a known spam magnet
  (new pages rank on unproven signals). Mitigation is a content-quality prior, not a
  freshness penalty.
- Ranker regression after a model rollout: A/B metrics catch most; the *revert path*
  must be minutes (NFR7), which requires the coarse and heavy models to be independently
  deployable, not one monolith.

## 6. Snippet generation and the query cache

The result page is not just ten links — the ~2–3 lines under each are generated to
match *this* query. Two separate caches sit here: per-(query, doc) snippets, and full
results per query. Both trade freshness for hit rate, and their hit rates feed back
into [02 §6](02-scale-estimates.md)'s fleet arithmetic.

**Snippet generation — where the text comes from:**

**Option A — pre-computed snippets (index-time)**:

- wins: zero query-time cost; snippet fetch is a key-value read.
- loses: a snippet must highlight query terms; the query is unknown at index time, so A
  can only produce a generic snippet — the first N "good" sentences — which demonstrably
  mismatches most queries (a page whose best passage for "how to X" is not its intro).
  Quality cost, not just cost.

**Option B — generate at query time for top results only (chosen)**: after the heavy
ranker fixes the ~10–30 docs, read a few KB of each doc from the page store
([02 §7](02-scale-estimates.md): ~5–15 ms), score candidate sentences by
query-term coverage + proximity, highlight. Wins: query-matched snippets at bounded cost
(only ~30 docs, not 5,000 candidates); loses: page-store read on the critical path —
absorbed by snippet caching (below) and by keeping the page store's hot slice in RAM;
a slow page store turns p50 into p99, so it gets the same deadline/degradation
treatment as index shards (missing snippet = title + URL, not a stalled response).

**Chosen: B**, because A's quality cost lands on every single result page — the product.
Snippet generation for 30 docs is cheap; a *wrong* snippet for those 30 docs is the
most visible bug in the system.

**Query results cache — the hit-rate trade-off:**

**Option A — long-TTL results cache (hours), maximize hit rate**:

- wins: hit rate 40%+ (Zipfian tail is deep — [02 §8](02-scale-estimates.md)); shards
  see ~40% less QPS (60% of queries miss vs 75%); burst events cost nearly nothing.
- loses: stale results are *visible* — a breaking-news query served for an hour shows
  yesterday's world on the most user-scrutinized page in the system, and the fresh-tier
  documents sit in the index unvisited. Freshness (NFR4) dies by cache, not by index.

**Option B — no results cache**:

- wins: results always as fresh as the index; no invalidation problem at all.
- loses: [02 §8](02-scale-estimates.md)'s cache-free shard QPS (~1.3× average load,
  ~2× at peak, [03 §5](03-architecture.md)), and hot-query bursts
  (100× single-query spikes) become shard-level incidents.

**Option C — cluster-local cache with short TTL + freshness-aware invalidation
(chosen)**: TTL minutes (not hours); entries for the *fresh tier* are invalidated on
promotion events that touch their documents (or simply never cached at the fresh-doc
level — cache only base-tier-only results); headroom sized for a full cache flush
([03 §5](03-architecture.md)). Wins: ~20–30% hit rate ([02 §8](02-scale-estimates.md)),
which is where the cost arithmetic was computed, minus the burst-flatlining benefit;
loses: hit rate lower than A's, and invalidation logic is one more correctness surface
(a stale cache entry is indistinguishable from a stale index, so debugging "why is this
query wrong" must check both).

**Chosen: C.** The product promise in a search engine is not "results are fresh", it is
"results are fresh *when the world's news is fresh*" — exactly the traffic a long-TTL
cache would freeze. A 20–30% hit rate is enough to make the fleet arithmetic work
without sacrificing that promise; A's extra 10–20% hit rate is not worth shipping the
one failure users actually notice.

**Caching misses and hot queries**: a single hot query (news event) is served with
nearly 100% cache hit after its first execution; the residual risk is the *miss storm*
at the start of the burst — thousands of concurrent identical misses. Mitigation:
request coalescing — concurrent identical cache misses collapse into one fanout and
share the response, the same single-flight pattern as
[10.distributed-cache](../10.distributed-cache/04-deep-dives.md)'s thundering-herd
protection.

**Failure modes**:
- Cache stampede on TTL expiry (not just flush): if 1 M cached entries share one expiry
  boundary, that instant looks like a cache death. Mitigation: jittered TTLs, staggered
  writes; expiry times must never be synchronized across entries.
- Personalization poisoning the cache key: if user signals leak into the cache key, hit
  rate collapses; keep personalization strictly as a *post-cache* reordering of the
  shared base result ([03 §3.7](03-architecture.md)).
- Snippet cache serving a snippet whose underlying text was updated (doc re-crawled):
  snippet cache keyed on (query, doc-version) or invalidated on promotion — otherwise
  the user sees a passage that the *current* page no longer contains, the worst kind of
  staleness because it is directly visible as text.
- Empty-result caching: a query that legitimately returns zero results because the
  index *was* missing the fresh doc must not be cached for the full TTL — a cached
  "no results" hides the fresh tier's fix for TTL minutes. Mitigation: short TTL
  (seconds) for zero-result entries or a "fresh-doc arrived since cache write" check.

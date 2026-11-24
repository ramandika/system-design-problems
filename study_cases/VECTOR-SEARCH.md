# Vector search across the case catalog

Where embeddings and approximate nearest-neighbor (ANN) retrieval appear in the 50 study
cases, what role they play, and how deep each case goes. Companion to [CONCEPTS.md](CONCEPTS.md) —
note that vector/embedding retrieval was **not** one of the concept-sweep unions there, so the
counts in this doc are mention-based first picks and are **not comparable** to the CONCEPTS.md
union ratios. Sibling doc: [SEARCH-INDEX.md](SEARCH-INDEX.md) covers lexical/full-text indexes
(the two catalogs deliberately do not overlap; its "semantic similarity is not lexical" boundary
points here).

Headline: **in this catalog, vector search is recommendation retrieval, not document search.**
No case designs semantic search or RAG-style document retrieval — the ANN appears only as the
cheap first stage of feed/recommendation funnels. Where it is designed seriously, the catalog
consistently prefers *sharded exact/IVF brute-force over an HNSW-style graph index*, because the
embedding table re-trains on a schedule and a graph index rots silently under drift. Rejections
are as informative as the adoptions: a semantic cache is rejected for correctness ([16.dalle](16.dalle/README.md)),
ANN is skipped because the catalog is small ([31.netflix](31.netflix/README.md)), and semantic search
is scoped out as someone else's box ([05.discord](05.discord/README.md), [10.distributed-cache](10.distributed-cache/README.md)).

The "vector" substring is heavily polluted, so every hit was verified in context. Case files
matching the substring: 13, of which **11 are false positives in every hit** (maps/rides vector
*tiles*, "attack/abuse/delivery vector", PromQL *vector matching*, cart *update vector*, search
*term vectors*). One (10.distributed-cache) is mixed: a false-positive sizing row plus a genuine
scope rejection. One (15.ml-ranking-system) is genuine. Word-level filtering decisions are in
the false-positive dictionary at the end.

- Cases with genuine vector/embedding-similarity usage: **11 / 50**
- Vector retrieval as a designed first-class component: **3** (15 = the case itself; 35, 33)
- Brief mention / artifact or cache sizing shorthand only: **4** (01, 08, 14, 17)
- Explicitly rejected, scoped out, or made unnecessary — each with a stated reason: **4** (16, 31, 05, 10)
- False-positive `vector` mentions filtered out entirely: **11 case files** (26, 22, 36, 49, 48, 40, 39, 37, 34, 21, 09)

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [15.ml-ranking-system](15.ml-ranking-system/README.md) | The case itself: ANN retrieval over item embeddings is stage 1 of the funnel, designed end-to-end | Full case |
| [35.tiktok](35.tiktok/README.md) | ANN over video embeddings + interest index over user embeddings; synchronous publish-hook upsert | Designed component |
| [33.instagram](33.instagram/README.md) | Explore/Reels retrieval stage: embedding similarity + graph narrows billions to ~1–2K | Designed component |
| [14.ml-production-system](14.ml-production-system/README.md) | Embedding tables as model artifacts to size, ship and load — no search over them | Brief (adjacent) |
| [17.elevenlabs](17.elevenlabs/README.md) | Speaker embeddings as voice identity objects, not similarity search; one estimated similarity screen | Brief (alternate use) |
| [08.chatgpt](08.chatgpt/README.md) | "RAG chunks" named only as prompt/prefix-cache content; retrieval never designed | Brief |
| [01.youtube](01.youtube/README.md) | Two-stage funnel designed; two-tower retrieval named as an era-dated evolution only | Brief |
| [16.dalle](16.dalle/README.md) | Semantic cache explicitly rejected; exact-key caching chosen | Rejected (trade-off) |
| [31.netflix](31.netflix/README.md) | ANN unnecessary: candidate pool is the whole catalog, small enough to rank directly | Rejected (by scale) |
| [05.discord](05.discord/README.md) | Full-text/semantic search scoped out as a separate search-service box | Scoped out |
| [10.distributed-cache](10.distributed-cache/README.md) | Vector/embedding search explicitly out of scope; "feature vectors" appear only as cache values | Scoped out |

## Per-case notes

### The case itself

**[15.ml-ranking-system](15.ml-ranking-system/README.md)** — the only case where the ANN is a
load-bearing design object. Retrieval is sized first: [02-scale-estimates.md](15.ml-ranking-system/02-scale-estimates.md)
runs the corpus arithmetic that everything else leans on — 500M items × 128 dims × 2 B (fp16) =
**~128 GB**, so the whole embedding table fits in RAM across a tens-of-nodes ANN fleet, and ~2K
neighbors of a 200 K-QPS lookup in ~10 ms is *the cheap stage* of a ~100 ms budget
(200 K × 10 ms = 2,000 core-seconds/s → a few thousand memory-bound ANN cores, estimated).
[04-deep-dives.md](15.ml-ranking-system/04-deep-dives.md), dive 2, stages the trade-off:

- **Option A — HNSW wins** low query latency and recall per GB plus incremental insertion;
  **loses** graph links on top of vectors, an EF-search parameter that needs re-tuning as data
  shifts, and deletions/retrain drift that *quietly rot the graph* (recall drops for hours
  before anyone notices).
- **Option B — sharded brute-force / IVF sweep** (two-tower embeddings, ~dozens of RAM nodes)
  wins exact, controllable recall and a structure that is trivially re-derivable from raw
  vectors when embeddings retrain — no incremental-structure debt; loses per-query efficiency.
- **Chosen: B**, because the table re-trains daily and *quietly stale recall corrupts the
  training loop's view of what users could see* — 2× more RAM is the cheaper failure. HNSW is
  admitted only where incremental insert dominates (per-item "more like this" rails).
  Ship items/user towers as one atomic version (embedding retrain changes the vector space
  mid-flight otherwise); cache neighbor lists per hot item for "related"-rail hot shards.

Also note where the catalog places the *training* contract: retrieval should be trained on
labels the ranker would rank top-50 (Covington et al., RecSys 2016), not merely items watched.

### Designed components

**[35.tiktok](35.tiktok/README.md)** — the recall plane is vector-based by construction:
[03-architecture.md](35.tiktok/03-architecture.md) lists "ANN over video embeddings" plus an
interest index over user embeddings among parallel retrievers, capped at ~10K candidates per
request ([02-scale-estimates.md](35.tiktok/02-scale-estimates.md) funnel table). The key vector
decision is *freshness*: [04-deep-dives.md](35.tiktok/04-deep-dives.md) names **stale recall index**
as a failure mode — a published video missing from the ANN is invisible — and the publish hook
upserts the video embedding *synchronously* with a small hard-coded exploration injection, so
a 30-second-old video is recallable. The same embedding is reused as a **content prior for
cold start** (dive 5): a new video's embedding (computed in the transcode pipeline) against
user interest clusters decides the guaranteed impression burst before any engagement exists.

**[33.instagram](33.instagram/README.md)** — the Explore/Reels deep dive
([04-deep-dives.md](33.instagram/04-deep-dives.md)) picks two-stage retrieval-and-rank over
full-rank scoring and over prebuilt per-user feeds. Stage 1 is *cheap approximate retrieval* —
embedding similarity alongside creator-graph expansion and engagement indexes — narrowing
billions to ~1–2K in milliseconds; the stated cost is that the retrieval index is "a hard piece
of infra" with its own freshness and consistency requirements, and that content-level
moderation must gate retrieval (a private or violating item must be invisible at stage 1, not
caught later). Architecturally ([03-architecture.md](33.instagram/03-architecture.md)), the home
feed and Explore differ *only in the retrieval stage* — one ranker, one feature store, one
signal loop.

### Brief mentions

- **[14.ml-production-system](14.ml-production-system/README.md)** — embeddings appear only as
  model *artifacts*: hot ranker ~0.5–1 GB with "embedding tables dominate"
  ([02-scale-estimates.md](14.ml-production-system/02-scale-estimates.md)), a promotion-gate
  exception for embedding *dimension* changes, and pre-flight artifact load against a canary
  node to catch an OOM on a fat embedding table
  ([04-deep-dives.md](14.ml-production-system/04-deep-dives.md)). No ANN anywhere; this case
  owns the shipping discipline that 15 and 35 lean on.
- **[17.elevenlabs](17.elevenlabs/README.md)** — embeddings used the other way: a speaker
  *embedding* is a KB-sized voice-identity object that makes voice cloning O(seconds)
  ([04-deep-dives.md](17.elevenlabs/04-deep-dives.md), the "instant clone" tiering trade-off).
  Genuine similarity-search content is one explicitly *estimated* design point: an
  embedding-similarity check of a requester's conditioning audio against a protected-voice
  index runs in tens of ms, negligible next to inference. No index design.
- **[08.chatgpt](08.chatgpt/README.md)** — "RAG chunks" appears once
  ([03-architecture.md](08.chatgpt/03-architecture.md)) as one of the prompt/prefix-cache
  tenants. Nothing retrieves; the case designs serving, not semantic retrieval.
- **[01.youtube](01.youtube/README.md)** — the two-stage (candidate generation → ranking)
  funnel from the Covington et al. paper is the canonical architecture
  ([04-deep-dives.md](01.youtube/04-deep-dives.md)), but the retrieval stage is left as "modern
  pipelines add retrieval evolutions (e.g., two-tower models)" — the shape is designed, the
  embedding indexes are not.

### Rejected with a stated reason

**[16.dalle](16.dalle/README.md)** — the cleanest trade-off in the catalog
([04-deep-dives.md](16.dalle/04-deep-dives.md)): Option A exact-request cache vs Option B
*semantic cache* (embed prompts, reuse results for "similar" prompts). Option B wins far more
hits and loses the correctness core — two similar prompts *should* produce different images,
and silently returning another user's variant breaks the product promise. Chosen: A, exact
keys only. Vector search is rejected here because the *unit of reuse it implies is wrong*, not
because the technology is immature.

**[31.netflix](31.netflix/README.md)** — ANN never comes up, deliberately: the candidate pool
is the whole catalog and the catalog is finite ("deep but finite"), so a strong ranker is
affordable at every request ([04-deep-dives.md](31.netflix/04-deep-dives.md)). The case states
the transfer boundary itself: the infra lessons of 15 apply, but candidate scale differs by
three orders of magnitude (1–2K vs 500M). Exact scan beats ANN when the corpus is small — the
same conclusion 15 reaches for a different reason (freshness, not size).

**[05.discord](05.discord/README.md)** and **[10.distributed-cache](10.distributed-cache/README.md)**
— both scope vector/semantic search out in their "Explicitly OUT of scope" sections: Discord
assumes a separate inverted-index service and designs only the interface
([01-requirements.md](05.discord/01-requirements.md)); the cache tier refuses vector/embedding
search and edge caching wholesale ([01-requirements.md](10.distributed-cache/01-requirements.md)).
Neither states a trade-off — they mark ownership boundaries. Note 10's sizing row "feature
vectors, ~10 KB, 1 h TTL" is *feature-store* caching, not ANN.

## The portable pattern

Across the catalog, vector retrieval is one shape: **the cheap recall stage of a two-stage
funnel**. The reusable contracts:

1. **The corpus-fits-in-RAM test comes first.** 15's arithmetic (500M × 128 dim fp16 = 128 GB)
   is the pivot: when total vectors fit across a few dozen RAM nodes, exact/IVF sharded scan
   is viable and its properties (exact recall, retrain-safe, cheap rollback) beat a graph
   index. Reach for HNSW-style indexes when the corpus is too big for that *or* incremental
   insert dominates.
2. **Prefer scheduled staleness over silent rot.** The catalog's consistent argument: a graph
   index rots quietly under embedding retrain; a sharded sweep has a known freshness clock.
   In recommendation, silently stale recall corrupts the training loop — the worst failure
   mode, worse than doubling memory.
3. **Freshness is a write-path problem.** 35 solves the "new item invisible until next index
   build" failure with a synchronous publish-hook upsert plus exploration injection; 15 keeps
   the fresh-item source outside the ANN entirely.
4. **Embeddings double as cold-start priors.** 35 uses the fresh video embedding against user
   interest clusters to allocate the first impression burst before engagement exists.
5. **Moderation gates retrieval, not ranking.** 33's rule: anything reachable by stage 1 must
   already be eligibility-checked, because candidates from a global index bypass per-owner
   scoping.
6. **Embeddings are also just artifacts.** 14 treats embedding tables as weight-like versioned
   blobs (dimension changes need sign-off; fat tables OOM on load) — the shipping discipline
   that makes pattern 3 possible.

**Boundaries:**

- **No RAG / semantic document search case exists.** 48.google-search stays lexical ("term
  vectors" are an inverted-index structure, a flagged false positive); 45.typeahead is lexical;
  08.chatgpt names RAG only as cache content. If a RAG case is added later, pgvector-style
  Postgres-as-vector-store would be its natural component note — it belongs in
  [POSTGRES.md](POSTGRES.md), which currently has no vector content (`pgvector`: zero hits).
- **Candidate pools without vectors are the majority.** 42.linkedin, 34.reddit (PYMK,
  candidate-pool caches) and 01.youtube's funnel build candidates from graph, engagement and
  trending structures only; 31.netflix ranks the whole catalog directly. Vectors are one recall
  source among several even where present (35 runs ANN *alongside* follow graph, trending,
  sound/hashtag indexes).
- **Similarity without embeddings.** 38.leetcode's plagiarism check is k-gram fingerprinting
  (MOSS-style winnowing), a lexical choice over code that implicitly declines embedding
  similarity — exact-ish, explainable, no model serving.
- **Same word, different technology:** 17.elevenlabs (voice identity) and 35's transcode-pipeline
  embeddings are embedding *producers* with retrieval bolted on; [REDIS.md](REDIS.md) and
  [BIGTABLE.md](BIGTABLE.md) etc. contain no vector-search content — REDIS.md's "amplification
  vector" is the security sense.

**False-positive dictionary** (filtering decisions, per the audit): "vector *tiles*" in
geospatial rendering (26.google-maps, 22.frontend-uber); "attack / abuse / amplification /
delivery vector" in the security sense (09.cdn, 34.reddit, 40.tinyurl); "*growth vector*"
(21.frontend-calendar); "*update vector*" in offline-first cart merge (39.e-commerce); the
"trust vector" of host traits (36.airbnb); seat-state "*bitmaps/vector*" (37.ticketmaster);
PromQL "*vector matching*" over time series (49.monitoring-metrics); search "*term vectors*"
for snippets (48.google-search); "wasm *embedding*" of an interpreter, not the ML sense
(19.frontend-chat-app); diffusion "latents" are model-internal tensors, not search embedding
(18.sora, 16.dalle elsewhere); "semantic" matched "semantics" in 40+ files (consistency
semantics, burst semantics) and only two hits are this topic (16.dalle's semantic cache, 05's
semantic search); the unguarded greps `RAG` (⊂ "storage") and `ANN` (⊂ "channel") require
word-boundary matching to be useful.

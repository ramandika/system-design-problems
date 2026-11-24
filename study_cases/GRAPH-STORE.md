# Graph stores across the case catalog

Where a social/follow graph (or another edge store) appears in the 50 study cases, what role it
plays, and how deep each case goes. Companion to [CONCEPTS.md](CONCEPTS.md) — note: the 50-case
sweep that produced CONCEPTS.md recorded no graph-store union (checked; zero graph rows), so this
doc is the first pass. Division of labor with the sibling docs: [KAFKA.md](KAFKA.md) owns the
fanout *log* (the event pipeline that carries edge changes and tweet copies);
[SEARCH-INDEX.md](SEARCH-INDEX.md) owns 42.linkedin's index side; this doc owns the **edge store
itself** — who follows whom, where it lives, and what it costs to walk.

Headline: **no case in this catalog serves its graph with a graph database.** Follow graphs and
edge stores are built out of general-purpose machinery — KV blobs and relational tables sharded
by one endpoint — and every traversal is hop- or degree-bounded. The one case that puts a native
graph DB on the table (42) evaluates it and rejects it with an explicit trade-off, and a TAO-style
lookaside-cache architecture (Facebook's publicly described design) appears nowhere — zero grep
hits; the closest thing is 02.twitter's timeline cache, which caches *fanout output*, not edges.
This is honestly a thinner topic than Kafka or Postgres in the catalog: one case (42) decides the
storage question end-to-end, three design the store as a core component, the rest mention or
contrast it.

- Cases using a graph-store pattern: **9 / 50** (mention-based, counted by grep + context read;
  see "Filtered mentions" below)
- Designed edge store as a core architectural component: **4** (02, 33, 34, 42)
- Explicit bounded/derived use: **2** (32, 41)
- Contrast or candidate-source mention only: **3** (05, 15, 35)
- Native graph database evaluated in a trade-off: **1** (42 — rejected)
- TAO-style lookaside cache architectures, graph partitioning engines: **0**

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [02.twitter](02.twitter/README.md) | Social graph service + follower lists sharded by followed_id, feeding fanout | Core component (decisions live in the fanout dive) |
| [33.instagram](33.instagram/README.md) | Follow graph (with consent twist) driving candidate pools for a ranked feed | Core component |
| [34.reddit](34.reddit/README.md) | Comment *tree* as relational edges (adjacency = truth, path = read store) | Deep, but tree ≠ social graph |
| [35.tiktok](35.tiktok/README.md) | Anti-case: the feed is explicitly *not* a follow graph; graph demoted to one recall source | Deep as a rejection |
| [42.linkedin](42.linkedin/README.md) | Symmetric consented edge store, dist-2 precompute, graph DB vs KV adjacency | The case that decides the question |
| [32.spotify](32.spotify/README.md) | Social graph as "modest-scale relational, bounded per user" | Brief stance |
| [41.web-crawler](41.web-crawler/README.md) | Derived web link-graph builder as one stream consumer | Brief |
| [15.ml-ranking-system](15.ml-ranking-system/README.md) | Follow graph as one retrieval candidate source | Brief |
| [05.discord](05.discord/README.md) | Contrast: guild membership is an explicit bounded set, not a follow graph | Brief |

## Per-case notes

### Designed graph stores

**[42.linkedin](42.linkedin/README.md)** — the flagship; the only case that stages the graph-DB
question and answers it. The graph is *undirected and consented* — a pending/accept state machine
Twitter never needs — sized in [02-scale-estimates.md](42.linkedin/02-scale-estimates.md) at
~300M active members × ~400 connections ≈ **~120B edge endpoints** (~60B unique pairs + tail),
~400 entries × 8-byte IDs ≈ 3.2 KB/member ≈ **~1 TB** of raw adjacency: small *per member*, huge
globally, so one shard per member works. The storage dive in
[04-deep-dives.md](42.linkedin/04-deep-dives.md) weighs **Option A — general-purpose graph
database** (native traversals): dist-2 is a one-line query, but online BFS is O(fanout) per hop —
a member at the publicly documented 30K-connection cap means ~30K shard-random adjacency fetches
per hop, per request; at ~10K QPS of degree lookups that is ~1–10M adjacency fetches/s with
unpredictable p99. **Option B — relational/KV adjacency (partitioned by member) + precomputed
dist-2** wins: point reads on one shard, dist-2 aggregated offline where the cost amortizes.
Chosen: **Option B**, because the requirements bound traversal at 1st–3rd degree
([01-requirements.md](42.linkedin/01-requirements.md): "no open-ended BFS"), so *what* to
precompute is known and bounded — per-member dist-2 **aggregates, never full lists** (a full
list for a 30K-cap member would be 9M pre-dedup endpoints ≈ 72 MB of 8-byte IDs, unservable hot;
aggregates + sampled paths ≈ ~15 TB), rebuilt stream-side from edge-change events
([KAFKA.md](KAFKA.md) covers that log). Sub-decision, and the doc's most portable line: **the
state machine lives where it's small, the graph lives where it's hot** — invite rows in a
relational table (transactional), accepted edges in KV blobs per member (hot, cacheable;
celebrity adjacency is read-mostly, so cache it aggressively). The derived-store failure mode is
stated: a stalled dist-2 job leaves badges stale rather than wrong, and paths older than a
threshold are hidden, not guessed.

**[02.twitter](02.twitter/README.md)** — the reference social graph store, but a quiet one: the
edge store never gets its own deep dive because the interesting decisions all happen in the
fanout that consumes it ([04-deep-dives.md](02.twitter/04-deep-dives.md); the log side is
[KAFKA.md](KAFKA.md)'s). What the architecture does specify
([03-architecture.md](02.twitter/03-architecture.md)) is the shape: a **social graph service +
store holding follower lists per user, shardable by followed_id** — i.e., partition by the
endpoint you enumerate, not the one you query. That single choice is what makes
fanout-workers-read-followers affordable at ~180B timeline inserts/day (~600M tweets × ~300
followers, [02-scale-estimates.md](02.twitter/02-scale-estimates.md)). The graph also surfaces
as its own abuse surface: a bot that follows 10K accounts exponentially amplifies every later
tweet, so rate limits apply *to the follower graph itself* before fanout. Its timeline cache is
the TAO-adjacent pattern (per-user cached lists, Redis sharded by user_id), but it caches fanout
*output* keyed by recipient — the edges stay in the graph store, so this is not a lookaside
cache in TAO's sense.

**[33.instagram](33.instagram/README.md)** — Twitter's graph with two architectural twists
([01-requirements.md](33.instagram/01-requirements.md)). First, **consent**: private accounts
make a follow a two-step request/approve act — the same half-edge-vs-consent problem 42 solves
with a state machine, kept here to a light approve step. Second, **ranking**: the product is a
ranked hybrid feed, so the graph stops being the timeline's ground truth and becomes the source
of *candidate pools* — fanout-on-write materializes potential posts per user, ordering happens
on read ([04-deep-dives.md](33.instagram/04-deep-dives.md) #3; ~230M posts × ~300 followers ≈
70B candidate-pool inserts/day, a 15:1 insert:read ratio vs Twitter's ~180:1). Costs of that
choice are stated: unfollow hygiene is now a real engineering job — a heavy unfollow triggers a
batch rewrite of affected pools. Explore/Reels goes the other way entirely: candidates come from
the *global* graph (embeddings, creator graph, engagement), where no fanout is possible, ever.
The case's framing of the whole system as a "**file**-based social graph" reads cleanly in
[02-scale-estimates.md](33.instagram/02-scale-estimates.md): read:write amplification ≈ 80:1
versus Twitter's ~2:1, which is why CDN-first, not graph-first, design.

### Graph as relational edges

**[34.reddit](34.reddit/README.md)** — included with a disclaimer: Reddit's defining graph is a
*comment tree*, not a social graph, and the storage dive in
[04-deep-dives.md](34.reddit/04-deep-dives.md) is the catalog's clean statement of the
relational-edge pattern in miniature. Option A **adjacency list** (`parent_id`) wins write
simplicity and single-column integrity and stays the Postgres source of truth, but loses every
read: subtree fetch is a per-level self-join crawl on a 50K-node thread. Option B
**materialized path** wins the dominant read (subtree = one prefix scan), loses self-contained
integrity — the adjacency record remains the arbiter and a daily reconcile job walks both.
Option C closure table wins query generality nobody needs and pays O(nodes × depth) write
amplification in exactly the hot window nobody can afford. **Chosen: B for reads, A as truth.**
Same discipline as 42, smaller graph. Where Reddit's *social* graph is concerned, the case makes
the opposite call from Twitter on purpose: fanout-on-read over per-community candidate pools,
because huge communities multiply fanout cost that "Twitter's flat follow graph doesn't incur"
([04-deep-dives.md](34.reddit/04-deep-dives.md), feed dive), with the merge bounded by
per-community top-N lists.

**[32.spotify](32.spotify/README.md)** — one sentence, but a real design stance rather than a
vibe: the social graph (follows, shares, collaborative-playlist membership) is "modest-scale
**relational** data, bounded per user" ([04-deep-dives.md](32.spotify/04-deep-dives.md)); the
Twitter-style fanout problem exists only in miniature, so the activity feed is pulled on app
open, not fanned out on write — no edge-store machinery built where the degree bound makes it
cheap. The catalog's smallest answer to the same sizing question.

### The anti-case

**[35.tiktok](35.tiktok/README.md)** — the useful counter-example, the 11.zookeeper of this
topic. The case's opening clause is the boundary statement: "the feed is **not a follow
graph**"; Twitter can precompute timelines "because the follow graph is the ground truth", and
here there is nothing to fan out to — candidates are computed from scratch per request
([README.md](35.tiktok/README.md)). The follow graph is not absent, it is *demoted*: one recall
retriever among embeddings, trending, and sound indexes in
[03-architecture.md](35.tiktok/03-architecture.md), and the same in the generic ranking funnel
([15.ml-ranking-system](15.ml-ranking-system/03-architecture.md): "follow graph / co-visit
item→item lists" returning a few hundred candidates in ~10 ms). Lesson: the edge store is a
retrieval source, not a product foundation — when the candidate universe is "accounts you
follow" you materialize; when it is "anything on the platform" you compute. No fanout, no
celebrity problem, no graph store to size.

### Brief mentions

- **[05.discord](05.discord/README.md)** — membership is "an explicit act... unlike Twitter's
  follow graph"; a user belongs to tens to hundreds of guilds, and delivery targets "the
  channel's member set, uniformly huge for large guilds" — a bounded membership set, not a
  degree-skewed edge table. Mentioned only to mark the difference.
- **[41.web-crawler](41.web-crawler/README.md)** — the web's link graph *is* an edge store in
  the catalog, but a derived, offline one: a link-graph builder is one of the stream consumers
  on the post-dedup crawl topics ([03-architecture.md](41.web-crawler/03-architecture.md)), its
  output feeds importance (backlink-derived) into the freshness scheduler. Never a serving path;
  [KAFKA.md](KAFKA.md) covers the log it consumes.

### Filtered mentions (false positives, excluded from all counts)

Grep hits that look like graph-store hits and are not:

- **[26.google-maps](26.google-maps/README.md)** — a literal "ROAD GRAPH STORE" and ~420K
  concurrent *graph searches* ([02-scale-estimates.md](26.google-maps/02-scale-estimates.md)):
  a road-network routing problem solved with hierarchical partitioning (CRP), a different
  discipline (spatial hierarchy, not bipartite edge sharding). Not a social-graph pattern; 26's
  own case covers it.
- **[45.typeahead](45.typeahead/01-requirements.md)** — "spell-correct adjacency": edit-distance
  neighborhoods, no graph.
- **[30.facebook-messenger](30.facebook-messenger/README.md)** — "sender-graph signals" in the
  abuse dives: graph/metadata *features* (frequency, account age) feeding spam filters; no
  graph store designed.
- **[04.deployment-system](04.deployment-system/README.md)** — a DAG executor "walks the
  graph": build-dependency DAG, an orchestration pattern.
- Consensus "follower" ([11.zookeeper](11.zookeeper/README.md), [13.kafka](13.kafka/README.md)),
  network "hop" (03, 09, 10, 40, 43, 47, 49, 50), CDN "nodes"
  ([08.chatgpt](08.chatgpt/README.md)): vocabulary collisions only.

## The portable pattern

Every substantive use reduces to one contract, cross-linked to
[42.linkedin/04-deep-dives.md](42.linkedin/04-deep-dives.md) as the reference implementation:

1. **Store edges in a general-purpose engine, sharded by the endpoint you enumerate.** 42's KV
   blobs per member, 02's follower lists "shardable by followed_id", Reddit's adjacency rows:
   one shard per member/endpoint, point-read shaped. Nobody partitions a graph the way
   graph-partitioning engines do, and no case ever wanted to.
2. **The state machine lives where it's small, the graph lives where it's hot** — 42's sub-
   decision, the catalog's cleanest: transactional invite rows in one store, hot accepted-edge
   blobs in another. Consent is modeled in data (42's pending/accept, 33's private-account
   approval); one-sided follow (02) is the degenerate case that skips it.
3. **Traversal must be bounded before storage is chosen.** 42 bounds at 1st–3rd degree by
   requirement and precomputes dist-2 *aggregates, never lists* (72 MB full lists for a 30K-cap
   member); 41 bounds redirect chains by a hop limit plus loop detection. Unbounded BFS is
   treated as a non-starter regardless of engine.
4. **Celebrity asymmetry is a read-caching problem, not a graph-engine problem.** 42 caches the
   hot, read-mostly adjacency blob; 02 keeps celebrity edges in the same store as everyone else
   and makes the fanout *consumer* pay. The edge store itself stays uniform.

Boundaries — where the pattern, or the whole idea of a graph store, stops:

- **Hot celebrity edges are a fanout trade-off, not a storage one.** The fanout economics
  (the ~180B inserts/day wall) live in [02.twitter/04-deep-dives.md](02.twitter/04-deep-dives.md)
  and [KAFKA.md](KAFKA.md)'s log; the edge store only has to enumerate followers fast. Choosing
  fanout-on-write vs on-read changes nothing about the follower lists.
- **Deep traversal explodes, and nothing here saves it.** Hop limits are the escape (42's
  degree bounds, 41's crawl caps); if the product genuinely needs arbitrary-depth traversal at
  interactive QPS, no case in the catalog offers a validation — that region is undiscussed,
  and 42's Option A rejection says why the default answer is wrong at 120B edges.
- **When the graph isn't the product, don't size a graph store.** 35 computes per-request from
  an interest model and demotes follows to one recall source; 05 never builds an edge table
  because membership is a bounded set; 32 sizes its graph as too small to need machinery. The
  first decision in the pattern is the 02-vs-35 question: is the follow graph the ground truth
  of your feed, or one signal among many?

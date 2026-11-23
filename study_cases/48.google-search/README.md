# Case 48 — Web-scale search (Design Google Search)

**The system**: a user types a query; within ~200 ms a ranked list of ten blue links
appears, each with a snippet drawn from a page indexed sometime in the last minutes-to-months.
Underneath: a crawler continually feeds extracted content into an inverted index of ~100
billion documents, the index is sharded across thousands of machines, and every single
query is answered by asking essentially *all* of them — then pruning, ranking, and merging
down to ten results in a latency budget shorter than a blink. Pair this with
[Case 41 — Web Crawler](../41.web-crawler/README.md): that case ends at "handoff to the
indexer"; this case starts exactly there and covers ranking, serving, and everything the
query path touches.

**Why it's hard**:

- **Cost per query is the product.** A search box is trivial; the inverted index is not.
  Every query reads posting lists that can be billions of entries long, fans out to
  thousands of index shards, and if each shard naively scored everything, the arithmetic
  ([02-scale-estimates](02-scale-estimates.md) §6) needs millions of CPU cores. Early
  termination (WAND-style pruning) is not an optimization, it is what makes the system
  affordable at all.
- **Index sharding is broadcast, not partition.** No term partitioning saves you: most
  queries contain a very common term whose posting list lives everywhere. Document
  partitioning means every query touches every shard — so shard count is a direct tax on
  both latency distribution and fleet cost (deep dive 2).
- **Freshness and index efficiency are enemies.** A fully-merged immutable index is the
  cheapest thing to serve from and the slowest thing to update; a document-level
  updatable index is fresh but slow and bloated. The industry answer — small mutable
  segments that periodically merge into the base index — has its own tail-latency cost
  when the merge fires (deep dive 3).
- **The tail latency is the user experience.** Median is easy; p99 and p99.9 are what a
  user actually feels, and they are inflated by exactly the things a search system is
  made of: a fanout to thousands of nodes where *any* laggard stalls the response
  (the "Tail at Scale" problem, deep dive 4).
- **Ranking is a pipeline under a deadline.** Retrieval is filter-and-score in the index;
  ranking is machine learning over features. The heavy model can only afford the top
  hundreds, the light model the top thousands, and the features themselves can go stale
  the moment the content they describe changed last night (deep dive 5).

**Reading order**: [01-requirements](01-requirements.md) →
[02-scale-estimates](02-scale-estimates.md) → [03-architecture](03-architecture.md) →
[04-deep-dives](04-deep-dives.md). Read [Case 41](../41.web-crawler/README.md) first if the
crawl side is new; its deep dive 6 (recrawl scheduling) is this case's freshness supply line.

**Patterns taught here**: inverted index as the canonical read-optimized secondary index
(posting lists, delta encoding, skip pointers); two-axis partitioning (document- vs
term-split) and why broadcast query cost dominates; the tail-at-scale playbook
(hedged requests, partial-result degradation) — a distributed-systems generalization of
the fanout in [Twitter](../02.twitter/README.md); multi-stage ML inference under a latency
budget, the same funnel as the [Ranking System](../15.ml-ranking-system/README.md) case;
tiered indices (fresh vs base) as the storage-tiering pattern of
[storage tiering](../13.kafka/README.md) applied to search; and Query Results caching,
the Zipfian case for [Distributed Cache](../10.distributed-cache/README.md).

**Try it first**: before reading 03, answer: (1) a query contains the word "the" — its
posting list covers most of the 100 B documents; why doesn't the shard have to read it all,
and what specifically lets it stop early? (2) you shard the index by term instead of by
document — what query shape destroys you? (3) it's 00:05 UTC and a major news event breaks;
one query now arrives 100× more often — what happens at each layer, and which layer absorbs
it without touching the index at all?

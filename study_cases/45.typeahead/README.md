# Case 45 — Typeahead

**The system**: search-box query suggestions — type "net", see "netflix", "new york times",
"nfl scores", each within milliseconds, ranked by global popularity plus recent trends, with
your own history woven in. The smallest big-system in this corpus: the index fits in tens
of gigabytes of RAM (est. sheet), and the entire design difficulty is *latency discipline* and
*freshness under load*, not storage.

**Why it's hard**:

- It sits **in the typing critical path**: a suggestion that arrives in 500 ms doesn't help —
  the user has finished the word. Budgets are sub-100 ms, and the p99 *is* the product
  (deep dives §5).
- Traffic is **extremely Zipfian**: two-letter prefixes like "ne" carry orders of magnitude
  more traffic than the 10M-plus tail prefixes — caching and hot keys dominate the serving
  story (deep dives §3).
- Users expect **two kinds of recency at once**: stable popularity (years of query logs) and
  minutes-fresh trending ("world cup"), which have opposite cacheability (deep dives §2).
- The corpus is **user-generated**: every prefix completion is a potential offensive,
  misleading, or brigaded term — filtering must happen in the pipeline, not per query
  (deep dives §2).

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: prefix index with materialized top-k per prefix, multi-tier cache
(client → edge → per-prefix cache) with hot-key coalescing, offline-build + streaming-boost
hybrid freshness, read:write asymmetry extreme enough to make on-write-indexing free (kin of
[40.tinyurl](../40.tinyurl/README.md)); query-log analytics as the input pipeline.

**Try it first**: before reading 03, answer: (1) where does the index live, RAM or disk, and
how do you know your answer from the traffic? (2) a breaking news term spikes 100× in 5
minutes — when can it *appear* in suggestions, and what stops an attacker from making
anything trend? (3) what does your service return when a prefix has *never* been seen?

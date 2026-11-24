# Probabilistic structures across the case catalog

Where Bloom filters, sketches and explicitly-approximate estimators appear in the 50 study
cases, what role they play, and how deep each case goes.
Companion to [CONCEPTS.md](CONCEPTS.md) (no tiered row exists for this concept — its
absolute count is too low for the union table; this doc is a per-case count, not a
CONCEPTS.md union count).

Structure families covered here: Bloom filters (incl. negative-lookup gates), similarity
sketches (SimHash/MinHash+LSH), count-min / frequency sketches, and explicitly-approximate
counters where the case names approximation as a design decision and states its trade-off.

Headline: **in this catalog, probabilistic structures almost never estimate — they gate and
they absorb.** Bloom filters appear exactly twice as designed components, and both times as
a *negative-lookup* gate ("definitely not there → stop paying now"): Bigtable skips SSTables
that cannot contain the key, TinyURL rejects unknown short-codes before the KV store. The
only sketch actually named and chosen (not merely discussed) is a frequency sketch in the
cache eviction path. Approximate counting appears as a *stated product decision* — "this
number may drift" — never as a named estimator. HyperLogLog, count-min sketch and
cardinality estimation appear **nowhere as deployed components**: count-min is named twice,
each time as an option that loses to exact memory. Every use sits inside the same
trade-off sentence: "lose exactness, buy memory/latency".

- Cases that name a probabilistic structure or an explicitly approximate estimator: **10 / 50**
- Designed component (named in the architecture): **5** (12 bigtable, 40 tinyurl, 41 web-crawler, 10 distributed-cache, 35 tiktok)
- Discussed option, not chosen (or chosen only for a secondary surface): **2** (38 leetcode, 09 cdn)
- Brief / policy-only mention (approximate accepted, no structure named): **3** (01 youtube, 34 reddit, 39 e-commerce)
- Adjacent, excluded from the count: **2** (47 rate-limiter = exact algorithmic counters with accepted drift, not sketches; 49 monitoring-metrics = "approximate" by staleness, "cardinality" = label cardinality)
- False positives filtered: see the list at the end of this doc

## Summary table

| Case | Role | Depth |
|---|---|---|
| [12.bigtable](12.bigtable/README.md) | Per-SSTable Bloom filter skipping files that cannot hold the key; read-amplification buy-back | Deep dive (§6) |
| [40.tinyurl](40.tinyurl/README.md) | Bloom filter of known codes on the redirect path: rejects unknown codes before the KV store | Deep dive (§2) |
| [41.web-crawler](41.web-crawler/README.md) | SimHash + MinHash/LSH probabilistic similarity pipeline for near-dup detection; exact-hash seen-set | Deep dive (§2) |
| [10.distributed-cache](10.distributed-cache/README.md) | LFU via frequency sketch (count-min + doorkeeper) chosen for object eviction | Deep dive (§6) |
| [35.tiktok](35.tiktok/README.md) | Sharded approximate counters behind a streaming aggregator; at-least-once loses exactness at the counters | Core component |
| [38.leetcode](38.leetcode/README.md) | Count-min / streaming heavy-hitters Option C: rejected for contest standings, acceptable for all-time leaderboards | Deep dive (rejected option) |
| [09.cdn](09.cdn/README.md) | Tag-purge via per-object tag bloom — option (b), lost to an asynchronous inverted index | Brief (rejected option) |
| [01.youtube](01.youtube/README.md) | View dedup approximate at the edge, precise in offline audit | Brief |
| [34.reddit](34.reddit/README.md) | Live vote scores tolerated as approximate (matching Reddit's own public fuzzing practice) | Brief |
| [39.e-commerce](39.e-commerce/README.md) | Facet result counts as sampled estimates on long-tail filter combinations | Brief |

## Per-case notes

### Negative-lookup Bloom gates (the catalog's real pattern)

Both designed Bloom filters answer the same question: *can this key possibly exist here?*
A "no" short-circuits an expensive probe; a "maybe" falls through and costs one wasted
lookup. Neither filter is on a delete-bound path — that is what makes them cheap to keep.

**[12.bigtable](12.bigtable/README.md)** — the engine's own debt-repayment structure. Deep
dive 4 (LSM choice) put ~10–15 file probes per point read on the read path on purpose;
[04-deep-dives.md](12.bigtable/04-deep-dives.md) §6 buys it back with a per-SSTable bloom
filter: ~10 bits/key, ~7 hashes (textbook m/n=10 arithmetic) → ~1% false positives.
[02-scale-estimates.md](12.bigtable/02-scale-estimates.md) traces the effect: expected
SSTables touched falls from ~8 to ~1–2, and at ~1.25 bytes/key against a ~1 KB median cell
that is under ~0.2% extra storage. Its own trade-off statement: the LSM "gave cheap writes
by fragmenting rows across k files; bloom filters and locality groups are the two levers
that decide whether k is 2 or 15 at read time" — blooms remove *files that can't have the
key*, locality groups remove *files that can't have the column family*. Failure modes are
named: 1% FP on a hot read pattern is ~1% of reads doing a full file probe (mitigated by the
block cache, not by more bits), and a stale bloom after major compaction must be rebuilt,
not patched. Per-table and per-locality-group tunability matters: scan-heavy tables skip
blooms entirely (scans read everything anyway).

**[40.tinyurl](40.tinyurl/README.md)** — a Bloom filter against *adversarial* traffic.
Unknown short-codes (scrape/guess traffic) hammer a path where negative results "don't cache
naturally", so every garbage code otherwise reaches the KV store. The fix in
[04-deep-dives.md](40.tinyurl/04-deep-dives.md) §2 and
[03-architecture.md](40.tinyurl/03-architecture.md): a **Bloom filter of all known codes**
in every redirect node — "Definitely not there" → 404 immediately, no cache hop. Its
trade-off, verbatim: "the Bloom filter wins K memory lookups at a cost of ~10 bits/code
(~1% false-positive rate) and a rebuild/invalidation path … a filter is approximate — it
never needs deletions if links never truly expire". False positives fall through to a normal
lookup and still return 404 — wrong only in cost, never in correctness. The write path
pays for it: creations must insert into the filter (one more thing to keep consistent).

### Similarity sketches: approximate because exact cannot define the target

**[41.web-crawler](41.web-crawler/README.md)** — the largest probabilistic structure in the
catalog, and the only one used for *estimation*. [04-deep-dives.md](41.web-crawler/04-deep-dives.md)
§2 splits dedup by layer. Exact SHA-256 (Option A) wins zero false positives but "detects
maybe 20–30% of the useful duplicates" — near-dup is the common case in a web corpus. The
chosen pipeline: **SimHash** (a single 64-bit fingerprint per page, ~8 B/page vs ~1 KB for a
128-shingle MinHash sketch, near-dup = Hamming distance under threshold) as the volume gate,
**MinHash + LSH** as the confirm/cluster step because SimHash "doesn't give you the actual
shared shingles" and needs a second stage for audit. Both options are stated with losses:
MinHash+LSH needs a real LSH index and "never fires on small shared fragments"; SimHash is
"a coarse signal" and depends on IDF-like token weighting. The README names the pattern
directly ([README.md](41.web-crawler/README.md)): "hash-based dedup with approximate
structures where exact is either too big or too slow". The boundary is in the same system: the
**seen-URL set stays exact** — ~500 B canonical hashes of 16 B each ≈ 8 TB
([02-scale-estimates.md](41.web-crawler/02-scale-estimates.md)) — because a Bloom false
positive there would permanently drop a URL from a crawler whose contract is full coverage.

**[10.distributed-cache](10.distributed-cache/README.md)** — the sketch as an eviction-policy
ingredient. [04-deep-dives.md](10.distributed-cache/04-deep-dives.md) §6 weighs approximate
LRU (segment/window approximations) against LFU "e.g., frequency-sketch based, as in Redis
4+ LFU or W-TinyLFU's count-min sketch + doorkeeper". LFU's losses: per-key counts (or a
sketch) cost memory and create the new-entry problem (count 0 → evicted before a fair trial;
mitigated with an initial count + decay, "divide by 2 every N minutes"). Chosen: **LFU for
the object class, plain LRU-or-nothing for sessions (which expire by TTL)**; trade-off
accepted: "slightly more bookkeeping memory (a sketch, doorkeeper) in exchange for a higher
hit rate on the skew". The general caveat is stated too: the approximate policies "sample
rather than scan", so eviction efficiency degrades non-linearly as utilization → 100% —
hence the 30% memory headroom rule. This is the second named component and, again, it is a
membership/frequency gate inside a control loop, not a published estimate.

### Approximate counters as a named product decision

**[35.tiktok](35.tiktok/README.md)** — the only case where approximation is an architectural
tier, not one lever. [03-architecture.md](35.tiktok/03-architecture.md) names **sharded
approximate counters** fed by a streaming aggregator as a core consumer family of the
observe-plane log. [04-deep-dives.md](35.tiktok/04-deep-dives.md) dive 4 states the trade at
~2.1T events/day: Option B (at-least-once + idempotency keys, dedup at consumers) "wins
order-of-magnitude cheaper ingestion … loses exactness at the counters (acceptable —
like-counts are approximate by design)". Notably the case does *not* sketch its way there:
no HLL, no count-min — approximation comes from sharding + aggregation + tolerated lag.
[02-scale-estimates.md](35.tiktok/02-scale-estimates.md) confirms "view/like counters are
approximate and lag". The case then guards its own decision: the feature-store consumer
*must* dedup by event UUID (a double-counted watch signal skews a model), and dive 4's
failure modes warn that naive cold-start scoring trusting "approximate and lagged" early
counters double-punishes new videos — velocity vs. baseline priors, never raw counts.

**[01.youtube](01.youtube/README.md)** — the same family at YouTube's scale, one trade-off
line deep: view counts are served from an event pipeline with local pre-aggregation and no
1-row-per-view table; "**dedup (bots/refresh) is approximate at the edge and precise in an
offline audit** — eventual consistency buys us the stream-able design"
([04-deep-dives.md](01.youtube/04-deep-dives.md)). Two-tier split: edge is cheap and
approximate, the offline audit is authoritative.

**[34.reddit](34.reddit/README.md)** — brief, and backed by public practice:
[02-scale-estimates.md](34.reddit/02-scale-estimates.md) — "Vote-score display is fuzzed by
Reddit publicly … so is vote counting precision. Our design tolerates approximate live
scores for the same reason — the ranking function only needs the score to be right within a
few percent." Approximation is justified by the consumer, not by memory pressure.

**[39.e-commerce](39.e-commerce/README.md)** — approximation scoped to exactly one display
surface. Facet counts are "pre-aggregated at indexing time … Counts can be approximate under
filter combinations (the 'brand=3,254 results' number may be a sampled estimate on
long-tail queries) — acceptable, because the number's job is navigation, not bookkeeping"
([04-deep-dives.md](39.e-commerce/04-deep-dives.md)). [01-requirements.md](39.e-commerce/01-requirements.md)
draws the line the same way: the ATP badge may be minutes-inexact; order-fulfillment invariants may not.

### Discussed and rejected — explicit "exactness wins here" statements

**[38.leetcode](38.leetcode/README.md)** — the sharpest boundary statement in the catalog.
[04-deep-dives.md](38.leetcode/04-deep-dives.md) dive on leaderboards: "**Option C —
approximate sketches (count-min over score buckets, streaming heavy-hitters)** wins sub-µs
updates, tiny memory; loses: approximate ranks are wrong in exactly the places users care
(the boundary around rank 100 becomes contested, ties at the cut are argued publicly)" —
and the verdict: "approximation here is acceptable for a *recommendation* surface, not a
scoreboard." Chosen instead: exact per-contest sharded ordering (O(log N) ZADD), with offline
recompute from the verdict log to converge provisional standings to canonical. The option is
not dead: "for the all-time/per-problem leaderboards (billions of rows), sketches or
page-based serving is fine and Option C returns there." Classification: explicitly rejected
for the product-critical surface, accepted in principle for secondary surfaces.

**[09.cdn](09.cdn/README.md)** — a Bloom-shaped structure proposed and set aside. In the
surrogate-key / tag-purge discussion, [04-deep-dives.md](09.cdn/04-deep-dives.md) weighs
(a) a global tag→keys inverted index against (b) "embed a tag bloom/bitmask as part of the
cached object's metadata and purge by *scanning* caches for the tag — no index, purge
latency ~scan time". Chosen: (a), built asynchronously from access logs. The structure here
is a membership *deletion* problem — the weak side of Bloom filters — and the scan-based
variant lost to an index on latency grounds, not on exactness grounds.

### Adjacent, excluded from the count

**[47.rate-limiter](47.rate-limiter/README.md)** — the case sits next to this concept and
explicitly refuses it. Its counters are *exact algorithmic* counters (token bucket / GCRA,
sliding-window counter blends) run on sharded state with leased batches; the approximation
it accepts is **drift from lag**, not a sketch statistic: [01-requirements.md](47.rate-limiter/01-requirements.md)
— "soft limits may be enforced with approximate, drifted counters; hard limits may not
drift far below what the customer paid for" ([01-requirements.md](47.rate-limiter/01-requirements.md)),
and it insists the approximation be deliberate: `RateLimit-Remaining` "reads the state that
made the decision … approximate by published design rather than approximate by accident"
([03-architecture.md](47.rate-limiter/03-architecture.md)).
Classified adjacent per the task's caution: exact counters with acknowledged drift are not
probabilistic structures unless the case names approximation as the estimator method — it
names drift. It is, however, the catalog's cleanest statement of *which* surfaces may be
approximate at all.

### False positives filtered (noted per the methodology rules)

- "sketch" in [31.netflix](31.netflix/README.md), [32.spotify](32.spotify/README.md),
  [01.youtube](01.youtube/README.md), [18.sora](18.sora/04-deep-dives.md) — "sketch a design"
  verb / generative-model vocabulary, no data structure.
- "Bloom" in [35.tiktok](35.tiktok/04-deep-dives.md) — "late bloomers", creator-cold-start wording.
- "cardinality" in [49.monitoring-metrics](49.monitoring-metrics/README.md) — label
  *dimensionality* (the cardinality bomb), not cardinality *estimation*; no HyperLogLog. Its
  query semantics are "approximate" only through staleness; one line does invoke the pattern
  by analogy ("same ledger as Bigtable's bloom filters + locality groups",
  [04-deep-dives.md](49.monitoring-metrics/04-deep-dives.md)).
- "probabilistic" in [08.chatgpt](08.chatgpt/04-deep-dives.md) — an "approximate affinity
  table (…TTL'd, probabilistic)" routing heuristic for KV-cache affinity; a heuristic cache,
  not a sketch structure. Related single-line "approximate" wording with no structure behind
  it: [20.frontend-newsfeed](20.frontend-newsfeed/04-deep-dives.md) ("approximately-ordered
  deep feed"), [19.frontend-chat-app](19.frontend-chat-app/04-deep-dives.md), [22.frontend-uber](22.frontend-uber/04-deep-dives.md),
  [36.airbnb](36.airbnb/04-deep-dives.md), [07.blob-store](07.blob-store/04-deep-dives.md),
  [26.google-maps](26.google-maps/README.md), [48.google-search](48.google-search/01-requirements.md),
  [33.instagram](33.instagram/04-deep-dives.md), [37.ticketmaster](37.ticketmaster/04-deep-dives.md).

## The portable pattern

Every substantive use reduces to one decision, with a boundary:

1. **Gate the negative, never the positive.** Both designed Bloom filters answer "could this
   key exist?" and only the "no" side is acted on (12 skips the file, 40 returns 404). A
   "maybe" costs one wasted probe; a false negative would cost data. That is why the filter
   is safe in a serving path and why it needs no deletions (40's links never expire; 12
   rebuilds blooms at compaction instead of patching them).
2. **Price the approximation in the same sentence as its consumer** (12: 1.25 bytes/key vs
   10–15 file probes; 10: sketch memory vs hit rate on skew; 35: like-counts approximate by
   design; 38: sub-µs updates vs a publicly contested rank-100 boundary). Where the doc can't
   justify the loss to a named consumer, the option loses — as in 09 and 38's scoreboard.
3. **Approximate where recompute is possible or where the consumer is itself approximate or
   adversarial traffic is the problem** (40's unbounded unknown-code traffic; 41's
   confirm/cluster second stage; 01's offline audit; 38's offline verdict-log recompute for
   canonical standings).
4. **The boundary — when approximating is WRONG:**
   - **Money and billable truth.** Nothing in the catalog sketches a payment count;
     [39.e-commerce](39.e-commerce/README.md) inlines the rule as a requirements split —
     the ATP badge may be inexact, fulfillment may not. [47.rate-limiter](47.rate-limiter/01-requirements.md)
     states the same for quotas: "hard limits may not drift far below what the customer paid for."
   - **Scoreboards and cut lines people argue about.** 38 rejects count-min because "approximate
     ranks are wrong in exactly the places users care" — the product's integrity statement.
   - **A false positive that deletes.** Bloom filters can say "present" wrongly; wherever a
     wrong "yes" means *lost content*, the catalog pays for exactness — 41 keeps its 8 TB
     exact-hash seen-set for exactly this reason.
   - **A false positive that leaks or over-grants.** No case uses a probabilistic membership
     test in an auth, permission, or takedown decision; 35 routes the takedown path through
     indexes and metadata, never through the approximate counters.
5. **Cardinality estimation and HyperLogLog are absent** — the honest limit of this pattern
   in the catalog: where distinct-unique counts would be the target (unique viewers, unique
   visitors), the cases either tolerate non-unique proxies (34, 35) or defer to an offline
   audit (01). No case designs an HLL.

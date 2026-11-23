# 04 — Deep dives

## 1. Index: materialized top-k per prefix vs query-time traversal (THE dive)

**Option A — query-time traversal** (a trie/FST walked per request, top-k heap over the
live subtree, e.g., Lucene-style prefix queries):
- wins: always-fresh (index *is* the corpus); no materialization pipeline; rare prefixes
  answered for free.
- loses: every request pays CPU proportional to subtree size and heap management — at 28K
  avg / 85K peak QPS (est. sheet), p99 variance shows up exactly on the hot big-subtree
  prefixes where traffic concentrates; ranking changes (boosts) require index-time score
  plumbing anyway for quality; a model-based ranking re-run per request is out of the
  latency budget forever.

**Option B — materialized top-k per prefix, built offline/streaming, RAM-point-lookup at
serving**:
- wins: serving is a hash + memcpy (~100s of µs p50, est. sheet); hot-key coalescing lands
  on one immutable answer; the ranking logic runs where latency is free; segment swap gives
  atomic index refresh.
- loses: freshness is bounded by pipeline lag unless an overlay layer is added (→ dive 2);
  rare prefixes are *not* in the index (unbounded storage if you materialize everything:
  all-time-different ~1B strings × millions of prefixes would be hundreds of GB and mostly
  pointless — the tail never produces top-8 traffic).

**Chosen: B, with a bounded fallback**: prefixes ≥ ~100 counts/90d (est.) → materialized;
below that → a small on-the-fly fuzzy path over the *same normalized corpus* (edit-distance
candidates, no ranking to speak of) or an honest empty answer. Bounded because the tail
prefixes are individually rare *and* their answer quality doesn't repay infrastructure: an
unfrequented prefix either autocorrects or says nothing.

Sub-decision — in-memory structure: full trie (pointer-heavy, fast incremental) vs sorted
segment/sparse-map of prefix → entry-list (compact, segment-swappable). **Chosen: sorted
segment files**: materialization makes deep traversal unnecessary; the segment format is
cache-locally flat and rebuilds hourly anyway. A trie is the right shape only when you
refuse to materialize (i.e., Option A).

**Failure modes**: build job stall → freshness degrades silently → monitor *last-build-age*
as an SLO metric, degrade by serving previous segment (never empty); shard hot-spotting →
replication factor keyed by observed shard QPS, not uniform (est. §Zipf); corrupted segment
→ double-buffer swap means a bad build is a rollback, not an outage.

## 2. Freshness & trending: two speeds, two layers

The stable index is hours old; users expect *events* ("election results") to appear within
minutes, and trending misfires are reputationally expensive.

**Option A — one index, rebuild faster** (e.g., every 60 s over full logs):
- wins: one structure, one cache TTL story.
- loses: full-corpus rebuild per minute is wasteful but tolerable; the real failure is
  *cache TTL*: a 60 s index forces 60 s response caches, so (a) the cache fleet churns
  constantly (hot keys re-fetch every 60 s fleetwide), and (b) trending still lags a
  counting window + build + swap + cache refill; you buy neither stability nor freshness.

**Option B — stable index + hot overlay (chosen)**:
- streaming counters (event bus → windowed counts with exponential-ish decay) compute a
  *trending score* = recent-velocity × recency-decay, capped by guardrails: per-term
  absolute cap, source (account/device) diversity floors, global boost budget — brigading
  and news-room spikes differ by those predicates, and neither should move the index more
  than its cap.
- hot entries merge onto the stable top-k at query time (replacing entries with a weight
  that dominates), with hot-TTL'd caching (seconds) versus stable-entry caching (minutes).
- policy: hot entries pass the *same* build-time filters by pre-screening candidate terms
  continuously (a streaming policy gate ahead of the hot layer) — nothing reaches serving
  unscreened.
- loses: two caches with two TTLs and a merge step at query time (~still µs, it is 8+
  3 entries); guardrails are policy code that must be exercised (test: "make 1,000 accounts
  query 'buy cheap xyz'" as a fixture).

**Failure modes**: trending feed lags → term appears late (acceptable); guardrail bug
promotes junk → kill-switch drops the whole overlay (stable index carries the product);
event-bus backlog → velocity counts *stale* → suppress boosts older than a freshness bound
(stale trending is worse than no trending — same discipline as
[27.ev-charging-network](../27.ev-charging-network/README.md) freshness guards).

## 3. Serving tiers & the hot-key problem

Zipf reality (est. sheet): a handful of prefixes carry a large share of requests. Serving
design by tier:

- **Client**: debounce (~50–100 ms, ≥2 chars) + per-session prefix cache + cancel-on-
  new-keystroke — kills ~half of raw keystroke requests (3 of ~6 candidates, est.) and, more importantly, kills multi-request bursts
  from one typing session (est. sheet's 3/session is *after* this work).
- **Edge**: static top-N snapshots for the hottest global prefixes (a refreshed
  page-of-answers, TTL ~minutes) — the Zipfian head costs ~zero origin traffic.
- **Per-prefix cache (Redis)** with **request coalescing**: one in-flight fetch per prefix
  per replica; concurrent duplicates await the leader. This is where the hot key lives;
  stale-on-error (serve slightly old at p99, 5–10 s TTL headroom) beats erroring to a user
  mid-keystroke.
- **RAM index nodes**: replicated by *observed* shard load, not uniform; the shard owning
  the most common openers is several replicas' worth of the same segment file (cheap RAM).

**Option A — uniform fleet, cache-aside everywhere**: simple config, but the head prefixes
pound one shard and one cache key until it is the fleet's bottleneck. **Option B — tiered
with skew-aware replication (chosen)**: more moving parts, each aimed at a measured tier of
skew; the cost of a wrong-tier miss is bounded (falls to the next tier unchanged).

**Failure modes**: Redis outage → origin (RAM index) serves, p99 rises from cache-miss
flood → coalescing caps origin lookups at *unique prefixes per interval*, not raw QPS; edge
stale-snapshot bug → hot prefix serves a bad list → fast invalidation path; client cache
poisoning (e.g., a botched personalization) → per-user, self-expiring, never global.

## 4. Ranking beyond frequency (and what personalization costs)

Score at build time = f(count with recency decay, quality of user, completion quality, 
diversity), plus at query time: small personal overlay (user's own recent searches/history).

- **Dedup/normalization is the ranking step people forget**: case/accent/whitespace variants
  ("YouTube"/"you tube"/"YOUTUBE") merge *before* counting, and top-k must dedup semantically
  near-equal completions, or the 8 slots fill with variants of one query.
- **Quality-weighted counts**: bot/low-trust traffic discounted (device/UA heuristics,
  rate-limited segments), or the corpus learns to say what scrapers ask.
- **Personalization**: user-history overlay is *additive* on a global answer — never enters
  the global index (est. 01 rule) and never the shared cache key. It is cache-hostile by
  nature, so it applies *after* the cache (cheap: a handful of user-scoped entries, or
  client-side merge entirely).
- **Diversity constraint**: at build time, cap per-source-type entries (e.g., no more than
  2 completions of the same domain intent) — a query-time constraint would re-break the
  point-lookup design, so it materializes into the top-k.

**Option A — personalization + diversity at query time** (a "real ranker" per request):
wins: flexibility. loses: query-time model and state per request = the latency budget dies
by a thousand microseconds, and the cache tiers of dive 3 all break (per-user keys everywhere).
**Chosen: build-time constraint solving + additive query-time overlays** — rank
once per ~30M prefixes per hour, merge cheaply 2.4B times per day; it loses freshness
granularity on the stable index, which dive 2's hot overlay is what buys back.

## 5. Latency & the p99 that is the product

Budget assembly (est. 01): p50 ≤ 30 ms server-edge; p99 ≤ 100 ms. Server compute < ~1 ms;
everything else is queueing + network + the rare cache miss.

- **Client behavior is the first tier**: debounce collapses bursts; an in-flight request for
  the same session is *cancelled* on the next keystroke (HTTP/2 multiplexing + client-side
  abort) — a request nobody needs is the purest p99 pollution.
- **Refresh-ahead for hot prefixes**: per-prefix cache entries near TTL are refreshed
  *before* expiry by a background fetch — the p99 payer is never the first request after a
  cache expiry (thundering-herd cousin of
  [10.distributed-cache](../10.distributed-cache/04-deep-dives.md) §4). **Trade-off**:
  lazy expiry (alternative) costs no extra fleet load but makes the first post-expiry
  request — usually a hot prefix's worst moment — the p99 payer; refresh-ahead (chosen)
  buys guaranteed-warm expiry for ~one background fetch per hot prefix per TTL per replica.
- **Timeout-and-degrade**: per-tier timeout (edge 10 ms, cache 3 ms, index 25 ms
  est.-shaped); miss the *last* tier → return empty immediately. An empty box costs the user
  a keystroke; a 400 ms suggestion list costs the whole interaction.
- **Tail diagnostics**: the request is 1 hash + 1 merge; if p99 > budget, the cause is always
  organizational (GC pause on an index node, hot-key eviction, cache TTL cliff, network RTT)
  measure per tier, or you will tune the wrong thing.

**Failure modes**: GC pauses on RAM-heavy nodes → heap discipline (segments off-heap or
compact objects) + per-node load-shed; replica failover mid-flight → coalescing + client
cancel hides it; a *regression* in normalization that shard-hashes differently → the shard
map must be built from the *same* normalizer, tested with the same fixtures (a classic
"works on the dev corpus" bug that sends "ne" to a shard that has never heard of it).

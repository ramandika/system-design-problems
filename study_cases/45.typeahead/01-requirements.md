# 01 — Requirements

## Functional

Core v1:

- **Prefix completion**: given a (debounced) prefix, return the top ~8 candidate queries,
  ranked by blended popularity/trending/personalization, with the typed substring *positioned
  mid-string* ("yt" → "youtube") not just as a literal prefix — mid-string matches come from
  a tokenized index (deep dives §1).
- **Input pipeline**: raw query logs (post-search, with locale/vertical/device) →
  normalization (case, accents, whitespace, dedup) → weighted counts with recency decay →
  materialized top-k per qualifying prefix.
- **Trending/boost layer**: recent-interval velocity counts applied as a score multiplicative
  boost; guardrails cap per-term and per-source velocity (deep dives §2).
- **Personalization**: user's own recent queries/history layered on top of the global answer
  (deep dives §4) — client-visible only; nothing about *this user* enters the global index.
- **Policy filtering**: offensive/unsafe/misleading candidate strings are *stopped in the
  ingestion pipeline* (blocklists, classifier gates at build time) — the online path never
  evaluates policy beyond cache-level exceptions.
- **Spell-correct adjacency**: for prefixes with too-few results, fall back to a fuzzy
  (edit-distance-1) match on the normalized corpus — bounded to the tail path, never the
  hot path.
- **Degradation**: on service failure return *nothing* fast (a fast empty box); never a
  slow good suggestion list (deep dives §5).

Out of scope v1: results themselves (this is suggestions only, → search system), vertical
entities (products/people entities in the dropdown — the POI-store retrieval shape in
[26.google-maps](../26.google-maps/README.md) is the nearest kin, though autosuggest proper
is not covered there), A/B framework.

## Non-functional

| Property | Target | Consequence |
|---|---|---|
| Suggestion latency | p50 ≤ ~30 ms, p99 ≤ ~100 ms edge-to-server | RAM-resident index; no per-request ranking model on the hot path; deep dives §5 |
| Client debounce | fire after ~50–100 ms idle, min-2-chars | removes roughly half of raw keystroke requests (:3 debounced of ~6 candidate keystrokes, est. 02; matches deep dives §3) and hides jitter |
| Freshness | batch rebuild ~hourly; trending boost minutes-to-sub-minute | two-tier freshness: stable index + hot layer (deep dives §2) |
| Availability | suggestions fail open→empty; search itself is unaffected | suggestions are an accelerator, never a blocker |
| Consistency | per-prefix top-k is a point-in-time snapshot; user history read-your-writes | derived data everywhere; there is nothing to transact on |
| Scale | ~2.4B requests/day (est.), 28K QPS avg / ~85K peak | tiny by storage, spiky by Zipf — see 02 |

## Scope decisions that shape everything

1. **Compute top-k at build time, not query time**: the query pattern is "same prefix,
   millions of times" — the read:write asymmetry (est. sheet: ≥ ~10^3:1, ~10^8 reads per
   rebuild event) makes every byte of
   query-time CPU waste indefensible; the hot path is a *point lookup*, not a search.
2. **Two freshness layers with different cache lifetimes**: a stable index (hours) plus a
   min-fresh trending overlay — because the cacheability demands are contradictory in one
   structure.
3. **Tail prefixes are handled by fallback, not by completeness**: the materialized index
   covers prefixes with meaningful counts; rare prefixes get fuzzy/empty answers and are
   *not* worth serving infrastructure (deep dives §1).

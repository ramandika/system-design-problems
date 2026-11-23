# 01 — Requirements

## 1. What it is

A general-purpose web search engine: the back half of
[Case 41 — Web Crawler](../41.web-crawler/README.md). Its inputs are the content records,
link records, and freshness signals that crawler's handoff stream emits; its outputs are
ranked result sets over the whole web corpus, served to users at hundreds of billions of
queries a year. Three jobs in one: **indexing** (turn the stream into a queryable inverted
index), **serving** (turn a query into candidate documents and merge them across
thousands of shards inside 200–600 ms), and **ranking** (turn candidates into an ordered
list worth clicking).

## 2. Functional requirements

- **FR1 — Query understanding**: parse the raw query string: tokenize, normalize
  (case, whitespace), detect language, detect intent hints (navigational, local,
  question-like), and correct apparent misspellings — with the corrected query presented
  ("showing results for X") and the original still allowed. Suggest-as-you-type shares
  infrastructure with [Case 45 — Typeahead](../45.typeahead/README.md) and is out of scope
  here except at its seam (a full query prefix → popular queries, FR5).
- **FR2 — Retrieval**: given the parsed query, find candidate documents in the inverted
  index across all shards. Must handle: conjunction (all terms), OR fallback in the
  spelling/broadening path, phrase queries (positional postings), quoted exact-match, and
  negation. Must support early termination: returning top-K is the contract, not returning
  *all* matches ranked.
- **FR3 — Ranking**: order candidates by predicted relevance/utility using a multi-stage
  pipeline: cheap index-time scores (retrieval), a coarse ML ranker on thousands of
  candidates, a heavy ML ranker (neural, cross-attention: query-vs-document similarity)
  on the top hundreds (same funnel as the [Ranking System](../15.ml-ranking-system/README.md)).
  Signals: text match, static document quality/query independency (link-derived), user
  interaction aggregates, freshness, and page-level features from the page store.
- **FR4 — Results page (SERP) composition**: for each top-result, render a *snippet* —
  the ~2–3 lines from the document that best match the query, not the document's title
  tag — plus URL, title, and cached-page link metadata. Snippets are query-dependent, so
  either generated at query time from stored text with query-term highlighting, or served
  from a query-aware snippet cache.
- **FR5 — Freshness tier**: recent content (news, breaking events, blog churn) must be
  findable in minutes-to-hours, not after the base-index cycle. Concretely: the fresh
  index layer is queryable side-by-side with the base index and its documents rank
  competitively with older, more authoritative pages when freshness matters. Supply side
  (which pages get re-crawled at what cadence) is [Case 41's deep dive 6](../41.web-crawler/04-deep-dives.md);
  the *serve-side* consequence is what's in scope here (deep dive 3 below).
- **FR6 — Vertical results**: compose the SERP from vertical sub-systems: news, images,
  video, places, shopping. Web results remain the main lane here; image/video/news ranking
  internals are separate indexes and separate problems. In scope: the architecture must
  fan out to verticals in parallel with the web-index retrieval and merge, and degrade the
  SERP gracefully when a vertical is slow or down.
- **FR7 — Spelling and query completion**: detect "did you mean" candidates from the
  query stream itself (queries are a huge implicit corpus of corrections) plus an
  edit-distance/prefix index; apply at query time both as suggestion and as automatic
  reformulation when confidence is high.

## 3. Non-functional requirements

- **NFR1 — Latency budget**: end-to-end p50 ≤ ~200 ms, p99 ≤ ~500 ms, p99.9 ≤ ~1 s from
  query receipt to first paint-ready payload (estimates for a consumer search product;
  real published figures are vague, but publicly-engineered goals cluster here).
  Decomposition in [02-scale-estimates](02-scale-estimates.md) §7; the tail behavior is
  deep dive 4.
- **NFR2 — Availability under degradation, never blank**: search is a read-only system;
  the right degradation is not "error page" but "fewer/less-recent results, returned
  fast". Concretely: if X% of index shards are unresponsive, return partial results
  (from the responsive ones) above a configured threshold rather than time out; if the
  heavy ranker is down, fall back to the coarse ranker's ordering; if a vertical is down,
  omit it. Target ~99.9%+ availability with graceful degradation; full-outage behavior is
  the same architecture with more of it missing.
- **NFR3 — Scale**: ~30,000 queries/s sustained average (arithmetic in
  [02-scale-estimates](02-scale-estimates.md) §5), diurnal peak ≈ 2×, burst events ≈ 10×
  overall with single hot queries spiking 10–100× (a big news moment), which the query
  cache absorbs more
  than the shards do. Horizontally scalable: no central index, no central ranker, no
  global lock.
- **NFR4 — Index freshness SLA**: high-churn tier pages are searchable within ~minutes
  of being crawled; the base index refreshes on a segment-merge cadence (hours-to-days
  for a given document). Freshness is a per-tier SLA, not corpus-wide.
- **NFR5 — Cost per query**: CPU-seconds per query is a first-class metric (the
  arithmetic in [02 §6](02-scale-estimates.md) shows it is the binding constraint, not
  latency). Every architectural choice — pruning, caching, shard count, model size —
  is evaluated against it.
- **NFR6 — Consistency model**: eventual, seconds-scale. A newly crawled document is
  missing from search for its tier's interval; a deleted document lingers in the index
  for the same interval. No transactional guarantees are given or needed; the *index
  build* is at-least-once from the crawler stream and idempotent by document ID
  ([13.kafka](../13.kafka/04-deep-dives.md) delivery semantics).
- **NFR7 — Ranking determinism/operability**: ranker models and their feature definitions
  are versioned artifacts (same discipline as the [Production ML System](../14.ml-production-system/README.md)):
  canaried, rolled back, and auditable; A/B ranking experiments are config-driven. A
  ranking regression is a product incident and needs a revert path measured in minutes.

## 4. Scope decisions

- **In scope**: inverted index build and layout; index sharding and query fanout; tiered
  index for freshness; retrieval + multi-stage ranking; snippet/page store; query and
  result caching; tail-latency mitigation; spelling correction and verticals at the seam
  (composition, not internals).
- **Out of scope (explicit)**:
  - **Crawling and fetch politeness**. Handled entirely in
    [Case 41](../41.web-crawler/README.md); this case consumes its handoff stream as
    given, including its freshness-tier scheduling.
  - **Ads ranking internals**: an auction over paid inventory with a completely
    different objective and latency shape; only the *composition* (ad slots occupy real
    estate in the SERP) is acknowledged.
  - **Personalization privacy policy**: what user signals may feed ranking is a policy
    problem; the architecture note is simply that personalization runs as a cheap
    post-rank reordering over features that stay server-side.
  - **Content quality/anti-spam engineering** as a full case: it is acknowledged as a
    signal in the ranker (deep dive 5), not designed.
  - **JavaScript-rendered / SPA content**: inherited from the crawler's stance
    ([41 §4 out-of-scope](../41.web-crawler/01-requirements.md)); such pages index
    whatever static content was extractable.
  - **Image/video/news ranking internals**: separate media indexes, separate retrieval
    physics; only their fan-out/merge composition is in scope (FR6).

## 5. The key tension, stated once

Everything trades against **cost per query (NFR5)** against **latency (NFR1)** and
**freshness (NFR4)**. A bigger index, more shards, deeper posting-list reads, a heavier
ranker, and more complete merging are each individually better for result quality — and
each is quadratic-ish in money. The architecture in [03](03-architecture.md) is organized
around one idea: spend cheap approximate effort on everything (pruned retrieval, cached
queries, coarse ranking) and expensive exact effort only where the top-10 results can be
changed by it.

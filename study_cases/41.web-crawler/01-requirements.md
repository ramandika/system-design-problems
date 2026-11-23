# 01 — Requirements

## 1. What it is

A continuously running system that (a) starts from a seed set plus the URLs its own
discoveries generate, (b) fetches pages over HTTP at a target scale, (c) obeys what each
host permits, (d) removes duplication, and (e) delivers extracted text, links, and metadata
to the indexing/search stack. It is infrastructure, not a product: its "users" are the
indexer, the link graph builder, and the freshness pipeline.

## 2. Functional requirements

- **FR1 — Fetch**: given a URL, download the HTTP response (status, headers, body) for
  HTML resources. Honor redirects (up to a small hop limit), HTTP ranges where useful,
  compression, and conditional requests (If-Modified-Since / ETag) on recrawls.
- **FR2 — Discover**: extract hyperlinks, including sitemap URLs and feed URLs, normalize
  them (resolve relative links, canonicalize), and feed new ones back into the frontier.
- **FR3 — Obey**: consult robots.txt for every host before fetching, honoring its
  disallow rules, crawl-delay, and sitemap directives. Send an honest, contactable
  User-Agent. Per-host: no concurrent request exceeds the host's stated budget.
- **FR4 — Extract**: from each fetched page produce a record: main content text (with
  boilerplate — nav, ads, cookie banners, repeated templates — separated), outgoing links
  with anchor text, metadata (title, canonical link tag, published/modified dates,
  language, content type), and a set of signatures for dedup.
- **FR5 — Dedup**: never store the same content (exact or flagged-near-duplicate) twice in
  the corpus; never enqueue the same canonical URL twice concurrently; identify pages that
  are near-copies of an existing indexed page.
- **FR6 — Recrawl**: maintain a per-URL next-attempt schedule, honoring each page's
  observed churn rate and importance; re-fetch changed pages and drop dead ones.
- **FR7 — Handoff**: stream extracted records downstream in a format the indexer can
  ingest, with at-least-once delivery and per-record replay from stored blobs
  (see [13.kafka](../13.kafka/04-deep-dives.md) for the delivery-contract reasoning).

## 3. Non-functional requirements

- **NFR1 — Throughput**: ~4 billion pages/day (arithmetic and derivation in
  [02-scale-estimates](02-scale-estimates.md)), which is roughly 46,000 fetches/s
  sustained, peaking ~2.5× that. The system should scale horizontally; no fixed
  ceiling where the frontier, dedup, or fetch capacity is a distinct bottleneck.
- **NFR2 — Politeness first**: politeness constraints outrank throughput. It is better to
  miss a crawl target than to violate a site's stated limits or degrade its service.
  Concretely: per-host request rate never exceeds (robots crawl-delay or internal default),
  robots.txt is re-fetched rather than assumed, and 429/503 responses with Retry-After
  are honored rather than retried through.
- **NFR3 — Freshness tiers**: important, fast-churning pages are current within minutes to
  a day; the long tail is current within the ~30-day refresh cycle. "Fresh" is a per-tier
  SLA, not a corpus-wide one.
- **NFR4 — Bounded cost**: storage of the corpus must remain proportional to *useful*
  pages after dedup and boilerplate removal, not to pages discovered. Bandwidth per
  stored-page is a tracked efficiency metric.
- **NFR5 — No host poisoning**: a single misbehaving host (slow responses, huge bodies,
  redirect loops, dynamic-content-forever pages) must not eat the crawl budget or the
  fetch fleet's connection capacity. Timeouts, size caps, and per-host quotas everywhere.
- **NFR6 — Recoverability**: the frontier and scheduling state must survive component
  restarts without mass re-fetching or mass forgetting. Lost fetch results should cost a
  re-fetch; lost scheduling state should mean a fresh start, not a duplicate corpus.
- **NFR7 — Operability**: per-host and per-priority visibility: what is queued, what is
  delayed for politeness, what is failing. Crawl-rate changes (add a host tier, throttle a
  misbehaving datacenter partner) must be configuration, not code.

## 4. Scope decisions

- **In scope**: HTML pages. Sitemaps, feeds, and robots.txt as input sources. Text
  extraction to the point the indexer can start. Per-URL scheduling state. Content
  fingerprinting for exact and near-duplicate detection (deep dive 2).
- **Out of scope (explicit)**:
  - Rendering JavaScript to fetch DOM-only content. A JS-rendering pass is a separate,
  much more expensive pipeline; the base crawler covers what static HTML exposes.
    (Failure consequence: SPA pages appear empty; the indexer sees this as sparse
    content, not missing content.)
  - Binary asset indexing (images, video). The crawler may discover their URLs, but an
    image search crawler is a separate fleet with its own politeness and dedup problems.
  - The search index, ranking, and the query path themselves. The crawler hands off
    records and stops.
  - Authenticated/paywalled fetching, dark-web sources, and any crawling that bypasses a
    host's access controls. Politeness (NFR2) is a hard boundary, not a preference.
  - Adversarial detection of hostile crawler traps (infinite URL spaces) beyond cheap
    heuristics — deep dive 3 covers the per-host budget that contains them.

## 5. The key tension, stated once

Every requirement above trades against throughput (NFR1). Politeness (NFR2) is load-bearing
and cannot be made a soft signal; dedup (FR5) determines whether NFR1 means 4B fetches/day
or 12B, and whether the corpus is ~60 PB or ~180 PB a year; scheduling (FR6) decides what fraction
of NFR1 is spent on content that already exists unchanged. The architecture in
[03](03-architecture.md) is organized around making each of those three a first-class
component rather than an optimization.

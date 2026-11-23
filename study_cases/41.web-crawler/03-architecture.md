# 03 — Architecture

## 1. The one-paragraph picture

The crawler is a pipeline of five loosely coupled stages, joined by durable stores and
queues, not direct calls: **URL intake → URL frontier (scheduler) → fetcher fleet →
extraction → downstream handoff**, with two cross-cutting state services — the **URL/content
table** (the truth of record: every URL known, its status, next-attempt time, fingerprints)
and the **frontier seen-set** (hash-level duplicate suppression). Nothing in the hot path
holds a lock on "the crawl" globally; everything is per-host or per-shard.

## 2. Diagram

```
                  ┌──────────────────────────────────────────────────┐
                  │                 FRONTIER SCHEDULER               │
  seeds/sitemaps  │  ┌───────────┐    ┌─────────────┐   ┌─────────┐  │
 ────────────────▶│  │ FRONT     │    │ BACK QUEUES │   │ seen-set│  │
  discovery feeds │  │ QUEUES    │───▶│ (1+ / host, ├──▶│ (hash   │  │
  link extraction │  │ (priority │    │  heap by    │   │  dedup) │  │
        ▲         │  │  tiers)   │    │  next-      │   └─────────┘  │
        │         │  └───────────┘    │  eligible)  │                │
        │         └──────────────────┼─────────────┴────────────────┘
        │                           │  work units (per-host batches)
        │                           ▼
        │                  ┌────────────────────┐      ┌────────────┐
        │                  │  FETCHER FLEET     │◀────▶│ robots.txt │
        │                  │  (stateless HTTP   │      │  cache     │
        │                  │   workers + DNS    │      │ (24h TTL,  │
        │                  │   cache + conn     │      │ per-host)  │
        │                  │   budget per host) │      └────────────┘
        │                  └─────────┬──────────┘
        │                            │ fetched blobs + headers
        │                            ▼
        │                  ┌────────────────────┐
        │                  │  EXTRACTION        │
        │                  │  charset/html parse│
        │                  │  boilerplate strip │
        │                  │  fingerprinting    │
        │                  │  link extraction   │
        │                  └─────┬───────┬──────┘
        │                        │       │
        │        links (normalize│       │ content records
        │        + canonicalize) │       │ (text+metadata+signature)
        └────────────────────────┘       ▼
                                   ┌──────────────────────┐
   ┌────────────────┐              │  HANDOFF             │
   │ URL/content    │◀─────────────│  dedup gate (SimHash │
   │ TABLE          │  write       │  vs corpus)          │
   │ (bigtable-style│  status/     └─────────┬────────────┘
   │  status+fp)    │  sched                 │
   └────────────────┘                        ▼
                                    ┌───────────────────┐
                                    │  STREAM (kafka-   │──▶ indexer
                                    │  style partitioned│──▶ link-graph builder
                                    │  at-least-once)   │──▶ freshness scheduler
                                    └───────────────────┘   (writes back to URL table)
```

## 3. Components

### 3.1 URL intake and intake normalization

Seeds, sitemaps, feeds, and every link extracted by 3.5 feed into one path. Before
anything is enqueued: parse URL, resolve relative references, apply canonicalization rules
(deep dive 2, part 1 — lowercase scheme/host, strip fragments and tracking params, default
port, HTTPS/http unification per-host policy), produce a canonical hash. Intake is a
stream: it never blocks the fetcher pipeline. Volume: ~40 B URLs/day arriving
(§2 of 02-scale-estimates).

### 3.2 URL frontier — Mercator-style two-level queue

The frontier is the core scheduling mechanism (deep dive 1):

- **Front queues (priority)**: a small set of ordered queues; a URL enters the front queue
  matching its priority (importance × freshness tier). Higher-priority front queues are
  polled more often.
- **Back queues (politeness)**: a large set of queues, each bound to a single host, holding
  URLs ordered by their next-eligible time. A per-host heap ensures the host is fetched at
  its politeness rate and never ahead of it.
- **Connector**: pops a work unit from a back queue whose host is eligible, preferring the
  one whose front-priority is highest. This is the two-axis composition point.

State lives on disk (12 TB of frontier records, §4 of 02-scale-estimates); only a working
set is in RAM. The frontier is sharded by host so a scheduler node owns a contiguous set
of hosts — no cross-node coordination in the hot path.

### 3.3 Fetcher fleet

Stateless HTTP workers: they receive a work unit (batch of URLs for one host), check the
robots cache, enforce the per-host connection cap, and fetch. Their only state is the
robots cache and DNS cache (both caches are read-mostly, refreshed concurrently with use).
They enforce NFR5 mechanically: per-request timeout, max body size, max redirect hops,
per-host concurrency ceiling, Retry-After honoring. Fetchers don't decide priority or
freshness — they execute it.

### 3.4 robots.txt cache

A per-host parsed-rules cache with a TTL (24 h) plus a per-host "last fetched at" floor
(refetch at most every N minutes even if TTL expired, so a flaky robots endpoint can't
stall everything). Cache miss = serialize the fetch of robots.txt before any page fetch on
that host. Served by a small fleet of robots workers; the page fetchers query them.

### 3.5 Extraction

Given a fetched blob: charset detection, HTML parse, boilerplate strip (identify repeated
block-level templates across the site; strip nav/ads/session chrome), main-content
extraction, outgoing link extraction, metadata capture (canonical link tag feeds back into
intake canonicalization; published/modified dates feed the freshness scheduler).
Output: a content record + dedup signatures (64-bit SimHash of the shingled token stream;
MinHash sketch for near-dup clustering — deep dive 2, part 2).

### 3.6 Handoff path

- **URL/content table** (bigtable-style, see [12.bigtable](../12.bigtable/03-architecture.md)):
  one row per canonical URL — status, last fetch, next attempt, priority tier, signature
  pointers, response-code history. Written by fetcher (status) and by extraction (record);
  read by scheduler (next attempt due?).
- **Dedup gate**: SimHash compared against a corpus index of existing signatures; exact
  matches discarded, near-matches flagged (does not block handoff, adds a flag).
- **Stream** (kafka-style partitioned log — see [13.kafka](../13.kafka/04-deep-dives.md)):
  content records and link records land in topics the indexer and link-graph builder
  consume. At-least-once; consumers dedup by URL+content-hash since the corpus is
  eventually consistent.
- **Freshness feedback**: the stream also carries signals (content changed, status 404,
  sitemap lastmod) which the freshness scheduler uses to update next-attempt times in the
  URL table — closing the loop for deep dive 6.

## 4. Data flows

1. **Discovery**: link extracted → canonicalized → seen-set lookup (skip if seen in-flight
   or very recently fetched) → front queue by priority → back queue for its host →
   eventually picked up by the connector.
2. **Fetch**: connector emits work unit per host (≤ politeness budget) → fetcher checks
   robots, resolves DNS, enforces budgets, fetches → writes blob + status to URL/content
   table → forwards blob to extraction.
3. **Extraction → handoff**: content record → dedup gate → stream to indexer. Links feed
   back into intake. Status/failure events go back to the URL table, which schedules
   retry/dead-letter.
4. **Recrawl**: freshness scheduler reads changed-pages/404/sitemap signals from the
   stream, computes per-URL next-attempt time, writes it into the URL table; the frontier
   picks URLs whose next-attempt ≤ now.

## 5. Failure handling at a glance

- **Fetcher box dies**: work unit is re-emitted by the connector (the frontier is the
  source of truth for what's in flight; see NFR6). Cost = a re-fetch, not lost data.
- **Frontier shard dies**: replay from the URL table (URLs with next_attempt ≤ now and not
  in flight). Frontier is a cache of scheduling state, not the truth itself.
- **Host goes down / robots unreachable**: fetcher backs off per host, marks host
  "degraded" in frontier; scheduler drains its back queue slowly rather than retrying hot.
- **robots.txt fetch fails**: apply last-known-good robots, capped at a conservative
  default (crawl-delay ≥ 10 s or stop crawling that host), never an assumption of
  permission. (Deep dive 3.)
- **Extraction overload**: buffered stream between fetcher and extraction; fetchers
  continue polite fetching while extraction catches up.

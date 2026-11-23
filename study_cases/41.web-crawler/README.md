# Case 41 — Large-scale web crawler

**The system**: a distributed crawler that fetches billions of pages per day, respects how
each target host wants to be treated, keeps the corpus fresh, and hands clean extracted
content to the indexer. The input is "the web" — a graph nobody owns — and the output is a
corpus that a search stack consumes. Every hard part comes from the same tension: the
crawler wants maximum fetch throughput, and every host it touches wants the opposite.

**Why it's hard**:

- The URL frontier is the heart: an ordered collection of tens of billions of URLs where
  two orders fight each other — "fetch the most valuable page next" (priority) and "fetch
  at most N pages per host per minute" (politeness). A greedy priority order hammers one
  host; a pure per-host round-robin wastes crawl budget on junk. Mercator-style front and
  back queues are the classic answer.
- Politeness is not optional. A crawler that ignores robots.txt, crawl-delay, or server
  load becomes abusive traffic — the same category as hostile crawler traffic in
  [09.cdn](../09.cdn/04-deep-dives.md) — and can take down small sites. And robots.txt at
  scale is a state-management problem: hundreds of millions of small files, each with a
  different set of disallow rules and delays.
- Everything is duplicated. The same URL appears under many spellings (http/https, domain
  vs subdomain, session IDs, cache busters); the same *content* appears at thousands of
  URLs (syndicated articles, boilerplate templates). Dedup has to run before you spend
  bandwidth, after you spend it, and in two different senses (identical vs near-identical).
- Freshness never ends. Pages churn at wildly different rates — a news front page changes
  hourly, a 2011 blog post never. Recrawl scheduling decides where a fixed crawl budget
  goes, and getting it wrong means either a stale corpus or recrawling dead pages forever.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: two-axis scheduling (priority × politeness) as an example of
orders that cannot be flattened into one sort — kin to per-key ordering and its limits in
[Kafka](../13.kafka/04-deep-dives.md); hash-based dedup with approximate structures where
exact is either too big or too slow; per-resource budgets (connections per host, robots
cache, DNS cache) as the distributed-systems version of rate limiting; and a work
distribution where state ownership is explicit — the frontier owns "what's left", the
URL/content tables are the truth of record, in the same spirit as
[12.bigtable](../12.bigtable/03-architecture.md). Storage-wise, a corpus of compressed
page blobs is a write-once workload that [07.blob-store](../07.blob-store/02-scale-estimates.md)
reasons about in detail.

**Try it first**: before reading 03, answer: (1) A single host holds 10M URLs in your
frontier and you want each of them crawled; what stops the polite answer from starving
everything else? (2) You discover ~40B URLs/day but only want to fetch 4B — where exactly
in the pipeline does the 10× drop happen, and what does filtering at the wrong stage cost?
(3) robots.txt fetches are 1 KB but pages are 150 KB — why does the small thing deserve
its own cache and its own failure mode?

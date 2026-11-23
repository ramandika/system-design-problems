# 02 — Scale estimates

All numbers below are estimates for a top-tier general-purpose crawler. Real numbers in the
industry are unpublished; public data points worth keeping in mind: Google has stated its
index contains "hundreds of billions of pages"; Common Crawl's public crawls fetch
~2–3 billion pages per month-long run; publicly reported crawl-delay values cluster around
1–10 s. Everything else here is derived arithmetic, clearly labeled.

## 1. Corpus and crawl rate

- **Corpus we aim to stay current**: ~100 billion pages (consistent with "hundreds of
  billions" being the ceiling of a major index; we target the low end).
- **Full refresh cycle**: 30 days (product choice: search results should be at most ~a
  month stale for the tail; deep dive 6 revisits this).

```
pages/day (baseline full sweep) = 100e9 / 30 = 3.33e9 ≈ 3.3 B pages/day
```

- **Delta re-crawl budget on top** for the high-churn tier (deep dive 6): assume 1% of the
  corpus (~1B pages) is high-churn/high-value and is re-checked on a ~few-day cycle; budget
  ~0.7 B/day for it after dedup and failure-recheck credit.

```
total crawl target = 3.3 B (full sweep) + 0.7 B (delta) ≈ 4 B pages/day
```

## 2. Fetch throughput

```
avg fetch rate  = 4e9 / 86400 s ≈ 46,000 pages/s   (call it ~40–50 k/s)
peak (2.5× for diurnal + retry storms)   ≈ 100–120 k fetches/s
```

Fleet sizing (estimate): what bounds one fetcher box is not hardware but politeness — an
active host contributes only ~0.1 req/s (§6), so a box assigned ~3–6 k active hosts does
~300–600 fetches/s, sitting as only ~60–120 concurrent connections at a ~200 ms p50
(Little's law: concurrent ÷ latency; fetchers are I/O-bound anyway, CPU goes to
extraction). Roughly **80–150 fetcher machines** at the average rate
(46,000 ÷ 300–600 ≈ 77–153), a few hundred when sized for the 2.5× peak.

## 3. Bandwidth

Mean downloaded HTML body: median ~50 KB, long tail of large pages, including some HTML-only
CDN-served resources → assume **mean 150 KB** (this is HTML only; a rendered page's total
resource weight is an order larger, but we don't fetch those).

```
bandwidth = 46,000 pages/s × 150 KB ≈ 6.9 GB/s ≈ 55 Gbps sustained
peak      ≈ 120–145 Gbps
```

Per-day volume (before compression):

```
4e9 pages/day × 150 KB = 600 TB/day raw (4e9 × 1.5e5 B = 6e14 B)
```

- Compression (gzip/brotli on transfer; storage compression 3–4×) → ~150–200 TB/day stored.
- Roughly **55–73 PB/year** compressed (150–200 TB/day × 365) — this is why dedup (FR5) and boilerplate removal are
  cost controls, not niceties. The write-once, largely-append characteristics of this blob
  store are the same shape as [07.blob-store](../07.blob-store/02-scale-estimates.md).

## 4. URL discovery and the frontier

Real crawlers see far more URLs than they fetch. Assume discovery of **~10× the fetch
target** — 40 B URLs/day emitted into the pipeline — from links, sitemaps, and redirects
(redirect chains alone multiply: a redirect is ~1 emitted URL per hop).

- **Filter rate**: URL-level filtering (robots disallow, duplicate canonicals, seen-before,
  spam heuristics) removes ~80–90% before a fetch. 4B fetched of 40B discovered = the 10×
  drop. Where this happens (before vs. after the polite queue) matters operationally
  (deep dive 1).
- **Frontier state**: the "unfetched, still-candidate" set can be large — tens of billions.
  At ~300 B/canonical-URL record (URL text + priority + next-attempt time + host id):

```
40 B candidate URLs × 300 B = 12 TB frontier records  (disk-backed, RAM only has the hot set)
```

- **Seen-URL set** (never re-enqueue in-flight; suppress duplicates across the whole
  history): ~500 B entries cumulatively, stored as fixed-size 128-bit canonical hashes:

```
500e9 × 16 B = 8 TB seen-set hashes
```

This is a key-value workload with range scans by host — the
[12.bigtable](../12.bigtable/03-architecture.md) access pattern; we cross-reference the
design there instead of repeating it.

## 5. Hosts, robots.txt, and politeness state

- Publicly reported web surveys put hosts in the hundreds of millions; Common Crawl's
  per-crawl stats list on the order of tens of millions of crawled domains per run (varies
  by crawl). Assume **~50–100 M hosts** (an estimate) where the crawler holds per-host state (last-fetch time, crawl-delay,
  connection budget, error history, back-queue assignment).

```
5 KB/host (robots parsed rules + scores) × 75 M hosts ≈ 375 GB (mostly cold, disk-backed)
```

- **robots.txt fetches**: with a 24 h TTL on parsed robots, ~75 M hosts →
  ~75 M / 86400 ≈ **~1,000 robots fetches/s**. Each is ~1–2 KB — tiny payload, but it is
  a mandatory serialized step before any page fetch on that host, which is why robots is
  its own cache layer (deep dive 3).

## 6. Politeness throughput ceiling

Default internal budget: ≤1 request/10 s per host unless crawl-delay says otherwise (deep
dive 3 explains the default). The polite ceiling on throughput is then a function of
concurrency across hosts, not per-host speed:

```
if ~1 M hosts are "active" in a given hour, each allowing ~0.1 req/s
   → 100 k fetches/s theoretical ceiling  — only ~2× the 46 k/s target
```

Conclusion: politeness is not the binding constraint at scale if the frontier spreads
across enough hosts (the ceiling scales linearly with active-host count, ~1 M fetches/s
at ~10 M active hosts) — **frontier depth per host is**. A host with a 10M-URL backlog can
starve everything else if scheduling doesn't mix hosts (deep dive 1).

## 7. DNS

- Assume ~60–80% of fetches hit an in-process DNS cache (TTL-resident, ~200 k hot hostnames
  in RAM at 100 B/entry = ~20 MB — trivially cacheable).

```
DNS lookups = 46,000 fetches/s × (1 − 0.7 hit rate) ≈ 14,000 lookups/s
```

- A small cluster of resolvers with aggressive negative-caching; the failure mode
  (resolver outage stalls the whole fleet) is worth separate treatment (deep dive 4).

## 8. Storage of results

```
150 TB/day stored blobs + extracted records for 4 B fetched pages/day
                          (URL-table row writes are for all ~40 B discovered URLs/day,
                          ~100 B record either way) + dedup signatures
                                          ≈ 5–10 TB/day metadata + signatures
```

Arithmetic: 4 B × 100 B = 0.4 TB/day of extracted records, ~4 TB/day of URL-table
bookkeeping across discovered URLs, ~4.4 TB/day of signatures — land in the 5–10 TB/day
band together (estimate).

- Dedup signatures: 64-bit SimHash (8 B) stored for every page, plus a ~128×8 B MinHash
  sketch (~1 KB) for SimHash-flagged candidates only (deep dive 2, part 2) → ~1.1 KB max
  per signature-carrying page → at 4 B pages/day, ~4.4 TB/day of signature data is the
  upper bound if every page carried the full sketch.

## 9. Summary table

| Quantity | Estimate | Derived from |
|---|---|---|
| Pages/day (crawl target) | ~4 B | 100 B corpus / 30 d + 0.7 B delta |
| Fetch rate | ~46 k/s avg, ~100 k/s peak | ÷ 86,400 s, × 2.5 peak factor |
| Fetcher fleet | ~80–150 boxes (avg; ~2–3× for peak) | ~300–600 fetches/s/box |
| Sustained bandwidth | ~55 Gbps | 150 KB mean × 46 k/s |
| Raw volume | ~600 TB/day | pages/day × mean size |
| Stored (compressed) | ~150–200 TB/day → ~55–73 PB/yr | 3–4× compression |
| Frontier records | ~40 B live; ~12 TB | 10× discovery, 300 B/record |
| Seen-set hashes | ~500 B; ~8 TB | 16-byte hash per canonical |
| Hosts with state | ~50–100 M | public surveys ~100 M+ PLDs |
| robots fetches | ~1 k/s | 75 M hosts / 24 h TTL |
| DNS QPS | ~14 k/s | 30–40% miss rate |

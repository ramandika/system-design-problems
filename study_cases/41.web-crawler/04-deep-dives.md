# 04 — Deep dives

## 1. URL frontier: priority vs. politeness (Mercator-style queues)

The frontier must serve two orders at once. **Priority** says: fetch what is most valuable
next — a page linked from many authoritative pages, on a host we're trusted to crawl
heavily, in a freshness tier the product promised. **Politeness** says: never more than a
host's allowed rate. A single global priority queue flattens both into one axis and always
loses one of them.

**Option A — one global priority queue**:

- wins: trivially simple; nothing is starved by host backlog; peak throughput is easy.
- loses: nothing prevents two adjacent entries pointing to the same host, so the crawler
  hammers that host at full fleet speed — a politeness violation on every such pair. To
  fix it you'd have to drop "global priority" and re-inspect the head of the queue per
  fetch, which is O(frontier) unless you add a per-host index anyway.

**Option B — pure politeness-first round-robin over hosts**:

- wins: politeness never violated; simple per-host heaps; each host served independently.
- loses: fairness inside a host is FCFS, not value-ordered — a host with a 10 M-URL
  backlog serves those URLs before any newly discovered, high-value page on the same host.
  Priorities are effectively ignored. Overall crawl value per hour drops badly.

**Option C — two levels (Mercator-style front/back queues), chosen**:

- back queues: one per host, ordered by next-eligible time, with a per-host heap so the
  scheduler pops always from the earliest-eligible eligible host.
- front queues: a small ladder of priority classes; when a URL arrives it's placed in both
  its front queue and its host's back queue. When a back queue is being fed into fetch,
  its order comes from front priority; when a host is picked to be served, it comes from
  the per-host heap.
- wins: both axes independent; politeness enforced locally at per-host heaps; priority
  enforced at front queues without touching politeness.
- loses: two stores for the same URL (the front queue bookkeeping is duplicated; the
  frontier has ~2× record count transiently); scheduling decisions are approximate — a
  high-priority URL whose host is rate-limited waits for the host regardless of how
  important it is, which is exactly the trade politeness asks for.

**Chosen: Option C.** The duplication cost is bookkeeping, not capacity (12 TB frontier
per §4 of 02-scale-estimates absorbs it), and the coupling loss in B is correctness of the
crawl's value, not speed.

**Trade on sharding**: the frontier is sharded by host range (frontier node owns a set of
hosts). Option A for sharding — hash by host — is what we pick: wins locality of per-host
state on one scheduler node, no cross-node coordination for politeness; loses global
priority ordering (a scheduler only knows its hosts' priorities). We pick A because
politeness correctness is a hard constraint (NFR2) and global priority is not — per-shard
priority is approximation enough.

**Failure modes**:
- A back queue for a host whose robots disallow everything grows unbounded → back-queue
  per-host cap, dead-letter at cap; the URL table records why.
- Priority inversion: a very important URL behind millions of lower-priority URLs in its
  back queue. Mitigation: front-queue priority also influences *when the back queue is
  opened* (an important host-level URL triggers an earlier fetch slot for that host, up
  to the host's politeness floor).
- Scheduler node restart with in-flight work units: connector re-emits from the URL table
  ("next_attempt ≤ now, not in flight"), so politeness state (per-host last-fetch) has to
  live in the frontier's durable store, not in RAM — otherwise a restart resets politeness
  and briefly hammers hosts. (Real incident class.)

## 2. Dedup at scale: URL canonicalization and content fingerprinting

Dedup happens at two different layers, for two very different questions. *"Have I fetched
this exact URL before?"* and *"Have I seen this content before, even at a different URL?"*

### 2a. URL canonicalization

The same page reachable as `http://example.com/a?x=1&y=2`, `https://example.com/a?y=2&x=1`,
`example.com/a`, `https://www.example.com/a/?x=1`, etc.

**Option A — exact string match**:

- wins: perfectly accurate for identical bytes; zero false positives.
- loses: catches almost nothing; discovery produces 5–20 spellings for the same target;
  they all get fetched, bandwidth and storage explode, and the index learns the same
  content at many URLs.

**Option B — a full canonicalization rule set** (lowercase scheme/host, strip default port
and fragment, sort query params, remove session id params, per-host canonical-link-tag
preference, HTTPS/http and www/apex unification):

- wins: removes most duplicates before any bandwidth is spent; deterministic and cheap
  (one pass on the URL string); aligns with what sites themselves declare via
  `<link rel="canonical">`.
- loses: rule sets are heuristic — aggressive rules merge genuinely distinct pages (e.g.
  stripping any param that looks numeric); a wrong merge loses content. Needs to be
  versioned and audited. Also can't detect duplicates whose URLs differ in ways rules
  can't generalize (hash-fragment routers, `?p=123` vs `/path`).

**Chosen: Option B**, with the rule set treated as versioned configuration (NFR7) and a
canary/audit process (sample merges, human review) because a silent bad merge is a
correctness bug with no error signal.

### 2b. Content fingerprinting: exact vs near-duplicate

**Option A — exact hash of the extracted text** (SHA-256 of the stripped content):

- wins: trivial, one 32-byte signature, O(1) lookup, zero false positives.
- loses: misses near-duplicates — syndicated articles, mirrored pages, boilerplate churn
  (an ad-slot rotation is one token change and the hash flips). In a web corpus, near-dup
  is the common case and exact-dup the rare case, so A detects maybe 20–30% of the useful
  duplicates. It also fails on template-level duplication (the same page skeleton with
  different content will look unique despite a 90% shared frame).

**Option B — MinHash + LSH**:

- wins: a principled probabilistic estimate of Jaccard similarity between shingle-sets;
  LSH banding turns candidate-pair finding into a sublinear bucket lookup rather than an
  all-pairs comparison. Good recall for true near-dups at meaningful similarity levels.
- loses: needs a real LSH index (buckets + a second-stage comparison pass per candidate),
  memory for the signatures, and it never fires on *small* shared fragments — a syndicated
  page sharing 20% with the source is invisible. Also false positives need a vetted
  threshold.

**Option C — SimHash (Charikar), a 64-bit fingerprint per page**:

- wins: each page is *one* 64-bit integer. Two pages' similarity is judged by Hamming
  distance of their fingerprints; near-dups are detected by bit-flips of nearby distance.
  Storage is 8 B/page (vs ~1 KB for a 128-shingle MinHash sketch) and comparison is a
  Hamming distance, not a Jaccard recompute or an LSH band probe. Works in small-memory
  settings: the index can be shard- or trie-organized so Hamming ≤ k lookups don't
  require a linear scan (Manku et al.'s near-dup detection approach).
- loses: it's a coarse signal — a page that shares 15% of its content with another page
  will look nearly identical; it doesn't give you the actual shared shingles (no
  explanation of *why* two pages match, so the audit pipeline has to recompute to show a
  human what matched); and SimHash needs a good token weighting (IDF-like) to avoid
  boilerplate dominating the fingerprint.

**Chosen: C as the first-pass gate, B as the confirm/cluster step.** SimHash is cheap and
covers the volume (4 B/day, ~4.4 TB/day of signatures per 02-scale-estimates §8);
candidates that pass SimHash (Hamming distance under threshold) are confirmed or clustered
with MinHash and shingle comparison — exact-match false positives at 64-bit get audited
before content is discarded. Option B alone (LSH over MinHash on everything) pays full
index cost on 100% of pages for signal that ~90% of pages don't need (most pages aren't
near anything).

**Failure modes**:
- Fingerprint collision → two unrelated pages flagged as duplicates → one is dropped from
  the index. Mitigation: threshold, plus exact-hash confirmation on the flagged pair;
  false-positive rate is a tracked metric, not a silent number.
- Boilerplate dominating the fingerprint → the same site's thousands of distinct pages
  all flag as near-dups of each other. Mitigation: boilerplate-strip before shingling, and
  weight terms by IDF (rare terms carry the fingerprint).
- Discovery-time (URL-level) dedup missed (§a fails) → fetcher bandwidth blows up;
  detection is a per-host "emitted vs actually new" ratio alert.

## 3. robots.txt compliance and crawl-delay honoring at scale

robots.txt is a per-host contract the crawler must honor. At ~50–100 M hosts, robots is not
"a file you fetch before the page" — it is a cache-management problem with correctness
consequences: ~1,000 robots fetches/s (§5 of 02-scale-estimates), a small payload, but
*mandatory serialization* before any page fetch on the host.

**Option A — fetch robots.txt at fetch time, no cache**:

- wins: never stale; never violates a host's current rules.
- loses: it doubles the request count per host and adds a serialized per-page latency; on
  hosts with crawl-delay of 10 s, the robots fetch competes with page fetches for the
  politeness budget, doubling the wait; at ~4 B robots requests/day (one per page fetch) it is not viable.

**Option B — cache robots.txt aggressively, long TTL (e.g. 24–72 h), no invalidation**:

- wins: cheap (robots fetch amortized to ~1 k/s); robots file rarely changes; the
  common case is correctness-preserving.
- loses: a host that *tightens* its rules (a site in trouble blocks the crawler) stays
  crawled at the old rules for up to TTL days — a real politeness violation. Also, a host
  that starts returning 404 for robots.txt under the old file's silence means "allow" by
  convention — but the actual file, if it were being served, may have said disallow.

**Option C — cache with TTL + explicit freshness hooks**:

- fetch on cache miss (default 24 h TTL, capped at some max like 48 h),
- proactive revalidation when the host's crawl is high-volume (top-hosts list polled more
  often, e.g. hourly), because that's where violation cost is largest,
- on robots fetch failure: apply last-known-good rules for a *bounded* grace window
  (say a few hours), then fall back to a conservative internal default (deep crawl-delay
  of e.g. 10 s or block that host), never an implicit "allow all",
- honor `Retry-After` on 503 and treat 429 rate-limit responses as a host-level signal to
  back off (even though 429 is not robots, it is a contract).

**Chosen: Option C.** It is B's cost with A's correctness guarantee within the grace
window. The conservative default-on-failure rule is non-negotiable (NFR2): the worst case
when robots is unreachable is a *throttle*, not a free pass.

The internal default crawl-delay when a host says nothing: **≤1 request/10 s** per host.
Sites commonly set crawl-delay 1–10 s in robots.txt (publicly reported values cluster
there); we honor the host's value if more restrictive, and never exceed our default
otherwise. Per-host concurrency cap of (say) 2 connections regardless of rate — the rate
alone isn't enough (one long download blocking another politeness-slot is fine, but two
concurrent requests to a slow host is two simultaneous connections to an origin that might
be a Raspberry Pi). This is the "connection budget" line of deep dive 4 too.

**Failure modes**:
- robots.txt is served by a different infrastructure than the content (rare but real):
  a robots fetch succeeds while content fetch will fail, and vice versa. Mitigation: robots
  service is *not* allowed to mask content service signals — a host whose content fails
  hard gets marked degraded regardless of robots success.
- Infinite-disallow loops: a site serving robots.txt only to us, blocking everything, with
  content 200 OK — the crawl budget drains. Mitigation: per-host robots disallow ratio goes
  into a "don't spend more budget here" score.
- The classic incident: a crawler honoring robots but ignoring crawl-delay. This happens
  when the rate limiter lives in the fetcher rather than in the scheduler (deep dive 4).

## 4. Distributed fetcher: DNS caching, connection budgets, timeouts

The fetcher is a fleet of mostly stateless boxes that translate one work unit into tens of
HTTP requests. Three per-host resources dominate correctness: DNS, connections, and time.

**DNS.**

- A per-fetcher process DNS cache (TTL-respecting) plus a resolver cluster behind it.
  ~14,000 upstream lookups/s at the 70% hit rate from 02-scale-estimates §7. Negative
  caching matters as much as positive — a host that fails DNS resolution should have its
  negative TTL honored rather than being looked up every fetch.
- **Option A — OS resolver per fetch process**: wins by simplicity; loses because every
  fetch box carries a full DNS stack and the resolver cluster sees a request per cache
  miss per box (amplification).
- **Option B — shared resolver fleet with in-process cache + prefetch** (chosen): wins on
  upstream load (single cache miss per hostname cluster-wide, then prefetch when TTL is
  about to expire, so the miss never lands on the fetch hot path); loses because now DNS
  is a service with its own SLA — resolver cluster outage stalls the fleet. Mitigation:
  serve from stale cache on resolver unavailability (DNS answers older than TTL are
  usually fine — hostnames on the web don't churn at minute granularity), and the per-host
  politeness delay provides a natural retry buffer.

**Connection budget per host.**

- Cap concurrent TCP connections per host (2) AND per host-pair-of-source-IPs (2 per
  source IP, so a fleet of N fetcher boxes doesn't multiply into 2N connections to one
  origin — politeness violations scale with fleet size unless the budget is a fleet-wide
  number). This is where the politeness state has to live: **in the frontier/scheduler,
  not in the fetchers.** Fetchers that self-throttle without knowing about the other
  fetchers talking to the same host are the classic "we all thought we were being polite"
  bug. The frontier assigns work units per host so at most one fetcher is talking to a
  given host at a time (or, for big hosts, an explicit fleet-wide concurrency token).

**Timeouts and hostile-page defense (NFR5).**

- Every fetch has: connect timeout, headers timeout, full-body timeout in addition to a
  max-size cap and a max-redirect-hop cap. This is where a misbehaving host loses the
  ability to eat the budget: a host that sends 1 B/s forever is capped by full-body
  timeout and max size, not by politeness.
- Redirect chains get a hop limit and a loop-detection set (same URL twice in a chain =
  stop). Redirect loops are the most common trap pattern that a crawler meets by accident
  without any adversary: a misconfigured CMS redirecting `/a` to `/a/` to `/a?x=1` etc.
- **Option A — treat misbehavior by per-request limits only**: wins simplicity; loses
  because per-request limits don't bound a *host* (a host serving 10,000 slow-but-valid
  responses at crawl-delay can still drain the budget).
- **Option B — per-request limits + a host health score (chosen)** (error rate, average
  latency, response-size anomaly) → host moved to "degraded" tier with a much lower
  budget or paused; wins budget protection; loses because a host health score can misfire
  (a host that is just genuinely slow gets throttled, even if its owner would have paid
  the cost). Mitigation: host health uses long windows (tens of minutes to hours), not
  single-request signals.
- **Chosen: B**, because a single page that wedges a fetcher slot for 60 s is bounded by
  the timeout, but a host that wedge-cycles repeatedly is a fleet-level threat.

**Failure modes**:
- A fetcher box loses DNS (upstream resolver unreachable): the box stalls, and unless it
  fails fast, work units pile up. Mitigation: fetchers fail work units fast on DNS
  unavailability (retry via frontier) rather than hanging.
- A host's TLS certificate is invalid/rotated badly: this is a *host* signal, not a
  request signal, so a cert-hostile host gets the same degradation path, and a cert pinning
  issue is not turned into a global TLS configuration rollback.
- A fetcher "succeeds" against HTTP 200 but returns a soft-404 (e.g. a login page for
  everything). Detected downstream by extraction and the freshness scheduler (deep dive 6),
  not by the fetcher.

## 5. Extraction and the downstream indexing handoff

Extraction turns a fetched HTML blob into records the index can consume. It is CPU-heavy
relative to the fetcher (which is I/O-heavy) and it's the place where we choose what to
*keep*, which is a cost lever (NFR4) as much as a correctness lever.

The extracted record, per page: (a) main content text after boilerplate removal,
(b) outgoing links with anchor text, (c) metadata (title, canonical tag, published and
last-modified dates, language, content type), (d) dedup signatures (deep dive 2),
(e) the blob reference (pointer to the stored page).

**Boilerplate removal** (nav, ad slots, cookie banners, repeated template chrome):

- **Option A — no boilerplate strip; index what came back**: wins extraction simplicity;
  loses because the indexer's per-page signal gets diluted — a nav-heavy site's pages all
  score for the same site-wide terms, which poisons both ranking and the SimHash
  fingerprint that deep dive 2 relies on.
- **Option B — template-detection approach**: across the pages of one host, find
  block-level elements that repeat with near-identical text across pages (nav, footer);
  strip them. Wins a clean text signal and, as a side effect, makes the dedup signature
  meaningful. Loses because it needs cross-page context at extraction time: a page must be
  compared against siblings of its host, which means either (i) holding site-level caches
  of block hashes at the extraction level, or (ii) a second pass over multiple pages of a
  host. We pick B with approach (i) — a small, LRU-capped per-host block-hash cache in
  extraction. The trade (approximation, imperfect strip on small sites) is worth it
  because B is what makes deep dive 2's SimHash usable.
- **Failure modes of B**: a small site with 3 pages looks 100% template at the block
  level (all blocks repeat across pages), and a strip could remove everything. Guard: the
  boilerplate decision requires a minimum number of observed pages per host and a minimum
  distinct-text ratio before stripping is active.

**Handoff.** Content records go onto a partitioned, at-least-once stream (the reasoning
about that choice is [13.kafka](../13.kafka/04-deep-dives.md) §1 — an append-only log that
consumers own the cursor on). Partitioning is by canonical-URL hash, which gives these
properties: (i) all events for one URL (fetch status, extraction, recrawl events) land in
order on one partition; (ii) indexer and link-graph builder both consume the same stream
without a second fetch; (iii) replay from the log is the recovery path when the indexer
fails (deep dive 6's scheduler also consumes from here).

- **Option A — synchronous RPC from extraction to indexer**: wins no intermediate log and
  no consumer lag; loses on coupling — an indexer outage stalls the crawler or forces
  buffer-and-push, and replay/recovery has no clean source.
- **Option B — stream (chosen)**: wins decoupling, replay, multi-consumer, and the ability
  to add the freshness scheduler and link-graph builder without touching the crawler.
  Loses at-least-once overhead (duplicate records must be tolerated by consumers, which
  dedup by URL + content hash) and an added moving part (the broker cluster) with its own
  operational story.
- **Chosen: B**, because the crawler is upstream of many consumers and NFR6's "lost fetch
  results cost a re-fetch" requirement is much easier to meet with a durable record stream
  than with RPC retries.

**Failure modes**:
- Extraction has a bug (e.g. new HTML pattern breaks the parser): it produces garbage for
  some traffic slice. Because the blob is retained (pointer, not parsed output), the
  recovery is a re-extraction pass, not a re-fetch — the blob is the recovery point. This
  is why the blob store retains raw HTML for the corpus TTL rather than just parsed
  output.
- Indexer consumer falls behind: stream backpressure, fetchers keep going, extraction
  buffers, and the whole system degrades gracefully because each stage is decoupled by a
  buffered queue (deep dive 1's back queues and this buffer are the same pattern).

## 6. Recrawl scheduling: freshness vs. cost (delta vs. full)

Once the corpus exists, most fetches are re-fetches. The scheduler's job: given a fixed
budget (~4 B fetches/day, of which ~0.7 B are delta re-checks on the high-churn tier per
02-scale-estimates §1), decide *which* URLs to re-fetch and *when*. This is the
Cho–Garcia-Molina problem: pages change at different rates, so the optimal recrawl
interval for a page is a function of its churn rate; and importance matters too, so the
interval trades off "how important is it to see the new value quickly" against "how often
does the value actually change".

**Option A — full recrawl** (every URL re-fetched once per cycle):

- wins: uniform freshness guarantees, trivially understandable scheduling, no per-page
  state beyond "next attempt at t".
- loses: huge waste on the long tail — a page that hasn't changed in 5 years is re-fetched
  every 30 days; ~most of the corpus is like that. Also loses on the high-value pages:
  they get the *same* 30-day staleness as everything else, which is far worse than the
  product promises (NFR3).

**Option B — pure churn-adaptive** (estimate each page's rate of change, re-fetch on a
schedule proportional to the estimated change rate):

- wins: cost goes where the changes are; long-tail near-static pages recrawl rarely;
  freshness on the hot set becomes product-plausible.
- loses: estimating churn per page is a noisy statistical problem (a page that changed
  twice in 6 months of samples is not "changed every 3 months"); the tail drifts stale and
  no one notices because the *system* has no per-page freshness SLA to violate, only tier
  aggregates; and pure adaptivity ignores importance — a churny but unimportant page
  recrawls more than a low-churn page a lot of users need.

**Option C — importance × churn, tiered** (chosen):

- every URL gets a tier from (importance, churn-estimate); each tier has a recrawl
  interval target (minutes for a handful of front pages, hours for a top slice, days for
  a mid tier, ~30 days for the long tail).
- recrawl is *not* proportional to churn alone; it's proportional to churn *and* bounded
  by importance-weighted caps, so the scheduler can be tuned per tier without touching
  per-page logic.
- delta re-crawl budget (0.7 B/day, §1 of 02-scale-estimates) is spent on the top tiers;
  the full-sweep 3.3 B/day covers the tail.
- **Chosen: C.** A wins on simplicity; B wins on adaptivity but loses on product
  relevance and estimation noise; C gives up some statistical optimality in exchange for
  tiers that the product actually promised (NFR3) and that operators can reason about
  (NFR7). A "we promise freshness to 30 days for the tail and minutes-to-days for top"
  requirement is inherently tiered, so the scheduler should be too.

**Signals that feed tier assignment** (also in 03-architecture §3.6): observed content
change (SimHash delta between recrawls), HTTP Last-Modified and ETag behavior, sitemap
lastmod, 404/rate responses, and importance (backlink-derived, from the link-graph
consumer on the same stream).

**Conditional requests (If-Modified-Since / ETag) are the cost lever that makes any of
this work**: a re-fetch that returns 304 costs a few bytes instead of 150 KB. Option A
(no conditional requests) wins bandwidth-simplicity; Option B (always send validators)
wins a ~10–100× bandwidth cut on unchanged re-crawls, and is the only way ~4 B fetches/day
at ~150 KB average survives its own arithmetic — the 600 TB/day raw volume in
02-scale-estimates §3 assumes every fetch returns a full 150 KB body; a high 304 rate on
the re-crawl share is what keeps actual transfer volume well below that bound.
**Chosen: B.**

**Failure modes**:
- Churn estimate goes wrong (a page changed once and never again is scheduled at a
  minutes-level interval forever). Mitigation: an exponential moving average with a floor
  and ceiling, plus a periodic re-estimation pass over every tier from actual observed
  deltas (from the stream), not from the schedule itself.
- The "dead page" problem: 404s and soft-404s (misconfigured hosts returning 200 for
  everything) accumulate forever in a corpus unless there's a pruning policy. Mitigation:
  repeated 404/410 over multiple full cycles → drop from scheduling; soft-404 detection
  at extraction (page content matches a known error template) → treat as 404.
- Hot-set starvation: a delta tier's pages can crowd out the full-sweep budget. Mitigation:
  the two budgets are separate line items at the scheduler level (0.7 B/day delta and
  3.3 B/day sweep); the sum, not either alone, is the 4 B/day target.
- Recrawl storms after a global change (a new extraction bug fixed and re-processing the
  whole corpus): the URL table's "next_attempt" is a mass-write that the frontier has to
  absorb without hammering every host at once — the same per-host politeness heap in deep
  dive 1 serves re-fetches, so politeness holds during storms too.

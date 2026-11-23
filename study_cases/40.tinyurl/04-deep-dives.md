# 04 — Deep dives

## 1. Short-code generation: counter vs hash vs pre-allocated ranges

**Option A — global sequential counter, base62-encoded** (`0,1,2…` → `aB3xK9`):
- wins: uses 100% of the key space (see 02-scale-estimates arithmetic); no collisions by
  construction; no uniqueness check needed on write; codes are short and stable; same
  allocation philosophy as a Snowflake ID (../02.twitter/04-deep-dives.md).
- loses: codes are *enumerable* — `aB3xK8` sits next to `aB3xK9`, so unlisted/private links
  are guessable by neighbor-probing; a single hot counter (or a coordinating service) is a
  write-path SPOF; monotonicity leaks volume (a competitor can graph our daily link count
  from code gaps) — acceptable, but worth knowing.

**Option B — hash the long URL (or random bits), check for collision**:
- wins: no coordination at all; codes look unguessable; deterministic hashing of the URL
  would even give free dedup (same URL → same code).
- loses: collision check adds a read before every write and a retry loop on collision —
  cheap now (~600 QPS) but it is the only part of the create path that can get weird; worse,
  as the space fills it degrades (02-scale-estimates: at 36.5B codes in a 6-char space the
  birthday bound makes collisions routine); deterministic URL hashing is actually a bug for
  a public shortener — it lets anyone test whether a URL was ever shortened (a leakage
  oracle), which is why "free dedup" is more of a liability than a feature.

**Option C — pre-allocated ranges** (each allocator node takes a block of, say, 1M integers
from an assigned coordinator — ZooKeeper-style coordination, see
../11.zookeeper/04-deep-dives.md on the sequencer/coordinator recipes — and
encodes them base62 with a per-node prefix mixed in):
- wins: no write-time coordination (node works from its in-memory block); counter
  monotonicity is broken up so codes are less obviously adjacent; survives coordinator
  outages for a long window (blocks are big); simplest scaling story.
- loses: wasted codes when nodes die holding unused ranges (~nothing at these rates);
  still partially enumerable within a node's block — the guessability objection persists;
  one more moving part (the coordinator).

**Chosen: C for auto codes (pre-allocated ranges + base62), with a per-code random suffix
scramble inside the range to defeat neighbor-probing.** Rationale: the write path is so
lightly loaded that any option *works*; the deciding factors are (1) no write-time collision
loop and no SPOF on shorten, which kills pure-A, and (2) unguessable private links, which
kills A and partially C unless we scramble. Scrambling inside pre-allocated ranges (a node
maps its range through a fixed random permutation per epoch — bijective, so no collisions,
no coordination) keeps both: full space use, no collisions, non-enumerable.

- Trade-off made explicit: we accept a permutation component in the generator (a non-trivial
  correctness detail: it must be bijective per range) in exchange for collision-free,
  coordinator-free allocation with unguessable codes. Pure hashing (B) is the fallback if we
  ever need determinism (e.g., dedup on shorten), accepting the leakage oracle and the
  collision check as its price.

**Failure modes**: coordinator down → nodes keep serving from in-memory ranges, new nodes
idle until it returns (degrade, don't block); node crashes with an unused range → thousands
of codes silently skipped, irrelevant at 6–7 chars; permutation bug → duplicate codes → the
KV write must still enforce uniqueness (insert-if-absent) as the last line of defense.

## 2. The redirect hot path: cache layers, negative lookups, 301 vs 302

Reads outnumber writes ~100:1 and the peak target is ~60K QPS. Every layer must assume the
one above it missed.

**Cache layers (in probe order)**:
1. **In-process LRU** (per redirect node): no network hop at all. Hot links (viral,
   campaign) will dominate; this absorbs most traffic.
2. **Shared cache** (Redis-class, consistent-hash sharded by code — see
   ../10.distributed-cache/README.md): covers the working set across nodes; also the place
   where `status` (blocked/expired) lives so abuse takedowns propagate quickly.
3. **KV store**: cold links. Fine — a miss is one ~few-ms lookup, and misses after layer 2
   are a small fraction if the cache is sized to the heavy-tail working set
   (02-scale-estimates: a few TB of RAM covers "clicked recently").

**Unknown codes are their own problem.** Scrape/guess traffic hammers codes that don't
exist; if every miss goes LRU → cache → KV, the KV store is paying for noise, and negative
results don't cache naturally. Fix: a **Bloom filter of all known codes** in every redirect
node's memory. "Definitely not there" → 404 immediately, no cache hop, and we negatively
cache 404s briefly at the edge too. False positives just fall through to the normal lookup
and return 404 — correct outcome, wasted only when they happen.

- Trade-off made explicit: the Bloom filter wins K memory lookups at a cost of ~10
  bits/code (~1% false-positive rate for an optimally sized filter; a few bits more or
  less scales accuracy linearly) and a rebuild/invalidation path (creations must insert, and a filter is
  approximate — it never needs deletions if links never truly expire, which our default
  policy guarantees). Loses: one more thing to keep consistent on write. We pay it because
  unknown-code traffic is unbounded (adversarial), unlike missing-link traffic.

**301 vs 302 — the decision that shapes everything**:
- **Option A — 301 (permanent)**: wins: browsers cache it, subsequent clicks never hit us
  (huge offload, the CDN dream for free). loses: we lose click analytics on repeat clicks;
  a blocked/edited/expired link keeps redirecting from browser caches indefinitely — a
  security and product nightmare.
- **Option B — 302 (found)**: wins: every click comes home, so counting is accurate and
  takedowns/expiry are instantly effective. loses: every click hits the system — the entire
  load is ours forever, mitigated only by CDN caching of the 302 itself (allowed with
  proper Cache-Control).

**Chosen: 302 by default, with short-lived, explicit `Cache-Control` on the 302 so edge
pops can serve repeat clicks without a full round trip to origin.** We pick B because
correctness of takedown and analytics is a core requirement (deep dives 4 and 5) and we buy
back most of A's offload with edge caching rather than handing control to client caches.

**Failure modes**: cache shard down → probe skips to KV (capacity loss, not outage);
KV down → serve stale from layers 1–2 with a header noting degraded mode, and for
uncached codes return a visible error page immediately — 01's fail-open clause:
cached data or a visible page, never an infinite hang; edge outage →
origin absorbs full load briefly — origin must be sized for some overshoot (~2–3× of peak
(estimate), not 100×).

## 3. Custom aliases and the reserved-word policy

Users want `sho.rt/summer-sale`. Now the code space is no longer opaque — aliases are words,
brand names, profanity, phishing bait (`sho.rt/paypal-secure`), and even shorteners' own
paths (`sho.rt/api` collides with real routes).

**Option A — free-for-all**, first-come-first-served on any string:
- wins: zero product friction; no list to maintain.
- loses: route collisions (aliases shadow real endpoints like `/api`, `/help`); profanity
  and homoglyphs of brands on our domain; brand squatting, which legal/abuse teams hate;
  aliases leak intent (phishing loves plausible custom aliases).

**Option B — strict whitelist** (only `^[a-z0-9-]{6,40}$`-style, plus profanity/brand block
list):
- wins: kills the worst classes cheaply; still flexible.
- loses: never complete — profanity lists and brand lists are living datasets, and
  homoglyph/spoof variants (`paypa1`, `app1e`) need per-case judgment.

**Chosen: B, expressed as three ordered checks — (1) structural (charset/length, must not
match reserved route prefixes), (2) static block lists (reserved words, profanity, known
brand homoglyphs), (3) human review queue for anything that looks brand-adjacent or is
flagged.** Custom aliases are additionally always served with the 302 + status-check path
and never cached longer than a minute at the edge, so a mistakenly-approved alias can be
revoked quickly (this reuses the takedown machinery from deep dive 5 rather than inventing
new policy).

- Trade-off made explicit: we accept a moderation queue and occasional false blocks of
  legitimate aliases in exchange for not hosting brand-squatting and profanity on what is
  effectively a public namespace over which we are the publisher.

**Failure modes**: block-list service down → fail closed for *new* custom aliases only
(auto codes unaffected); alias race (two users claim the same alias) → the KV insert must
be conditional (insert-if-absent); winner takes it, loser gets an error — no lock needed at
~600 QPS.

## 4. Click analytics without slowing the redirect

The redirect is ~10 ms end-to-end from cache; a synchronous analytics write would add a
database round trip to 60K QPS. The requirement is "counts are eventually right", not
"counts are instantly durable" — perfect fit for async.

**Option A — synchronous counter update on redirect**:
- wins: strong consistency; no pipeline to operate.
- loses: adds a write to the hottest path; counter hot rows (viral link) become a write
  hotspot; analytics load couples to availability of the redirect (pipeline down = redirect
  down, the exact opposite of what we want).

**Option B — fire-and-forget events → log → aggregation**:
- wins: redirect cost is one non-blocking emit (often to a local buffer, batched, UDP-flavor
  or in-process ring → shipped); backpressure drops events rather than slowing redirects;
  aggregators can batch, dedupe, and write cold counters — see ../13.kafka/README.md for the
  log semantics (replay, ordering by key).
- loses: counts lag seconds-to-minutes; drops under overload are real (we lose clicks, not
  just delay them); two sources of truth (raw log vs rollup counters) that can disagree.

**Chosen: B.** The redirect wins any argument with analytics. Design details that matter:
(1) counters are maintained in the shared cache with periodic flush, because a viral link
getting 50K clicks/min cannot hit the KV 50K times/min for one row — batch/merge in cache,
flush periodically; (2) time-bucketed rollups (per link per hour/day) are precomputed by a
stream job; (3) "drop on backpressure" is a stated, accepted loss — we prefer a slightly
undercounted link over a slow redirect, and we mark degraded sampling in the pipeline rather
than pretending numbers are exact.

- Trade-off made explicit: we accept approximate, eventually-consistent counts (and
  occasional undercounting under extreme load) in exchange for keeping the redirect path
  free of every dependency analytics might have.

**Failure modes**: pipeline down → events buffer in-process briefly then drop (redirects
unaffected); viral-link counter hot row → cache-side merging must be per-node, flushed
in batch, never per-click; aggregation job lag → dashboards stale but consistent on catch-up.

## 5. Expiry and abuse: shorteners as a trust layer

A shortener performs a bait-and-switch-adjacent function on trust: the email said
`sho.rt/x`, the destination is a credential harvester. Publicly reported data (Google Safe
Browsing transparency reports, roughly) has shown millions of unsafe destinations flagged
at any given time, and shorteners are a known delivery vector — we should design as if a
measurable fraction of links are hostile.

**Takedowns (malware/phishing)**:
- **Option A — scan at creation, trust forever**: wins: zero redirect-path cost; simple.
  loses: a link whose destination was clean at creation can serve malware a week later
  (shorteners redirect, destinations rotate); attackers also probe at create-time with
  benign URLs and swap later.
- **Option B — scan at creation + periodically re-scan verdicts + status check on
  redirect**: wins: takedown applies to already-created links (essential — a flagged link
  might have a million clicks in front of it); redirects consult cached
  `code → status ∈ {active, blocked}` so enforcement is seconds-fast.
  loses: a status check on the hot path (cheap when cached, and shared-cache status is
  already the plan, see deep dive 2); verdict churn can cause brief flapping.

**Chosen: B**, implemented as: creation-time URL scan (blocked outright if known-bad),
asynchronous re-scan of *destinations* (a fraction per sweep, prioritized by click volume),
verdict → `status=blocked` in KV → cache invalidation → redirect serves an interstitial,
never the destination. The status field rides in every cached record, so takedown is a
single-field update, not a rewrite.

**Expiry**: links live forever by default (shorteners that expire links break the internet's
bookkeeping — citations, emails, invoices); expiry exists only when the creator asked for a
TTL. Expired codes resolve to an expiry page (not a bare 404) because the click still means
something to the user. Implementation piggybacks on `status ∈ {active, expired}` — same
mechanism as takedown, no separate path.

**Rate limits / abuse of the *creation* side**: anonymous creation is the free tier for
spammers. Limits are per-IP and per-account, tiered (anonymous: small, authenticated: more),
with a fail-closed cap on new custom aliases (deep dive 3) and normal auto-code creation
degrading gracefully (reject with 429, not hang). Rate limiting is at the API edge only —
the redirect path never rate-limits by account, because a redirect victim (the person who
clicked) is not the abuser.

- Trade-off made explicit: we accept redirect-latency cost of a cached status check and
  operational cost of a re-scan pipeline in exchange for the ability to kill a bad
  destination within seconds of a verdict, instead of trusting creations forever.

**Failure modes**: scanner down → takedowns stall (existing links keep serving; new link
creation fails closed for suspicious URLs only — don't let a broken scanner stop all
shortening); false-positive takedown → restore is one status flip (cheap, reversible —
this is why blocking is a `status` value and not a row delete); interstitial page cache
poisoning by attacker-controlled long URLs → long URLs are percent-encoded and the
interstitial never renders them as links/HTML.

# CDN across the case catalog

Where a CDN (or an edge-cache tier) appears in the 50 study cases, what role it plays, and how
deep each case goes. Companion to [CONCEPTS.md](CONCEPTS.md) ("CDN / edge delivery", 24/50 as a
committed union concept; also "Cache tiering (local, Redis, CDN)"). The raw substring counts
differ from the union by design: a plain `cdn` grep hits **37 / 50 cases**, but that includes
out-of-scope pointers, origin-side boundaries, and two explicit rejections that the union
correctly excludes. This file is the union's perimeter drawn precisely.

Full case: [09.cdn](09.cdn/README.md) — multi-customer CDN: anycast POPs, cache keys, tiering,
purges, hot objects, DDoS. This doc never re-derives it.

Headline: **in this catalog a CDN is not a component, it is a validity contract.** Every case
that succeeds with one does the same two things: *make the object immutable* and *make the
cache key carry the validity* — a content hash, a version, a release, an epoch. Once that holds,
the edge serves ~90–95%+ and "invalidation" is a key swap, not a purge. Every case that cannot
do that (personalized, per-request-computed, or latency-critical data) either rejects the CDN
explicitly or relegates it to a shell. The exceptions — Netflix, Spotify — are not violations;
they change only *who owns the same edge*.

- Cases mentioning a CDN (substring): **37 / 50**
- CDN/edge is a designed architectural component: **26** (1 full case + 25 peers)
- Brief mention, out-of-scope pointer, or origin-side boundary: **7** (07, 10, 15, 27, 28, 41, 43)
- Explicitly rejected with a stated reason: **3** (08, 38, 47)

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [09.cdn](09.cdn/README.md) | The case itself: multi-tenant CDN, 23M req/s, 100 PB/day | Full case |
| [01.youtube](01.youtube/README.md) | CDN as the byte plane + own edge→shield→origin hierarchy; ~1.4 EB/day | Deep |
| [31.netflix](31.netflix/README.md) | Own prepositioned CDN vs rented, chosen and argued | Deep |
| [32.spotify](32.spotify/README.md) | Multi-CDN vs operated POPs vs origin — mix chosen; signed-URL TTLs | Deep |
| [06.apple-app-store](06.apple-app-store/README.md) | CDN edges + signed URLs + pre-warm; ~75 PB/day, 85–90% byte hit | Deep |
| [26.google-maps](26.google-maps/README.md) | Tile CDN with epoch-in-key invalidation; ~22.5 PB/day at edge | Deep |
| [33.instagram](33.instagram/README.md) | Media CDN at 1.2M QPS; takedown invalidation; shields | Deep |
| [39.e-commerce](39.e-commerce/README.md) | CDN edge as the L0 armor: media + pre-warmed page shells | Moderate |
| [20.frontend-newsfeed](20.frontend-newsfeed/README.md) | Client SSR/CDN tier; variants vs cache-key collision; ~5 PB/day | Moderate |
| [37.ticketmaster](37.ticketmaster/README.md) | Edge as CDN + WAF + bot-filter front door | Moderate |
| [40.tinyurl](40.tinyurl/README.md) | Edge/CDN POP caches the redirect itself, negative-caches misses | Moderate |
| [45.typeahead](45.typeahead/README.md) | Edge serves static top-N snapshots for hot prefixes | Brief |
| [02.twitter](02.twitter/README.md) | Media CDN; bytes bypass app servers (~1.5 PB/day, ~95% hit) | Component |
| [03.whatsapp](03.whatsapp/README.md) | Content-hashed E2E-encrypted blobs via CDN | Component |
| [30.facebook-messenger](30.facebook-messenger/README.md) | blob store → CDN → receivers; same contract as 03 | Component |
| [05.discord](05.discord/README.md) | Object storage + CDN for attachments/avatars | Brief–moderate |
| [34.reddit](34.reddit/README.md) | CDN for media & static; posts store URLs only | Brief–moderate |
| [36.airbnb](36.airbnb/README.md) | Photos: image URLs immutable, 90%+ never hit origin | Moderate |
| [29.uber-eats](29.uber-eats/README.md) | Version-keyed catalog + CDN; publish storm | Moderate |
| [16.dalle](16.dalle/README.md) | Result blob store + CDN, signed URLs, push derived sizes | Moderate |
| [17.elevenlabs](17.elevenlabs/README.md) | Object store + CDN for audio deliverables | Brief–moderate |
| [18.sora](18.sora/README.md) | CDN for renditions/watermarked artifacts; "exactly YouTube's stack" | Moderate |
| [35.tiktok](35.tiktok/README.md) | blob + CDN as given substrate; CDN-pin first segments | Component (given) |
| [25.mobile-newsfeed](25.mobile-newsfeed/README.md) | Variant CDN media; ≥90% edge hits absorb ~230K peak QPS | Moderate |
| [19.frontend-chat-app](19.frontend-chat-app/README.md) | Client fetches media by content hash from CDN | Brief–moderate |
| [22.frontend-uber](22.frontend-uber/README.md) | Tiles CDN-bound; client HTTP cache defers to CDN cache-control | Brief–moderate |
| [23.frontend-video-streaming](23.frontend-video-streaming/README.md) | Consumes the 09 edge as given; designs the player side | Brief (by-reference) |
| [43.zoom](43.zoom/README.md) | Broadcast boundary: >5–10K viewers → prepositioned CDN | Boundary |
| [27.ev-charging-network](27.ev-charging-network/README.md) | Firmware artifacts via object storage/CDN, range/resume | Brief |
| [41.web-crawler](41.web-crawler/README.md) | "CDN-served resources" = crawling-world sizing fact only | Brief (non-component) |
| [15.ml-ranking-system](15.ml-ranking-system/README.md) | Out-of-scope pointer only | Brief (out of scope) |
| [28.uber](28.uber/README.md) | Out-of-scope pointer only | Brief (out of scope) |
| [07.blob-store](07.blob-store/README.md) | Origin-side boundary: "the CDN is someone else's layer" | Boundary |
| [10.distributed-cache](10.distributed-cache/README.md) | Adjacent layer: edge caching is explicitly out of scope | Boundary |
| [08.chatgpt](08.chatgpt/README.md) | Responses uncacheable by design; CDN for static UI assets only | Explicitly rejected |
| [38.leetcode](38.leetcode/README.md) | "No CDN/content plane" — nothing is worth caching | Explicitly rejected |
| [47.rate-limiter](47.rate-limiter/README.md) | Config is "CDN-able"; enforcement must never be | Explicitly rejected |

## Per-case notes

### The full case

**[09.cdn](09.cdn/README.md)** — Scaled to the shape of CloudFront/Cloudflare: ~300 POPs,
~23M req/s average, ≥95% request / ≥85% byte hit ratio, ~100 PB/day egress, 2×10^12 requests/day
of purge blast radius. The five dives are the general tools every other case borrows:
[04-deep-dives.md](09.cdn/04-deep-dives.md) — (1) anycast base + DNS steering for the tail;
(2) cache-key economics, "the CDN's P&L is hit rate": moving byte hit 85%→92.5% halves origin
bytes (15/7.5 = 2×); (3) edge → regional shield → origin tiering, one hop bought to collapse
uncorrelated per-POP miss singletons; (4) invalidation as a per-key log propagated in seconds —
watermark hard purge vs soft purge (stale-while-revalidate as a purge semantic) vs surrogate-key
/tag purge backed by an async log-built index; and the stated boundary that *in-flight* requests
may still see old bytes; (5) hot-object handling — per-key coalescing, TTL jitter (+0–10%),
replicate-to-all-servers serving split from ownership; (6) the same fleet as DDoS absorbent
~70 Tbps-class floods because anycast *is* the weapon. All CDN-as-system questions live here.

### Video platforms: who owns the edge

The owned-vs-rented split is decided by one variable: **how much of today's demand is
predictable**.

- **[01.youtube](01.youtube/README.md)** — the demand-driven shape: edge POPs pull on miss,
  pre-warm best-effort at publish, ~1.4 EB/day through the CDN (from
  [02-scale-estimates.md](01.youtube/02-scale-estimates.md): 15% of ~3.4 ZB/yr global traffic).
  Its own edge→shield→origin hierarchy ([04-deep-dives.md](01.youtube/04-deep-dives.md), dive 3)
  is what [09.cdn](09.cdn/README.md) generalizes; the tier nearest users deliberately stores the
  *least* (metro real estate is the constraint), and CDN cache-fill signals feed the cold-start
  pre-warm decision for videos published hours before they explode. YouTube *buys adaptivity*
  with a viral-tail catalog — the counterweight to Netflix's argument.
- **[31.netflix](31.netflix/README.md)** — the same question answered the other way
  ([04-deep-dives.md](31.netflix/04-deep-dives.md), dive 2): 50M concurrent streams, ~175 Tbps
  peak, delivered ~945 PB/day against ~12 TB ingest (~80,000:1) of a *forecastable* catalog.
  Option A ([09.cdn](09.cdn/README.md)'s shape) "wins adaptivity, loses guarantees — a reactive
  cache can only promise what it happens to hold at 9 PM". Chosen: Open Connect, appliances
  inside ISPs, nightly fill of forecast-popular titles, steering by continuous probes not DNS;
  the cost is that inventory forecasting becomes the core problem. Netflix historically ran
  multi-CDN before consolidating — per-session quality-based steering survives as the
  selection mechanism.
- **[32.spotify](32.spotify/README.md)** — the middle case, where the mix is the answer
  ([04-deep-dives.md](32.spotify/04-deep-dives.md), dive 3): ~59K/s track-start requests but
  modest bytes (~40 KB first-chunk working sets) — "request volume is enormous, byte volume
  modest", the regime where owned edge caching beats renting CDN bytes because per-request
  pricing dominates. Option C chosen: operated POPs first-hop, third-party multi-vendor CDNs
  as overflow/fallback (a vendor outage becomes a cache-miss retry, not a player error).
  Multi-vendor here is explicitly *a retry play, not a cost play*.
- **[18.sora](18.sora/README.md)** and **[17.elevenlabs](17.elevenlabs/README.md)** and
  **[16.dalle](16.dalle/README.md)** — generation products whose serving plane is ordinary CDN:
  18 states "CDN design is exactly YouTube's stack (edge → shield → origin, negative caching)"
  and severs generation from serving plane entirely; 16 pushes derived sizes to the CDN
  proactively on job completion with signed URLs (95%+ hit on creation-hot content, origin
  ~2 TB/day); 17 calls ~6 TB/day of audio egress "CDN-class" and lets bytes bypass the synthesis
  path. None of them designs the edge; all of them depend on the 09 contract.

### Media-byte planes: immutable bytes bypass the app

The dominant pattern in the catalog. A content-addressed or version-keyed blob store is the
origin, the CDN is the front, and app servers never touch 90%+ of bytes.

- **[03.whatsapp](03.whatsapp/README.md)** — E2E-encrypted blobs keyed by content hash; upload
  goes to edge POPs, download rides the CDN; the store holds only dedup + reference.
- **[30.facebook-messenger](30.facebook-messenger/README.md)** — identical contract: sender
  sends an ID, the CDN serves the bytes, "bytes never in chat path".
- **[02.twitter](02.twitter/README.md)** — ~1.5 PB/day of media bypassing app servers
  (~17 GB/s, peak ~700 Gbps), [02-scale-estimates.md](02.twitter/02-scale-estimates.md)
  prices origin load at ~5% of QPS behind a ~95%+ CDN hit ratio.
- **[34.reddit](34.reddit/README.md)**, **[05.discord](05.discord/README.md)** — media CDN as
  an architecture line: object store + CDN, posts/attachments store only URLs.
- **[36.airbnb](36.airbnb/README.md)** — photos own the bandwidth budget (~5.5K QPS vs ~2K peak
  search; ~2–6 Gbps), "the CDN is part of the product", 90%+ of image requests never arrive;
  [04-deep-dives.md](36.airbnb/04-deep-dives.md) applies 09's cache-key economics to
  resize/filter/jittered viewports and reuses its invalidation trade-off on listing edits.
- **[25.mobile-newsfeed](25.mobile-newsfeed/README.md)** — per-variant CDN media keyed by the
  item's variant list; ~230K peak QPS absorbed by edges (public hit ratios ≥ 90%), origin sees
  ~6K QPS ([02-scale-estimates.md](25.mobile-newsfeed/02-scale-estimates.md)).

### Frontend cases: designing *against* a given edge

These own the client/SSR side and treat the edge as a given substrate (23 even names it):
"CDN edge (given; Case 09)".

- **[23.frontend-video-streaming](23.frontend-video-streaming/README.md)** — the delivery plane
  is delegated wholesale; the case designs ABR, retry-abort ordering, and cache-friendly URL
  forms, and points at [09.cdn/04-deep-dives.md](09.cdn/04-deep-dives.md) §5 for the hot-key
  fix on segment 1. (See [REALTIME-TRANSPORT.md](REALTIME-TRANSPORT.md) for the shared framing.)
- **[22.frontend-uber](22.frontend-uber/README.md)** — ~1.5–3 MB tile bytes per session from a
  CDN; the client deliberately routes through the platform HTTP cache so CDN cache-control does
  the eviction honesty for free — client-side design that lets the edge's TTL semantics own the
  trade-off.
- **[19.frontend-chat-app](19.frontend-chat-app/README.md)** — render skeleton first, fetch the
  full asset by content hash from the CDN on viewport entry, cancel on scroll-away.
- **[20.frontend-newsfeed](20.frontend-newsfeed/README.md)** — the one with a real CDN
  collision: SSR outputs are experiment-variant-stamped, threatening CDN cache keys;
  [04-deep-dives.md](20.frontend-newsfeed/04-deep-dives.md) resolves the experiments-vs-hit-rate
  fight in favor of cache-efficiency, exactly 09's "narrow by default, widen for correctness"
  rule mirrored to the requester side. ~5 PB/day CDN egress.

### Cache keys and invalidation as the design discipline

Cases where the CDN contract *drives* the data model:

- **[26.google-maps](26.google-maps/README.md)** — tiles immutable, so invalidation is an
  *epoch bump in the affected region*: "CDN sees new keys", no purge call ever fires at scale.
  ~22.5 PB/day edge egress at ~95% hit; [04-deep-dives.md](26.google-maps/04-deep-dives.md) —
  "what is a tile" *is* a CDN decision: immovable keys with bounded cardinality is why the hit
  ratio is the product.
- **[29.uber-eats](29.uber-eats/README.md)** — menu pages CDN-cached, but content is
  *version-keyed* so merchant edits can't corrupt live pages
  ([04-deep-dives.md](29.uber-eats/04-deep-dives.md)); the residual purge is bounded by the
  immovable-key win from 09, and the known failure is a publish storm (all prices edited at
  17:00) stampeding CDN keys.
- **[33.instagram](33.instagram/README.md)** — 1.2M QPS of media GETs must be served by CDN
  edge caches, or even 5% origin residual is ~60K QPS against blob storage
  ([02-scale-estimates.md](33.instagram/02-scale-estimates.md)); URLs immutable per rendition;
  the hard part is the exception: a takedown must invalidate a CDN that has no natural reason
  to expire — plus shields + single-flight for spike fan-out.
- **[39.e-commerce](39.e-commerce/README.md)** — the CDN is armor *layer* L0 in the flash-sale
  ladder: media + page shells pre-warmed before start time, "losers cost the edge bytes, not
  origin QPS (09.cdn pattern)"; ~90%+ media cache hit at peak
  ([02-scale-estimates.md](39.e-commerce/02-scale-estimates.md), ~600 TB corpus, CDN-served).

### Edge as front door: WAF, bots, and absorption

- **[37.ticketmaster](37.ticketmaster/README.md)** — the edge tier is "CDN + WAF + bot
  filtering": waiting-room assets served at the edge, per-IP rate limits declared nearly useless
  alone, fingerprinting + JS/proof-of-work challenges tagging risk downstream — 09's deep dive
  6 (per-session clearance, not per-request checks) reused as a component. The same logic is why
  e-commerce and app-store traffic patterns assume the edge absorbs the hostile head.

### Edge caching non-media content: redirects, suggestions, fragments

- **[40.tinyurl](40.tinyurl/README.md)** — the edge/CDN POP terminates the hottest fraction of
  redirects entirely and negative-caches misses; the 302 itself is browser/CDN-cacheable, and
  [04-deep-dives.md](40.tinyurl/04-deep-dives.md) weighs long-lived client caching as "the CDN
  dream for free" against losing click analytics on repeat clicks.
- **[45.typeahead](45.typeahead/README.md)** — edge serves static top-N snapshots for Zipfian
  hottest prefixes, protecting the service tier from the head of the distribution.

### Boundaries: the origin side and the adjacent layers

- **[07.blob-store](07.blob-store/README.md)** — explicitly *not* its layer: "we are the origin;
  see ../01.youtube for how an origin behind a CDN gets shielded". Its contribution to the CDN
  story is the other half of the contract — immutable content-addressed blobs are precisely
  what makes edge caching safe everywhere above. (See [OBJECT-STORAGE.md](OBJECT-STORAGE.md),
  which owns the storage side of this handoff.)
- **[10.distributed-cache](10.distributed-cache/README.md)** — the interior counterpart: its
  dives name CDN caching as a separate layer out of scope, and cite YouTube's CDN hierarchy as
  the thundering-herd precedent for request coalescing.
- **[43.zoom](43.zoom/README.md)** — a *mode boundary*, not a component: interactive meetings
  ride SFU relays; beyond ~5–10K viewers "stop pretending it's a meeting — prepositioned CDN
  delivery does the rest" ([04-deep-dives.md](43.zoom/04-deep-dives.md)), pointing at 01/31.
- **[27.ev-charging-network](27.ev-charging-network/README.md)** — firmware artifacts (~100 GB
  total, tiny next to media) served from object storage/CDN with HTTP range/resume; plus the
  security edge: an unsigned or mis-routed CDN artifact is a fleet-wide bricking vector, so
  signature verification happens client-side regardless of where bytes came from.
- **[41.web-crawler](41.web-crawler/README.md)** — "CDN-served resources" is a sizing fact
  about the crawled web (mean 150 KB HTML), not a component in the design.
- **[15.ml-ranking-system](15.ml-ranking-system/README.md)** and
  **[28.uber](28.uber/README.md)** — one-line out-of-scope pointers to the serving/delivery
  cases; no CDN in the architecture.

### Explicitly rejected

- **[08.chatgpt](08.chatgpt/README.md)** — CDN/edge caching of *responses* is rejected by
  design: "there is no re-serving cached answers"; each answer is generated. Only static UI
  assets ride a CDN. The cleanest statement of the dynamic-data boundary in the catalog.
- **[38.leetcode](38.leetcode/README.md)** — "No CDN/content plane — problem statements and
  static assets are small and cached"; at its scale the CDN would carry nothing worth its
  invalidation surface.
- **[47.rate-limiter](47.rate-limiter/README.md)** — split decision: the *config* is
  "CDN-able" (static rule snapshots pushed to every node, last-known-good pinned), but the
  *enforcement* must never ride an edge hop: an edge PoP RTT of 10–100 ms fails the ≤1 ms
  budget by 10–100× ([01-requirements.md](47.rate-limiter/01-requirements.md),
  [04-deep-dives.md](47.rate-limiter/04-deep-dives.md)). This is the boundary against reading
  09's L7 rate limiting too generously — the CDN can shield a *system*, it cannot *be* the
  decision point for a per-request admission check.

## The portable pattern

Every substantive use reduces to one contract, with [09.cdn](09.cdn/README.md) as the
reference implementation:

1. **Immutable bytes move to the edge.** The precondition for CDN serving is that the object
   never mutates: content-hash keys (03, 30, 19), release/version keys (06, 29), rendition
   URLs fixed forever (33, 36). Mutability is moved into the key, not into a callback.
2. **The key carries validity; invalidation is a key swap, not a purge.** Epoch bumps (26),
   version-keyed catalog pages (29), new content hash (06, 16). Purges are for exceptions only:
   a kill switch (39's sale-page kill, 06's takedown path, 33's takedown) — and then 09's
   soft-purge/stale-while-revalidate semantics keep the purge from being its own thundering herd.
3. **Hit ratio is the P&L.** Targets are consistent across cases: ≥95%/85% of requests/bytes
   (09), 85–90% byte hit for app delivery (06: 75 PB/day → ~9 PB/day origin), 90%+ images
   (36, 39), ≥90% feed media (25); each derives simply — origin load = (1 − hit) × demand.
4. **Tiering collapses miss fan-out**: edge → regional shield → origin, with coalescing and
   collapse-forwarding (09, mirrored inside 01's hierarchy, 33's shields + single-flight, and
   10's thundering-herd dive).
5. **Authorization is decoupled from the key**: signed URLs authenticate then key *without* the
   token (09, 06, 16, 32) — which makes the signed-URL TTL the entire revocation SLA for
   already-issued URLs (32's takedown problem), and verification-before-key-formation a
   hard requirement because a bug is a cache-poisoning vector (09 §2).
6. **Ownership of the edge is an economics decision, not an architecture one**: rent (09's
   customers), own (31 — *predictable* demand at 80,000:1), or mix (32 — request-heavy/
   byte-light; 01 — buys adaptivity). The catalog's rule of thumb: the more of today's demand
   you can name tonight, the more of the edge you can own.

**Boundaries — where a CDN is the wrong tool:**

- **Per-request-personalized or generated data** (08): there is nothing to cache; the edge can
  carry static UI assets only. Same logic in 33's feed responses vs feed media — only bytes go
  to the edge, decisions do not.
- **Latency-critical in-line decisions** (47): an edge hop costs 10–100 ms; a ≤1 ms budget
  cannot ride it, however good the edge's rate-limit product looks.
- **Interactive real-time media** (43, and the transport framing in
  [REALTIME-TRANSPORT.md](REALTIME-TRANSPORT.md)): SFU relays carry interactive audio/video;
  the CDN enters only when the product degrades into broadcasting (one-way segmented delivery).
- **Write-heavy, small, or tail-dominant object sets**: tinyurl's redirect traffic is cacheable
  but its writes are not; typeahead must pre-materialize snapshots first; 38's content is too
  small to be worth a second failure domain. The CDN pays for itself only above the
  hit-ratio math, and every case above shows its arithmetic.
- **Revocation you cannot tolerate being TTL-bound** (32's stale takedown'd track, 33's
  takedown): if a previously-issued URL must die *now*, the CDN contract guarantees propagation
  in seconds, not atomicity — in-flight fetches see old bytes (09 §4). Anything stricter needs
  the data model (version keys) or the kill path to change, not the CDN.

## Boundary and false-positive notes

Method note, mirroring [REALTIME-TRANSPORT.md](REALTIME-TRANSPORT.md) §filtering. Two greps ran
over `*/[0-9]*.md`: substring `cdn` and the regex `edge cache|edge PoP|cache hit|signed
URL|origin server|purge|invalidat`.

- **Substring `cdn` is clean**: all 37 dirs that match mention content delivery or point at
  [09.cdn](09.cdn/README.md). No accidental matches of the three letters (unlike the
  `"DASH"` → "dashboard" match REALTIME-TRANSPORT.md had to filter).
- **The secondary regex hits 13 additional dirs that are NOT about CDNs — all filtered**:
  [04.deployment-system](04.deployment-system/README.md) (build-cache hits/keys),
  [11.zookeeper](11.zookeeper/README.md) (watches as invalidation signals),
  [13.kafka](13.kafka/README.md) (page-cache hit rate),
  [21.frontend-calendar](21.frontend-calendar/README.md) (UI-side invalidation units),
  [42.linkedin](42.linkedin/README.md) (profile-cache invalidation, profile-viewer ACL),
  [46.google-docs](46.google-docs/README.md) (layout cache per changed region),
  [48.google-search](48.google-search/README.md) (results cache is server-side; no edge),
  [12.bigtable](12.bigtable/README.md), [14.ml-production-system](14.ml-production-system/README.md),
  [24.mobile-chat-app](24.mobile-chat-app/README.md), [44.payment-system](44.payment-system/README.md),
  [49.monitoring-metrics](49.monitoring-metrics/README.md),
  [50.stock-exchange](50.stock-exchange/README.md) (only "acknowledged"/"ledger"-adjacent
  words matched). These belong to [CONCEPTS.md](CONCEPTS.md)'s general caching union (row 1,
  36/50), not to the CDN row.
- **Counting**: 37/50 dirs mention a CDN. "Brief mention" includes two pure out-of-scope
  pointers (15, 28) and three non-component factual/boundary hits (41, 43's mode boundary,
  27's firmware path) — kept separate from the three explicit rejections (08, 38, 47), each
  of which states a reason. The CONCEPTS.md union row ("CDN / edge delivery", 24/50) is
  narrower than this file's raw mention count because it counts committed edge-delivery
  concepts, not pointers and boundaries; both numbers are correct under their definitions.
- **Overlap with companion docs (deliberate, link-don't-own)**: the media↔CDN handoff from the
  origin side belongs to [OBJECT-STORAGE.md](OBJECT-STORAGE.md) (03, 30, 16, 34, 39, 27 there;
  described here only as "the object store is the origin"). The segmented-delivery framing of
  23/31 belongs to [REALTIME-TRANSPORT.md](REALTIME-TRANSPORT.md). What this file owns that
  neither does: cache keys and TTLs, purge/invalidation propagation, shield/tiering, signed
  URLs, hit-ratio economics, DDoS/WAF role, and the owned-vs-rented decision.
- **Risk note**: 37.ticketmaster's edge detail was read from architecture only (no per-file
  scale table on edge egress); its classification as a designed component is unambiguous, but
  the per-case note above is depth-limited correspondingly. Depth labels for 05, 34, 19, 22
  ("Brief–moderate") are judgment calls on single-component mentions; no case was dropped
  after a partial read.

# 04 — Deep dives

## 1. Audio delivery: progressive download vs segmented streaming vs cache-first

Question: what protocol/shape delivers the first sample fastest? Three options:

- **Option A — monolithic progressive download over HTTP range requests.** One file per
  (track, rendition); the client opens a connection, requests the first N KB, starts
  decoding, keeps pulling in the background. *Wins*: simplest possible server shape; the file
  is a plain cacheable blob, so every byte is CDN-cache-friendly and offline-sync can reuse
  the exact same object; seek is a new range request. *Loses*: no mid-stream server-side
  adaptation (you must switch *files* to change quality — accept a 1–2 s re-buffer on tier
  change); range-heavy access patterns defeat some CDN cache configurations.
- **Option B — HLS/DASH-style segmented streaming** (2–4 s segments + manifest), like video.
  *Wins*: uniform infrastructure with video products; segment-granular CDN caching; natural
  quality switching mid-track. *Loses*: for audio it adds failure modes with no benefit —
  a manifest fetch (extra RTT) per track start directly attacks the time-to-first-sample
  budget, and "switching quality" of a 3-minute music track matters ~never mid-play. We would
  pay one round trip per start to buy a feature we don't need.
- **Option C — everything local first**: the player treats *its own disk cache* as the primary
  source, network only as backfill. *Wins*: removes the network from the common case
  entirely; gapless becomes a disk-read problem. *Loses*: cache identity, eviction and
  invalidation become correctness problems (wrong cached bytes = legally embarrassing
  playback of a takedown; see deep dive 6 and 2).

**We pick A + C as a hybrid, because B's costs (per-start RTT, manifest state) land exactly
on the budget metric we care most about, while its benefit (mid-stream ABR) is near-worthless
for 3-minute audio.** Concretely: per (track, tier) there is one range-addressable file;
the client streams the first ~1–2 s (≈20–40 KB at 160 kbps) then continues downloading into cache in the
background; a tier switch is a new file request at a silent boundary, or simply deferred to
the next track — nobody hears the difference *within* a track. This mirrors the conclusion
in ../01.youtube/04-deep-dives.md (segments exist to serve *seek + quality switch*) and lands
on the opposite choice, because for audio only seek survives as a requirement — and byte
ranges on a CBR audio file serve seek fine.

**Time-to-first-sample budget** (the whole deep dive in one line): cache hit → decode start in
tens of ms; cache miss → 1 RTT to edge + ~40 KB transfer at 160 kbps + decode ≈ 150–300 ms.
Every architectural decision elsewhere is downstream of refusing to add round trips here —
this is also why the entitlement/metadata/URL gateway call is *batched* (one call covers the
next several tracks) rather than per-track.

**Failure modes**: (a) first-chunk cache miss at an overloaded POP → client falls back to a
second CDN vendor for the retry — multi-vendor CDN isn't a cost play, it's a retry play
(../09.cdn/04-deep-dives.md); (b) stale cache holding a takedown'd track → signed-URL denial
only helps streaming downloads, so cached playback *without* a valid revalidation window must
be treated as expired (deep dive 2); (c) prefetch of a skipped playlist (users skip 20–30% of
prefetched tracks, roughly) → wasted egress; cap prefetch depth at ~2 tracks and prefer
prefetching metadata (cheap, never wasted) over audio (expensive, sometimes wasted).

## 2. Offline sync + DRM on downloaded content

Question: how do offline downloads survive airplane mode for weeks while the licensed catalog
changes underneath them?

**Design**: downloads are *encrypted at rest* with a device-bound key, wrapped per user; each
downloaded file set carries a **licence manifest** — (track, region, tier, offline-until
timestamp) — independently renewable. The client revalidates rights opportunistically
(connect → batch revalidate all manifests in one call) and *must* revalidate before the
offline window ends. Files are stored as the same range-addressable blobs fetched for
streaming (one download path, not two).

- **Trade-off — expiry length.** *Option A: long offline windows (weeks) win* user experience
  (a commuter's device just works) *and lose* licensing enforcement latency: a Thursday
  takedown is unplayable-on-streaming but still on disk through the window, which labels
  will not accept past some bound. *Option B: day-scale windows* win enforcement and lose
  the "offline forever" marketing story and burn revalidation traffic. **We pick weeks-scale
  expiry capped at the contractual ~30-day offline bound (01), auto-renewed on every
  connectivity window: while online, the renewal is invisible and enforcement latency is
  dominated by tombstone push (deep dive 6), and the outer bound — how long a device may stay
  fully offline before playback must stop — is a label negotiation outcome, not an
  engineering choice we can move.**
- **Trade-off — revalidation granularity.** Per-track manifests (fine-grained enforcement,
  many objects) vs per-library bundle (one manifest for all saves, cheap but coarse —
  takedown of one song pulls the whole bundle). **Bundle-with-tombstone-overlay wins in
  practice**: one cheap renewal call covers the whole library, and takedowns arrive as a
  small tombstone list the client subtracts locally.
- **The 10k-track cap** (publicly documented) is not just a limit to tell users about: it
  bounds worst-case per-device state (84 GB at 320 kbps — 02), which is what makes the
  offline store and sync scheduler *provably* budgetable rather than merely tuned.

**Failure modes**: (a) device offline longer than expiry → playback stops, client shows
"a few tracks need a refresh" — degrade per-track, never lock the whole offline library on one
failed call; (b) clock tampering to extend offline windows → treat device time as advisory,
encode expiry in the encrypted manifest and fail closed when the last *server-signed*
renewal is older than the window plus slack; (c) sync storm after a mass outage reconnects
millions of devices at once → revalidation is rate-shaped at the gateway with a *stale-manifest
grace period*, because failing closed during a reconnect storm is exactly the wrong moment
to be strict.

## 3. Catalog delivery: third-party CDN vs operated POPs vs origin

Question: where do the audio bytes physically come from?

- **Option A — pure third-party CDN** (multi-vendor). *Wins*: zero hardware to run, global
  reach for free, per-GB price is market-negotiated. *Loses*: you don't control the cache
  layer's eviction, your per-track-start request rate (59K/s, mostly *requests* for popular
  catalog) is expensive at edge pricing, and the long tail (the ~50%+ of catalog nobody
  played this week) is either no-hit-forever or evicted-on-arrival.
- **Option B — pure operated POPs** (own servers in colos + peering). *Wins*: full control of
  cache tiering (cheap disk for the long tail), predictable cost at scale, the operator
  relationship with the big ISPs peers *content* for free. *Loses*: you now run a global
  delivery fleet — capacity planning per region, flash-buyout failures your on-call owns.
- **Option C — mix**: operated POPs as first-hop where volume justifies them, third-party
  CDNs as overflow/fallback and for regions where peering is impractical; origin blob store
  (07.blob-store) sees only misses.

**We pick C**, with the reasoning made explicit: music's *request volume is enormous but its
byte volume modest* (deep dive 1's ~40 KB-first-chunk working sets), which is precisely
the regime where owned edge caching beats renting CDN bytes; the long tail still needs huge
cheap disk nobody wants to rent at CDN markup; and the fallback layer turns vendor POP outages
into cache-miss retries rather than player errors. Netflix's Open Connect reasoning
(../31.netflix/04-deep-dives.md) drives to owned-POPs-first for the same workload-shape
reason, at video scale; here the same logic holds but with lower stakes per POP.

**Cache-tiering inside a POP**: RAM/SSD for the hot head (a few tens of thousands of tracks
cover a huge share of plays — popularity follows a steep power law, same shape as video),
cheap disk for the tail, origin fetches for cold items — the tier logic is exactly
../10.distributed-cache/04-deep-dives.md applied to content bytes instead of query results.

**Failure modes**: (a) a new album drops at midnight globally → cold-cache thundering herd on
~10 tracks; mitigate with *pre-push* of known-release assets to POPs on schedule (the write
side knows release dates ahead — unlike video virality, releases are advertised); (b) POP
saturation during a regional event (a song goes viral) → overflow to CDN B is the *design*,
not an escalation; (c) origin fan-out from misses → origin blob store serves renditions, not
raw masters, and is itself sharded/replicated per region.

## 4. Playlists, sync and social

The hardest *correctness* problem in the system, hiding inside an apparently trivial feature.
A playlist is a mutable *ordered* list of ~10–200 items, edited from multiple devices, read at
playback-start (the hot path!), and displayed with derived fields (total duration, follower
count).

**Design**:
- **Server-authoritative identity, op-log-based sync.** Edits are recorded as operations
  (add at position, remove URI, reorder) against a server-ordered log per playlist, not
  snapshots. Devices keep a local copy in SQLite and apply ops; conflict resolution is
  *merge both sides* by op order (inserts reconcile, deletions win on identity, ordering
  resolved on the log's total order). No merge-conflict dialog is ever shown to a user —
  music UX forbids it — **because the domain tolerates imprecise ordering far better than it
  tolerates friction: a human won't notice a re-order landed after their friend's reorder,
  but everyone notices "your edit failed, pull to resolve."** That asymmetry is the entire
  argument for CRDT-ish merging over pessimistic locking.
- **Read path**: playlists sit in the playback hot path (the player needs the next-track URI
  before the current one ends). Playlist pages and their "next items" are therefore cached
  aggressively at the edge (10.distributed-cache) with short TTLs, *plus* push invalidation
  on edit — the only feature where the eventual-consistency window is user-visible enough to
  justify pushing.
- **Derived counters** (followers, total plays) are eventually consistent, computed offline
  by aggregation — never in the edit transaction. Nobody cares that a follower count is
  minutes stale; the edit *itself* must be instantly reflected for the editor.
- **Social graph** (follows, shares, collaborative-playlist membership) is modest-scale relational
  data (bounded per user) — the newsfeed problem (../02.twitter/04-deep-dives.md fan-out)
  exists only in miniature; the activity feed is pulled on app open, not fanned out on write.

- **Trade-off — sync push vs poll.** *Push (event bus → device)* wins latency of collaborative
  edits and cross-device queue-following, *loses* connection maintenance at 200M-user scale and
  breaks silently through mobile NAT lifetimes. *Poll* wins simplicity, loses the "live" feel
  and adds QPS. **We pick push with poll fallback** (long-lived session push when online; poll
  or sync-on-event when the push layer is idle), because cross-device queue handoff is a
  marketed feature whose failure is user-visible, while the poll path keeps the system correct
  when push degrades.

**Failure modes**: (a) two devices offline-editing the same playlist for days → both apply
their logs on reconnect; deletions of the *same* track converge, parallel inserts both land
(op-log wins over snapshot diff — snapshot diff would fight over "position 7"); (b) a
takedown removes a track from a playlist *view* everywhere but the op-log keeps the entry
(tombstoned), so a later license restoration undeletes it — playlists must tolerate absent
tracks rather than "fix" them; (c) event-bus backlog → edits arrive out of order → logs are
rebased by server timestamp ordering, and the client UI never renders an unresolved state.

## 5. Search over 100M tracks (+ artists, albums, playlists, podcasts)

Requirements: *prefix as-you-type* (<100 ms served p99), across heterogeneous entities,
with personalization (your followed artists, your market) and region-gating applied to
results. QPS is modest (~1.7K/s avg per 02) — the difficulty is *latency + quality*, not load.

**Design**:
- **Index is split by entity type and by *vocabulary*:** a typeahead/prefix index (the
  as-you-type experience, built over a compacted term dictionary — FST/trie-style, small
  enough to hold in RAM on edge nodes) answers keypresses; a ranked search index answers
  submitted queries. Different data structures because the access patterns differ: typeahead
  needs *fast prefix enumeration over a small candidate set*, ranking needs *scoring over a
  big candidate set*.
- **Popularity priors + personalization on top**: music search is dominated by
  *"artist + song name"* and *"artist"* queries where the correct answer is the popular
  recording, not the exact-match deep catalog cut — so the first ranking feature is global
  play popularity, corrected per-user (language/market, follows, history) and per-region
  catalog availability. Deep-catalog tail items must still be findable by *exact* match —
  popularity ranking plus an exact-match escape hatch, not popularity alone.
- **Region gating is an index *filter*, not a post-filter.** Options: (i) filter results
  at query time by rights flags (*loses* latency and can return "10 results, 7 dead links"
  pages), (ii) shard indexes per region (*wins* query simplicity, *loses* index fleet size
  ~linearly with markets), (iii) store rights-mask per doc, filter in the retrieval pipeline
  at scoring time. **We pick (iii): a per-document rights bitmap consulted during retrieval —
  it keeps a single index fleet, costs a few branches per candidate, and guarantees the result
  list never contains unplayable items; region-sharding (ii) would 3–4× index infrastructure
  to save almost nothing.**
- **Personalization never reaches into the playback-start path**: ranked search may call the
  recommendation service; *playing a saved track* may not (03).

**Failure modes**: (a) index freshness lag after a Friday release → catalog-service
authoritative rows hold the truth (playable immediately), search lags minutes — acceptable,
but *takedowns* must invalidate search within the same window to avoid dead results; (b)
hot-term thundering herds (a viral artist name) → typeahead index shards serve from cache and
prefix fan-out is inherently shard-replicated (10.distributed-cache); (c) personalization
leak across users (cached per-user results) → typeahead is *never* personalized at the
transport-cache level, only within the served request; region bitmaps are cache-safe,
personal signals are not.

## 6. Licensing-driven catalog constraints (region gating, takedowns, expiry)

The constraint that everything above orbits: the platform *rents* its catalog. Rights are
granted per region, per track, per tier (and differ between streaming and offline-download
windows); deals start, expire and renegotiate continuously.

**Design — rights as first-class data, not application logic**:

- Every track row carries a **right-to-play mask** keyed by (region, tier, media: stream vs
  offline, valid-from/valid-to). The catalog's "exists" and the rights' "playable here, now"
  are *different questions*, and the schema must never collapse them into one flag — a track
  can be present-in-catalog, not-playable-in-JP.
- **The entitlement/resolution layer treats a rights "no" as a normal, fast, cacheable
  answer** — not an error. Geo-gating must be *silent and instant* ("this track isn't
  available in your region" rendered from cached metadata), never a latency-producing
  server round trip, or the whole region-gated catalog punishes only its own users.
- **Takedown = tombstone + fast propagation.** Streaming: the signed-URL issuer denies on
  next request (minutes). Offline: tombstone list pushed in the licence renewal (deep dive 2),
  enforced in hours-to-days. The *asymmetry of these two guarantees is contractual* — the
  design's job is to make sure it's enforced faithfully on both channels, not to pretend
  they're symmetric.
- **Re-adding is as important as removing**: rights come back (a label drops a region ban,
  an album returns from exclusivity). Tombstones must be reversible — playlists, offline sync
  and search must all restore silently. This is why playlists tombstone rather than delete
  (deep dive 4) and why offline manifests are renewed-not-deleted on expiry.

- **Trade-off — where to enforce rights.** *Option A: enforce only at the CDN URL issuer*
  (single chokepoint, simplest audit trail) *loses* the ability to gate cache-hit playback of
  previously-URL'd content — signed URLs' TTL becomes your entire revocation SLA; *Option B:
  enforce at entitlement, at issuer, and in the client's revalidation* (three layers, more
  code) *wins* defense-in-depth where a breach is a legal event, not just a product glitch.
  **We pick B, because a wrong "yes" costs a label contract, not a one-user bug — the
  asymmetry of the failure costs justifies the redundant enforcement path.** Short URL
  TTLs (minutes, not the usual hour-scale of video CDNs) are the mechanism that makes even a
  streaming-side slip bounded.

- **Failure modes**: (a) stale rights-flag cache says "playable" after a takedown →
  TTL shortness + push invalidation on takedown specifically (the only catalog change worth
  an invalidation storm); (b) region mis-detection (VPN, travel) → enforce on *issuer-side*
  region per session, tolerate a play session continuing across a region change (a song
  doesn't stop mid-flight over a border) but re-gate at next track start — silent, and
  matches how rights actually read; (c) ingest mismatch (label feed's track matched to the
  wrong recording) → wrong-rights flags, audible to nobody but legally wrong → the ingest
  pipeline's matching stage is the single most review-heavy component on the write side
  (recall 01's decision: ingest is a correctness problem).

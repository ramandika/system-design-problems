# 04 — Deep dives

## 1. The image pipeline: fetch → decode → cache at the right size (THE dive)

The device's memory is spent on bitmaps or it is wasted. The pipeline is a keyed
request/response system: key = `(image_url, target_size, mode)`, and every stage answers
a request from the previous stage.

- **Decode at display size, never at source size.** View target size comes from the bound
  view's measured width × aspect-ratio-from-metadata. The decoder receives a
  sample factor so a 4,000 × 3,000 source decodes to ~1,000 × 750 (3 MB at ARGB_8888,
  ~1.5 MB RGB_565 for opaque photos — 02's arithmetic) instead of 48 MB. Oversized sources
  (> ~8,192 px on a side, estimate — the common GPU texture-size cap) are rejected before
  decode — the OOM p99.9 guard.
- **Two caches, two tiers, opposite policies.**
  - *Memory tier*: decoded bitmaps, LRU, ~48 MB cap; keys are (URL, size). Holds
    ~16 display tiles — ~5–6 screens of scroll-back at ~2.5–3 tiles visible per screen
    (02's arithmetic). Bitmaps for views still on screen are
    pinned (never evicted while visible).
  - *Disk tier*: raw/variant file bytes, LRU, 512 MB cap. Files, not bitmaps — one decode
    is paid per display-size request; the same file can serve several sizes.
- **Memory-pressure response**: the OS says "trim" (Android `onTrimMemory` levels; iOS
  `didReceiveMemoryWarning`). Response is tiered: at the first level, drop
  prefetched-not-visible bitmaps; at heavier levels, shrink the memory cap (48 → 16 → 4 MB)
  and keep only on-screen tiles; never drop the disk tier (disk is bounded and cheap). The
  degraded cost is a re-decode per tile (~a disk read plus resize, tens of ms, off main
  thread) — a session-long perf dip, not a crash.
- **Cache key collision with the mode**: data-saver requests a different variant, so mode
  is part of the key; changing modes invalidates the memory tier gracefully (disk tier
  holds both variants; evict-on-entry wins over "delete everything" — bounded churn).

**Trade-off.** Option A — cache *decoded bitmaps* only (one tier, memory): instant bind,
loses everything on trim/restart and dedupes only per size. Option B — cache *file bytes*
only: dedupes everywhere and survives restarts, but pays a resize decode on every bind
(~10–50 ms each, estimate) — unaffordable during a fling. **Chosen: both tiers, B-then-A**
— bytes on disk, one decoded copy in memory per (url, size). We pay a modest duplicate
storage (bytes + one copy of decode) for the bind path being a hash hit. This is the
native twin of [Case 19's media pipeline](../19.frontend-chat-app/04-deep-dives.md)
(dive 6): placeholder/aspect-ratio reservation, fetch-by-variant, decode-at-display — the
web case gets it free from the browser's cache; here we build it.

**Failure modes**: (a) *decode callback after view recycled* → bind checks request-key
matches the view's current item; mismatched bitmaps are the classic feed bug (a flicker of
someone else's image). (b) *trim during fling* → pinned on-screen set must be recomputed
before dropping tiers, or the user watches tiles blank and re-decode mid-scroll (accepted
at heavy trim; avoided at light trim). (c) *same image, several sizes* → per-size keys
mean 3 sizes = 3 decodes; mitigated by always requesting one display width per tier of
device and letting the server offer few variants.

## 2. Infinite scroll in RecyclerView/UICollectionView: pagination + scroll-state jank

An infinite list must never let the user reach an unfilled seam, and must never drop a
frame while they fling. The machinery:

- **View recycling is the engine**: a fixed set of view *types* (photo, video, text,
  mixed) with pools per type; a cold pool is a 10–50 ms inflate spike (estimate) at exactly
  the worst moment (fast fling). Mitigation: stable, few types; pre-inflate one extra view
  per type at idle; stable item IDs so reuse doesn't rebind unrelated state.
- **Pagination trigger is position-based, not "last item reached"**: fire the next page at
  < ~5–10 pending items behind the last rendered index (~1–2 s of margin even at the
  ~4 items/s fling speed from 02's frame arithmetic, several times that at a leisurely
  scroll — estimate; comfortably more than one RTT). Trigger *in flight* at most once (state
  machine, not a boolean) — double-fires create duplicate windows and cursor confusion.
- **Direction-aware prefetch**: prefetch decodes ahead in the scroll direction for the
  next ~2 items; cancel (or deprioritize) work for items scrolled *past* — past-items
  decodes are pure waste. The same lesson as Case 19's "render ahead in scroll direction,
  cancel behind."
- **Main-thread discipline**: bind does layout and cached-bitmap attach only. Decode,
  resize, disk I/O, and response parsing all live off the main thread; the frame budget
  (16.7 ms; 8.3 ms at 120 Hz — 02) is defended by *what's absent* from the frame.
- **Height stability**: aspect-ratio reservation from item metadata means an image
  arriving never changes layout (no scroll jump under the finger — the same anchoring
  guarantee Case 19's dive 2 depends on). Unknown aspect ratios (rare, estimate < 5%) get
  a default box with the placeholder pattern.

**Trade-off.** Option A — prefetch pages eagerly (next page always in flight): zero seams
ever, but pays bytes and server load for pages users often never reach, and doubles memory
for buffered rows. Option B — lazy (fetch at seam): zero waste, guaranteed seam jank —
the user sees a spinner at the worst perceptual moment. **Chosen: A gated** — one page
ahead, position-triggered, prefetch paused by the mode governor (dive 4/5). Bounded waste
is the price of an unbroken scroll; the *gate* is what keeps that price from compounding
(12B impressions ÷ 50M DAU at ~30 items/open implies most sessions are short — buffering
10 unused pages per session would be the A-un-gated failure).

**Failure modes**: (a) *cursor race* — a refresh and a page-fetch in flight together yield
overlapping item sets; resolved by stable-ID append with dedup, never by reindexing the
live list. (b) *pool starvation* — a view type that never recycles (e.g., a one-off
animated header) leaks its pool; types are audited by "can this view rebind?" (c)
*fling-through-N-pages* — a super-fling crosses multiple unfetched pages; the protocol
answers with the pages' cursor chain sequentially; if the user lands mid-skip, render
placeholders and fill in place (a bounded hole beats blocking — the same call as
[Case 19](../19.frontend-chat-app/04-deep-dives.md)'s live-gap decision).

## 3. Cold start: lazy init, persisted UI state, the splash contract

Cold start is the most repeated UX moment; every design decision is "what is not on the
critical path."

- **Splash contract**: splash holds until *restored content is painted* (never until
  network). The user's return-launch view comes entirely from the local DB + disk cache —
  rows from the persisted window, bitmaps from the disk tier. A stale-but-instant feed
  beats an empty-but-fresh one; freshness arrives by delta refresh merging *above* the
  anchor (F4/F7).
- **Persisted UI state**: controller saves on background/kill — window rows (cursor,
  items, ~400 B each), scroll position (index + offset within that item), mode, outbox.
  Restore is one indexed read: position = `idx + offset` on stable IDs, safe against the
  delta-merge because the anchor item is fetched first (if it's gone from the server now,
  clamp to the nearest surviving neighbor — position loss is bounded to one item).
- **Lazy init**: the critical path is (DB open, image-pipeline cache open, network stack,
  controller) — everything else (analytics, experiments, players, share SDKs) initializes
  on first use. An init that touches ~150 ms (02's budget) must either be lazy or be moved
  off the first frame; "it's fast, I measured once on a flagship" is the classic cold-start
  lie — mid-tier p95 is the target hardware.
- **Splash timeout**: if restore fails (corrupt DB first run), splash ends at ~1 s with
  skeleton tiles rather than hanging — the contract has an explicit failure shape.

**Trade-off.** Option A — restore-then-merge (above): instant interactive content, costs a
persisted schema and anchor-based merge logic, and risks a visible "new rows appeared
above me" moment (mitigated by the F7 affordance). Option B — always fetch page 1 fresh,
show skeleton: simplest data path, always fresh, but every launch pays full RTT plus
decode before *anything* is interactive — 2–4 s worst case (estimate) and the offline case
(F5) is unservable. **Chosen: A.** The return-visit is the dominant case (8 opens/DAU/day,
02), and "resume where you were" is also a product requirement, not a nicety. B's clarity
is preserved *inside* A: the delta refresh is exactly page-1-fetch; A is B plus a persisted
head start.

**Failure modes**: (a) *stale window after ranking shifts* → position anchor clamps
(bounded by one item); never reposition silently mid-restore. (b) *DB migration on old
installs* → additive, version-gated, done pre-first-frame or the cold-start budget is dead
on upgrade. (c) *restore of a trimmed disk cache* → rows paint with placeholders, decodes
re-race the network — degraded, not broken; restore must tolerate a cache that no longer
has what the rows reference.

## 4. Prefetch gating on network type & battery

Prefetch is the single biggest lever on both UX (no seams, no blank tiles) and cost
(user's data plan, battery). The prefetch engine is a gate, not a queue: *conditions are
evaluated per decision*, with hysteresis.

- **Inputs**: network type (unmetered Wi-Fi vs cellular/metered), battery %, charging
  state, OS battery-saver / data-saver flags (Android Saver/Doze, iOS Low Power Mode),
  mode governor tier (dive 5), scroll velocity/direction, foreground state.
- **Gate matrix** (defaults, thresholds are estimates):
  - Wi-Fi unmetered + normal battery: prefetch 2 images ahead + next page early (aggressive).
  - Cellular + normal battery: prefetch 1 image ahead *only while actively scrolling*;
    next page only at the standard trigger (conservative).
  - Battery saver or ≤ 15% battery: image prefetch off; pagination only on demand.
  - Charging + Wi-Fi: also prefetch the first video segment of the next video item.
- **Hysteresis**: thresholds debounced (e.g., battery flips on/off at 15%/20%) so a
  phone hovering at the edge doesn't oscillate work on/off — the same anti-flap discipline
  as any control system.
- **Cancellation**: every prefetch is a keyed cancellable request; scroll-away cancels.
  Untracked prefetches are how a "conservative" client still burns 3× the bytes (estimate).

**Trade-off.** Option A — always-prefetch (aggressive): best UX, treats the data plan as
free; on cellular users this is the #1 complaint driver in data-sensitive markets, and in
the 02 arithmetic (~9 MB/day images) the un-gated version plausibly doubles or triples it
with unused prefetches. Option B — prefetch only on Wi-Fi: cheap everywhere, but the
majority-mobile user gets seam spinners and blank tiles, i.e., a worse product where
engagement is largest. **Chosen: A gated by the matrix above** — the gate converts a
policy question ("how much do we prefetch") into a per-context decision, and the
hysteresis converts it into stable behavior rather than thrashing. The point is not the
specific thresholds; it's that prefetch *policy becomes state-machine logic evaluated on
device signals*, not a constant.

**Failure modes**: (a) *OS says otherwise* — background prefetch while Doze/lockscreen
gets killed mid-flight; prefetch only runs foreground, background work defers to the OS's
scheduling windows. (b) *gate misreads meters* — "unmetered" detection is per-OS and
fallible (VPNs, captive portals); a wrong read burns data, so the conservative tier is the
fallback on any uncertainty. (c) *thrash* — velocity spikes trigger bursts of prefetch that
cancel each other; rate-limit prefetch issuance (e.g., ≤ 1 burst per ~300 ms, estimate).

## 5. Low-power / data-saver modes

User-visible modes (and OS signals) collapse into one internal **quality tier** that every
byte- and work-spending subsystem consults:

- **Tier 0 (full)**: best variant, video autoplay on, gated prefetch aggressive.
- **Tier 1 (data saver)**: one-width-lower image variant; no video autoplay (poster frame
  + tap-to-play); prefetch 1-ahead only; no video-segment prefetch. Target ≥ 50% byte cut
  (N4) — arithmetic: bytes 0.6× (variant ~40% smaller) × 0.85 (+15% fewer fetched via
  prefetch cut) ≈ 0.5× → ~50% cut on image bytes (estimate), and the video bytes
  (≈ 40% of egress, 02 — 216 TB of the 516 TB) drop to ~zero without a tap.
- **Tier 2 (low power)**: Tier 1 + prefetch off entirely, smaller memory caps, animations
  trimmed, autoplay off, background refresh minimal. Optimized for battery slope, not bytes.
- **Ranking is untouched** — the server is a given and is not mode-sensitive; modes degrade
  *presentation cost*, never *content* (the same "correctness never degrades, only
  latency/perf" line Case 19 draws on eviction).

**Trade-off.** Option A — per-subsystem flags (each feature reads its own settings):
fine-grained, but drifts — image pipeline and video player disagree about what "saver"
means, and a new feature forgets to consult it. Option B — one tier enum consulted
centrally: one source of truth, coarse but consistent; new features get degradation for
free and *cannot opt out silently*. **Chosen: B** — consistency of behavior is what users
actually verify ("did saver change anything?"); per-feature knobs are engineering sugar
that rots. The cost is that a subsystem with a legitimate exception (e.g., the explicit
tap-to-play video) must model itself as a tier-aware decision rather than ignore the tier.

**Failure modes**: (a) *tier flip mid-scroll* — the user toggles saver with images
in-flight; in-flight requests complete, new requests use the new tier keyed by (url, size,
mode); no invalidation storm. (b) *poster-only videos read as "the app broke"* → tap-to-
play affords a clear surface, poster is a real frame, not a gray box. (c) *saver hides
quality regressions* — with saver on, nobody notices the normal tier regressed; keep
byte-count telemetry per tier so the full-tier path doesn't rot.

## 6. Offline feed & last-good state

No network is a *state*, not an error. The offline contract:

- **Last-good snapshot**: the local DB *is* the last good feed (rows + cursors + position,
  persisted continuously — not a separate "offline copy"). Offline render = hydrate from
  DB + disk tier, with a staleness banner ("updated X ago") rather than a spinner. The
  snapshot is bounded (last N pages, ~400 KB for 1,000 rows — 02) so "last-good" has a
  finite footprint and a defined eviction.
- **Interactions**: like/save/share append to the durable outbox (before any UI optimism
  is shown — write-then-render, the only unforgivable order), flushed on reconnect with
  idempotent operation IDs. Server conflict resolution is trivial here (last-write-wins on
  a monotone action — the shape of [Case 03's](../03.whatsapp/04-deep-dives.md)
  at-least-once + client dedup, applied to taps instead of messages).
- **Reconnect**: flush outbox → top-slice refresh → anchor-preserved merge (dive 3's
  restore path, reused — offline recovery is cold start minus the cold, plus an outbox
  drain).
- **Staleness policy**: content older than a bounded window (estimate: 48 h) gets a
  stronger banner and a manual refresh affordance; the client never silently pretends a
  week-old feed is current — but it also never replaces it with a blank screen, because
  the former is the user's only content and the latter is a product-killing moment.

**Trade-off.** Option A — offline = last-good rows + outbox (above): instant always,
costs the persisted window schema, staleness ambiguity, and dedup logic. Option B —
offline = an explicit "you are offline" screen: zero ambiguity, zero cost, but it throws
away the app's own last state and renders a dead end the user cannot scroll. **Chosen: A**
— a feed app's offline minutes are exactly commute minutes; empty-state B converts those
sessions to zero engagement. B's honesty is retained structurally: the staleness banner
carries A's ambiguity out into the open. This mirrors Case 19's outbox + last-good
persistence one layer down (its messages, our rows), and the freshness-reconciliation is
the same anchored merge that paged-timeline serving implies ([Case 02's
timeline](../02.twitter/04-deep-dives.md)), replayed on the client.

**Failure modes**: (a) *outbox replayed twice* (crash between flush and ack) →
idempotent operation IDs server-side make the replay a no-op. (b) *snapshot vs mode tier*
— an offline save under data-saver renders lower-tier variants; keyed by (url, size,
mode), so a tier change post-offline re-decodes from disk rather than showing a wrong-sized
image. (c) *clock-independent staleness* — "updated X ago" must survive device clock jumps
(reboots, travel): store `now_at_save` alongside rows and compute deltas from that pair,
not from wall-clock timestamps alone.

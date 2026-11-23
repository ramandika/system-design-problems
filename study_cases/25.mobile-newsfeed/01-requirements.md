# 01 — Requirements

## The server contract (given, not designed here)

The ranking backend is a given. The client may assume exactly this interface:

- `GET /feed?cursor=<opaque>&n=20` → `{items: [{item_id, score, kind, payloads, aspect_ratio,
  image_variants: [w360, w720, w1080], counts...}], next_cursor}`. Rankings are computed
  server-side (see [Case 15](../15.ml-ranking-system/) for the ranking
  tier's shape); the client *renders a page, it never ranks*.
- CDN-served media keyed by the variants in each item (the client picks the variant).
- Image variant URLs are content-addressed and immutable — hence cacheable forever locally.

Everything else server-side (fanout, ranking, storage) is out of scope — the timeline
*shape* the server contract implies is [Case 02's](../02.twitter/04-deep-dives.md)
time-sorted list; the client is that contract's last endpoint.

## Functional requirements

- **F1. Infinite ranked feed**: scroll down forever; new pages load seamlessly; ranking
  order per page is exactly what the server sent.
- **F2. Image-first rendering**: each item renders full-width imagery at the device's
  display size, with zero layout shift when the real image arrives (aspect ratio reserved
  from item metadata).
- **F3. Interactions**: like/react, comment entry, share, save/offline-flag. Optimistic
  locally, reconciled with the server.
- **F4. Resume where you left off**: after kill/background, the feed opens at (near) the
  last scroll position — not at the top, which silently loses position forever.
- **F5. Offline behavior**: with no network, show the last known feed (bounded staleness)
  and queue interactions; never a blank screen with a spinner as first resort.
- **F6. Data saver / low-power modes**: user-visible modes (plus OS battery-saver)
  that reduce bytes and work: smaller image variants, no video autoplay, truncated prefetch.
- **F7. Refresh**: pull-to-refresh re-ranks; refresh does not destroy the user's position
  mid-list (new content appears above with an affordance, not by yanking the list).

## Non-functional requirements

- **N1. Frame budget**: 16.7 ms (60 Hz), 8.3 ms (120 Hz); dropped-frame rate while
  scrolling a steady feed < 1% (target, not a measurement — the budget the jank work in 04
  dive 2 defends).
- **N2. Cold start**: time-to-*interactive-content* (feed painted with real or cached
  content, scrollable) ≤ 2 s p50 on a mid-tier 4G device (estimate target; platform
  guidance treats 5 s cold start as failing territory — e.g., Android vitals' thresholds —
  we set the bar well above that). First frame ≤ 500 ms.
- **N3. Memory**: total RSS target ≤ ~150 MB for the app (mid-tier, estimate) with the
  bitmap cache capped explicitly (~48 MB default, see 02); must respond to OS memory-trim
  callbacks gracefully (never crash, never visibly reload everything).
- **N4. Data**: default-mode bytes ≤ ~15 MB/day average user (02's arithmetic); data-saver
  mode cuts ≥ 50%.
- **N5. Battery**: no background work except a bounded daily refresh when granted;
  prefetch stops on battery saver or ≤ 15% battery (estimate threshold, hysteresis in 04
  dive 4).
- **N6. Robustness**: no OOM crash at p99.9 under pathological input (a 30,000-px-tall
  image, 50 images on screen via overlapping animations, storage pressure evicting caches).

## Scope decisions (what is explicitly OUT of scope)

- **Ranking, candidate retrieval, feature computation** — server side, given (Case 15's
  territory). The client consumes scores; it never re-ranks.
- **Video upload/creation pipeline, camera, filters** — capture-side; this case is about
  the *feed* client. Video *playback* appears only where it touches the feed (autoplay
  off in data saver; player is treated as a component boundary).
- **Server-side image resizing/CDN internals** — [Case 09](../09.cdn/README.md)'s domain;
  the client just picks variants from the offered set.
- **Push notifications, badges, background sync scheduling** — treated as OS-platform
  plumbing; only the client's *policy* on receiving them is in scope.
- **Web/desktop clients** — web equivalents reuse the same patterns with DOM
  virtualization instead of view recycling (see the client-side machinery in
  [Case 19](../19.frontend-chat-app/04-deep-dives.md)); not designed here.
- **A/B experiment framework, analytics** — assumed present as plumbing; not designed.

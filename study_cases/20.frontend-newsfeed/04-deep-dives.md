# 04 — Deep dives

## 1. Infinite scroll without jank: windowing + edge-triggered pagination

Full mount is dead per [02](02-scale-estimates.md) (~1 GB RAM for a 1 h power session,
estimate). The design is the [Case 19 virtualized-window machinery](../19.frontend-chat-app/04-deep-dives.md)
(size cache, overscan, anchoring) with two feed-specific differences:

- **Heights are mostly uniform, occasionally not.** Feed items cluster around a template
  height (~400 px, estimate); images/ads/video cards and expanded threads vary. Strategy:
  estimate from the template, measure-on-first-mount into the size cache — the same
  estimated-prefix-sums approach as Case 19, but with better priors, so a fling through
  thousands of estimated items keeps cumulative-position error bounded; re-anchoring when
  a measured height replaces an estimate (compensate `scrollTop` by the delta in-frame,
  Case 19's trick verbatim — it's the price of any variable-height window).
- **Pagination is edge-triggered, not threshold-polled.** A sentinel element just past the
  window's end fires mount/intersection callbacks; the store appends page N+1 keyed by the
  anchor token of page N (dive 4's cursor). Race control matters more than here-it-comes:
  (a) only one page fetch in flight per direction — a second sentinel fire while page N+1
  is loading must be a no-op, or a fast fling fires 5 fetches; (b) pages append in token
  order regardless of response arrival order (a retried slow page N+1 must not be
  overtaken by N+2); (c) failed page fetch shows an inline retry affordance *at the edge*,
  never a silent stop — infinite scroll that silently dies is the classic kill-bug.

- **Trade-off**: Option A — windowed list over a paged log (above): unbounded session
  length at bounded memory, but carries size-cache, anchoring, and trim bookkeeping.
  Option B — plain pagination with DOM accumulation ("keep appending HTML"): trivial to
  build, correct for ~100 items, then memory/layout creep kills the long session — and
  feed sessions are precisely the long-session product. Option C — native list
  components (RecyclerView/UICollectionView on mobile): best fling performance for free,
  but splits the codebase per platform. We pick A where one web codebase must serve all
  surfaces, C where a native app exists anyway — the *protocol* (paged log + anchor
  tokens + edge triggers) is identical either way, and that protocol is the portable part.
- **Failure modes**: (a) *trim-regret* — user scrolls up after pages behind the window
  were trimmed; keep a one-page ring buffer above the window, and beyond that jump via a
  re-fetch anchored to the top-mounted item (state: "back to newer posts"). (b) *anchor
  churn during fling* — an image decode resizing a row mid-fling; reserved slots from
  media metadata (aspect ratio travels in the payload, as in Case 19's dive 6) prevent it
  before anchoring is ever needed. (c) *sentinel starvation* — fetch gate (dive 3) closed
  for minutes while the user idles near the edge; on gate-open, re-check distance-to-end.
  A paused gate must never mean a stalled feed.

## 2. SSR vs CSR: stream the paint, hydrate the interaction

The feed's value is content; the interaction plane (like buttons, comment boxes, video
players) is JS. Options at the architecture root:

- **Option A — pure CSR**: a shell HTML + JS bundle, then API fetch, then render. Wins:
  one code path, trivially cacheable shell, no server render cost. Loses: first paint
  waits for (parse 150 KB) + (execute) + (feed API RTT) + (render) — serialized on the
  slowest device on the worst network, i.e. exactly the user the budget targets (< 1.5 s
  p75 fails).
- **Option B — full SSR**: render the first page of items to HTML; browser paints on HTML
  arrival. Wins: fastest meaningful paint (paint ≈ TTFB). Loses: the *interaction*
  mismatch — HTML paints, JS hydrates the whole tree later; until then taps do nothing
  ("painted but dead"), and the server pays render CPU per request on a *uncacheable*
  ranked page.
- **Option C — streaming SSR + partial (island) hydration**: server streams the first
  viewport as HTML with an inline payload (page-1 entries + cursor + variant flags);
  hydration is per-island and scheduled — shell/hydration for the list surface first,
  interaction islands on-visible or on-idle, video players last. Wins: A's cheapness where
  JS isn't needed and B's paint speed where content is; the 150 KB budget is spent on the
  scroll path, not the comment box. Loses: three-way contract (server render, inline
  payload, client re-render) must agree or hydration mismatch errors multiply.

- **Chosen: C**, with the discipline that the inline payload is the *single source* both
  sides render from — the SSR tier emits HTML *and* the JSON that generated it, so client
  hydration is "adopt existing DOM" not "re-render and diff". This is what turns C's cost
  into a testing problem instead of a flicker factory. We accept B's server CPU (SSR emits
  ~one viewport, not the page) and accept a bounded "painted-but-dead" window for
  non-critical islands only — every item's *primary* tap target (expand/open) must hydrate
  in the first shell pass, which is a product decision with an engineering floor.
- **Failure modes**: (a) *hydration mismatch* — server-rendered variant/coin differs from
  client assignment; prevented structurally by dive 6 (variant travels in the payload).
  (b) *stream stalls* — SSR chunked response hanging behind a slow hydration service call;
  the stream must have a deadline and degrade to shell-only paint (still a win vs CSR).
  (c) *dead-island taps* — user taps an unhydrated button; the shell must either defer the
  event (queue + replay on hydration) or show a skeleton; silent swallowing is the bug
  users report as "the app ignores me."

## 3. Prefetch & predictive pagination: spend bytes the way scroll speed predicts

Pagination on response (fetch page N+1 *when* the sentinel fires) means every edge
crossing eats a full RTT of visible spinnner. The fix is *predict*: start the fetch when
predicted time-to-reach-edge drops under the fetch latency budget.

- **Viewport prediction**: maintain scroll velocity (EMA over frames, robust to flings);
  predict time-to-prefetch-point = (distance_to_edge − buffer) / velocity; fire when that
  estimate < p75 page-fetch RTT (~200–400 ms, estimate, mobile). Buffer ≈ 2 viewport
  heights. During a fling, velocity is high → fetch fires early and covers the landing
  point; slow browsing fires late → few wasted pages.
- **Gating**: predictive prefetch *wastes* bytes when prediction is wrong (user stops,
  reverses, leaves). So fetches carry a gate: (a) effective connection — full-speed
  prefetch only on reported fast/uncapped connections (`navigator.connection`-class
  signals or platform equivalents); on metered/slow, fall back to edge-triggered only.
  (b) `Save-Data`-style user preference and OS data-saver honored absolutely. (c) battery/
  foreground state — background tabs prefetch nothing. (d) Media (video especially) is
  gated a notch stricter than metadata: item JSON prefetch is cheap ([02](02-scale-estimates.md):
  ~11 KB/page wire); a 2 MB video is not.
- **Impression hygiene**: prefetched ≠ viewed. Impressions count on viewport dwell, never
  on fetch — otherwise prefetch inflates the ranker's training signal
  ([Case 15](../15.ml-ranking-system/README.md) consumes these beacons; a prefetch-shaped
  bias in them is a *ranking quality* bug, not just a metrics bug).

- **Trade-off**: Option A — aggressive prediction (fetch at ≤ 3 viewports ahead whenever
  gate open): scroll feels telepathic; wastes an estimated ~30–50% of prefetched pages
  (estimate — reversal/abandonment) and multiplies API load by up to 2× at 200M-user scale
  ([02](02-scale-estimates.md): ~55K QPS baseline; every ×2 is a backend project).
  Option B — conservative (edge-trigger only): zero waste, but the feed wears its RTT on
  every page boundary — jank-adjacent. We pick a middle: predictive on fast + Save-Data
  absent, with the threshold adaptive (start at 2 viewports; widen when consecutive
  predictions were consumed, tighten after repeated non-consumption). The per-user
  adaptivity is what "smart" prefetch means; the gate list is what makes it deployable.
- **Failure modes**: (a) *wasted-video prefetch* — most expensive class; video islands
  prefetch poster frames + first segment only, never the full asset, until visible.
  (b) *prefetch storm on gate flapping* — connection quality oscillating (subway Wi-Fi)
  retriggers; debounce gate transitions with hysteresis (require N seconds of stable
  fast-class before re-enabling). (c) *beacon inflation from prefetched-but-rendered
  items* — items mounted ahead of the viewport in overscan must not count as dwell; the
  winder's overscan rows are beacon-exempt, enforced in one place (the impression
  checker), not in each component.

## 4. Ranking changes vs a stable UI: the mutating-floor problem

The ranked list the backend serves is **not immutable**: ranker updates, new posts outrank,
ads inject at fractional positions. Two client failures fall out:

**(a) Pagination dupes/skips.** Offset cursors ("give me items 30–45") count positions in
a *moving* list: an insertion above shifts everything, so page 3 re-returns seen items
(dupes) or skips unseen ones. **Fix**: the page token anchors to *the last entry of the
previous page* — "entries ranked before anchor E (and not seen this session)". This is the
client-side inheritance from [Case 02's time-sortable Snowflake cursor](../02.twitter/04-deep-dives.md):
sortability/stability of the anchor is everything, offsets are structural bugs here. Residual
rank-order drift between pages (items that should appear "between" pages 2 and 3 but land
in page 4) is unavoidable without a server-held frozen snapshot — accepted: the top of the
feed is fresh, the deep feed is approximately-ordered, and no one can tell (the same
"nobody can tell" bar as Case 02's delete-tombstones). Dedup-by-entry-id in the store is
the safety net; ordering drift is absorbed, silently.

**(b) Insertion scroll-jumps.** Three items outrank the current top. Naïve `prepend` +
implicit scroll anchoring produces one of: content jumping under the finger (the exact
bug Case 19's scroll anchoring kills), the reading position silently sliding down, or a
browser-scroll-anchoring fight with the framework's list diffing. **Fix**: position-aware
insertion policy —
- items ranking *below* the current viewport bottom: insert immediately (no visible
  effect; anchor-safe);
- items ranking *above* the viewport top: never insert mid-read. Hold them in a pending
  inset; render the "N new posts" pill pinned above the list. Insertion happens on
  scroll-to-top or pill tap, and the viewport is explicitly anchored to the first
  previously-visible item across the insert (compensate `scrollTop` in the same frame).

- **Trade-off**: Option A — always insert immediately with full anchor compensation: the
  freshest possible list mid-scroll; loses reading stability — the user's place moves,
  which on a feed reads as "the app is haunted" and measurably hurts long sessions
  (estimate: any involuntary position shift > 1 viewport is a rage-quit-shaped event).
  Option B — pill + consent: stable reading, at the cost of staleness (median "new posts"
  age grows if users ignore the pill). We pick B for above-viewport changes and the
  un-pilled immediate path for below-viewport ones: the split is cheap (one boundary test:
  is the insertion point above viewport top?) and it matches what the large feed products
  visibly do ("new stories" buttons are public UI, not speculation).
- **Failure modes**: (a) *pill flood* — the ranker injecting 50 items in 10 min makes the
  pill lie by aggregation ("100 new posts" while the top 3 are duplicates of what the user
  just saw); cap and dedup the inset against the visible log's entry IDs. (b) *anchor
  drift across insert* — compensation assumes static heights; if an image above the
  viewport loads during the same frame, apply both deltas then re-anchor once — order the
  writes (single-writer principle from Case 19's architecture applies to scroll state too).
  (c) *ads at fractional positions* — injection indexes ("insert at rank 3.5") can
  collide with page boundaries; injection positions must be resolved against the *store's*
  current page layout at append time, not against an abstract rank index, or ads
  teleport between renders.

## 5. Interaction latency: bundle hygiene is a budget, hydration is the price

Budgets from [01](01-requirements.md): tap → visible < 100 ms perceived; no long tasks
(> 50 ms) on the scroll path. The enemies, in order of harm at 200M DAU:

- **Critical-path JS**: 150 KB gz budget means, on a low-tier phone, an estimated
  ~150 ms parse/compile + execution *before the app can respond at all* — every KB added
  is paid by every session ([02](02-scale-estimates.md)). Discipline: the shell carries
  the store, winder, telemetry, router — and nothing else. Ads SDK, video players, and
  comment surface are islands loaded on-demand (dive 2's registry). Route-level and
  *feature-level* code splitting; no library is imported "because a component might use
  it" on the critical path.
- **Long tasks from hydration**: hydrating the whole list at once is a 100+ ms main-thread
  task mid-scroll (estimate) — self-inflicted jank exactly at the moment of interaction.
  Island scheduling (on-visible / on-idle via `requestIdleCallback`-class scheduling)
  bounds each hydration slice; the work-stealing rule: when the user is scrolling,
  hydration yields to frames.
- **The interaction path itself**: like/comment taps are optimistic store mutations + one
  paint, network reconciled later (Case 19's optimistic pattern minus the ordering
  problem). No tap handler waits on a network, a layout read, or another island's
  hydration.

- **Trade-off**: Option A — one bundle, hydrate everything up front: simplest mental
  model, all interactions instantly live after TTI; loses the paint budget (critical-path
  KB × every user) and creates hydration long tasks. Option B — aggressive islanding with
  deferred hydration: budget-clean; loses "instantly live" for deferred surfaces, and
  adds mismatch/deferred-tap complexity (dive 2's failure modes). We pick B, with the
  product-floor exception that *primary* tap targets hydrate in the first pass —
  engineering budgets lose to perceived-brokenness when they conflict, so which surfaces
  are "primary" is a written contract, not an accident of split boundaries.
- **Failure modes**: (a) *budget erosion* — the 150 KB number decays a few KB per
  sprint; keep it as a CI-enforced budget with per-chunk allocations, or it becomes a
  wiki page. (b) *island waterfall* — an island that needs another island's module chain
  turns "deferred" into "cascading"; enforce depth ≤ 2 in the registry and bundle
  island dependencies together. (c) *third-party SDK bloat* — ads/analytics SDKs arrive
  with their own bundles and main-thread habits; load them late, wrap their timers,
  and hold a long-task budget for them separately — they are the usual suspect when
  scroll jank appears in production but not in dev (estimate from the recurrence of this
  pattern; it's the first thing to check, not a measurement).

## 6. A/B on the client without flicker

Requirement: a variant surfaces from the first frame, stays sticky across sessions, and
produces trustworthy exposure logs (the ranker's eval depends on them —
[Case 15](../15.ml-ranking-system/README.md)). The naive client-side design fails twice:

- **Assign in JS after paint**: variant computed post-hydration → first frame renders
  control, next frame swaps to variant → visible flash + layout shift (CLS from the
  experiment itself) + interaction states built for control mis-prime. Fatal by our own bar.
- **Assign in JS before paint**: a blocking script computes the bucket pre-render. No
  flash, but: assignment logic ships on the critical path (dive 5 budget), and the CDN
  can no longer cache SSR HTML keyed only by URL — unless variants are cache-safe, every
  cache decision now depends on experiments.

**Chosen: assignment at the edge, embedded in the SSR payload** (03's A/B decider): the
edge hashes a sticky key (user/anonymous ID + experiment salt) and writes the variant
flags into the streamed payload *before* HTML exists, so server render, first client
render, and hydration all read the same variant with zero flash. Assignment logic lives
off the client's critical path; the client experiment module reads, exposes, and logs.

The cost must be stated and managed:
- **Caching collision**: SSR outputs are now variant-stamped, threatening CDN cache keys.
  Fix: variant flags live in the *inline payload + data-attributes*, not in cache-level
  HTML identity where possible; where the variant *is* the HTML (layout experiments),
  those routes are split into their own cache partitions or bypassed to the SSR tier —
  correctness of experiments and cache hit-rate genuinely fight, we pay cache-efficiency
  on experiment-heavy surfaces rather than integrity of either.
- **Sticky bucketing**: hash of a stable ID keeps assignment stable across sessions and
  devices (identity permitting); re-bucketing mid-experiment corrupts exposure data
  worse than a small sample ever could.

- **Trade-off**: Option A — edge assignment (above): zero flicker, one assignment source
  of truth, off the client budget; couples the SSR hot path to the experiment service
  (latency dependency on the decide call — mitigated by assigning *before* SSR begins
  and by failing closed to the default variant on decider timeout: the control experience
  is always shippable). Option B — client pre-paint assignment: decouples the edge, works
  for no-SSR surfaces; pays critical-path bytes (dive 5) and duplicates assignment logic
  in two places (a consistency hazard). We pick A wherever SSR exists, B as the defined
  fallback for pure-CSR shells — the *contract* (variant fixed before first paint, sticky,
  exposure-logged once per surface) is identical, which is what keeps the data clean
  across both paths.
- **Failure modes**: (a) *decider outage with fail-open to variant* — the wrong default;
  fail **closed** (control) is the rule: an experiment degrades to less data, never to a
  broken UX. (b) *double exposure* — a surface re-renders and re-logs exposure; exposures
  logged once per (user, experiment, surface) per session, idempotent by key. (c)
  *variant leakage across surfaces* — list-order variants leaking into component-style
  variants through shared assignment state; experiments declare their surface scope and
  the client treats cross-scope flags as unknown, not as false.

# Case 25 — Mobile System Design: Newsfeed

**The system**: the *native mobile client* of an infinite, image-heavy newsfeed app
(Instagram/TikTok-lite shape) — the application on the phone, not the servers behind it.
The ranking backend is a *given*: it exposes one contract — `GET /feed?cursor=...` returns
a ranked page of items (IDs, scores, image variants, metadata). This case designs what
runs inside the app: the image pipeline, the list machinery, cold start, prefetch policy,
power/data modes, and the offline state.

**Why it's hard** (all client/device problems):

- **Images dominate everything.** One full-width decoded tile at display size is ~3 MB of
  bitmap; a naive full-res decode is ~8–50+ MB (estimate, see 02). The memory budget of a
  mid-tier phone is spent on bitmaps, not on feed data — 1,000 feed rows are ~400 KB, one
  screenful of 2×-oversampled decodes (2,160 × 1,400 px × 4 B ≈ 12 MB/tile,
  × ~2.5 tiles, estimate) is ~30 MB. A feed client is a memory-cache system
  wearing a UI.
- **Infinite scroll at 60–120 Hz.** Recycle views, decode off the main thread, prefetch in
  the scroll direction, paginate before the user reaches the seam — all inside a 16.7 ms
  (8.3 ms at 120 Hz) frame budget. Every mistake is a dropped frame the user feels.
- **Cold start is the most repeated UX moment.** Every launch replays the same question:
  what can be painted *before* any network byte arrives — persisted last feed, cached
  bitmaps, placeholders — and what work can be removed from the critical path entirely
  (lazy init).
- **The device is not yours.** Metered networks, battery-saver mode, OS memory trims (the
  app can be told mid-session that it's using too much), background-execution limits,
  caches wiped under storage pressure. Prefetch policy is a negotiation with the OS and
  the user's data plan, not a feature flag.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: decode-at-display-size + two-tier image cache under memory
pressure, direction-aware prefetch with cancellation, persisted-UI-state cold start,
prefetch gating on network/battery with hysteresis, explicit degraded modes (data saver /
low power), and offline last-good state with an interaction outbox. These are the
client-side counterparts of the server-side patterns in
[Case 02](../02.twitter/04-deep-dives.md) (timeline shape) and the web-client
sibling [Case 19](../19.frontend-chat-app/04-deep-dives.md) (media pipeline, offline
queues, scroll machinery).

**Try it first**: before reading 03/04, answer: (1) a screen is 1080 px wide, the source
image is 4,000 px — how many bytes does the bitmap cost in each case, and who downsamples?
(2) The user is flinging at 10 tiles/s — what must NOT run on the main thread, and what do
you do with decodes for tiles that just flew past? (3) The app was killed 6 hours ago.
At next launch, what exactly is on screen at t=0?

**Primary sources**: none vendor-specific — this is a synthesis case. All numbers are
labeled estimates; platform facts (memory-trim callbacks, view recycling, Doze/battery
saver) are standard OS behavior, not statistics.

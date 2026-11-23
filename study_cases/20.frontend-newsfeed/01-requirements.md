# 01 — Requirements

The ranking backend is Case 02/15's system and is a **given**: the client consumes a
timeline API that returns ordered feed entries (IDs + hydrated payloads). This file states
what the *client surface* must do and how fast it must feel. All budgets are **design
targets** (estimates to engineer against), not measured facts.

## Functional requirements

- **Infinite scroll feed**: mixed content — text, images, video, link cards, ads and
  system-payload items (injections) interleaved by the ranker. Scroll down forever; old
  items are fetched lazily in pages.
- **New stories above**: while reading, fresher ranked content appears above; surfaced as
  a pill ("N new posts") that inserts on demand — never mid-scroll without consent.
- **Interactions per item**: like/react, comment entry, share, expand (thread, media),
  follow/mute author, hide ad. Each must respond within the interaction budget (NFR below).
- **Pull-to-refresh / top navigation**: explicit reload gets a fresh ranked page from the
  top (new ranking snapshot), replacing the pill state.
- **Media**: images and video stream in lazily with reserved layout space (no layout
  shift); video plays when tapped/autoplays in viewport according to product rules.
- **Session continuity**: returning to the tab resumes the scroll position and feed state
  within the session; cross-session resume is best-effort (jump to a recent rank anchor).
- **Experimentation**: every rendered surface (list order, component styles, injection
  logic) is experiment-treatable, assigned per-user with sticky bucketing, zero perceived
  flicker, and exposures logged.
- **Telemetry**: item impressions (viewport dwell), scroll depth, interaction events,
  Core-Web-Vitals-style field metrics — the ranking backend's training and eval data
  partly *is* this pipeline's output.

## Non-functional requirements (client budgets)

| Latency/memory | Target | Why |
|---|---|---|
| First meaningful paint (feed stream visible) | < 1.5 s p75 on mid-tier mobile, < 2.5 s p95 | the product is the content, not the chrome |
| Time to first scroll interaction | < 3 s p75 | users scroll before anything else |
| Page fetch → new items painted | < 300 ms after response, items < fold+overscan | scroll feels connected to the wire |
| Scroll jank | 60 fps: < 16.7 ms/frame on 60 Hz hardware | infinite scroll is one continuous animation |
| Tap → interaction visible | < 100 ms perceived; long tasks > 50 ms eliminated on the scroll path | interaction latency = trust |
| New-stories insertion | zero scroll-jump when done below viewport; explicit pill above | dive 4 is entirely about this |
| Layout shift (CLS-style) | ≈ 0 from client decisions | media slots reserved from metadata |
| Initial JS on critical path | ≤ ~150 KB gzipped parse+execute before paint | mid-tier phones parse JS at an estimated ~1 ms/KB |
| Steady-state memory (1 h session) | ≤ ~250 MB including decoded images | phones; windowing bounds it |
| Device media/text cache | ≤ ~500 MB, evictable, purely a latency optimization | |

The budgets shape the architecture: paint-affecting work may not wait on JS hydration
(hence streaming SSR + partial hydration); scroll-path work may not touch layout except
batched window updates (hence windowing); nothing on the interaction path may wait on the
network (hence optimistic interaction states).

## Scope decisions

**In scope**: the client surface — windowed list over a paged feed log, pagination
cursor protocol against a *mutating ranked list*, streaming SSR / hydration strategy,
prefetch and predictive pagination, scroll-anchored insertion, bundle & interaction
budget hygiene, client-side experimentation (assignment, flicker-freedom, exposure logs),
impression/telemetry pipeline, per-device caches.

**Explicitly out of scope** (each is its own design problem elsewhere):

- Ranking, candidate generation, fanout-on-write/read, timeline caching —
  [Case 02](../02.twitter/04-deep-dives.md) and [Case 15](../15.ml-ranking-system/README.md).
  We consume their contract (ordered entries + hydration + a stable page token).
- Media origin/CDN delivery internals — [Case 09](../09.cdn/README.md). We consume CDN
  URLs and do client-side caching/decoding choices.
- Ad *selection* and auction logic — the client renders whatever item the ranker injects,
  and reports impressions; targeting is not ours.
- Notification systems, feed item composition/authoring tools, profile pages.
- The chat/DM surface — its own client case ([Case 19](../19.frontend-chat-app/README.md)),
  which shares the windowing vocabulary but not the ranked-mutation problem.

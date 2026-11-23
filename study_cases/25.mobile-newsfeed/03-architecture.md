# 03 — Architecture

## High-level shape

The client is three engines plus a store, coordinated by a scroll-state-driven controller.
The server side is a given; only its contract appears here.

```
                     ┌───────────────────────────────────────────────────┐
                     │                    THE DEVICE                     │
                     │                                                   │
 ┌──────────┐        │  ┌──────────────┐     ┌─────────────────────────┐  │
 │  Splash  │──────► │  │ Cold-start   │     │  Feed UI                │  │
 │ contract │        │  │ coordinator  │     │  (RecyclerView /        │  │
 └──────────┘        │  │ (lazy init,  │     │   UICollectionView)     │  │
                     │  │ restore-UI)  │     └───────────┬─────────────┘  │
                     │  └──────┬───────┘                 │ scroll events, │
                     │         │                         │ bind(view,i)   │
                     │         ▼                         ▼                │
                     │  ┌─────────────────────────────────────────────┐  │
                     │  │            FEED CONTROLLER (state machine)  │  │
                     │  │  window = [cursor, items...]                 │  │
                     │  │  states: idle / fetching-page / offline      │  │
                     │  └──┬───────────────┬──────────────┬───────────┘  │
                     │     │               │              │              │
                     │     ▼               ▼              ▼              │
                     │  ┌─────────┐   ┌───────────┐  ┌──────────────┐   │
                     │  │ Local   │   │  IMAGE    │  │  PREFETCH    │   │
                     │  │ feed DB │──►│  PIPELINE │◄─│  ENGINE      │   │
                     │  │ (SQLite │   │ mem LRU → │  │  (gated by   │   │
                     │  │ /Core   │   │ disk LRU →│  │  Network &   │   │
                     │  │  Data)  │   │ decode →  │  │  Battery &   │   │
                     │  │ rows +  │   │ request   │  │  Scroll &    │   │
                     │  │ cursor  │   │ queue     │  │  Mode)       │   │
                     │  │ + UI    │   └─────┬─────┘  └──────┬───────┘   │
                     │  │ state   │         │               │           │
                     │  └────┬────┘         ▼               ▼           │
                     │       │        ┌──────────┐   ┌────────────┐     │
                     └───────┼────────┤   CDN    │   │ /feed API  │─────┼──┐
                             │        └──────────┘   │ (ranking   │     │  │
                             │              ▲        │  backend:  │     │  │
                             └──────────────┼────────┘  GIVEN)    │◄────┼──┘
                                            │                     │     │
                                            └── image variants ───┘     │
                                                                        │
                          interactions (like/save) ◄── outbox queue ────┘
```

## Components

- **Splash contract (launch shell)**: a fixed rule — the splash screen shows until either
  (a) restored content is painted (no network needed) or (b) a timeout (~1 s of *rendered
  frames*, not of waiting). Never blocks on network. See 04 dive 3.
- **Cold-start coordinator**: owns app-level init ordering. Critical-path init is explicit
  and tiny; everything else (analytics, experiment framework, players, emoji tables) is
  lazy-initialized on first use. Restores persisted UI state from the local DB.
- **Feed controller (the state machine)**: the single owner of the scroll window: the
  item list, the current cursor, fetch state, offline state. Views are dumb; they bind to
  what the controller serves. All fetches and prefills go through it — one writer, the
  same single-writer discipline [Case 19](../19.frontend-chat-app/03-architecture.md)
  applies to its sync engine.
- **Local feed DB**: rows (item metadata, ~400 B each), the fetched pages' cursors, the
  last scroll position, and the interaction outbox. Durable; the source for offline
  last-good state and cold-start restore.
- **Image pipeline**: request-keyed, three stages — memory LRU of decoded display-size
  bitmaps (~48 MB cap) → disk LRU of file bytes (512 MB cap) → network fetch → decode at
  target size off the main thread → hand the bitmap to the view. Responds to OS memory
  trims by shrinking the memory tier. See 04 dive 1.
- **Prefetch engine**: decides *what to fetch before it's visible* (next page of rows,
  next images in scroll direction) and *whether* (network unmetered? battery? mode?
  scrolling?), with cancellation on scroll-away. See dives 2 and 4.
- **Mode governor**: data saver / low power / OS battery-saver collapse into one internal
  "quality tier" signal that the image pipeline (variant choice), prefetch engine, and
  video autoplay all consult. See 04 dive 5.
- **Interaction outbox**: like/save/share taps are appended durably, flushed in background
  windows, reconciled on reconnect (optimistic counters locally). Same shape as Case 19's
  outbox.

## Data flows

1. **Cold start**: splash → coordinator reads DB (last window + position) → controller
   hydrates rows → grid binds with *cached bitmaps or placeholders* (no network) → first
   frame < 500 ms → concurrently, page-1-refresh request is in flight → response merges
   (new items above, position preserved via anchor). UI is interactive before any network
   byte arrives.
2. **Scroll down**: adapter requests binds; controller reports scroll position to the
   prefetch engine; prefetch gates, then issues image-decode-ahead for the next ~2 items
   in the scroll direction (and, at a threshold — e.g., < 30% of the last page remaining —
   the next page fetch with `next_cursor`). Images fly in via the pipeline; decodes the
   user already scrolled past are cancelled.
3. **Scroll to seam**: page fetch resolves → controller appends rows → adapter notifies
   with stable IDs → recycling continues without layout jumps.
4. **Background/kill/resume**: on background, controller persists window + position +
   outbox; on resume, persists are read, a *delta* refresh (small page, top slice) is
   fetched, merge preserves position (the same paged-timeline shape as
   [Case 02](../02.twitter/04-deep-dives.md)'s timeline, anchored client-side).
5. **Offline**: fetches fail → controller switches to offline state → last-good window
   from DB renders with a staleness banner → interactions go to the outbox → on reconnect,
   flush outbox, then refresh.
6. **Memory trim (OS callback)**: image pipeline shrinks memory tier (e.g., 48 → 16 MB),
   prefetched-but-not-visible bitmaps drop first, on-screen tiles reload from disk if
   needed. The session survives; at worst the user re-decodes a tile (a disk-read + decode,
   ~tens of ms hidden above the fold).

## What the client does NOT do

- No ranking, no retries-with-new-scores of its own, no second ordering. The window is the
  server's ranking, paged by opaque cursor — the client's job is fidelity to it under
  device constraints, not interpretation.

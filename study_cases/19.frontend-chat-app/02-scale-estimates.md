# 02 — Scale estimates (from the device's point of view)

Server-side numbers live in [Case 03](../03.whatsapp/02-scale-estimates.md); here we size
what one device holds, renders, and transmits. Assumptions are order-of-magnitude estimates;
change one and redo the arithmetic.

## Assumptions

- **Power user profile** (the Discord-like case; a WhatsApp-like pure consumer is 10–50×
  lighter): ~30 conversations, ~10 "active" (locally kept in full), a few very hot
  (community) channels.
- **Stored history per device**: ~100k messages across active conversations.
- **Message row size**: ~200 B payload (text + envelope) + ~100 B metadata/index → ~300 B
  per message stored locally.
- **Traffic per user**: ~50 received + ~50 sent messages/day (a heavy consumer profile;
  publicly reported totals for consumer messaging apps work out to roughly tens of messages
  per user per day — treat our 100/day-power-user split as an estimate, not a measurement).
- **Media**: ~20% of messages carry media; ~1 MB average after client-side compression.
- **Typical row height on screen**: ~120 px average (quotes/images make it highly variable).

## Local storage

- Messages: 100,000 × 300 B ≈ **30 MB** text store; with indexes, fragmentation and
  WAL overhead, budget **~60 MB on disk**. Trivial. The design question is never total
  capacity — it's *write batching and eviction per conversation* (see 04 dive 3).
- Growth: 100 msgs/day × 300 B ≈ 30 KB/day → the 100k history takes ~3 years of power
  usage to accumulate. History is effectively unbounded by storage; it's bounded by the
  *view*, not the DB.
- Media cache: 20% of 100 msgs/day ≈ 20 media/day × 1 MB ≈ **20 MB/day** downloaded. A
  1 GB LRU cache holds ~50 days of media; after that, cache misses re-fetch from CDN. Cache is pure
  performance — evicting it costs latency, never correctness.

## Render cost: why 100k is a number that matters

- Naive render of all messages: 100k DOM nodes ≈ 1–2 KB per node (engine-dependent
  estimate) ≈ **100–200 MB RAM**, plus full layout O(n) — first paint of 100k nodes takes
  seconds (estimate) and every mutation relayouts. Dead on arrival; hence windowing
  (04 dive 2).
- Virtualized steady state: ~40–60 mounted rows × ~2 KB ≈ **≤ ~200 KB of nodes**.
- **Height cache**: one number per message (Float64) + integer key. A JS `Map` costs
  ~50–100 B/entry → 100k entries ≈ **5–10 MB** — acceptable, but it's the largest
  render-side structure; typed-array storage (seq-indexed) would cut it ~10× if needed.
- Scroll-through-the-whole-log: 100k rows × 120 px ≈ **12M px of scroll distance**.
  A fling at ~3,000 px/s crosses ~25 rows/s (3,000 ÷ ~120 px/row); a hard peak fling at
  ~10k–20k px/s (estimate) ≈ 80–170 rows/s — far from "thousands of rows", but still enough
  that most rows flown past were never measured. Heights for unmeasured regions must come
  from *estimates*, refined on demand.
- Frame arithmetic: at ~3,000 px/s and 60 fps the viewport advances ~50 px/frame ≈ 0.4 new
  rows/frame; even the ~80–170 rows/s peak is ~1.4–2.8 rows/frame. (A 5–10 row reflow batch
  would imply a ~36k–72k px/s fling — not a state to design the steady frame budget
  around.) Reflow of 1–3 new rows costs ~1–2 ms (estimate) — fits the 16.7 ms budget, but
  only if batched and only if nothing else (image decode!) runs on the main thread.

## Traffic

- **Steady state**: 100 msgs/day × 300 B ≈ 30 KB/day of messages per user. Nothing.
- **Control-plane churn is the real inbound load**: typing indicators ~1 event/3 s per
  typing user (throttled), presence pings, receipts. A hot 50-person conversation with ~5
  people typing at once ≈ 5/3 s ≈ 1.7 events/s; across 20 visible conversations → **tens
  of events/s inbound, each trivial, collectively requiring coalescing** — a 50-member
  group can generate more inbound events than messages. This is why typing/presence are
  rendered from ephemeral state, never persisted.
- **Keep-alive**: WS ping every ~60 s (below typical carrier NAT timeouts ~minutes; the
  trade-off mirrors Case 03's server-side argument, from the battery's side): ~60 B/min.
  Negligible bytes, but it is what keeps the connection — and the *live* feeling — alive.
- **Reconnect**: mobile foregrounding reconnects 1–3×/hour (estimate). Each reconnect =
  resume handshake (~1 RTT) + one batched sync request for all conversations (~1 RTT).
  Design the protocol so reconnect cost is O(1) requests, not O(conversations).

## Backfill arithmetic (drives dive 4)

- **Typical**: offline/away 12 h, receiving ~50/day → ~25 missed messages × 300 B ≈
  **~8 KB** per day of absence. Fine as one response.
- **Worst case**: a 1,000-member community channel sustaining ~5 msg/s (peak-of-peak
  estimate). 8 h away = 5 × 3,600 × 8 ≈ **144k messages ≈ ~43 MB** of backfill for one
  channel — unacceptable on any client. The protocol must support a *truncation point*:
  "skip ahead to seq N, accept the gap" with a jump marker in the UI. No client protocol
  survives hot channels without this.

## Memory arithmetic (drives dive 6)

- One decoded 12 MP photo = 12M px × 4 B = **48 MB RGBA** — 20% of the whole memory budget
  for a single image, if decoded at full size at display width ~300 px. Decode-at-display
  size (~0.5 MB per thumbnail) is not an optimization, it's the design.
- Media bitmaps therefore dominate memory, not messages, not DOM. Everything else in the
  budget is rounding error compared to one naive image decode.

## Consistency check

- 100k messages = 30 MB stored, ≤ 10 MB height cache, ~200 KB of live DOM — the *client*
  is nowhere near any hard limit; every design choice in 04 is about **latency and
  correctness**, not capacity. That's a useful takeaway: frontend scale problems are
  jank-and-state problems, not terabyte problems.

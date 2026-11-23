# 02 — Scale estimates (device first, then the sync contract it implies)

All numbers are order-of-magnitude estimates; change an assumption and redo the arithmetic.
No public per-event statistics for consumer calendars are cited as measurement — where a
figure is publicly known in shape only (e.g., calendars share workday-heavy traffic), it's
labeled as shape, not size.

## Assumptions

- **User profile**: a working professional, ~15–20 meetings/events added per week
  (~1,000 events/year, estimate). Personal + subscribed calendars: 1 personal (writable),
  ~10 read-mostly subscriptions on a work account.
- **Device store**: 5 years of history → ~5,000 stored event rows (~15% recurring masters
  → ~750 series + ~4,250 singles; each series carries a rule and typically a few exception
  rows, say ~2 → ~1,500 exception rows).
- **Row size**: event ≈ 400 B (title, times, tz IDs, location, attendees metadata, rule);
  exception row ≈ 300 B; attendee list adds ~50 B per attendee on top of the base row.
  Wire payloads are sized at ~500 B/op below (envelope, client op ID, sync-token
  metadata on top of the ~400 B stored row) (estimate).
- **Visible density**: personal month view ≈ 300–500 occurrences; "everything" view over
  10 subscribed calendars ≈ **1,000–2,000 occurrences in one month grid** (the render
  problem in 04 dive 6 is sized from this, not from the personal view).

## Local storage

- Events: 5,000 × 400 B ≈ **2 MB**; exceptions + attendees + indexes ≈ ×2 → budget
  **~5 MB on disk**. Trivial — like Case 19, capacity is never the design question;
  *correctness under sync and latency* is.
- Occurrence expansion cache: for the current ±3 months, ~750 masters expand to roughly
  750 × 12 occurrences ≈ 9,000 occurrences × ~250 B (expanded start/end per window) ≈
  **~2 MB**; this is a pure cache — evictable, rebuilt on demand (04 dive 3).
- Layout cache (04 dive 6): one layout record per (day × view) rendered ≈ a few hundred
  bytes per day → **< 1 MB** for months of cached day layouts.
- Growth: 1,000 new events/year × 400 B ≈ 400 KB/year of new data. The store is bounded
  for practical purposes by history retention policy, not by space.

## Sync traffic (shapes the sync engine design)

- **Per session**: app opens ~6×/day/user (estimate); each open = 1 delta pull using the
  stored sync token + 1 push-flush of any queued mutations; ~2 extra foreground refetches
  per session (returning to the app). → ~18 pulls/day, ~6 pushes/day per user.
- **Delta pull size**: steady state ~10 changed events/day/user × 500 B ≈ **5 KB/pull**
  typical; an offline-3-days device pulls 3 × 10 events × 500 B ≈ 15 KB; the pathological case (first login,
  full initial sync of 5 years ≈ 5,000 events × 500 B ≈ 2.5 MB + series) is the one that
  must be paged/tokenized, not optimized away.
- **Push size**: 6 mutations/day × ~500 B ≈ 3 KB/day. Nothing.
- Device-level conclusion: sync bytes are a rounding error; sync *correctness* (token
  validity, op idempotency, ordering) is the entire game — hence dive 1.

## Server-side load implied by the client contract (context, estimate)

- DAU 10M (Google-Calendar-scale product, estimate). Pulls: 10M × 18/day ≈ 180M/day ≈
  **2,100 QPS average**; with a 4× peak-hour factor ≈ **8,400 QPS peak** (workday mornings;
  calendar traffic is heavily bimodal — the tail is ~zero overnight, so peak factor is above
  the 24h-smoothed average by more than a flat-traffic system would need).
- Pushes: 10M × 6/day ≈ 60M/day ≈ **700 QPS average**, same peak factor.
- Oplog retention server-side: 60M mutations/day × 500 B ≈ 30 GB/day; 30-day retention ≈
  **~1 TB** — cheap; the oplog is what makes delta pulls correct (03).

## Reminder/notification burst arithmetic (shapes dive 5)

- ~5 reminders/user/day (estimate) × 10M users = 50M device-pushes/day ≈ 580/s average —
  but meetings start on wall-clock boundaries (:00, :30), so 30–50% of a day's reminders
  land inside ~60-second bursts. A 40% burst: 20M in 60 s ≈ **333K pushes/s during bursts**,
  sustained for seconds-to-minutes around each boundary — a spike of ~576× the average
(20M ÷ 60 s = 333K; 333K ÷ 579/s ≈ 576).
  The client-visible consequence: reminder payloads must be small (id + summary + fire
  time), delivery is best-effort within the minute, and the client must tolerate a reminder
  arriving at or just after its fire time (render "now", not "3 min late").

## Memory arithmetic (shapes dive 6)

- 1,500 mounted occurrence nodes × ~1–2 KB DOM cost (engine-dependent estimate) ≈
  **2–3 MB** — fine. The cost is not memory; it is **layout**: naive per-render slot
  packing over 1,500 occurrences per frame blows the 16 ms budget; layouts must be cached
  per day-key and invalidated per-day (04 dive 6).
- Drag-to-reschedule: every drag frame mutates one occurrence's time → the invalidation
  unit is *one day column* (or two, if the drag crosses midnight), never the grid.

## Consistency check

- Device store ~5 MB, caches ~3 MB, sync ≈ 8 KB/day of unique data (the day's ~10 changed
  events × 500 B ≈ 5 KB, each change delivered once under the monotonic token, + ~6
  pushes/day × 500 B ≈ 3 KB): the calendar client is nowhere near
  any capacity limit at personal scale. Every hard problem in 04 is **concurrency,
  semantics (recurrence, timezones), and latency** — the frontend-scale lesson of
  [Case 19](../19.frontend-chat-app/02-scale-estimates.md) repeats: frontend scale
  problems are state-and-jank problems, not terabyte problems. The one genuinely large
  number is the notification burst, and it is a server-contract shape the client designs
  its display path around.

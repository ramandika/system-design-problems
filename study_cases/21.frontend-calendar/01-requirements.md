# 01 — Requirements

## Functional

- **F1 — Calendar views**: day / week / month grid; list (agenda) view. Navigate across
  any date range, including year view for a single calendar.
- **F2 — CRUD events**: create, read, update, delete events with title, time, location,
  description, attendees, calendar membership, color/label.
- **F3 — Recurring events**: define a series with a recurrence rule (daily/weekly/monthly/
  yearly, with interval and count/until); edit one occurrence ("this one only"), this-and-
  following, or the whole series; delete occurrences; per-occurrence exceptions survive
  offline edits.
- **F4 — Offline-first**: full read and write offline; edits queue locally and sync later;
  the app starts and renders from the local store with no network round-trip.
- **F5 — Multi-device sync**: same account on phone/laptop/web converges; a change made on
  device A appears on device B within seconds when both are online, or on next sync otherwise.
- **F6 — Concurrent edits converge**: two devices editing the same event offline produce a
  deterministic merged result on both (see 04 dive 2); the user never sees a merge dialog
  for ordinary field edits.
- **F7 — Multi-timezone**: display events correctly wherever the device is; per-event source
  timezone is preserved (a 9am Tokyo meeting stays 9am Tokyo when viewed from Berlin);
  optional fixed secondary-timezone column.
- **F8 — Reminders/notifications**: per-event reminders (e.g., 10 min before); delivered on
  every signed-in device; dismissal is acknowledged and not re-delivered.
- **F9 — Shared calendars**: subscribed calendars (team, room, holidays) render alongside
  personal events; shared events are read-mostly for the subscriber.
- **F10 — Search**: local instant search over stored events; server-side search for ranges
  not yet on device (out of scope for deep dives here).

## Non-functional

- **N1 — Cold start to first usable frame**: < 1 s on mid-range hardware from local store
  alone (no network dependency) — estimate, design target not a measurement.
- **N2 — Offline durability**: queued mutations survive app kill, OS reboot, days offline.
- **N3 — Convergence latency**: two online devices converge on a shared change in < ~5 s
  (push channel) or on next sync (offline).
- **N4 — Timezone correctness (functional, but hard)**: no stored representation may lose
  the source timezone; DST-ambiguous times must resolve deterministically and identically
  on all devices (04 dive 4).
- **N5 — Render budget**: steady 60 fps scroll/pan in dense views; interaction (click-drag
  to create, drag to move) responds within one frame (~16 ms budget); month grid with
  ~1,000 visible occurrences stays smooth (04 dive 6).
- **N6 — Battery/network politeness**: sync is batched and delta-based; no polling that
  keeps radios awake when a push channel would do.
- **N7 — Storage footprint**: the local store for a typical user is megabytes, not
  gigabytes (02 shows the arithmetic); eviction only for caches, never for pending mutations.

## Scope decisions

**In scope** (the client-side system): local-first store and its schema; sync engine
(push/pull, tokens, ordering); conflict resolution; recurring-event model and expansion;
timezone modeling and display; reminder scheduling semantics (who decides when a reminder
fires); rendering and layout performance for dense views.

**Explicitly OUT of scope**:

- **Server event store design** — the counterpart server ( shards, storage) is a separate
  distributed-systems case; we define only the client-visible contract (delta pull by token,
  mutation push by client op ID, push channel for change notifications).
- **Invitation/meeting-negotiation flows** (RSVP propagation, organizer scheduling loops,
  free/busy lookup) — its own protocol problem.
- **E2E encryption of calendar data** — changes the server contract
  fundamentally; the WhatsApp-style trade-offs are covered in
  [Case 03](../03.whatsapp/04-deep-dives.md).
- **Third-party calendar federation** (CalDAV adapters, Exchange quirks) — the client
  designs against one canonical model.
- **Collaborative real-time co-editing of one event by many online users** (Google-Docs-
  style live cursors) — calendar edits are discrete, not continuous text; CRDT treatment
  is reduced to the structured-record case in 04 dive 2.

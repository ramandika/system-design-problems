# 04 — Deep dives

## 1. The offline-first sync engine: local-first store + outbox/push-pull (THE dive)

**Option A — full CRDT state replication** (Yjs/Automerge-style: every replica merges
all replicas' states; no server authority):
- wins: any-to-any sync without an ordering authority; mathematically clean convergence;
  peer-to-peer sync becomes possible.
- loses: events are small structured records with a *natural* authority (the server) and
  few fields; CRDT machinery buys per-character granularity we don't need at the cost of
  tombstone growth, per-field metadata overhead, harder queries (you query your CRDT
  snapshot, not SQL), and a much harder story for recurrence expansion (an "occurrence"
  is derived, not a CRDT-merged entity). Calendar edits arrive at ~6/user/day (02) —
  this is not a throughput problem CRDTs solve.

**Option B — classic cache + fetch-on-demand** (store is a cache; open app → fetch range →
render):
- wins: trivially simple client.
- loses: violates F4 outright — no offline write, no instant cold start, every navigation
  is a network call. The subway case kills it.

**Chosen: a third option — local-first store + durable outbox + token pull, server as
sequencer.** It borrows CRDT's goal (convergence without coordination) while keeping B's
relational store; each device keeps full data locally and its own oplog of pending
mutations (the outbox); the server is the single sequencer of truth:

- **Push**: drain the outbox oldest-first; every op carries `client_op_id` (uuid) for
  idempotent application — a retry after a dropped response replays safely. Same shape as
  the at-least-once wire + client dedup contract in
  [Case 03 dive 1](../03.whatsapp/04-deep-dives.md): transport retries blindly; correctness
  is proven at the application layer by ID, not by delivery guarantees.
- **Pull**: `pull(sync_token)` returns all changes since the token + a new token. The token
  is a server-side sequence/cursor — one cursor for all calendars (per-calendar tokens add
  failure modes without saving anything at this scale). Token is persisted in the same
  transaction as the applied deltas (invariant I3).
- **Order on reconnect**: flush outbox completely, then pull. Two reasons: (a) the user's
  own edits are visible immediately (read-your-writes without a merge step); (b) every
  conflict lands at exactly one reconciliation point — after the push acks, so the server's
  response to each op *is* the reconciliation result.
- **Why not server push payloads**: the push channel carries "changes exist → pull", never
  data. A missed/dropped hint costs one delayed pull; a payload channel would need
  guaranteed ordered delivery, retransmit, and dedup — reinventing the pull path badly.

- **Trade-off made explicit**: we accept a reconciliation step with real conflict logic
  (dive 2) in exchange for (a) a relational store with indexed interval queries, (b) simple
  idempotent ops instead of CRDT metadata on every row, and (c) one sequencer, which makes
  "what is the current version of event E" always decidable. Full CRDTs win only if the
  peer-to-peer or offline-multi-writer-no-server case is a requirement — it isn't here.

**Failure modes**:
- *Crash mid-flush*: unacked ops replay on next flush (idempotency by op ID); a partially
  applied multi-op batch is fine because ops are individually atomic.
- *Token gap*: server prunes its change log behind a device's token → pull returns
  "token too old" → client falls back to a full resync of that calendar (rare, ~MBs, 02);
  never fabricates a delta.
- *Clock skew between devices*: no local timestamps are trusted for ordering — ordering
  is the server's sequence numbers and the client op order within the outbox; local clocks
  are for display only.
- *Outbox growth*: offline for weeks with hundreds of ops — the outbox is not a message
  history; bulk edits (e.g., "delete this series" done offline) must be stored as one op
  on the *master/series*, not expanded to N occurrence deletes in the outbox.

## 2. Concurrent-edit conflict resolution: field-level last-writer-wins

The core case: device A renames "Q3 planning" to "Q3 planning (moved)" while offline;
device B moves the same event 2 hours later and adds an attendee. Both flush after the
flight. What's the merged state?

**Option A — whole-event last-writer-wins** (one timestamp per event; latest write replaces
the row):
- wins: trivial logic.
- loses: destroys data visibly and pointlessly — A's rename *and* B's reschedule both
  survive only in the later writer; a user who sees one edit silently vanish loses trust
  in the product. For an event (a handful of fields, one writer per edit session), the
  merge is nearly free to do better.

**Option B — 3-way merge per field with an interactive resolver**:
- wins: maximal preservation.
- loses: needs a UI nobody wants ("A changed title, B changed time — pick") for a conflict
  class that field-level resolution handles silently and correctly; interactive resolution
  is the right medicine for text documents, overkill for 8-field rows.

**Chosen: field-level LWW with B's merge granularity and a silent policy.** The server
stamps each accepted op with `(server_seq)`;
the client sends per-field deltas (only fields actually touched in the editor), so an op
says *what changed*, not the whole row. Reconciliation per field: the field's latest
touched value by server sequence wins; untouched fields keep the other device's value.
A–B above merges to: renamed title + moved time + added attendee. All three edits survive.

The two genuinely ambiguous cases get explicit policies:

- **Delete vs edit**: delete wins with a restore path. A delete-and-edit race almost
  always means one device didn't know the other deleted; resurrecting silently (edit
  wins) produces zombie meetings that attendees can't explain. The deleting writer's tombstone
  wins; the edited content is preserved in the tombstone for an "event was deleted" banner
  (one-tap restore, 30-day default, matching the industry-typical ~30-day deleted-item
  retention of publicly documented calendar trash windows —
  labeled as a design convention, not a statistic).
- **Move vs resize-duration** (both touch time fields): treat start/end as one composite
  time field resolved atomically — partial application (new start, old end) can silently
  turn a 30-min meeting into a 3-h meeting, which is worse than picking one writer's time
  pair.

- **Trade-off made explicit**: field-level LWW wins silent, near-always-correct convergence
  for ordinary edits; loses true convergence under *simultaneous* edits to the *same field*
  of an *event two people share* (the later server-seq edit wins, the earlier is unrecoverable
  from the UI). We pick it because the alternative — per-field CRDT registers — provides
  exactly that last case's improvement at the cost of CRDT machinery on every row
  (dive 1's argument), and the lost case is rare (two humans editing the same event's
  title within the offline window) and self-healing socially (they can see and re-edit).
  Where teams actually need live co-editing semantics, that's a different product surface.

**Failure modes**:
- *Op ordering ambiguity within one device*: the client sends ops in outbox order and the
  server applies them in received order per device with per-field sequence ties broken by
  op index — otherwise two offline edits on the same field by the *same* device can
  interleave with pulls nondeterministically.
- *Attendee collections*: attendees are a set, edited by add/remove ops (not replace),
  so set-union merges naturally; a replace-style op would make the A/B attendee case
  above lose data.
- *Tombstone retention*: deleted events keep content only until retention expiry; a
  device offline past expiry reconciles to a bare tombstone ("event was deleted") —
  restore is then impossible; acceptable, and it bounds tombstone growth.

## 3. Recurring events: never materialize infinitely

A series with no end date ("Dinner with parents, every Friday") cannot be stored as rows.
The entire design question: when do you turn a rule into occurrences?

**Option A — materialize a horizon**: expand each series N occurrences (or M months) into
real rows, eagerly, on write:
- wins: reads are plain interval queries over uniform rows; search ("find all events with
  title X") sees occurrences directly; simplest possible read path.
- loses: write amplification and update storms — editing the master ("standup moved
  30 min") must rewrite every materialized occurrence; horizons expire and must roll
  forward (a background job on every device?); a series created with `count = 999999`
  is unexpandable; storage is paid for occurrences nobody views. Twitter's fanout-on-write
  lesson applies verbatim: eager materialization shifts cost to write and to storage for
  reads that mostly never happen — see [Case 02 dive 1](../02.twitter/04-deep-dives.md),
  which pays that cost only because *reads dominate* there; a calendar's reads are
  windowed, so it doesn't.

**Option B — expand on read** (store the rule; derive occurrences per requested window):
- wins: one row per series regardless of length; master edits are one-row edits; no
  horizon maintenance.
- loses: read-side derivation (RRULE evaluation per window — cheap, microseconds per rule
  for a few months of expansion); *queries over occurrences* ("next occurrence of series
  X", "free/busy across calendars") must expand-to-window under the hood instead of
  scanning uniform rows; indexing "all events overlapping range" needs the union of
  single events and per-master window expansion.

**Chosen: B, with a bounded expansion cache.** Storage model:

```
events:          id, master_id? (null for singles), rrule?, dtstart_tz_utc,
                 source_tz, wall_fields, ...          # one row per series
exceptions:      master_id, recurrence_id (occurrence start), patched_fields,
                 cancelled (bool)                      # one row per overridden occurrence
```

- **Expansion**: for window [a, b], iterate masters overlapping the window, evaluate the
  rule, subtract exceptions, yield occurrences. Cache per (master_id, window_bucket);
  windows are bucketed by month so a cache entry is reusable across adjacent views.
  Invalidate exactly (master_id) on master edit — the change hub's keying (invariant I4)
  makes this one line of invalidation, precise, not a cache wipe.
- **Editing one occurrence**: writes an exception row (exception-first design: the
  series stays one row; "this Friday moved" becomes `exceptions(master, friday_start,
  {start: new_time})` or `cancelled=true`). Editing "this and following": split the
  series — terminate the old rule with `until = <that occurrence>` and create a new
  series continuing from it. That's the standard RRULE trick; it keeps every stored
  entity finite and every edit a single-row transaction.
- **The rule is stored verbatim (RFC 5545 RRULE), not as expanded semantics** — display
  and edit both round-trip the original text; re-deriving semantics from a rule we
  parsed into our own shape is where subtle drift (BYDAY vs BYSETPOS edge cases, monthly
  31st behavior) creeps in.

- **Trade-off made explicit**: expand-on-read wins bounded storage and O(1) master edits;
  loses uniform read rows — every occurrence-consuming feature (search, free/busy,
  "conflicting meeting?" logic) must expand a window first. We pick it because windows
  are small (a view never asks for more than a few months, 02) while series length is
  unbounded — asymmetric in exactly the direction that favors laziness. Free/busy
  computation expands a month per master: fine per request, and it is precisely the
  operation Option A would have made free; if that becomes a hot path server-side,
  *that's* where a materialized horizon belongs (as a server cache, not device storage).

**Failure modes**:
- *Bad rules from imports* (iCal files with pathological RRULEs, e.g., monthly on the 31st
  with `count=∞` effectively yielding ~1/year): expansion is cached and bounded per
  window; a rule that yields nothing in the window must still terminate its expansion
  attempt — cap iterations per master expansion (e.g., skip-ahead window arithmetic,
  not brute-force iteration) or a hostile import becomes a device DoS.
- *Exception keyed by wall-clock start that DST shifts*: the `recurrence_id` for an
  exception must be the occurrence's original wall-clock time in the series' source tz,
  not a UTC instant — otherwise a tzdata change re-maps every exception to the wrong
  occurrence (connects directly to dive 4).
- *Offline master edit + offline exception both sync later*: exception rows carry their
  own LWW fields (dive 2); a cancelled-then-restored race resolves by server seq, and the
  restoration path in the tombstone banner covers the ugliest UX case.

## 4. Timezones: there is no such thing as "a local datetime stored"

The meeting is "9am, Tokyo time, March 15". Users in Berlin and New York see different
clock times; users see the same clock time if they travel *to* Tokyo. Three traps:

1. **Naive storage** (`start: "2026-03-15 09:00"`): ambiguous. You need a tz ID or it's
   meaningless — but see trap 3 before storing only the derived UTC value.
2. **The traveler**: the Tokyo meeting must display 09:00 when the user is *in* Tokyo and
   the correct Berlin equivalent when not. Display tz is the *viewer's* property;
   event anchoring is the *event's* property. Conflating them is the most common calendar
   bug in the wild.
3. **DST + tzdata drift** (the deep one): "9am on the yearly anniversary in Tokyo" or
   "9am in a country that abolishes DST next year". If you compute UTC once at creation
   and freeze it, the event drifts one hour off its wall-clock meaning after the rule
   change. IANA tzdata updates *do* change past instants when governments retroactively
   change rules (public, documented tzdb behavior) — a frozen UTC row silently violates
   what the user said.

**Chosen representation** (per event):

```
start_utc, end_utc          # computed, for range queries only (indexed)
source_tz_id                # IANA ID ("Asia/Tokyo")
wall_fields                 # {y, m, d, hh, mm} in source tz  <-- the truth
```

- **Wall fields are the authority; start_utc is a derived, indexed projection** —
  recomputed on read-path whenever (wall_fields × source_tz_id) under the *current*
  tzdata maps differently than the stored UTC. This is the "derive at read vs derive at
  write" trade-off:
  - derive-at-write wins simpler reads; loses wall-clock stability under tzdata updates.
  - derive-at-read wins correctness across rule changes; loses: a recompute pass and a
    migration story when tzdata ships (devices get tzdata with OS updates; on update,
    the client recomputes affected projections and re-pulls nothing).
  We pay the read-side recompute because "the 9am meeting stayed 9am" is a
  trust-preserving invariant; an hour-silent drift is exactly the class of bug users
  cannot forgive a calendar for.
- **DST ambiguity is resolved deterministically**: 01:30 on a fall-back date in the US
  occurs twice → pick the second (post-transition) occurrence by convention on *write*,
  or flag if the creator is editing in source tz; spring-forward-nonexistent times
  (02:30 on the missing hour) shift to the post-transition equivalent. The convention is
  fixed client-side so all devices agree without communicating (invariant N4).
- **All-day events are dates, not times**: store `{y, m, d}` with no tz (RFC 5545 "floating"),
  and render them as full-day blocks in the *viewer's* tz — they must not shift when the
  user flies, which is exactly what floating semantics give you.
- **Query layer**: interval queries (overlaps [a,b]) run on the indexed UTC projection;
  the correctness layer is the recompute-on-tzdata-update pass. The two-layer design —
  derived projection for performance, wall truth for correctness — is why "no local
  datetime storage" is about *authority*, not about never having UTC fields.

- **Trade-off made explicit**: storing wall fields + tz + derived UTC triples the time
  columns versus a single UTC instant; we pay storage (hundreds of bytes/user-day,
  02) and one recompute pass (per tzdata update) for wall-clock stability and
  retroactive-rule correctness. The rejected alternative — UTC-only — is simpler every
  day until a government changes a rule (or a user reinterprets "the meeting is at 9am
  *there*" after a tzdata update), at which point it requires a data migration that can't
  recover information it never stored: you cannot derive back the source timezone from
  a UTC instant.

**Failure modes**:
- *Tzdata version skew across devices*: device A on old tzdata shows a drifted time until
  its OS update lands. Mitigation: ship tzdata with the app package (apps that need
  correctness bundle their own copy of the IANA db rather than deferring to the OS) and
  version the projection recompute by tzdata release, not by app start.
- *Source tz vs user tz in recurrence exceptions*: covered in dive 3 — exception keys are
  wall-clock in source tz; the same tzdata-update recompute applies to `recurrence_id`
  matching or exceptions detach silently.
- *Copy-paste/DST bugs on drag*: dragging an event across a DST boundary in the grid —
  the drag encodes wall-clock minutes ("drag it 2 hours down") in the *viewer's* tz, which
  is what the user's hand means; writing that as wall_fields in *source* tz requires the
  boundary conversion at drag-commit, not at display.

## 5. Reminder/notification scheduling: the server decides, the client displays

**Option A — client schedules everything locally** (each device computes fire times for
its known events and asks the OS for local notifications):
- wins: works fully offline (the OS fires local notifications with no connectivity);
  zero server infrastructure for the trigger.
- loses: (a) *multi-device duplication* — phone and laptop both fire; dismissal on one
  can't cancel the other's already-armed OS alarm; (b) *stale schedules* — a device that
  was offline when the event moved still fires the old time; (c) *app-killed/OS-evicted*
  state on aggressive mobile OSes loses scheduled alarms silently; (d) reminder policy
  changes (accounts, per-device settings) fragment into per-device truth.

**Option B — server decides, server pushes**:
- wins: single authority on fire time; device-precise targeting and cross-device dedup by
  reminder ID; pushes reach the device even when the app process is dead (FCM/APNs wake
  the display path); changed events recompute centrally.
- loses: needs connectivity at fire time; pushes can be delayed by a few minutes or
  dropped by the platform — a reminder that arrives late is inherently "close enough" or
  not fired at all.

**Chosen: B as the authority, with a narrow client-side fallback.** The server pipeline
computes fire times from the event store (its copy of the wall-truth, dive 4), schedules
per-device pushes, and clients:

- **Display on push receipt**, deduping by `(event_id, reminder_id)` — the same reminder
  fired to a phone and a laptop is one notification in the user's mind; dismissal acks
  propagate back so a device that comes online late doesn't re-show an acknowledged
  reminder (pull path carries acknowledged-reminder IDs).
- **Fallback local scheduling, horizon ≤ 24 h, only while fully online**: while the client
  has fresh data it arms OS-local notifications for the next day's reminders as a
  hedge against brief connectivity loss at fire time. These are *hedged*, not authority:
  on reconnect the client reconciles (event moved while away → local alarm cancelled,
  server fire wins). The 24 h bound keeps the hedge cheap (a bounded set of alarms, never
  the whole calendar as in Option A) and bounds the staleness window.
- **Late/missed delivery tolerance**: the burst shape (02: ~333K pushes/s at wall-clock
  boundaries) means a reminder commonly arrives at-or-after its fire minute. The design
  answer is semantic: display "now" with the *origin* visible in content ("Starts now:
  1:1 with Bob"), never "late by 3 min" arithmetic that invites users to distrust delivery.
  A reminder arriving > 1 h late is dropped if the event has started (a "meeting began
  an hour ago" bleep is noise), kept if it's still upcoming.

- **Trade-off made explicit**: server-decided wins correctness (move an event, every
  device's reminder moves) and dedup, loses guaranteed offline trigger — we buy a hedge
  (bounded local alarms) rather than the full Option A, because the failure Option A
  *always* has (cross-device duplication + staleness) is every-day visible, while the
  failure B *sometimes* has (push delayed a minute) is rare and mostly invisible to a
  user glancing at their wrist. This is a case-by-case call, not a doctrine: for a
  single-device or offline-first-only product, Option A would win.

**Failure modes**:
- *Push silently dropped* (platform-level): no client retry is possible for a reminder
  the platform lost; the local hedge is the mitigation; metrics track per-device delivery
  as "reminder shown within ±5 min of fire time" (design metric, not a fabricated rate).
- *Device clock mismatch*: display path uses the fire-time embedded in the payload
  relative to event start, not device-local comparison for "late" logic.
- *Reminder storm on a shared calendar*: a company calendar edit touching 500 attendees'
  devices must not fan out 500 pushes per edit — reminder computation happens against
  *confirmed* attendee state (RSVP flows are out of scope, 01) and the pipeline coalesces
  reminder updates per (event, user) before arming pushes, same bounded-fanout reasoning
  as [Case 03 dive 4](../03.whatsapp/04-deep-dives.md).

## 6. Rendering 1,000+ occurrences without jank: per-day layout caching + lazy windows

A month grid over 10 calendars shows 1,000–2,000 occurrences (02). The naive pipeline —
expand range → build list → render all → layout per frame — dies on layout, not memory:
slot packing (stacking overlapping events into columns per day) over 1,500 occurrences
per frame is far past the 16 ms budget.

- **Layout is the expensive step, so cache it per day**: a day column's layout (which
  occurrences, what slots, what overlap clusters) depends only on that day's sorted
  occurrences + the day's constants (column width, tz projection). Cache keyed by
  `(day_key, occurrence_set_version)`; a change to any event touching that day bumps
  the day's version. Rendering a month = composing ~30 cached day layouts; an edit
  re-lays-out **one or two day columns** (a drag across midnight invalidates both), never
  the grid. This is the same "invalidate the smallest correct unit" discipline as
  Case 19's height-cache-per-message — coarse invalidation is the death of smooth UI
  ([Case 19 dive 2](../19.frontend-chat-app/04-deep-dives.md) covers the list-view twin;
  the grid view has the advantage that day heights are fixed, so there's no variable-height
  estimation problem — the layout cache trades that for overlap-packing).
- **Overlap packing is done once per cluster**: within a day, group overlapping
  occurrences into clusters, pack columns inside the cluster. A day with 40 overlapping
  meetings doesn't force re-packing of the 200 lone events around it — clusters are
  the invalidation unit within a day.
- **Lazy windows, virtualized grid**: only the visible month (+1 month of prefetched
  expansion cache, 02) is ever expanded/attached; month navigation mounts the next
  window from cache or triggers its expansion off the interaction path. Year view degrades
  by design to dots/counts (aggregates), never 12 months of full grids — aggregate counts
  come from the expansion cache precomputed per month bucket.
- **Interaction path discipline**: drag-to-create and drag-to-reschedule are projection
  updates on the view model (a ghost occurrence), committed to the store on release, which
  triggers the *normal* change-hub path (invalidate affected day keys, one-column
  relayout). Interaction latency stays one frame because no work is done in the drag loop
  beyond moving the ghost's absolute position.
- **Batch notification application**: a delta pull that touches 50 events across 20 days
  applies to the store in one transaction; the change hub coalesces the day-key
  invalidations so the UI relayouts each affected day exactly once, after the batch —
  not per-event.

- **Trade-off made explicit**: per-day layout caching wins O(1)-per-edit relayout and
  constant-time revisits of seen days; loses some memory (cached layouts, < 1 MB, 02)
  and adds an invalidation-correctness obligation (every mutation must bump the exact
  set of day keys its occurrence spans — get that wrong and a moved event renders in the
  old slot until a manual refresh). We pick it because the correctness obligation is
  centralized in the change hub (one choke point, testable) while the alternative —
  recompute-on-render — caps density at a few hundred events before dropping frames,
  and density only grows (more subscribed calendars is the product's own growth vector).

**Failure modes**:
- *Invalidation misses* (edit bumps wrong day keys): symptom is a stale slot; guard is a
  debug-mode assertion that recomputes layouts for a rendered day from scratch and diffs
  against the cache in development builds (a self-check on the non-trivial logic, cheap
  because it's dev-only).
- *Expansion-cache thrash on wide navigation*: jumping month-to-month in a loop evicts
  and re-expands; bucketed windows (02) make adjacent-month hits the common case; a true
  pathological navigator costs re-expansion of ~2 MB cache — an LRU miss, never a hang,
  because expansion is capped per master (dive 3).
- *Tzdata update mid-session*: the projections recompute → day-key versions bump for
  every visibly-affected day → one coalesced relayout; the failure mode to avoid is
  *partial* recompute (some days projected with old tzdata), so the recompute pass runs
  over the whole loaded window in one pass before notifying the hub.

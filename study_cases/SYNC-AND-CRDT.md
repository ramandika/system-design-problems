# Client sync & reconciliation across the case catalog

Where offline-first client sync and concurrency reconciliation (CRDTs, OT, outbox + gap-sync,
last-write-wins vs merge, versioned ops, tombstones) appear in the 50 study cases, what role
each plays, and how deep each case goes.
Companion to [CONCEPTS.md](CONCEPTS.md): the "Cursor & sequence sync — client side" union
(6/7 cases, 86% of clients) and the related-concepts row "Collaborative sync: server-sequenced
ops vs CRDTs" in [README.md](README.md). Those rows track the cursor/sequence mechanism; this
doc covers the broader sync-and-merge surface. Counts here are mention-based, counted
independently of CONCEPTS.md.

Headline: **every case that needs client sync rejects the CRDT and keeps a server sequencer.**
No case in the catalog ships a peer-merge CRDT; the two cases that evaluate one explicitly
(46, 21) both reject it for the same reason — an authoritative server already exists for
permissions or versioning, so the CRDT's central win (convergence without coordination) is
dead weight. What the catalog actually designs, over and over, is one shape: *client keeps a
local system-of-record replica, mutations go out through a durable outbox keyed by
client-generated IDs, and reconnect reduces to "last sequence I saw, catch me up" plus a
narrow, bounded merge policy*.

- Cases that use client-sync / reconciliation technology meaningfully: **13 / 50**
- Full case (collaborative editing, OT vs CRDT): **1** (46; no separate sync case exists —
  21, 24 are its client-side siblings)
- Offline-first client sync engine as a designed core component: **5** (21, 24, 32, 19, 25)
- Offline-autonomous devices reconciled by the backend: **2** (27, 22)
- Server-side merge semantics (no offline client): **1** (39)
- Delivery buffering / ephemeral-or-recompute state, explicitly no merge semantics: **3**
  (03, 30, 05)
- Adjacent: offline-verifiable state without sync: **1** (06)
- CRDT evaluated and rejected explicitly: **2** (46, 21); "CRDT-flavored" merge instead of a
  full CRDT: **1** (32)

## Summary table

| Case | Role in architecture | Depth |
|---|---|---|
| [46.google-docs](46.google-docs/README.md) | OT vs CRDT decision, offline reconnect transform, TTL'd presence hub | Full case |
| [21.frontend-calendar](21.frontend-calendar/README.md) | Outbox/pull-token sync engine + field-level LWW + delete tombstones; CRDT rejected | Deep |
| [24.mobile-chat-app](24.mobile-chat-app/README.md) | Local-first DB, seq-cursor reconciliation, durable send queue, idempotent uploads | Deep |
| [32.spotify](32.spotify/README.md) | Weeks-stale client, op-log playlist merge, offline downloads + rights/reconciliation | Deep |
| [19.frontend-chat-app](19.frontend-chat-app/README.md) | Optimistic index + durable outbox + seq gap reconciliation | Moderate |
| [25.mobile-newsfeed](25.mobile-newsfeed/README.md) | Last-good snapshot + interaction outbox, LWW on monotone actions | Moderate |
| [27.ev-charging-network](27.ev-charging-network/README.md) | Offline-autonomous chargers, idempotent transaction lifecycle, backend as reconciler | Moderate |
| [22.frontend-uber](22.frontend-uber/README.md) | Driver intents queued offline, degraded/terminal events resolved on reconnect | Moderate |
| [39.e-commerce](39.e-commerce/README.md) | Guest→account cart merge: union by SKU, quantity add, tombstone guest cart | Moderate |
| [05.discord](05.discord/README.md) | Unread counters eventually consistent, reconciled on foreground; ephemeral presence | Brief |
| [03.whatsapp](03.whatsapp/README.md) | Server-side offline delivery queue (store-and-forward), no merge; the dedup contract others borrow | Brief |
| [30.facebook-messenger](30.facebook-messenger/README.md) | Parked offline replay in `(thread, seq)` order, reusing 03's machinery | Brief |
| [06.apple-app-store](06.apple-app-store/README.md) | Offline-validatable signed receipts — money without sync | Brief |

## Per-case notes

### The case itself

**[46.google-docs](46.google-docs/README.md)** — the catalog's canonical collaborative-editing
case; everything else here is a lighter variant of one of its dives.
[04-deep-dives.md](46.google-docs/04-deep-dives.md) §1 is the OT-vs-CRDT decision: Option A
(CRDT, per-character IDs) wins server-optional P2P sync and loses on metadata — a 100K-char
doc carries 100K+ tombstones and IDs, a ~10× doc-model cost for a product whose median doc has
no conflict at all, and cross-replica tombstone GC is called "the genuinely hard open problem."
Option B (server-sequenced OT) wins: ops are small, one sequencer gives total order + ACL +
history, and OT bugs are contained (few op types, property-tested against TP2).
§2 is the op loop: client applies locally, server gates → sequences → stores durable →
broadcasts; subscribers transform incoming ops against their own un-acked ops; undo is a
transformed inverse, not a reverse. Fanout arithmetic (02): 20 typing collaborators = ~40
ops/s on a 200-participant doc ⇒ ~8K msg/s outbound; a 100-busy-doc shard at k^2 fanout is
the stress point.
§4 is the offline story: offline is *the same op loop with the server disconnected* — local
mirror + ordered op queue, and on reconnect the client's queued ops are transformed two-way
against everything the server did meanwhile. Last-write-wins-on-reconnect (Option A) is
rejected in one line: it silently drops the *other person's* text in exactly the cases
offline exists for.
§5 separates three channels by guarantee: ops (durable, ordered), comments (durable, own
store, anchors transform-managed), presence (ephemeral, TTL'd ~3–10 s, ~1–2 Hz per
participant ⇒ a 200-viewer doc pushes ~200–400 msg/s outbound — the biggest zero-durability
traffic in the system). Presence-hub outage ⇒ cursors vanish, editing unaffected: deliberate
degradation.

### Core component: offline-first client sync engines

These cases make the client's local store a *system of record for reads*, then reconcile it
against the server with an outbox + cursor protocol. The recurring contract: client-generated
operation IDs make retries free; the server is the sole sequencer; reconnect = flush outbox,
then pull from a cursor; the merge policy is a narrow, named decision.

- **[21.frontend-calendar](21.frontend-calendar/README.md)** — the richest merge-policy case.
  [04-deep-dives.md](21.frontend-calendar/04-deep-dives.md) §1 sizes all three options:
  full CRDT (Yjs/Automerge) rejected — per-character machinery buys granularity a 5,000-event
  × 400 B (~5 MB/device) store never needs, and recurrence expansion is a derived entity no
  CRDT merge produces; plain fetch-on-demand rejected (no offline writes); chosen is
  local-first store + durable outbox + server sequence token, so every conflict resolves at
  exactly one reconciliation point (after the push acks, the server's per-op response *is*
  the reconciliation result). Sync traffic arithmetic (02): ~18 pulls + ~6 pushes/day/user at
  ~500 B/op; a full resync is 5,000 × 500 B ≈ 2.5 MB — rare and bounded.
  §2 is the merge policy: field-level last-writer-wins by server sequence (per-field deltas,
  untouched fields keep the other device's value — all three of A's rename + B's reschedule +
  B's attendee survive), with two named policies: *delete wins over edit, with a restore
  path* (tombstone keeps the content for one-tap restore, ~30-day retention, then the
  tombstone is bare) and *start/end resolved as one composite field* (a partial move must not
  turn a 30-min meeting into a 3-h one). Explicitly conceded: simultaneous edits to the same
  field of a shared event lose the earlier write, and full CRDT registers would have bought
  that back — rejected at per-row metadata cost for a rare, socially self-healing case.
- **[24.mobile-chat-app](24.mobile-chat-app/README.md)** — the offline-first *mobile OS*
  case. The client DB (~100 MB budgeted, ~110 MB estimated at 02) is the system of record;
  launch renders from local rows and the sync engine reconciles *after* first paint.
  [03-architecture.md](24.mobile-chat-app/03-architecture.md): durable send queue writes
  `state=queued` before the UI says "*queued*" (not "sent"); reconnect = compare
  per-conversation cursors with the server's sequence high-water marks, fetch only gaps, dedup
  client-side. [04-deep-dives.md](24.mobile-chat-app/04-deep-dives.md) §: foreground sync
  reconciles *always*, so missed pushes cost only latency, not correctness; token gap beyond
  cursor TTL falls back to bounded catch-up, never fabricated deltas; uploads are resumable
  and keyed by content hash, so a 12-network-switch video upload replays safely.
- **[32.spotify](32.spotify/README.md)** — the weeks-stale client. The client is "a
  distributed system node that can be weeks stale" (03). [04-deep-dives.md]
  (32.spotify/04-deep-dives.md) §4: playlist edits are recorded as operations on a
  server-authoritative identity and *merged by op order* — inserts reconcile, deletions win on
  identity, ordering resolves on the log's total order; no merge-conflict dialog is ever
  shown, the "users who co-edit playlists" argument for CRDT-ish merging without a full CRDT.
  §2 is the offline-drift side: downloads carry a licence manifest (region, tier,
  offline-until) revalidated even offline; two devices offline-editing the same playlist for
  days both apply their ops and converge. Offline sync volume (02): 30M premium users ×
  ~50 tracks/month × 8.4 MB ≈ 12.6 PB/month ≈ **0.4 PB/day**, ~2% of streaming egress, bursty
  by trip — a client-side scheduling problem, not a server one.
- **[19.frontend-chat-app](19.frontend-chat-app/README.md)** — the web sibling of 24's
  client. [03-architecture.md](19.frontend-chat-app/03-architecture.md): server log is the
  source of truth; the client writes the outbox keyed by `client_msg_id` *before* rendering
  any optimism, renders from a separate optimistic index, and folds it into the seq-ordered
  log only on ack (crash mid-send and duplicate sends both covered by the same ID).
  [04-deep-dives.md](19.frontend-chat-app/04-deep-dives.md): reconnect = `sync(last_seq per
  conversation)` + range-fetch the gap; the rejected Option B (send-and-wait-for-ack) is named
  for completeness — zero reconciliation logic, but every send *feels* 200 ms–s.
- **[25.mobile-newsfeed](25.mobile-newsfeed/README.md)** — read-mostly offline. The local DB
  *is* the last-good feed (~400 KB for 1,000 rows, persisted continuously, not a separate
  offline copy); likes/saves append to a durable outbox flushed with idempotent operation IDs;
  server conflict resolution is deliberately trivial — LWW on a monotone action — and
  staleness is surfaced ("updated X ago", 48 h window, clock-jump-safe by storing
  `now_at_save`). The rejected Option B ("you are offline" screen) converts commute minutes to
  zero engagement — the honesty moves into the banner instead.

### Offline-autonomous devices reconciled by the backend

**[27.ev-charging-network](27.ev-charging-network/README.md)** and
**[22.frontend-uber](22.frontend-uber/README.md)** push the same pattern to devices whose
*actions*, not documents, must survive disconnection. 27's requirement is "offline auth +
queued billing is a *requirement*, not a fallback" (01): chargers authorize on a local
allowlist / EMV floor limits, run the transaction lifecycle locally, and replay buffered
meter data on reconnect. No merge at all — identity is idempotency: session key
`(charger ID, tx ID)`, ingest assigns and persists the dedup key, records that arrive after
settlement route to a *manual reconciliation path*, never silently to the PSP; the backend is
described in 01 as "a reconciler of" local device state. 22's client keeps "a small outbox
for driver-app intents sent while offline" (03); degraded states render honestly ("car
position: last seen 40 s ago") and a *degraded → terminal* transition (ride completed while
offline) is resolved by event replay on reconnect.

### Merge semantics on the server

**[39.e-commerce](39.e-commerce/README.md)** — no offline client, the same merge problem
anyway. Guest cart and account cart are two line-item lists that must reconcile at login
([04-deep-dives.md](39.e-commerce/04-deep-dives.md) §2): union by SKU with quantities
*added* and capped, promo codes re-evaluated, the merged cart written to the account and the
guest cart tombstoned; the merge is keyed by session id so a repeated session→account
mapping cannot double quantities. Same family as 21's attendee sets (add/remove ops merge as
union; a replace-style op is what loses data) and 32's op-order list merge: *set/list union
semantics instead of whole-object LWW*.

### Store-and-forward delivery, no merge

- **[03.whatsapp](03.whatsapp/README.md)** — offline here is *delivery-side buffering*, not
  state sync: messages park in per-recipient ordered queues with a ~30-day TTL and replay on
  reconnect in `conversation_id + seq` order (03, dive §3). No concurrent writers exist, so
  no merge exists — and the case's lasting export to this pattern is the wire contract
  everyone above reuses: at-least-once delivery + client-side dedup by client-supplied ID.
- **[30.facebook-messenger](30.facebook-messenger/README.md)** — the same parked-queue
  machinery, declared by reference to 03 (03 §"Receiver offline").
- **[05.discord](05.discord/README.md)** — the deliberate opposite choice on state
  reconciliation: live pushes are lossy by design and *unread counters are eventually
  consistent, reconciled on foreground* (04); presence is ephemeral heartbeat state that
  "self-heals on the next heartbeat," never persisted. Discord accepts approximate counters
  when reconnecting because exact real-time counts for offline users would cost 98%-wasted
  per-message fanout work — the same ephemeral-vs-durable line 46 draws inside one product.

### Adjacent: offline truth without sync

**[06.apple-app-store](06.apple-app-store/README.md)** — entitlements must survive airplane
mode with *no* reconciliation path: the ledger stays server-authoritative and every purchase
issues a signed receipt the device validates offline; late-landing purchase events simply
land later (02). The pattern this contributes to the boundary below: when state can be proven
instead of merged, sync disappears.

### What was filtered and why

- **Ledger/ops "reconciliation"** — [36.airbnb](36.airbnb/README.md),
  [44.payment-system](44.payment-system/README.md), [17.elevenlabs](17.elevenlabs/README.md),
  [08.chatgpt](08.chatgpt/README.md), [01.youtube](01.youtube/README.md),
  [47.rate-limiter](47.rate-limiter/README.md): these reconcile two *server-side* views
  (ledger vs PSP statements, counters, quota estimates) with a finding/drift backstop, not
  two concurrent writers. That union is catalogued in [README.md](README.md) as "Double-entry
  ledger & reconciliation as truth backstop" — this doc does not duplicate it.
- **Derived-store rebuilds and failover replay** — [34.reddit](34.reddit/README.md),
  [14.ml-production-system](14.ml-production-system/README.md),
  [50.stock-exchange](50.stock-exchange/README.md), [12.bigtable](12.bigtable/README.md),
  [09.cdn](09.cdn/README.md): a view re-derivable from a log is a consistency-backstop
  mechanism (see [KAFKA.md](KAFKA.md), [CONSENSUS-AND-QUORUM.md](CONSENSUS-AND-QUORUM.md)),
  not client-state merge.
- **"Offline" = batch/training/pipeline** — 14, 15, 26, 31, 38, 43, 45, 42, 35, 33, 18: the
  word, not the mechanism. (26.google-maps *does* cache tiles in clients, but its offline
  story is precomputed raster packs — static data, no writes to reconcile.)
- **Storage-engine tombstones** — [02.twitter](02.twitter/README.md) (deleted tweets in
  feeds), [49.monitoring-metrics](49.monitoring-metrics/README.md) (LSM),
  [16.dalle](16.dalle/README.md), [17.elevenlabs](17.elevenlabs/README.md),
  [48.google-search](48.google-search/README.md), [07.blob-store](07.blob-store/README.md),
  and the REDIS/BIGTABLE/CASSANDRA/SEARCH-INDEX sibling docs: deletion markers consumed by
  compaction, a different tombstone from the client-sync one (delete-wins with retention and
  a restore path) that 21 and 32 use.
- **Presence-heavy cases** — [03](03.whatsapp/README.md), [05](05.discord/README.md),
  [22](22.frontend-uber/README.md), [30](30.facebook-messenger/README.md),
  [42.linkedin](42.linkedin/README.md), [08.chatgpt](08.chatgpt/README.md): ephemeral
  transport state is [REALTIME-TRANSPORT.md](REALTIME-TRANSPORT.md)'s subject; this doc only
  claims the *contrast* (46's TTL'd presence hub carrying zero-durability traffic beside the
  durable op log).

## The portable pattern

Every substantive use reduces to one contract, with
[21.frontend-calendar](21.frontend-calendar/README.md) and
[46.google-docs](46.google-docs/README.md) as the reference pair:

1. **The server is a sequencer, not a peer.** All four designs with real merge semantics
   (46, 21, 32, 39) keep one authority that assigns the order of record — server revisions,
   server sequence, op log, SKU-union — and no case ships a peer-merge CRDT. CRDT is
   evaluated exactly where it might be expected (Docs, Calendar) and rejected for the same
   reason both times: an authoritative server already exists, so its main win is unused, and
   per-character/field metadata cost ~10×.
2. **Client-generated IDs make every retry free.** `client_op_id` (21), `client_msg_id` (19,
   24), content-hash uploads (24), session-keyed carts (39), `(charger, tx)` session keys
   (27): at-least-once transport + dedup by identity is the one contract this catalog
   repeats most, because transport retries blindly and correctness is proven by ID.
3. **The local store is a system of record for reads; the server is the system of record for
   order.** Offline reads come from local rows (24, 25, 21, 32); offline *writes* accumulate
   in a durable outbox written before any UI optimism (25's "write-then-render, the only
   unforgivable order").
4. **Reconnect = flush outbox, then pull from a cursor.** One reconciliation point, one
   ordering token, gap-fill by sequence, never fabricated deltas — token too old falls back
   to a bounded full resync (21: 2.5 MB; 24: bounded catch-up).
5. **Merge granularity is a named decision, never whole-object LWW.** Field-level LWW with
   delete-wins-and-restore (21), op-order insert/delete merge on ordered identity (32),
   SKU-union with quantity-add (39), op transformation (46). Where merges are truly trivial
   (monotone taps in 25), LWW is fine and *saying so* is the design.
6. **Deletions are tombstones with bounded retention and a restore path** — 21's ~30-day
   deleted-event content, 32's bundle tombstone overlay. Two explicit costs in every case:
   tombstone size (46's 100K+ per char is the reductio) and a device offline past retention
   reconciling to a bare tombstone (21: accepted, bounded).
7. **Presence never rides the sync channel.** Ephemeral state is a TTL'd, lossy, separate
   lane in every case that has it (46's ~3–10 s hub, 03, 05) — a dropped cursor update is a
   cosmetic nothing; a dropped op is a divergence.

**Boundaries — where eventual-merge is unacceptable.** The pattern holds for user-authored
state that is *convergence-tolerant* (text, calendars, playlists, feeds, carts). It stops at
three lines, all explicit in the catalog:

- **Money and seat inventory do not merge.** [37.ticketmaster](37.ticketmaster/README.md)
  forces per-shard serialization of the seat/order state machine and
  [44.payment-system](44.payment-system/README.md) never merges balances — a reconciliation
  sweep *finds* breaks and escalates, it never auto-merges money. 27 carries the same rule to
  devices: optimistic local actions are fine upstream of settlement, but records that reach
  the ledger after settlement go to manual reconciliation, "never silently to the PSP."
- **User-visible merge mistakes are worse than coordination cost** — exactly 46's argument
  against LWW ("silently drops the other person's text in exactly the offline cases offline
  exists for") and 21's argument against whole-event LWW ("a user who sees one edit silently
  vanish loses trust").
- **Single-writer vs merge is itself a fork in the pattern.** With one writer per key, the
  merge policy disappears and the outbox contract alone suffices (27, 22, 19, 24 — a
  device/user is the sole writer of its own intents); the merge machinery above is needed
  only when multiple writers can concurrently change the *same* record (46, 21, 32, 39).
  [CONSENSUS-AND-QUORUM.md](CONSENSUS-AND-QUORUM.md) is the server-side sibling of that
  choice: sequencing vs merging, decided by whether the reducer is deterministic.

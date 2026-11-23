# 04 — Deep dives

## 1. Concurrency model: OT vs CRDT (THE dive)

Two people type into the same document; the system must make their screens agree and stay
agreed — for *any* interleaving, offline gaps included.

**Option A — CRDT (conflict-free replicated data type, per-character identity:
unique IDs + position-aware merge)**:
- wins: peer-to-peer sync possible (no server needed); each character is immutable-ish
  metadata (ID, tombstone) so merges are locally computable; offline-first systems love it
  (cf. [21.frontend-calendar/04-deep-dives.md](../21.frontend-calendar/04-deep-dives.md),
  which evaluates full CRDT replication as Option A and rejects it there for the same
  reason Docs does: an authoritative server is already required).
- loses: metadata per character balloons (a 100K-char doc carries 100K+ tombstones and IDs;
  memory and wire overhead); garbage collection of tombstones across offline replicas is the
  genuinely hard open problem; for text with an authoritative server already required (ACLs,
  history, sharing), CRDT's main win (server-optional) is not needed.

**Option B — Operational Transformation (server-sequenced ops; clients and server transform
concurrent ops against each other)**:
- wins: ops are small (chars + positions + author + revision); a central sequencer gives a
  total order (one authority, one permission check, one history); transformation for text
  has few op types (insert/delete/style) and is well-understood (this is Google Docs'
  lineage, publicly discussed since ~2010 as OT-based).
- loses: transform correctness is subtle and *its bugs show up as user-visible merge errors*;
  requires online server (offline needs a client transform-queue discipline — dive §4);
  the server does per-op work (sequencing + fanout) that a fully-local CRDT avoids.

**Chosen: B (server-sequenced OT).** The deciding facts: (1) Docs *already* requires an
authoritative server for permissions/history/sharing — the CRDT's central win is dead weight;
(2) per-character metadata (CRDT) multiplies the doc model cost ~10× for a product whose
median doc has no collaborative conflict at all; (3) OT bugs are contained: op types are few
and transforms are exhaustively testable (property-based tests over op interleavings are a
standard practice here). Cross-links: [21.frontend-calendar](../21.frontend-calendar/README.md)
reaches the same answer on the same decision axis — both products already require an
authoritative server (ACLs/history here, event versioning there), so CRDT's
no-server-coordination win is dead weight in both; Calendar picks server-sequenced idempotent
ops, Docs picks server-sequenced OT.

**Failure modes**: a transform bug ⇒ divergence — the mitigation is *tests*, not design:
property-based tests generate op interleavings and assert t(1): "applying ops in any order
yields the same doc state"; server-client transform implementations must be the *same
library* (two implementations drift — dive §2's contract).

## 2. The op loop: sequencer, revision numbers, and the undo problem

**Topology. Option A — P2P (clients exchange ops directly, each converging):** wins: no
server in the latency path; loses: N×N connections, no single permission point, no
authoritative history — and convergence requires every peer to do the full transform dance.
**Option B — star (server sequencer, clients talk to the server only, chosen):** wins: total
order per doc from one writer-visible place; fanout linear; ACL enforced where ops land;
offline = transform-against-server on reconnect. Loses: server hop in the ~200 ms path (fine:
one hop is inside budget — est. sheet), and 1 sequencer per doc is a per-doc SPOF until
failover (op log is durable before ack, so failover replays, clients re-sync by revision).

**The loop.** Client op carries an id + the client's known server revision:
1. client applies locally (instant typing, est. 01's < 50 ms);
2. server: ACL gate → sequence (assign server rev) → store durable → broadcast;
3. subscribers (including author): transform the incoming op against their own un-acked
   local ops (two-way transform, the "TP1/TP2" discipline in the OT literature) → apply.

Batching: coalesce keystrokes into ~2 ops/s while typing (est. sheet) with a ~50 ms max-batch
window — a larger window breaks the remote-visibility budget; smaller multiplies op volume.

**Undo/redo is a transform, not a reverse.** Naive undo (delete the chars I added) deletes a
teammate's *reply* that landed inside your text meanwhile. Correct undo: select the inverse op
of your last op — then transform that inverse against every op that happened after it, so it
only undoes your change in *today's* document coordinates. Same machinery as any other op
once transformed; the trap is only in believing undo is special.

**Failure modes & mitigations**: duplicate op delivery (at-least-once broadcast) →
server-revision dedup per client; client crashes mid-op → its mirror rebuilds from
snapshot+ops (revision = recovery token); sequencer failover → op log durable-before-ack ⇒
new leader serves at the last revision; a client falls behind > threshold → full re-sync is
cheaper than replay.

## 3. Document model & incremental rendering

Model: text as a character array (or chunked array) plus *sparse annotations*: style runs,
link ranges, comment anchors, all positioned by offset. Positional identity under concurrent
edits: positions are *transforms-of-record*, not stored coordinates — every concurrent op
must remap positions (a comment anchored at offset 512 survives an edit that inserts 20
chars before it, because the anchor moves by transformation, deep dives §2's same math).

- Sub-decision — fractional/unique per-char IDs (CRDT-style) vs transform-on-write positions
  (OT-style): IDs guarantee stable anchors *for free* but cost per-character metadata (dive 1's
  arithmetic); transform-remapped positions cost transform correctness but keep the model lean.
  **Chosen: transform-remapped** for the doc content, with anchors stored as positions in the
  op stream and remapped by the same transform pipeline — one math, one implementation, no
  dual identity systems.

Rendering: a 200K-char doc re-laid-out per keystroke is a client-killer. **Incremental
layout**: line/page cache invalidated per changed region; typing in paragraph 4,000 does not
re-flow paragraphs 1–3,999 (except at page-break boundaries); viewport-first rendering
(visible lines prioritized — a windowed-list cousin of
[19.frontend-chat-app](../19.frontend-chat-app/README.md) §virtualization, with the added
wrinkle that *content above the viewport can change height* from other users' edits — handled
by anchoring the viewport to a *document position*, not a pixel offset).

**Failure modes**: annotation drift (anchor not transformed) ⇒ comments point into the void
— anchors are ops in the log and get the same transformational guarantee as text; pathological
doc (one paragraph = 500K chars) ⇒ chunk the array (tree-of-chunks) so position→offset is
O(log n) and edits split/merge chunks, never one contiguous buffer copy; style-table growth on
paste-bombs (paste 1 MB with 100K styles) ⇒ styles dedup'd/compressed at mirror level.

## 4. Offline: local mirror + reconnect transformation

Offline editing is *just* the op loop with the server disconnected; the client's ops queue
locally, and the hard part is the reconnect merge.

- Client keeps: local mirror @ client_rev, an ordered op queue (its own, since disconnect).
- On reconnect: fetch server snapshot+ops (client_rev → head); transform each *local* op
  against all *server* ops (two-way transform, so both sides' positions stay honest) — the
  local queue shrinks op-by-op as it is replayed onto the new world; ack each op as it lands
  server-side.
- **Same user, two tabs/devices** = two clients by design (no special casing): each transforms
  against the other through the server; merge behavior is *identical* to two humans — this is
  a feature (no device-profile forking in the model).

**Option A — offline queue and hope (last-write-wins on conflict)**: trivial code; loses:
silently drops the *other* person's text in exactly the offline cases offline exists for.
**Option B — transform-on-reconnect (chosen)**: the transform engine already exists for
online use (dive 2) — offline reuses it; the cost is a bounded replay window (offline for a
week vs the compaction cadence — compaction is snapshot-based so the replay window is
snapshot-age, not doc-history-age).

**Failure modes**: clock skew irrelevant (server revisions, never timestamps — est. 01);
offline queue vs *permission revoked while offline* → reconnect: ops rejected, doc downgraded
to view (with the pre-rejection local state preserved locally for the user to copy out — an
explicit UX state, not a silent loss); corrupted local mirror → re-fetch snapshot, replay
queue (the snapshot is the recovery anchor).

## 5. Presence, comments, permissions — three channels, three guarantees

**Presence** (cursors, selections, "User X is typing", avatars): ephemeral pub/sub, TTL'd
(~3–10 s liveness), rate-limited (a client may send cursor updates at ~1–2 Hz; the *op* stream
carries no presence) — lossy by design: a dropped cursor update is a cosmetic nothing; a
dropped *op* is a divergence. Presence fanout for a 200-viewer doc is the biggest *traffic*
in the system that has zero durability requirement — the per-doc fanout math of 02 applies
at a higher rate here (estimate: ~1–2 Hz per participant means a 200-viewer doc pushes
~200–400 msg/s outbound, more than its op stream).

**Comments**: durable data with anchors (dive 3), delivered via the event bus to
notifications (dedup'd/throttled — same consumer discipline as
[42.linkedin](../42.linkedin/README.md)'s derived stores). Comment *reply resolution states*
ride the same doc op log? No — comments are a separate store keyed by anchor id with their own
ops; only the *anchor position* is transform-managed. Chosen split: comment threads are
conversational state (their own history, their own notifications), anchors are document
state (transform-managed) — merging them corrupts two things at once.

**Permissions**: per-doc ACL + link sharing; checked at open; **live re-check on change**
(demote/revoke applies to active sessions within ~seconds — a security boundary, not a
 UX nicety, so it is enforced at the collaboration service gate, not in the client UI);
viewer-sessions receive snapshots + coarse progress, *not* the op stream (a viewer neither
needs op-level traffic nor should receive edit-path detail — least-privilege at the channel
level, the same "consumers stay consumers" posture as
[42.linkedin/04-deep-dives.md](../42.linkedin/04-deep-dives.md) §5).

**Failure modes**: presence hub outage → cursors vanish, editing unaffected (explicit
degradation, by design); notification flood on a 500-viewer shared doc → per-user/per-doc
dedup windows in the notification service (a document that changes 1,000×/hour must never
generate 1,000 emails); ACL race (revoke lands mid-edit) → the *next* op from the removed
editor is server-rejected; their local mirror keeps their own text (their data, not the
collab doc's), and the session downgrades — fail closed on the server, never trust the client
to enforce.

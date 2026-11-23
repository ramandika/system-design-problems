# 01 — Requirements

## Functional

Core v1:

- **Document editing**: rich text with styles, lists, tables; content is text plus *sparse
  annotations* (a style range, a comment anchor, a link) — not styled-character soup and not
  a binary format.
- **Real-time collaboration**: multiple editors see each other's edits within ~200 ms p95;
  convergence is guaranteed for *any* interleaving (deep dives §1); cursors and selections of
  others visible (presence).
- **The server is the sequencer**: every edit is an op in the doc's ordered log; revision
  numbers, not timestamps, define order; clients state their revision on every op (deep
  dives §2).
- **Undo/redo per user**: undoes *your* last operation transformed against whatever happened
  meanwhile — others' edits stay intact (deep dives §2, sub-problem 3).
- **Offline editing**: full doc mirror locally; ops queue while offline; on reconnect, local
  ops transform against server ops and converge; multiple tabs of the same doc = two
  independent clients (deep dives §4).
- **Sharing & permissions**: per-doc ACL + link sharing (view/comment/edit), owners/editors/
  viewers/commenters; permission changes apply to *live sessions* (a removed editor stops
  editing within seconds).
- **Comments**: anchored to text positions that survive edits and offline merges; threads
  with resolve state; email/push notifications (dedup'd — same discipline as
  [42.linkedin](../42.linkedin/README.md) WVYP notifications).
- **Version history + export**: revision log browsable to any prior version (from the op
  log — deep dives §2); export to open formats as a background conversion.

Out of scope v1: Sheets/Slides internals (cell ops add a dimension, text ops remain the
canonical case), embedded drawing editors (link-out objects), Apps-script, offline *conflict*
UX beyond auto-merge, comments' ML suggestions.

## Non-functional

| Property | Target | Consequence |
|---|---|---|
| Local keystroke render | < ~50 ms | client-side; nothing blocks typing on the network |
| Remote edit visibility | p95 ≤ ~200 ms (typing partner's text appears) | op broadcast path is the latency-critical route; batching must stay inside it |
| Presence freshness | ≤ ~1 s, lossy, ephemeral | never enters the durable op stream (deep dives §5) |
| Durability | an acknowledged edit survives server loss | op persisted (log + snapshot) before ack; local mirror before that |
| Convergence | *guaranteed* for any concurrent-interleaving | the OT/CRDT decision (deep dives §1) |
| Consistency (doc content) | server-sequenced total order; clients converge to it | no client-side "winner" logic — ever |
| Availability shape | editing: high; presence: degrade silently; export/history: eventually | sync path must lose capabilities, not correctness |
| Scale | ~10M concurrent editors peak (est.); ~4M ops/s peak (est., derived in 02) | small bytes, unmissable latency — see 02 |

## Scope decisions that shape everything

1. **One authoritative sequencer per document** (server), star topology — not peer sync:
   convergence follows from one total order; permissions have one enforcement point; the
   client transform logic handles "my op vs the world's" and nothing more (deep dives §2).
2. **Edits are operations, not bytes**: insert/delete/style ops against a document model
   with positional identity; the log is replayable to any revision (deep dives §2–3).
3. **Presence, cursors, comments-notifications are separate channels** with separate
   guarantees — conflating them with the op stream is the classic first-design mistake
   (deep dives §5).

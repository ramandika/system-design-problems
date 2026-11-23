# Case 46 — Google Docs

**The system**: a collaborative document editor — hundreds of keystrokes per second land on
the same document, each participant sees everyone else's typing in ~200 ms, edits survive
offline, and nothing ever visibly "merges wrong". The canonical real-time collaboration case;
pairs with [Case 21 — Frontend: Calendar](../21.frontend-calendar/README.md), which covers the
same sync problem from the client side.

**Why it's hard**:

- **Concurrent text edits converge**: two people insert into the same word simultaneously;
  the only acceptable outcomes are a merge both intended — this requires an operational
  transform (OT) or CRDT discipline, not a last-write-wins column (deep dives §1).
- **The server is a sequencer, not a database**: every edit becomes an ordered operation in a
  per-document log which the server transforms and broadcasts; revision numbers, not
  timestamps, are the truth (deep dives §2).
- **Undo must undo *yours*, not the world's**: after other users edit in between, undo is a
  mathematical transform, not a reverse operation (deep dives §2's hardest sub-problem).
- **Keystrokes are tiny but sacred**: the doc model is text + sparse annotations, and
  rendering a 200K-character doc on every keystroke would sink the client; incremental layout
  is a client-systems problem (deep dives §3).
- **Offline is mandatory**: a laptop on a plane keeps typing; on reconnect its local ops must
  transform against everything the world did meanwhile (deep dives §4).

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: server-sequenced operation logs with client-side transformation
(OT) vs peer-capable CRDTs, star topology vs P2P sync, document models with positional
identity (fractional/unique IDs), sparse annotations over text, ephemeral presence channels
separate from durable op streams, comments anchored to moving text.

**Try it first**: before reading 03, answer: (1) you type "hello" at position 100 exactly
when a teammate types "world" at 100 — who ends up first, and how does *both* of your
screens agree? (2) your teammate presses undo right after you typed — whose text comes back?
(3) a 50-person meeting doc gets a 100th viewer — what does the *viewer* receive, and why
not the same stream as editors?

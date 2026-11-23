# 01 — Requirements (the client's)

The server is Case 03's system; this file states what the *application* must do and how
fast it must feel. All latency/memory budgets are **design targets** (estimates to engineer
against), not measured facts.

## Functional requirements

- 1:1 and group conversations (groups up to ~1,024 members — Case 03's bound).
- Text messages: emoji, links, lightweight markdown (quote/reply blocks, code spans).
- Media: photos, video, files. Send from camera/gallery/picker; receive inline.
- Receipts visible to the user: sent / delivered / read (read receipts for groups are
  aggregated by the server, not per-member — Case 03's constraint).
- Unread counts per conversation, with @-mention counts for muted channels.
- Typing indicators and presence (online/last seen), best-effort.
- Offline: read history, compose, and send; sends flush on reconnect. Never lose a message
  the user pressed send on.
- History: infinite scroll back, jump-to-message (from a mention or search hit), search
  within locally stored messages.
- Multi-device: same account on phone + desktop; read/send on one device converges on the
  other. Push notifications wake the client when backgrounded (the notification content
  is computed server-side; the client just displays and syncs).

## Non-functional requirements (client budgets)

| Latency/memory | Target | Why |
|---|---|---|
| Keystroke → paint | < 16 ms (one frame) | typing is the most sensitive input |
| Send tap → message bubble visible | ~0 server time; < 16 ms local | optimistic render |
| Send → server ack (informs state machine) | ~50–300 ms same-region, s–multi-s tails on mobile (network, not us) | drives sent/failed states |
| Receive event → bubble painted | < 50 ms p95 target | parse → store → render |
| Scroll over 100k messages | 60 fps, < 16.7 ms/frame, no jump artifacts | the core interaction |
| Media placeholder paint | < 16 ms; full image swaps in later | no layout shift allowed |
| Reconnect → live again | < 1 s excluding backfill | backfill proceeds in background |
| Cold start → first conversation painted | < 2 s native, < 4 s web (targets) | |
| Steady-state memory | ≤ ~250 MB including maps and decoded media | phones |
| Local storage | text store tens of MB; media cache capped ~1 GB | |

These drive the architecture directly: the 16 ms budgets mean nothing in the send tap
handler may wait on I/O; the 60 fps budget means nothing layout-affecting may happen
during scroll without being pre-reserved.

## Scope decisions

**In scope**: the application layer — UI state, message state machine, virtualized list,
local persistence, offline outbox, reconnect/backfill client protocol, unread computation,
multi-device reconciliation, media preview pipeline.

**Explicitly out of scope** (each is its own design problem elsewhere):

- Server architecture, fanout, message store — [Case 03](../03.whatsapp/README.md). We
  *consume* its contract (client_msg_id dedup, conversation-local seq, store-and-forward).
- End-to-end encryption design — Case 03's dive 5; but E2E appears here as a *constraint*
  the client inherits (e.g., the server cannot thumbnail encrypted media, so the client must).
- Voice/video calling (WebRTC is a different system design).
- Push notification infrastructure and OS notification plumbing (we specify the
  client-visible behavior only).
- Desktop app packaging, platform notification quirks, keyboard shortcuts.
- Moderation, spam, compliance — server-side; and with E2E the client is mostly outside
  that story too.
- Bots, integrations, rich embeds beyond a preview pipeline.

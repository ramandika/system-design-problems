# 01 — Requirements

## Functional

Core v1:

- **Send message**: 1:1 and group (~ up to 1024 members). Text, media refs (photos/video
  bytes ride a parallel upload path — the metadata "here's my photo, ID+key" rides messaging).
- **Delivery state machine** per message: sent → delivered → read. Read receipts are
  themselves messages.
- **Offline queueing**: receiver offline (bad network, app closed for hours/days) must still
  get the message, intact, in order.
- **Typing/online indicators**: ephemeral, best-effort, never persisted as durable state.
- **Multi-device**: same account, 2–5 devices, each must receive messages and each must be
  able to *initiate* E2E encryption handshakes.

Out of scope v1 (note them): voice/video calls (real-time media, separate stack), stories,
payments, business messaging, backup/restore policies beyond "media is deduped client-side".

## Non-functional

| Property | Target | Consequence |
|---|---|---|
| Delivery latency | < 1 s when both online, mobile network | drives persistent connections & protocol efficiency, not poll-heavy design |
| Delivery guarantee | exactly-once as-experienced (at-least-once wire protocol + client dedup); no messages silently dropped | the single most important property |
| Connection availability | ~2B users holding persistent connections to a global fleet | the "connection problem" dominates infra design |
| Offline retention | store queued messages for receiver for N days (say 30), then drop or re-request (client can re-pull) | bounded storage cost |
| Consistency | per-conversation strict ordering; cross-conversation ordering not required | dramatically simplifies ordering vs global order |
| Privacy | E2E encryption: server never sees message content | server routes opaque blobs; also simplifies state machine (only envelope, not content) |
| Battery / mobile cost | keep-alives tuned so phone radio sleeps between messages | pushes the "don't poll" design & protocol compression |
| Availability | sending: must work; indicators/stories: can fail | sending failure is the only real outage |
| Abuse | spam / mass-messaging must be rate-limited per account | same shape as Twitter (see deep dives) |

## Scope decisions that shape everything

1. **Connections, not polling.** Polling 2B users every 5 s = ~400M req/s for near-zero
   information. A persistent socket is *cheaper* in both latency and (over a day) in
   requests — the design cost is holding connections, which becomes its own subsystem.
2. **Per-conversation ordering, not global.** Users only notice wrong ordering *within* a
   conversation. This makes ordering trivially solvable per conversation.
3. **E2E encryption by default.** Design consequence at the infrastructure layer, not just
   the crypto layer: the server is a *relay+mailbox*, not an inspector — most "content-aware
   optimizations" (e.g., server-side dedupe of identical text) are off the table.

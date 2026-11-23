# Case 19 — Frontend System Design: Chat Application

**The system**: the *client* of a WhatsApp/Discord-like chat app — the mobile/web/desktop
application itself, not the servers behind it. The server contract is whatever
[Case 03](../03.whatsapp/README.md) defines: persistent socket, at-least-once wire with
client-side dedup, conversation-local sequence numbers. This case designs what runs inside
the app: state, rendering, storage, sync.

**Why it's hard** (all client problems):

- **Perceived latency is the product.** The server round-trip is 200 ms–1 s+; the user
  expects the sent bubble instantly. The client must fake it (optimistic UI) and reconcile
  it against reality, including reordering.
- **History unbounded, device bounded.** A Discord-style power user accumulates ~100k
  stored messages per device; rendering them all is memory suicide. The view is a
  variable-size virtualized window over an append-only log.
- **The network is always eventually broken.** Offline sends, reconnects, seq gaps, days
  of missed messages, hot channels that moved 100k messages while away — the client needs
  a durable outbox and a backfill protocol, mirroring the server's store-and-forward.
- **State lives in several places at once**: view state, in-memory store, durable local
  store, server state, and (on multi-device) *several devices' local states* — with
  last-writer-wins semantics everywhere the user doesn't notice.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: optimistic UI + client-side state machines, list virtualization,
client-side durable queues, gap-filling sync protocols, deriving computed UI state
(unread counts) from raw log + markers, media thumbnail pipelines. These are the client-side
twins of the server patterns in [Case 03](../03.whatsapp/04-deep-dives.md) (delivery
protocol, offline queues, seq numbers).

**Try it first**: before reading 03/04, answer: (1) what state machine does a single
outgoing message live in, and who owns the transitions — the view, the store, or the sync
engine? (2) Why can you not render 100k messages with just "more" pagination, and what does
the scroll position actually anchor to? (3) When the app was offline for 8 hours, exactly
what does the client send first on reconnect, and what can it render immediately?

**Primary sources**: none vendor-specific — this is a synthesis case. Numbers are labeled
estimates; where a public figure is cited (media sizes, WhatsApp/Discord design behavior),
it's marked "publicly reported" and the counterpart case carries the sourcing.

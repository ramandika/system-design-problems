# 01 — Requirements

## Functional

**Core messaging**

- 1:1 and group conversations; text messages with near-instant delivery when the
  receiver is online.
- **Conversation history lives server-side**: a new device (fresh install, web login)
  opens a thread and sees years of scrollback. No requirement to keep history on the
  primary device first — the server copy is the primary copy. (Historical Messenger;
  this is the decision that scopes everything else.)
- **Inbox**: a paginated list of threads ordered by recency, with unread counts, per-thread
  snippets of the latest message, and jump-to-search within a thread. Web, iOS, Android
  share the same server truth.
- Typing indicators, delivery/read receipts, reactions, threads stay editable/annotated
  (reactions, reply-quote) — richer state surface than a pure relay system.

**Rich media**

- Send photos/videos; the sender's client compresses and uploads; recipients get an inline
  preview plus full-resolution download on tap.
- **Link messages**: paste a URL and every participant sees a titled thumbnail preview in
  the thread — meaning *someone* fetched that URL server-side once.
- Stickers, GIFs, voice clips: catalog-driven media types, not raw uploads.

**Integration surface (scoped, see decisions)**

- Business accounts messaging users (one-way and two-way threads, templates).
- Third-party platform apps at the historical "Messenger platform" layer (bots, games in
  threads) — in scope as a *load and abuse model*, out of scope as a product spec.

**Multi-device & sync**

- Any subset of the account's devices is simultaneously active; all of them converge on
  the same thread state (read markers, reactions, message list) because the mailbox is
  server-side. No primary-device gating.

## Non-functional

- **Delivery latency**: sub-second for an online receiver on a healthy network — same bar
  as WhatsApp; the inbox, however, is *read-optimistic* (scrollback can be 100ms+).
- **Durability**: a sent message is never lost and never reordered within a thread,
  even across datacenter failover — as the user sees it.
- **Scale**: ~1B monthly actives, messaging volume in the tens of billions of
  messages/day (see [02](02-scale-estimates.md) for the arithmetic and the source caution).
- **Consistency**: the inbox may be a few seconds stale (badge count lags one message);
  a message list must not be stale (what you append is what you see).
- **Privacy posture**: two regimes inside one system — default-plaintext (historical,
  enables server-side ML/ranking/moderation/search) and the migrated default-E2EE regime.
  The design must name which features die in the second regime.

## Scope decisions

**In**:

- The **mailbox as a storage engine**: per-user history, inbox ordering, unread counters,
  search-index integration — the parts WhatsApp punts on because it doesn't need them.
- **Write-fanout vs per-conversation log** as the central storage trade-off, and the
  hybrid Messenger uses.
- **Media as a first-class path**: preview generation, link fetch/caching, catalog media.
- The **E2EE migration** as a design problem: which server features are lost and how.

**Out** (deliberate, and where this case differs from siblings):

- The **connection layer itself** — gateways, keep-alive tuning, reconnect storms,
  ACK/dedup wire protocol. Messenger's needs are the same family as WhatsApp's and the
  mechanism is covered exhaustively in [WhatsApp deep dives](../03.whatsapp/04-deep-dives.md);
  we treat "a persistent connection per user" as an existing primitive here.
- Large community/room broadcast mechanics — that is Discord's problem
  ([Discord deep dives](../05.discord/04-deep-dives.md)); Messenger groups are sized in
  the hundreds, not hundred-thousands.
- Voice/video calling subsystems (WebRTC SFU shape is in Discord's case).
- Client-side rendering problems (message list virtualization, optimistic UI) — covered by
  [frontend-chat-app](../19.frontend-chat-app/README.md) and the mobile client case
  [mobile-chat-app](../24.mobile-chat-app/README.md).
- Voice-message transcription, payments, specific recommendation models — named, not designed.

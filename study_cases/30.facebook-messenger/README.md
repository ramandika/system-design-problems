# Case 30 — Facebook Messenger

**The system**: high-throughput messaging with rich media (photos, links, stickers, GIFs),
group chats, and a threaded **inbox** the user browses across web, iOS and Android. The
defining trait versus [WhatsApp](../03.whatsapp/README.md): history has historically lived
**server-side** — any device can open a decade-old thread — and only recently became
end-to-end encrypted, inverting the order of the two hard problems.

**Why it's hard**:

- **Per-user mailbox model at web scale**: the server holds a durable, queryable copy of
  everyone's history, organized as an inbox of threads with unread counters, search, and
  scrollback — a mail-like system with chat-like latency requirements (see
  [deep dives](04-deep-dives.md)).
- **Rich media is the load, not a side path**: link previews fetched server-side, image
  pipelines, stickers/GIF libraries — content flows through servers that historically see
  plaintext, which is what makes ML ranking, moderation and search possible at all.
- **An integration surface**: business messaging, platform apps, bots — third parties write
  into the same message store, so abuse and rate-limiting are externalized problems.
- **The E2EE migration**: retrofitting default E2EE onto a system designed around plaintext
  server truth — search, previews, moderation all move client-side. Doing it in the other
  order than WhatsApp (encryption first, features after) is the intellectual core of this case.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md). Pair with
[WhatsApp](../03.whatsapp/README.md) for the connection-layer/delivery-protocol material it
shares — this case deliberately does not repeat it.

**Patterns taught here**: write-fanout into per-user mailboxes vs per-conversation logs (the
Twitter-fanout question, [Twitter](../02.twitter/README.md), in a messaging shape — compare
[Discord's](../05.discord/README.md) per-channel model), inbox indexing and unread counters at
scale, link/media preview pipelines as a crawl-at-user-request problem, migration of a
multi-billion-user store, and what plaintext on the server buys you (and costs you).

**Try it first**: before reading 04, answer: (1) where is message history — one row per
message, or one per (message, recipient)? what does each choice buy? (2) how does an unread
counter stay correct across 4 devices and a web inbox that loads the last 20 threads, without
counting on read? (3) after default E2EE, which of those answers survives?

**Primary sources** (public engineering talks/blog posts around 2016–2018 unless noted):
Facebook engineering talks on Messenger's storage and mailbox architecture (per-user mailbox,
unidirectional log, mail inbox service); public reporting on the Messenger inbox HBase →
MySQL/MyRocks migration (~2018); public reported numbers — Messenger ~1 billion monthly users
(2016), Messenger+WhatsApp 100B+ messages/day combined (self-reported, Jan 2019); "Secret
Conversations" (Signal protocol, opt-in 2016) and Meta's announcement of default E2EE rollout
beginning December 2023. All figures are self-reported or estimates — see the caveats in
[02](02-scale-estimates.md).

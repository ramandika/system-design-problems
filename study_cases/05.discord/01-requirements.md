# 01 — Requirements

## Functional (in scope)

**Guilds & channels**
- Users create and join **guilds** (servers). A guild is a container: id, name, icon,
  member list, role model (owner / roles with per-channel permission overrides).
- Guilds contain **channels** of two kinds:
  - **text channels**: chat with persistent, paginated history.
  - **voice channels**: live audio/video rooms; joining one is an ephemeral "I am in this
    room" state, not a subscription to stored content.
- Membership is explicit: joining a guild is a real act (invite links, discovery), unlike
  Twitter's follow graph. A user can be in tens to hundreds of guilds.
- Per-channel **permission checks** gate reads and writes (role/channel-level, cached).

**Text chat**
- Send/receive messages in a text channel; messages are text, emoji, attachments (images,
  video), embeds, and reactions (emoji aggregates on a message).
- Full **persistent history** per channel, paginated oldest→newest and newest→oldest
  ("jump to message"), searchable in principle (search itself is a separate out-of-scope
  subsystem — see below).
- **Presence**: online / idle / do-not-disturb / offline, surfaced to the member list and
  to friends; typing indicators per channel; read-state per user per channel (the "new
  message" divider and unread counts).
- **Direct messages & group DMs**: classic bounded group chats (≤ ~10 members), same
  message store, no guild involved.
- **Mentions & notifications**: @user, @role, @everyone produce push notifications
  (push to devices offline / in another channel). Notification *policy* (mute channels,
  mentions-only, etc.) is per-user per-channel state.

**Voice/video**
- Join a voice channel → live audio (opus) and, on demand, video/screen-share, mediated by
  an SFU per room; participants see the participant list and speaking state live.

## Non-functional

| Dimension | Target (rough, engineering targets not SLAs) |
|---|---|
| Message delivery | p99 guild-local delivery < 500ms; DM/typing < 100ms best effort |
| History read | p99 recent-history read < 50ms; deep (months-old) pages < 500ms |
| Durability | A message accepted (200-OK) must NEVER be lost. Presence/typing may be lost freely |
| Ordering | Strict per-channel order as observed by clients, even across servers/guilds sending |
| Availability | Degrade, don't black out: history read-only beats chat down; voice degrades independently of text |
| Concurrency | Millions of concurrent gateway sessions (est.); a 100k-member guild must not be a hot loop anywhere |
| Voice latency | < ~100ms SFU forwarding target; total mouth-to-ear budget ~150ms |

**The durability split is the most important NFR line in this case** (same principle as
[WhatsApp §6](../03.whatsapp/04-deep-dives.md)): messages are must-not-lose; presence/typing
are must-not-cost-anything; read-state is a middle class (may lose the last few minutes of
unread positions, never the whole unread state).

## Explicitly OUT of scope

- **Full-text / semantic search**: a separate inverted-index subsystem (Discord runs a
  dedicated service); we assume a "search service" box in the architecture and design only
  its interface.
- **Bots and the bot API surface**: bots are ordinary gateway/Webhook clients for our
  purposes; their rate-limit economics are mentioned but not designed.
- **Moderation/abuse ML pipelines**: kept to an interface (a screening hook on message
  submit, async); the models and pipelines are out of scope.
- **E2EE for guilds**: guild chat realistically needs server-side features (search,
  moderation, mention routing, embed unfurling) that E2EE removes — see the E2EE trade-off
  analysis in [WhatsApp deep dives §5](../03.whatsapp/04-deep-dives.md), which applies in
  force. We choose centralized trust + transport TLS. Group DMs in a real product may
  differ; out of scope here.
- **Multi-region active-active for the message store**: we design region-affinity with
  cross-region read fallback and a documented failure mode instead. Rationale is in
  `04-deep-dives.md` §5.
- **Threading / forum channels / forum posts** as a distinct data model — we treat them as
  "message tree on top of the same channel store" and note the schema implication, without
  designing the UI semantics.
- **Payment / subscription / Nitro-adjacent state.**
- **Client platforms** (we assume the same client logic everywhere; protocol is the design).

## Scope decisions worth defending

- **One gateway protocol for chat + presence + voice signaling.** Trade-off: one protocol
  means one connection per client and shared auth/session state (cheap, uniform), but the
  voice path's bandwidth/latency needs get coupled to chat's socket economics. The real
  Discord splits this: one persistent *gateway* socket for events, and voice media rides a
  separate UDP connection after UDP handshake renegotiation. We pick the split, because
  chat over TCP-with-TLS and voice over UDP have disjoint loss/latency curves, and forcing
  voice packets through a TLS/TCP proxy would make congestion control fight the voice
  jitter buffer.
- **History is eventually paginated from the store, not fully pushed on join.** A 100k-member
  guild with 10 years of history cannot be "joined" by streaming every message. Joining
  delivers *state* (channels, permission snapshot, last-read positions, presence for
  relevant members) and the recent message window per visible channel; older history is
  fetched on demand.

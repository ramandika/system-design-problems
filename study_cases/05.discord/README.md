# 05 — Discord

## What the system does

A real-time communication platform organized around **servers (guilds)** — long-lived
communities a user joins as a member — each containing **channels** (text chat, voice,
forum-style). Inside a channel, chat behaves like a group chat (WhatsApp-style: bounded
membership per conversation, persistent history), but the membership envelope is not
bounded: a guild can have 100k+ members or, for a few famous guilds, millions. Users also
join **voice/video rooms** (WebRTC SFU-mediated), see **presence** (online/idle/status)
across all of their guilds, and get **push notifications** for messages in channels they
care about even when the app is closed.

Publicly reported scale (Discord engineering blog, various years, roughly): ~200M+ MAU, a
guild with millions of members (Midjourney's community), peak concurrent voice usage in the
millions of simultaneous participants across ~millions of live sessions (per-VM session caps
~500k in their Erlang layer, publicly documented), message store on the order of
**trillions** of messages (the Cassandra→ScyllaDB migration post, ~2022, cites a >1 trillion
row move). Treat all of these as "publicly reported roughly X", not audited figures.

## Why it's hard

Discord sits at the junction of the two previous cases, and inherits the hard half of each:

1. **The connection layer of WhatsApp** — millions of persistent sockets across a fleet,
   presence, gateway state — but *more* of it than a 1:1 messenger needs, because presence
   and typing are fanned out across every guild a user is in, and users are in tens of
   guilds. (See [WhatsApp deep dives](../03.whatsapp/04-deep-dives.md) §2 — Discord's own
   Manifold fix is the centerpiece of that section.)
2. **The fanout problem of Twitter** — a message in a 100k-member guild is a "write to
   one logical object, deliver to 100k subscribers" event, the same shape as a celebrity
   tweet. But unlike Twitter, everyone in a guild sees everything in a channel; there's no
   "skip fanout for celebrities, merge on read" hybrid because the recipient set is not
   follower-dependent — it is the channel's member set, uniformly huge for large guilds.
   (See [Twitter deep dives](../02.twitter/04-deep-dives.md) §1.)
3. **A storage problem neither of them has at this shape** — bounded, ever-growing,
   chronologically-sorted per-channel history, trillion-row scale, hot at the tail
   (recent messages) and cold at the head (years-old history in legacy guilds).
4. **A third workload, voice/video**, which shares nothing with chat's store-and-forward
   model: it is latency-critical (~150ms budget end to end), connection-oriented,
   mesh-vs-SFU, and suffers a completely different failure mode class (packet loss, NAT
   traversal, SFU capacity).

The combination is the difficulty. Each workload individually has textbook answers; the
design pressure here is that all four share one connection layer, one identity, and one
permission model, and a single user action ("join big guild") can change the load shape of
every one of them.

## Reading order

1. `01-requirements.md` — what we build, what we explicitly do not.
2. `02-scale-estimates.md` — the numbers the design must survive.
3. `03-architecture.md` — gateway / backend-API / pub-sub / store / SFU layout.
4. `04-deep-dives.md` — the core: gateway fleet, broadcast fanout without storms, pub/sub
   topic routing, voice SFUs, per-channel ordering & storage.

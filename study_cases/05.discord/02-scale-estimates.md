# 02 — Scale estimates (all numbers are rough estimates unless marked "publicly reported ~X"; decimal units throughout: 1 GB = 10^9 B, 1 TB = 10^12 B)

## Users & concurrency

- **MAU: ~200M** (est., order of magnitude from publicly reported ~200M+ MAU).
- **Peak concurrent gateway sessions**: MAU does not bound concurrency. Concurrent sessions
  ride on DAU × devices × session stability. Assume:
  - DAU ≈ 25% of MAU (rough) ≈ **50M DAU**.
  - Peak *concurrent* sessions ≈ 15% of DAU online simultaneously at peak hour (Web chat
    skews to long sessions; mobile to background reconnects) ≈ **~8M concurrent sessions**,
    call it **10M sessions** as the design envelope.
  - Sessions per user: mobile app + desktop app + browser tab frequently coexist → design
    for **1.5 connections/user**, i.e. ~6.7M distinct users across ~10M sockets (estimate).
- **Sessions/gateway box**: a non-trivial engineering question — WhatsApp publicly held ~2M
  sockets/box (Erlang, 2012); Discord publicly reported ~500k live Erlang *sessions*
  (guild-level session processes, not raw sockets) per VM at peak, later fixed via Manifold.
  For our design envelope, assume **100–500k sockets per gateway node**. The 10M-session
  envelope is the *global* peak, with sessions spread across regions — but each active
  region is provisioned to carry the full envelope alone, because a regional failure is
  exactly when sibling regions inherit the sockets (a stateful tier cannot lean on
  region-summing) →
  **10M / 250k ≈ ~40 gateway nodes** per region × 3 regions + headroom ≈
  **~150 gateway nodes** fleetwide at this mid-socket assumption (estimate; at the full
  100–500k sockets/box range this spans ~75–400 nodes fleetwide — the figure quoted again
  in the derived capacity lines below).

## Message volume

- Messages/day (est.): publicly reported roughly **1B+ messages/day** for Discord in prior
  years; assume a range and design at **~1.5B/day** envelope.
- Peak message rate: daily volume is NOT flat — peak hour ≈ 2× the mean hour, and mean is
  1.5B/86 400s ≈ **~17K msg/s** mean → peak ≈ 2 × 17K ≈ **~35K msg/s** app-level. Design envelope
  **100K msg/s** (a viral event inside one giant guild can spike one channel hard, but the
  fleet total is what the store must survive).

## The fanout math (where Discord ≠ Twitter)

A message in a channel of N members triggers up to N deliveries. Distribution of guild
sizes is heavily long-tailed (estimate): median guild < 100 members, but thousands of
guilds with >100k and a handful with millions.

- Assume 30% of traffic lands in "huge" channels (N ≥ 10 000) (estimate).
- Per-second delivery work at peak: if 100K msg/s × 30% × avg 20K members in that bucket ≈
  **~600M member-deliveries/s** — clearly the fleet cannot afford an eager **push of the
  message body** to every member.
- This is the central divergence from Twitter's hybrid
  ([Twitter §1](../02.twitter/04-deep-dives.md)): there, fanout-on-write is the *common*
  case and fanout-on-read is the celebrity exception. Here, most *messages* are read by
  only a tiny fraction of each channel's members (a 100k guild's channel sustains maybe
  1–5k humans actively watching it — estimate), so blind delivery to members is wasted
  work. The design consequence developed in the deep dives: **membership ≠ delivery
  subscription**; delivery is to *connected sockets in that channel* or *clients with an
  unread-count need*, not to *members*.
- Actual delivery: if 2K sockets watch a huge channel and 100K msg/s arrive with 30% in
  huge channels → ~30K msg/s × 2K ≈ **60M socket-writes/s** just for big channels — still
  the dominant cost. This is why (a) fanout goes through per-core worker batching (the
  Manifold lesson), and (b) unread counts are decoupled from message delivery (§2/§3 of the
  deep dives).

## Storage

Per-message footprint (estimate, post-write only; not counting attachments):

- average text message ≈ 300 bytes payload + 200 bytes metadata/row overhead ≈ **500 B/row**.
- **1.5B msg/day × 500 B ≈ 750 GB/day** of new rows.
- Attachment media: assume avg 1 attachment per 10 messages at 1 MB (estimate, image-heavy
  skew) → 150M files × 1 MB = **150 TB/day** into object storage (decoupled from the row
  store; CDN-fronted).
- Yearly row volume: 1.5B msg/day × 365 ≈ **~550B rows/year**, ~275 TB/year for rows
  alone. Retention policy: keep
  every message forever (Discord's stance, publicly ~trillion messages stored), which is
  precisely what pushes toward time-bucketed partition keys and cold-tiering (§5).
- **Reactions, read-state, presence state**: reactions ≈ 2/day/user × 50M DAU ≈ ~100M/day
  tiny rows;
  read-state is one row per (user, channel) updated on read — ~200M active-state rows
  hot (estimate); presence is not persisted at all (in-memory only).

## Bandwidth

- Chat inbound (upstream): 100K msg/s peak × ~1 KB mean payload → **~100 MB/s inbound**
  (estimate) — trivial per-fleet, but bursty per-guild.
- Chat outbound (downstream): from the fanout math above, ~60M socket-writes/s × mean
  ~1 KB hydrated payload ≈ **60 GB/s** at peak (estimate). This, not inbound, is the
  bandwidth curve to architect for — and it explains why the gateway tier is sized by
  outbound egress, not socket count alone.
- Voice: 50 ms opus frames ≈ ~12 kbps opus voice (opus VOIP mode; the codec floor is
  ~6 kbps, typical voice runs ~16–30 kbps) + ~40–50 B of IP/UDP/RTP overhead per packet
  (~7 kbps at 20 packets/s) ≈ ~2.4 KB/s per sending participant (estimate). 1M concurrent
  speakers × ~2.4 KB/s ≈ **~2.4 GB/s** SFU-side; SFU receives and
  re-sends, so traffic multiplies by the receiving group per SFU room (this is why SFU
  capacity, not chat, is where voice egress lives).

## Derived capacity lines (all rough)

- Pub/sub event rate: each chat message fans out into ≥ 1 topic event + presence/typing
  events. Presence+typing updates across all guilds are the noisy majority: est. 10 events
  per message-equivalent in aggregate → pub/sub bus must carry **~1M events/s** peak (estimate).
- Gateway egress is the binding constraint: ~75–400 gateway nodes fleetwide (each region
  sized for the full 10M-socket envelope at 100–500k sockets/box, × 3 regions + headroom)
  each pushing 60 GB/s ÷ 75–400 ≈ **0.15–0.8 GB/s** at peak → NICs and hydrated-payload size
  are the two knobs.
- Message store ingest: 100K msg/s peak, write-biased, sequential per channel → per-channel
  append-only partition design, quorum writes (publicly, the Discord ScyllaDB deployment
  reported p99 inserts ~5 ms steady, historical reads ~15 ms after their migration (down
  from 40–125 ms) — see [WhatsApp §3](../03.whatsapp/04-deep-dives.md)).

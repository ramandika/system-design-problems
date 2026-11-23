# 02 — Scale estimates

Assumptions (order-of-magnitude; adjust and redo arithmetic if they change):

- ~2B registered users, ~500M daily active, ~100M concurrent online at once on average
- ~2B users → the space of possible 1:1 conversation pairs is huge; assume ~50B messages/day
  for 1:1 plus media — the headline assumption, (estimate). The widely cited 100B+/day is
  the combined Messenger+WhatsApp total — attributed in ../30.facebook-messenger, which
  itself assumes ~40B/day for Messenger; WhatsApp alone was publicly reported in the
  tens-of-billions/day range, so ~50B/day is the top of that band, not another app's volume.
- ~2 connected users → 1 conversation → ~50B 1:1 messages/day ⇒ ~500M actively-talking
  pairs/day at ~100 messages per active pair (500M × 100 = 50B)
- Median message size (text) ≈ ~100 bytes (envelope, small content, protocol overhead)
- Media: ~20% of messages are media; average media file ~1 MB after client-side compression

## Primary datapoints (historical, self-reported — not the current stack)

Verified against WhatsApp's own blog and Rick Reed's Erlang Factory talks. All vendor
self-reports, and all from the 2012–2014 FreeBSD/Erlang era — WhatsApp migrated off that
stack after the Facebook acquisition, so read these as *what the constraint looks like*,
not as today's config:

- Jan 2012 blog: one server held **~2.3M concurrent TCP sockets** ("sysctl
  kern.ipc.numopensockets: 2277845", quoted in Reed's 2012 talk) at ~42% CPU idle
  (~58% busy), 24-core Xeon X5675 / 96 GB RAM, FreeBSD 8.2, Erlang R14B03.
- 2012 talk: density trajectory ~200k → 1M → 2M conns/server within ~a year (briefly 2.8M),
  won almost entirely by **contention fixes, most of them patches to the BEAM VM itself**
  (scheduler spin, timer wheel, ets/mnesia partitioning).
- 2014 talk ("That's 'Billion' with a B") at acquisition: **465M monthly users**,
  **19B in / 40B out messages/day** (this is ~2014 scale; the current "100B+/day"
  combined Messenger+WhatsApp figure is non-primary for WhatsApp alone —
  our headline assumption stays ~50B/day,
  and the 2014 numbers validate the arithmetic shape), peak **147M concurrent
  connections**, peaks 230K logins/sec / 342K in / 712K out msgs/sec (peaks, not means)
  on **~550 servers total (~150 chat servers @ ~1M phones each, ~250 MMS/media)**,
  **~10 Erlang engineers doing dev + ops**.

Sanity check against our assumptions: our "100M concurrent online" guess would be ~150
servers at the *demonstrated* ~1M conns/server — consistent with the 2014 fleet of
~150 chat servers at 147M peak connections. The connection layer estimate below holds;
the message-volume estimate above (~50B/day) is ~2.5× the 2014 daily volume at ~4× the
user base — the same order, so the 2014 shape carries (estimate).

## Connection layer (the distinctive estimate)

**~100M concurrent online users** per day. Persistent connections terminate on chat servers
via a gateway layer. If a gateway server holds ~100K–1M TCP connections each (each
long-lived connection with small buffers + no per-message state when idle — mostly idle
sockets):

100M / 1M = 100 gateway servers. This is CHEAP in hardware. The hard parts are not CPU or
NIC — they are (a) stable socket bookkeeping at OS level, (b) re-attach storms when
cell-towers blip, (c) keep-alive tuning to let phone radios sleep. A WhatsApp-scale chat
system is a *network programming* problem more than a "big database" problem.

**Re-connect rate**: phones re-handshake on connectivity blips. At ~100M online and, say,
~10% reconnecting per hour → 10M reconnects/hour ÷ 3,600 s ≈ ~2.8K connects/sec average
(estimate). The average is not the load — reconnects arrive in storms (tower blips, a
whole region returning at once) at many times that rate, and each connect = TLS handshake
+ auth. For scale: the 2014 fleet reported peaks of ~230K logins/sec. Storm capacity, not
steady-state throughput, is the connection layer's real load.

## Throughput

- **Message sends**: 50B/day ÷ 86,400 s ≈ ~0.6M msg/s average (578,000/s), peak ~3× →
  ~1.7–2M msg/s through the relay (estimate). Each is tiny (~hundreds of bytes).
- **Deliveries**: ≥1× per recipient — 1:1: messages delivered 1× per account (plus
  multi-device ~2× per account); groups (a group message lands in K recipients' queues)
  multiply this. Group fanout contributes ~2–5× → net ~2–7× the send rate ⇒ ~1.2–4M
  deliveries/s (at 578K/s: ×2 = 1.16M, ×7 = 4.0M) (estimate).
## Store-and-forward

- **Store-and-forward**: users offline for hours. With ~500M DAU out of ~2B registered users, and
  messages stored ~30 days for offline receivers: at 50B messages/day and, say, 30% going
  to a currently-offline recipient → 50B × 0.30 = 15B messages/day buffered, ~100 bytes
  each → 15e9 × 100 B = 1.5e12 B = **~1.5 TB/day** of envelope-scale records (media bytes
  follow the CDN path; the store holds dedup + reference). Over 30 days: ~45 TB of small
  objects. Write rate alone: 15e9 ÷ 86,400 ≈ ~175K writes/s sustained, before fanout
  peaks. That is not a trivial side store — it is a sharded, per-conversation append log
  holding ~45 TB, and per-message queue indexing is on top of it. (Estimate: the 30%
  offline-buffers share is a guess; scale day volume linearly if it changes.)

## Media

- 20% of 50B = 10B media messages/day × 1 MB avg = 10e9 × 1e6 B = **~10 PB/day of media
  delivered (estimate)** — roughly the same order as YouTube's *ingest* bytes (~6.5 PB/day)
  and ~1% of its delivered bytes (~1.4 EB/day, see [YouTube estimates](../01.youtube/02-scale-estimates.md));
  in absolute terms far too much to ride the chat path either way. Mitigations: content
  addressing (same
  photo forwarded to 100 chats = one upload, 100 links; media is deduped by hash client-side),
  and CDN offload for download. Delivery via CDN (see [YouTube](../01.youtube/03-architecture.md));
  upload/ingest via edge POPs (resumable, chunked).
- Media *keys* used for E2E: because content is E2E encrypted, dedup is by hash — two
  different users uploading the same meme yield distinct ciphertexts, so dedup only works
  within a sender/recipient E2E pair (or when the same plaintext key is reused) — a subtle
  design consequence worth calling out.

## Consistency check

- 50B messages/day ÷ 2B users ≈ 25 sent messages/day/user on the registered basis; over an
  assumed ~1B active users ≈ ~50 sent/user/day — plausible for heavy consumer messaging (estimate).
- ~50B 1:1 deliveries/day (account level) over ~500M DAU ≈ 100 received messages per DAU?
  That's high —
  adjust DAU to ~1B and it's 50/day — still plausible; note this sensitivity explicitly.

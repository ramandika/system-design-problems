# 02 — Scale estimates

Assumptions (order-of-magnitude, labeled; adjust and redo the arithmetic if they change):

- ~2B+ Facebook-registered users with Messenger installed; **~1B monthly actives**
  (publicly reported ~1B, 2016), estimate **~600M DAU**.
- Message volume: the combined Messenger+WhatsApp figure was publicly reported at
  **100B+ messages/day** (self-reported, Jan 2019); assume Messenger's share is
  **~40B/day**, which is an estimate — treat it as the headline driver, not a datapoint.
- Mix: ~30% 1:1-ish, ~70% flowing through groups averaging 5–8 members (messaging apps
  skew to small groups; Discord's guild model is the opposite pole).
- ~60% of messages carry rich content (media, link, sticker, reaction-only); of raw media,
  ~80% images (~300 KB average after client compression), ~20% video (~3 MB average).

## Throughput

- **Sends**: 40B/day ÷ 86,400 ≈ ~460K msgs/s average. Diurnal peak ~2–3×: ~0.9–1.4M msgs/s.
  Each message is small; the bytes are nothing — the row-writes and fanout are the load.
- **Deliveries** (what the fanout multiplies): from the mix above — a 1:1 send delivers to
  1 recipient, a ~6-member group send delivers to the other ~5. Weighted: 0.3×1 + 0.7×5 ≈
  ~4 recipients per send; plus the sender's own copy ≈ **~5 mailbox rows per send**.
  40B sends × ~5 ≈ **~200B mailbox write-entries per day** ≈ ~2.3M writes/s sustained. *This* number, not the send rate, sizes the store
  — it is the Messenger-shaped version of Twitter's fanout question
  (see [deep dives](04-deep-dives.md#1-the-mailbox-model-one-row-per-message-vs-per-message-recipient)).
- **Multi-device delivery**: a delivered message may be pushed to 1–3 devices per account:
  live-push bandwidth ~1–2× the send rate as socket pushes; cheap (few hundred bytes).

## Storage — the part that makes Messenger Messenger

History is retained **long-term on the server** (historical posture; contrast WhatsApp's
relay-and-forget). Corpus model:

- Write entries: ~200B/day of mailbox entries. An entry (envelope, sender, thread id,
  text ~100–500 B, media reference, sequence/read-state metadata) ≈ ~1 KB total on disk.
- Hot-ish retention (~2 years at this posture): 200e9 × 1 KB × 730 ≈ **~150 PB** of
  message rows — before metadata. (Estimate; note how much of the 200B/day assumption
  this inherits — linear sensitivity, state it if you argue about the number.)
- Add per-user inbox/index structures (thread lists, unread bookkeeping, search index
  entries at ~1 per message) — comparable order of magnitude to the entries themselves.

**This is why the store's engine choice is a real engineering case, not a footnote**: a
system whose corpus grows at ~70 PB/year of small rows cannot live on a generic KV store
with 3× replication of hot data. The documented public trajectory — inbox on HBase
(~2013–2016 era), then migrated to MySQL with the RocksDB-derived MyRocks storage engine
(publicly discussed ~2018) — is precisely a fight about bytes-amplification and SSD write
endurance at this scale; it is the same lesson Discord's publicly documented Cassandra→ScyllaDB store migration
taught at trillion-message scale (see [Discord deep dives](../05.discord/04-deep-dives.md)
and [WhatsApp deep dives](../03.whatsapp/04-deep-dives.md#3-offline-queue--ordering-per-conversation)
for the relay-side contrast).

## Media & link previews

- Media uploads: 40B msgs/day × ~15% media-bearing ≈ 6B media/day (estimate) — well under
  the ~60% rich-content share, because links, stickers and reaction-only content carry no
  upload — × ~0.84 MB avg
  (0.8×300 KB + 0.2×3 MB) ≈ **~5 PB/day of bytes ingested**; downloads are several times that → CDN-offloaded,
  near [YouTube](../01.youtube/03-architecture.md) territory but with far shorter content
  lifetime (memes, not catalog content) and dedup by content hash on upload.
- **Link previews**: pasting a URL means the server fetches it once (title, thumbnail,
  favicon). Roughly 10% of messages contain a link → ~4B fetches/day — this is a crawler
  attached to the chat path (rate-limited, cache-first, hostile to open-redirect and
  private-IP fetches; see [deep dives](04-deep-dives.md#4-link-previews-and-media-pipelines-crawling-as-a-side-effect-of-text)).
- Dedupe leverage: the same photo reposted to many chats is one stored object plus
  references, but *only while the server can see plaintext* — under E2EE each (sender,
  thread) pair's ciphertext differs, so dedupe collapses to per-pair or per-upload (a
  consequence WhatsApp already carries: see the media-keys note in
  [WhatsApp estimates](../03.whatsapp/02-scale-estimates.md)).

## Consistency check

- 40B msgs/day ÷ 600M DAU ≈ ~65 messages sent/day/user (received adds the group-fanout
  multiplier on top) — plausible for a
  messaging app with heavy group traffic; halving DAU doubles it, still plausible, which
  is exactly why the DAU assumption is called out as the soft one.
- Inbox reads: ~600M DAU opening the app ~5–10 times/day (estimate), each loading the thread list
  (~20 threads latest, snippets) plus one thread's scrollback → ~3–6B thread-list loads/day
  ≈ **~35–70K inbox-list reads/s** average (estimates), plus search. Read fanout is what per-user indexing buys; it is cheap in compute
  and is paid for with the write amplification computed above.
- Cross-check the fanout arithmetic against a *demonstrated* number: the publicly reported
  **combined Messenger+WhatsApp 100B+/day** figure, fanned out the same way, implies a
  combined fleet write path of hundreds of billions of mailbox entries/day — our 200B
  entries/day for Messenger's ~40B-send share is a consistent slice of that, not a
  contradiction (the combined figure counts *sends*, our 200B counts *rows*).

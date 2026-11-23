# 04 — Deep dives

## 1. Message state machine & optimistic UI

Every outgoing message is a small machine living in the app, not on the server:

```
composing -> queued(outbox) -> sending -> sent(acked, seq assigned)
                                  |            \-> delivered/read (receipts)
                                  v
                               failed -> retrying(n, backoff) -> failed-permanent
```

- `client_msg_id` (client-generated UUID) is the primary key from the moment of the tap.
  The server echoes it on ack — this is exactly [Case 03's at-least-once wire +
  client-side dedup](../03.whatsapp/04-deep-dives.md) contract: because *every* retry
  reuses the same ID, blind retries are safe and duplicates collapse.
- **Optimistic render**: the bubble appears in the same frame as the tap (budget from
  [01](01-requirements.md), < 16 ms). It lives in a separate *optimistic index*, keyed by
  `client_msg_id`, rendered at the tail; on ack it folds into the seq-ordered log at its
  server-assigned seq and leaves the index. State transitions mutate the store, never the
  view directly.

**The race that forces this split**: while your message is in flight (100 ms–1 s+), other
messages arrive. If optimistic bubbles were inserted directly into the ordered log, on ack
your bubble teleports backward above messages the user already saw arrive — a visible
reorder jump. Keeping optimistic entries separate until they have a seq means the log only
ever contains settled ordering: the fold is *ordered insertion at the seq position* — it
can never teleport a bubble above rows the user already saw. What the split does **not**
guarantee is that the splice lands at the tail. In a multi-sender conversation, another
participant can send *after* you (higher seq) but have their message delivered before your
ack arrives; the fold then inserts your message *below* messages already in the log — a
mid-log splice. Tail insertion holds only for messages sent before yours; the guarantee is
ordering, never position.

- **Trade-off**: Option A — optimistic UI with the separate-index reconciliation above;
  wins near-zero perceived send latency and clean merge semantics. Loses: extra state
  (index + fold logic), and tail-position is a lie until acked. Option B —
  send-and-wait-for-ack: zero reconciliation logic, but every send feels 200 ms–s on
  mobile networks — a killer for a chat app. We pick A; the complexity is bounded and
  testable in the engine alone.
- **Failure modes**: (a) *double-send* — retry with a fresh ID after a timeout that was a
  late ack; prevented by the fixed `client_msg_id`. (b) *crash mid-send* — outbox is
  written before render; drain resumes at launch. (c) *permanent failure* — over-cap
  backoff with a user-visible retry affordance; auto-retry forever is battery/traffic
  hostile and hides dead state from the user.

## 2. Rendering a 100k-message list

Full render is dead per [02](02-scale-estimates.md): 100k DOM nodes ≈ 100–200 MB plus O(n)
layout. The design: a **virtualized window** — render only rows intersecting the viewport
plus an overscan margin (~40–60 rows, ~200 KB).

The hard part is **variable heights**. Rows differ wildly (quotes, images, code blocks,
markdown), so scroll position math needs a height per message:

- **Measured-once cache**: each row's height is measured from the real DOM the first time
  it renders and cached by message id ([02](02-scale-estimates.md): ~5–10 MB for 100k
  entries — acceptable, typed-array indexed by seq if not).
- **Estimated prefix sums for unmeasured regions**: scroll math (total height, index for
  offset) is computed over prefix sums of heights; unmeasured heights are estimates
  (average, or per-type template estimates), refined as regions render. Block-based prefix
  sums (fixed-size chunks with cached chunk totals) keep lookups O(chunks) instead of
  O(n) per frame.
- **Scroll anchoring**: when heights *change above* the viewport (an image loads and its
  row grows), compensate `scrollTop` by the delta in the same frame — without it the
  content jumps under the user's finger. The inverse trick on jump-to-message:
  estimate the target offset, seek, re-measure, adjust once.
- **Paging history** is "rows with seq < cursor", so the store only needs seq-sorted
  messages — the same time-sortable-ID pagination trick as
  [Twitter's timeline cursor](../02.twitter/04-deep-dives.md), one level up the stack.

**Trade-off**: Option A — fixed row height (crop/ellipsis): O(1) scroll math, trivially
correct; loses the content (unacceptable — quotes and code must render). Option B — full
variable heights with measure-on-estimate: real layout, at the cost of the height cache,
anchor logic, and estimate-refinement bookkeeping. We pick B because chat content is
inherently variable; the machinery is the price of showing the content at all.

**Failure modes**: (a) *jump under scroll* when images load without reserved aspect-ratio
slots — solved by reserving space from payload metadata (dive 6). (b) *fast-fling
thrashing* — fetch + render work for rows flying past at ~80–170 rows/s in a peak fling
(02); solved by direction-aware prefetch (render ahead in scroll direction, cancel behind). (c) *sticky
bottom* — user is pinned to the newest message; new arrivals must auto-scroll unless the
user scrolled up, handled with a "distance from bottom" flag, not a timer.

## 3. Local persistence & offline queue

Two stores with opposite requirements:

- **Messages**: append-mostly, keyed `(conversation_id, seq)`, batch-written. 100 inbound
  messages = 1 transaction, not 100 (single-transaction cost ~a few ms vs ~100× that,
  estimate, on mobile storage) — the receive-path budget in [01](01-requirements.md)
  depends on this batching.
- **Outbox**: a durable queue keyed by `client_msg_id`, written *before* the optimistic
  render. A tap that doesn't survive a process kill is a lost message — the only
  unforgivable client bug. Drain is by the sync engine only (03 — single writer).

Platform trade-off: **Option A — IndexedDB** (web): free, but weak transactionality at
scale, eviction under storage pressure (browser-dependent, from tens of MB to ~GBs —
labeled: varies by engine and OS), poor on iOS Safari. **Option B — SQLite** (native/wasm):
strong transactions, real indexes, survives storage pressure better; costs embedding (wasm
in the browser). We pick: SQLite wrapper where embeddable, IndexedDB on plain web — the
*outbox-first write order and batched applies are portable across both*, which is what
actually matters for correctness.

**Eviction**: per-conversation LRU over *content*, never over the outbox or read markers;
media cache separately capped ~1 GB (02). Eviction is a latency decision only — the
server log re-materializes anything evicted via backfill. This is the client-side mirror
of Case 03's TTL'd store-and-forward: both sides accept "cold data costs a slower path,"
never "cold data costs correctness."

- **Failure modes**: (a) *two tabs double-draining the outbox* → duplicate sends; fixed
  with a Web Lock/leader-election single-writer (or a SQLite file lock natively) — again
  the single-writer rule. (b) *iOS storage pressure wipes IndexedDB* → stored history gone;
  healed by re-backfill; the *outbox* is the reason we write it first and drain eagerly.
  (c) *schema migration on 100k rows* → migrations must be additive/IndexedDB-versioned;
  a blocking migration is a multi-second cold-start regression.

## 4. Reconnect + backfill: seq gaps with the server

The client's sync protocol rests on the server's conversation-local monotone seq
([Case 03, dive 1](../03.whatsapp/04-deep-dives.md)):

1. Client persists `last_seq` per conversation (a checkpoint, part of the store).
2. On reconnect: one batched `sync(conversations -> last_seq)` — reconnect costs O(1)
   requests, not O(conversations).
3. Server streams events with `seq > last_seq`; client applies in seq order. A
   **gap** (seq jumps) means a missed shard/event → client range-fetches the gap, then
   continues. Live frames arriving while a gap-fill is in flight are buffered by seq and
   applied in order — the same "dedup + order + fill" layer the server side specifies for
   its receivers; the client is simply that protocol's final endpoint.
4. **Truncation**: for a hot channel the gap is unbounded (02: ~144k messages after an
   8 h absence), so the server may answer with a *jump point*: "start from seq N, drop the
   middle." UI renders a divider ("N new messages — jump"). No client can backfill a
   firehose; the protocol needs this escape hatch explicitly.

- **Trade-off**: Option A — fill-then-render (history never shows holes): correct by
  construction, costs the reconnect-time perception of waiting when a gap is being fetched.
  Option B — render-with-holes and fill in place: instant history, but flickering holes
  and re-layout (dive 2 anchor churn). We pick A for bulk backfill (a batched range is
  one request; waiting is bounded) and a B-flavored exception for a *live* gap in the
  hot window, where a bounded hole beats blocking the live stream.
- **Backfill bytes** are cheap in the common case (02: ~8 KB per 12 h away) and only
  dangerous in hot channels — hence truncation, not cleverness.
- **Failure modes**: (a) *stale checkpoint after a server migration* (seq spaces
  reset/re-sharded) → gap-detected-forever; fixed by a version epoch in the checkpoint —
  mismatched epoch = full re-sync. (b) *backfill storm* — one reconnect per blip times
  heavy users; the batched single-request shape and randomized backoff keep the client from
  amplifying the server's reconnect storms (Case 03's concern, mirrored client-side).
  (c) *out-of-order application* when the engine trusts arrival order instead of seq —
  the classic bug the seq-ordered apply exists to kill.

## 5. Unread counts & multi-device state sync

Unread is **derived, not stored**: with server `max_seq` per conversation and a local
`last_read_seq` marker, unread = count of messages with `seq > last_read_seq` and
`author != me`, counted against `max_seq` — not against the local log alone. The log the
client holds is *partial* by design: LRU eviction (dive 3) drops old content and hot-channel
jump markers (dive 4) skip seq ranges outright (02's worst case hides ~144k messages behind
one divider), so messages the client does not hold cannot be counted as rows. Each skipped
range therefore counts as a **pseudo-entry**: a skipped-range marker at start seq S whose
contribution is the server-stated message count ("N new messages — jump"), estimated from
`max_seq − S` when no exact count arrives. Counting only held rows would silently
undercount in exactly the cases dives 3–4 highlight.

**Who computes it?**

- **Option A — server-computed counts**: simple clients, works when offline; but with E2E
  the server cannot distinguish "unread for you" from noise (see Case 03's E2E dive —
  the server sees no content), and per-user × per-conversation counters churn on every
  message (server fanout burden). **Option B — client-computed from markers**: E2E-clean,
  offline-correct, no server write path; costs that the client must hold the read markers,
  the skipped-range markers and the retained log locally, and that @-mentions in muted
  channels need a server-supplied flag on the message (it can't be derived locally for muted content the client doesn't surface).
  We pick B, with the mention flag as the one server-provided annotation.

**Multi-device**: the `last_read_seq` marker is *user state, not device state* — reading on
desktop must clear the phone's badge. So the marker syncs to the server and propagates to
other devices, exactly like a message write, at seq semantics. Muted-channel badge rules
are per-device device-local preference; counts are shared.

- **Trade-off**: *propagating read markers eagerly* (a write per "conversation opened")
  keeps badges honest across devices; *debouncing* saves traffic. We pick eager for the
  active conversation (badge semantics users actually react to) and debounced elsewhere.
- **Failure modes**: (a) *drift until sync* — device B was offline; its badge counts
  stale; converges on next sync — accepted eventual consistency, and the notification
  badge on lock screens is server-computed at push time anyway (approximate by design,
  self-corrects on open). (b) *double-read race* — two devices read simultaneously;
  markers are `max()`-merged (monotone), so no lost update. (c) *mention leaks* — a
  muted-channel badge count from cached local state after a server-side revoke of the
  message; heals on backfill, but the UI must treat mention flags as advisory only.

## 6. Media preview pipeline

Every media message carries, in its payload, ~30 B of **placeholder data** (blurhash/
thumbhash-style: a tiny decoded blur of the image) plus the true aspect ratio. The pipeline:

1. **Paint instantly**: render the placeholder into a small canvas, upscale, blurred —
   sub-millisecond decode, off the main thread. The layout box is reserved **from the
   payload's aspect ratio**, so the full image's arrival causes zero layout shift (this is
   the guarantee dive 2's anchoring depends on).
2. **Sender generates thumbnails at upload time** (small + placeholder hash travel with
   the message). With E2E this is *forced*, not chosen: the server cannot thumbnail
   ciphertext it can't see — the same constraint as Case 03's "no server-side content
   features" [dive](../03.whatsapp/04-deep-dives.md). Cost: the send path grows by
   thumbnail generation (a few hundred ms for video first-frames, estimate) — amortized
   against every recipient skipping a full download just to render a preview.
3. **Fetch with priority**: full asset fetched from CDN by content hash only when in or
   near the viewport, prefetching along scroll direction; concurrent fetches capped and
   cancelled on scroll-away. Same CDN-offload shape as [YouTube's](../01.youtube/03-architecture.md)
   delivery path; dedup by content hash means a forwarded meme is fetched once per device.
4. **Decode at display size**, off the main thread (02: a 12 MP decode is ~48 MB at full
   size vs ~0.5 MB at display size), then LRU cache by content hash (~1 GB cap).

- **Trade-off**: Option A — client-generated thumbnails: E2E-compatible, server-free,
  sender pays once per send; loses send-path latency and requires clients to agree on the
  placeholder format. Option B — server-side thumbnails: cheap sends; breaks under E2E and
  adds a server image pipeline. E2E forces A; the trade-off is real and stated because a
  non-E2E deployment (internal chat) would pick B in a heartbeat.
- **Failure modes**: (a) *placeholder/aspect mismatch with the real image* → visible
  layout pop; mitigated by deriving both from the same decode at send time. (b) *cache
  eviction mid-session* → re-fetch/re-decode churn while scrolling; mitigated by pinning
  the visible window's bitmaps and evicting outside it. (c) *EXIF orientation and video
  keyframes* — metadata the client must own (again: nobody else can, under E2E);
  mishandling shows rotated photos — the classic last-bug-in-the-pipeline.

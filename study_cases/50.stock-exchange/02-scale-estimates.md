# 02 — Scale estimates

Back-of-envelope for one equities venue. All figures rough estimates; adjust the
assumptions and redo the arithmetic. Anchors that are publicly observable/reported:
consolidated US equities volume runs in the ~10B shares/day range, individual venues
list thousands of symbols, and modern venues publicly describe matching in the
tens-of-µs range; order-to-trade ratios on HFT-heavy symbols are publicly discussed as
tens of cancels per execution. Numbers below stay inside that envelope and are labeled
where they are estimates.

## Assumptions (state them, everything else follows)

- One venue with **~8,000 listed symbols** (large US venues publicly list in the
  thousands; est. 8,000 as the design point).
- Our venue carries **~15% of consolidated flow** (est.) → ~10B shares/day × 15% ≈
  **1.5B shares/day**.
- Average execution size **~200 shares** (round-lot dominated, est.) → 1.5B ÷ 200 ≈
  **7.5M executions/day**.
- **Order-to-execution ratio ~25:1** (est.; HFT-heavy symbols publicly discussed as
  tens of events per trade) → lifecycle messages (new/cancel/replace) ≈ 7.5M × 25 ≈
  **~190M inbound msgs/day** → round **200M inbound msgs/day**.
- Outbound market data ≈ **2.5× inbound** (depth updates on every book change, trades,
  status) ≈ **500M MD msgs/day** (est.).
- Intraday shape: US open and close concentrate ~**30× average** in the first/last
  minutes; volatility events (news, meme-style frenzies) produce bursts est.
  **200×** the daily average on aggregate, with the hottest symbol taking up to
  ~2–5% of the whole venue's messages.

## Message rates

- Average inbound: 200M/day ÷ 86,400 s ≈ **~2.3K msgs/s**.
- Open/close burst: 30× ≈ **~70K msgs/s**.
- Volatility burst design point: 200× ≈ **~460K msgs/s** → round **~500K msgs/s
  aggregate** for an hour or two. This is the rate the inbound path, sequencer, and
  journal must survive without dropping a single message *anywhere*.
- Hottest symbol during the burst: 2% of 500K ≈ **10K msgs/s**; meme-day worst case,
  est. 5× that ≈ **50K msgs/s on one symbol**. At 50K msgs/s the matcher gets
  1,000,000 µs/s ÷ 50,000 ≈ **20 µs per message** — on one core, with no lock waits.
  This single number drives the data-structure and threading decisions (deep dives §1–2).
  A 500K msgs/s single-symbol rate would demand 2 µs/msg: beyond single-message
  processing on general-purpose hardware — the design answer there is batching and
  pipelining, not more machines (deep dives §2).

## Order book size (memory)

- Average liquid symbol: **~1–10K resting orders** (est.); hottest symbol mid-frenzy:
  **~100K–500K resting orders** across full depth (est. — penny-wide spreads induce
  laddering; ~$50 price at $0.01 tick ≈ thousands of active levels/side with tens of
  orders each).
- Per-order footprint target ≤ **~128 B** (order ID, firm/account, price, qty, queue
  prev/next, state — est.) → hottest book ≈ 500K × 128 B ≈ **64 MB**.
  Two-thousand hot books at 10K orders ≈ 2,000 × 1.3 MB ≈ **2.6 GB** — the whole
  live book set fits in RAM easily; the engineering problem is *layout*, not capacity
  (deep dives §1).
- Price-level index: $500 max price / $0.01 tick = **50K slots per side** × 8 B
  pointer ≈ **400 KB/side** per symbol → 8,000 symbols × 800 KB ≈ **6.4 GB** of level
  arrays. Trivial, and it buys O(1) level lookup by price (deep dives §1).

## Market data fanout

- MD message ≈ **~64 B** (fixed-width binary, ITCH-style, est.) → burst feed of
  500K msgs/s ≈ 500K × 64 B ≈ **32 MB/s ≈ 260 Mbps per feed copy**.
- **~500 direct-feed subscriber firms** (est.; venues report subscribers in the
  hundreds) — naive per-subscriber TCP unicast: 500 × 260 Mbps ≈ **130 Gbps egress**
  and 500 independent TCP congestion states to maintain. Multicast makes egress
  O(1): one feed ≈ 260 Mbps on one network (deep dives §5).
- Single-symbol quote burst: 10K MD msgs in ~10 ms ≈ **640 KB in 10 ms** — trivial on
  datacenter links, meaningless on a broadband subscriber; this asymmetry is exactly
  why depth updates may be coalesced for slow transport tiers while trades never are
  (deep dives §5).

## Journal storage

- What is journaled: every inbound message and every outbound event, **~120 B/event**
  (est., fixed-width binary) → (200M + 500M) × 120 B ≈ **~84 GB/day** → round
  **~100 GB/day**.
- Regulators require books-and-records retention measured in **years** (long-horizon
  audit, not operational storage) → 100 GB/day × 365 ≈ **~36 TB/year** — archive tier,
  compressed and checksummed; write-once, never edited. Storage is a non-problem;
  *replay determinism* is (deep dives §3).

## Latency budget (the arithmetic that hurts)

p50 target ~50 µs end-to-end (est. aligned with publicly reported tens-of-µs matching):

| Hop | Budget (est.) |
|---|---|
| Gateway parse + risk check | ~2 µs |
| Sequencer stamp + journal append (buffered write, background fsync) | ~3 µs |
| Handoff to matcher (shared-memory queue, no syscall) | ~2 µs |
| **Match (the only "real work")** | **~10–20 µs** |
| Outbound event emit + market data multicast write | ~5 µs |
| **Sum** | **~22–32 µs** — leaves headroom inside 50 µs |

One same-datacenter network RTT (~30–50 µs without tuning) would consume the entire
budget if inserted mid-path — which is why gateway, sequencer, matcher, and feed
writers live on pinned cores of shared hosts, and why "add a microservice" is banned
on this path. The failure mode this table encodes: a single added serialization step
(journal fsync inline, RPC to a risk service, lock contention on book state) blows the
p99 (deep dives §2, §4).

## Consistency check (arithmetic must close)

- Shares sanity: 7.5M executions × 200 shares = 1.5B shares/day ✓ round-trips to the
  venue-share assumption.
- Events sanity: 7.5M executions × 25 lifecycle msgs = ~190M ✓ matches the 200M
  inbound/day line.
- Per-symbol burst: 500K aggregate × 2% = 10K/s; ×5 extreme case = 50K/s → 20 µs/msg ✓
  the same figure quoted in the deep-dive threading argument.
- Journal: 700M events/day × 120 B = 84 GB ≈ ~100 GB/day rounding ✓ consistent with
  ~36 TB/year retention.
- Multicast O(1) vs unicast O(N): 500 × 260 Mbps = 130 Gbps vs one 260 Mbps feed — a
  500× fanout factor, the entire argument for multicast (deep dives §5).

## Where the design bends

- If the design target moves to **options or venues with 100× symbol counts**, the
  book-memory and level-index numbers scale linearly (~640 GB level arrays) — no
  longer trivial; per-priority-group level arrays and tick-aware indexing become
  required (deep dives §1).
- If volatility doubles the burst to **1M msgs/s aggregate**, inbound/sequencer/journal
  all still hold (they are aggregate-parallel), but the hottest symbol's 50–100K msgs/s
  is the wall: no aggregate sharding helps it. The correct answers are per-symbol
  batching and a pipelined matcher, and they must be built *before* the meme day —
  retrofitting determinism-preserving batching under fire is how exchanges end up on
  the news (deep dives §2).

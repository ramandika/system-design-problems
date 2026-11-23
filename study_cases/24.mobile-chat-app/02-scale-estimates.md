# 02 — Scale estimates (per-device arithmetic)

All numbers are **rough estimates** chosen to be internally consistent and consistent
with publicly reported chat-app magnitudes (WhatsApp was publicly reported at roughly
65B messages/day at ~2B users — the widely cited ~100B/day is the combined
Messenger+WhatsApp total, attributed in [case 03's estimates](../03.whatsapp/02-scale-estimates.md)
— which at ~500M DAU is the same order as our 40 sent / 80 received per active device). The unit of scale here is the device, not the server.

## Baseline population and message rates

- DAU: **500M devices** (estimate).
- Messages sent per device: **40/day**, received: **~80/day** (2× fanout from 1:1 and
  small groups; estimate).
- Global send rate: 500M × 40 / 86,400 s ≈ **231K msg/s average**; peak-hour factor
  ~3× → **~700K msg/s peak** (server-side; the client sees ~1 msg/36 min average,
  ~1 msg/12 min at peak per device: 86,400 s ÷ 40 sent ≈ 2,160 s ≈ 36 min, and
  ÷ ~3 peak factor ≈ 12 min — estimate).

## Per-device traffic (the client-visible math)

Text message envelope on wire: ~200 B payload + ~200 B headers/TLS overhead ≈
**400 B**. Sent traffic: 40 × 400 B = 16 KB/day. Received: 80 × 400 B = 32 KB/day.
Text is noise; the real traffic is:

- **Keep-alives / receipt pings while connected.** Assume the socket lives 2 h/day in
  foreground use with a 4-min keepalive interval: 30 keepalives/day × ~120 B ≈
  **3.6 KB/device/day**. Across 500M devices: 500M × 3.6 KB ≈ **1.8 TB/day upstream
  of keepalives** — small per device, but it is *wakeup-driven* traffic, and battery
  cost scales with wake count, not bytes (see 04 §4).
- **Push notifications while backgrounded** (mostly "you have a message" with no
  payload decrypt): estimate 20/day × 100 B ≈ 2 KB/device/day ≈ **1 TB/day** fleet-wide.

## Media upload math (estimates)

Assume 5% of sent messages are photos and 0.5% are videos (estimate; media is a
minority of messages but ~100% of the traffic — a lower per-device media share than
case 03's ~20% server-side assumption, also an estimate; the conclusions below hold at
either share):

- Photos/day: 500M × 40 × 5% = **1.0B photos/day**, avg 2 MB → 1.0e9 × 2e6 B =
  **2 PB/day uploaded** fleet-wide.
- Videos/day: 500M × 40 × 0.5% = 100M/day, avg 15 MB → 100e6 × 15e6 B =
  **1.5 PB/day uploaded**.
- Per-device peak burst: a 40 MB video on a 10 Mbps uplink takes 40×8/10 ≈ **32 s of
  full radio-on time** — radio power is why resumable chunks matter (04 §2).
- Upload retry waste: if 15% of uploads see a network break (estimate on cellular) and
  non-resumable upload re-sends from zero, average wasted bytes = 0.15 × (fraction
  completed ≈ 0.5) × 3.5 PB (photos 2 PB + videos 1.5 PB) ≈ **~260 TB/day of pure
  re-send** (photos alone: 0.15 × 0.5 × 2 PB ≈ 150 TB). Resumability is not polish;
  it is a ~7.5% upload-bandwidth cut (0.15 × 0.5; ~150–260 TB against the 3.5 PB/day
  uploaded, estimate).

## Local storage per device (estimate)

- Text DB: 5 years × 120 msgs/day (40 sent + 80 received — F2 requires the received
  history to persist too) × 365 ≈ 219K messages × ~500 B (payload + metadata +
  index) ≈ **110 MB** — slightly above but the same order as the budgeted
  **~100 MB**, plus WAL/index double-count and receipts/typing churn in the same DB. This is why the migration chain has to run fast and safely on
  a device that skipped 5 app versions (04 §3).
- Media cache: hard cap **~2 GB** default, LRU-evicted; originals re-fetchable from
  blob store (content-addressed by hash, so re-fetch is free of dedup ambiguity).
- Key material: identity key + signed prekey + ~100 one-time prekeys + per-peer
  session state (~1–2 KB per active conversation × ~200 conversations ≈ 0.4 MB).
  Bytes are trivial; *where it is stored* is the design question (04 §5).

## Reconnect math (estimate)

- Network transitions per device: WiFi↔cell, NAT rebinding, doze cycles, elevators —
  estimate **20 reconnects/device/day**.
- Fleet-wide: 500M × 20 = 10B reconnect events/day ≈ **116K reconnects/s average**.
  The server is given, but the client's contribution to its cost is exactly this
  number: jittered backoff and resumption (TLS session tickets, sync cursors) are
  what keeps a regional event (base-station outage restoring in one instant) from
  being a 100× instantaneous spike. Same storm shape as case 03 §2.

## Cold-start budget (estimate, mid-range device, N2 = 2 s)

| Phase | Budget |
|---|---|
| Process init + first frame (OS) | 300 ms |
| Local DB open (+ pending migrations) | 150 ms steady / 2–10 s if a migration runs |
| Key store unlock (session resumption via OS keystore) | 50 ms |
| Sync check with server (cursor compare, not full fetch) | 300 ms |
| First conversation render from local DB | 200 ms |
| Buffer | 1,000 ms |

The binding constraint: cold start must **not** include fetching conversation list
content — everything on the first screen comes from the local DB (04 §6). A server
round-trip in the critical path is the design failure, not a slow phone.

## What to notice

Every derived number above follows from one stated assumption chain; recompute any
line and the rest hold. The surprising shape: bytes are cheap (KB/day for text),
*events* are expensive (wakeups, reconnects, migrations), and the whole client design
is an event-budget problem wearing a bandwidth costume.

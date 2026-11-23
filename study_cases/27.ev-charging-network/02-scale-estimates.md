# 02 — Scale estimates

All numbers are **estimates** built from stated assumptions (arithmetic shown). Design
target: a large European CPO with **100,000 public charge points** — the EU Alternative
Fuels Observatory (EAFO) reports 1,061,890 public charge points across the EU27 at
end-2025 (rising toward ~1.2M by late 2026; wider Europe is higher), so this is a ~10%
slice of the EU27. **Deliberately futuristic**: no European CPO operates anywhere near
100K — Europe's largest fast-charging networks are IONITY (~4,400 charge points) and
Fastned (~2,700 charge spots) as of 2024–2026, i.e. this fleet is 20–40x the largest
operator, and we use it to force the hard problems at full scale.

## The fleet

- **15,000 DC fast chargers** (50–350 kW, one or two connectors) at motorway hubs and urban
  fast sites; **85,000 AC chargers** (11–22 kW) on streets and destination parking.
- Every charger: embedded Linux, 4G/Ethernet, persistent OCPP 2.0.1 WebSocket to the
  backend. Target online ratio ≥ 95% (charger-side failures dominate, not ours).

## Sessions and energy (the money-shaped load)

- DC: ~5 sessions/charger/day × 15,000 = **75,000 DC sessions/day** (estimate; a busy
  motorway charger does many more, an urban one fewer).
- AC: ~1.5 sessions/charger/day × 85,000 = **127,500 AC sessions/day**.
- Total ≈ **200K sessions/day** ≈ 2.3 sessions/s average, ≈ 2× midday peak ≈ 5/s
  (Little's law: the same 2× daytime skew assumed for concurrency below).
- Energy: DC 35 kWh avg × 75K = **2.6 GWh/day**; AC 7 kWh × 127.5K = 0.9 GWh/day →
  **~3.5 GWh/day** of metered energy.
- Revenue (rough, at typical public tariffs: DC ~€0.55/kWh, AC ~€0.35/kWh, plus session
  fees): 2.6 GWh × €0.55 ≈ €1.4M + 0.9 GWh × €0.35 ≈ €0.3M → **~€1.7M/day** at the meter,
  before energy costs, grid fees, and partner splits. The billing pipeline is structurally
  low-volume but every record carries real money in both directions.

## Concurrent sessions and OCPP traffic

- Concurrent sessions ≈ sessions/day × duration ÷ 24 h: DC 75K × 0.5 h ≈ 1,600; AC
  127.5K × 1 h ≈ 5,300 → **~7,000 average, ~14,000 at midday peak** (2× daytime skew).
- Idle-charger floor (charger-initiated heartbeats, 1/min/charger, 95K online):
  95K ÷ 60s ≈ **1.6K msg/s**.
- Session telemetry (meter values every 10–60 s while charging; assume ~15 s effective):
  7K average concurrency × 1/15s ≈ 0.5K msg/s average; **1.0K msg/s** at the 14K peak.
- Transaction/status events: 200K sessions × ~8 events + 100K chargers × ~10 status
  transitions/day ≈ 2.6M/day ≈ 30 msg/s average — event-driven, spiky.
- **OCPP ingest: ~2.1K msg/s average (1.6K heartbeats + 0.5K meter + 0.03K events).
  Heartbeats scale with charger count, not session load, so the session-shaped peak is
  ~1.6K + 1.0K (meter at doubled concurrency) + ~0.1K (event bursts) ≈ ~2.7K msg/s core.
  Budget the gateway and bus for ~5–7K msg/s**: the core plus a stated ~2× burst headroom
  covering event bursts (a few × of the 30/s average is still small) and replay drain of
  the ~0.5–1.5M queued-message burst below. **~180M messages/day**, ~0.3–1 KB each →
  **~100 GB/day raw ingest** — of which heartbeats (~75% of message count) are pure
  liveness and are never stored.

## Connections (the gateway-shaped load)

- ~95K persistent WebSocket connections (95% online) — compare Discord's concurrent
  gateway connections in [05.discord](../05.discord/README.md), but TLS-terminating a
  4G modem that may half-open at any moment.
- 4G NAT timeouts of 2–5 min force keepalive pings ≥ every 60 s; each heartbeat is a
  server→charger round trip.
- **Reconnect storm** (the design point): a 10-min carrier outage affects ~50K chargers;
  on restoration all reconnect within ~2 min, softened to 0–60 s by mandated reconnect
  jitter → **~400 new WebSocket/s sustained for 2 min**, each charger replaying ~10–30
  buffered device events → a **0.5–1.5M message drain burst** ≈ 4–12K msg/s over the
  window, i.e. 0.6–2.5× the ~5–7K peak budget (up to ~4.5× the 2.7K core peak) *on top
  of* steady state — the shaped-drain budget the
  gateway and bus must be sized for ([04-deep-dives](04-deep-dives.md) §1), not bandwidth.
  Tiny by the standards of [26.google-maps](../26.google-maps/02-scale-estimates.md) — this
  system's difficulty is *correctness over flaky links*, not volume.

## Storage

- Stored business events (transactions, status transitions, meter samples — the ~75%
  heartbeat share of message count is dropped at ingest): ~40–60M events/day after dedup
  ≈ **3–10 GB/day → 1–4 TB/year**. Session meter samples alone: ~200K/day × ~120–240
  samples (15 s effective sampling × 0.5–1 h sessions) × ~100 B ≈ **2.5–5 GB/day**.
  Trivial volume; retention is the only constraint (18 months raw meter data
  for disputes, 10+ years CDRs for tax law) → hot/warm/cold tiering, not a storage-engine
  problem ([07.blob-store](../07.blob-store/02-scale-estimates.md) is where volume actually
  lives; firmware artifacts below are the closest thing here).
- Firmware artifacts: ~10–150 MB per package × ~5 hardware platforms × several versions
  live → < 100 GB total, served from [09.cdn](../09.cdn/README.md)-style object storage.

## External fanout (OCPI roaming — the fanout on *companies*)

- Assume ~60% of sessions arrive via partner eMSPs (roaming is the norm, not the
  exception, on European networks — estimate):
  **~120K roaming authorization decisions/day** — but partner token lists are pushed to us
  for caching, so the large majority resolve from the local cache; only cache misses send
  our `Authorize` request *outbound* to the partner's endpoint, so outbound partner-API
  calls are a small fraction of 120K,
  **~120K CDRs/day** outbound to partners.
- Occupancy events: ~1M status transitions/day; each partner receives only *its own*
  stations, so pushed volume is 1M × the average number of partners a station is relevant
  to (assume 2–10) → **~2–10M pushed events/day**, bounded above by the no-filter ×30
  worst case (**≤30M/day**). The push volume is Twitter's celebrity-fanout shape
  ([02.twitter](../02.twitter/README.md)) with per-recipient API rate limits and outage
  behavior we do not control.
- OCPI explicitly tells push senders **not to queue** failed pushes (OCPI 2.2.1 §4.4,
  "Offline behaviour": SHOULD NOT — receiver-side pull is the recovery model): a partner API outage
  does *not* accumulate a day of queue on our side; the partner resynchronizes by pulling
  (GET) from our endpoints after reconnect. We therefore keep retention-capable pull
  endpoints per module, and bound per-partner push queues anyway because some partner
  implementations retry despite the SHOULD NOT.

## What this scales like

Bandwidth and even request volume are *small*. The hard numbers are:

0. **Driver-app reads**: map search and live session telemetry for a fraction of ~14K
   concurrent sessions with the app open ≈ ~1–2K concurrent app sessions, low QPS, small
   in every dimension — listed because it is the "normal" web-shaped load against which
   the four unusual loads below must not be confused.
1. **~95K long-lived connections** that reconnect en masse (deep dive 1),
2. **~200K money-bearing records/day** that must survive at-least-once delivery and
   offline replay (deep dive 2 and 3),
3. **~30 external companies** with individual rate limits, schemas, and SLAs (deep dive 5),
4. **100K embedded devices across 5 hardware generations** receiving firmware we cannot
   take back (deep dive 6).

If a design meeting optimizes QPS here, it is optimizing the wrong axis.

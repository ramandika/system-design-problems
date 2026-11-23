# 02 — Scale estimates (back-of-envelope)

All numbers are rough estimates from stated assumptions, scaled to be in the documented
neighborhood of major commercial CDNs (Akamai, Cloudflare, Fastly, CloudFront — publicly
described as hundreds of POPs, fleets peaking in the tens of millions of requests/second
on ordinary days, and multi-Tbps volumetric absorption capacity, vendor-reported).

## Traffic: requests and bytes

Assumptions:

- ~800M daily end users across all customer properties (est.), each issuing on average
  ~2,500 CDN-fronted requests/day (a typical page is tens of assets; APIs and long-lived
  apps poll more).

```
requests/day  = 800e6 × 2500                   = 2.0e12 req/day
avg requests  = 2.0e12 / 86,400                ≈ 23M req/s average
peak (3× diurnal, regional stacking)           ≈ 70M req/s
```

- Average response size: mix of small HTML/JSON (2–20 KB), images/JS/CSS (50–500 KB),
  larger software/video-segment downloads. Weighted average ≈ 50 KB over all requests (est.).

```
egress bandwidth = 23e6 × 50e3 B/s             = 1.15e12 B/s ≈ 9.2 Tbps average
peak egress      = 3×                           ≈ 28 Tbps
egress/day       = 9.2 Tbps × 86,400 s        ≈ 100 PB/day
```

Sanity check against a known anchor: publicly reported major-CDN egress is in the range of
tens of Tbps and tens of exabytes per month / hundreds of PB per month — 100 PB/day ≈
3 EB/month, on the high end of plausibility for a top-tier CDN. Estimate accepted.

- Client→edge : edge→origin ratio. Design hit rate ≥ 95% of requests. Misses skew heavy
  (cold large objects; hot small objects are always cached), so assume misses are 15% of
  *bytes*:

```
origin fetch ingress ≈ 15% × 100 PB/day      ≈ 15 PB/day   (write to edge cache: full 15)
origin egress ≈ same 15 PB + dynamic non-cached pass-through ≈ 20 PB/day ingress upstream
```

## Per-POP sizing

Assume ~300 edge POPs in ~120 metros (publicly reported range ~200–400 POPs for major
CDNs, vendor-reported). POPs are sized by traffic, not equal:

```
avg req/s per POP = 23e6 / 300                ≈ 77K req/s   (median POP much less,
peak req/s per POP ≈ 230–700K req/s           top core-POPs 10–100× median)
```

- Cache storage per large POP. Working set = content requested in the TTL window × retention.
  Rule of thumb used: a POP holding ~10% of the byte-working-set gets ~97% hit rate given
  power-law popularity (same skew shape as [YouTube, deep dive 3](../01.youtube/04-deep-dives.md)).
  Take large-POP cache = ~200 TB NVMe + ~2 TB DRAM tier (hottest objects and metadata);
  median POP ~10–30 TB.

```
fleet cache footprint ≈ 300 POPs × ~70 TB avg ≈ 20 PB of (heavily duplicated) cache
```

Note the duplication: the same 15 PB/day of origin fill appears at many POPs. Tiering with
regional shields (deep dive 3) is what keeps duplication bounded rather than proportional
to POP count.

## Connection and TLS load

```
concurrent TCP conns ≈ conn-arrival rate × conn lifetime
  conn arrival ≈ 23e6 req/s ÷ ~30 req per keep-alive conn ≈ 0.77M conn/s
  × ~10–20 s keep-alive lifetime ≈ ~8–15M concurrent conns fleet-wide (est.);
  traffic is heavily skewed → median POP tens of K conns, design point
  ~100K–1M conns per large POP (est.)
TLS handshakes ≈ new-visitor share ≈ 5–10% of requests ≈ 1–2M hs/s fleet (est.)
  per-server with ECDSA: ~5–10K hs/s/core-class (est.) → handshake is cheap per request
  but resumption (session tickets, PSK in QUIC) is what keeps it at ~5–10% share.
```

## Ingest of logs (the sleeper)

Every request logs 1–2 KB of attributes (URL, cache status, timings, billing fields):

```
log data/day = 2e12 × 1.5e3 B = 3e15 B ≈ 3 PB/day raw per-fleet before compression
             → compression + sampling bring analytics to ~1 PB/day (est.)
```

3 PB/day at 23M req/s means 3e15 / 86,400 ≈ 34.7 GB/s of log stream average — a firehose,
not an afterthought; the design problem is absorbing that volume *and* not adding per-request
latency (write to local disk buffers, ship compressed async) and not losing billing accuracy
on POP failure.

## Storage of configuration

Trivial volume: ~1M customer domains × config (~50 KB rules) ≈ 50 GB, but must be
consistent at every POP within seconds of change — it is a distribution-latency problem,
not a storage problem.

## DDoS exposure vs. daily load

```
sustained clean traffic peak     ≈ 28 Tbps, 70M req/s
attack headroom required         ≥ 2–3× clean peak at the edge tier
publicly reported mitigations: ~200M+ req/s L7 floods, tens-of-Tbps volumetric floods
(up to ~70 Tbps single-event; vendor-reported, 2023–2025 era) → design POPs and fleet
headroom for this class (2–3× clean peak = 56–84 Tbps covers it).
```

## What these numbers drive

1. 23M req/s × <10 ms of edge logic → every per-request byte of work matters; cache lookup
   must be a lock-free read path (deep dive 5).
2. 100 PB/day egress — bandwidth is the cost floor; hit-rate percent points are the P&L
   (deep dive 2).
3. 20 PB duplicated cache → tiering/shielding (deep dive 3).
4. 2e12 requests/day hitting the invalidation system's blast radius → purge design is a
   propagation-time problem (deep dive 4).
5. 2e12 log lines/day → billing pipeline shapes what the data path records, not vice versa.

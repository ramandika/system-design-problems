# 02 — Scale estimates

Assumptions (adjust and redo arithmetic if they change):

- Total payment volume **~$1.4T/year** (design point; Stripe publicly reported ~$1.4T processed
  in FY2024) → $1.4T ÷ 365 ≈ **~$3.8B/day**.
- Mean payment ~$25 (est.) → 3.8B ÷ 25 ≈ **~150M payment attempts/day**.
- Auth success rate ~90% (est.) — declines are full-cost requests too (they traverse to the
  issuer), so request count *is* attempt count.
- ~12% of write requests carry an idempotent retry attempt (network flakiness, mobile) —
  mostly harmless, occasionally concurrent (deep dives §1).
- Refunds ≈ 5% of charges; disputes ≈ 0.1%; payout runs ≈ daily per active merchant.

## Request rates

- Write (auth/capture/refund/payout APIs): 150M/day ÷ 86,400 s ≈ **~1.7K QPS average**;
  weekday peak (lunch-hour concentration ~3–5×) ≈ **~5–9K QPS**; annual peak days (Black
  Friday–shaped, ~10× average) ≈ **~17K QPS**.
- Reads (dashboard, lists, exports): ~5–10× writes ≈ **~9–17K QPS average** — cacheable and
  tolerant; the interesting design constraints are all on the write path.
- **This is a tiny QPS by the standards of this corpus** (cf. ~28K QPS just for typeahead
  suggestions in [45.typeahead](../45.typeahead/README.md)) — the difficulty is correctness
  under exactly the concurrency these numbers still admit (a peak second has ~17K attempts,
  some of them the *same* payment).

## Webhooks

- ~3 events per payment (created/succeeded/updated) × ~2 live webhook endpoints per merchant
  (est.) ≈ 150M × 6 ≈ **900M delivery attempts/day** ≈ **~10K QPS**, with retry amplification
  against unreachable endpoints (cap per-endpoint retry, honor backoff — same discipline as
  [03.whatsapp](../03.whatsapp/README.md) §store-and-forward).

## Ledger volume (the storage that must live forever)

- Rows per payment: 1 charge row + ~2–4 ledger postings (fees, capture, refund legs, payout
  legs amortized) ≈ ~150M × 4 ≈ **600M ledger rows/day** ≈ **~7K row-writes/s average, ~30K/s
  peak**.
- ~500 B/posting → **~300 GB/day** raw; a year ≈ 220B rows ≈ **~110 TB/year** raw; ~2–3×
  with indexes and replication → order **~2–3 PB per decade** — sharded ledger by account, no
  exotic storage needed; immutability is the requirement, not exotic scale.

## State machine traffic

- Authorization holds live 5–7 days (card-network auth lifetime, ballpark); captures on our
  platform: ~70% of auths captured within 7 days, ~25% explicit void/expiry, ~5% long-tail
  (est.). Expiry sweeps and reversal jobs therefore churn the void/expiry + long-tail share —
  ~30% of daily auths ≈ ~45M auth-lifecycle rows/day, ~300M/week (est.).

## Consistency check

- Money sanity: 1.7K QPS × $25 ≈ $42.5K/s ≈ $3.7B/day ✓ round-trips to the total-volume
  assumption.
- The numbers deliberately do *not* create systems pressure (no fanout, no petabytes, no
  GPU fleets). Every estimate above is small infrastructure with absolute accounting: that
  inversion — *small scale, absolute stakes* — is the defining shape of this case, and the
  opposite of [40.tinyurl](../40.tinyurl/README.md) (huge read scale, no accounting at all).
- Webhook × endpoints: 900M/day ÷ 150M payments ≈ 6 deliveries/payment — matches 3 events ×
  2 endpoints ✓.

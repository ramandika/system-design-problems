# 06 — Apple App Store

## What the system does

The App Store is four systems wearing one trench coat:

1. **A catalog + search engine** over ~2M apps (publicly reported ~1.8–2M listings),
   sliced into ~175 regional storefronts with per-region availability, pricing tiers,
   and rankings.
2. **A binary distribution machine**: stores app packages, cuts them into
   device-optimized variants (app thinning), generates delta patches between versions,
   signs everything, and pushes tens of PB/day of payloads over CDN edges to the
   ~2B-device active base.
3. **A human-in-the-loop review pipeline**: every submission of every app version on
   earth passes through an approval state machine gate that ends in a cryptographic
   signing/provisioning step. Nothing ships unsigned.
4. **A financial transaction ledger**: in-app purchases, paid apps, subscriptions,
   refunds — with entitlements that must render as signed receipts the device can
   validate offline, since the purchase must survive airplane mode.

## Why it's hard

- **The unit of distribution is huge and changes constantly.** A video site serves bytes
  that mostly don't change; an app store delivers 100MB+ payloads where every app ships
  a new version every few weeks, so delta computation and cache invalidation are the core
  problem, not an optimization.
- **Trust is a hard dependency.** The review gate is not a feature — it is the platform's
  security root. The signing pipeline, review state machine, and distribution path are one
  trust chain: a bypass of any tier leaks arbitrary code onto ~2B devices
  (publicly reported active-device figure).
- **Money with offline reads.** The ledger is server-authoritative, but the *proof* of
  entitlement must live on-device and validate without a network. That splits the system
  into a strongly consistent write core and an eventually consistent, cryptographically
  verifiable read path.
- **Two very different traffic shapes coexist**: spiky, huge-bandwidth binary pulls
  (a hit game update launch) and a torrent of tiny high-QPS calls (update check-ins,
  search, chart queries).

## Reading order

1. `01-requirements.md` — functional scope, NFRs, and what is explicitly out of scope.
2. `02-scale-estimates.md` — device base, delivery events, bandwidth, review throughput,
   transaction rate. All arithmetic shown; numbers are estimates unless labeled otherwise.
3. `03-architecture.md` — component diagram and the write/run data flows.
4. `04-deep-dives.md` — the core: binary storage + deltas, search freshness, the review
   state machine, the purchase ledger, and multi-region consistency.

## Patterns this case drills

Delta/dedup binary storage (vs the immutable-files pattern in the blob-store case),
edge delivery economics (CDN case), freshness vs complexity in ranking, idempotent
payment pipelines, and "push consistency to the signed client" — a trick WhatsApp and
the App Store share for opposite reasons.

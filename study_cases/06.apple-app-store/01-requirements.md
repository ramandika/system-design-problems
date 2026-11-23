# 01 — Requirements

## Functional requirements

### Developer side (App Store Connect)

- **F1 — Submit a binary + metadata**: upload an app package (ipa/pkg), set store
  metadata (description, screenshots, price tier, availability per storefront,
  age rating questionnaire). Uploads are multi-GB and must be resumable.
- **F2 — Review workflow**: submit a version for review; see state; respond to
  rejections with either metadata-only revisions or a new binary; request expedited
  review; manage release modes (automatic, manual, **phased release** — percentage
  rollout to users over days with automatic pause on crash-regression).
- **F3 — Version management**: a released version stays downloadable until the
  developer removes it; new versions replace it for new installs but updates follow
  the user's opt-in (auto-update or manual). N released versions must remain
  retrievable (rollback/audit, and users stuck on old OS versions).
- **F4 — Pricing & availability changes**: change price tier or storefront
  availability without a new binary.

### Client side (hundreds of millions of devices)

- **F5 — Browse/search catalog**: featured pages, genre charts, search with
  suggestions, per-storefront content. Search must rank by relevance *and* platform
  quality signals.
- **F6 — Install/update**: fetch the correct per-device variant (thinned package for
  device model + OS), verify the signature chain locally, install. Updates run as
  background auto-updates (default) or manual. A failed delta patch must silently
  fall back to a full download.
- **F7 — In-app purchases**: buy products/subscriptions, restore purchases on new
  devices, download hosted In-App Purchase content, family sharing. Purchases must
  work under flaky networks and the *result* must validate offline.
- **F8 — Receipts & entitlements**: every device holds signed receipts for its
  purchased apps and subscriptions; the OS and apps validate them locally and refresh
  against a receipt service.
- **F9 — Charts/analytics for surfacing**: top free/paid/grossing charts per genre
  and storefront computed from real transaction/impression events.

### Platform side

- **F10 — Sign & provision**: approved binaries are (re)signed with the platform's
  distribution certificate; provisioning profiles embed entitlements (push, iCloud,
  IAP capability IDs). Signing is the release gate — the un-signed and the
  not-yet-approved binary are indistinguishable to clients: neither exists publicly.
- **F11 — Developer payouts hooks**: transaction records feed financial reporting
  (payout computation itself is out of scope here).

## Non-functional requirements

- **NFR1 — Trust above all**: it must be computationally infeasible to install
  non-approved code through store endpoints. Every artifact is signature-verified
  at two points: pipeline-in (what is this, whose is it) and device-in (was this
  signed by the approved pipeline).
- **NFR2 — Delivery scale**: ~75 PB/day-class client-facing egress on estimate
  (see 02 — the dominant NFR by far); served via CDN edges; origin egress reduced
  by high cache-hit ratios.
- **NFR3 — Catalog latency**: browse/search p99 in the low hundreds of ms on the
  client-perceived path; the catalog is small (2M rows) so this is about abuse
  resistance and per-region slicing, not index size.
- **NFR4 — Purchase correctness**: a purchase is a money event. It must be exactly-once
  from the user's perspective under any retry, timeout, or crash — client dedup +
  server idempotency keys (see deep dive 4).
- **NFR5 — Review throughput vs quality**: publicly reported, the large majority of
  submissions get a result within ~24h. The state machine must sustain that with
  human reviewers in the loop and support re-review loops without losing audit trail.
- **NFR6 — Offline-tolerant entitlements**: receipt validation on device with no
  network for the lifetime of the receipt; revalidation opportunistic, not required.
- **NFR7 — Multi-region**: ~175 storefronts; catalog changes propagate worldwide in
  minutes, not synchronously (trade-off discussed in deep dive 5).

## Scope decisions

### In scope

Catalog + search, binary storage/delta/thinning, review + signing pipeline, IAP
transaction pipeline + receipts, top-chart event analytics, multi-region consistency.

### Explicitly OUT of scope

- **Refund adjudication, disputes, chargebacks** — mentioned as ledger events only.
- **Payout/settlement accounting** (bank transfers, tax per storefront).
- **The App Store advertising auction** (search-ad placement) — a separate ad system.
- **TestFlight/beta distribution** — same delivery machinery, different trust gate.
- **Review content intelligence** (malware scanning internals, static analysis) —
  we treat review as a human-workflow + verdict problem, not a scanner design.
- **Client OS-side install machinery** (filesystem staging, space management).
- **Developer account fraud/identity systems.**

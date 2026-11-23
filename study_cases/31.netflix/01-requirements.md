# 31.01 — Requirements

## Functional

**Content lifecycle (operators + licensors, not end users)**

- F1: Ingest masters delivered by studios/licensors as mezzanine files (multi-GB, 4K,
  uncompressed audio tracks) over fat pipes/physical drop, in batches against release dates.
- F2: Validate, normalize (frame rate, color space, loudness to broadcast standards), then
  transcode into the full playback ladder — multiple codecs (e.g. H.264/HEVC/AV1),
  resolutions from sub-480p to 4K HDR, per-title bitrate optimization.
- F3: Localize: dub audio into tens of languages per title, subtitles into more; artwork
  and synopsis per region (and per user, deep dive 4).
- F4: Quality-control pass before publish; a broken asset must never reach a member's
  device — there is no "edit and republish in 5 min" affordance for a season premiere.
- F5: Package and replicate: one canonical playable form, pushed to origin storage, then
  prepositioned to CDN appliances ahead of viewing peaks.

**Playback (members)**

- F6: Browse a personalized catalog: rows (Trending, Continue Watching, My List, genre
  rows), search, per-profile state (progress, ratings, age filters).
- F7: Stream a title with adaptive bitrate, full scrubbing, resume exactly where they
  left off — the "can I watch episode 6 minute 41 from my phone" state must be durable.
- F8: Enforce entitlements: active subscription, plan limits (number of concurrent
  streams, resolution ceiling e.g. 4K only on premium, ad-supported tier cadence).
- F9: Secure playback: multi-DRM license issuance (Widevine / PlayReady / FairPlay /
  hardware-backed L1), per-device registration, no offline key escape.
- F10: Multi-language playback selection, per-language audio/subtitle tracks chosen at
  start without re-buffering the container.

**Operational**

- F11: Regional isolation of the control plane: a broken region degrades alone; playback
  keeps working via cached/impersonal defaults.
- F12: Observability of playback quality per session (startup time, rebuffer ratio, exit
  reason) feeding both on-call and the experimentation system.

## Non-functional

- N1: **Startup latency**: UI render in low seconds; playback start target well under 2 s
  on decent links — this dominates perceived quality more than almost anything else.
- N2: **Rebuffer ratio** held near zero in the steady state; stall is the #1 churn signal.
- N3: **Availability graded by blast radius**: playback > browse > personalization depth.
  If personalization is down, play *Something* — a member who opens the app and gets an
  empty screen is a cancellation event; a member who gets a generic row set is annoyed.
- N4: **Egress cost is the unit economics**: ~1 EB/day-class delivery means every ladder
  rung, codec, and CDN placement is an economic decision (see 02).
- N5: **Security over convenience** at the license/entitlement boundary, but *convenience
  over security* inside the member household (they own the devices).
- N6: **Predictability beats peaks** on the delivery path: capacity is added via off-peak
  fill, not on-demand scaling (deep dive 2).
- N7: Durability of masters is absolute; durability of playback telemetry is best-effort.
- N8: Release-window deadlines: transcoding and localization for a scheduled premiere are
  SLA work orders with dates, not a queue that drains eventually.

## Scope decisions

**In scope**: ingest/ladder, packaging/replication, own-CDN prepositioning vs on-demand,
ABR + playback QoE telemetry, recommendation/Personalization surface, entitlements/DRM,
multi-region + chaos-engineering resilience.

**Out of scope** (with where to go instead):

- The client player mechanics — [Case 23](../23.frontend-video-streaming/README.md)
  (ABR algorithm, buffer model, browser schedulers).
- General CDN internals (anycast, POP design, TLS, purge, DDoS) —
  [Case 09](../09.cdn/README.md).
- The ranking funnel's internals (retrieval, feature stores, training loop) —
  [Case 15](../15.ml-ranking-system/README.md); here we cover only the *surface* that
  ranks must personalize (rows, artwork) over a small catalog.
- UGC upload, moderation, creator economy — not the product; contrast with
  [Case 01](../01.youtube/README.md).
- Live events (sports, one-off premieres streamed live). Netflix has publicly run live
  events, but they are a different system (edge origin + egress scaling under a known
  start time); this case covers the VOD steady state.
- Content production systems (studios, edit chains, physical deliverables) — ends at the
  mezzanine we ingest.
- Billing/payment pipelines — treated as "some system that flips an entitlement flag".

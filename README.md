# System Design Knowledge Base

A growing collection of system design knowledge, built case by case. Each study case
dissects one real product (YouTube, Twitter, WhatsApp...) into requirements, back-of-envelope
math, architecture, and the deep dives where the interesting decisions live.

## Structure

```
README.md                          <- this file
CLAUDE.md                          <- conventions for writing new cases
study_cases/
  README.md                        <- index of all cases + common patterns
  01.youtube/                      <- products (01-09)
    README.md                      <- case overview, reading order
    01-requirements.md             <- functional + non-functional requirements
    02-scale-estimates.md          <- back-of-envelope math
    03-architecture.md             <- high-level design, components, data flow
    04-deep-dives.md               <- the hard parts, trade-offs, failure modes
  02.twitter/ ... 46.google-docs   <- more case groups (full list: study_cases/README.md)
engineering_levels/
  README.md                        <- the ladder, how to read level files
  01.associate-engineer.md
  02.mid-level-engineer.md
  03.senior-engineer.md
  04.principal-engineer.md
  05.engineering-manager.md
```

## How to use it

- **Interview prep**: for each case, read the case README first, then try to design it
  yourself before reading `03-architecture.md`. Compare. Note what you missed.
- **Cross-case patterns**: fanout strategies, ID generation, CDN vs origin, at-least-once
  delivery vs exactly-once — the same trade-offs show up everywhere. The study cases
  README maps which case covers which pattern.
- **Career growth**: the levels in `engineering_levels/` define what each engineering level
  can design and be held accountable for — use them to calibrate promotions and read the
  study cases at the right depth for your level.
- **Adding a case**: copy the folder skeleton, follow `CLAUDE.md`.

## Cases

Full index with pattern map: [study_cases/README.md](study_cases/README.md).

| # | System | Core problem it teaches |
|---|--------|------------------------|
| **Products** | | |
| 01 | YouTube | Massive video processing pipelines & content delivery |
| 02 | Twitter | Write vs read fanout, timelines at celebrity scale |
| 03 | WhatsApp | Millions of persistent connections & delivery guarantees |
| 04 | Deployment System | CI/CD queues, artifact stores, canary & auto-rollback |
| 05 | Discord | Chat when 100k-member guilds are the norm |
| 06 | Apple App Store | One release gate, delta-updating binaries worldwide |
| 07 | Blob Store | Exabyte object storage, 11 nines durability |
| 08 | ChatGPT | LLM serving: GPU pools, batching, token streaming, quotas |
| 09 | CDN | Edge geography vs origin economics |
| **Building blocks** | | |
| 10 | Distributed Cache | Consistent hashing, invalidation races, herds |
| 11 | ZooKeeper | Coordination: consensus, watches, locks |
| 12 | Bigtable | Wide-column stores, tablets, LSM trees |
| 13 | Kafka | The distributed append-only log |
| **ML infrastructure** | | |
| 14 | Production ML System | Feature skew, registries, drift → retrain |
| 15 | Ranking System | Retrieval + ranking in a latency budget |
| 16 | DALL·E | Async GPU jobs for text→image |
| 17 | ElevenLabs | Streaming text→speech at interactive latency |
| 18 | Sora | Text→video GPU-minute economics |
| **Client systems** | | |
| 19 | Frontend: Chat App | Optimistic UI, virtualized message lists |
| 20 | Frontend: Newsfeed | Infinite scroll, streaming SSR, prefetch |
| 21 | Frontend: Calendar | Offline-first sync, field-level LWW conflicts, timezones |
| 22 | Frontend: Uber | 60fps live maps, ride lifecycle state machines |
| 23 | Frontend: Video Streaming | ABR logic and buffers in the player |
| 24 | Mobile: Chat App | Socket death by OS, local schema evolution |
| 25 | Mobile: Newsfeed | Image pipelines, cold start, prefetch gating |
| 26 | Google Maps | Tiles, geocoding, hierarchical routing, live ETA |
| **Fleet & money systems** | | |
| 27 | EV Charging Network | 100K flaky chargers: offline autonomy, idempotent billing, OCPI roaming, fleet OTA |
| **Marketplaces & transaction systems** | | |
| 28 | Uber (Backend) | Geo indexing, surge as a control loop, ride state machines |
| 29 | Uber Eats | 3-sided dispatch: order as the only strong-consistency unit, courier allocation |
| 30 | Facebook Messenger | Per-user mailbox vs conversation canon, group fanout, the E2EE retrofit |
| 31 | Netflix | VOD at 80,000:1 read:write: prepositioned CDN, ABR + QoE loop, chaos as design |
| 32 | Spotify | Range-file delivery, offline sync DRM, rights as first-class data |
| 33 | Instagram | Media upload pipeline, stories ephemerality, hybrid fanout for a ranked feed |
| 34 | Reddit | Threaded comment trees, vote ledger + Wilson ranking, community-sharded moderation |
| 35 | TikTok | Interest-graph feed, no timeline store: recall→coarse→fine funnel, cold-start exploration |
| 36 | Airbnb | Spatial search, per-listing booking single-writer, escrow ledger, payout-delay fraud backstop |
| 37 | Ticketmaster | Waiting-room fairness, TTL holds as CAS leases, ~300× oversubscription |
| 38 | LeetCode | Online judge: sandboxing untrusted code, warm judge pools, SSE results, leaderboards |
| 39 | E-Commerce | 2000:1 read:write: reservation-with-expiry inventory, idempotent checkout saga |
| **Focused classics** | | |
| 40 | TinyURL | 100:1 redirect:write asymmetry, base62 keys, pre-allocated code ranges, bloom-gated redirects |
| 41 | Web Crawler | Frontier scheduling, SimHash+MinHash dedup, fail-closed robots, 304-driven recrawl |
| 42 | LinkedIn | Consented symmetric edges, degree-bounded graph computation, ACL-at-service caching |
| 43 | Zoom | SFU over MCU, sender-side simulcast, audio engineered separately, reconnect state machines |
| 44 | Payment System | Idempotency keys for exactly-once money, double-entry ledger as truth, bank-rail sagas |
| 45 | Typeahead | ~30 ms p50 / ~100 ms p99 typing path: materialized top-k per prefix, Zipfian hot keys, two-speed freshness |
| 46 | Google Docs | Server-sequenced OT op logs vs CRDTs, star topology, transform-based undo, offline mirror |

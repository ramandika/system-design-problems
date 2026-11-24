# Study Cases

Index of cases and the patterns each one covers. Start with any case's `README.md`.
Cross-case concept frequencies and the shared contracts: [CONCEPTS.md](CONCEPTS.md).

## Cases

### Products (01–09)

| # | Case | One-line essence |
|---|------|------------------|
| 01 | [YouTube](01.youtube/README.md) | Ingest → transcode → deliver petabytes of video cheaply |
| 02 | [Twitter](02.twitter/README.md) | Serving timelines when one write is read by 100M |
| 03 | [WhatsApp](03.whatsapp/README.md) | Persistent connections to billions, guaranteed delivery, offline sync |
| 04 | [Deployment System](04.deployment-system/README.md) | CI/CD at 100k concurrent pipelines: queues, artifacts, canary, auto-rollback |
| 05 | [Discord](05.discord/README.md) | Chat where the 100k-member guild is the normal case, not the celebrity exception |
| 06 | [Apple App Store](06.apple-app-store/README.md) | One release gate, ~2M apps, delta-updating binaries to billions of devices |
| 07 | [Blob Store](07.blob-store/README.md) | S3-like: durability 11 nines, data plane vs metadata plane at exabyte scale |
| 08 | [ChatGPT](08.chatgpt/README.md) | Serving LLMs to ~500M DAU: GPU pools, batching, streaming, quotas |
| 09 | [CDN](09.cdn/README.md) | Edge geography vs origin economics: cache keys, shielding, invalidation |

### Building blocks (10–13)

| # | Case | One-line essence |
|---|------|------------------|
| 10 | [Distributed Cache](10.distributed-cache/README.md) | Hash slots vs consistent hashing, invalidation races, thundering herds, hot keys, eviction |
| 11 | [ZooKeeper](11.zookeeper/README.md) | Coordination service: consensus, ephemerals, watches, locks done right |
| 12 | [Bigtable](12.bigtable/README.md) | Wide-column store: tablets, LSM trees, and why row keys hot-spot |
| 13 | [Kafka](13.kafka/README.md) | The distributed append-only log: partitions, ISR, consumer groups |

### ML infrastructure (14–18)

| # | Case | One-line essence |
|---|------|------------------|
| 14 | [Production ML System](14.ml-production-system/README.md) | Feature skew, feature stores, model registry, drift → retrain triggers |
| 15 | [Ranking System](15.ml-ranking-system/README.md) | Millions → dozens: retrieval + ranking in a latency budget |
| 16 | [DALL·E](16.dalle/README.md) | Async GPU jobs for text→image: queues, cost per image, moderation |
| 17 | [ElevenLabs](17.elevenlabs/README.md) | Streaming text→speech at interactive latency, per-customer voice models |
| 18 | [Sora](18.sora/README.md) | Text→video: GPU-minute economics, chunked job DAGs, review gates |

### Client systems (19–26)

| # | Case | One-line essence |
|---|------|------------------|
| 19 | [Frontend: Chat App](19.frontend-chat-app/README.md) | Message state machines, optimistic UI, virtualized 100k-message lists |
| 20 | [Frontend: Newsfeed](20.frontend-newsfeed/README.md) | Infinite scroll, streaming SSR, prefetch — jank-free at ranking speed |
| 21 | [Frontend: Calendar](21.frontend-calendar/README.md) | Offline-first sync, field-level LWW conflicts, timezone math, recurring events |
| 22 | [Frontend: Uber](22.frontend-uber/README.md) | 60fps moving markers, lifecycle state machines, degraded-network mode |
| 23 | [Frontend: Video Streaming](23.frontend-video-streaming/README.md) | ABR logic, buffer strategy, low-latency live in the player itself |
| 24 | [Mobile: Chat App](24.mobile-chat-app/README.md) | Socket death by OS, battery-aware reconnects, local schema evolution |
| 25 | [Mobile: Newsfeed](25.mobile-newsfeed/README.md) | Image decode pipelines, cold start, prefetch on the network you actually have |
| 26 | [Google Maps](26.google-maps/README.md) | Tiles, geocoding 100M POIs, hierarchical routing, live ETA |

### Fleet & money systems (27+)

| # | Case | One-line essence |
|---|------|------------------|
| 27 | [EV Charging Network](27.ev-charging-network/README.md) | OCPP gateways for 100K flaky chargers, offline autonomy, idempotent billing, OCPI roaming fanout, fleet firmware OTA |

### Marketplaces & transaction systems (28–39)

| # | Case | One-line essence |
|---|------|------------------|
| 28 | [Uber (Backend)](28.uber/README.md) | Matching, pricing, driver location: the backend half 22.frontend-uber defers — geo indexing, surge as a control loop, ride state machine |
| 29 | [Uber Eats](29.uber-eats/README.md) | 3-sided dispatch: order as the only strong-consistency unit, courier allocation, double-sided timing |
| 30 | [Facebook Messenger](30.facebook-messenger/README.md) | Per-user mailbox vs conversation canon, group fanout, the E2EE retrofit per-feature fate table |
| 31 | [Netflix](31.netflix/README.md) | VOD inverted vs YouTube: tiny ingest, 80,000:1 read:write → prepositioned CDN, ABR + QoE loop, chaos as design |
| 32 | [Spotify](32.spotify/README.md) | Bytes cheap, latency expensive: range-file delivery, offline sync DRM, rights as first-class data |
| 33 | [Instagram](33.instagram/README.md) | Media upload pipeline, stories ephemerality mechanics, hybrid fanout for a ranked photo feed |
| 34 | [Reddit](34.reddit/README.md) | Threaded comment trees (materialized path), vote ledger + hot/Wilson ranking, community-sharded moderation |
| 35 | [TikTok](35.tiktok/README.md) | Interest-graph feed, no timeline store: recall→coarse→fine funnel, staged publish, cold-start exploration |
| 36 | [Airbnb](36.airbnb/README.md) | Spatial search + date filters, per-listing booking single-writer, escrow ledger, payout-delay fraud backstop |
| 37 | [Ticketmaster](37.ticketmaster/README.md) | Scarcity engineering: waiting-room fairness, TTL holds as CAS leases, strip-as-shard, ~300× oversubscription |
| 38 | [LeetCode](38.leetcode/README.md) | Online judge: sandboxing untrusted code, warm judge pools, SSE results, contest leaderboards |
| 39 | [E-Commerce](39.e-commerce/README.md) | 2000:1 read:write storefront: reservation-with-expiry inventory, idempotent checkout saga, flash-sale ladder |

### Focused classics (40–50)

| # | Case | One-line essence |
|---|------|------------------|
| 40 | [TinyURL](40.tinyurl/README.md) | 100:1 redirect:write asymmetry, base62 key space, pre-allocated code ranges, bloom-gated edge redirects |
| 41 | [Web Crawler](41.web-crawler/README.md) | Mercator front/back frontier, SimHash+MinHash dedup, fail-closed robots, 304-driven recrawl |
| 42 | [LinkedIn](42.linkedin/README.md) | Consented symmetric edges, degree-bounded graph computation, canonical-profile caching vs per-viewer ACL |
| 43 | [Zoom](43.zoom/README.md) | Every participant is a sender: SFU over MCU, sender-side simulcast + receiver-driven layers, audio engineered separately, reconnect state machines |
| 44 | [Payment System](44.payment-system/README.md) | Small QPS, absolute stakes: idempotency keys for exactly-once money, double-entry ledger as truth, bank-rail sagas, reconciliation backstop |
| 45 | [Typeahead](45.typeahead/README.md) | Sub-100ms in the typing critical path: materialized top-k per prefix, Zipfian hot keys, two-speed freshness, tail-prefix fallback |
| 46 | [Google Docs](46.google-docs/README.md) | Server-sequenced op logs (OT) vs CRDTs, star-topology convergence, transform-based undo, offline mirror, ephemeral presence channels |
| 47 | [Rate Limiter](47.rate-limiter/README.md) | Distributed enforcement at µs overhead: quota leases vs centralized counters, atomic read-modify-write, hot-key chunking, fail-open/fail-closed per class |
| 48 | [Google Search](48.google-search/README.md) | Web-scale retrieval: sharded partitioned index, fresh-tier vs base-tier freshness, WAND early termination, staged ranking funnel |
| 49 | [Monitoring Metrics](49.monitoring-metrics/README.md) | Prometheus-style pull monitoring: label cardinality economics, local-first TSDB (head/WAL/chunks), no-consensus HA, Thanos-style long-term federation |
| 50 | [Stock Exchange](50.stock-exchange/README.md) | Correctness over availability: sequenced deterministic matching, price-time priority, event-sourced journal replay, multicast fanout of market data |

## Cross-case pattern map

| Pattern | Covered in |
|---------|-----------|
| Fanout on write vs on read | Twitter (celebrity fanout), Discord (huge guilds), WhatsApp (group messages) |
| Content-delivery networks (CDN) | CDN (edge caching as the product: anycast, cache keys, shielding, purges), YouTube, Twitter media, Apple App Store (binary delivery), Google Maps (tiles), Sora (generated video) |
| Async processing pipelines / DAG | YouTube (transcoding), Twitter (search ingest DAG), Deployment System (pipeline runs), Sora (chunked generation) |
| Cache tiering (local, Redis, CDN) | CDN (edge → shield → origin tiering), YouTube, Twitter, Distributed Cache, Google Maps |
| Distributed ID generation | Twitter (Snowflake), Discord (snowflake-ordered message IDs), WhatsApp (message IDs) |
| Delivery guarantees: at-least-once, ACK, idempotency | WhatsApp, Deployment System (outbox + fencing), Apple App Store (purchase ledger) |
| Storage tiering (hot/warm/cold) | YouTube, Twitter, Discord (channel history) |
| Sharding & partitioning keys | Twitter (timeline shards), WhatsApp (per-conversation shards), Bigtable (tablets + row-key design), Kafka (partitions), Distributed Cache (hash rings) |
| Backpressure & rate limiting | Rate Limiter (distributed quota enforcement, leases vs counters), WhatsApp, Twitter, ChatGPT (GPU admission control), DALL·E / ElevenLabs (GPU queues) |
| End-to-end properties users can feel | WhatsApp (E2E encryption), Discord (voice), Deployment System (fenced rollouts), ChatGPT (first-token latency), Zoom (reconnects + audio-under-loss), Google Docs (200ms remote-keystroke bar) |
| Consensus & coordination | ZooKeeper (ZAB vs Raft), Kafka (ISR election), Deployment System (single-writer controller) |
| Storage engines (erasure coding, LSM, append-only) | Blob Store (RS(8,4) vs replication), Bigtable (MemTables/SSTables), Kafka (segmented log), Monitoring Metrics (TSDB head + chunk compaction), Stock Exchange (event-sourced journal as the only truth) |
| Streaming inference & latency budgets | ChatGPT (token streaming), Ranking System (retrieval + rank split), ElevenLabs (audio chunks), Sora (GPU-minute cost) |
| Feature stores & training/serving skew | Production ML System, Ranking System, Google Maps (ETA) |
| Offline-first client sync | Frontend Calendar (server-sequenced pull + outbox), Frontend/Mobile Chat (queue + backfill), Mobile Newsfeed (last-good state) |
| Client rendering at scale | Frontend Chat (virtualization), Frontend Newsfeed (windowed lists), Frontend Uber (canvas markers), Mobile Newsfeed (image decode) |
| Adaptive bitrate / player economics | Frontend Video Streaming (ABR + buffers), YouTube (server side of the same bargain) |
| Degrade-toward-autonomy (edge keeps working when the backend is down) | EV Charging Network (offline auth + queued billing), Deployment System (fenced rollouts), Mobile Chat (local queue + backfill) |
| At-least-once ingest → exactly-once money effect | EV Charging Network (CDR ledger), Apple App Store (purchase ledger), Deployment System (outbox), Uber Eats / E-Commerce (idempotent checkout) |
| Geospatial indexing (geohash/H3/quadtree) | Uber (driver cells, surge tiles), Airbnb (posting lists), Google Maps (tiles/POI priors) |
| Admission control & fairness under flash load | Ticketmaster (waiting room, TTL holds as leases), E-Commerce (flash-sale ladder), ChatGPT (GPU admission), Stock Exchange (sequence-position fairness, journaled throttles) |
| Sandboxing untrusted execution | LeetCode (gVisor/microVM judge pools), Deployment System (pipeline runners) |
| Deterministic replay / event sourcing as recovery | Stock Exchange (byte-equal journal replay, active–standby), Google Docs (op-log), Kafka (log as truth) |
| Staged funnel: coarse filter → expensive ranker | Ranking System (retrieval + rank), Google Search (WAND early termination + staged ranking), Typeahead (top-k pruning) |
| Reservation / hold semantics with expiry | Airbnb (hold→confirm per-listing single writer), E-Commerce (checkout reservations), Ticketmaster (seat holds), EV Charging Network |
| Live media fanout (SFU, simulcast, receiver-driven layers) | Zoom (video conferencing), Discord (voice), Frontend Video Streaming (player-side ABR — the on-demand sibling) |
| Double-entry ledger & reconciliation as truth backstop | Payment System (immutable postings, settlement files), Airbnb (escrow ledger), EV Charging Network (CDR ledger), Apple App Store (purchase ledger) |
| Exactly-once money effects via idempotency + serialization | Payment System (idempotency keys + bank-side references), Uber Eats / E-Commerce (idempotent checkout), Deployment System (outbox + fencing), Apple App Store (purchase ledger) |
| Prefix index & multi-tier suggestion serving | Typeahead (materialized top-k, client→edge→per-prefix cache), Google Maps (geocoding/POI autosuggest), Google Docs (doc search metadata) |
| Collaborative sync: server-sequenced ops vs CRDTs | Google Docs (OT, star topology, transform-based undo), Frontend Calendar (server-sequenced ops, field-level LWW), Mobile Chat (queue + backfill) |

# 01 — Requirements

## Functional

- **Buckets**: named containers with globally unique names; flat namespace of keys inside
  a bucket; per-bucket configuration (storage class, lifecycle rules, region placement).
- **PUT object**: store bytes (1 B to ~5 TB, following S3's published limit) at
  `bucket/key`; idempotent per unique request token; objects are immutable in place —
  replace = write-new-key or versioned overwrite.
- **GET object** / **GET byte range**: read whole object or arbitrary range; streaming,
  seekable downloads.
- **DELETE object**: delete, or delete-marker when versioning is on; deletes are soft
  internally even when the API reports them as done (see deep dive 5).
- **LIST**: enumerate keys under a prefix, lexicographic order, paginated by marker;
  supports delimiter grouping ("directories" as a convention, not a real hierarchy).
- **Multipart upload**: initiate → upload N independent parts (1 part ≈ 5 MB–5 GB, up
  to 10,000 parts per upload, per S3's published limits) → complete (assembles logically)
  → abort. Parts of an in-progress
  upload are durable and individually retrievable — a paused 5 TB upload survives client
  and datacenter restarts.
- **Versioning opt-in**: every overwrite creates a new immutable version; restores are
  pointer flips.
- **Auth/authz**: request signing (HMAC-style, replay-protected by timestamp), per-bucket
  policies, IAM identities.

## Non-functional

- **Durability**: target 11 nines annualized durability (S3's published design target:
  99.999999999%/yr). This is the headline requirement; everything about layout and repair
  flow from it (deep dive 5).
- **Availability**: read-after-write on the object that was just written (the client must
  be able to GET the key it successfully PUT); ~99.99% availability design target
  (S3-published; the S3 SLA commits to a 99.9% monthly uptime bound); listing eventually
  consistent, with the lag bounded by the small-delta absorption window (deep dive 2).
- **Latency (rough targets, first-byte)**: ~50–100 ms GET first byte from a warm datacenter
  path; PUT ack after data is on durable media across enough shards, not after EC compute
  completes end-to-end.
- **Throughput per flow**: a single client should be able to saturate a multi-Gbps link
  (parallelism per object, not per request).
- **Scale**: exabytes of capacity, trillions of objects (S3 is publicly reported at
  hundreds of trillions of objects); capacity is the binding constraint, request rate is
  secondary.
- **Multi-tenancy**: hard isolation between tenants in bandwidth, request rate, and
  capacity accounting; one tenant's 10 TB restore storm must not evict another's latency.

## Scope decisions

**In scope**: the storage service itself — HTTP API gateway, metadata/index plane, data
plane, multipart coordinator, scrubber/repair/rebalance background infrastructure,
durability design, cross-datacenter layout.

**Explicitly out of scope:**

- **Higher-level services on top** (`Notify`, event subscriptions, lifecycle-managed
  analytics, query-over-objects) — separate products; we only expose the raw
  event/iteration hooks they'd need.
- **The CDN and edge caches** — we are the origin; see ../01.youtube/04-deep-dives.md for
  how an origin behind a CDN gets shielded from client fanout.
- **Consensus/queue systems built by tenants on the API** — we don't optimize semantics
  for it beyond keeping PUT/GET per-key behavior simple and documented.
- **File semantics**: no locking, no appends after PUT (multipart is the only way to
  assemble something big), no directory rename. Flat keys are a documented, deliberate
  limitation — hierarchy is a client-side convention.
- **Encryption key management internals** — we do server-side encryption at rest, but key
  ceremony/KMS is another system's problem.
- **Billing pipeline** — we emit metered usage records; invoicing is downstream.

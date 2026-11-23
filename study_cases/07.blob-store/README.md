# Case 07 — Blob Store (S3-like object storage)

**The system**: an object store exposing HTTP PUT/GET/DELETE/LIST over buckets and keys,
storing objects from a few bytes to multiple TB. The product promise is unusual: data you
can never lose (S3's published design target is 11 nines of durability,
99.999999999%/year), always reachable, metered and multi-tenant — backed mostly by cheap
hard disks that fail routinely.

**Why it's hard**:

- **Durability is a background property, not a request property.** No PUT can make data
  durable 11 nines deep — only erasure coding, continuous scrubbing, and repair faster
  than hardware fails can. The hard work happens when no user is watching.
- **Two planes with opposite needs**: metadata (small, strongly consistent, key-ordered,
  listing-heavy) vs data (huge, append-heavy, parallel I/O). Forcing both into one store
  fails both ways — this is the single biggest architectural fork in the case.
- **Listing consistency** is notoriously subtle: the "small-delta list" problem, where a
  bucket that gets a few new puts is listed thousands of times before the delta is
  absorbed (see deep dive 2).
- **Multipart uploads** make 5 TB objects possible over unreliable links — which means
  the store must hold half-finished uploads durably for days and stay resumable.
- **The cost floor dominates**: at exabyte scale, storage-media cost per GB and repair
  bandwidth are the design constraints, not QPS.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: control/data plane separation (compare Twitter's fanout tier,
which also moves heavy write work off the read-serving path — [Twitter deep
dives](../02.twitter/04-deep-dives.md)), erasure coding
and placement (contrast the 3× replication on master copies that YouTube's estimates budget
— [YouTube estimates](../01.youtube/02-scale-estimates.md)), idempotent retryable work units
(same discipline as resumable upload/transcode segments in YouTube), anti-entropy repair.

**Try it first**: before reading 04, answer: (1) why not just replicate 3× like a
database, (2) why can't LIST be served straight off the same index as GET, (3) what
happens when an entire datacenter's disks go offline at once — how many shards can you
afford to lose simultaneously?

**Primary sources**: S3's published durability and API design targets (AWS docs: 11 nines
durability, 5 TB max object, 10,000 parts per upload) are public specs, used as targets —
not measurements. Internal mechanisms described here are standard published practice
(erasure coding, scrubbers, index sharding), labeled as such where the real system's
implementation is not public.

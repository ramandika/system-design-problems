# Case 11 — ZooKeeper

**The system**: a coordination service for clusters. Not the system that serves users —
the system that tells the systems that serve users who is in charge. A small replicated
store of tiny nodes (**znodes**) with sessions, ephemeral nodes, one-shot **watches**, and
a total order on every update, on top of which client libraries build leader election,
distributed locks, and group membership. ZooKeeper is the open-source sibling of Google's
Chubby, and for ~a decade it coordinated Kafka, HDFS NameNode HA, HBase, Mesos, Solr, and
YARN.

**Why it's hard** (and it is not about scale):

- ZooKeeper is correctness-critical at *small* scale: a few GB of memory, thousands of ops/s,
  3–5 servers. Its difficulty is that **every other system's safety depends on it** — a
  wrong leader, a stale read trusted as fresh, or a lock that lets two holders through is
  a data-corruption bug in the *using* system.
- Coordination is where networks bite hardest: a 30 s GC pause is indistinguishable from a
  dead client, a partitioned leader keeps believing it is the leader, and session expiry is a
  probabilistic judgment call, not a fact.
- The API is deliberately awkward (one-shot watches, no server-side locks) because pushing
  complexity to clients is how the service stays simple — but that pushes every classic
  bug (thundering herds, missed events, stale caches) into user code.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**What this case teaches that the big-traffic cases don't**: all other cases here scale
*data* or *users* (Twitter's fanout, WhatsApp's sockets, Kafka's logs). ZooKeeper is the
counter-case: it must scale *trust* — a total ordered log, quorum math, session/lease
semantics, and notification under partition. Its throughput ceiling is a **disk fsync**, not
a CPU. If you only read one file, make it [04-deep-dives](04-deep-dives.md) — ZAB vs Raft,
slightly-stale reads, ephemeral nodes during partitions, and why a ZooKeeper lock is never
sufficient alone are the pieces every distributed system re-implements badly.

**Patterns taught here**: consensus (ZAB; the same slot [Kafka's KRaft](../13.kafka/03-architecture.md)
eventually filled itself), quorum reads/writes (compare [Bigtable's](../12.bigtable/04-deep-dives.md)
Chubby-style metadata), lease/ephemeral state (compare WhatsApp's [presence tier](../03.whatsapp/04-deep-dives.md)),
watch-based notification, and herd avoidance.

**Try it first**: before reading 04, answer: (1) why can a *read* from ZooKeeper be stale
if every *write* went through a majority quorum? (2) the leader's pod freezes for 40 s with
a 30 s session timeout — what exactly happens, on both sides? (3) why are watches one-shot,
and what bug does that force every client to think about?

**Primary sources**: Hunt, Konar, Junqueira, Reed, ["ZooKeeper: Wait-free coordination for
Internet-scale systems"](https://www.usenix.org/legacy/event/atc10/tech/full_papers/Hunt.pdf)
(USENIX ATC 2010) — the design/API paper with the traffic benchmarks; Junqueira, Reed,
Serafini, ["ZAB: High-performance broadcast for primary-backup systems"](https://marcoserafini.github.io/papers/zab.pdf)
(DSN 2011) — the protocol; Burrows, ["The Chubby lock service for loosely-coupled distributed
systems"](https://research.google/pubs/the-chubby-lock-service-for-loosely-coupled-distributed-systems/)
(OSDI 2006) — the design philosophy counterpoint; Apache Kafka KIP-500 materials for why
Kafka replaced it. All performance figures below are 2010-paper-era or explicitly labeled
estimates.

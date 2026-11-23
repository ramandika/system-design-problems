# 04 — Deep dives

## 1. ZAB vs Raft: same quorum, different commit story (THE dive)

Both protocols answer "how do 2f+1 replicas agree on one totally-ordered log and survive f
failures". Both are leader-based, use majority quorums, and are log-replicated state
machines. The differences are in how they commit, recover, and what they order.

**Option A — ZAB**, designed (Junqueira et al., DSN 2011) around two properties:
*primary order* — all followers apply the leader's broadcasts in the same order; and
*causal order* — a broadcast implies all prior broadcasts were already delivered.

- Commit is proposal-then-quorum-ACK-then-COMMIT, per transaction, **pipelined** — writes
  stream through without waiting for the previous one's COMMIT except through the log order.
- Ordering lives in a single 64-bit **zxid** = (epoch, counter): a leader change bumps the
  epoch and resets the counter, so every zxid is globally comparable across history.
- Recovery is explicit and content-based: the new leader syncs each follower with
  DIFF/SNAP/TRUNC — replaying committed tail, snapshotting stale ones, truncating
  proposals a dead leader had proposed-but-not-committed. The invariant: a committed
  transaction's zxid must be below the first zxid of every later epoch (so a new leader
  can never resurrect an aborted proposal).

**Option B — Raft** (concurrent, later — Raft's authors cite ZAB among ancestors):
leader elected by explicit vote with log comparison, commit index advanced implicitly, snapshots
replaced by InstallSnapshot, term/index kept separate.

- wins: a simpler, cleaner model to reason about and re-implement (hence dozens of clones:
  etcd, Consul, TiKV, Kafka's KRaft — see [13.kafka](../13.kafka/03-architecture.md)).
- loses: slightly weaker single-number ordering for comparison across server restarts
  (you check term+index together); no built-in "here is why my prefix is right" transfer
  story at the ZAB level (Raft handles it with AppendEntries consistency checks).

**Chosen in the real world: both.** ZooKeeper shipped ZAB and kept it forever — the commit
is two-phased and content-driven precisely because ZooKeeper needed an *atomic broadcast*
for a state machine that also serves *reads* from replicas' local trees (deep dive 2), and
zxid-as-single-comparable-number is exactly what version fencing (deep dive 3) and the
sequential-node recipe (deep dive 6) build on. Newer systems picked Raft because
consensus is no longer their whole reason to exist and they want it as a small embedded
library. ZAB is not wrong; it is a committed design with a decade of operation behind it —
the trade-off its authors took (extra recovery machinery like TRUNC, in exchange for a
pipelined, fsync-barrier-driven broadcast that doubles as a read-prefix protocol) is paid
for in code complexity, not runtime cost.

- Trade-off made explicit: ZAB wins *comparable ordering + pipelined quorum fsync* and
  loses *protocol simplicity*; Raft wins simplicity and gives nothing up on guarantees.
  Pick ZAB-shaped systems when the log's ordering number is your API surface; pick Raft
  when consensus is an embedded component.

## 2. Quorum math and why reads can be slightly stale

Quorum math first, because everything else sits on it: with `2f+1` voting servers, a
write commits on an fsync'd ACK majority — `f+1` voting servers *including the leader*,
i.e. the leader plus `f` follower ACKs (a `WRITE-QUORUM`), and the leader needs that same
majority alive to keep ordering (overlap of any two write-quorums guarantees a surviving
follower saw every committed transaction). 3 servers tolerate 1 failure, 5 tolerate 2, and each
extra server *lowers* write throughput (more fsyncs per commit) while *raising* failure
tolerance — see the arithmetic in [02-scale-estimates](02-scale-estimates.md).

**Why reads are stale.** Naive intuition says "every write went through `f+1` servers, so
any server can serve fresh reads". That is wrong, and for a good reason:

- Committed transactions reach each server on a **best-effort schedule** after the COMMIT.
  A follower can be behind by a few transactions — it serves the *committed prefix* it has.
  That prefix is never wrong (it is a true subset of the global order), but it can *lag*.
- The read is answered from memory **without consulting the leader**. That is intentional:
  95% of real traffic is reads (02-estimates), and routing reads through the leader would
  collapse the fsync-pipelined write path and the read path into one bottleneck — the
  ensemble would behave like one server regardless of size.

- **Option A — linearizable reads through the leader.** wins: reads always fresh.
  loses: read throughput = leader throughput; a hot read now contends with every write's
  proposal, and a single overloaded server becomes the whole service. The ATC'10 paper's
  read-scaling result (reads scale with servers, writes don't) exists only because reads
  are local.
- **Option B — stale-tolerant local reads + explicit `sync()`.** wins: read throughput
  scales linearly with ensemble size, read latency is one memory lookup + one RPC.
  loses: a read can miss the last few transactions; applications that read-then-act on
  coordination data need to know this.

**Chosen: Option B**, with a specific rule: **a read you are about to act on, or whose
staleness affects a coordination decision, must be either (a) issued against the server
your session is currently writing to (own-write visibility is guaranteed), or (b) preceded
by `sync(path)`, which forces the serving server to catch up to the leader before reading.**
Everything else — config, membership, "roughly what's here" — reads locally and accepts a
few-ms staleness window, the same discipline as any read-replica system
([Bigtable's](../12.bigtable/04-deep-dives.md) client-cached tablet locations — stale
entries fall back to a root/METADATA lookup — and [Twitter's](../02.twitter/04-deep-dives.md)
async fanout, whose materialized timelines can briefly lag the fanout workers; ZooKeeper's
escape hatch is just cheaper because the whole dataset is in one memory tree).

- Trade-off made explicit: we give up "any read from any server is linearizable" in exchange
  for read throughput scaling with ensemble size and read latency of memory-only cost;
  we keep correctness by (1) constraining what clients may *conclude* from a stale read and
  (2) exporting `sync()` as the freshness escape hatch.

**Failure modes**:

- *Stale read used for a safety decision*: a client reads "no leader exists" from a lagging
  follower, launches its own election, and now two leaders think they're in charge. This is
  not a ZooKeeper bug — it is a recipe bug. The recipes in deep dive 6 work because they
  use **sequential node creation (a write)** to observe the election, not a read.
- *Stale read of ephemeral existence*: `exists()` on a lagging follower can return true for
  a node whose session just died. Same shape, same answer: don't build failover logic on
  stale reads; build it on ephemeral watches fired from the server that owns the session.
- *`sync()` is not linearizable* — it is ordered-read ("you'll see everything committed
  before your sync ran"). If a client needs a true compare-and-swap decision, that is
  `setData(path, data, expectedVersion)` — a write — which is always linearized.

## 3. Sessions and ephemeral semantics on a partition: the 40-second GC story

Sessions are the crux of the whole service, because they are how liveness is *decided*.
Concrete story: a Kafka broker (or HDFS NameNode, or Spark driver — same everywhere) that
was elected leader via an ephemeral node hits a garbage-collection pause for 40 s with a
30 s session timeout.

**What actually happens, on both sides:**

```
t=0     leader freezes (JVM GC pause / VM stall)
t≈32s   server-side session timeout hits at the negotiated 30 s plus up to one tick of
        expiry granularity (tick ~2 s, 01-requirements F7): the ensemble deletes the
        ephemeral node that marked leadership
t≈34s   a watch fires on the election path → new election starts
t≈36s   NEW leader elected, takes over work
t=40s   old process wakes up, still believes it holds the lock
```

The system now has two processes that both "won". Fixing this is **not a ZooKeeper feature**
— ZooKeeper cannot reach into the frozen process. The fix is three disciplines:

1. **Fencing.** Every coordination decision must carry a monotonically increasing token
   from ZooKeeper (the zxid of the write that created the lock, or the sequential suffix of
   the ephemeral). The *resource being guarded* (a Kafka log, an HDFS namespace, a chunk
   server's data) checks the token and rejects anything older. The frozen leader comes back
   with token 41; the cluster has already accepted writes under token 42; the resource
   drops it. Fencing converts "I believe I am leader" into "the resource has not advanced
   past me".
2. **Client-side defensive check.** On reconnect after a long pause, a well-behaved client
   checks `isAlive` state for its session — a session the library knows was expired forces
   the client to tear down and re-acquire everything (ephemerals, locks, election roles),
   never silently resume.
3. **Timeout tuning as a load parameter, not a magic constant.** `minSessionTimeout`–
   `maxSessionTimeout` bound what a client can negotiate; a too-short timeout turns routine
   GC pauses into mass expirations and spurious failovers; a too-long one stalls failover
   after a real crash. Tune it for the *worst expected pause*, not the average ping
   (02-estimates).

- **Trade-off**: shorter sessions detect real crashes faster (less downtime) but fail more
  often during pauses (more spurious churn); longer ones are patient with GC but slow real
  failover. Both directions cost something; the fencing token (discipline 1) is what makes
  the failure non-catastrophic either way — get the token right and mis-tuned timeouts are
  an availability bug, not a correctness one.

**Partition semantics — the other half of the story.** When the ensemble splits:

- The **majority side** keeps committing. The minority side's leader cannot get quorum for
  its proposals and steps down after failing to reach followers; its servers serve reads of
  their committed prefix (correct but frozen) and reject writes.
- **Old leader on the wrong side of the partition**: it is fenced by epoch, not by
  politeness — every proposal it sends after the new leader's election carries an old
  zxid-epoch and is refused. The same zxid-fencing that protects against GC pauses protects
  against partitions; there is one mechanism for both.
- **Clients on the minority side** keep reconnecting and expiring sessions until
  connectivity restores. Their ephemerals are already gone (deleted by the majority-side
  session expiry). When they return, they must re-establish everything from scratch.
- **Contrast with WhatsApp's presence tier** ([03.whatsapp](../03.whatsapp/04-deep-dives.md)):
  presence is ephemeral-in-memory *without* any fencing, because nothing safety-critical
  reads it. ZooKeeper's ephemeral nodes look similar on the surface — session-scoped,
  self-cleaning — but they are *consensus-ordered*, which is what makes fencing possible
  and what makes this service trusted for leadership rather than just liveness hints.

- Trade-off made explicit: session-based liveness (ephemeral nodes) wins *zero client-side
  cleanup code and a self-healing membership* and loses *the possibility of a false leader
  during pauses/partitions*; we pay for that with fencing tokens at the resource, not with
  anything ZooKeeper itself can do.

**Failure modes**:

- *No fencing token checked at the resource*: two leaders write. This has happened
  historically in real systems (the HDFS NameNode and Kafka controller both gained explicit
  fencing mechanisms after exactly this class of bug — HDFS via block-id generation
  numbers/epoch tokens, Kafka via controller epoch and fenced broker registration).
- *Ephemeral node + long-lived work*: the leader's session expires mid-deployment because
  the deployment froze; the new leader overlaps work. Mitigate with generous-but-bounded
  timeouts and with work-side fencing, never by "make sessions longer and hope".
- *Mass expiry burst*: a network blip expires hundreds of sessions simultaneously →
  hundreds of ephemeral deletes in a burst of txns, briefly saturating the fsync-bound
  write path (02-estimates arithmetic). ZooKeeper handles it (it's a bounded burst), but it
  is why operators watch session churn as a metric.

## 4. Watches as a notification primitive: one-shot on purpose

A ZooKeeper watch is not pub/sub. It is **a one-shot invalidation signal attached to a
read**. You call `exists(path, watch)` or `getData(path, watch)`; the server records "tell
this session once, on the next change of kind `X` to this path"; when it changes, one event
fires, and the watch is consumed.

Why one-shot rather than persistent:

- **Option A — persistent subscriptions** (a log or topic per watched path): win — the
  client never re-arms and can never miss an event between re-arms. loses — the server now
  owns per-subscriber state that (a) scales with *subscribers × watched paths*, (b) must
  decide what to do when a subscriber is slow or disconnected (buffer? drop? block the
  writer?), and (c) contradicts "the server does not know what a client wants this data
  for" (01-requirements scope: no server-side recipes).
- **Option B — one-shot + re-arm** (ZooKeeper's choice): win — zero per-subscription state
  beyond one list entry; the client drives everything and a slow client just gets fewer
  events, never affecting other clients. loses — *between fire and re-arm, events are
  lost*, and clients must re-read the watched node to catch up on whatever they missed.

**Chosen: Option B.** ZooKeeper's whole design pushes per-subscriber knowledge and retry
policy into the client, where it is a small local loop (`on event → re-read → re-arm`),
rather than server state that must be managed for thousands of heterogeneous clients with
different tolerance for loss. The cost is a contract every client must understand explicitly:
a watch is a *hint that something changed*, not *a stream of every change*. Read-back after
fire is not optional.

- Trade-off made explicit: we win server-side simplicity and per-client isolation at the
  cost of losing events in the re-arm gap — acceptable precisely because the client can
  always just re-read the node, which is the authoritative state.

**Thundering herd: the quantified trap.** A config znode with 1,000 watchers (02-estimates):
one `setData` fires 1,000 watch events in microseconds, and each client immediately
re-reads the node and re-arms — 1,000 reads land on the ensemble within the next instant.
At ~9,500 reads/s/server average (~47,500/s cluster-wide, 02-estimates), a single
hot-znode write is a ~2% spike cluster-wide (1,000 burst reads out of ~47,500 read/s);
on a small ensemble the spike is much bigger proportionally. The
classic recipe bugs that turn this spike into an outage:

- *The "watch the parent, list the children" bug*: 1,000 workers all watch `/jobs` for
  `ChildEvent`, and every job create/delete triggers 1,000 `getChildren` calls + full list
  comparisons. Fix: the **herd-avoidance recipe** (deep dive 6) — each candidate watches
  *only its predecessor's* sequential node, so one notification goes to one worker per
  change.
- *Re-arm stampede after a server restart*: a follower restarting drops all its watches;
  every affected client reconnects (to other servers) and re-watches — the same herd shape
  as WhatsApp's [reconnect storms](../03.whatsapp/04-deep-dives.md). Backoff on
  re-establishment, randomized jitter, and batching re-reads are the same mitigations.

- Trade-off: per-watch notification cost is paid at every change, so the design must
  minimize *watchers per path* (recipe-level), not optimize the event delivery itself.

**Failure modes**:

- *Missed events in the re-arm gap* leading to stale decisions in client logic — this is
  the expected behavior, not a bug; clients must check state, not infer it from absence of
  notifications.
- *Watch on ephemeral by a different session*: if the ephemeral's owner crashes, watchers
  get a delete event — but they race against other watchers (all get notified at once) and
  against new candidates; the sequential+predecessor recipe exists precisely to linearize
  this race (deep dive 6).
- *Watch event ordering vs. other client's reads*: the watch event is sequenced with the
  change on the same server, so a client sees the event before it sees the newer data on its
  connection — but only on that connection. Cross-server timing still requires the
  staleness discipline from deep dive 2.

## 5. Scaling reads with observers — and why that is the *only* scaling lever

Ensembles do not shard. One ordered log, one memory tree — that is a requirement
(01-requirements), not a limitation inherited by accident. So when read load grows past
what a 5-server ensemble serves comfortably (~9,500 reads/s/server average, ~28,500/s
per-server peak from the 142,500/s total peak — see [02-estimates](02-scale-estimates.md)), the options are:

- **Option A — bigger voting ensemble** (7, 9, 11 servers): wins — reads scale linearly.
  loses — write throughput **falls** (each commit needs `f+1` fsync barriers, so 7-server
  writes are ~1.33× the fsync work of 5-server writes for less than 2× the failure
  tolerance); each added server also adds a network hop of proposal broadcast per write.
- **Option B — observers**: servers that receive COMMITs and apply them, serve reads, and
  *do not participate in quorum math*. Adding an observer costs the leader one extra
  broadcast per transaction (~750 KB/s per observer at this case's average load —
  2,500 txns/s × ~300 B, estimate, 02-estimates) and nothing else. wins — read capacity scales with zero write-quorum cost; observers can even be
  placed near a consumer-heavy service or a second datacenter. loses — observers cannot
  commit (they must never be counted in quorums), no availability benefit (if the voting
  ensemble is down, observers serve a frozen committed prefix), and they add one more
  component whose lag must be watched.

**Chosen: Option B for load, Option A only for failure tolerance.** The quorum math is
optimized for few voting servers (3–5) because writes pay for every extra voter; observers
exist precisely so read scale and quorum latency do not fight each other. This is the same
split the whole corpus keeps rediscovering — decouple the *consensus domain* from the
*serving domain* ([Bigtable](../12.bigtable/04-deep-dives.md) serves tablet traffic from
non-consensus tablet servers while Chubby/ZooKeeper-class systems hold the metadata;
[Kafka](../13.kafka/04-deep-dives.md) serves consumer reads from
replicas' page cache without those reads paying the acks/ISR commit gate; Twitter fans reads
into cache tiers that answer without touching the source
of truth).

- Trade-off made explicit: observers win read scalability and topology flexibility (put
  reads wherever the readers are) and lose any availability or quorum contribution; they
  are a caching tier with a consensus-fed refill, not redundancy.

**Failure modes**:

- *Treating observers as redundancy*: an "ensemble" of 3 voting + 2 observers is still a
  3-member voting cluster that tolerates exactly one voting-server fault — observers going
  down costs read capacity, not availability or quorum strength, and counting them as
  capacity for a *write* surge is a mistake.
- *Observer lag as a correctness bug*: an application pinned (by a connection-aware load
  balancer) to a far-behind observer makes the staleness of deep dive 2 worse than a
  follower's. Cap observer lag expectations and never let a client make coordination
  decisions from observer-only reads (the `sync()` discipline applies here too).
- *Observer storm on commit*: a hot znode with 1,000 watchers pinned to observers fires 1,000
  re-reads, each landing on observers (deep dive 4's herd) instead of voting servers —
  good (isolates the herd), but it also means the *herd is invisible* to voting-server
  metrics; monitor observers too.

## 6. Locks, elections, and the recipes that don't stampede

The server has no lock concept (01-requirements). Everything here is a *client recipe*,
and the recipes are where most real-world ZooKeeper bugs live.

**Naive lock (broken).** `create /lock/leader` with ephemeral=true — whoever creates it
first wins. Under contention, N clients hammer the same parent directory with creates and
failed creates: every retry is a server-side attempt, at N× request load per lock decision
(failed creates are rejected before becoming transactions — they cost RPCs, not fsyncs).
Worse: on release, *all* N losers get a watch fired
simultaneously and all re-create and retry — a thundering herd every time a lock turns over
(deep dive 4's herd, amplified because that retry burst *is* real write-transaction load,
feeding the fsync path).

**The herd-avoidance recipe (used correctly everywhere).** Each candidate creates
`/election/n_` with sequential+ephemeral, then:

```
children = getChildren("/election", watch=false)  # read only, NO watch on parent
me = my sequential node
if me is the lowest → I am leader
else → watch(children[the one directly below me]) and nothing else
```

One notification goes to exactly one candidate per turn-over, and only the candidate whose
predecessor disappeared re-checks. All the other N−1 candidates stay quiet with one
outstanding watch each. This is the pattern that makes the *cost* of an election O(1)
notifications rather than O(N), which matters when the same ensemble is also serving
critical metadata traffic (02-estimates).

- Trade-off (naive vs. herd-free recipe): the naive recipe wins "one API call" and loses
  O(N) write load per contention round plus an O(N)-notification release; the sequential
  recipe wins O(1) notification cost and fairness (strict FIFO by sequence number) and
  costs one extra read+watch per candidate and slightly more client code. Pick naive
  only for N≤2 (primary/backup), pick sequential for anything else.

**Failure modes of the recipes (the ones that actually happen):**

- *Lock held during a long GC pause or partition*: session expires → node deleted → new
  holder; old holder resumes unaware. **This is not a recipe bug — it is the semantics of
  sessions, and there is no ZooKeeper-side fix.** Every correct system pairs the recipe
  with fencing: the *work itself* carries a token (zxid, version, sequential suffix) and
  the guarded resource rejects stale tokens (deep dive 3). If your system fails without a
  ZooKeeper lock, you have built the lock wrong — ZooKeeper tells you *something changed*,
  it cannot stop two processes from being wrong about who is acting.
- *Lock released on network blip while work continues*: same shape — the holder is alive,
  its session is not. Mitigation: keep the pause short (small session timeout on the
  *lock* session, separate from application traffic) and fence anyway.
- *Election with too-short session timeout* → leaders flap (up/down cycles, churn of
  ephemeral deletes, 02-estimates burst signature). Election recipes should use a
  session timeout set *explicitly* for the election role, not the client's default.
- *Zombie leader in a minority partition* (deep dive 3): handled by epoch-fencing, not the
  recipe. A client library that doesn't expose fencing tokens is not production-complete.

**Where this actually failed at scale: Kafka.** Kafka used ZooKeeper from 2011 to ~2022 for
brokers' session registration, partition-replica ISR tracking, controller election, ACLs,
and quotas — all correct recipes, all with the failure modes above, and eventually the
*coordination pattern itself* became the bottleneck: (1) two consistency domains (ZAB for
ZooKeeper's log, Kafka's own replication for data) with hard-to-reason-about interplay;
(2) on controller failover, the new controller had to reload *full cluster metadata* (for
the biggest clusters, ~hundreds of thousands of partitions) from watched-writes into its
own memory — minutes of latency every failover; (3) every metadata write paid a fsync
quorum for a service whose data plane wrote millions of records/s to its own replicated
log — the coordination tier's capacity ceiling sat lower than the data tier's. KIP-500
replaced ZooKeeper with **KRaft**: metadata written to a Raft-ordered, broker-local log,
partitioned and snapshotted in place, so the controller replays its *own* recent log
instead of re-fetching a foreign system's watch stream. The lesson is not "ZooKeeper is
bad" — it is the boundary check: use an external coordination service for *things that
change slowly and are decided rarely* (membership, config, election); once a stream of
decisions itself becomes a high-rate workload with its own ordering needs, it belongs in
your own log ([13.kafka](../13.kafka/04-deep-dives.md) covers its replication design).
The [Bigtable](../12.bigtable/04-deep-dives.md) case draws the same line from the other
side: it kept Chubby-class coordination for assignment and lock semantics, and pushed all
hot metadata (tablet locations) into its own replicated table because Chubby-tier QPS
could not survive hot-path traffic.

- Trade-off (what KIP-500 gave up and gained): external coordination wins an
  already-correct, well-operated, system-independent service and loses latency of
  *cross-system* ordering plus a fragile dependency boundary during failure; moving
  consensus inside wins single-domain reasoning and failover speed and costs the entire
  burden of running a consensus protocol yourself — Kafka judged that trade worth it only
  after ~a decade of ZK operations had made the cost of the external boundary clear.

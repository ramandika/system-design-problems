# Case 38 — LeetCode (Online Judge / Coding Practice Platform)

**The system**: users submit source code to a problem; the platform **compiles it, runs it
inside a sandbox against hidden test cases, scores it, and ships the verdict back** —
interactively in practice mode, and under hard write bursts during contests, where a live
leaderboard of ~150k concurrent participants must stay coherent and anti-cheat has to work
in real time. Think LeetCode / HackerRank / Codeforces: the product *is* the judge.

**Why it's hard**:

- It runs **adversarial, untrusted code by design**. Every submission is an attacker's
  artifact: fork bombs, `fork()` loops, clock tampering, syscalls that read the judge's own
  test data, or code that just allocates 8 GB. The sandbox is the load-bearing component —
  a cousin of the CI runner problem in [Case 04](../04.deployment-system/README.md), but
  stricter: CI tenants are semi-trusted customers; here every run is hostile.
- **Latency is the product.** A judge round-trip much past ~10 seconds strains the
  practice loop (the target is p50 ≤ 5 s, p99 ≤ 15 s — see
  [01](01-requirements.md)), yet each submission needs a language image, a compile, and N
  test-case runs.
  Everything about the worker fleet is warm-pool engineering.
- **Contests are a synchronized write burst**: 150k+ participants all start the clock at
  the same second, submit in waves (each submission triggers leaderboard writes), and
  expect a live top-k ranking that is fair to the second. This inverts the usual
  read-heavy assumption — the hour of a contest looks like a flash-sale write storm.
- **Anti-cheat**: contestants literally collude during the contest, and post-hoc
  similarity checks have to run over every submission without slowing the judge down.

**Reading order**: [01-requirements](01-requirements.md) → [02-scale-estimates](02-scale-estimates.md)
→ [03-architecture](03-architecture.md) → [04-deep-dives](04-deep-dives.md).

**Patterns taught here**: sandboxing untrusted code by layering isolation boundaries
(microVM vs sealed container vs namespace jail), judge workers as a warm-pool queueing
system (the CI-runner pattern of Case 04 applied to 15-second latency budgets), push
delivery of async results (poll vs WebSocket vs SSE, the same decision WhatsApp and Discord
make), top-k leaderboards under write bursts (exact counters vs approximate sketches —
contrast with feed-ranking in [Case 15](../15.ml-ranking-system/README.md)), and abuse
control for a free product with a costly per-request backend.

**Try it first**: before reading 04, answer: (1) a submission calls `sleep(10)` then
outputs a wrong answer — who pays for those 10 seconds, and how do you bill for wall-clock
vs CPU time? (2) the contest starts at 19:30:00 UTC and 500k submissions arrive in the
first minute (stress hypothetical, ~30× the design point of ~16k first-minute
submissions in [02](02-scale-estimates.md)) — do you judge them in FIFO order, and what does that do to participant
experience? (3) the leaderboard says rank 37, the participant's own page says rank 41,
both refreshed "just now" — which design choice made that possible and do you accept it?

**Primary sources**: LeetCode's public stats pages have reported millions of registered
users and contests drawing tens of thousands of participants, with the largest events
exceeding a hundred thousand (order-of-magnitude
anchors only); the gVisor, Firecracker, and nsjail/bwrap projects document their own
isolation trade-offs directly; Codeforces has publicly described its judge and
plagiarism-checking practice (MOSS-derived checks, disqualification waves after contests).
This case is a design exercise: all numbers below are rough estimates, not measurements.

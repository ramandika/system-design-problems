# 01 — Requirements

## Functional scope (functional requirements)

- **Problem bank**: problems with statement, constraints, examples, editorials; per-language
  starter code (function stubs vs full-program modes); difficulty tiers and topic tags.
  Read-only, cached, ~few thousand problems.
- **Submission pipeline**: user submits source in one of ~15–20 supported languages →
  platform compiles (where applicable) → runs against the problem's hidden test cases →
  returns a verdict: Accepted, Wrong Answer, Time Limit Exceeded (TLE), Memory Limit
  Exceeded (MLE), Runtime Error (RE), Compile Error, or Output Limit Exceeded. Verdict
  includes CPU time and peak memory for a correct run.
- **Two execution modes**:
  - *Practice*: a few test cases run eagerly so results come back fast; the full hidden
    suite runs only on demand (an Accepted answer may later be re-validated against the
    full suite without telling the user much more).
  - *Contest*: full hidden suite, strict limits, and all verdicts feed the leaderboard.
- **Contests**: scheduled events (weekly-shape) with a fixed start time, live standings,
  per-problem scoring with penalty time, and post-contest review/plagiarism sweep before
  rating updates.
- **Leaderboards**: global (all-time, per-problem), contest (live), and friends/pagination
  views.
- **Anti-cheat**: similarity detection across submissions (same contest, same era of the
  site), behavioral signals that a submission "knows" hidden answers (ghost submissions —
  see [04](04-deep-dives.md)), rate limits and abuse controls.
- **Sessions**: authenticated users, submission history, streaks/subscriptions — plain
  CRUD off the shelf; not a design focus here.

## Non-functional requirements

| Dimension | Target (design point) | Why |
|---|---|---|
| Judge latency (practice, small test set) | p50 ≤ 5 s, p99 ≤ 15 s | the loop is: submit → fix → submit; any slower and practice stutters |
| Judge latency (contest, full suite) | p50 ≤ 20 s, p99 ≤ 60 s under burst | fairness matters more than speed at contest start |
| Verdict delivery | push, delivered ≤ 2 s after verdict; user may close tab | people navigate away mid-judge |
| Leaderboard freshness (contest) | top-100 visible updates ≤ 10 s; own rank ≤ 30 s | live-feel beats exactness |
| Sandbox guarantees | submission cannot read judge data, exfiltrate via network, or corrupt state; worst case = it wastes its own slot | this is existential, not a trade-off |
| Availability | judge plane degrades gracefully (queue, not fail); web plane 99.9% | a judge outage is "slow", a data loss is real |
| Durability | verdicts & submissions durable forever; live leaderboard is reproducible from the verdict log | verdicts are product history |

## Out of scope

- IDE-in-browser execution of arbitrary user projects (only bounded problem-sized runs) —
  a different product (repl-shaped) with different isolation economics.
- Editorial content, discussion forums, ML recommendations — covered by feed-pattern cases.
- Payments for premium: ordinary billing, nothing architecturally novel.

## The two-sided nature of this system

The interesting requirement tension: the **web/API side** is an ordinary read-heavy, cache-
able product ([Case 10](../10.distributed-cache/README.md) patterns apply). The **judge
side** is a batch job system with an interactive latency budget and a security posture
closer to a malware lab. These meet exactly at the submission queue — that queue's
priorities and warm pools are the design center, the way CI runner scheduling is the
center of [Case 04](../04.deployment-system/01-requirements.md).

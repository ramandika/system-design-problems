# Scoring rubric for a system design exercise

This rubric turns a design exercise (interview answer, design doc, or a study case attempt
from `../study_cases/`) into a score you can compare and map to the level files in this
directory. It exists so the question "at what level is this design work?" gets an answer
that two different evaluators would roughly agree on.

**What it is not**: this scores one exercise. A promotion case is built from repeated,
verified evidence over time (see the level files' Judgment signals) — a rubric score is one
artifact of it, not the case itself.

Its structure borrows from publicly documented practice, roughly: a standardized 0–4 anchored
scale in the style of Google's re:Work structured-interviewing guide (poor / borderline /
solid / outstanding); criteria drawn from the behaviors all reviewed public rubrics repeat —
explicit trade-offs, justified decisions, calculated estimates with context, and failure
modes acknowledged on purpose; failure-mode credit follows MIT 6.033's design-project rubric
("addresses the additional requirement either by solving it or explaining where your system
breaks down"); and the architecture-critique dimensions (trade-off points, risks, sensitivity
points) echo CMU SEI's ATAM.

## The scale — per criterion

Anchored at 1 / 3 / 4. 0 and 2 interpolate by definition, stated once here:

- **0 — absent**: the criterion was not addressed at all.
- **2 — present but weak**: the behavior happened, but single-sided, unjustified, or with
  numbers pulled from air (no derivation, no context).

Level files referenced below: [01](01.associate-engineer.md), [02](02.mid-level-engineer.md),
[03](03.senior-engineer.md), [04](04.principal-engineer.md), [05](05.engineering-manager.md).

### A. Requirements & scope framing

| Score | Anchor |
|---|---|
| 1 | Restates the problem; no scope decisions surface. Reads like 01.md's "reads designs, doesn't write them". |
| 3 | Functional + non-functional requirements separated; scope decisions named ("we do this, explicitly not that"). Matches 02.md's expectation. |
| 4 | Reframes the problem: questions whether the requirement should exist, or defines scope from the constraint side. 04.md's "frames problems" behavior. |

### B. Scale estimates

| Score | Anchor |
|---|---|
| 1 | Numbers with no arithmetic or stated assumptions. |
| 3 | DAU/QPS/storage/bandwidth derived from stated assumptions; the arithmetic is shown and internally consistent. |
| 4 | Uses the estimates to change the design (e.g., "at this QPS a single writer is the bottleneck, so the data model splits") — the math drives decisions, not decoration. |

### C. Architecture & components

| Score | Anchor |
|---|---|
| 1 | A box diagram with unexplained components; technologies named by brand. |
| 3 | Components justified against the requirements; data flows drawn; storage chosen by access pattern (keys, indexes, partitioning), not by brand. |
| 4 | Boundaries chosen with the org in mind (team ownership, failure isolation, what a team of ~10 can own) — 04.md's second-order-effects behavior. |

### D. Explicit trade-offs

| Score | Anchor |
|---|---|
| 1 | One solution presented as the only conceivable one. |
| 3 | States "Option A wins X, loses Y, we pick A because Z" — alternatives named and the choice justified from the requirements. |
| 4 | Surfaces hidden dimensions of the trade-off (consistency vs availability *under this access pattern*, cost to operate, reversibility) and names which dimension dominates. |

### E. Deep dives & failure modes

| Score | Anchor |
|---|---|
| 1 | Happy path only; failures unaddressed or hand-waved ("we cache it"). |
| 3 | Hard sub-problems identified and worked through with degradation behavior: what breaks first under overload, what the failure does to data. |
| 4 | Explicitly states *where the design breaks down* and what it would take to fix — acknowledged limits score, they don't deduct (MIT 6.033 criterion). |

### F. Judgment: smallest design that works

| Score | Anchor |
|---|---|
| 1 | Overbuilt: components no stated requirement calls for. |
| 3 | Smallest design that meets the requirements; anything skipped is named as skipped. Matches the EM/Senior interface norm in 05.md. |
| 4 | Deletes requirements or components the problem didn't need — a decision that removes work, not adds it (04.md's "deletes work" behavior). |

### G. Operations & cost

Amazon's design round (per candidate-reported accounts of interviewer behavior) probes how a
design is *run*, not just drawn — "run what you build" is the premise. The recurring probes,
useful as sub-anchors everywhere:

detection (which metrics alarm at 3 a.m.), blast radius (what a bad deploy takes down and
what limits it), slow-then-degraded dependencies (timeout, backoff, what is served instead),
staged deployment and rollback, data repair after a bug (replay from log/queue), and cost
(which component dominates the bill, and the cheaper design at 10x smaller scale).

| Score | Anchor |
|---|---|
| 1 | Reliability hand-waved ("we cache it"); cost never mentioned. |
| 3 | Answers most of the probes: names metrics/alarms, degradation behavior for slow dependencies, and a rollout/rollback story. |
| 4 | Cost shapes the design: names the dominant cost component and the signal that justifies adding the next component — rather than adding everything upfront. |

## Mapping scores to levels

Rule: the level read is set by the **lowest** of criteria D, E, and G plus the level's
baseline profile — trade-off articulation, failure-mode honesty, and operational reality are
the load-bearing criteria; a 4 on estimates with a 1 on trade-offs or ops is not a Senior
performance. This is a weakest-link rule, chosen for calibration: it makes a single strong
dimension unable to mask the next level's required behavior.

| Level (file) | Expected profile | One-line read |
|---|---|---|
| [01 Associate](01.associate-engineer.md) | 1–2 on most, a 3 on A or B *with guidance* | Consumes a design more than it produces one |
| [02 Mid](02.mid-level-engineer.md) | 2 across the board, 3 on A and B | Sound within an established design |
| [03 Senior](03.senior-engineer.md) | 3 across the board; at least one 4 on D or E | Defends trade-offs and failure modes unprompted |
| [04 Principal](04.principal-engineer.md) | 4 on A (framing) and one of D/E; 3 elsewhere | The problem statement itself is part of their output |
| [05 EM](05.engineering-manager.md) | scored as an evaluator, not an author | Uses these anchors to question someone else's design (see below) |

The EM row works differently: an EM is expected to *evaluate, not necessarily author*
(05.md). Their exercise is scored by whether their questions align with these anchors — what
breaks first, what does it cost, what was NOT considered, is the scope smallest? A design
doc review with all 0–4 anchors addressed in comments is the EM-equivalent 3.

## Calibration guidance

- Score all criteria **before** looking at the level table; map afterwards. The anchors are
  per-criterion, not per-level.
- A single exercise rarely measures all of A–G at a level's full depth. Treat one score as a
  signal to investigate, and a promotion case as the aggregate of many.
- 04.md's Judgment signals section warns that half its evidence can't be cited from one
  sitting. This rubric covers the authored-design half (A, D, F); the influence half (rooms
  they don't control, written decisions that outlive meetings) still needs real artifacts.
- **Level-altitude, not prompt-difficulty**: the same prompt is graded from every level;
  what changes is the expected altitude. Candidate-reported Amazon framing: a mid-level
  performance assembles the core components correctly, while a senior performance also
  raises security, testability, cross-team API surface, and org-level scale unprompted —
  this mirrors 02 → 03 in the level files. Score the altitude of what was volunteered,
  not the number of topics touched.
- **Alternative to weakest-link — weighted sheet**: a vendor template (Ryz Labs)
  multiplies per-criterion 1–4 scores by weights (requirements 15%, high-level design 20%,
  data model 15%, trade-offs 20%, reliability 15%, communication 15%) and applies a decision
  rule: proceed at ≥3.0 with no 1s, discuss 2.6–2.9. Pick one rule and write it down; two
  evaluators using different aggregation rules will disagree about candidates, not designs.
- **Red flags in scoring itself** (same template, consistent with this repo's rules): never
  reward vocabulary — naming Kafka/Redis is not design, score how data moves and why; don't
  let the evaluator drive the exercise — note how much prompting was needed, because a
  performance shaped by prompting is a lower-level one (this is exactly 01's "with
  guidance" vs 02's "unprompted"); and beware memorized answers to famous prompts
  ("design Twitter") — prefer a prompt close to a real system in `../study_cases/`.
- On why a structured rubric at all: the defensible justification is **inter-rater
  consistency and fairness** — two evaluators scoring the same exercise should land within
  a point of each other. Public psychometric evidence for structured over unstructured
  interviews has been contested by later reanalyses (McDaniel et al. 1994 reported .44 vs
  .33 job-performance validity, but Oh et al. 2013 showed the gap can reverse under better
  bias corrections), so cite consistency, not prediction, when justifying the format.

## Sources (public, verified 2026-11)

- MIT 6.033 Spring 2018 design-project grading rubric (design 40 / justification 35 /
  presentation 25; failure-mode credit): ocw.mit.edu —
  `6-033-computer-system-engineering-spring-2018 ... MIT6_033S18dpr_rubric.pdf`
- CMU SEI, Architecture Tradeoff Analysis Method (trade-off points, risks, utility tree):
  sei.cmu.edu/library/architecture-tradeoff-analysis-method-collection/
- Google re:Work, structured interviewing guide (standardized rubrics, 4 anchored levels,
  judging analytical thought process): rework.withgoogle.com —
  "a-guide-to-structured-interviewing-for-better-hiring-practices"
- SJSU BSCS assessment rubrics (1–3 per-indicator scale, trade-off comprehension as an
  assessed outcome): sjsu.edu/cs/about-us/assessments/assessment-rubrics-bscs.php
- McDaniel, Whetzel, Schmidt & Maurer (1994); Oh, Postlethwaite & Schmidt (2013) reanalysis —
  validity magnitudes and their caveat, as summarized above.
- Candidate-reported accounts of interview loops (blog-grade; directionally useful about
  *how the round is run*, not citable as company policy): techinterview.coach,
  "Amazon System Design Interview: What the Round Tests" (round shape, ops/cost probes,
  written-feedback expectations, level-altitude framing), Nov 2026.
- Ryz Labs, "System design interview rubric (scoring template)" — six weighted dimensions,
  per-level anchors, decision rule, scoring red flags (vendor template, blog-grade).

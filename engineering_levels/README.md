# Engineering Levels

Competency definitions for five engineering levels. Each level is defined by **scope**,
**judgment**, and **independence** — not years of service. Use these files to assess system
design competency, set growth goals, and calibrate promotion cases.

## The ladder

| # | Level | One-line essence |
|---|------|------------------|
| 01 | [Associate Engineer](01.associate-engineer.md) | Delivers well-defined tasks with feedback loops close by |
| 02 | [Mid-Level Engineer](02.mid-level-engineer.md) | Owns features end-to-end; designs within an established system |
| 03 | [Senior Engineer](03.senior-engineer.md) | Owns systems and their trade-offs; designs beyond a single service |
| 04 | [Principal Engineer](04.principal-engineer.md) | Owns direction across systems and orgs; decides what gets designed |
| 05 | [Engineering Manager](05.engineering-manager.md) | Owns outcomes through people, systems, and priorities — reached as a role change from the IC track, not a promotion |

## How to read a level file

Every level file has the same six sections, so levels can be diffed directly:

1. **Scope** — what slice of the system/org this person is accountable for
2. **System design competency** — what they can design, and at what correctness level
3. **Technical execution** — the hands-on skills expected
4. **Judgment signals** — observable behaviors that distinguish this level from the one below
5. **Not expected** — anti-requirements; what should NOT be graded at this level
6. **Growth to next level** — the gap that has to close

## The one-axis summary

Across levels, the **size of the design problem** grows (task → feature → system → portfolio
of systems) and the **cost of being wrong** grows (a bad line of code → a bad system design →
a bad multi-year direction). The EM changes the axis rather than extending it: the unit of
leverage is people and priorities, and the cost of being wrong is stunted people and lost
quarters. Everything else in these files follows from that.

## How they connect to the study cases

- [01.youtube](../study_cases/01.youtube/README.md), [02.twitter](../study_cases/02.twitter/README.md),
  [03.whatsapp](../study_cases/03.whatsapp/README.md) are level-anchor material: an Associate
  should read them to learn; a Mid should reproduce them; a Senior should redesign one
  trade-off and defend it; a Principal should challenge the problem framing itself.

To score a design attempt against the levels (interview, design doc, or a study-case
exercise), use [rubric.md](rubric.md) — per-criterion 0–4 anchors mapping to these files.

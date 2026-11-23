# CLAUDE.md — conventions for this knowledge base

This repo is a system design knowledge base. When adding or editing content here:

## File layout (fixed)

Every study case lives in `study_cases/NN.<kebab-name>/` and contains exactly:

1. `README.md` — overview: what the system does, why it's hard, reading order
2. `01-requirements.md` — functional + non-functional requirements, scope decisions
3. `02-scale-estimates.md` — back-of-envelope math (DAU, QPS, storage, bandwidth)
4. `03-architecture.md` — high-level diagram (ASCII or Mermaid), components, data flows
5. `04-deep-dives.md` — 3–6 hard sub-problems, each with trade-offs and failure modes

Number cases `01.`, `02.`, ... — never renumber existing ones; append the next number.

## Engineering levels (`engineering_levels/`)

Six files: `README.md` (ladder overview) + one file per level, named
`NN.<kebab-level>.md`. Every level file uses the same fixed sections — Scope, System design
competency, Technical execution, Judgment signals, Not expected, Growth to next level — so
levels remain directly diffable. Levels are defined by scope/judgment/independence, never by
years of experience. When editing one level file, keep the "step-up from the level below"
consistent with that neighbor file's definition.

## Content rules

- Numbers may be rough but must be internally consistent: derived values (QPS, storage/day)
  must follow from the stated assumptions. Show the arithmetic.
- Every deep dive must state the trade-off explicitly: "Option A wins X, loses Y, we pick
  A because Z" — never present a design decision as if there were only one choice.
- Write from knowledge, don't invent fake statistics; cite public numbers as "roughly"
  or "publicly reported ~X" and label estimates as estimates.
- Cross-reference other cases with relative links like `../02.twitter/04-deep-dives.md`
  when the same pattern appears (fanout, ID generation, CDN, delivery guarantees).
- Keep files Markdown, plain ASCII diagrams or Mermaid blocks, no generated images.
- Language: English, direct technical prose. No filler.

## Non-goals

- No code implementations, no Dockerfiles, no charts-as-images. This is a design text corpus.

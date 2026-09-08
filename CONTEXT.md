# Context

Shared vocabulary for this repo. Skills and issues use these terms with these meanings.

## Skill kinds

- **rubric skill** — a passive checklist of judgement calls on one axis. It does not run a
  procedure; it gives the reviewer criteria and before/after examples. `rust-error-design`,
  `rust-idioms`, `rust-async`, `rust-api-design`.
- **workflow skill** — an active procedure with steps and gates. `rust-review` (apply the
  rubrics to a diff), `skill-maintainer` (fold learnings into rubrics).

## The clippy line

The boundary that decides whether a point belongs in a rubric skill:

> If `cargo clippy` (with `pedantic`) already flags it, it does not go in a skill **unless**
> the skill adds a design rationale clippy cannot express.

"Prefer `?` over `match ... return`" is below the line (clippy has it). "This error enum is
not a usable API because the caller can't tell a retryable failure from a permanent one" is
above it.

## The learnings loop

- **learning** — one field correction: `avoid X` / `prefer Y` / `why`. Captured in
  `LEARNINGS.md` from real work in a downstream project.
- **capture** — appending a learning to `LEARNINGS.md`. Cheap, no judgement, safe mid-task.
  Downstream projects do this with their own `/learn` skill.
- **fold** — promoting a captured learning into a rubric skill's body and checklist, then
  bumping the plugin version and recording it in `CHANGELOG.md`. Done by `skill-maintainer`,
  batched, behind a review gate.
- **status** of a learning: `pending` → `folded v<x.y.z>` or `rejected (<reason>)`.

## House style for skill bodies

- Every point: the non-idiomatic version, the idiomatic version, and the counter-case where
  the "wrong" one is right.
- Every skill ends with a `## Review checklist` of mechanical `- [ ]` items.
- No em dashes. No decorative horizontal rules in prose.
- Error messages and identifiers in examples follow the conventions the skill teaches.

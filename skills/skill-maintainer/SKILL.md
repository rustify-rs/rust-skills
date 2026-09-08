---
name: skill-maintainer
description: >
  Fold pending entries from LEARNINGS.md into the rubric skills: for each entry decide
  promote / merge / reject, edit the target SKILL.md body and checklist in house style,
  bump the plugin version, record it in CHANGELOG.md, and mark the entry folded. Batched
  and behind a review gate: it never edits a skill without showing the plan first. Use when
  the user runs /rust-skills:skill-maintainer or asks to fold learnings / update the skills
  from the inbox.
disable-model-invocation: true
---

# Skill Maintainer

The stage-2 half of the learnings loop. Stage 1 (capture) appends raw entries to
`LEARNINGS.md`. This skill turns the good ones into durable rubric content.

## Process

### 1. Collect

Read `LEARNINGS.md`. Take every entry with `status: pending`. Ignore `example`, `folded`,
`rejected`. If there are none, say so and stop.

### 2. Classify each entry

| Decision | When | Action |
|---|---|---|
| **promote** | a general Rust judgement no rubric states yet | add a new point to the target rubric |
| **merge** | refines or sharpens a point a rubric already makes | edit the existing point, don't add a second |
| **new-skill** | a coherent axis no rubric covers (e.g. macros, unsafe) | add a row to the README roadmap, keep the entry `pending` with a note |
| **reject** | one-off project quirk, contradicts an existing point without justification, already fully covered, or has no clear before/after | set `status: rejected (<reason>)` |

Reject freely. The rubrics stay sharp by keeping weak learnings out, not by absorbing all
of them.

### 3. Show the plan, wait

Print a table before touching any file:

```
entry (date + title)        decision   target skill        note
--------------------------   --------   -----------------   ----------------------
2026-09-10 io error from     merge      rust-error-design   sharpen the #[from] point
2026-09-11 spawn in Drop     promote    rust-async          new checklist item
2026-09-12 our repo layout   reject     -                   project-specific
```

Do not proceed until the user confirms. They may override any row.

### 4. Apply

For each promote / merge:

- Edit the rubric `SKILL.md`. Match house style exactly (see `CONTEXT.md`): the
  non-idiomatic version, the idiomatic version, the counter-case. No em dashes.
- Update that rubric's `## Review checklist` with a matching `- [ ]` line.
- Keep the file under ~250 lines. If it would overflow, that is a `new-skill` signal
  instead; take it back to the user.

### 5. Version and record

- Bump `.claude-plugin/plugin.json` `version`: minor if any point was promoted, patch if
  every change was wording only.
- Add a `CHANGELOG.md` entry under a new version heading, one bullet per rubric touched,
  naming the learning.
- In `LEARNINGS.md`, set each applied entry's `status:` to `folded v<x.y.z>`. Leave the
  entry text itself unchanged.

### 6. Commit

```
chore(skills): fold learnings v<x.y.z>
```

Path-scoped to the files actually changed. Tag `v<x.y.z>` (annotated). Push. Tell the user
downstream projects pick it up with `/plugin update`.

### 7. Report

The final table: each entry, its decision, the skill and checklist line it became, the new
version.

## Boundaries

Only touches this repo's `skills/`, `plugin.json`, `CHANGELOG.md`, `LEARNINGS.md`. Never
edits a rubric without step 3 confirmation. Does not invent learnings that are not in the
inbox.

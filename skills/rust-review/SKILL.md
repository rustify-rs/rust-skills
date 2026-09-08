---
name: rust-review
description: >
  Review a Rust diff or file by applying the rubric skills (rust-error-design, rust-idioms,
  rust-async, rust-api-design) that are relevant to what changed, then report findings as
  one line each, severity-tagged, grouped by nothing but severity. Use when the user asks
  to review Rust changes, a Rust PR, a branch, or a single Rust file, or says "rust review"
  / invokes /rust-skills:rust-review. Reviews only: does not edit code and does not run
  clippy.
---

# Rust Review

Orchestrates the rubric skills over a concrete diff. Each finding cites the rubric it came
from so the author can go read the reasoning.

## Process

### 1. Pin what is under review

- A ref, ref range, or merge base the user gave: `git diff <base>...HEAD`.
- `--staged` / working tree: `git diff` or `git diff --staged`.
- A path or file argument: review that file as it stands.
- Nothing given: `git diff` against the merge base with the default branch, and say so.

Confirm the ref resolves and the diff is non-empty before going further. Collect the
changed `.rs` files and the hunks.

### 2. Select rubrics by what changed

| Signal in the diff | Rubric to apply |
|---|---|
| always, on any Rust change | `rust-idioms` |
| an `Error` enum, `Result` signature, `?`, `map_err`, `From` for an error | `rust-error-design` |
| `async fn`, `.await`, `tokio::`, `select!`, `spawn`, `Stream` | `rust-async` |
| a change to a `pub` item, trait, or `Cargo.toml` `[features]` / `rust-version` | `rust-api-design` |

Load each selected rubric's `SKILL.md` and run its `## Review checklist` against the hunks.
If rubrics run as parallel sub-agents, give each one only its rubric plus the diff, then
aggregate here.

### 3. Report

One line per finding:

```
<path>:<line>: <emoji> <severity>: <problem>. <fix>. [<rubric>]
```

Severity:

- `🔴 bug` behaviour is wrong or will break (panic on input, cancel-unsafe state, deadlock)
- `🟡 risk` compiles and usually works, but fragile or a semver trap
- `🔵 nit` style or shape; author may ignore

No praise lines. No restating what the code does. If the fix is not obvious from the
problem, add the *why* in the same sentence, not a paragraph.

End with a checklist summary, one line per applied rubric:

```
rust-error-design  3 findings (1🔴 2🟡)
rust-idioms        1 finding  (1🔵)
rust-async         clean
rust-api-design    2 findings (2🟡)
```

### 4. When a finding is not covered by a rubric

If you spot a real problem no rubric addresses, report it under severity with `[uncovered]`
instead of a rubric name, and tell the user it is a candidate for `/learn` so it can be
folded into a rubric later.

## Boundaries

Review only. Does not edit code, does not run `cargo clippy` or `cargo check` (tell the
user to run both separately), does not approve or merge. Output is the finding list, ready
to paste into a PR.

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

### 5. HTML report (optional)

Only when the user asks for a report (`--report`, "html report", "generate the report").
The terminal finding list from step 3 is still the primary output; this is a second copy.

Pattern mirrors unlighthouse: a hidden dir at the root of the reviewed repo, a static shell
copied verbatim, the run's data written next to it.

1. In the reviewed repo, create `.rust-review/`.
2. Copy this skill's `report/template.html` to `.rust-review/index.html` unchanged.
3. Build the run data:

   ```json
   {
     "meta": {
       "target": "<what was reviewed, e.g. src/domain/google_calendar/>",
       "base": "<ref range or 'working tree' or 'file'>",
       "generated": "<YYYY-MM-DD>",
       "reproduce": "/rust-skills:rust-review <args> --report"
     },
     "findings": [
       { "id": "F-01", "sev": "bug", "rubric": "rust-idioms", "loc": "client.rs:250",
         "problem": "<inline HTML, use <code>...</code>>", "fix": "<inline HTML>",
         "before": "<raw Rust>", "after": "<raw Rust>", "note": "<inline HTML, optional>" }
     ]
   }
   ```

   `sev` is `bug` / `risk` / `nit`. `rubric` is the rubric name or `uncovered`. `before` and
   `after` are optional and come as a pair; include them only where the fix is mechanical.
   Give findings stable ids in report order (`F-01`, `F-02`, ...).

4. Write that JSON two ways:
   - to `.rust-review/findings.json`
   - inline into `.rust-review/index.html`, replacing the one line inside
     `<script id="rust-review-data" type="application/json">`.

   The inline copy makes double-click work; `findings.json` is for diffing and re-render.
5. Ensure the reviewed repo ignores it: add `.rust-review/` to its `.gitignore` if absent.
6. Tell the user the path and that `open .rust-review/index.html` shows it.

Do not invent findings to fill the report. It contains exactly what step 3 reported.

## Boundaries

Review only. Does not edit code (the HTML report is the one file it writes, and only on
request), does not run `cargo clippy` or `cargo check` (tell the user to run both
separately), does not approve or merge. Primary output is the finding list, ready to paste
into a PR.

# Learnings inbox

Append-only. Each entry is one field correction captured from real work in a downstream
project. Entries are folded into rubric skill bodies later by the `skill-maintainer` skill.

**Do not rewrite or delete past entries.** The only edit allowed on an existing entry is
setting its `status:` line when it is folded or rejected.

Newest entries go at the top of `## Entries`.

## Entry format

```
### YYYY-MM-DD  short title
- **skill:** rust-<name>   (or `new` if no rubric covers this yet)
- **avoid:** the pattern to stop doing
- **prefer:** the pattern to do instead
- **why:** the reason, naming the concrete failure it prevents
- **source:** <project> - <session URL or commit SHA>
- **status:** pending
```

`status` values: `pending`, `folded v<x.y.z>`, `rejected (<reason>)`, `example`.

## How it flows

```
downstream project session
   you correct a pattern
        |
        v  /learn "..."   (downstream project's own skill)
   append entry here  (status: pending)
        |
        v  /rust-skills:skill-maintainer   (batched, review gate)
   promote / merge / reject each pending entry
   edit the rubric SKILL.md + its checklist
   bump plugin.json version, add CHANGELOG line
   set entry status: folded v<x.y.z>
        |
        v  git push  ->  downstream `/plugin update`
```

## Entries

### 2026-09-08  example entry, safe to delete once real ones exist
- **skill:** rust-error-design
- **avoid:** `#[error("db error: {0}")]` on a variant that also has `#[from] sqlx::Error`
- **prefer:** `#[error("database error")]` with the source carried by `#[from]`; let the
  chain printer add the cause
- **why:** the cause is printed twice in `{:?}` output and once more by any handler that
  walks `source()`, so logs show `db error: db error: connection refused`
- **source:** (none, illustrative)
- **status:** example

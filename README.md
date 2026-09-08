# rust-skills

Opinionated [Agent Skills](https://docs.claude.com/en/docs/claude-code/skills) for writing
and reviewing **good** Rust, not just compiling Rust.

`cargo clippy` already checks the mechanics. These skills check the decisions clippy can't
see: how your error type reads as an API, when a module should become a crate, whether an
`async fn` is cancel-safe, how a public signature ages across releases.

## Why this exists

The Rust toolchain is excellent at "is this correct". It says little about "is this a good
interface". Most real review time goes to the second question, and it's the part an AI
agent gets wrong most often: agents reach for `unwrap`, `String` errors, `Box<dyn Error>`,
one giant `lib.rs`, and `Arc<Mutex<_>>` where a channel belonged.

Each skill encodes a reviewer's judgement on one axis, with before/after examples and a
checklist an agent can actually apply.

## Install

### As a plugin (recommended)

```
/plugin marketplace add rustify-rs/rust-skills
/plugin install rust-skills@rustify-rs
```

Skills are then namespaced: `/rust-skills:rust-review`, `/rust-skills:skill-maintainer`.
The rubric skills trigger automatically on matching Rust work. Pull later improvements with
`/plugin update`.

### Manually

```bash
git clone https://github.com/rustify-rs/rust-skills.git
cp -r rust-skills/skills/* ~/.claude/skills/
```

## Skills

### Rubric skills (model-invoked)

A rubric is a checklist of judgement calls on one axis, with before/after examples and a
counter-case for every rule.

| Skill | Question it answers | Status |
|---|---|---|
| `rust-error-design` | Can the caller *do* anything with what you return? | ✅ ready |
| `rust-idioms` | The shape choices clippy stays silent on (`&str`, newtype, `Cow`, enum params) | ✅ ready |
| `rust-async` | What happens when this future is dropped mid-poll? | ✅ ready |
| `rust-api-design` | What breaks downstream when you change this `pub` item? | ✅ ready |
| `rust-crate-structure` | When does this module become a crate? Feature-flag layout. | 🚧 planned |
| `rust-testing` | `nextest`, fixtures, unit vs integration boundary, `matches!` asserts | 🚧 planned |
| `rust-perf` | Gratuitous `clone`, allocation in hot paths, `Cow`, when to bench | 🚧 planned |

### Workflow skills

| Skill | Does | Invoke |
|---|---|---|
| `rust-review` | Pins a diff or file, applies the relevant rubrics, reports findings one line each | model or `/rust-skills:rust-review` |
| `skill-maintainer` | Folds `LEARNINGS.md` entries into rubric bodies, bumps the version | `/rust-skills:skill-maintainer` |

Seeded deliberately small. A great skill beats three mediocre ones.

## Self-improving

The rubrics are meant to get sharper from real use, not just from up-front authoring.

```
downstream project session
   you correct a Rust pattern in review
        |
        v  /learn "avoid X, prefer Y, because Z"     (the project's own capture skill)
   entry appended to LEARNINGS.md   (status: pending, no judgement, safe mid-task)
        |
        v  /rust-skills:skill-maintainer             (batched, review gate)
   promote / merge / reject each entry
   edit the rubric SKILL.md + its checklist
   bump plugin.json version, add CHANGELOG line, mark entry folded
        |
        v  git push  ->  downstream runs /plugin update
   the rubric is now sharper everywhere it's installed
```

Two stages on purpose: appending a raw learning is cheap and safe to do in the middle of
other work; editing a published rubric needs a review pass so weak or contradictory
learnings don't accumulate. See [`LEARNINGS.md`](LEARNINGS.md) for the entry format and
[`CONTEXT.md`](CONTEXT.md) for the vocabulary.

Downstream projects need their own tiny capture skill that appends to this repo's
`LEARNINGS.md`. A reference implementation lives in `RUSTIFY-APP` at
`.claude/skills/learn/`.

## Design principles

1. **Differentiate from clippy.** If a skill just restates a lint, it doesn't ship. See
   "the clippy line" in [`CONTEXT.md`](CONTEXT.md).
2. **Judgement, not rules.** Every guideline carries the *why* and a counter-case.
3. **Before / after.** Every point shows the bad version and the good version.
4. **Checklist at the end.** An agent needs something mechanical to run last.
5. **Sharpen from use.** Corrections from real reviews get folded back in, versioned.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). New skills start as an issue describing the axis of
judgement and two or three before/after pairs.

## License

Dual-licensed under [MIT](LICENSE-MIT) and [Apache 2.0](LICENSE-APACHE), at your option.

## Maintainers

Maintained by [rustify.rs](https://rustify.rs).

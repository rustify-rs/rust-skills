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

Drop a skill directory into `~/.claude/skills/` (user-wide) or `.claude/skills/` (per
project):

```bash
git clone https://github.com/rustify-rs/rust-skills.git
cp -r rust-skills/skills/rust-error-design ~/.claude/skills/
```

Claude Code picks it up on next launch. Invoke explicitly with `/rust-error-design` or let
it trigger on matching work.

## Skills

| Skill | Question it answers | Status |
|---|---|---|
| `rust-error-design` | Can the caller *do* anything with what you return? | ✅ ready |
| `rust-api-design` | Will this public signature still be right in 6 months? | 🚧 planned |
| `rust-idioms` | The mechanics clippy misses (`Cow`, `impl Trait`, newtype, `matches!`) | 🚧 planned |
| `rust-async` | Is this `async fn` cancel-safe? `spawn` or `block_in_place`? | 🚧 planned |
| `rust-crate-structure` | When does this module become a crate? Feature-flag layout. | 🚧 planned |
| `rust-testing` | `nextest`, fixtures, unit vs integration boundary, `matches!` asserts | 🚧 planned |
| `rust-perf` | Gratuitous `clone`, allocation in hot paths, `Cow`, when to bench | 🚧 planned |

Seeded deliberately small. A great skill beats three mediocre ones.

## Design principles

1. **Differentiate from clippy.** If a skill just restates a lint, it doesn't ship.
2. **Judgement, not rules.** Every guideline carries the *why* and a counter-case.
3. **Before / after.** Every point shows the bad version and the good version.
4. **Checklist at the end.** An agent needs something mechanical to run last.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). New skills start as an issue describing the axis of
judgement and two or three before/after pairs.

## License

Dual-licensed under [MIT](LICENSE-MIT) and [Apache 2.0](LICENSE-APACHE), at your option.

## Maintainers

Maintained by [rustify.rs](https://rustify.rs).

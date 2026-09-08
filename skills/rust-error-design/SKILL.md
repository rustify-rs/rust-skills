---
name: rust-error-design
description: >
  Review and design Rust error types as a public API, not just plumbing. Covers thiserror
  vs anyhow, one-enum-per-boundary, source-chain preservation, structured payloads over
  strings, non_exhaustive, retryability, panic vs Result, and expect-message discipline.
  Use when writing or reviewing error enums, Result signatures, `From` impls, `?` chains,
  error handling in a library or service, or when the user says "error handling", "error
  types", "thiserror", "anyhow", "how should this fail". Not a lint pass: judges whether
  the error type is a good interface.
---

# Rust Error Design

Clippy checks whether your `?` compiles. This checks whether the caller can *do* anything
with what you return.

An error type is an API surface. Callers match on it, log it, decide whether to retry, and
surface it to users. Design it with the same care as the success type.

## Decision: what kind of error

| Context | Error strategy |
|---|---|
| Library crate, public API | Typed enum. `thiserror` or hand-rolled. Never `anyhow` in the signature. |
| Binary / service top layer | `anyhow` / `eyre` with `.context()` at every boundary. |
| Internal module inside an app | Typed enum if callers branch on it, else bubble up to the app's `anyhow`. |
| Invariant that must hold | `panic!` / `expect` / `unreachable!`. Not `Result`. |

Rule: `Result` is for failures the caller might reasonably expect and handle. `panic` is for
bugs. "File not found" is a `Result`. "I already validated this index two lines ago" is an
`expect`.

## Typed errors (libraries)

### One enum per boundary, not per function

Define one error type per crate, or per major module that callers cross. Not one per
function. Ten single-variant error enums is worse than one ten-variant enum: the caller
writes ten `match`es and ten `From` impls.

```rust
// good: one type at the crate boundary
#[derive(Debug, thiserror::Error)]
#[non_exhaustive]
pub enum Error {
    #[error("user {id} not found")]
    UserNotFound { id: UserId },

    #[error("invalid credentials")]
    BadCredentials,

    #[error("database error")]
    Database(#[from] sqlx::Error),

    #[error("token expired at {expired_at}")]
    TokenExpired { expired_at: OffsetDateTime },
}

pub type Result<T, E = Error> = std::result::Result<T, E>;
```

### `#[non_exhaustive]` on every public error enum

Adding a variant is otherwise a breaking change. `#[non_exhaustive]` forces downstream
`match` to keep a `_ =>` arm and lets you grow the enum in a minor release.

### Preserve the source chain

Every wrapping variant carries the cause with `#[from]` or `#[source]`. Never flatten a
cause into a string:

```rust
// bad: cause is now unrecoverable text, no backtrace, no downcast
#[error("database error: {0}")]
Database(String),

// good: full chain, caller can downcast, `{:?}` prints the whole chain
#[error("database error")]
Database(#[source] sqlx::Error),
```

`#[from]` implies `#[source]` and generates the `From` impl. Use plain `#[source]` when you
want the conversion to be explicit at the `?` site (see next point).

### Do not `#[from]` everything

`#[from]` is a blanket, context-free conversion. If two different operations both produce
`std::io::Error`, a single `#[from] io::Error` variant erases which one failed. Prefer a
named variant plus `.map_err`:

```rust
let config = fs::read_to_string(&path)
    .map_err(|e| Error::ReadConfig { path: path.clone(), source: e })?;
```

Reserve `#[from]` for one unambiguous lower layer (e.g. your DB crate).

### Structured fields, not formatted strings

Put the id, the path, the count in a field. Format it only in the `#[error("...")]`
string. A caller that needs the `UserId` back should not parse your error message.

```rust
// bad
#[error("user {0} not found")]
UserNotFound(String),

// good
#[error("user {id} not found")]
UserNotFound { id: UserId },
```

### Expose retryability, don't make callers guess

If some variants are transient, say so on the type instead of forcing a string match:

```rust
impl Error {
    pub fn is_retryable(&self) -> bool {
        matches!(
            self,
            Error::Database(sqlx::Error::PoolTimedOut)
                | Error::Database(sqlx::Error::Io(_))
        )
    }
}
```

Same pattern for `is_not_found()`, `status_code()`, `kind()` — whatever callers branch on.

### No `Box<dyn Error>` in a library's public signature

It is not `Send + Sync` by default, callers cannot `match`, and it advertises "I did not
design this". Fine as an internal implementation detail, never in the `pub fn` return type.

## Untyped errors (applications)

### `anyhow` + context at every boundary

`anyhow::Error` is right for the top of a binary. The value is `.context()`: each `?` that
crosses a layer adds a frame.

```rust
let raw = fs::read(&path)
    .with_context(|| format!("reading snapshot {}", path.display()))?;
let snapshot: Snapshot = serde_json::from_slice(&raw)
    .with_context(|| format!("parsing snapshot {}", path.display()))?;
```

Without context: `invalid type: string "x", expected u64`. With it: you know which file.

Use `with_context` (closure, lazy) when the message needs formatting; `context` (eager)
for a literal.

### Don't force `anyhow` on your library's users

A library returning `anyhow::Result` pushes an opinion and a dependency onto everyone.
Return a typed error; let the application wrap it in `anyhow` if it wants.

## Cross-cutting rules

### Handle or propagate — never log-and-return

```rust
// bad: this error gets logged here, then again by the caller, then again at the top
Err(e) => {
    tracing::error!("failed to load user: {e}");
    return Err(e);
}
```

Log where the error is *handled* (turned into a response, retried, swallowed). In between,
just `?`. One error, one log line.

### `expect`, not `unwrap`; message states the invariant

`expect` message describes *why this cannot fail*, not what failed:

```rust
// bad
let port = env::var("PORT").unwrap();
// bad
let port = env::var("PORT").expect("failed to get PORT");
// good
let port = env::var("PORT").expect("PORT is set by the launcher; see deploy.sh");
```

Read as: "panicked because {message}". Zero `unwrap`/`expect` on external input in library
code — that's a `Result`.

### Error message style (Rust convention, clippy-adjacent but not enforced)

- lowercase first letter, no trailing period
- describe the failure, not the function: `"connection refused"`, not `"connect() failed"`
- no `"Error: "` prefix — the printer adds context
- no redundant cause: `#[error("database error")]` not `#[error("database error: {0}")]`
  when `{0}` is already the `#[source]` (double-printed in the chain)

### Tests match on variants, not strings

```rust
// bad: breaks when you reword the message
assert!(err.to_string().contains("not found"));
// good
assert!(matches!(err, Error::UserNotFound { .. }));
```

## Review checklist

- [ ] Public error enum is `#[non_exhaustive]`
- [ ] Every wrapping variant has `#[source]` / `#[from]` — no `String` causes
- [ ] Payloads are structured fields, not pre-formatted into the message
- [ ] No `#[from]` on an ambiguous type (`io::Error`, `String`, `anyhow::Error`)
- [ ] No `anyhow` / `Box<dyn Error>` in a library's public signature
- [ ] Transient failures are queryable (`is_retryable`, `kind`, `status_code`)
- [ ] No log-and-return; logging only at the handling site
- [ ] `expect` messages state the invariant; no `unwrap` on external input
- [ ] Messages: lowercase, no trailing period, no `Error:` prefix, no doubled cause
- [ ] Error assertions in tests use `matches!`, not `.to_string().contains`

## Boundaries

Design and review only. Does not run `cargo clippy` or `cargo check`. Pairs with
`rust-api-design` (the success half) and `rust-idioms` (the mechanics).

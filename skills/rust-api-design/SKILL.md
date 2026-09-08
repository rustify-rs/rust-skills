---
name: rust-api-design
description: >
  Review a Rust crate's public surface for how it ages across releases. Covers accept-
  generic/return-concrete, `#[non_exhaustive]`, leaking dependency types into your API
  (semver coupling), builders over long argument lists, sealed traits, returning `Result`
  before you need it, `dyn` vs generics at the boundary, additive-only feature flags, and
  MSRV as API. Use when designing or reviewing `pub` items, trait definitions, a crate
  boundary, or a pre-1.0 stabilisation pass. Not a lint pass: reasons about the next
  breaking change you'll be forced into.
---

# Rust API Design

Every `pub` item is a promise. This skill asks, for each one: what breaks downstream when
you need to change it in six months?

## Accept generic, return concrete

```rust
// rigid input, leaky output
pub fn new(name: String) -> Self {}
pub fn tags(&self) -> &Vec<String> {}

// generous input, stable output
pub fn new(name: impl Into<String>) -> Self {}
pub fn tags(&self) -> &[String] {}          // or `impl Iterator<Item = &str>`
```

Generic *inputs* cost the caller nothing and let you accept `&str`, `String`, `Cow`.
Generic/borrowed *outputs* that expose the inner container (`&Vec`, `&HashMap`) lock you
into that representation forever. Return a slice, an `impl Iterator`, or a newtype.

## `#[non_exhaustive]` on anything that might grow

```rust
#[non_exhaustive]
pub struct RequestOpts {
    pub timeout: Duration,
    pub retries: u32,
}

#[non_exhaustive]
pub enum Event { Connected, Disconnected }
```

Without it, adding a field or a variant is a major-version break: downstream `match Event`
without a wildcard, and struct literal construction, both stop compiling. With it, they're
minor-version additions. Cost: callers can't use struct-literal syntax, so pair it with a
constructor or `Default` + builder.

## Don't leak dependency types into your public API

```rust
// your crate's semver is now welded to url 2.x and http 0.2.x
pub fn fetch(&self, target: url::Url) -> Result<http::Response<Bytes>, Error> {}
```

If `url` releases 3.0, you can't upgrade without a breaking release of your own. Options:
re-export the type (`pub use url::Url;`) so at least the coupling is explicit and
versioned, wrap it in your own newtype, or accept `&str` and parse internally.

Exception: types from `std`, and from crates that are de-facto stable ABI for the ecosystem
(`serde`, `bytes`, `http` once 1.0) are acceptable to expose.

## Builder for optional or numerous arguments

```rust
// unreadable at the call site, every new option is a breaking change
pub fn connect(host: &str, port: u16, tls: bool, timeout: Duration,
               retries: u32, keepalive: Option<Duration>) -> Result<Conn, Error> {}

// each option is chainable, additive, self-documenting
Conn::builder("db.internal")
    .port(5432)
    .tls(Tls::Required)
    .timeout(Duration::from_secs(5))
    .connect()?;
```

Threshold: more than ~3 args, or any optional arg, or a `bool`/enum that reads as noise at
the call site.

## Return `Result` if it might ever fail

Adding a `Result` return later is a breaking change; every caller's `?` / `.unwrap()`
site changes. If a function does IO, parsing, or validation now, or plausibly will, return
`Result<T, Error>` from day one even if the current body can't fail.

Counter-case: a pure getter or arithmetic helper that will never fail should not return
`Result` "just in case" — it makes every call site noisier for nothing.

## Sealed trait when downstream must not implement it

If a public trait exists only for you to implement (a closed set of backends, a marker),
seal it so adding a method isn't a breaking change:

```rust
mod sealed { pub trait Sealed {} }

pub trait Backend: sealed::Sealed {
    fn query(&self, sql: &str) -> Result<Rows, Error>;
}

impl sealed::Sealed for Postgres {}
impl Backend for Postgres { /* ... */ }
```

Don't seal traits you *want* users to implement (that's the point of `trait`).

## `dyn` at the boundary is often kinder than generics

```rust
// every downstream type that stores a Store is now generic too; error messages
// mention closures; compile times and binary size grow
pub struct Client<S: Store> { store: S }

// one concrete type, trait objects at the edge
pub struct Client { store: Box<dyn Store + Send + Sync> }
```

Use generics when the hot path needs monomorphisation or the bound is `Fn`. Use `dyn` when
the type is stored in a struct users name, or the indirection cost is negligible.

## Feature flags are additive only

Enabling a feature may only *add* items. It must never remove an item, change a signature,
or alter behaviour. `cargo` unifies features across the dependency graph: if crate A
enables your `foo` feature, crate B gets it too, and B must still compile.

- No `#[cfg(not(feature = "x"))]` on a `pub` item that also exists under the feature with a
  different shape.
- A `default` feature set is part of your API; removing something from `default` is
  breaking even if the item still exists behind an explicit feature.

## MSRV is part of the contract

`rust-version` in `Cargo.toml` is a promise. Bumping it can break users on older
toolchains. Decide the policy (e.g. "latest stable minus 2") and treat a bump as at least
a minor release with a changelog note.

## Smaller ones

- Take `&self`, not `self`, unless the method consumes the value (builder `.build()`,
  typestate transitions). Don't require `&mut self` you don't use.
- Don't `impl Deref` for a non-smart-pointer to fake inheritance; it makes autocomplete
  and method resolution lie.
- `#[doc(hidden)]` on items you had to make `pub` for macros/internal use, so they're not
  part of the observed API (Hyrum's law still applies, but it signals intent).
- Implement the obvious std traits (`Debug` always; `Clone`, `PartialEq`, `Default`,
  `Hash` where they make sense) up front. Adding `Debug` later is not breaking; the lack
  of it is a constant papercut.

## Review checklist

- [ ] Inputs are `impl Into<_>` / `impl AsRef<_>` / borrowed; outputs don't expose `&Vec` / `&HashMap`
- [ ] `#[non_exhaustive]` on public structs and enums that may gain fields/variants
- [ ] No third-party non-stable types in `pub` signatures without a `pub use` re-export
- [ ] Functions with >3 args or optional args use a builder
- [ ] IO / parse / validate functions return `Result` even if the body can't fail yet
- [ ] Traits meant only for internal impl are sealed
- [ ] Struct-stored abstractions use `dyn` unless a perf bound needs generics
- [ ] No feature flag removes or reshapes a `pub` item; `default` set is intentional
- [ ] `rust-version` set, with a documented MSRV policy
- [ ] `Debug` on every public type; other std traits derived where sensible

## Boundaries

Public-surface review only. Does not check the implementation or run `cargo semver-checks`
(do run that too). Pairs with `rust-error-design` (the `Error` type is public API) and
`rust-idioms` (signature-level shape).

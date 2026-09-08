---
name: rust-idioms
description: >
  Review Rust for the idioms clippy doesn't flag: borrowed parameter types (`&str` over
  `&String`), newtypes for domain values, `Cow` for maybe-owned, enums over bool params,
  `From`/`TryFrom` over ad-hoc converters, `let else` and slice patterns, iterator chains
  over index loops, and `.clone()` used to dodge the borrow checker. Use when writing or
  reviewing function signatures, data modelling, conversions, or any "make this more
  idiomatic" request. Not a lint pass: these are shape choices clippy stays silent on.
---

# Rust Idioms

The mechanical stuff clippy catches. The idioms below it usually doesn't.

Each point: the non-idiomatic version, the idiomatic version, and when the "wrong" one is
actually right.

## Parameters: borrow the slice, not the container

```rust
// stiff: caller must have an owned String / Vec / PathBuf
fn greet(name: &String) {}
fn sum(xs: &Vec<i32>) {}
fn load(path: &PathBuf) {}

// idiomatic: accepts more, copies nothing
fn greet(name: &str) {}
fn sum(xs: &[i32]) {}
fn load(path: impl AsRef<Path>) {}
```

`&String` forces an allocation the caller might not have. `&str` also accepts `&String`
via deref, string literals, and substrings. Same for `&[T]` vs `&Vec<T>`.

Counter-case: if you need `Vec`-specific capacity/mutation, take `&mut Vec<T>`.

## Newtype domain values instead of primitive soup

```rust
// every id is a String; nothing stops you passing an email where a user id goes
fn transfer(from: String, to: String, memo: String) {}

// the compiler now rejects argument-order mistakes
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
struct UserId(u64);
#[derive(Debug, Clone, PartialEq, Eq)]
struct Memo(String);

fn transfer(from: UserId, to: UserId, memo: Memo) {}
```

Cheap wrapper, zero runtime cost, kills a class of bugs. Derive `Copy` only when the inner
type is `Copy` and small.

Counter-case: a short-lived internal helper with two `&str` args of obviously different
meaning doesn't need ceremony.

## `Cow<'_, str>` for "borrowed most of the time"

```rust
// always allocates, even when input needed no change
fn normalize(s: &str) -> String {
    s.trim().to_lowercase()
}

// allocates only on the path that actually modifies
fn normalize(s: &str) -> Cow<'_, str> {
    if s.chars().all(|c| c.is_ascii_lowercase()) && s.trim() == s {
        Cow::Borrowed(s)
    } else {
        Cow::Owned(s.trim().to_lowercase())
    }
}
```

Counter-case: if callers almost always own the result anyway, `Cow` is just friction.
Return `String`.

## Enum over `bool` parameter

```rust
// call site reads: write_file(path, data, true, false) — true what? false what?
fn write_file(path: &Path, data: &[u8], overwrite: bool, sync: bool) {}

// call site reads: write_file(path, data, Overwrite::Yes, Sync::No)
enum Overwrite { Yes, No }
enum Sync { Yes, No }
```

One `bool` is sometimes fine (`set_visible(true)`). Two adjacent `bool`s are always a
readability trap.

## `From` / `TryFrom`, not `to_x` / `parse_x` methods

```rust
// ad-hoc, not discoverable, doesn't compose with `?` or `.into()`
impl Config {
    fn from_json_str(s: &str) -> Result<Config, Error> {}
}

// standard, works with `.parse()`, `?`, generic bounds
impl FromStr for Config {
    type Err = Error;
    fn from_str(s: &str) -> Result<Self, Self::Err> {}
}
impl TryFrom<&RawConfig> for Config { /* ... */ }
```

Counter-case: a conversion that needs extra arguments (a base URL, an allocator) can't be
`From`. Keep it a named method.

## `let else` for the early-return pattern

```rust
// pyramid
let user = match repo.find(id) {
    Some(u) => u,
    None => return Err(Error::NotFound),
};

// flat
let Some(user) = repo.find(id) else {
    return Err(Error::NotFound);
};
```

## Slice patterns instead of index + length checks

```rust
// bounds-check soup
if args.len() >= 2 {
    let cmd = &args[0];
    let rest = &args[1..];
}

// destructure
let [cmd, rest @ ..] = args else {
    return Err(Error::Usage);
};
```

## Iterator chains over manual index loops, up to a point

```rust
// noisy, off-by-one surface
let mut out = Vec::new();
for i in 0..items.len() {
    if items[i].active {
        out.push(items[i].name.clone());
    }
}

// intent-revealing
let out: Vec<_> = items.iter()
    .filter(|it| it.active)
    .map(|it| it.name.clone())
    .collect();
```

Counter-case: an eight-adapter chain with three closures spanning a `flat_map` and a
`scan` is not more readable than a `for` loop. Break it, name the intermediate, or loop.

## `.collect::<Result<Vec<_>, _>>()` to short-circuit

```rust
// stops at the first Err, returns it
let parsed: Vec<i32> = lines.iter()
    .map(|l| l.parse::<i32>())
    .collect::<Result<_, _>>()?;
```

Same trick with `Option`. Also `partition` when you want both halves.

## `.clone()` to satisfy the borrow checker is a smell

A `.clone()` added because the compiler complained (not because you need a second owned
copy) usually means the code wants restructuring: narrow the borrow's scope, split the
struct, take the value out with `mem::take`, or reach for `Rc`/`Arc` *deliberately* with a
comment saying why.

Counter-case: cloning a small `Copy`-ish value (`String` of a few bytes, an enum) in
non-hot code to keep the code simple is fine. Don't contort a cold path to save an alloc.

## `ok_or_else` / `unwrap_or_else` when the fallback is non-trivial

```rust
// builds the error even on the happy path
opt.ok_or(Error::Missing { key: key.to_string() })?;

// lazy
opt.ok_or_else(|| Error::Missing { key: key.to_string() })?;
```

Eager `unwrap_or(Vec::new())` allocates every call; `unwrap_or_default()` or
`unwrap_or_else(Vec::new)` don't.

## Review checklist

- [ ] `&str` / `&[T]` / `impl AsRef<Path>` in params, not `&String` / `&Vec` / `&PathBuf`
- [ ] Domain ids and quantities are newtypes, not bare `String` / `u64`
- [ ] No adjacent `bool` params; use enums
- [ ] Conversions are `From` / `TryFrom` / `FromStr` where they fit
- [ ] `let else` / slice patterns instead of match-to-return pyramids and index checks
- [ ] Iterator chains for transforms, but none longer than they are readable
- [ ] Fallible mapping uses `collect::<Result<_, _>>()`, not a manual loop with `break`
- [ ] No `.clone()` that exists only to silence the borrow checker
- [ ] `*_or_else` / `*_or_default` where the eager form allocates or errors needlessly

## Boundaries

Shape review only. Run `cargo clippy` separately for the mechanical lints. Pairs with
`rust-api-design` (public surface) and `rust-error-design` (the failure half).

---
name: rust-async
description: >
  Review async Rust for cancel safety, `spawn` discipline, and shared-state choice. Covers
  state held across `.await` under `select!`, locks across await points, `Arc<Mutex>` vs
  actor/channel, detached tasks that swallow panics, CPU-bound work blocking the runtime,
  sequential `.await` loops that should be concurrent, and cleanup that can't run in `drop`.
  Use when writing or reviewing `async fn`, `tokio::spawn`, `select!`, `Stream` consumers,
  or anything touching a Tokio runtime. Not a lint pass: reasons about what happens when a
  future is dropped mid-poll.
---

# Rust Async

The compiler proves your async code is memory-safe. It does not prove it is *correct* when
a future gets cancelled halfway through. Most async bugs live there.

## Cancel safety: what survives a dropped future

A future can be dropped at any `.await` point without resuming. `tokio::select!` does
exactly this to every branch that loses. Anything the future had half-done is now
permanently half-done.

```rust
// NOT cancel-safe: if this select! branch loses, `buf` holds a partial line
// that is silently lost, and the socket is left mid-message
select! {
    _ = read_line(&mut socket, &mut buf) => { /* ... */ }
    _ = shutdown.recv() => { /* buf discarded */ }
}
```

Rules:

- In a `select!` loop, only await operations documented as cancel-safe
  (`tokio::sync::mpsc::Receiver::recv`, `Notify::notified`, a `&mut JoinHandle`), or ones
  where a dropped partial result is genuinely fine.
- If you must run a non-cancel-safe operation alongside a cancel signal, `spawn` it and
  `select!` on its `JoinHandle`, so the operation runs to completion on its own task.
- State that must stay consistent lives in a value that is *fully updated or not at all*
  between await points, never spread across two `.await`s.

## Don't hold a lock across `.await`

```rust
// std Mutex across await: clippy flags this (await_holding_lock), and it can deadlock
let mut g = state.lock().unwrap();
g.count += fetch_delta().await;   // guard held across IO

// even tokio::Mutex across await is usually wrong: every task now serialises
// through this lock for the whole duration of the IO
```

Fix, in order of preference:

1. Do the `.await` first, then take the lock only to write the result.
2. If the critical section is non-trivial or contended, don't share the state with a lock
   at all: give it to one task and talk to it over a channel (actor pattern).
3. `tokio::sync::Mutex` only when the lock genuinely must be held across an await *and*
   contention is low (e.g. a connection handshake).

## `Arc<Mutex<T>>` vs actor / channel

`Arc<Mutex<T>>` is fine for: a counter, a config snapshot swapped wholesale, a small map
read often and written rarely.

Prefer an owning task + `mpsc` when: the critical section does real work, invariants span
multiple fields, or you're tempted to hold the lock across `.await`.

```rust
// actor: state is never shared, so no lock, no cross-await hazard
enum Cmd { Add(i64, oneshot::Sender<i64>), Snapshot(oneshot::Sender<State>) }

fn spawn_counter() -> mpsc::Sender<Cmd> {
    let (tx, mut rx) = mpsc::channel(32);
    tokio::spawn(async move {
        let mut state = State::default();
        while let Some(cmd) = rx.recv().await {
            match cmd {
                Cmd::Add(n, reply) => { state.count += n; let _ = reply.send(state.count); }
                Cmd::Snapshot(reply) => { let _ = reply.send(state.clone()); }
            }
        }
    });
    tx
}
```

## A spawned task you don't hold is fire-and-forget

`tokio::spawn` returns a `JoinHandle`. Drop it and: the task keeps running detached, its
panic is swallowed (only visible if you `.await` the handle), and shutdown can't wait for
it.

```rust
// panic here vanishes; graceful shutdown can't drain this
tokio::spawn(async move { process(job).await });

// keep handles, or use a JoinSet / TaskTracker
let mut set = JoinSet::new();
set.spawn(async move { process(job).await });
while let Some(res) = set.join_next().await {
    if let Err(e) = res { tracing::error!("worker failed: {e}"); }
}
```

## CPU-bound work blocks the whole runtime

An `async fn` that spends 50ms hashing or parsing holds its worker thread the entire time.
On a multi-threaded runtime that's one fewer worker; on `current_thread` everything stalls.

```rust
// blocks the executor
let hash = argon2_hash(&password);

// hand it to the blocking pool
let hash = tokio::task::spawn_blocking(move || argon2_hash(&password)).await?;
```

Same for `std::fs`, `std::net`, big `serde_json` on megabytes, `rayon` joins.

## Sequential `.await` in a loop that should be concurrent

```rust
// N round-trips, one after another
let mut out = Vec::new();
for id in ids {
    out.push(fetch(id).await?);
}

// bounded concurrency
use futures::stream::{self, StreamExt, TryStreamExt};
let out: Vec<_> = stream::iter(ids)
    .map(|id| fetch(id))
    .buffer_unordered(16)
    .try_collect()
    .await?;
```

Keep the concurrency bounded (`buffer_unordered(n)`, `JoinSet` with a cap). Unbounded
`join_all` over a user-controlled list is a resource-exhaustion bug.

## Cleanup that must run cannot live in `Drop`

`Drop` is synchronous. An async "close the session / flush the buffer / send goodbye" in a
`Drop` impl can only block or spawn-and-hope.

Use an explicit `async fn close(self)`, or a `CancellationToken` the task checks so it can
run its own async cleanup before exiting. Don't rely on drop order for correctness.

## Smaller ones

- `async fn` that never `.await`s should not be `async`. It just defers the work to poll
  time and infects callers.
- `select!` in a loop: hold futures in `let mut fut = ...;` outside the loop or pin them,
  or they restart every iteration.
- Always give network `.await`s a `tokio::time::timeout` and handle the elapsed branch.
- `Stream` consumption needs `StreamExt` in scope and often `tokio::pin!`.

## Review checklist

- [ ] Every operation awaited inside `select!` is cancel-safe, or a dropped partial is fine
- [ ] No lock guard (`std` or `tokio`) held across an `.await`
- [ ] Shared mutable state with a non-trivial critical section is an actor, not `Arc<Mutex>`
- [ ] Spawned tasks are tracked (`JoinSet` / `JoinHandle` / `TaskTracker`), panics surfaced
- [ ] CPU-bound and blocking-IO work goes through `spawn_blocking` / `rayon`
- [ ] Concurrent fan-out is bounded (`buffer_unordered(n)`), never unbounded on user input
- [ ] Async cleanup is an explicit call or token-driven, not in `Drop`
- [ ] Network awaits have a timeout
- [ ] No `async fn` that never awaits

## Boundaries

Concurrency-correctness review only. Does not run the code or a race detector. Pairs with
`rust-error-design` (cancellation and timeout as error variants).

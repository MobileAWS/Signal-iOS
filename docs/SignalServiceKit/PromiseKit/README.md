# Signal iOS — PromiseKit Subsystem

This documentation set reconstructs Signal iOS's in-house **PromiseKit** module
from the first-party source in `SignalServiceKit/PromiseKit/` (17 files). It is a
small, self-contained Promise/Future async library that long predates (and
interoperates with) Swift Concurrency. A `Promise<Value>` represents an async
computation that will either **succeed with a `Value`** or **fail with an
`Error`**; a `Guarantee<Value>` is the same idea for computations that **cannot
fail**. The library drives most of the app's older callback-based async flows and
provides explicit bridges (`wrapAsync` / `awaitable`) to and from `async`/`await`.

> **Confidence labels.** Each claim is tagged with a confidence level:
> - **[High]** — directly read from source; behavior is explicit in code.
> - **[Medium]** — inferred from code with reasonable certainty, but depends on
>   collaborators defined outside this directory (notably `UnfairLock`,
>   `AtomicValue`, `CancellableContinuation`, `owsFail*`, and the
>   `DispatchQueueIsCurrentQueue` / `_CurrentStackUsage` runtime helpers).
> - **[Low]** — inferred from naming/comments; not fully verified in-tree.
>
> Where the code does not reveal *why* something is done, this is stated as
> "intent undetermined — no evidence in source".
>
> All file+line citations refer to the state of the tree at the time of writing.
> Line numbers are approximate anchors; use the cited symbol name to locate code
> if lines have drifted.

---

## The central concept: a `Future` holds the result; everything else is a view over it

The real state lives in **`Future<Value>`**: a lock-guarded box that holds an
optional `Result<Value, Error>` and a list of observers. Resolving/rejecting the
future seals the result exactly once and flushes all pending observers; observing
after sealing fires immediately. **[High]** —
`Future.observe` / `Future.sealResult`
(`SignalServiceKit/PromiseKit/Future.swift:28-60`).

Everything a caller touches — `Promise`, `Guarantee`, chaining operators, the
combinators — is a thin layer that **creates a pending future, wires an observer
onto an upstream, and resolves/rejects the new future**. `Promise` and
`Guarantee` are reference types (`final class`) that each wrap one `Future`;
`Thenable` is the read-side protocol they both conform to, and `Catchable` adds
the error-handling operators. **[High]** —
`Promise` (`Promise.swift:14-16`), `Guarantee` (`Guarantee.swift:8-10`),
`Thenable` (`Thenable.swift:8-13`), `Catchable` (`Catchable.swift:8`).

```mermaid
graph TD
    Future["Future&lt;Value&gt;<br/>UnfairLock + result + observers<br/>Future.swift"]
    Thenable["protocol Thenable<br/>result, observe(on:block:)<br/>+ map/then/done/asVoid"]
    Catchable["protocol Catchable: Thenable<br/>+ catch/recover/ensure/cauterize"]
    Promise["final class Promise&lt;Value&gt;<br/>: Thenable, Catchable<br/>can fail"]
    Guarantee["final class Guarantee&lt;Value&gt;<br/>: Thenable<br/>cannot fail (owsFail on error)"]

    Promise -->|wraps one| Future
    Guarantee -->|wraps one| Future
    Promise -.conforms.-> Thenable
    Promise -.conforms.-> Catchable
    Guarantee -.conforms.-> Thenable
    Catchable -.extends.-> Thenable
    Future -->|resolve(on:with:)| Thenable
```

---

## Document map

This module is small enough that all behavior is documented in this single file.
The table maps each concern to its primary source.

| Area | What it covers | Primary source |
| --- | --- | --- |
| Core state | The lock-guarded result box, observer flushing, seal-once semantics | `Future.swift` |
| Read protocol | `Thenable` requirements + `map`/`then`/`done`/`asVoid`/`value` | `Thenable.swift` |
| Error protocol | `Catchable` + `catch`/`recover`/`ensure`/`cauterize` | `Catchable.swift` |
| Fallible type | `Promise<Value>`, `wait()`, async bridges, `pending()` | `Promise.swift` |
| Infallible type | `Guarantee<Value>`, `GuaranteeFuture`, infallible `then`/`map`/`done` | `Guarantee.swift` |
| Scheduling | `Scheduler` protocol, `DispatchQueue` conformance, `SyncScheduler` | `Scheduler.swift`, `DispatchQueue+Promise.swift`, `SyncScheduler.swift` |
| Entry point | `firstly` overloads | `firstly.swift` |
| Combinators | `race`, `when(fulfilled:)`/`when(resolved:)`, `after`, `timeout`/`nilTimeout` | `Thenable+Race.swift`, `Guarantee+Race.swift`, `Thenable+When.swift`, `Thenable+After.swift`, `Thenable+Timeout.swift`, `Guarantee+Timeout.swift` |
| Integration | One-shot notification observation | `NotificationCenter+Promise.swift` |
| Tests | Chaining, errors, recovery, when, timeout, deep chains, async bridge | `PromiseTests.swift` |

---

## Core types and their relationships

| Type | Kind | Can fail? | Conforms to | Role |
| --- | --- | --- | --- | --- |
| `Future<Value>` | `final class` | yes (`reject`) | — | The underlying result box; written via `resolve`/`reject`, read via `observe` |
| `Thenable` | `protocol: AnyObject` | — | — | Read side: `result`, `observe(on:block:)`, plus `map`/`then`/`done` in an extension |
| `Catchable` | `protocol: Thenable` | — | `Thenable` | Error side: `catch`/`recover`/`ensure`/`cauterize` |
| `Promise<Value>` | `final class` | yes | `Thenable`, `Catchable` | Public fallible async value |
| `Guarantee<Value>` | `final class` | **no** | `Thenable` | Public infallible async value; a failure is a programmer error |

**`Promise` is both `Thenable` and `Catchable`; `Guarantee` is only `Thenable`.**
This is the type-level encoding of "guarantees can't fail": there is no `catch`
on a `Guarantee` because it has no error channel to catch. **[High]** —
`Promise.swift:14`, `Guarantee.swift:8`, `Catchable.swift:8`.

### What "infallible" means mechanically

A `Guarantee` still wraps a `Future`, which *can* be rejected — but every
`Guarantee` code path that encounters a `.failure` calls `owsFail(...)` (a fatal
assertion) rather than propagating it. So reaching an error inside a `Guarantee`
is treated as an unrecoverable programmer error, not a runtime condition.
**[High]** — e.g. `Guarantee.wait` (`Guarantee.swift:60`), `Guarantee.awaitable`
(`Guarantee.swift:90`), `Guarantee.asPromise` (`Promise.swift:150`).

### `Future` sealing and observation

| Operation | Behavior | Source |
| --- | --- | --- |
| `resolve(_:)` | Seals `.success(value)` | `Future.swift:62-64` |
| `reject(_:)` | Seals `.failure(error)` | `Future.swift:74-76` |
| `resolve(on:with:)` | Seals from another `Thenable` by chaining its `done`/`catch` | `Future.swift:66-72` |
| `observe(on:block:)` | If already sealed, dispatch block now; else append to observers | `Future.swift:28-45` |
| `sealResult` | Sets result **only if currently `nil`** (seal-once), snapshots+clears observers under lock, then fires them outside the lock | `Future.swift:47-60` |

All mutation is serialized through an `UnfairLock`; observers are invoked
**outside** the lock to avoid re-entrancy/deadlock when a callback schedules more
work (`Future.swift:47-60`). **[High]** The seal-once guard means a second
`resolve`/`reject` is silently ignored — the basis for the `race` and `timeout`
combinators below. **[High]**

---

## The default scheduler is the main queue (a deliberate PromiseKit-compat choice)

When `observe` is called with **no** scheduler, work is dispatched on
`DispatchQueue.main`. The source comment states this matches "the behavior we
expect from PromiseKit" and that "eventually we'll want to switch this default".
**[High]** — `Future.observe` inner `execute` (`Future.swift:30-38`).

This is visible in the tests: a `.done` with no explicit queue runs on main even
when the upstream `.map` ran on `.global()`
(`PromiseTests.swift:16-30`, `test_simpleQueueChaining`). **[High]**

---

## Scheduling model

`Scheduler` is a protocol-ization of `DispatchQueue` so tests can substitute
scheduling behavior. The source comment is explicit: "In production code usage,
DispatchQueues _are_ schedulers and should be used directly as such." **[High]** —
`Scheduler.swift:8-30`.

| Member | Purpose | Source |
| --- | --- | --- |
| `async(_:)` | Fire-and-forget | `Scheduler.swift:12` |
| `sync(_:)` (3 overloads) | Run synchronously (void, returning, throwing) | `Scheduler.swift:14-18` |
| `asyncAfter(deadline:_:)` | Schedule after a `DispatchTime` (mach clock, pauses while suspended) | `Scheduler.swift:20` |
| `asyncAfter(wallDeadline:_:)` | Schedule after a `DispatchWallTime` (wall clock, ticks while suspended) | `Scheduler.swift:22` |
| `asyncIfNecessary(execute:)` | Run inline if already safe, else `async` | `Scheduler.swift:24` |

The extension adds `async(_ namespace: PromiseNamespace, execute:)` overloads
that wrap a block's result into a `Guarantee<T>` (non-throwing) or `Promise<T>`
(throwing). **[High]** — `Scheduler.swift:32-50`.

### `DispatchQueue` is a `Scheduler`

`DispatchQueue` conforms by forwarding to its native `execute:` APIs. The one
piece of real logic is `asyncIfNecessary`: it runs the work **inline** only when
the current queue *is* this queue **and** the current stack usage is below 80%;
otherwise it defers via `async`. **[High]** —
`DispatchQueue+Promise.swift:32-38`. The stack-depth guard
(`_CurrentStackUsage() < 0.8`) prevents inline execution from overflowing the
stack during very deep synchronous chains — exercised by `test_deepPromiseChain`
(1000-deep chain) in `PromiseTests.swift:224-240`. **[Medium]** — relies on the
externally-defined `_CurrentStackUsage` / `DispatchQueueIsCurrentQueue` helpers.

### `SyncScheduler`

Runs all work **synchronously on the current thread** with the least overhead.
`async`, `sync`, and `asyncIfNecessary` all just invoke the block directly.
**[High]** — `SyncScheduler.swift:17-45`.

The *scheduled* methods are intentionally hostile: `asyncAfter(deadline:)` and
`asyncAfter(wallDeadline:)` call `owsFailDebug` and fall back to
`DispatchQueue.main`, because a sync scheduler has no notion of "later". The
header comment warns against using it for scheduled methods such as `after`.
**[High]** — `SyncScheduler.swift:9-13`, `:33-42`. It is the scheduler used by the
async bridges (`awaitable`) so the continuation resumes without an extra hop.
**[High]** — `Promise.swift:124`, `Guarantee.swift:88`.

| Scheduler | `async`/`asyncIfNecessary` | `asyncAfter` | Typical use |
| --- | --- | --- | --- |
| `DispatchQueue` | real GCD dispatch (inline when safe) | real GCD timer | production |
| `SyncScheduler` | inline on current thread | `owsFailDebug` → main queue | tight-loop/no-hop work, `awaitable` bridges |

---

## Chaining operators

### On `Thenable` (available to both `Promise` and `Guarantee`)

| Operator | Signature shape | Returns | Semantics | Source |
| --- | --- | --- | --- | --- |
| `map` | `(Value) throws -> T` | `Promise<T>` | Transform the value; a throw becomes a rejection | `Thenable.swift:15-21` |
| `then` | `(Value) throws -> T: Thenable` | `Promise<T.Value>` | Flat-map into another async value | `Thenable.swift:28-47` |
| `done` | `(Value) throws -> Void` | `Promise<Void>` | Side-effect on success | `Thenable.swift:23-27` |
| `asVoid` | — | `Promise<Void>` | Discard the value | `Thenable.swift:55-59` |
| `value` | — | `Value?` | Synchronous peek: the success value if already sealed, else `nil` | `Thenable.swift:50-53` |

The `Thenable` `map`/`done` funnel through a private `observe<T>` helper that
builds a pending `Promise<T>`, resolves it from the block on success, and rejects
it on upstream failure or a thrown error. `then` is similar but resolves the new
future *from the returned thenable* via `future.resolve(on:with:)`. **[High]** —
`Thenable.swift:62-80`, `:28-47`.

### On `Guarantee` (non-throwing variants)

`Guarantee` provides its **own** `map`/`then`/`done` whose blocks **cannot throw**
and that return `Guarantee`s, not `Promise`s. `then` takes `(Value) ->
Guarantee<T>`. These shadow the `Thenable` defaults for the infallible case.
`done` and `then` are `@discardableResult`. **[High]** — `Guarantee.swift:98-125`.

### On `Catchable` (error handling — `Promise` only)

| Operator | Block | Returns | Semantics | Source |
| --- | --- | --- | --- | --- |
| `catch` | `(Error) -> Void` | `Promise<Void>` (`@discardableResult`) | Terminal error handler | `Catchable.swift:11-17` |
| `recover` | `(Error) -> Guarantee<Value>` | `Guarantee<Value>` | Replace error with a value → result cannot fail | `Catchable.swift:19-24` |
| `recover` | `(Error) throws -> T: Thenable` | `Promise<Value>` | Replace error with another fallible async value | `Catchable.swift:26-31` |
| `ensure` | `() -> Void` | `Promise<Value>` | Run on **both** success and failure (finally) | `Catchable.swift:33-42` |
| `cauterize` | — | `Self` | No-op marker that an error is intentionally ignored | `Catchable.swift:44-45` |
| `asVoid` | — | `Promise<Void>` | Discard value | `Catchable.swift:47` |

`recover` has `Value == Void` specializations taking `(Error) -> Void` /
`(Error) throws -> Void` (`Catchable.swift:159-175`). All operators route through
private `observe(...)` helpers that construct the appropriate downstream
(`Promise` when a throw/error is still possible, `Guarantee` when the error has
been fully absorbed). **[High]** — `Catchable.swift:50-157`.

Behavioral evidence from tests:
- A thrown error inside `map` skips `done` and lands in `catch`
  (`PromiseTests.swift:54-80`, `test_queueChainingWithErrors`). **[High]**
- `recover` returning `.value("xyz")` turns a failed chain back into success so
  `catch` never fires (`PromiseTests.swift:82-100`, `test_recovery`). **[High]**
- `ensure` runs on both the failure and success chains
  (`PromiseTests.swift:102-131`, `test_ensure`). **[High]**

---

## `firstly` — the chain entry point

`firstly` runs a block and lifts its result into a `Promise` (or `Guarantee`),
giving a chain a clean starting point. There are four overloads: **[High]** —
`firstly.swift:8-57`.

| Overload | Scheduler | Block | Returns | Source |
| --- | --- | --- | --- | --- |
| `firstly(on:_:)` | optional, default `nil` | `() throws -> T: Thenable` (non-escaping) | `Promise<T.Value>` | `firstly.swift:8-20` |
| `firstly(on:_:)` | required | `@escaping () throws -> T: Thenable` | `Promise<T.Value>` | `firstly.swift:22-35` |
| `firstly(on:_:)` | required | `@escaping () throws -> T` (plain value) | `Promise<T>` | `firstly.swift:37-50` |
| `firstly(on:_:)` | required | `@escaping () -> T` (non-throwing) | `Guarantee<T>` | `firstly.swift:52-57` |

When a scheduler is supplied, the block runs via `asyncIfNecessary`; the
`Thenable`-returning overloads seal the new future with `future.resolve(on:with:)`
so the chain adopts the returned async value. **[High]** — `firstly.swift:26-33`.

---

## Combinators

### `race` — first to settle wins

| Variant | On | Behavior | Source |
| --- | --- | --- | --- |
| `Thenable.race` | `Promise` | First thenable to settle (success **or** failure) resolves/rejects the result; later settlements are dropped via `future.isSealed` guard | `Thenable+Race.swift:8-37` |
| `Guarantee.race` | `Guarantee` | First to produce a value wins; requires a non-optional `Scheduler` | `Guarantee+Race.swift:8-30` |

Both rely on `Future`'s seal-once guard plus an explicit `!future.isSealed` check
to ignore all but the first settlement. **[High]** — `Thenable+Race.swift:24-32`,
`Guarantee+Race.swift:20-27`.

### `when` — aggregate multiple async values

| Variant | Returns | Semantics | Source |
| --- | --- | --- | --- |
| `when(fulfilled: [T])` → values | `Promise<[Value]>` | Resolves when **all** succeed; **rejects on the first failure**; empty input resolves immediately | `Thenable+When.swift:9-15`, `:74-98` |
| `when(fulfilled:_:…)` 2–4 tuple arities | `Promise<(…)>` | Same, but preserves heterogeneous element types via `asVoid` + `value!` | `Thenable+When.swift:17-57` |
| `when(fulfilled:)` on `Void` | `Promise<Void>` | Variadic/array convenience | `Thenable+When.swift:60-71` |
| `when(resolved: [T])` | `Guarantee<[Result<Value, Error>]>` | Waits for **all** to settle regardless of success/failure; never rejects | `Thenable+When.swift:103-135` |

The private `_when(fulfilled:)` tracks a remaining count in an
`AtomicValue<Int>`, decrementing on each success and resolving at zero; any
failure rejects the aggregate immediately. `_when(resolved:)` only counts
settlements and never rejects. **[High]** — `Thenable+When.swift:74-98`,
`:120-135`. Tests confirm `fulfilled` short-circuits on first error while
`resolved` waits for every chain (`PromiseTests.swift:133-213`). **[High]**

### `after` — a timer promise

`Guarantee<Void>.after` resolves after a delay. Two clock variants: **[High]** —
`Thenable+After.swift:8-50`.

| Variant | Clock | Suspended behavior | Source |
| --- | --- | --- | --- |
| `after(on:seconds:)` | `mach_absolute_time` via `asyncAfter(deadline:)` | **pauses** while the process is suspended | `Thenable+After.swift:10-39` |
| `after(on:wallInterval:)` | `gettimeofday` via `asyncAfter(wallDeadline:)` | **ticks** while suspended | `Thenable+After.swift:42-49` |

The `seconds` variant logs a warning when running in an **app extension**
(detected by a `.appex` bundle-path suffix) with a delay over 2 seconds, because
an extension may be killed before the future resolves, leaking captured objects;
the comment suggests the walltime variant in that case. Default scheduler is
`DispatchQueue.global()`. **[High]** — `Thenable+After.swift:17-37`.

### `timeout` / `nilTimeout`

Timeouts are built **on top of** `after` + `race`: the real work races against a
timer. **[High]**

| Operator | On | On timeout | Source |
| --- | --- | --- | --- |
| `Thenable.timeout(seconds:substituteValue:)` | `Promise` | Resolve with `substituteValue` | `Thenable+Timeout.swift:28-49` |
| `Thenable.nilTimeout(seconds:)` | `Promise` | Resolve with `nil` (logs "Timed out, returning nil value.") | `Thenable+Timeout.swift:8-26` |
| `Thenable.timeout(seconds:)` on `Void` | `Promise<Void>` | Substitute `()` | `Thenable+Timeout.swift:118-124` |
| `Promise.timeout(…timeoutErrorBlock:)` | `Promise` | **Reject** with the error from `timeoutErrorBlock()`; `ticksWhileSuspended` selects wall vs. mach clock | `Thenable+Timeout.swift:52-104` |
| `Guarantee.timeout(seconds:substituteValue:)` | `Guarantee` | Substitute value via `race` | `Guarantee+Timeout.swift:11-24` |
| `Guarantee.nilTimeout(seconds:)` | `Guarantee` | Substitute `nil` | `Guarantee+Timeout.swift:7-10` |

The substitute-value timeouts race `self` tagged `false` against a timer tagged
`true`, then branch on the `didTimeout` flag and log on timeout
(`Thenable+Timeout.swift:36-48`). The erroring `Promise.timeout` races against a
timer that throws an internal `TimeoutError` (`.wallTimeout` / `.relativeTimeout`,
`:106-110`), then `recover`s: a `TimeoutError` is converted to the caller's
`timeoutErrorBlock()` error, any other error is re-propagated unchanged.
**[High]** — `Thenable+Timeout.swift:83-103`.

Test evidence: a 15 s task with a 1 s timeout yields `"substitute"`; a 1 s task
with a 3 s timeout yields the real `"default"`
(`PromiseTests.swift:242-271`, `test_timeout`). **[High]**

---

## Integrations and bridges

### `NotificationCenter`

`NotificationCenter.observe(once:object:)` returns a `Guarantee<Notification>`
that resolves on the **first** matching notification and then removes its own
observer (the cleanup is chained via `.done`). **[High]** —
`NotificationCenter+Promise.swift:8-16`.

### Swift Concurrency bridges

| Bridge | Direction | Notes | Source |
| --- | --- | --- | --- |
| `Promise.wrapAsync` | `async throws` → `Promise` | Spawns a `Task` with default args; resolve/reject from result | `Promise.swift:88-99` |
| `Guarantee.wrapAsync` | `async` → `Guarantee` | Spawns a `Task`; resolves (cannot fail) | `Guarantee.swift:68-78` |
| `Promise.awaitable()` | `Promise` → `async throws` | `withCheckedThrowingContinuation`, observed on `SyncScheduler()` | `Promise.swift:101-108` |
| `Promise.awaitableWithUncooperativeCancellationHandling()` | `Promise` → `async throws` | Uses `CancellableContinuation`; throws `CancellationError` on Task cancellation and stops waiting | `Promise.swift:110-128` |
| `Guarantee.awaitable()` | `Guarantee` → `async` | `withCheckedContinuation`; `owsFail` on an unexpected error | `Guarantee.swift:80-99` |

Both `wrapAsync` doc comments note the `Task` is created with **default**
priority; a caller who needs a specific priority must build its own instance.
**[Medium]** — intent for the default-priority choice is stated only as a comment;
`Promise.swift:85-88`, `Guarantee.swift:66-68`.

### Blocking waits

| Method | On | Blocking strategy | On failure | Source |
| --- | --- | --- | --- | --- |
| `Promise.wait()` | `Promise` | If unsealed, observe on `.global()` + `DispatchGroup.wait()`; returns value or **throws** | throws the error | `Promise.swift:63-84` |
| `Guarantee.wait()` | `Guarantee` | Same blocking strategy | `owsFail` (cannot fail) | `Guarantee.swift:48-66` |

`test_wait` confirms the success path returns and the throwing path propagates
(`PromiseTests.swift:215-222`). **[High]**

---

## Constructing promises/guarantees

| Constructor | Produces | Source |
| --- | --- | --- |
| `Promise.value(_:)` / `Promise(error:)` | Pre-sealed success/failure | `Promise.swift:25-35` |
| `Promise { future in … }` | Imperative sealing; a throw rejects | `Promise.swift:37-45` |
| `Promise(on:_:)` | Same, but body runs on a scheduler via `asyncIfNecessary` | `Promise.swift:47-57` |
| `Promise.pending()` | `(Promise, Future)` pair for manual resolution | `Promise.swift:131-134` |
| `Guarantee.value(_:)` | Pre-sealed success | `Guarantee.swift:22-26` |
| `Guarantee { resolve in … }` / `Guarantee(on:_:)` | Imperative resolution (no error channel) | `Guarantee.swift:28-39` |
| `Guarantee.pending()` | `(Guarantee, GuaranteeFuture)` pair | `Guarantee.swift:12-15` |
| `Guarantee.asPromise()` | Lift an infallible value into the fallible world (an error would `owsFail`) | `Promise.swift:149-161` |

`GuaranteeFuture` is a `struct` wrapper that exposes only `resolve` (no
`reject`), enforcing infallibility at the write side; a `Void` extension adds a
no-arg `resolve()`. **[High]** — `Guarantee.swift:163-176`.

---

## Cross-cutting conventions observed in the code

- **Seal-once, observe-outside-lock.** `Future` can only be sealed once; the
  guard in `sealResult` is what makes `race`/`timeout`/`observe(once:)` safe to
  wire multiple resolvers (`Future.swift:47-60`). **[High]**
- **No-scheduler defaults to the main queue** for `Future.observe`, matching
  legacy PromiseKit behavior (`Future.swift:30-38`). **[High]**
- **`Guarantee` never surfaces errors** — every failure path is an `owsFail*`
  programmer-error trap, not a recoverable condition
  (`Guarantee.swift:60`, `:90`; `Promise.swift:155`). **[High]**
- **The type system encodes fallibility**: only `Promise` is `Catchable`, so
  `catch`/`recover`/`ensure` are unavailable on a `Guarantee`
  (`Catchable.swift:8`, `Promise.swift:14`, `Guarantee.swift:8`). **[High]**
- **Timeouts and `when` are composed, not primitive** — `timeout` = `after` +
  `race`; `when` = atomic counter over `observe`
  (`Thenable+Timeout.swift:28-49`, `Thenable+When.swift:74-98`). **[High]**
- **Deep chains are supported** — the `asyncIfNecessary` stack-usage guard keeps
  a 1000-deep synchronous chain from overflowing
  (`DispatchQueue+Promise.swift:32-38`, `PromiseTests.swift:224-240`). **[Medium]**
- **`cauterize()` is documentation, not behavior** — it returns `self` unchanged
  to mark an intentionally-unhandled error (`Catchable.swift:44-45`). **[High]**
- **Intent for the "eventually switch the default scheduler" comment is
  undetermined** — the code states the intent to change but no evidence of the
  target or timeline exists in source (`Future.swift:31-34`).
  Intent undetermined — no evidence in source.

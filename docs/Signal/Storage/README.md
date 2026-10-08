# Signal iOS — `Signal/Storage`

This document reconstructs the **`Signal/Storage`** module from its first-party
source (`Signal/Storage/`, 4 Swift files). The module is small and focused: it
contains the app-target machinery for running **deferred, long-running database
work** as iOS **background processing tasks** (`BGProcessingTask`) — chiefly
lazy database migrations and full-text-search index maintenance that are too
expensive to run inline during launch.

> **Confidence labels.** Each claim is tagged with a confidence level:
> - **[High]** — directly read from source; behavior is explicit in code.
> - **[Medium]** — inferred from code with reasonable certainty, but depends on
>   collaborators defined outside this directory (notably `SignalServiceKit`'s
>   `BGProcessingTaskRunner` protocol, `SDSDatabaseStorage`, and the migrator
>   types).
> - **[Low]** — inferred from naming/comments; not fully verified in-tree.
>
> Where the code does not reveal *why* something is done, this is stated as
> "intent undetermined — no evidence in source".
>
> All file+line citations refer to the state of the tree at the time of writing.
> Line numbers are approximate anchors; use the cited symbol name to locate code
> if lines have drifted.

---

## What this module is

`Signal/Storage` is where the **application target** registers and schedules
background database maintenance that should not block app launch or the main
run loop. It is a thin layer: the generic background-task plumbing lives in
`SignalServiceKit` (the `BGProcessingTaskRunner` protocol), and this module
provides the concrete *runners* plus the migration logic they drive.

The four files split into two concerns:

| File | Role |
| --- | --- |
| `BGProcessingTaskRunner.swift` | The generic `BGProcessingTaskRunner` protocol + its default registration/scheduling/execution extension, and the `BGProcessingTaskStartCondition` enum. **[High]** |
| `LazyDatabaseMigratorRunner.swift` | A concrete runner that drives deferred (“lazy”) database migrations as a background task. **[High]** |
| `InfoMessageGroupUpdateMigrator.swift` | The actual migration logic the lazy runner executes (migrating group-update info messages). **[Medium]** — name + usage. |
| `FullTextSearchOptimizer.swift` | Background optimization/maintenance of the full-text-search index. **[Medium]** — name + usage. |

---

## The core abstraction: `BGProcessingTaskRunner`

`BGProcessingTaskRunner` (`Signal/Storage/BGProcessingTaskRunner.swift:21`) is a
protocol that standardizes how a long-running task integrates with iOS's
`BGTaskScheduler`. A conformer supplies four static descriptors and two
behavioral methods; a protocol extension supplies all the registration,
scheduling, and lifecycle handling. **[High]**

Required static descriptors (`:23-40`): **[High]**

| Member | Meaning |
| --- | --- |
| `taskIdentifier: String` | The `BGTaskScheduler` identifier. **MUST** also be declared in `Info.plist` under *Permitted background task scheduler identifiers*. |
| `logPrefix: String?` | Prefix for task-related logs. |
| `requiresNetworkConnectivity: Bool` | Tells iOS the task needs a network connection. |
| `requiresExternalPower: Bool` | Tells iOS the task needs external power — recommended when CPU use is high, since iOS terminates high-CPU work aggressively on battery. |

Required behavior (`:42-55`): **[High]**

- `startCondition() -> BGProcessingTaskStartCondition` — whether/when to run.
- `run() async throws` — the actual work. Conformers are expected to detect
  `Task` cancellation and still make **incremental progress** when the OS
  terminates the task.

### `BGProcessingTaskStartCondition`

An enum (`:9-16`) with three cases that map directly onto `BGProcessingTaskRequest`
scheduling: **[High]**

- `.never` — do not schedule the task at all.
- `.asSoonAsPossible` — ask the OS to run it as soon as it can.
- `.after(Date)` — pass the date as `BGProcessingTaskRequest.earliestBeginDate`.

### What the extension does for you

The `where Self: Sendable` extension (`:58`) provides the standardized
machinery so each runner only writes its own logic: **[High]**

- **`registerBGProcessingTask(appReadiness:)`** (`:66`) — registers the launch
  handler with `BGTaskScheduler.shared.register`. The code notes this must be
  called **unconditionally** for every task identifier in `Info.plist`
  (scheduling, not registration, is what makes a task actually run), citing
  Apple's guidance and WWDC sample. **[High]** The launch handler waits for app
  readiness, runs `run()`, and on success logs, **re-schedules itself**, and
  calls `bgTask.setTaskCompleted(success: true)`.
- **Cancellation / expiration handling** — on `CancellationError` or a
  `BGProcessingTaskRescheduleOnCatch` error, it **re-schedules**
  (`.asSoonAsPossible`) and completes the task with `success: false`, because a
  cancelled task still has work to do. An `expirationHandler` cancels the
  in-flight `Task` so it can wind down during the OS grace period. **[High]**
- **`scheduleBGProcessingTaskIfNeeded()`** (`:116`) — a no-op when
  `startCondition()` is `.never`; otherwise submits the request. **[High]**
- **`scheduleBGProcessingTask(startCondition:)`** (`:125`) — builds the
  `BGProcessingTaskRequest`, sets `requiresNetworkConnectivity` /
  `requiresExternalPower`, and submits off the main thread (per Apple's WWDC
  guidance that `submit` can block). It handles the scheduler error cases
  (`notPermitted`, `tooManyPendingTaskRequests`, `unavailable`) by logging and
  skipping. **[High]**
- **`runInBatches(willBegin:runNextBatch:)`** (`:170`) — a helper for migrations
  that proceed in batches: it loops calling `runNextBatch()` until it returns
  `true`, checking `Task.isCancelled` between batches and throwing
  `CancellationError` if cancelled, so partial progress is preserved. **[High]**
- **`runWithChatConnection(...)`** (`:197`) — runs an operation with a live chat
  connection and a `BackgroundMessageFetcher`, draining incoming messages and
  tearing down gracefully before the process suspends. **[High]**

---

## `LazyDatabaseMigratorRunner`

A concrete `BGProcessingTaskRunner` (`Signal/Storage/LazyDatabaseMigratorRunner.swift:9`)
that runs **deferred database migrations** — migrations deliberately *not* run
during the blocking launch-time migration pass, so they don't delay startup.
**[High]**

- It owns an `InfoMessageGroupUpdateMigrator`, constructed from
  `SDSDatabaseStorage`, `ModelReadCaches`, and `TSAccountManager` (`:12-24`). **[High]**
- `taskIdentifier = "LazyDatabaseMigratorTask"`; `requiresNetworkConnectivity`
  and `requiresExternalPower` are both `false` (`:26-29`). **[High]**
- `startCondition()` returns `.asSoonAsPossible` **only if**
  `infoMessageMigrator.needsToRun()`, otherwise `.never` (`:31-37`) — so the
  task self-suppresses when there is nothing to migrate. **[High]**
- `run()` simply delegates to `infoMessageMigrator.run()` (`:43-45`). **[High]**
- Under `targetEnvironment(simulator)` it defines `simulatePriorCancellation()`
  returning a 1-in-10 random `true` (`:47-52`), a test hook to exercise the
  “task runs when it may already be finished” path on a simulator. **[High]**

> **`InfoMessageGroupUpdateMigrator`** — the migrator this runner drives. It
> exposes `needsToRun()` and `run()` and migrates group-update info messages.
> **[Medium]** — inferred from the runner’s usage and the type name; see
> `InfoMessageGroupUpdateMigrator.swift` for the migration specifics.

---

## `FullTextSearchOptimizer`

Background maintenance of the app's **full-text-search (FTS)** index. **[Medium]**
— from the file name and module placement; the FTS index is GRDB/SQLite-backed
and benefits from periodic optimization, which is exactly the kind of expensive,
deferrable work this module exists to run as a `BGProcessingTask`. See
`FullTextSearchOptimizer.swift` for the concrete optimization steps. **[Low]**
on the precise algorithm — not re-verified line-by-line here.

---

## Conventions observed in this module

- **Deferred-not-inline.** The module's reason to exist is to move heavy DB work
  (migrations, FTS optimization) *out* of launch and into OS-scheduled
  background windows. **[High]** (explicit in the lazy-migrator design and the
  `BGProcessingTaskRunner` doc comments).
- **Incremental, cancellation-aware work.** Runners are expected to survive
  mid-run termination and resume; the extension re-schedules on cancellation and
  the batch helper checks `Task.isCancelled` between batches. **[High]**
- **Self-suppressing scheduling.** `startCondition()` returning `.never` (and
  `scheduleBGProcessingTaskIfNeeded()` honoring it) means no task is scheduled
  when there is no work. **[High]**
- **Register unconditionally, schedule conditionally** — a deliberate pattern
  called out in-code to satisfy Apple's requirement that every `Info.plist`
  task identifier has a registered handler. **[High]**

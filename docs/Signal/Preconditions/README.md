# Signal — Preconditions module

This directory documents the main-app `Preconditions` module, located at
[`Signal/Preconditions/`](../../../Signal/Preconditions/). The module contains a
single source file:

- `Signal/Preconditions/AppActivePrecondition.swift`

> **Confidence labels.** Each claim is tagged with a confidence level:
> - **[High]** — directly read from source in full; behavior is explicit in the code.
> - **[Medium]** — based on a signature/partial read, or depends on collaborators
>   defined outside `Signal/Preconditions/`.
> - **[Low]** — inferred from naming/usage; not fully verified in-tree.
>
> **Citations** are given as `File.swift:line` relative to the repository root.
> Line numbers reflect the state of the tree at authoring time and may drift as
> code changes; use the cited symbol name to re-locate code if lines have moved.

## Responsibility

The module provides exactly one type, `AppActivePrecondition`, a `Precondition`
that is satisfied while the app is in the foreground and active. **[High]**
(`AppActivePrecondition.swift:10`)

`AppActivePrecondition` conforms to `SignalServiceKit`'s `Precondition`
protocol **[High]** (`AppActivePrecondition.swift:10`), whose contract is: when
`waitUntilSatisfied()` returns, the precondition **must** be satisfied (or the
Task was canceled). **[High]** (`SignalServiceKit/Preconditions/Preconditions.swift:9-26`)

It is a thin adapter: its single stored property `_precondition` is a
`NotificationPrecondition`, and all of its behavior is delegated to that value.
**[High]** (`AppActivePrecondition.swift:11`, `AppActivePrecondition.swift:20-22`)

### Construction

`init(appContext:)` builds its backing `NotificationPrecondition` with:

- `notificationName: UIApplication.didBecomeActiveNotification` — the trigger
  that causes the satisfaction check to re-run. **[High]**
  (`AppActivePrecondition.swift:14-15`)
- `isSatisfied: { appContext.isAppForegroundAndActive() }` — the predicate,
  which is considered satisfied while the injected `AppContext` reports the app
  is foregrounded and active. **[High]** (`AppActivePrecondition.swift:16`)

The `AppContext` is injected rather than fetched globally, so the precondition's
notion of "active" comes from whichever context is supplied. **[High]**
(`AppActivePrecondition.swift:12-17`) `isAppForegroundAndActive()` is a method on
the `SignalServiceKit` `AppContext` protocol. **[Medium — defined outside this
module]** (`SignalServiceKit/Util/AppContext.swift:68`) In the main app, that
implementation returns `reportedApplicationState == .active`. **[Medium — defined
outside this module]** (`Signal/AppLaunch/MainAppContext.swift:119`)

### Waiting

`waitUntilSatisfied()` simply forwards to the backing
`NotificationPrecondition.waitUntilSatisfied()` and returns its `WaitResult`.
**[High]** (`AppActivePrecondition.swift:20-22`)

The delegated behavior (defined outside this module) is:

1. Register an observer for `UIApplication.didBecomeActiveNotification`. **[High]**
   (`SignalServiceKit/Preconditions/NotificationPrecondition.swift:25-33`)
2. If `isSatisfied()` already returns `true`, return `.satisfiedImmediately`.
   **[High]** (`SignalServiceKit/Preconditions/NotificationPrecondition.swift:39-40`)
3. Otherwise suspend until the notification fires and `isSatisfied()` becomes
   `true`, then return `.wasNotSatisfiedButIsNow`; return `.canceled` if the Task
   is canceled while waiting. **[High]**
   (`SignalServiceKit/Preconditions/NotificationPrecondition.swift:27-31`,
   `SignalServiceKit/Preconditions/NotificationPrecondition.swift:42-43`,
   `SignalServiceKit/Preconditions/NotificationPrecondition.swift:44`,
   `SignalServiceKit/Preconditions/NotificationPrecondition.swift:46`)

`WaitResult` is `Precondition`'s typealias for `_PreconditionWaitResult`, whose
cases are `.satisfiedImmediately`, `.wasNotSatisfiedButIsNow`, and `.canceled`.
**[High]** (`SignalServiceKit/Preconditions/Preconditions.swift:10`,
`SignalServiceKit/Preconditions/Preconditions.swift:29-39`)

## Where this type is used

`AppActivePrecondition` is consumed by main-app callers that wrap it in a
`SignalServiceKit` `Preconditions` collection and await it before doing
foreground-only work:

- `FullTextSearchOptimizer` stores
  `Preconditions([AppActivePrecondition(appContext: appContext)])` and awaits it
  so FTS maintenance runs only while active. **[Medium — defined outside this
  module]** (`Signal/Storage/FullTextSearchOptimizer.swift:29`)
- `AuthorMergeHelperBuilder` awaits
  `Preconditions([AppActivePrecondition(appContext: appContext)]).waitUntilSatisfied()`
  before proceeding. **[Medium — defined outside this module]**
  (`Signal/Contacts/AuthorMergeHelperBuilder.swift:68`)

The `Preconditions` aggregator re-checks all preconditions whenever any one of
them transitions from unsatisfied to satisfied, so an `AppActivePrecondition`
combined with other preconditions is re-evaluated if the app is backgrounded
while waiting. **[High]** (`SignalServiceKit/Preconditions/Preconditions.swift:41-105`)

## Related modules

- [`SignalServiceKit/Preconditions/`](../../../SignalServiceKit/Preconditions/)
  — defines the `Precondition` protocol, the `Preconditions` aggregator, and the
  `NotificationPrecondition` that this module adapts. **[High]**
  (`SignalServiceKit/Preconditions/Preconditions.swift:9`,
  `SignalServiceKit/Preconditions/Preconditions.swift:41`,
  `SignalServiceKit/Preconditions/NotificationPrecondition.swift:10`)
- `AppContext` — the protocol supplying `isAppForegroundAndActive()`; the main
  app's `MainAppContext` provides the implementation used here. **[Medium —
  defined outside this module]** (`SignalServiceKit/Util/AppContext.swift:68`,
  `Signal/AppLaunch/MainAppContext.swift:119`)

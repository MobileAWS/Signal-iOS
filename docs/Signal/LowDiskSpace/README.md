# `Signal/LowDiskSpace/` — Critically-low-disk-space monitor

Documentation of the main-app `Signal/LowDiskSpace/` subsystem. It contains a
single file, `LowDiskSpaceMonitoring.swift`, which hosts the app-side background
poller that crashes the process when free disk space falls below a critical
threshold.

Consistent with the parent doc set, `SignalServiceKit/` is treated as a
**boundary**: the actual disk-space measurement and policy live in
`SignalServiceKit/LowDiskSpace/LowDiskSpaceManager.swift` and are documented here
only insofar as the app target consumes them.

## Conventions

- **Citations** are `path:line` (specific declaration) or `path` (file). Line
  numbers reflect the working tree at authoring time and may drift; relocate via
  the symbol name.
- **Confidence labels**:
  - **[High]** — directly observed (file read in this session).
  - **[Medium]** — inferred from signatures/names/cross-file convention.
  - **[Low]** — educated inference from naming alone.

## Scope

- `Signal/LowDiskSpace/LowDiskSpaceMonitoring.swift` — the only file in this
  folder; declares `LowDiskSpaceMonitoringManager` **[High]**.

## Responsibility

`LowDiskSpaceMonitoringManager` is "responsible for continuously monitoring for
'critically low disk space' in the main app, and terminating the process if
detected" (`Signal/LowDiskSpace/LowDiskSpaceMonitoring.swift:7-8`, `:10`)
**[High]**. It is purely a scheduling/polling shell; it owns no disk-space logic
of its own and delegates every measurement to the injected
`LowDiskSpaceManager` **[High]**.

## Key type

### `LowDiskSpaceMonitoringManager`

A `class` (not actor; thread-safety is handled with an `AtomicValue`) with:

- **Dependencies / configuration** (constructor-injected,
  `Signal/LowDiskSpace/LowDiskSpaceMonitoring.swift:20-26`):
  - `lowDiskSpaceManager: LowDiskSpaceManager` — the SSK boundary type that
    performs the actual check (`:15`).
  - `monitoringInterval: TimeInterval` — delay between polls (`:17`).
- **State** (`:11-13`, `:18`): a private `struct State` holding an optional
  `monitoringTask: Task<Void, Never>?`, wrapped in
  `AtomicValue(State(), lock: .init())`. The atomic value is the concurrency
  guard around starting/holding the task **[High]**.
- **`start()`** (`:28-39`): under `state.update`, asserts via `owsPrecondition`
  that `monitoringTask == nil` ("Attempted to start monitoring multiple times!")
  and then spawns an unstructured `Task` running
  `continuouslyMonitorDiskSpace()`. Starting twice is a programmer error and
  traps **[High]**.
- **`continuouslyMonitorDiskSpace()`** (`:41-52`): a `while true` loop that, each
  iteration, calls `lowDiskSpaceManager.isDiskSpaceCriticallyLow()` and, if true,
  calls `owsFail("Disk space is critically low; crashing.")` — i.e. it
  intentionally terminates the process (`:43-45`). Otherwise it awaits
  `Task.sleep(nanoseconds: monitoringInterval.clampedNanoseconds)`; any thrown
  error (e.g. cancellation) simply `return`s and ends the loop (`:47-50`)
  **[High]**.

Notes:
- `clampedNanoseconds` is the SSK `TimeInterval` helper at
  `SignalServiceKit/Util/TimeInterval+SSK.swift:48` **[High]**.
- There is no public `stop()`; the only way the loop ends short of a crash is the
  stored `Task` being cancelled/deinitialized, which makes `Task.sleep` throw and
  the loop `return` (`:48-50`) **[Medium]** (inferred from the catch arm; no
  explicit cancellation call observed in this folder).

## Interactions with the rest of the app

### Construction and start (main app only)

`LowDiskSpaceMonitoringManager` is instantiated and owned by `AppEnvironment`:

- `AppEnvironment` holds `private(set) var lowDiskSpaceManager: LowDiskSpaceManager!`
  (`Signal/AppLaunch/AppEnvironment.swift:44`) and
  `private var lowDiskSpaceMonitoringManager: LowDiskSpaceMonitoringManager!`
  (`:45`) **[High]**.
- Both are created during setup: `LowDiskSpaceManager()` then
  `LowDiskSpaceMonitoringManager(lowDiskSpaceManager:monitoringInterval:)` with
  `monitoringInterval: 5 * .second`
  (`Signal/AppLaunch/AppEnvironment.swift:174-178`) **[High]**.
- `start()` is invoked from a `appReadiness.runNowOrWhenAppWillBecomeReady`
  ready-block, alongside the other monitors (`clockSkewMonitoringManager`,
  `screenshotBlockingManager`) (`Signal/AppLaunch/AppEnvironment.swift:361`)
  **[High]**.

This poller is a main-app concern only. The share extension and NSE do not run
it; they perform a one-shot launch check instead (see below) **[High]**.

### Boundary: `LowDiskSpaceManager` (SignalServiceKit)

`LowDiskSpaceManager` is "responsible for periodically checking the device's
available disk space, and taking remedial action if we're running low"
(`SignalServiceKit/LowDiskSpace/LowDiskSpaceManager.swift:6-7`, `:8`) **[High]**.
Relevant surface consumed by the app:

- **`isDiskSpaceCriticallyLow() -> Bool`** (`:56-66`): returns `true` when
  measured free space is below `minBytesAvailableBeforeCriticallyLow`
  (`400 * .megabyte`, `:14`). This is the predicate the monitor polls **[High]**.
- **`static additionalBytesRequiredToLaunch() -> UInt64?`** (`:42-53`): returns
  the shortfall below `minBytesAvailableToLaunch` (`500 * .megabyte`, `:11`) or
  `nil`. Used as a pre-launch gate, not by the monitor **[High]**.
- **`getNeedsWarning(now:tx:)` / `setShowedWarning(now:tx:)`** (`:71-96`): the
  softer, 3-day-throttled warning path (threshold = min of 5% of total or
  `2 * .gigabyte`, `:16-24`), persisted in a `NewKeyValueStore` under collection
  `"LowDiskSpaceWarningManager"` (`:33-36`) **[High]**.
- All three read the volume that holds the GRDB database
  (`SDSDatabaseStorage.grdbDatabaseFileUrl`) via `OWSFileSystem` space queries
  (`:116-134`) **[High]**.

### Other consumers of the boundary (for orientation, outside this folder)

- **Launch preflight gate**: `AppLifecycleManager` calls
  `LowDiskSpaceManager.additionalBytesRequiredToLaunch()`; non-nil yields a
  `.lowStorageSpaceAvailable(bytesRequired:)` `LaunchPreflightError`
  (`Signal/AppLaunch/AppLifecycleManager.swift:1180-1182`) **[High]**.
- **Share extension**: `ShareViewController` refuses to proceed when
  `additionalBytesRequiredToLaunch()` is non-nil
  (`SignalShareExtension/ShareViewController.swift:76`) **[High]**.
- **NSE**: `NotificationService` proceeds only when
  `additionalBytesRequiredToLaunch() == nil`
  (`SignalNSE/NotificationService.swift:104`) **[High]**.
- **In-app warning UI**: `ChatListFYISheetCoordinator` uses
  `getNeedsWarning(now:tx:)` to show a low-disk-space FYI sheet and
  `setShowedWarning(...)` to record it
  (`Signal/src/ViewControllers/HomeView/Chat List/ChatListFYISheetCoordinator.swift:186`,
  `:631`); it receives `AppEnvironment.shared.lowDiskSpaceManager` from
  `ChatListViewController` (`.../ChatListViewController.swift:386`) **[High]**.

These are the manager's consumers, not the monitor's; the folder documented here
contributes only the background crash-on-critical poller **[High]**.

## Data flow / state

Steady-state, once the app is ready:

```
AppEnvironment.setUp
  └─ runNowOrWhenAppWillBecomeReady
       └─ lowDiskSpaceMonitoringManager.start()        (AppEnvironment.swift:361)
            └─ Task { continuouslyMonitorDiskSpace() }  (LowDiskSpaceMonitoring.swift:36-38)
                 └─ loop every 5s (monitoringInterval):
                      ├─ LowDiskSpaceManager.isDiskSpaceCriticallyLow()
                      │     └─ free(grdb volume) < 400 MB ?  (LowDiskSpaceManager.swift:56-66)
                      │          ├─ yes → owsFail("…crashing.")  (LowDiskSpaceMonitoring.swift:44)
                      │          └─ no  → continue
                      └─ Task.sleep(5s); throw → return (loop ends)
```

- **Threshold tiers** (all measured against the GRDB volume) **[High]**:
  - **500 MB** — hard launch gate (`additionalBytesRequiredToLaunch`,
    `LowDiskSpaceManager.swift:11`).
  - **400 MB** — "critically low"; triggers the monitor's crash
    (`:14`).
  - **~2 GB / 5% of total** — soft, throttled in-app warning
    (`:16-24`).
- **Owned state**: only the in-flight `monitoringTask` (atomic, in the monitor)
  and the `lastWarningDate` key-value entry (in `LowDiskSpaceManager`, used by the
  warning path rather than the monitor) **[High]**.
- The monitor is deliberately crash-oriented: there is no graceful remediation in
  this folder — detection equals process termination
  (`LowDiskSpaceMonitoring.swift:44`) **[High]**.

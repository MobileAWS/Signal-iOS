# `Signal/ClockSkew/` — Clock-Skew App Blocking

Documentation of the first-party `Signal/ClockSkew/` subsystem: the main-app glue
that **blocks use of the app while the device's clock is skewed too far from the
server's**, and unblocks it the moment the user corrects their clock.

This subsystem is deliberately thin. The actual skew measurement and the authoritative
"am I skewed?" state live in `SignalServiceKit` (`ClockSkewManager`), which this doc set
treats as a **boundary** — described here only insofar as the app target observes it. The
clock-skew UI window lives in `Signal/AppLaunch/` (`WindowManager`), also a sibling that
this subsystem drives via a single flag. See
[../scene-window-management.md](../scene-window-management.md) for the window stack and
[../app-environment.md](../app-environment.md) for where this manager is constructed/started.

## Scope

In scope — the entire contents of `Signal/ClockSkew/`:

- `ClockSkewMonitoringManager.swift` — the only file in the folder; a main-thread
  observer that mirrors `ClockSkewManager.isClockSkewed` onto
  `WindowManager.isClockSkewBlockActive` (`Signal/ClockSkew/ClockSkewMonitoringManager.swift:10`) **[High]**.

Out of scope (boundaries, cited where they interact):

- `ClockSkewManager` and its notifications/`ClockSkewError`
  (`SignalServiceKit/ClockSkew/ClockSkewManager.swift:35`) — the skew source of truth **[High]**.
- `WindowManager` and the `clockSkewBlocking` window / `ClockSkewAppBlockingViewController`
  (`Signal/AppLaunch/WindowManager.swift:98`) — the UI that gets shown **[High]**.

## Conventions

- **Citations** are `path:line` (specific declaration) or `path` (directory/file).
  Line numbers reflect the working tree at authoring time and may drift; relocate via
  the symbol name. Every factual claim about behavior cites a path.
- **Confidence labels** on claims about *purpose/behavior*:
  - **[High]** — directly observed (file read / declaration located in this session).
  - **[Medium]** — inferred from signatures, names, and cross-file convention; not
    every referenced file was read in full.
  - **[Low]** — educated inference from naming alone.
- Any uncited claim is a **defect**.

## Responsibility

`ClockSkewMonitoringManager` has exactly one job: keep the app's clock-skew blocking
window in sync with the service layer's skew state, on the main thread. It owns no skew
logic of its own — it reads a `Bool` from `ClockSkewManager` and writes a `Bool` to
`WindowManager`, driven by a notification (`Signal/ClockSkew/ClockSkewMonitoringManager.swift:25-30`) **[High]**.

The comment in `updateIsBlocked()` records the design intent: because the skew is
re-measured when the clock changes or the app becomes active, "a user who corrects their
clock gets unblocked without relaunching" (`Signal/ClockSkew/ClockSkewMonitoringManager.swift:42-43`) **[High]**.

## Key type

### `ClockSkewMonitoringManager` (`Signal/ClockSkew/ClockSkewMonitoringManager.swift:10`)

- A plain `class` (not a singleton); holds two injected collaborators:
  `clockSkewManager: ClockSkewManager` and `windowManager: WindowManager`
  (`Signal/ClockSkew/ClockSkewMonitoringManager.swift:11-20`) **[High]**.
- `start()` — asserts main thread, registers for `.clockSkewDidChange`, then calls
  `updateIsBlocked()` once to establish the initial state
  (`Signal/ClockSkew/ClockSkewMonitoringManager.swift:22-33`) **[High]**.
- `clockSkewDidChange()` — the `@objc` notification handler; asserts main thread and
  re-runs `updateIsBlocked()` (`Signal/ClockSkew/ClockSkewMonitoringManager.swift:34-39`) **[High]**.
- `updateIsBlocked()` — the single state transfer:
  `windowManager.isClockSkewBlockActive = clockSkewManager.isClockSkewed`
  (`Signal/ClockSkew/ClockSkewMonitoringManager.swift:41-45`) **[High]**.

Note it only observes `.clockSkewDidChange` (the `isClockSkewed`-changed notification),
not the connection-blocking or re-measure notifications — those are consumed by the
network layer, not the app window (see below) **[High]**.

## Interactions

### Construction and startup (`Signal/AppLaunch/`)

- Constructed in `AppEnvironment` as a non-optional-after-init field
  `clockSkewMonitoringManager` (`Signal/AppLaunch/AppEnvironment.swift:41`), wired with
  `DependenciesBridge.shared.clockSkewManager` and the app's `windowManagerRef`
  (`Signal/AppLaunch/AppEnvironment.swift:142-145`) **[High]**.
- `start()` is invoked from a `runNowOrWhenAppWillBecomeReady` ready-block in
  `AppEnvironment`, alongside the other monitoring managers
  (`Signal/AppLaunch/AppEnvironment.swift:360`) **[High]**.
- The `view-controllers-map` groups it with the other app-readiness/monitoring helpers
  (`docs/Signal/view-controllers-map.md:123`) **[High]**.

### Downstream: the blocking window (`Signal/AppLaunch/WindowManager.swift`)

- Setting `isClockSkewBlockActive` triggers a `didSet` that calls `ensureWindowState()`
  on the main thread (`Signal/AppLaunch/WindowManager.swift:204-209`) **[High]**.
- In `ensureWindowState()`, clock-skew blocking is the third priority: it shows only when
  neither the screen block nor an active call view is up, then shows `clockSkewBlocking`
  and hides every other window (`Signal/AppLaunch/WindowManager.swift:272-278`) **[High]**.
- The window sits at level `_clockSkewBlocking = normal + 1`, deliberately *behind* an
  ongoing call (`Signal/AppLaunch/WindowManager.swift:24`), and hosts
  `ClockSkewAppBlockingViewController`, whose "submit debug logs" action is tagged
  `"ClockSkew"` (`Signal/AppLaunch/WindowManager.swift:98-108`) **[High]**.

### Upstream: the skew source of truth (`SignalServiceKit/ClockSkew/ClockSkewManager.swift`, boundary)

- `isClockSkewed` returns whether the last reported skew exceeds
  `maximumAllowedClockSkew` (= `.day`); the state is held in a lock-guarded `AtomicValue`
  (`SignalServiceKit/ClockSkew/ClockSkewManager.swift:38-61`) **[High]**.
- Skew is (re)measured when the server reports a timestamp over the auth chat connection:
  `serverDidReportTimestamp(_:)` computes `device - server` and updates state
  (`SignalServiceKit/ClockSkew/ClockSkewManager.swift:90-99`); the chat websocket forwards
  the server timestamp here (`docs/SignalServiceKit/Network/02-chat-websocket.md:162`) **[High]**.
- `ClockSkewManager` itself observes `.NSSystemClockDidChange` and
  `.OWSApplicationDidBecomeActive` to re-measure / unblock connections
  (`SignalServiceKit/ClockSkew/ClockSkewManager.swift:72-85`) — this is why the user does
  not need to relaunch after fixing the clock **[High]**.
- When `isClockSkewed` flips, `ClockSkewManager` posts `.clockSkewDidChange` on the main
  thread (`SignalServiceKit/ClockSkew/ClockSkewManager.swift:146-148`) — the one
  notification this subsystem listens for **[High]**.

### Siblings that consume the *other* clock-skew signals (not this subsystem)

For completeness, the two other clock-skew notifications are handled elsewhere, which is
why `ClockSkewMonitoringManager` ignores them:

- `OWSChatConnection` observes `.clockSkewShouldBlockConnectionsDidChange` and
  `.clockSkewShouldBeRemeasured`, and short-circuits connects with `ClockSkewError` when
  `shouldBlockConnections` is set (`SignalServiceKit/Network/OWSChatConnection.swift:116-117, 250-251, 398, 964`) **[High]**.
- The NSE checks `clockSkewManager.isClockSkewed` directly rather than via a window
  (`SignalNSE/NotificationService.swift:248-249`; `docs/SignalServiceKit/Notifications/nse.md:195`) **[Medium]**.
- The share extension special-cases `ClockSkewError` with a dismiss-only message
  (`SignalShareExtension/SharingThreadPickerViewController.swift:457`;
  `docs/SignalShareExtension/SharingThreadPickerViewController.md:110`) **[Medium]**.

## Data flow / state

This subsystem holds **no mutable state of its own**; all state lives in
`ClockSkewManager`. The flow is a one-way mirror:

```
server timestamp (auth chat connection)
   │  OWSChatConnection → ClockSkewManager.serverDidReportTimestamp()
   ▼
ClockSkewManager.State.lastReportedClockSkew  (AtomicValue, lock-guarded)
   │  isSkewed(skew) crosses the .day threshold → changes isClockSkewed
   ▼
NotificationCenter.default.post(.clockSkewDidChange)   [main thread]
   │
   ▼
ClockSkewMonitoringManager.clockSkewDidChange()        [AssertIsOnMainThread]
   │  updateIsBlocked()
   ▼
WindowManager.isClockSkewBlockActive = isClockSkewed   [didSet → ensureWindowState]
   │
   ▼
clockSkewBlocking window shown/hidden (ClockSkewAppBlockingViewController)
```

Citations for the chain: `SignalServiceKit/ClockSkew/ClockSkewManager.swift:90-99`
(measure) → `:148` (post `.clockSkewDidChange`) →
`Signal/ClockSkew/ClockSkewMonitoringManager.swift:34-45` (observe + transfer) →
`Signal/AppLaunch/WindowManager.swift:204-209, 272-278` (window) **[High]**.

Threading: every hop in this subsystem asserts or runs on the main thread —
`ClockSkewManager.postNotification` posts via `postOnMainThread`
(`SignalServiceKit/ClockSkew/ClockSkewManager.swift:155-161`), and both
`start()`/`clockSkewDidChange()` call `AssertIsOnMainThread()`
(`Signal/ClockSkew/ClockSkewMonitoringManager.swift:23, 36`) **[High]**.

Lifecycle: there is no `stop()`/`removeObserver`; the observer is registered for the
process lifetime of the main app, consistent with the manager being a long-lived field on
`AppEnvironment` (`Signal/AppLaunch/AppEnvironment.swift:41`) **[High]**.

## Tests

The folder `Signal/ClockSkew/` contains no tests. The skew logic is covered by
`SignalServiceKit/tests/ClockSkew/ClockSkewManagerTest.swift`, which asserts the threshold
behavior and the exact notifications posted on measure/activate transitions
(e.g. `[.clockSkewDidChange, .clockSkewShouldBlockConnectionsDidChange]`)
(`SignalServiceKit/tests/ClockSkew/ClockSkewManagerTest.swift:71, 82, 100`) **[High]**.

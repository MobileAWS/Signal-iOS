# ShareAppExtensionContext.swift

Source: [`SignalShareExtension/ShareAppExtensionContext.swift`](../../SignalShareExtension/ShareAppExtensionContext.swift)

## Purpose

**[High]** The extension's implementation of the shared `AppContext` protocol. Shared
code (`SignalServiceKit`, `SignalUI`) calls through `CurrentAppContext()` to learn where
data lives, whether it is the main app, the application state, etc. This type makes those
answers correct for the share-extension process — pointing storage at the **app-group
container** and disabling main-app-only behaviours.

> **[High]** A header comment states: "This is _NOT_ a singleton and will be instantiated
> each time that the SAE is used." It is created in
> [`ShareViewController.loadView()`](ShareViewController.md) and installed via
> `SetCurrentAppContext`.

## Type

### `ShareAppExtensionContext` (`final class: NSObject`, extends `AppContext`)

Stored state **[High]**:
- `rootViewController: UIViewController` — supplied at init; used for `frame` and
  `frontmostViewController()`.
- `mainWindow: UIWindow?`.
- `@Atomic internalReportedApplicationState: UIApplication.State` — defaults `.active`,
  updated by host-lifecycle notifications.
- `appLaunchTime = Date()`.
- `notificationCenterObservers: [NSObjectProtocol]`.
- `isRTL` (static) — computed from `Bundle.main.preferredLocalizations` because extensions
  may not touch `UIApplication.shared`.

## Initialization and lifecycle bridging

### `init(rootViewController:)` — [`:30`](../../SignalShareExtension/ShareAppExtensionContext.swift)

**[High]** Registers four observers on `NotificationCenter.default`, each mapping an
**extension-host** notification to the app's internal state + an **OWS** notification
(re-posted inside `BenchManager.bench` to log slow posts):

| Host notification | sets state | re-posts |
| --- | --- | --- |
| `NSExtensionHostDidBecomeActive` | `.active` | `OWSApplicationDidBecomeActive` |
| `NSExtensionHostWillResignActive` | `.inactive` | `OWSApplicationWillResignActive` |
| `NSExtensionHostDidEnterBackground` | `.background` | `OWSApplicationDidEnterBackground` |
| `NSExtensionHostWillEnterForeground` | `.inactive` | `OWSApplicationWillEnterForeground` |

**Cross-process concern:** this is the bridge that lets shared code (and
[`ShareViewController.applicationDidEnterBackground`](ShareViewController.md)) observe the
host's lifecycle via Signal's own notification names. In particular, the background
re-post drives the screen-lock dismissal.

### `deinit`

**[High]** Removes all registered observers.

## AppContext conformance (selected members)

**[High]** Values that mark this as a non-main-app, extension context:

| Member | Value / behaviour | Line |
| --- | --- | --- |
| `type` | `.share` | [`:115`](../../SignalShareExtension/ShareAppExtensionContext.swift) |
| `isMainAppAndActive` / `...Isolated` | `false` | — |
| `isRunningTests` | `false` | — |
| `reportedApplicationState` | `internalReportedApplicationState` | — |
| `isInBackground()` | state `== .background` | — |
| `isAppForegroundAndActive()` | state `== .active` | — |
| `frame` | `rootViewController.view.frame` | — |
| `frontmostViewController()` | `rootViewController.findFrontmostViewController(ignoringAlerts: true)` | — |
| `hasUI` | `true` | — |
| `canPresentNotifications()` | `false` | — |
| `shouldProcessIncomingMessages` | `false` | [`:194`](../../SignalShareExtension/ShareAppExtensionContext.swift) |
| `debugLogsDirPath` | `DebugLogger.shareExtensionDebugLogsDirPath` | — |

### Disabled / unsupported operations **[High]**

- `beginBackgroundTask(with:)` → returns `.invalid`
  ([`:139`](../../SignalShareExtension/ShareAppExtensionContext.swift)); `endBackgroundTask`
  asserts the id is `.invalid`. Extensions have no background-task budget here.
- `openSystemSettings()` and `open(_:completion:)` → no-ops (extensions can't open URLs
  the way the app can).
- `runNowOrWhenMainAppIsActive(_:)` → `owsFailBeta`
  ([`:152`](../../SignalShareExtension/ShareAppExtensionContext.swift)); cannot run main-app
  blocks from the extension.
- `mainApplicationStateOnLaunch()` → `owsFailBeta`, returns `.inactive`.

## App-group / shared storage (the core cross-process mechanism)

**[High]** All shared storage paths resolve from the App Group container identified by
`TSConstants.applicationGroup`:

- `appSharedDataDirectoryPath()` — [`:168`](../../SignalShareExtension/ShareAppExtensionContext.swift) —
  `FileManager.containerURL(forSecurityApplicationGroupIdentifier: TSConstants.applicationGroup)`;
  `owsFail` if missing.
- `appDatabaseBaseDirectoryPath()` — [`:179`](../../SignalShareExtension/ShareAppExtensionContext.swift) —
  same directory; this is where the shared GRDB database lives.
- `appUserDefaults()` — [`:183`](../../SignalShareExtension/ShareAppExtensionContext.swift) —
  `UserDefaults(suiteName: TSConstants.applicationGroup)!`, the suite shared with the main app.
- `appDocumentDirectoryPath()` — the extension's own documents dir (not shared);
  `owsFail` if missing.

Because the database and defaults suite are the same objects the main app uses, the
extension sees the main app's registration state, threads, identity keys, and so on. The
app-group identifiers are declared in
[`SignalShareExtension.entitlements`](../../SignalShareExtension/SignalShareExtension.entitlements).

## Side-effects summary

- **[High]** Mutates `internalReportedApplicationState` from host notifications.
- **[High]** Posts `OWSApplication*` notifications observed across shared code.
- **[High]** `owsFail`/`owsFailBeta` on unsupported operations or missing containers.

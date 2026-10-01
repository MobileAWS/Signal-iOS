# ShareViewController.swift

Source: [`SignalShareExtension/ShareViewController.swift`](../../SignalShareExtension/ShareViewController.swift)

## Purpose

**[High]** The extension's **principal class** and root view controller
([`Info.plist`](../../SignalShareExtension/Info.plist) → `NSExtensionPrincipalClass`).
It is an `OWSNavigationController` that also conforms to
[`ShareViewDelegate`](ShareViewDelegate.md). It owns the whole SAE lifecycle:
installing the app context, booting shared infrastructure, gating on screen lock,
building attachments, driving the picker/approval flow, and tearing the process
down when finished.

## Types

### `ShareViewController` (`public class`, `OWSNavigationController`, `ShareViewDelegate`)

**[High]** Hub view controller. Holds:
- `appReadiness = AppReadinessImpl()` — the extension's own readiness object.
- `connectionTokens: [OWSChatConnection.ConnectionToken]` — chat-connection
  requests kept alive for the process lifetime.
- `initialLoadViewController: SAELoadViewController?` — the first loading screen.

### `ShareViewControllerError` (nested `enum: Error`)

**[High]** Error cases used to cancel/complete the extension:
`obsoleteShare`, `screenLockEnabled`, `tooManyAttachments`, `nilInputItems`,
`noInputItems`, `noConformingInputItem`, `nilAttachments`, `noAttachments`.

## Lifecycle methods

### `loadView()` — [`:34`](../../SignalShareExtension/ShareViewController.swift)

**[High]** Runs first. Side-effects, in order:
1. Creates `ShareAppExtensionContext(rootViewController: self)` and installs it
   globally with `SetCurrentAppContext(...)`. This is explicitly "the first thing we
   do" so all shared code can resolve app-group paths.
2. Enables TTY + file logging via `DebugLogger.shared` (file logging with
   `canLaunchInBackground: false`) and `DebugLogger.registerLibsignal()`.
3. Shows an initial `SAELoadViewController`, mimicking the recipient picker header
   only when there is no `INSendMessageIntent` (`extensionContext?.intent == nil`).

### `viewDidAppear(_:)` — [`:56`](../../SignalShareExtension/ShareViewController.swift)

**[High]** Takes `initialLoadViewController` (consuming it via `.take()`) and, on the
next run loop (`DispatchQueue.main.async`), launches `Task { setUp(...) }`. The run-loop
hop ensures the spinner is painted even if `setUp` blocks the main thread. Errors from
`setUp` are logged, not surfaced (the individual error paths inside `setUp` present UI).

### `setUp(initialLoadViewController:)` — [`:74`](../../SignalShareExtension/ShareViewController.swift)

**[High]** The main boot sequence. Steps and their side-effects / error paths:

1. **Low disk space guard** — if `LowDiskSpaceManager.additionalBytesRequiredToLaunch()`
   is non-nil, calls `showLowDiskSpaceView()` and returns (blocking view).
2. **Database open** — builds `KeychainStorageImpl` and `SDSDatabaseStorage` against
   `SDSDatabaseStorage.grdbDatabaseFileUrl`. On failure → `showNotRegisteredView()` and
   return. **[Medium]** The DB lives in the app-group container (see
   [ShareAppExtensionContext.md](ShareAppExtensionContext.md)).
3. **Shared boot** — `AppSetup().start(...).migrateDatabaseSchema().initGlobals(...)`
   wiring extension-appropriate no-op collaborators
   (`NoopCallMessageHandler`, `CurrentCallNoOpProvider`, `NoopNotificationPresenterImpl`,
   `PaymentsEventsAppExtension`, `MobileCoinHelperMinimal`, no battery/sleep managers).
4. **UI env** — `SUIEnvironment.shared.setUp(...)`.
5. **Data migrations + caches** — `migrateDatabaseData()` then
   `runLaunchTasksIfNeededAndReloadCaches()`.
6. **Registration check** — `setUpLocalIdentifiers(...)`:
   `.corruptRegistrationState` → `showNotRegisteredView()`, return;
   `nil` → `setAppIsReady()`.
7. **Screen lock gate** — if `ScreenLock.shared.isScreenLockEnabled()`, present
   `SAEScreenLockViewController` inside a `withCheckedContinuation`; if `!didUnlock`
   → `shareViewWasCancelled()` and return. Sets `didDisplaceInitialLoadViewController`.
8. **Attachment providers** — `buildTypedItemProviders()`; on throw →
   `presentAttachmentError(error)` and return.
9. **Chat connection** — requests an *unidentified* connection
   (`requestUnidentifiedConnection()`), token retained in `connectionTokens`, for bulk
   identity-key lookups.
10. **Picker** — builds `SharingThreadPickerViewController` with a stories-compat
    precheck and attachment limits.
11. **Routing** — `fetchPreSelectedThread()`:
    - no pre-selected thread → push the picker immediately; spinner shows on it.
    - pre-selected thread + displaced load VC → new `SAELoadViewController` for progress.
    - pre-selected thread + initial load VC still shown → reuse it for progress.
12. **Build + validate** — `buildAndValidateAttachments(...)` with a 200ms-delayed
    "show load view" `async let` race (if building is fast, the load view is never
    swapped in). On throw → `presentAttachmentError(error)`.
13. **Hand off** — sets `conversationPicker.typedItems`; if pre-selected thread, builds
    the approval VC directly and uses `ObjectRetainer.retainObject(conversationPicker,
    forLifetimeOf: approvalViewController)` so the off-hierarchy picker (the "brains")
    isn't deallocated.
14. **Observe background** — adds observer for `.OWSApplicationDidEnterBackground`.

### `applicationDidEnterBackground()` — [`:258`](../../SignalShareExtension/ShareViewController.swift)

**[High]** On `OWSApplicationDidEnterBackground`, if screen lock is enabled, dismisses
with `ShareViewControllerError.screenLockEnabled`. Protects content from showing after
the device locks. Asserts main thread. See
[screen-lock section in README](README.md#screen-lock-handling).

### `setAppIsReady()` — [`:269`](../../SignalShareExtension/ShareViewController.swift)

**[High]** Preconditions `!appReadiness.isAppReady`, then `appReadiness.setAppIsReady()`
(runs deferred blocks). Logs local ACI, dumps app version, calls
`updateFirstVersionIfNeeded()` and `saeLaunchDidComplete()`.

### `viewDidDisappear(_:)` — [`:520`](../../SignalShareExtension/ShareViewController.swift)

**[High]** If nothing is being presented (`presentedViewController == nil`), treats the
disappearance as the user leaving and calls `shareViewWasCancelled()`. Guards against
cancelling when it merely presented a child (e.g. image editing tools).

## ShareViewDelegate conformance

**[High]**
- `shareViewNavigationController` → `self`.
- `shareViewWillSend()` — requests an *identified* chat connection (token retained);
  called right before sending.
- `shareViewWasCompleted()` → `dismissAndCompleteExtension(error: nil)`.
- `shareViewWasCancelled()` → `dismissAndCompleteExtension(error: .obsoleteShare)`.
- `shareViewFailed(error:)` → `owsFailDebug` + `dismissAndCompleteExtension(error:)`.

## Process teardown

### `dismissAndCompleteExtension(error:)` — [`:355`](../../SignalShareExtension/ShareViewController.swift)

**[High]** Calls `extensionContext?.cancelRequest(withError:)` on error or
`completeRequest(returningItems: [], completionHandler: nil)` on success, then flushes
logs and calls **`exit(0)`** ([`:370`](../../SignalShareExtension/ShareViewController.swift)).
The comment states share extension processes may be reused and the codebase's statics
can't be safely cleaned, so the process is killed. **Cross-process concern:** this is a
hard process exit rather than a graceful return to the host.

## Attachment helpers

- `fetchPreSelectedThread()` — reads an `INSendMessageIntent.conversationIdentifier` and
  resolves the `TSThread` via `TSThread.fetchViaCache` in a DB read. **[High]**
- `buildTypedItemProviders()` — reads `extensionContext?.inputItems`, selects one
  extension item, maps attachments to `TypedItemProvider`; prefers a web-URL candidate,
  else visual-media candidates, else the first. Throws `nilInputItems`/`nilAttachments`/
  `noAttachments` on empty inputs. **[High]**
- `selectExtensionItem(_:)` — picks the item carrying `public.data`/`public.fileURL`/
  `com.apple.pkpass` when Safari splits a share into two items; throws `noInputItems`/
  `noConformingInputItem`. **[High]**
- `buildAndValidateAttachments(...)` — sets up nested `Progress`, calls
  `buildAttachments`, checks cancellation, enforces
  `MessageBodyAttachmentLimits.maxAllowedVisualMedia` (throws `tooManyAttachments`). **[High]**
- `buildAttachments(...)` (`@concurrent`) — builds attachments **serially** on purpose; a
  code comment notes `SignalAttachment` loads/resizes in RAM and parallelism could
  exhaust memory. **[High]**
- `presentAttachmentError(_:)` — `tooManyAttachments` → max-items toast; otherwise a
  generic "unable to build attachment" action sheet. Both cancel the share. **[High]**

## Blocking views

**[High]** `showNotRegisteredView()`, `showLowDiskSpaceView()`, and the shared
`showBlockingView(_:)` present an `AppBlockingViewController` with a cancel button wired
to `shareViewWasCancelled()`.

## Cross-process / shared-code notes

- **[High]** Installs the global `AppContext` before anything else; everything shared
  depends on it (see [ShareAppExtensionContext.md](ShareAppExtensionContext.md)).
- **[Medium]** Runs the same `AppSetup` + migrations as the main app because it opens the
  shared app-group database and may be first to open it after an update.
- **[High]** Requests unidentified then (at send time) identified chat connections and
  retains the tokens for the process lifetime.

# Notification Service Extension (NSE)

Covers the `SignalNSE/` target:
- `SignalNSE/NotificationService.swift`
- `SignalNSE/NSEEnvironment.swift`
- `SignalNSE/NSEContext.swift`
- `SignalNSE/NSECallMessageHandler.swift`
- `SignalNSE/NSELogger.swift`

> Confidence labels: **[High]** read from source · **[Medium]** inferred from
> strong local evidence · **[Low]** plausible but unverified. Where intent is not
> evidenced in source, the text says **"intent undetermined — no evidence in
> source."** Citations use `SignalNSE/<file>:line`.

---

## 1. What the NSE is for

The Notification Service Extension is a separate App Extension process iOS wakes
when an encrypted push arrives. Because Signal pushes carry no plaintext, the NSE
must set up (a slice of) the full app environment, fetch and **decrypt** the
pending messages from the chat server, run side effects (which post the real
local notifications via `NotificationPresenter`), update the badge, and complete
within the OS time budget.

[High] The lifecycle is documented in a header comment
(`SignalNSE/NotificationService.swift:9-30`):
1. App receives a push.
2. System instantiates the extension class and calls `didReceive` in the
   background.
3. The extension processes messages / displays notifications.
4. It signals completion by calling the `contentHandler`.
5. If it exceeds ~30s, it is warned (`serviceExtensionTimeWillExpire`) and then
   terminated.

The same comment warns that the NSE does **not** always spawn a new process per
notification, may process notifications in **parallel**, and `didReceive` may be
called multiple times (always on different threads, possibly on the same or
different `NotificationService` instance). [High] Consequently a **process-global**
`NSEEnvironment` singleton is kept so app context / DB / logging are set up once
per process (`SignalNSE/NotificationService.swift:31`). [High]

---

## 2. Push → NSE → notification flow

```mermaid
sequenceDiagram
    participant APNs
    participant OS as iOS
    participant NS as NotificationService
    participant ENV as NSEEnvironment
    participant Fetch as BackgroundMessageFetcher
    participant NP as NotificationPresenterImpl
    participant UNC as UNUserNotificationCenter

    APNs->>OS: Encrypted push
    OS->>NS: didReceive(request, contentHandler)
    NS->>NS: nseDidStart(); store contentHandler
    NS->>NS: fetchQueue.enqueueCancellingPrevious
    NS->>ENV: setUpLogging(logger)
    alt Low disk space
        NS->>UNC: post nse_lowDiskSpace (once)
        NS-->>OS: contentHandler(empty content)
    else
        NS->>ENV: setUpDatabase(logger)
        alt Keychain not allowed (device locked)
            NS->>UNC: post nse_dbNotAvailable (once)
            NS-->>OS: contentHandler(empty content)
        else DB ready
            NS->>ENV: reload caches, set up local identifiers
            alt Not registered
                NS-->>OS: contentHandler(empty content)
            else Registered
                ENV->>ENV: setAppIsReady()
                opt verification-code push
                    NS->>NS: store timestamp; skip fetch
                    NS-->>OS: contentHandler(empty content)
                end
                NS->>Fetch: start() + waitForFetchingProcessingAndSideEffects()
                Fetch->>NP: side effects post real notifications
                NP->>UNC: add notification requests
                alt Clock skewed
                    NS->>UNC: post nse_clockSkew (once)
                end
                NS->>NS: badgeCountFetcher.fetchBadgeCount
                NS-->>OS: contentHandler(content with badge)
            end
        end
    end
    Note over NS,OS: serviceExtensionTimeWillExpire cancels fetch,<br/>completes silently with empty content
```

---

## 3. `NotificationService` — the entry point

`SignalNSE/NotificationService.swift`, subclass of `UNNotificationServiceExtension`.

### 3.1 Concurrency state

[High] (`SignalNSE/NotificationService.swift:36-73`)
- `contentHandler: AtomicOptional<ContentHandler>` — the OS-supplied completion,
  stored atomically and swapped to `nil` on completion so it's called at most once.
- `fetchQueue: SerialTaskQueue` — serializes fetch work; `didReceive` uses
  `enqueueCancellingPrevious`, so a newer request **cancels** the previous one.
  [High]
- `nseDidStart()` / `nseDidComplete()` maintain a static reference-counted
  `_nseCounter` under an `UnfairLock`, and (only if `DebugFlags.internalLogging`)
  manage a 1-second repeating `OffMainThreadTimer` that logs memory usage until
  the counter hits zero (`:42-73`). [High]

### 3.2 `didReceive(_:withContentHandler:)`

[High] (`SignalNSE/NotificationService.swift:83-91`)
Creates a per-request `NSELogger`, calls `nseDidStart()`, stores the content
handler, then enqueues (cancelling previous) a task that awaits `_didReceive`
and then `completeSilently(content:logger:)`.

### 3.3 `completeSilently(content:logger:)`

[High] (`SignalNSE/NotificationService.swift:71-81`) Thread-safe: flushes the
logger, atomically swaps the `contentHandler` to `nil` (no-op if already called),
calls `nseDidComplete()`, and invokes the handler with the given content. The
"silent" naming reflects that it never shows a banner itself — it hands iOS a
content object (often empty or badge-only). [High]

### 3.4 `serviceExtensionTimeWillExpire()`

[High] (`SignalNSE/NotificationService.swift:182-194`) On the OS "about to be
terminated" callback: logs, `fetchQueue.cancelAll()`, and
`completeSilently(content: UNMutableNotificationContent())` with **empty**
content. The comment explains completing silently prevents iOS from presenting
the raw original push content to the user. [High]

### 3.5 `_didReceive(_:logger:)` — the main sequence (`@MainActor`)

[High] (`SignalNSE/NotificationService.swift:101-166`), in order:

1. **Logging:** `globalEnvironment.setUpLogging(logger:)`.
2. **Low-disk-space check:** `LowDiskSpaceManager.additionalBytesRequiredToLaunch()`.
   If space is needed, post `nse_lowDiskSpace` (once, guarded by
   `hasShownLowDiskSpaceWarning`) and return empty content. If space is fine,
   reset the "already shown" flag. [High]
3. **Database setup:** `globalEnvironment.setUpDatabase(logger:)`.
   - On `KeychainError.notAllowed` (device locked → keychain inaccessible): post
     `nse_dbNotAvailable` (once, guarded by `hasShownDBNotAvailableWarning`) and
     return empty content. [High]
   - On any other error: `owsFail("Couldn't load database: …")` (fatal). [High]
4. **Warm caches & identifiers:** `runLaunchTasksIfNeededAndReloadCaches()` to
   pick up changes made by the main app, then `setUpLocalIdentifiers(...)`.
   - `.corruptRegistrationState` → log and return empty content (don't process
     when not registered). [High]
   - `nil` (success) → `globalEnvironment.setAppIsReady()`. [High]
5. **Verification-code push shortcut:** if
   `verificationCodeRequestTimestampMs(userInfo:)` is present
   (`userInfo["verificationCodeRequested"]["timestamp"]`), write the timestamp via
   `SafetyTipsManager.setLastVerificationCodeRequestedTimestampMs` — but **only on
   the primary device** (else log + ignore) — then skip fetch and return empty
   content. [High]
6. **Mark APNs working:** concurrently (`async let`) write
   `APNSRotationStore.didReceiveAPNSPush(transaction:)`; the comment says to do
   this as early as possible but after the app is ready / migrations run. [High]
7. **Fetch & process:** `await fetchAndProcessMessages(logger:)`, then await the
   APNs-received write, and return its content. [High]

### 3.6 `fetchAndProcessMessages(logger:)` — fetch/decrypt path (`@MainActor`)

[High] (`SignalNSE/NotificationService.swift:209-...`):

1. **App-expiry gate:** if `appExpiry.isExpired(now:)`, log and return empty
   content (don't fetch for an expired build). [High]
2. **Signal proxy:** `startProxyIfEnabled(logger:)` — if `SignalProxy.isEnabled`,
   start the relay server and wait (via `Preconditions` /
   `NotificationPrecondition` on `.isSignalProxyReadyDidChange`) until
   `SignalProxy.isEnabledAndReady`; a `defer` stops the relay server afterward.
   [High]
3. **Message fetching:** build a fetcher from
   `DependenciesBridge.shared.backgroundMessageFetcherFactory.buildFetcher()`,
   `await backgroundMessageFetcher.start()`, kick off `cron.runOnce(ctx:)`
   concurrently, then
   `await backgroundMessageFetcher.waitForFetchingProcessingAndSideEffects()`
   (captured into a `Result`). Then await cron, call
   `stopAndWaitBeforeSuspending()`, and `fetchResult.get()` to surface errors.
   [High]
   - The **decryption + message processing + side effects** (which include
     posting the real notifications through `NotificationPresenterImpl`) are
     performed by the `backgroundMessageFetcher` pipeline, defined outside this
     target. [Medium — the method name
     `waitForFetchingProcessingAndSideEffects` and the NSE wiring of a
     `NotificationPresenterImpl` make this strongly evident; the pipeline body is
     not in `SignalNSE/`]
4. **Cancellation:** `catch is CancellationError` → log and return empty content.
   Other errors are logged (not fatal). [High]
5. **Clock-skew check:** if `clockSkewManager.isClockSkewed`, post `nse_clockSkew`
   (once, guarded by `hasShownClockSkewWarning`); otherwise clear the flag.
   The comment explains a skewed clock prevents fetching, so the user is warned.
   [High]
6. **Badge update:** read `badgeCountFetcher.fetchBadgeCount(tx:)` and set
   `content.badge = unreadTotalCount` on a fresh `UNMutableNotificationContent`,
   returned as the completion content. [High]

### 3.7 Ad-hoc NSE notifications

[High] (`SignalNSE/NotificationService.swift` bottom helpers) The NSE posts its
own error notifications **without** going through `NotificationPresenter`,
because singletons may be unavailable: `postNotification(category:body:identifier:)`
builds a `UNMutableNotificationContent` with the category + body and a **fixed
identifier** (so duplicates coalesce), adds it to
`UNUserNotificationCenter.current()`, `owsFailDebug` on error. [High]
Wrappers: `postLowDiskSpaceNotification` (`nse_lowDiskSpace`, id
`"Signal.LowDiskSpaceNotification"`), `postDBNotAvailableNotification`
(`nse_dbNotAvailable`, id `"Signal.DBNotAvailableNotification"`, body embeds the
device model), `postClockSkewNotification` (`nse_clockSkew`, id
`"Signal.ClockSkewNotification"`). The `hasShown…` flags are `@MainActor` globals
preventing repeat posts within the process. [High]

---

## 4. `NSEEnvironment` — process-wide setup

`SignalNSE/NSEEnvironment.swift`.

### 4.1 Construction

[High] (`SignalNSE/NSEEnvironment.swift:9-16`) On `init`, creates an `NSEContext`,
installs it as the current app context via `SetCurrentAppContext(_, isRunningTests:
false)`, and creates an `AppReadinessImpl`. A single instance is held as the
process-global `globalEnvironment` in `NotificationService.swift`. [High]

### 4.2 `setUpLogging(logger:)` (`@MainActor`)

[High] (`SignalNSE/NSEEnvironment.swift:28-41`) Guarded by `didStartAppSetup` so
the one-time setup runs once per process: enables file logging
(`canLaunchInBackground: true`), TTY logging if needed, and registers libsignal +
RingRTC loggers. Then logs pid + memory usage and flushes. [High]

### 4.3 `setUpDatabase(logger:) async throws -> AppSetup.FinalContinuation` (`@MainActor`)

[High] (`SignalNSE/NSEEnvironment.swift:44-78`) Memoized via `finalContinuation`
(returns the cached one on subsequent calls). Otherwise:
1. Build `KeychainStorageImpl(isUsingProductionService:)`.
2. Construct `SDSDatabaseStorage(appReadiness:databaseFileUrl:keychainStorage:)`
   at `SDSDatabaseStorage.grdbDatabaseFileUrl`; `setUpDatabasePathKVO()`.
   (This is where `KeychainError.notAllowed` propagates when the device is
   locked.) [Medium — the throwing call is here; the specific error is caught in
   `NotificationService._didReceive`]
3. Run `AppSetup().start(appContext:databaseStorage:).migrateDatabaseSchema()
   .initGlobals(...).migrateDatabaseData()`. [High]
4. Cache and return the `FinalContinuation`. [High]

**`initGlobals` wiring (NSE-specific dependencies)** (`SignalNSE/NSEEnvironment.swift:63-73`):
- `deviceBatteryLevelManager: nil`, `deviceSleepManager: nil`
- `paymentsEvents: PaymentsEventsAppExtension()`
- `mobileCoinHelper: MobileCoinHelperMinimal()`
- `callMessageHandler: NSECallMessageHandler()`  ← see §6
- `currentCallProvider: CurrentCallNoOpProvider()`
- `notificationPresenter: NotificationPresenterImpl()`  ← the NSE posts real
  notifications through the same impl as the main app. [High]

### 4.4 `setAppIsReady()` (`@MainActor`)

[High] (`SignalNSE/NSEEnvironment.swift:81-93`) Idempotent (returns if already
ready). Calls `appReadiness.setAppIsReady()` (which also runs deferred blocks),
then dumps/updates app version info and calls `appVersion.nseLaunchDidComplete()`.
[High]

---

## 5. `NSEContext` — the `AppContext` for the extension

`SignalNSE/NSEContext.swift`. [High] Implements `AppContext` with
`type = .nse` (`SignalNSE/NSEContext.swift:9-12`). Key behaviors:

- `isMainAppAndActive = false`, `isInBackground() = true`,
  `isAppForegroundAndActive() = false`, `mainApplicationStateOnLaunch() = .inactive`.
  [High]
- `shouldProcessIncomingMessages = true`, `hasUI = false`,
  `canPresentNotifications() = true`. [High]
- Directory paths resolve to the shared **App Group** container
  (`containerURL(forSecurityApplicationGroupIdentifier: TSConstants.applicationGroup)`),
  and `appUserDefaults()` uses the same suite; both `owsFail` if unavailable.
  `appDatabaseBaseDirectoryPath()` == shared data dir. [High]
- **Memory-pressure monitoring:** a `DispatchSource.makeMemoryPressureSource`
  logs normal/warning/critical pressure + current memory usage via the
  uncorrelated `NSELogger` (`SignalNSE/NSEContext.swift:49-71`,
  helper extension at bottom). [High]
- Many `AppContext` members are no-ops/invalid for an extension:
  `beginBackgroundTask` → `.invalid`, `frontmostViewController()` → `nil`,
  `openSystemSettings()`/`open(_:)`/`runNowOrWhenMainAppIsActive` → no-op,
  `mainWindow = nil`, `frame = .zero`, `reportedApplicationState = .background`.
  [High]
- `debugLogsDirPath` → `DebugLogger.nseDebugLogsDirPath`. [High]

Because `isMainAppAndActive` is `false`, the `NotificationPresenterImpl`
suppression logic computes a `nil` suppression rule in the NSE (it only inspects
the frontmost VC when main-app-and-active), so notifications are not
view-suppressed in the extension. See
[notification-presenter.md §5.4](./notification-presenter.md#54-shouldpresentnotification--view-based-suppression).
[Medium — follows from `_notificationSuppressionRuleIfMainAppAndActive` guarding
on `CurrentAppContext().isMainApp` / `isMainAppAndActive`]

---

## 6. `NSECallMessageHandler` — call messages in the extension

`SignalNSE/NSECallMessageHandler.swift`. [High] A `CallMessageHandler` whose job
is to decide whether an incoming call message should wake the **main app**, and
to hand it off via a VoIP push relay.

### 6.1 Collaborators

[High] (`SignalNSE/NSECallMessageHandler.swift:14-20`) Lazily resolves
`databaseStorage`, `groupCallManager`, `identityManager`,
`messagePipelineSupervisor`, `notificationPresenter` (force-cast to
`NotificationPresenterImpl`), `profileManager`, `tsAccountManager` from the shared
environment. [High]

### 6.2 `receivedEnvelope(...)` — routing by `CallEnvelopeType`

[High] (`SignalNSE/NSECallMessageHandler.swift:27-...`):

Computes a RingRTC "message age" from server timestamps plus a 10-second
`bufferSecondsForMainAppToAnswerRing`. Then switches:

- **`.offer`:** requires `offer.opaque`; constructs a `CallOfferHandlerImpl` and
  `startHandlingOffer(...)`. If RingRTC's `isValidOfferMessage(opaque:messageAgeSec:
  callMediaType:)` says the offer is **not valid** (e.g. too old), it logs
  "missed a call because it's not valid" and inserts a **missed-call interaction**
  (`.incomingMissed`) rather than ringing, then returns. Valid offers fall through
  to the relay. [High]
- **`.opaque`:** only handled when `urgency == .handleImmediately` and
  `isValidOpaqueRing(opaqueCallMessage:messageAgeSec:validateGroupRing:)` passes.
  `validateGroupRing` (a DB read) accepts rings from our own linked devices
  unconditionally (important for cancellations); otherwise it validates the group
  id, that the group thread exists, that `GroupMessageProcessorManager.discardMode`
  is `.doNotDiscard`, and that the group isn't larger than
  `RemoteConfig.current.maxGroupCallRingSize`. Invalid → log + ignore. Valid →
  fall through to the relay. [High]
- **`.answer, .iceUpdate, .hangup, .busy`:** dropped with "the main app should be
  connected" — these mid-call messages are not relayed by the NSE. [High]

### 6.3 `externallyHandleCallMessage(...)` — relay to the main app

[High] (`SignalNSE/NSECallMessageHandler.swift:` `externallyHandleCallMessage`):
1. `CallMessageRelay.enqueueCallMessageForMainApp(...)` persists the envelope +
   plaintext for the main app (`SignalServiceKit/Util/CallMessageRelay.swift:81`).
   [High]
2. **Suspends message processing** in the NSE via
   `messagePipelineSupervisor.suspendMessageProcessing(for: .nseWakingUpApp(...))`
   and schedules the suspension to `invalidate()` after **10 seconds** — so the
   NSE doesn't consume call messages the main app needs. The in-code comment
   states this gives the main app a chance to wake and take over. [High]
3. `CXProvider.reportNewIncomingVoIPPushPayload(payload.payloadDict)` tells
   CallKit/the main app to handle the incoming call; logs success/failure. [High]
4. Wraps everything in `do/catch` with `owsFailDebug` on relay-payload failure.
   [High]

### 6.4 `receivedGroupCallUpdateMessage(...)`

[High] (`SignalNSE/NSECallMessageHandler.swift:` end) Delegates to
`groupCallManager.peekGroupCallAndUpdateThread(forGroupId:peekTrigger:
.receivedGroupUpdateMessage(eraId:messageTimestamp:))`. [High]

---

## 7. `NSELogger`

[High] `SignalNSE/NSELogger.swift:8-18`. A `PrefixedLogger` subclass. The default
`init()` tags each log line with prefix `[NSE]` and a per-instance random UUID
suffix (`{{<uuid>}}`) so interleaved logs from parallel/overlapping NSE
invocations can be correlated to a single `didReceive`. A shared
`static let uncorrelated = NSELogger(prefix: "uncorrelated")` is used where no
per-request logger is available (e.g. memory-pressure handler,
`serviceExtensionTimeWillExpire`). [High]

---

## 8. Error paths, edge cases & validation rules (summary)

| Condition | Behavior | Citation |
| --- | --- | --- |
| Low disk space | Post `nse_lowDiskSpace` once; return empty content (skip fetch) | `NotificationService.swift:98-...` [High] |
| Device locked / keychain not allowed | Post `nse_dbNotAvailable` once; return empty content | `NotificationService.swift` DB catch [High] |
| Other DB load failure | `owsFail` (fatal) | `NotificationService.swift` [High] |
| Not registered (`corruptRegistrationState`) | Log; return empty content | `NotificationService.swift` [High] |
| Verification-code push | Store timestamp (primary only); skip fetch; return empty content | `NotificationService.swift` [High] |
| App build expired | Log; return empty content | `fetchAndProcessMessages` [High] |
| Signal proxy enabled | Wait until proxy ready before fetching; stop after | `startProxyIfEnabled` [High] |
| Fetch cancelled (newer push / time expiry) | Return empty content | `fetchAndProcessMessages` / `serviceExtensionTimeWillExpire` [High] |
| Other fetch error | Logged, non-fatal; still badge/complete | `fetchAndProcessMessages` [High] |
| Clock skewed | Post `nse_clockSkew` once | `fetchAndProcessMessages` [High] |
| Time budget (~30s) exceeded | `serviceExtensionTimeWillExpire` cancels fetch, completes silently with empty content | `NotificationService.swift:182-194` [High] |
| Invalid/old call offer | Insert missed-call interaction instead of ringing | `NSECallMessageHandler.swift` `.offer` [High] |
| Group ring: unknown group / wrong discard mode / too-large group | Discard the ring | `NSECallMessageHandler.swift` `validateGroupRing` [High] |
| Mid-call messages (answer/ice/hangup/busy) | Dropped (main app handles) | `NSECallMessageHandler.swift` [High] |
| Call relay | Suspend NSE processing 10s + report VoIP push to main app | `externallyHandleCallMessage` [High] |

### Feature flags in the NSE

- `DebugFlags.internalLogging` (`build <= .internal`): enables the periodic
  memory-usage log timer (`NotificationService.swift:47`). [High]
- `BuildFlags.improvedNotifications` indirectly affects the badge value the NSE
  writes, via `BadgeCountFetcher` (see [badge-count.md](./badge-count.md)). [High]

### Idempotency / dedup rules

- Per-process `hasShown…` flags prevent duplicate low-disk / DB-unavailable /
  clock-skew notifications; combined with **fixed notification identifiers**,
  repeats coalesce in `UNUserNotificationCenter`. [High]
- `NSEEnvironment.setUpLogging` (`didStartAppSetup`) and `setUpDatabase`
  (`finalContinuation`) are memoized once per process. [High]
- `completeSilently` swaps the content handler to `nil`, guaranteeing a single
  completion. [High]

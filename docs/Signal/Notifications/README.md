# `Signal/Notifications/` — App-Level Notifications

Deep documentation of the **first-party `Signal/Notifications/` directory only** —
the main-app-side glue that registers for push, keeps push tokens synced,
responds when the user taps/acts on a system notification, drives the app-icon
badge number, and schedules a background keepalive fetch.

This set treats `SignalServiceKit/` and `SignalUI/` as **boundaries**: it
documents *how the app target uses them*, not their internals. In particular, the
framework-level notification machinery — the `NotificationPresenter` protocol and
impl, the `UNUserNotificationCenter` wrapper, notification preferences, badge-count
*computation*, and the NSE — lives in a separate target and is documented under
[../../SignalServiceKit/Notifications/](../../SignalServiceKit/Notifications). This
document covers only the six files directly under `Signal/Notifications/`.

For the main-app launch/environment context these types plug into, see
[../README.md](../README.md), [../app-environment.md](../app-environment.md), and
[../launch-sequence.md](../launch-sequence.md).

## Scope

In scope (every file under `Signal/Notifications/`):

- `PushRegistrationManager.swift` — requests the APNs ("vanilla") push token from
  the OS, owns the PushKit VoIP registry used to relay NSE calls, and brokers
  pre-auth challenge tokens.
- `SyncPushTokensJob.swift` — uploads/rotates the APNs token to the chat server.
- `NotificationActionHandler.swift` — executes the user's response to a notification
  (tap, reply, mark-as-read, react, call back, show thread/lobby, re-register, …).
- `BadgeManager.swift` — observes DB changes and fans out a recomputed `BadgeCount`
  to registered observers.
- `AppIconBadgeUpdater.swift` — a `BadgeObserver` that writes the count to the
  app-icon badge.
- `MessageFetchBGRefreshTask.swift` — a `BGAppRefreshTask` keepalive that fetches
  messages periodically to keep registration lock alive.

Out of scope: `SignalServiceKit/Notifications/` (presenter, preferences, badge-count
computation, NSE), the Notification **Settings** UI
(`Signal/src/ViewControllers/AppSettings/Notifications/`), and the registration /
provisioning coordinators that *consume* `PushRegistrationManager` via shims.

## Conventions

- **Citations** are `path:line` (specific declaration) or `path` (file/directory).
  Line numbers reflect the working tree at authoring time and may drift; relocate
  via the symbol name. Every factual claim about behavior cites a path.
- **Confidence labels** on claims about *purpose/behavior*:
  - **[High]** — directly observed (file read / declaration located in this session).
  - **[Medium]** — inferred from signatures, names, and cross-file convention; not
    every referenced collaborator was read in full.
  - **[Low]** — educated inference from naming alone.
- Any uncited claim is a **defect**.
- Where the source gives no evidence of design intent, the text says
  **"intent undetermined — no evidence in source."**

## One-paragraph orientation

These six types are the main app's *edges* of the notification system. On launch,
`AppEnvironment` constructs `PushRegistrationManager`
(`Signal/AppLaunch/AppEnvironment.swift:58`) and, during `setUp`, a `BadgeManager`
plus an `AppIconBadgeUpdater` (`Signal/AppLaunch/AppEnvironment.swift:79-86`), then
starts the badge observers (`Signal/AppLaunch/AppEnvironment.swift:358-359`) **[High]**.
`AppLifecycleManager` registers the background fetch task
(`Signal/AppLaunch/AppLifecycleManager.swift:180`), kicks off `SyncPushTokensJob`
(`.../AppLifecycleManager.swift:922`), and routes incoming notification responses to
`NotificationActionHandler.handleNotificationResponse`
(`.../AppLifecycleManager.swift:2161`) **[High]**. The *building* and *presenting* of
notifications is done by the SSK `NotificationPresenter` (see the SSK doc set); the
files here are what runs **before** a token exists and **after** the user interacts.

## Files and responsibilities

### `PushRegistrationManager`

A singleton that integrates with the OS push services
(`Signal/Notifications/PushRegistrationManager.swift:20`, documented as "Singleton
used to integrate with push notification services" at `:18`) **[High]**. It is a
`PKPushRegistryDelegate` (`:20`) **[High]**.

Responsibilities observed in source:

- **Vanilla (APNs) token.** `requestPushTokens(forceRotation:timeOutEventually:)`
  (`:66`) first registers user-notification settings, then obtains the APNs token via
  `registerForVanillaPushToken` (`:238`), wrapping
  `UIApplication.shared.registerForRemoteNotifications()` in a `Promise` resolved by
  `didReceiveVanillaPushToken` (`:124`) / rejected by `didFailToReceiveVanillaPushToken`
  (`:147`) — both called from the `AppDelegate` **[High]**. On a simulator against
  production it throws `pushNotSupported` (`:74-78`) **[High]**.
- **Force-rotation** unregisters before re-registering (`:264-267`) **[High]**.
- **Timeout / "susceptible to failure" heuristic.** `_registerForVanillaPushToken`
  (`:281`) waits 10s; on timeout it only gives up if
  `isSusceptibleToFailedPushRegistration()` (`:218`) is true — i.e. background refresh
  is `.denied` *and* alert/badge/sound settings are all not `.enabled` — otherwise it
  keeps waiting, because "sometimes registration can just take a while" (`:306-309`)
  **[High]**.
- **If the system volunteers a token** we didn't request, it kicks off a
  `SyncPushTokensJob(mode: .normal)` to sync (`:116-123`) **[High]**.
- **VoIP registry.** `createVoipRegistryIfNecessary` (`:316`) lazily builds a
  `PKPushRegistry` with `desiredPushTypes = [.voIP]` (`:319-321`); the delegate
  callback `pushRegistry(_:didReceiveIncomingPushWith:for:)` (`:159`) synchronously
  waits for app-ready, then relays an `CallMessagePushPayload` from the NSE into the
  call system via `CallMessageRelay.handleVoipPayload` and sets
  `earlyRingNextIncomingCall` (`:174-186`). The comment notes this branch *must* start
  a CallKit call before returning or risk a PushKit penalty (`:170-172`) **[High]**.
  VoIP *token updates* are a no-op — "voip tokens are no longer supported" (`:193`)
  **[High]**. The 5-second guarantee timeout (`:181-185`) and
  `didFinishReportingIncomingCall()` (`:132`) coordinate releasing the callout queue;
  the latter is invoked from `CallKitCallUIAdaptee`
  (`Signal/Calls/UserInterface/CallKitCallUIAdaptee.swift:232`) **[High]**.
- **Pre-auth challenge tokens.** `receivePreAuthChallengeToken()` (`:98`),
  `clearPreAuthChallengeToken()` (`:101`), and `didReceiveVanillaPreAuthChallengeToken`
  (`:115`) expose a `Guarantee<String>` the registration flow awaits
  (`RegistrationCoordinatorImpl.swift:3203`, `:3217`) **[High]**.

Consumers reach this object via `AppEnvironment.shared.pushRegistrationManagerRef`
(`Signal/AppLaunch/AppEnvironment.swift:27`), including the registration/provisioning
coordinators through dedicated shims
(`Signal/Registration/RegistrationCoordinatorDependencies.swift:82`,
`Signal/Provisioning/UserInterface/ProvisioningController.swift:44`) **[High]**.

### `SyncPushTokensJob`

Uploads the APNs token to the chat server, with three modes: `.normal`,
`.forceRotation`, `.rotateIfEligible` (`Signal/Notifications/SyncPushTokensJob.swift:9-13`)
**[High]**. `run()` dispatches by mode; `.rotateIfEligible` consults
`APNSRotationStore.canRotateAPNSToken` and no-ops when not eligible (`:24-37`) **[High]**.
The core `run(shouldRotateAPNSToken:)` (`:41`) asks `PushRegistrationManager` for a
token (`:46`), records the rotation in `APNSRotationStore` (`:48-52`), and only uploads
when the token **changed** or we have not uploaded at least once this process (the
`hasUploadedTokensOnce` `AtomicBool`, `:21`, `:62-67`) **[High]**. The upload itself —
`updatePushTokens` (`:97`) — retries up to 3× with backoff over an authed chat
connection calling `setPushToken(apns:)` (`:98-102`) **[High]**. Tokens are **redacted**
in release logs (prefix/suffix only; full value only in `DEBUG`) via the file-private
`redact(_:)` (`:110-117`) **[High]**.

Entry points: fire-and-forget `class func run(mode:)` (`:71`) used at launch
(`Signal/AppLaunch/AppLifecycleManager.swift:922`, `:933`) and a
`SyncPushTokensJob(mode: .forceRotation)` triggered from the notification settings UI
(`Signal/src/ViewControllers/AppSettings/Notifications/NotificationSettingsViewController.swift:323`)
**[High]**.

### `NotificationActionHandler`

Executes the user's response after they interact with a presented notification. The
single `@MainActor` entry point `handleNotificationResponse(_:appReadiness:screenLockUI:)`
(`Signal/Notifications/NotificationActionHandler.swift:15`) is called from
`AppLifecycleManager` (`Signal/AppLaunch/AppLifecycleManager.swift:2161`), asserts the
app is ready (`:21`), and switches on the `UNNotificationResponse.actionIdentifier`
**[High]**:

- **Default action** (tap): reads `AppNotificationUserInfo.defaultAction` (default
  `.showThread`, `:27`) and dispatches to `showThread`, `showMyStories`, `showMessage`,
  `showCallLobby`, `submitDebugLogs`, `reregister`, `showChatList`, `showLinkedDevices`,
  or `showBackupsSettings` (`:30-57`) **[High]**.
- **Dismiss**: currently a no-op with a `TODO - mark as read?` (`:59-62`) **[High]**.
- **Custom actions** (`AppNotificationAction`, `:65`): `callBack`, `markAsRead`, `reply`,
  `showThread`, `reactWithThumbsUp` (`:71-89`). `callBack` first waits for screen unlock
  via `screenLockUI.waitForScreenUnlockThrowingPrevious()` (`:73`) **[High]**.

Key behaviors worth noting:

- **Reply** (`:122`) builds a `TSOutgoingMessage`, re-attaching a quoted-reply draft and
  honoring the thread's disappearing-message timer — except **group story replies**, which
  keep the story context and no DM timer (`:150-168`); on send failure it surfaces
  `notifyUserOfFailedSend` and rethrows (`:191-195`); on success it marks the message read
  (`:197`) **[High]**.
- **React** (`:270`) applies a 👍 via `ReactionManager.localUserReacted` then marks read
  (`:280-299`) **[High]**.
- **Show thread** distinguishes group-story-reply threads (presents a
  `StoryPageViewController` / `StoryGroupReplier`) from normal threads (scrolls to first
  unread) and skips animation when the app is not active (`:200-213`, `:258-312`) **[High]**.
- **Show call lobby** (`:315`) resolves either a group thread or a call-link target,
  returns to an in-progress matching call, initiates a new video call when none is active,
  or falls back to showing the thread (`:360-385`) **[High]**.
- A private `NotificationMessage` struct + `notificationMessage(forUserInfo:)`
  (`:389-456`) resolve the thread/interaction/story and `hasPendingMessageRequest` from the
  `userInfo` payload; `markMessageAsRead` defers to `receiptManagerRef.markAsReadLocally`
  (`:458-470`) **[High]**.

This type reads `AppEnvironment.shared.callService` (`:12`) and leans on `SignalApp`,
`SSKEnvironment`, and `DependenciesBridge` for navigation, DB writes, and sending **[High]**.

### `BadgeManager` + `AppIconBadgeUpdater`

`BadgeManager` (`Signal/Notifications/BadgeManager.swift:15`) is the app-side observable
cache of the badge count. It does **not** compute the number itself — it wraps a
`FetchBadgeCountBlock`, and the convenience init reads a `BadgeCount` from the SSK
`BadgeCountFetcher` inside a DB read (`:27-39`) **[High]**. It:

- Keeps weak `BadgeObserver`s (`:42`, `BadgeObserver` protocol at `:8`), lazily fetches on
  a serial queue only when there is at least one observer and a fetch is pending
  (`fetchBadgeValueIfNeeded`, `:50`), and caches `mostRecentBadgeCount` to hand to new
  observers immediately (`:85-93`) **[High]**.
- Drops its cached value when the last observer goes away, to avoid serving a stale count
  later (`:66-70`) **[High]**.
- `invalidateBadgeValue()` (`:76`) forces a refresh; it is called directly from several
  settings screens (e.g.
  `NotificationSettingsViewController.swift:284`, `NotificationSettingsBadgeCountViewController.swift:43`)
  **[High]**.
- As a `DatabaseChangeDelegate` (`:99`), it invalidates whenever interactions, threads, or
  `CallRecord` rows change, or on external update/reset (`:100-120`) **[High]**.

`AppIconBadgeUpdater` (`Signal/Notifications/AppIconBadgeUpdater.swift:10`) is a tiny
`BadgeObserver` whose only job is to write `badgeCount.unreadTotalCount` to
`UIApplication.shared.applicationIconBadgeNumber` (`:21-24`) **[High]**. `AppEnvironment`
owns both and starts them together — `badgeManager.startObservingChanges(in:)` +
`appIconBadgeUpdater.startObserving()` (`Signal/AppLaunch/AppEnvironment.swift:358-359`)
**[High]**. Other observers register on the same `BadgeManager`, e.g.
`HomeTabBarController` (`Signal/src/ViewControllers/HomeView/HomeTabBarController.swift:145`,
`:276`) and `CallsListViewController`
(`Signal/Calls/UserInterface/CallsListViewController.swift:61`) **[High]**.

### `MessageFetchBGRefreshTask`

A `BGAppRefreshTask` "keepalive" used specifically while **registration lock** is active,
so the server keeps the account active and reglock alive even if the app/NSE don't launch
(`Signal/Notifications/MessageFetchBGRefreshTask.swift:11-17`) **[High]**. It is a lazily
created singleton gated on app-readiness (`getShared(appReadiness:)`, `:21-39`) **[High]**.

- `register(appReadiness:)` (`:62`) registers the launch handler for the task identifier
  `"MessageFetchBGRefreshTask"` — which the comment notes must stay in sync with Info.plist
  (`:48-50`) — and is called from `AppLifecycleManager`
  (`Signal/AppLaunch/AppLifecycleManager.swift:180`) **[High]**.
- `scheduleTask()` (`:75`) no-ops unless registered, computes `earliestBeginDate` from
  `RemoteConfig.current.backgroundRefreshInterval`, submits a `BGAppRefreshTaskRequest`, and
  logs (without crashing) the specific `BGTaskScheduler.Error` codes
  (`notPermitted`/`tooManyPendingTaskRequests`/`unavailable`) (`:76-108`) **[High]**. It is
  (re)scheduled from `AppLifecycleManager` (`.../AppLifecycleManager.swift:960`) and again at
  the end of each run **[High]**.
- `performTask(_:)` (`:112`) builds a `BackgroundMessageFetcher`, runs fetch +
  processing + side-effects under a 27-second cooperative timeout, stops the fetcher before
  suspending, **reschedules the next run**, and reports success/failure to the OS
  (`:112-140`) **[High]**.

## Interactions summary

| This type | Depends on (SSK/SignalUI/app) | Driven by |
| --- | --- | --- |
| `PushRegistrationManager` | `AppDelegate` push callbacks, `UNUserNotificationCenter`, PushKit, `CallMessageRelay`, `notificationPresenterRef.registerNotificationSettings` | Registration/provisioning coordinators, `SyncPushTokensJob` |
| `SyncPushTokensJob` | `PushRegistrationManager`, `APNSRotationStore`, `preferencesRef`, `chatConnectionManager` | App launch, notification settings UI |
| `NotificationActionHandler` | `SignalApp`, `callService`, `SSKEnvironment`, `DependenciesBridge`, `ScreenLockUI` | `AppLifecycleManager` (OS notification response) |
| `BadgeManager` | SSK `BadgeCountFetcher`, `DatabaseChangeObserver` | `AppEnvironment`, settings screens, home/calls VCs |
| `AppIconBadgeUpdater` | `UIApplication` badge, `BadgeManager` | `AppEnvironment` |
| `MessageFetchBGRefreshTask` | `BGTaskScheduler`, `BackgroundMessageFetcherFactory`, `RemoteConfig`, `TSAccountManager`, `OWS2FAManager` | `AppLifecycleManager` |

All six are confirmed to be wired through `AppEnvironment` / `AppLifecycleManager`
(citations above) **[High]**.

## Notable UI / behavior considerations

- **Animation gated on app state.** Navigation from a notification skips animation when the
  app is not already active, so the destination is visible immediately on cold open
  (`NotificationActionHandler.swift:252-256`, and `UIApplication.shared.applicationState ==
  .active` checks at `:207`, `:238`, `:248`) **[High]**.
- **Screen-lock before sensitive actions.** `callBack` awaits screen unlock before dialing
  (`:73`) **[High]**.
- **Story-aware replies.** Group story replies keep the story context and bypass the DM
  timer; the viewer may be re-presented or popped to reach the right story
  (`:150-168`, `:258-312`) **[High]**.
- **Badge freshness vs. cost.** `BadgeManager` only fetches when observed and caches/clears
  deliberately to avoid stale or wasteful work (`BadgeManager.swift:50-70`) **[High]**.
- **Token privacy.** Push tokens are redacted outside `DEBUG` builds
  (`SyncPushTokensJob.swift:110-117`) **[High]**.
- **PushKit penalty avoidance.** The VoIP delegate must start a CallKit call before
  returning (`PushRegistrationManager.swift:170-172`) **[High]**.

## File inventory

| File | Primary type(s) |
| --- | --- |
| `Signal/Notifications/PushRegistrationManager.swift` | `PushRegistrationManager`, `PushRegistrationError` |
| `Signal/Notifications/SyncPushTokensJob.swift` | `SyncPushTokensJob` |
| `Signal/Notifications/NotificationActionHandler.swift` | `NotificationActionHandler` |
| `Signal/Notifications/BadgeManager.swift` | `BadgeManager`, `BadgeObserver` |
| `Signal/Notifications/AppIconBadgeUpdater.swift` | `AppIconBadgeUpdater` |
| `Signal/Notifications/MessageFetchBGRefreshTask.swift` | `MessageFetchBGRefreshTask` |

Directory/file existence was produced by listing the working tree in this session **[High]**.

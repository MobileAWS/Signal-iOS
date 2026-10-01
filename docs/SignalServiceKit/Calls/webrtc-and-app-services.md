# WebRTC Integration & App-Side Services

This doc covers the main-app orchestration layer (`Signal/Calls/`): the
`CallService` that owns RingRTC's `CallManager`, the shared call-state types,
the pre-flight `CallStarter`, ICE/TURN fetching, CallKit/audio integration, and
the UI layer. It ends with a complete **file inventory** cross-referencing every
first-party file in all three call directories.

## `CallService` — the orchestrator

**File:** `Signal/Calls/CallService.swift`

[High] `final class CallService: CallServiceStateObserver, CallServiceStateDelegate`
(`CallService.swift:18-974`). It is the central hub:

- [High] `typealias CallManagerType = CallManager<SignalCall, CallService>` and it
  owns the `callManager`, constructed with `CallHTTPClient.ringRtcHttpClient` and
  `RingrtcFieldTrials.trials(with: remoteConfig)`
  (`CallService.swift:19-24`, `:110-116`). `CallService` sets itself as the
  `callManager.delegate` (`:142`).
- [High] It owns the per-mode services and helpers: `individualCallService`,
  `groupCallRemoteVideoManager`, `callLinkManager`, `callLinkFetcher`,
  `callLinkStateUpdater`, `callingAssetsFetcher`, `callUIAdapter`,
  `callServiceState`, and a lazily-built `audioService`/`groupCallAccessoryMessageDelegate`/
  `groupCallRecordRingUpdateDelegate` (`CallService.swift:26-205`).
- [High] On app-ready it sets RingRTC's self-UUID
  (`callManager.setSelfUuid(aci)`) and observes registration/reachability/
  background/orientation changes (`CallService.swift:150-205`).
- [Medium] It conforms to `DatabaseChangeDelegate` and defines group-call join
  helpers, a `RingAction`, and `SSKProtoCallMessageOpaqueUrgency` glue later in
  the file (`CallService.swift:976-1757`); these were summarized from the symbol
  map, not line-by-line — **[Medium]**.

### `CallServiceState`

**File:** `Signal/Calls/CallServiceState.swift`

[High] Holds the single `currentCall` as an `AtomicValue<SignalCall?>`; it may be
read off-main but must be **set on the main thread**
(`CallServiceState.swift:19-50`). `setCurrentCall` asserts you never transition
directly from one call to another, and notifies `CallServiceStateObserver`s.
`terminateCall` clears the current call and notifies the delegate
(`CallServiceState.swift:52-85`).

## Shared call-state types

### `SignalCall` / `CallMode`

**File:** `Signal/Calls/SignalCall.swift`

[High] `class SignalCall: CallManagerCallReference` wraps a `CallMode` enum —
`individual(IndividualCall)` / `groupThread(GroupThreadCall)` /
`callLink(CallLinkCall)` (`SignalCall.swift:9-240`). `CallMode` provides unified
accessors (`commonState`, `joinState`, `isFull`, `caller`, `isOutgoingAudioMuted`,
`videoCaptureController`) and a `matches(_ callTarget:)` test. [High] For 1:1
calls it bridges the `CallState` machine into a group-style `JoinState`
(`SignalCall.swift:62-96`).

[High] `enum CallError` lives here: `providerReset`, `disconnected`,
`externalError(underlyingError:)`, `timeout(description:)`, `signaling`,
`doNotDisturbEnabled`, `contactIsBlocked`
(`SignalCall.swift:150-180`). `shouldSilentlyDropCall()` is `true` only for
`doNotDisturbEnabled`/`contactIsBlocked`; `wrapErrorIfNeeded` normalizes
arbitrary errors.

### `CurrentCall`

**File:** `Signal/Calls/CurrentCall.swift`

[High] A value wrapper over the shared `AtomicValue<SignalCall?>` that conforms
to `CurrentCallProvider` so `SignalServiceKit` code (e.g. `GroupCallManager`)
can ask "is there a current call? for which group?" without importing app types
(`CurrentCall.swift:8-30`).

### `CommonCallState`

**File:** `Signal/Calls/CommonCallState.swift`

[High] State common to every call mode (`CommonCallState.swift:9-90`): an
`AudioActivity`, a `localId: UUID` (used for CallKit), the `connectedDate`
(`MonotonicDate`, set once), `connectionDuration()`, and a CallKit `SystemState`
machine (`notReported`→`pending`→`reported`→`removed`) with a `deinit` assertion
that a reported call was removed.

### `CallTarget`

**File:** `Signal/Calls/CallTarget.swift`

[High] `enum CallTarget` = `individual(TSContactThread)` /
`groupThread(GroupIdentifier)` / `callLink(CallLink)`, plus `canCall`
computed properties: a contact thread can call unless it's Note-to-Self; a group
can call only if it's a V2 group where the local user is a full, non-terminated
member (`CallTarget.swift:9-30`).

## Starting a call — `CallStarter`

**File:** `Signal/Calls/CallStarter.swift`

[High] `struct CallStarter` runs the pre-flight checks before a call
(`CallStarter.swift:11-250`). `startCall(from:)` (`:70-150`):

1. If the thread is **blocked**, show the unblock sheet → `.promptedToUnblock`.
2. If a call with this target is already ongoing, **return to it** →
   `.callStarted`.
3. Reject **announcement-only** groups (non-admins can't start calls).
4. Enforce `canCall` (Note-to-Self / V2-full-member), then whitelist the thread.
5. `context.callService.initiateCall(to:isVideo:)` → `.callStarted`.

[High] `StartCallResult` = `callStarted` / `callNotStarted` / `promptedToUnblock`.
[High] `prepareToStartCall(from:shouldAskForCameraPermission:)` checks
registration and microphone/camera permissions, throwing `PrepareToStartCallError`
(`.notRegistered`/`.missingMicrophonePermission`/`.missingCameraPermission`),
with `showPrepareToStartCallError` presenting the matching UI
(`CallStarter.swift:195-250`).

## WebRTC / network integration points

### `RTCIceServerFetcher`

**File:** `Signal/Calls/RTCIceServerFetcher.swift`

[High] `getIceServers()` fetches TURN/STUN relays via
`OWSRequestFactory.callingRelaysRequest()`, parses the `CallingRelays` JSON into
`WebRTC.RTCIceServer`s, and caches them with a server-provided TTL guarded by an
`NSRecursiveLock` (`RTCIceServerFetcher.swift:9-160`). [High] Parsing orders
servers by response order, preferring URLs with embedded IPs; the minimum valid
TTL across relays governs cache lifetime (0 disables caching).

### `CallHTTPClient`

See [group-calls.md § CallHTTPClient](./group-calls.md#callhttpclient--ringrtcs-http-transport)
— the RingRTC `HTTPDelegate` used by `CallManager`, `SFUClient`, and the peek
client.

### CallKit & audio

[Medium] These bridge RingRTC/`CallService` to the OS. Behavior summarized from
file names/structure, not read line-by-line — **[Medium]**:

- `Signal/Calls/CallKitCallManager.swift`, `UserInterface/CallKitCallUIAdaptee.swift`,
  `Signal/Calls/CallKitIdStore.swift` — map calls to CallKit `CXCall`s and track
  CallKit UUIDs.
- `Signal/Calls/CallAudioService.swift`, `AudioSession+WebRTC.swift`,
  `AudioSource.swift` — manage the `AVAudioSession` and route/ringtone/sound
  effects for calls (observes `CallServiceState`).
- `UserInterface/CallUIAdapter.swift` / `SimulatorCallUIAdaptee.swift` — the
  adapter layer that `CallService`/`IndividualCallService` call into to present
  and update the system/in-app call UI.

### Group-call accessory managers

[Medium] (summarized from structure — **[Medium]**):

- `Signal/Calls/GroupCallRemoteVideoManager.swift` — manages remote video track
  subscriptions for group calls (a `CallServiceState` observer).
- `Signal/Calls/GroupCallAccessoryMessageDelegate.swift` — sends
  `OutgoingGroupCallUpdateMessage`s and records join/leave accessory events.
- `Signal/Calls/GroupCallRecordRingingCleanupManager.swift` — cleans up stale
  ringing `CallRecord`s on launch/termination.
- `Signal/Calls/CallQualitySurvey.swift` — the `CallQualitySurveyManager` shown
  by `GroupCall.onEnded`.
- `Signal/Calls/CallRecordLoader.swift` — loads/pages `CallRecord`s for the
  Calls Tab via `CallRecordQuerier`.
- `Signal/Calls/CallingAssetsFetcher.swift` — periodically downloads calling
  assets (e.g. ringtones/sounds), scheduled by `CallService`'s cron job.
- `Signal/Calls/CallStrings.swift` — localized strings for calling UI (e.g.
  `CallStrings.signalCall`).

## UI layer (`Signal/Calls/UserInterface/`)

[Low–Medium] The `UserInterface/` directory holds the SwiftUI/UIKit presentation
for calls. These were not read line-by-line; purposes are inferred from names and
are **[Low]** unless otherwise noted. They are enumerated in the inventory below.
Highlights:

- `IndividualCallViewController` / `GroupCallViewController` / `NewCallViewController`
  — the full-screen call screens.
- `CallsListViewController` (+`+ViewModelLoader`, `+Strings`) — the Calls Tab.
- `CallControls*`, `CallButton`, `CallHeader`, `CallDrawerSheet*`,
  `IncomingCallControls`, `ReturnToCallViewController`, `FlipCameraTooltip` —
  in-call controls & chrome.
- `CallMember*`, `GroupCallVideo*`, `LocalVideoView`, `RemoteVideoView` — video
  rendering & layout.
- `RaisedHandsToast`, `Reactions*`, `IncomingReactionsView`, `RemoteMuteToast`,
  `GroupCallNotificationView`, `GroupCallSwipeToastView`, `GroupCallErrorView`,
  `CallMemberWaitingAndErrorView`, `CallControlsConfirmationToast`,
  `SupplementalCallControlsForFullscreenLocalMember` — ephemeral in-call UI.
- Call-link UI: `CallLinkViewController`, `CreateCallLinkViewController`,
  `EditCallLinkNameViewController`, `CallLinkApproval*`, `CallLinkBulkApprovalSheet`,
  `CallLinkDeleter`, `CallLinkProfileKeySharingManager`.
- `Survey/` — the post-call quality survey flow.

---

## Appendix: first-party file inventory

Every first-party file in the three call directories, with the doc that covers
it. "✓ (listed)" means the file is referenced here primarily for completeness.

### `SignalServiceKit/Calls/`

| File | Covered in |
| --- | --- |
| `RingrtcVp9Config.swift` | [ringrtc-configuration.md](./ringrtc-configuration.md) |
| `RingrtcSvcConfig.swift` | [ringrtc-configuration.md](./ringrtc-configuration.md) |
| `RingrtcFieldTrials.swift` | [ringrtc-configuration.md](./ringrtc-configuration.md) |
| `CallMessageHandler.swift` | [call-message-handling.md](./call-message-handling.md) |
| `NoopCallMessageHandler.swift` | [call-message-handling.md](./call-message-handling.md) |
| `OutgoingCallMessage.swift` | [call-message-handling.md](./call-message-handling.md) |
| `GroupCallManager.swift` | [group-calls.md](./group-calls.md) |
| `GroupCallPeekClient.swift` | [group-calls.md](./group-calls.md) |
| `CallHTTPClient.swift` | [group-calls.md](./group-calls.md) |
| `CancelledGroupRing.swift` | [group-calls.md](./group-calls.md) |
| `CallLinkState.swift` | [call-links.md](./call-links.md) |
| `CallLinkRecord.swift` | [call-links.md](./call-links.md) |
| `CallLinkRecordStore.swift` | [call-links.md](./call-links.md) |
| `OutgoingCallLinkUpdateMessage.swift` | [call-links.md](./call-links.md) |
| `OutgoingCallEventSyncMessage.swift` | [call-records-and-sync.md](./call-records-and-sync.md) |
| `OutgoingCallLogEventSyncMessage.swift` | [call-records-and-sync.md](./call-records-and-sync.md) |
| `Individual/TSCall.h`, `Individual/TSCall.m`, `Individual/TSCall.swift`, `Individual/TSCall+SDS.swift` | [individual-calls.md](./individual-calls.md) |
| `Individual/CallOfferHandler.swift` | [individual-calls.md](./individual-calls.md) |
| `Individual/CallEventInserter.swift` | [individual-calls.md](./individual-calls.md) |
| `Group/OWSGroupCallMessage.h`, `Group/OWSGroupCallMessage.m`, `Group/OWSGroupCallMessage.swift`, `Group/OWSGroupCallMessage+SDS.swift` | [group-calls.md](./group-calls.md) |
| `Group/GroupCallInteractionFinder.swift` | [group-calls.md](./group-calls.md) |
| `Group/OutgoingGroupCallUpdateMessage.swift` | [group-calls.md](./group-calls.md) |

### `SignalServiceKit/Calls/CallRecord/`

| File | Covered in |
| --- | --- |
| `CallRecord.swift`, `CallRecord+CallStatus.swift`, `CallRecord+Sorting.swift` | [call-records-and-sync.md](./call-records-and-sync.md) |
| `CallRecordStore.swift`, `CallRecordStoreNotification.swift` | [call-records-and-sync.md](./call-records-and-sync.md) |
| `CallRecordQuerier.swift`, `CallRecordCursor.swift` | [call-records-and-sync.md](./call-records-and-sync.md) |
| `CallRecordAssociatedInteraction.swift`, `CallRecordLogger.swift` | [call-records-and-sync.md](./call-records-and-sync.md) |
| `InteractionStore+CallRecord.swift` | [call-records-and-sync.md](./call-records-and-sync.md) |
| `IndividualCallRecordManager.swift`, `GroupCallRecordManager.swift` | [call-records-and-sync.md](./call-records-and-sync.md) |
| `AdHocCallRecordManager.swift` | [call-links.md](./call-links.md) + [call-records-and-sync.md](./call-records-and-sync.md) |
| `CallRecordDeleteManager.swift`, `CallRecordMissedCallManager.swift` | [call-records-and-sync.md](./call-records-and-sync.md) |
| `GroupCallRecordRingUpdateDelegate.swift` | [call-records-and-sync.md](./call-records-and-sync.md) |
| `CallEventConversation.swift`, `CallRecordSyncMessageConversationIdAdapter.swift` | [call-records-and-sync.md](./call-records-and-sync.md) |
| `IncomingCallEventSyncMessageManager.swift`, `IncomingCallEventSyncMessageParams.swift` | [call-records-and-sync.md](./call-records-and-sync.md) |
| `OutgoingCallEventSyncMessageManager.swift` | [call-records-and-sync.md](./call-records-and-sync.md) |
| `IncomingCallLogEventSyncMessageManager.swift`, `IncomingCallLogEventSyncMessageParams.swift` | [call-records-and-sync.md](./call-records-and-sync.md) |

### `SignalServiceKit/Calls/DeletedCallRecord/`

| File | Covered in |
| --- | --- |
| `DeletedCallRecord.swift` | [call-records-and-sync.md](./call-records-and-sync.md) |
| `DeletedCallRecordStore.swift` | [call-records-and-sync.md](./call-records-and-sync.md) |
| `DeletedCallRecordExpirationJob.swift` | [call-records-and-sync.md](./call-records-and-sync.md) |

### `SignalUI/Calls/`

| File | Covered in |
| --- | --- |
| `CallLink.swift` | [call-links.md](./call-links.md) |
| `CallLinkFetcher.swift` | [call-links.md](./call-links.md) |
| `CallLinkTest.swift` | [call-links.md](./call-links.md) |

### `Signal/Calls/` (non-UI)

| File | Covered in |
| --- | --- |
| `CallService.swift`, `CallServiceState.swift` | this doc |
| `SignalCall.swift` (incl. `CallMode`/`CallError`), `CurrentCall.swift`, `CommonCallState.swift`, `CallTarget.swift` | this doc |
| `CallStarter.swift` | this doc |
| `RTCIceServerFetcher.swift` | this doc |
| `IndividualCall.swift`, `IndividualCallService.swift` | [individual-calls.md](./individual-calls.md) |
| `GroupCall.swift`, `GroupThreadCall.swift`, `CallLinkCall.swift` | [group-calls.md](./group-calls.md) / [call-links.md](./call-links.md) |
| `WebRTCCallMessageHandler.swift` | [call-message-handling.md](./call-message-handling.md) |
| `CallLinkManager.swift`, `CallLinkStateUpdater.swift`, `CallLinkAdminManager.swift`, `CallLinkFetchJobRunner.swift`, `CallLinkUpdateMessageSender.swift` | [call-links.md](./call-links.md) |
| `AdHocCallStateObserver.swift` | [call-links.md](./call-links.md) |
| `CallAudioService.swift`, `AudioSession+WebRTC.swift`, `AudioSource.swift` | this doc (CallKit & audio) |
| `CallKitCallManager.swift`, `CallKitIdStore.swift` | this doc (CallKit & audio) |
| `GroupCallRemoteVideoManager.swift`, `GroupCallAccessoryMessageDelegate.swift`, `GroupCallRecordRingingCleanupManager.swift` | this doc (accessory managers) |
| `CallQualitySurvey.swift`, `CallRecordLoader.swift`, `CallingAssetsFetcher.swift`, `CallStrings.swift` | this doc (accessory managers) |

### `Signal/Calls/UserInterface/` (+ `Survey/`)

[Low] All UI files below are enumerated for completeness; see the "UI layer"
section above for grouped descriptions.

| File |
| --- |
| `CallButton.swift`, `CallControls.swift`, `CallControlsConfirmationToast.swift`, `CallControlsOverflowView.swift` |
| `CallDrawerSheet.swift`, `CallDrawerSheetDataSource.swift`, `CallHeader.swift` |
| `CallKitCallUIAdaptee.swift`, `CallUIAdapter.swift`, `SimulatorCallUIAdaptee.swift` |
| `CallLinkApprovalRequestDetailsSheet.swift`, `CallLinkApprovalRequestView.swift`, `CallLinkApprovalViewModel.swift`, `CallLinkBulkApprovalSheet.swift` |
| `CallLinkDeleter.swift`, `CallLinkProfileKeySharingManager.swift`, `CallLinkViewController.swift` |
| `CallMemberCameraOffView.swift`, `CallMemberChromeOverlayView.swift`, `CallMemberVideoView.swift`, `CallMemberView.swift`, `CallMemberWaitingAndErrorView.swift` |
| `CallsListViewController.swift`, `CallsListViewController+ViewModelLoader.swift`, `CallsListViewController+Strings.swift` |
| `CreateCallLinkViewController.swift`, `EditCallLinkNameViewController.swift` |
| `FlipCameraTooltip.swift`, `GroupCallErrorView.swift`, `GroupCallNotificationView.swift`, `GroupCallSwipeToastView.swift` |
| `GroupCallVideoContextMenuConfiguration.swift`, `GroupCallVideoGrid.swift`, `GroupCallVideoGridLayout.swift`, `GroupCallVideoOverflow.swift`, `GroupCallViewController.swift` |
| `IncomingCallControls.swift`, `IncomingReactionsView.swift`, `IndividualCallViewController.swift` |
| `LocalVideoView.swift`, `NewCallViewController.swift`, `RaisedHandsToast.swift` |
| `ReactionsBurstView.swift`, `ReactionsSink.swift`, `RemoteMuteToast.swift`, `RemoteVideoView.swift` |
| `ReturnToCallViewController.swift`, `SupplementalCallControlsForFullscreenLocalMember.swift` |
| `Survey/CallQualitySurveyCustomIssueViewController.swift`, `Survey/CallQualitySurveyDebugLogViewController.swift`, `Survey/CallQualitySurveyIssuesViewController.swift`, `Survey/CallQualitySurveyNavigation.swift`, `Survey/CallQualitySurveyRatingViewController.swift` |

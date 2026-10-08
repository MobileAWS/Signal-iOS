# Signal — `util/` module

This documentation set covers the grab-bag utility module in the **main app
target** at [`Signal/util/`](../../../Signal/util/) (note the lowercase `util`
directory name). It is a small, flat collection of 14 Swift source files (plus a
non-Swift `Launch Screen.storyboard`) that do **not** share a single
responsibility. Instead, each file provides a focused, app-layer helper that
either (a) implements a `SignalServiceKit` protocol with UIKit-dependent
behavior that cannot live in the framework, or (b) wraps a tricky
platform/private API behind a safe, testable Swift surface.

Everything here lives in the `Signal` app target rather than in
`SignalServiceKit`/`SignalUI` because each file depends on something
target-specific: `UIApplication`/`UIDevice` singletons, obfuscated private
Apple selectors, or app-only types such as mock messages. Where a file is the
concrete half of a protocol-based seam, the protocol itself is cross-referenced
to `SignalServiceKit/Util/`.

> **Confidence labels.** Each claim is tagged with a confidence level:
> - **[High]** — directly read from source in `Signal/util/`; behavior is
>   explicit in the code.
> - **[Medium]** — inferred from code with reasonable certainty, but depends on
>   collaborators defined outside `Signal/util/` (e.g. `ScreenLock`,
>   `ChatConnectionManager`, `DarwinNotificationCenter`).
> - **[Low]** — inferred from naming/comments or private-API usage that cannot
>   be fully verified in-tree.
>
> Where the source gives no evidence for a design question, the text says
> **"intent undetermined — no evidence in source"** rather than guessing.
>
> **Citations** are given as `path:line` relative to the repository root,
> pointing at the definition being described. Line numbers reflect the state of
> the tree at authoring time and may drift as code changes; use the cited symbol
> name to re-locate code if lines have moved.

## File index

| File | Primary symbol(s) | One-line summary |
| --- | --- | --- |
| `VolumeButtons.swift` | `PassiveVolumeButtonObservation`, `AVVolumeButtonObservation`, `VolumeButtons`, `LegacyGlobalVolumeButtonObserver` | Observe hardware volume-button presses (camera shutter / passive), with an iOS 17.2 `AVCaptureEventInteraction` path and a legacy private-API path |
| `ScreenLockUI.swift` | `ScreenLockUI` | Drives the screen-blocking window + biometric unlock flow and App-Switcher content protection |
| `PaymentDetailsValidity.swift` | `PaymentMethodFieldValidity`, `CreditAndDebitCards`, `SEPABankAccounts` | Field-level validity logic for card numbers (Luhn), expiry, CVV, and SEPA IBANs |
| `DisplayableText.swift` | `DisplayableText` | Renderable text model: truncation, jumbomoji counting, link-ification gating |
| `BatchUpdate.swift` | `BatchUpdate`, `BatchUpdateType`, `BatchUpdateValue` | Diffs old/new value lists into a safe `UITableView`/`UICollectionView` batch-update item list |
| `TextHelper.swift` | `TextFieldHelper`, `TextViewHelper`, `TextHelper` | Length-limited (byte/scalar/glyph) `shouldChange…` delegate helpers that respect Unicode boundaries |
| `RingerSwitch.swift` | `RingerSwitch`, `RingerSwitchObserver` | Observes the mute/ringer switch via an obfuscated Darwin notification |
| `ProxyConnectionChecker.swift` | `ProxyConnectionChecker` | Awaits an unidentified chat connection (with timeout) to confirm a proxy works |
| `DeviceSleepManagerImpl.swift` | `DeviceSleepManagerImpl` | Block-object based idle-timer (screen-sleep) suppression |
| `DeviceBatteryLevelManagerImpl.swift` | `DeviceBatteryLevelManagerImpl`, `DeviceBatteryLevelMonitorImpl` | Reference-counted battery-level monitoring + low-power-mode queries |
| `ASWebAuthenticationSession+Util.swift` | `ASWebAuthenticationSession.resultify` | Collapses the two-argument auth-session callback into one `Result<URL, Error>` |
| `TSMessage+RenderableContent.swift` | `TSMessage.hasRenderableContent(tx:)` | App-target convenience to test whether a `TSMessage` has renderable content |
| `Progress+Signal.swift` | `Progress.remainingUnitCount` | Trivial `total − completed` convenience on `Foundation.Progress` |
| `SDAnimatedImage+Duration.swift` | `SDAnimatedImage.animationDuration` | Sums per-frame durations to work around `SDAnimatedImageView` returning 0 |

## Thematic groupings

Although the directory is flat, the files fall into a few natural clusters.

### 1. Protocol implementations backing `SignalServiceKit` seams

Three files are the UIKit-bound concrete implementations of protocols that are
declared in `SignalServiceKit/Util/`. The framework declares the abstraction;
the app target supplies the `UIApplication`/`UIDevice`-dependent body.

- `DeviceSleepManagerImpl` conforms to `DeviceSleepManager` and prevents the
  screen from sleeping while any registered block object is alive. **[High]**
  (`Signal/util/DeviceSleepManagerImpl.swift:17`). It keeps a `@MainActor`
  array of `Weak<DeviceSleepBlockObject>` and, on every add/remove, culls dead
  entries and sets `UIApplication.shared.isIdleTimerDisabled` to
  `!blockObjects.isEmpty`. **[High]**
  (`Signal/util/DeviceSleepManagerImpl.swift:38-56`). The protocol it satisfies
  lives at `SignalServiceKit/Util/DeviceSleepManager.swift:16`. **[Medium]**
- `DeviceBatteryLevelManagerImpl` conforms to `DeviceBatteryLevelManager`
  (`SignalServiceKit/Util/DeviceBatteryLevelManager.swift:21` **[Medium]**) and
  reference-counts monitoring by a set of `reason` strings: it enables
  `UIDevice.current.isBatteryMonitoringEnabled` only while at least one reason
  is active. **[High]**
  (`Signal/util/DeviceBatteryLevelManagerImpl.swift:36-69`). `batteryLevel`
  returns a hardcoded `1` on the simulator because the simulator reports an
  invalid `-1.0`. **[High]**
  (`Signal/util/DeviceBatteryLevelManagerImpl.swift:24-31`). It also surfaces
  `isLowPowerModeEnabled` and the `NSProcessInfoPowerStateDidChange`
  notification name. **[High]**
  (`Signal/util/DeviceBatteryLevelManagerImpl.swift:70-72`).

See [SignalServiceKit `Util/`](../../SignalServiceKit/) for the protocol side
of these seams.

### 2. Hardware / system-event observation (private-API heavy)

- `RingerSwitch` is a singleton (`RingerSwitch.shared`,
  `Signal/util/RingerSwitch.swift:13` **[High]**) that reports the silent-switch
  state to `RingerSwitchObserver`s. It observes an **obfuscated** Darwin
  notification whose name is reconstructed via `decodedForSelector` from a
  base64-ish blob (the comment reveals it decodes
  `com.apple.springboard.ringerstate`). **[High]**
  (`Signal/util/RingerSwitch.swift:63-64`). State is read through
  `DarwinNotificationCenter.getState(observer:)`, treating `> 0` as "not
  silenced". **[High]** (`Signal/util/RingerSwitch.swift:80-101`). The private
  nature of the notification name is the reason this lives in the app target
  and is tagged **[Low]** for correctness guarantees across OS versions.
- `VolumeButtons.swift` provides two observation front-ends over a shared
  `VolumeButtons` namespace enum (`Signal/util/VolumeButtons.swift:85`
  **[High]**):
  - `PassiveVolumeButtonObservation` reports only "some volume button was
    tapped" without interfering with the system volume change, keying off a
    decoded `SystemVolumeDidChange` notification and filtering for
    `userInfo["Reason"] == "ExplicitVolumeChange"`. **[High]**
    (`Signal/util/VolumeButtons.swift:18`,
    `Signal/util/VolumeButtons.swift:40`,
    `Signal/util/VolumeButtons.swift:73-76`).
  - `AVVolumeButtonObservation` reports discrete press/release/tap/long-press
    events (used for a camera-style shutter). On iOS 17.2+ it uses the public
    `AVCaptureEventInteraction` attached to a `CapturePreviewView`; on older OSes
    it falls back to `LegacyGlobalVolumeButtonObserver`. **[High]**
    (`Signal/util/VolumeButtons.swift:106`,
    `Signal/util/VolumeButtons.swift:157-192`). Long-press detection is a
    `WeakTimer` gated by `VolumeButtons.longPressDuration` (0.5 s). **[High]**
    (`Signal/util/VolumeButtons.swift:91`,
    `Signal/util/VolumeButtons.swift:214-247`).
  - The legacy path (`LegacyGlobalVolumeButtonObserver`,
    `Signal/util/VolumeButtons.swift:292` **[High]**) is `init?`-nil on iOS
    17.2+ and uses private selectors (`setWantsVolumeButtonEvents:`) and
    obfuscated `_UIApplicationVolume…` notification names, plus an off-screen
    `MPVolumeView` to nudge the system volume. **[Low]** (private-API usage)
    (`Signal/util/VolumeButtons.swift:293-299`,
    `Signal/util/VolumeButtons.swift:421-470`).

### 3. Screen lock / content protection UI

`ScreenLockUI` owns a dedicated `screenBlockingWindow` (an `OWSWindow` at
window-level `._background`) whose root is a `ScreenLockViewController`, and it
drives a state machine over app foreground/background/active notifications.
**[High]** (`Signal/util/ScreenLockUI.swift:9`,
`Signal/util/ScreenLockUI.swift:106-221`). Key behaviors:

- A monotonic-clock **countdown** (`clock_gettime_nsec_np(CLOCK_MONOTONIC_RAW)`)
  starts when the app leaves the foreground; once it exceeds
  `ScreenLock.shared.screenLockTimeout()`, the app is marked locked. **[Medium]**
  (`Signal/util/ScreenLockUI.swift:103`,
  `Signal/util/ScreenLockUI.swift:372-420`). Depends on `ScreenLock`
  (`SignalServiceKit/Util/ScreenLock.swift:9`).
- `waitForScreenUnlockThrowingPrevious()` suspends via a
  `CheckedContinuation` until the screen is unlocked, resuming any prior waiter
  with a `ScreenUnlockActionReplacedError`. **[High]**
  (`Signal/util/ScreenLockUI.swift:84-100`).
- `desiredUIState()` chooses between `.none`, `.screenLock`, and
  `.screenProtection`; screen protection is shown when
  `preferencesRef.isScreenSecurityEnabled` is on **or** any registered
  "sensitive content" view controller is on screen, so sensitive screens are
  obscured in the App Switcher. **[High]**
  (`Signal/util/ScreenLockUI.swift:184-189`,
  `Signal/util/ScreenLockUI.swift:259-285`).
- Biometric unlock is delegated to `ScreenLock.shared.tryToUnlockScreenLock(…)`;
  the UI itself (`ScreenLockViewController`) is defined in `SignalUI`
  (`SignalUI/ViewControllers/ScreenLockViewController.swift:14`). **[Medium]**
  (`Signal/util/ScreenLockUI.swift:288-351`).

### 4. Text handling & display

- `TextHelper.shouldChangeCharactersInRange(…)` is the Unicode-aware core used by
  the thin `TextFieldHelper`/`TextViewHelper` wrappers. It enforces any
  combination of `maxByteCount` / `maxUnicodeScalarCount` / `maxGlyphCount`,
  measuring **both** the filtered-for-display and unfiltered strings so that
  (a) trailing whitespace can't be padded past the limit and (b) display
  filtering that *grows* the string (e.g. Bidi characters) can't overflow.
  **[High]** (`Signal/util/TextHelper.swift:89`,
  `Signal/util/TextHelper.swift:142-205`). On paste it greedily keeps the longest
  acceptable prefix instead of rejecting the whole insertion. **[High]**
  (`Signal/util/TextHelper.swift:170-203`).
- `DisplayableText` is a thread-safe (`AtomicValue`/`AtomicOptional`) model
  holding full and optional truncated `CVTextValue` content plus natural
  alignment. **[High]** (`Signal/util/DisplayableText.swift:9-26`). It computes a
  jumbomoji count (0 unless the string is emoji-only and `<= kMaxJumbomojiCount`
  == 5) and a lazy `shouldAllowLinkification` that runs an `NSDataDetector`
  and per-match host validation via `LinkValidator`, rejecting mixed
  ASCII/non-ASCII hosts. **[High]**
  (`Signal/util/DisplayableText.swift:70`,
  `Signal/util/DisplayableText.swift:117-200`). The `displayableText(withMessageBody:transaction:)`
  factory hydrates mentions (`ContactsMentionHydrator`) and truncates to
  `kMaxTextDisplayLength` (512) glyphs / `kMaxSnippetNewLines` (15) lines,
  appending `"…"`. **[High]**
  (`Signal/util/DisplayableText.swift:233-307`). Depends on `SignalUI`
  (`CVTextValue`, `MessageBody`) — **[Medium]** for those collaborators.

### 5. Payment field validation

`PaymentDetailsValidity.swift` is a pure-logic file (no UI) built around the
generic `PaymentMethodFieldValidity<Invalidity>` enum with
`potentiallyValid` / `fullyValid` / `invalid(Invalidity)` cases. **[High]**
(`Signal/util/PaymentDetailsValidity.swift:11-26`). It is `Equatable` only under
`#if TESTABLE_BUILD`. **[High]**
(`Signal/util/PaymentDetailsValidity.swift:42-57`).

- `CreditAndDebitCards` determines card type by prefix (Amex `34`/`37`,
  UnionPay `62`/`81`), gates CVV length (4 for Amex, else 3), and validates card
  numbers with a focus-aware Luhn check — UnionPay is accepted without Luhn.
  **[High]** (`Signal/util/PaymentDetailsValidity.swift:59`,
  `Signal/util/PaymentDetailsValidity.swift:85-150`,
  `Signal/util/PaymentDetailsValidity.swift:266-285`).
- `SEPABankAccounts` validates IBANs: it looks up the expected length per
  ISO country code in `expectedIBANLengthByCountryCode` and runs the mod-97
  check (`doesIBANPassValidationCheck`) piecewise to avoid overflowing a
  `UInt64`. **[High]**
  (`Signal/util/PaymentDetailsValidity.swift:289`,
  `Signal/util/PaymentDetailsValidity.swift:371-393`,
  `Signal/util/PaymentDetailsValidity.swift:403`).

### 6. Collection/table-view diffing

`BatchUpdate<T: BatchUpdateValue>` turns old/new/changed value lists into an
ordered list of `.delete` / `.insert` / `.move` / `.update` `Item`s that can be
safely applied inside `performBatchUpdates`. **[High]**
(`Signal/util/BatchUpdate.swift:8-19`,
`Signal/util/BatchUpdate.swift:81`,
`Signal/util/BatchUpdate.swift:172`). The algorithm:

1. Builds value maps (throwing on duplicate ids), then computes delete/insert
   sets and *simulates* each stage to assert the transformed id list matches the
   target before returning. **[High]**
   (`Signal/util/BatchUpdate.swift:181-230`).
2. Chooses `.move` vs `.delete`+`.insert` based on `ViewType`: `UITableView`
   moves update the cell, `UICollectionView` moves do **not**, so collection-view
   moves of changed items are expressed as delete+insert. `canUseMoveInCollectionView`
   is hardcoded `false`. **[High]**
   (`Signal/util/BatchUpdate.swift:342-376`,
   `Signal/util/BatchUpdate.swift:542`).
3. Finds the minimal move set greedily via `findValueIdsToMove` /
   `findValueIdsToMoveStep`, repeatedly extracting the "wanderer" with the
   greatest index distance until the orders match. **[High]**
   (`Signal/util/BatchUpdate.swift:473-537`).

`BatchUpdate` cannot be instantiated — its only `init` is marked `@available(*,
unavailable)`; it is used purely through the static `build(…)`. **[High]**
(`Signal/util/BatchUpdate.swift:83`).

### 7. Small extensions / adapters

- `ProxyConnectionChecker.checkConnection()` confirms a proxy by awaiting
  `chatConnectionManager.waitForUnidentifiedConnectionToOpen()` wrapped in
  `withCooperativeTimeout(seconds: OWSRequestFactory.textSecureHTTPTimeOut)`,
  returning `true`/`false` instead of throwing. **[High]**
  (`Signal/util/ProxyConnectionChecker.swift:8-23`). Depends on
  `ChatConnectionManager` — see
  [SignalServiceKit `Network/`](../../SignalServiceKit/Network/) for that
  transport. **[Medium]**
- `ASWebAuthenticationSession.resultify(callbackUrl:error:)` normalizes the
  two-argument completion into one `Result<URL, Error>`, asserting the invariant
  that exactly one argument is non-nil (and `owsFail`-ing if neither is).
  **[High]** (`Signal/util/ASWebAuthenticationSession+Util.swift:9-28`).
- `TSMessage.hasRenderableContent(tx:)` is deliberately app-target-only: for an
  **uninserted** message it asserts the message is a mock
  (`MockIncomingMessage`/`MockOutgoingMessage`) and reconstructs renderability
  from the builder; otherwise it defers to `insertedMessageHasRenderableContent`.
  **[High]** (`Signal/util/TSMessage+RenderableContent.swift:12-33`). Depends on
  `SignalServiceKit` (`TSMessage`, `TSMessageBuilder`). **[Medium]**
- `Progress.remainingUnitCount` is a one-liner (`totalUnitCount −
  completedUnitCount`). **[High]** (`Signal/util/Progress+Signal.swift:8-9`).
- `SDAnimatedImage.animationDuration` sums `animatedImageDuration(at:)` across
  all frames, working around `SDAnimatedImageView`'s occasional `0` duration;
  returns `nil` when there are no frames. **[High]**
  (`Signal/util/SDAnimatedImage+Duration.swift:11-19`). Depends on the
  third-party `SDWebImage` module. **[Low]**

## Cross-references

- **`SignalServiceKit/Util/`** — declares the protocols implemented here
  (`DeviceSleepManager.swift:16`, `DeviceBatteryLevelManager.swift:21`) and the
  `ScreenLock` engine (`ScreenLock.swift:9`) that `ScreenLockUI` drives.
  **[Medium]**
- **[`SignalServiceKit/Network/`](../../SignalServiceKit/Network/)** — defines
  `ChatConnectionManager`, used by `ProxyConnectionChecker` to probe the
  unidentified connection. **[Medium]**
- **`SignalUI`** — supplies `CVTextValue`, `MessageBody`,
  `TextViewWithPlaceholder`, and `ScreenLockViewController`
  (`SignalUI/ViewControllers/ScreenLockViewController.swift:14`) consumed by
  `DisplayableText`, `TextHelper`, and `ScreenLockUI`. **[Medium]**
- **[`SignalServiceKit/Jobs/`](../../SignalServiceKit/Jobs/)** — unrelated
  durable-queue subsystem; listed only to contrast scope: `Signal/util/` holds
  stateless/UI helpers, not persisted background work. **[High]**

## What this module is *not*

- It is **not** a single cohesive subsystem; the only thing the files share is
  being app-target utilities that don't fit cleanly in the frameworks. **[High]**
  (survey of all 14 files above).
- Several files rely on **private Apple APIs** reconstructed from obfuscated
  strings (`RingerSwitch.swift:63-64`, `VolumeButtons.swift:421-470`). Their
  long-term correctness across OS releases is **intent undetermined — no
  evidence in source** beyond the inline comments. **[Low]**

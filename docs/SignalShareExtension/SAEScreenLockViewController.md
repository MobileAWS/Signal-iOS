# SAEScreenLockViewController.swift

Source: [`SignalShareExtension/SAEScreenLockViewController.swift`](../../SignalShareExtension/SAEScreenLockViewController.swift)

## Purpose

**[High]** The screen-lock gate for the share extension. When Signal's screen lock is
enabled, [`ShareViewController.setUp()`](ShareViewController.md) presents this controller
and awaits its completion before letting the share proceed. It subclasses the shared
`ScreenLockViewController` and conforms to `ScreenLockViewDelegate`, driving biometric /
passcode unlock via `ScreenLock.shared`.

## Type

### `SAEScreenLockViewController` (`final class: ScreenLockViewController, ScreenLockViewDelegate`)

Stored state **[High]**:
- `completion: ((_ didUnlock: Bool) -> Void)?` — one-shot callback back to the
  continuation in `ShareViewController`.
- `hasShownAuthUIOnce: Bool` — guards presenting auth UI only once on first appearance.
- `isShowingAuthUI: Bool` — reentrancy guard around the unlock attempt.

### `init(completion:)` — [`:19`](../../SignalShareExtension/SAEScreenLockViewController.swift)

**[High]** Stores the completion and sets `self.delegate = self`.
`init?(coder:)` is `fatalError` (not supported).

### `invokeCompletion(didUnlock:)` — [`:13`](../../SignalShareExtension/SAEScreenLockViewController.swift)

**[High]** Reads and nils `completion`, then invokes it — ensuring it fires at most once.

## View lifecycle

**[High]**
- `viewDidLoad()` — sets the title (`SHARE_EXTENSION_VIEW_TITLE`) and a left "stop" bar
  button that calls `cancelShareExperience()`.
- `viewWillAppear(_:)` / `viewDidAppear(_:)` — call `ensureUI()`; on first appearance set
  `hasShownAuthUIOnce = true` and call `tryToPresentAuthUIToUnlockScreenLock()`.

## Unlock state machine

### `tryToPresentAuthUIToUnlockScreenLock()` — [`:69`](../../SignalShareExtension/SAEScreenLockViewController.swift)

**[High]** Guards on `!isShowingAuthUI`, sets `isShowingAuthUI = true`, and calls
`ScreenLock.shared.tryToUnlockScreenLock(...)` with four callbacks (all assert main thread):

| Callback | Side-effects |
| --- | --- |
| `success` | `isShowingAuthUI = false`; `invokeCompletion(didUnlock: true)` |
| `failure(error)` | `isShowingAuthUI = false`; `ensureUI()`; show failure alert with `error.userErrorDescription` |
| `unexpectedFailure(error)` | `isShowingAuthUI = false`; re-arm UI on next main-queue tick (comment: Local Authentication sometimes recovers after a wait) |
| `cancel` | `isShowingAuthUI = false`; `ensureUI()` |

Note: only `success` resolves the completion. Failure/unexpected/cancel leave the gate up;
the user retries via the unlock button or stops via the stop button.

### Supporting methods **[High]**

- `ensureUI()` → `updateUIWithState(.screenLock)`.
- `showScreenLockFailureAlertWithMessage(_:)` — action sheet
  (`SCREEN_LOCK_UNLOCK_FAILED`), then `ensureUI()`.
- `cancelShareExperience()` → `invokeCompletion(didUnlock: false)` (wired to the stop
  button) → `ShareViewController` cancels the share.
- `unlockButtonWasTapped()` (`ScreenLockViewDelegate`) → `tryToPresentAuthUIToUnlockScreenLock()`.

## Cross-process / shared notes

- **[High]** Unlock is delegated to the shared `ScreenLock.shared`, so the extension honors
  the same screen-lock configuration as the main app (read from shared storage).
- **[Medium]** `didUnlock == false` propagates back to the `withCheckedContinuation` in
  `ShareViewController.setUp()`, which then calls `shareViewWasCancelled()`
  ([ShareViewController.md](ShareViewController.md)). Separately, backgrounding while
  locked triggers dismissal via `applicationDidEnterBackground`.

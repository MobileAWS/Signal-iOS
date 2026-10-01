# ShareViewDelegate.swift

Source: [`SignalShareExtension/ShareViewDelegate.swift`](../../SignalShareExtension/ShareViewDelegate.swift)

## Purpose

**[High]** The protocol that decouples the sending UI
([`SharingThreadPickerViewController`](SharingThreadPickerViewController.md),
[`SharingThreadPickerProgressSheet`](SharingThreadPickerProgressSheet.md),
[`SAELoadViewController`](SAELoadViewController.md)) from the extension's root
([`ShareViewController`](ShareViewController.md)). It lets those controllers report the
outcome of a share — will send / completed / cancelled / failed — and reach the shared
navigation controller, without holding a concrete reference to `ShareViewController`.

## Type

### `ShareViewDelegate` (`public protocol: AnyObject`)

**[High]** A header comment states: "All Observer methods will be invoked from the main
thread." Members:

| Member | Meaning | Implemented by `ShareViewController` as |
| --- | --- | --- |
| `shareViewWillSend()` | about to send; prepare for sending | request an identified chat connection |
| `shareViewWasCompleted()` | send finished successfully | `dismissAndCompleteExtension(error: nil)` |
| `shareViewWasCancelled()` | user/flow cancelled | `dismissAndCompleteExtension(error: .obsoleteShare)` |
| `shareViewFailed(error:)` | unrecoverable failure | `owsFailDebug` + `dismissAndCompleteExtension(error:)` |
| `var shareViewNavigationController: OWSNavigationController { get }` | shared nav controller | `self` |

See [ShareViewController.md](ShareViewController.md) for the concrete implementations and
how each terminal callback leads to `exit(0)`.

## Side-effects / notes

- **[High]** `AnyObject`-constrained so implementers (and the picker's `weak` reference)
  can be held weakly.
- **[Medium]** The main-thread guarantee is a documented contract, not enforced in this
  file; callers in the target invoke it on the main thread (e.g. the picker's
  `@MainActor` `send(...)` flow and `AssertIsOnMainThread()` sites).

# SAELoadViewController.swift

Source: [`SignalShareExtension/SAELoadViewController.swift`](../../SignalShareExtension/SAELoadViewController.swift)

## Purpose

**[High]** The loading screen shown while the share extension boots and while attachments
are being built. It displays either an activity spinner or a determinate progress bar
(when a `Progress` is attached), plus a "loading" label. It can optionally mimic the
recipient picker's header so the transition into the picker is seamless.

## Type

### `SAELoadViewController` (`class: UIViewController, OWSNavigationChildController`)

Stored state **[High]**:
- `weak var delegate: ShareViewDelegate?`.
- `shouldMimicRecipientPicker: Bool` — whether to style the view/header like the picker.
- `activityIndicator: UIActivityIndicatorView!`, `progressView: UIProgressView!`.

### `progress: Progress?` — [`:19`](../../SignalShareExtension/SAELoadViewController.swift)

**[High]** `didSet` updates visibility and binds `progressView.observedProgress`. Set by
`ShareViewController` while building attachments
(`setProgress: { loadViewControllerForProgress?.progress = $0 }` in
[ShareViewController.md](ShareViewController.md)).

### `updateProgressViewVisibility()`

**[High]** No `progress` → show spinner (animating), hide progress bar; `progress` present
→ hide spinner, show progress bar. Guards against nil views.

### `init(delegate:shouldMimicRecipientPicker:)`

**[High]** Stores delegate + flag. `init?(coder:)` is `@available(*, unavailable)` →
`fatalError`.

## View

### `loadView()` — [`:60`](../../SignalShareExtension/SAELoadViewController.swift)

**[High]** Builds the UI. When `shouldMimicRecipientPicker`:
- sets title to `ConversationPickerViewController.Strings.defaultTitle`;
- adds a disabled cancel bar button (so the header matches the picker but does nothing);
- uses the presented table background color.

Otherwise uses the plain background. Then lays out the activity indicator (centered), the
progress bar (observing `progress`, accent tint), and the loading label
(`SHARE_EXTENSION_LOADING`). A code comment notes it is "not (currently) safe to create a
`SharingThreadPickerViewController` while the Share Extension is launching," so this
header-mimicking is a deliberate hack (with a `TODO` to remove).

### `OWSNavigationChildController` conformance **[High]**

- `preferredNavigationBarStyle` → `.solid` when mimicking, else `.blur`.
- `navbarBackgroundColorOverride` → presented table background when mimicking, else `nil`.

## Side-effects / notes

- **[Medium]** Does not itself call the delegate; it only displays state. The delegate is
  held weakly and passed through from `ShareViewController`.
- **[High]** Progress is a pure UIKit `Progress` binding (`observedProgress`); it reflects
  the nested `Progress` tree built in
  [`ShareViewController.buildAndValidateAttachments`](ShareViewController.md).

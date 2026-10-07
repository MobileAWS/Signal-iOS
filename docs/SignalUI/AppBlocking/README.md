# App Blocking

Sources: `SignalUI/AppBlocking/` — `AppBlockingViewController.swift` and
`ClockSkewAppBlockingViewController.swift`.

> Confidence: **[High]** read in full; **[Medium]** signature/partial; **[Low]**
> inferred. Citations are `File.swift:line`. An uncited claim is a defect.

## Responsibility

**[High]** The AppBlocking subsystem provides full-screen "dead-end" views that
block use of the app (or share extension), explaining to the user why they can't
proceed and what they can do about it. `AppBlockingViewController` is documented
as blocking app use, "explaining to the user why and what they can do about it"
(`SignalUI/AppBlocking/AppBlockingViewController.swift:7-8`).

**[High]** The base class is designed for subclassing: "Subclass to describe a
specific blocking scenario. Presentation, and any navigation affordances such as
a cancel button, are the caller's responsibility"
(`SignalUI/AppBlocking/AppBlockingViewController.swift:10-12`). The subsystem
owns only the content/layout of the blocking screen — not how it's presented or
dismissed.

## Key types

### `AppBlockingViewController`

**[High]** An `open class` extending `OWSViewController`
(`SignalUI/AppBlocking/AppBlockingViewController.swift:14`). Its initializer takes
a `headerImage`, `title`, and `subtitle`
(`SignalUI/AppBlocking/AppBlockingViewController.swift:34`) and renders them in a
centered vertical stack: a tinted header `UIImageView`, a semibold headline title
label, and a secondary-colored subheadline subtitle label
(`SignalUI/AppBlocking/AppBlockingViewController.swift:43-82`).

**[High]** `subtitle` is a public mutable property; its `didSet` updates
`subtitleLabel.text` when the view is loaded, "for subclasses whose subtitle
changes while they're displayed"
(`SignalUI/AppBlocking/AppBlockingViewController.swift:24-31`). This is the hook
that lets `ClockSkewAppBlockingViewController` live-update its clock readout.

**[High]** Layout is centered within directional layout margins of `40`
(`Constants.margin`), with stack spacing `16` and `24` after the header image
(`SignalUI/AppBlocking/AppBlockingViewController.swift:16-21`,
`:84-104`). The background uses `.Signal.background` and the header image is
tinted `.Signal.label` (`:48`, `:89`).

### `ClockSkewAppBlockingViewController`

**[High]** A `public` subclass that blocks the app "because the device's clock is
too far skewed from the server's for us to connect"
(`SignalUI/AppBlocking/ClockSkewAppBlockingViewController.swift:6-10`). It
hardcodes the `"timer"` header image and localized clock-skew title, and seeds
its subtitle from `buildSubtitle()`
(`SignalUI/AppBlocking/ClockSkewAppBlockingViewController.swift:15-30`).

**[High]** Its init takes an `onSubmitDebugLogs: @MainActor (UIViewController) ->
Void` closure
(`SignalUI/AppBlocking/ClockSkewAppBlockingViewController.swift:13-14`), invoked
from a hidden affordance — an 8-tap gesture recognizer added in `viewDidLoad`
(`:40-50`). The comment explains the rationale: the user is dead-ended, so this
hidden gesture lets them submit debug logs "in case something with clock-skew
calculations is buggy" (`:37-39`).

## Important data flows / state

**[High]** Clock-skew live update: on `viewWillAppear`, the controller rebuilds
its subtitle and schedules a repeating 1-second `Timer` that reassigns
`self.subtitle` each tick
(`SignalUI/AppBlocking/ClockSkewAppBlockingViewController.swift:52-64`). Because
the base class's `subtitle.didSet` repaints the label, the displayed device time
ticks forward live. The timer is invalidated and nilled in `viewDidDisappear`
(`:66-71`) and again in `deinit` (`:32-34`), avoiding a retained repeating timer.

**[High]** `buildSubtitle()` joins a localized explanation with the device's
current time, formatted via a static `DateFormatter` pinned to UTC
(`SignalUI/AppBlocking/ClockSkewAppBlockingViewController.swift:76-98`). The UTC
choice is deliberate: "so what we show is unambiguous regardless of the device's
time zone (which may itself be wrong)" (`:74-75`).

## Interactions with the rest of SignalUI and the app

**[High]** The base class builds on SignalUI foundations: `OWSViewController`,
the `.Signal.*` semantic colors, and the `UILabel.headlineLabel` /
`UILabel.subheadlineLabel` helpers
(`SignalUI/AppBlocking/AppBlockingViewController.swift:14`, `:55`, `:62`, `:48`,
`:89`).

**[High]** Main app (clock skew): `WindowManager` hosts
`ClockSkewAppBlockingViewController` as the root view controller of a dedicated
`clockSkewBlocking` window at window level `._clockSkewBlocking`, wiring the
`onSubmitDebugLogs` closure to `DebugLogs(...).promptToSubmitLogs(...)` with
support tag `"ClockSkew"` (`Signal/AppLaunch/WindowManager.swift:95-108`).
Presentation and window layering are the caller's responsibility, consistent with
the base class's contract. See also `docs/Signal/scene-window-management.md:118`.

**[High]** Share extension: `ShareViewController` instantiates
`AppBlockingViewController` directly (not a subclass) for two scenarios — "not
registered" (`SignalShareExtension/ShareViewController.swift:292-304`) and "low
disk space" (`:306-318`) — then presents it via `showBlockingView(_:)`, which
adds a `"Signal"` nav title and a cancel bar button that cancels the share
(`:321-331`). Here the blocking is cancelable, demonstrating the "caller owns
navigation affordances" design. See also
`docs/SignalShareExtension/ShareViewController.md:157`.

## Notable UI considerations

- **[High]** Centered, margin-constrained layout with a `greaterThanOrEqualTo`
  top/leading constraint plus center constraints keeps content centered but
  prevents overflow past the margins
  (`SignalUI/AppBlocking/AppBlockingViewController.swift:97-104`).
- **[High]** Dead-end UX by design: no built-in dismissal; callers decide whether
  the screen is cancelable (share extension) or terminal (clock-skew window).
- **[High]** Hidden debug-log escape hatch via an 8-tap gesture avoids exposing a
  support affordance in normal UI while still letting a stuck user recover
  diagnostics (`SignalUI/AppBlocking/ClockSkewAppBlockingViewController.swift:40-50`).
- **[High]** UTC time formatting avoids confusion when the device's own time zone
  is misconfigured (`:72-98`).
- **[High]** Both controllers ship DEBUG `#Preview`s for iOS 17+
  (`AppBlockingViewController.swift:108-120`,
  `ClockSkewAppBlockingViewController.swift:102-114`).

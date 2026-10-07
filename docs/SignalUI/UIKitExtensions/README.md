# UIKitExtensions

Source: `SignalUI/UIKitExtensions/`. This subsystem is the pile of
category/extension helpers on bare UIKit types (`UIView`, `UIButton`,
`UIStackView`, `UITableView`, `UIViewController`, `UIFont`, `UIImage`,
`UILabel`/`UITextView`, geometry types, `CALayer`, etc.). It gives the rest of
SignalUI and the app a consistent vocabulary for Auto Layout, animation, image
processing, text measurement, button/bar-button styling, typography, and
view-controller presentation — so feature code doesn't re-implement these
primitives.

> This README is the **deep dive** for `SignalUI/UIKitExtensions/`. The
> higher-level [`../Extensions.md`](../Extensions.md) summarizes both this
> directory and `SignalUI/SwiftUIExtensions/`; it is intentionally left
> unchanged. Where the two overlap (PureLayout, `SpacerView`, the ObjC
> deprecation workaround), this document goes into the per-file detail.

> Confidence: **[High]** read in full; **[Medium]** signature/partial read or
> depends on code outside `SignalUI/UIKitExtensions/`; **[Low]** inferred.
> Citations are `File.swift:line`. An uncited claim is a defect.

## Responsibility & boundaries

**[High]** Every file in this directory is a Swift `extension`/`public
extension` (or one Objective-C category) on a UIKit, Core Animation, or Core
Graphics type. There are only two *new* types declared in the whole directory:
`SpacerView` (`SignalUI/UIKitExtensions/UIView+SignalUI.swift:11`) and
`OWSTableViewDiffableDataSource`
(`SignalUI/UIKitExtensions/OWSTableViewDiffableDataSource.swift:9`), plus a
nested `UIView.TransparentView`
(`SignalUI/UIKitExtensions/UIView+SignalUI.swift:93`) and a
`ReusableTableViewCell` protocol
(`SignalUI/UIKitExtensions/UITableView+ReusableCell.swift:8`). The rest is pure
API surface bolted onto existing classes.

**[High]** Almost every file `import SignalServiceKit` (SSK) — e.g.
`UIView+SignalUI.swift:7`, `UIKit+Text.swift:6`, `UIFont+OWS.swift:6` — because
the helpers reach for SSK utilities such as `owsFailDebug`/`OWSAssertionError`,
`CurrentAppContext()` (RTL/frame/`isMainApp`), `CGFloat.clamp`/`lerp`, and
`CommonStrings`. Two files re-export their dependency with `public import`
(PureLayout in `UIView+AutoLayout.swift:6`, SSK in `UIKit+Image.swift:6` and
`UIFont+TextStyle.swift:5`), so those symbols flow through to SignalUI
consumers.

## Auto Layout (`UIView+AutoLayout.swift`)

**[High]** Built on the **PureLayout** dependency (`public import PureLayout`,
`SignalUI/UIKitExtensions/UIView+AutoLayout.swift:6`), this file adds Signal's
convenience constraint vocabulary on top of PureLayout's `ALEdge`/`autoPin…`
primitives. Representative helpers:

- Superview pinning: `autoPinEdge(toSuperviewEdge:relation:)`
  (`:13`), `autoPinEdges(toSuperviewEdgesExcludingEdge:)` (`:18`),
  `autoPinEdges(toSuperviewSafeAreaExcludingEdge:)` (`:24`).
- Margin pinning: `autoPinLeadingToSuperviewMargin(withInset:)` (`:31`),
  `autoPinWidthToSuperviewMargins(relation:)` (`:48`).
- Edge-to-edge matching against another view: `autoPinEdges(toEdgesOf:with:)`
  (`:123`), `autoPinLeading(toTrailingEdgeOf:offset:)` (`:133`),
  `autoPinHorizontalEdges(toEdgesOf:)` (`:146`).
- Dimension matching: `autoPinWidth(toWidthOf:…)` (`:187`),
  `matchWidthsOfViews(_:)` / `matchHeightsOfViews(_:)` (`:195`, `:206`).
- Aspect ratio: `autoPin(toAspectRatio:relation:)` clamps the ratio to
  `[0.05, 95.0]` and `owsFailDebug`s if the caller passed something out of range
  (`:233-262`).
- Content hugging / compression resistance convenience setters
  (`setContentHuggingLow()` etc., `:266-325`), and `deactivateAllConstraints()`
  (`:327`).

**[High]** A subtle, well-commented detail: several width/height helpers invert
the caller's `NSLayoutConstraint.Relation` because "width ≤ superview" must be
expressed as "leading ≥ … / trailing ≤ …" at the edge level. The inversion is
done via `NSLayoutConstraint.Relation.inverse`
(`SignalUI/UIKitExtensions/UIView+AutoLayout.swift:333-344`) and the reasoning
is spelled out inline (e.g. `:52-58`, `:90-96`). This is a correctness trap for
anyone adding similar helpers.

## Core view helpers (`UIView+SignalUI.swift`)

**[High]** `SpacerView` is a non-rendering spacer: it overrides `layerClass` to
`CATransformLayer` (which draws nothing) and exposes a settable
`intrinsicContentSize` seeded from a `preferredSize`
(`SignalUI/UIKitExtensions/UIView+SignalUI.swift:11-39`). Alongside it is a
family of factory spacers built on the nested, likewise non-rendering
`UIView.TransparentView` (`:93-97`): `spacer(withWidth:)` / `spacer(withHeight:)`
(`:43-51`), `hStretchingSpacer()` / `vStretchingSpacer(minHeight:maxHeight:)`
(`:60-82`), and `transparentContainer()` (`:155-159`). The file comment notes
the efficiency rationale: when a container needs no background color, prefer the
non-rendering view (`:151-153`).

**[High]** Other broadly-used helpers in this file:

- `setShadow(radius:opacity:offset:color:)` (`:99`),
  `addBorder(with:)` / `addRedBorder()` (`:161-170`),
  `addBottomStroke(color:strokeWidth:)` (`:303`).
- `addCircleBadge(color:onCircleView:circleDiameter:badgeDiameter:overlap:)`
  draws a small overlapping circular badge at a 45° diagonal, using
  `OWSLayerView.circleView(size:)` (`:172-198`).
- Accessibility identifier helpers that namespace the id as
  `"\(type(of: container)).\(name)"`
  (`:112-124`) — used so UI tests can address views deterministically.
- Hierarchy traversal: `traverseHierarchyUpward`/`Downward`
  (`:247-266`) and `firstAncestor(ofType:)` (`:268`).
- Hit-testing: `containsGestureLocation(_:hotAreaInsets:)`, where **negative**
  insets grow the tappable "hot area" (`:270-293`).
- `hairlineWidth` on `UIView`/`UITraitCollection`/`UIViewController`, defined as
  `1 / displayScale` (`:297`, `:311-317`,
  `UIViewController+SignalUI.swift:176`).
- `UIWindow.shouldHideStatusBarForFullScreenPresentation` encapsulates the
  "legacy phone with ≤20pt top safe area" check (`:343-349`).
- `UIApplication.hideKeyboard()` resigns first responder app-wide (`:331`).
- Manual-layout frame conveniences `left`/`right`/`top`/`bottom`/`width`/`height`
  (`:203-215`) and `#if DEBUG` frame/hierarchy logging (`:219-243`).

## Animation (`UIKit+Animations.swift`, `UIViewPropertyAnimator+SignalUI.swift`)

**[High]** `UIViewPropertyAnimator.init(duration:springDamping:springResponse:…)`
derives UIKit `UISpringTimingParameters` (stiffness/damping) from the more
designer-friendly *damping ratio* and *response time*
(`SignalUI/UIKitExtensions/UIKit+Animations.swift:8-28`).

**[High]** `UIView.setIsHidden(_:animated:…)` / `setIsHidden(_:using:)` fade a
view's `alpha` while toggling `isHidden`, carefully short-circuiting when the
state is unchanged and restoring `alpha` to 1 after hiding
(`SignalUI/UIKitExtensions/UIKit+Animations.swift:118-199`).
`animateDecelerationToVerticalEdge(…)` implements a flick-to-edge physics
animation: it projects the current velocity to find which bounding edge is hit
first and springs the frame there (`:33-116`).

**[High]** `UIView.AnimationCurve.asAnimationOptions` (and the `Optional`
variant) bridge a curve to `UIView.AnimationOptions`
(`SignalUI/UIKitExtensions/UIKit+Animations.swift:201-220`).

**[High]** `UIViewPropertyAnimator.addAnimations(withDurationFactor:_:)` lets a
sub-animation finish earlier than the animator's inherited duration while still
respecting its timing curve, by wrapping a single keyframe
(`SignalUI/UIKitExtensions/UIViewPropertyAnimator+SignalUI.swift:7-20`).

### Core Animation / Core Graphics bits (same file)

**[High]** `UIKit+Animations.swift` also carries non-animation CA/CG helpers:

- `UIBezierPath.roundedRect(…)` builds bubble-shaped paths with independent
  per-corner radii, including a two-radius `sharpCorners`/`wideCorners` variant
  (`:242-370`). This underpins the message-bubble rendering described in
  [`../Views.md`](../Views.md).
- RTL-aware corner mapping `UIView.uiRectCorner(forOWSDirectionalRectCorner:)`
  flips leading/trailing corners based on `CurrentAppContext().isRTL`
  (`:224-240`).
- `CALayer.disableAnimationsWithDelegate()` installs a shared `CALayerDelegate`
  that returns `NSNull` from `action(for:forKey:)` to suppress implicit layer
  animations (`:372-399`).
- `CGAffineTransform` static/instance `translate`/`scale`/`rotate` sugar
  (`:401-423`) and `CACornerMask` named sets `.top`/`.bottom`/`.left`/`.right`/
  `.all` (`:425-432`).

## Buttons & bar buttons (`UIButton+SignalUI.swift`)

**[High]** This is the largest file (~23 KB) and the de-facto button
design-system entry point. It is built almost entirely on
`UIButton.Configuration` factories that branch on **iOS 26** for the Liquid
Glass material, falling back to pre-26 styles:

- Named title styles: `largePrimary`, `largeSecondary`, `mediumSecondary`,
  `mediumBorderless`, `smallSecondary`, `smallBorderless`
  (`SignalUI/UIKitExtensions/UIButton+SignalUI.swift:139-215`). Primary uses
  `.prominentGlass()` on iOS 26 else `.borderedProminent()` and tints with
  `.Signal.accent` (`:110-124`).
- Round/media styles: `round(image:)` / `round(themeIcon:)`,
  `roundMaterial(image:)`, `roundGray(image:)`, `roundMedia(…)`,
  `tintedRoundMedia(…)`, `capsuleMedia(…)` (`:218-337`), several of which fall
  back to a `UIVisualEffectView(UIBlurEffect)` background on pre-26.
- Corner handling is centralized in a private `applyCorners()` that uses
  `.capsule` on iOS 26 and a fixed 14pt radius otherwise (`:100-108`).
- Fonts flow through `UIConfigurationTextAttributesTransformer.defaultFont(_:)`,
  which applies a default font only when the attributed title hasn't set one
  (`:85-98`).

**[High]** Imperative `UIButton` helpers: `setTemplateImage(_:tintColor:)` /
`setTemplateImageName(…)` (`:25-45`), cross-dissolve `setImage(_:animated:)` /
`setImage(_:withAnimationDuration:)` (`:47-62`), `enableMultilineLabel()`
(`:64-78`), and `enclosedInVerticalStackView(isFullWidthButton:)` which delegates
to `UIStackView.verticalButtonStack` (`:80-91`).

**[High]** `UIBarButtonItem` factories wrap `UIAction` closures and encode
Signal's nav-bar conventions: `button(title:action:)` /
`prominentButton(title:action:)` (`:347-369`), icon/image variants
(`:371-401`), and a rich family of **Cancel/Done/Close/Set/Next** buttons.
Notably `cancelButton(dismissingFrom:hasUnsavedChanges:…)` presents
`OWSActionSheets.showPendingChangesActionSheet` before dismissing when there are
unsaved edits (`:470-490`), and `contextMenuButton(actions:)` builds the "•••"
overflow menu (`:558-568`). iOS-26-specific prominence/tint handling is folded
in throughout (e.g. `prominentBarButtonItemStyle`, `:341`). **[Medium]** The
`CommonStrings`/`ThemeIcon`/`Theme.iconImage` usages come from SSK/SignalUI
outside this directory.

**[High]** `UIToolbar.clear()` produces a fully transparent toolbar by setting an
empty background image and clipping the 1px top border
(`:572-585`). A `#if DEBUG` `#Preview` renders every button style for visual
regression checking (`:589-640`).

## The one Objective-C file (`UIButton+DeprecationWorkaround.{h,m}`)

**[High]** `UIButton+DeprecationWorkaround` is the only Objective-C source in the
directory. It re-exposes three UIKit `UIButton` properties that Apple deprecated
— `adjustsImageWhenHighlighted`, `contentEdgeInsets`, `imageEdgeInsets` — under
`ows_`-prefixed names, with the deprecation warning locally silenced via
`#pragma clang diagnostic ignored "-Wdeprecated-declarations"`
(`SignalUI/UIKitExtensions/UIButton+DeprecationWorkaround.m:7-45`;
header at `UIButton+DeprecationWorkaround.h:10-16`). It exists so Swift callers
can keep using these properties without triggering deprecation warnings in Swift
(which cannot `#pragma`-suppress them per-call). **[High]** Its header is the
single file re-exported by the SignalUI umbrella header
(`SignalUI/SignalUI.h:8`), making it visible to the whole module.

## Stack views (`UIStackView+SignalUI.swift`)

**[High]** Arranged-subview conveniences: `addArrangedSubviews(_:)` (`:19`),
`removeArrangedSubviewsAfter(_:)` (`:25`), and hairline insertion
`addHairline(with:)` / `insertHairline(with:at:)` (`:34-45`). Background
decoration helpers add a subview pinned behind the stack:
`addBackgroundView(_:)` / `addBackgroundView(withBackgroundColor:cornerRadius:)`
(`:47-66`), and `addBackgroundBlurView(blur:accessibilityFallbackColor:…)` which
substitutes a solid color when `UIAccessibility.isReduceTransparencyEnabled`
(`:68-96`) — an accessibility consideration worth preserving. `addBorderView(…)`
adds a non-interactive stroked overlay (`:98-110`).

**[High]** Layout factories: `verticalButtonStack(buttons:isFullWidthButtons:)`
(`:112-120`, the backing for `UIButton`'s stack helper) and
`bulletPointsStack(content:)` for image+text bullet rows (`:122-152`). A shared
`NSDirectionalEdgeInsets.buttonContainerLayoutMargins` constant lives here too
(`:9-14`).

**[High]** `UIView.isHiddenInStackView` works around a long-standing
`UIStackView` bug where repeatedly setting `isHidden` can make hidden subviews
re-appear; it guards the assignment and also drives `alpha`
(`:155-168`).

## Table views (`UITableView+ReusableCell.swift`, `+RowHeights.swift`, `OWSTableViewDiffableDataSource.swift`)

**[High]** `ReusableTableViewCell` is a tiny protocol exposing a static
`reuseIdentifier` (`SignalUI/UIKitExtensions/UITableView+ReusableCell.swift:8`),
with typed `register(_:)` / `dequeueReusableCell(_:)` /
`dequeueReusableCell(_:for:)` wrappers that remove stringly-typed reuse
identifiers from call sites (`:11-41`).

**[High]** `UITableView.recomputeRowHeights()` forces a height recomputation via
an empty `beginUpdates()`/`endUpdates()` pair, and intentionally `owsFailDebug`s
if the data source's type name contains `"Diffable"`, because mixing it with a
diffable snapshot can crash — the doc comment steers callers to
`reconfigureItems` instead
(`SignalUI/UIKitExtensions/UITableView+RowHeights.swift:7-32`).

**[High]** `OWSTableViewDiffableDataSource<SectionIdentifier, ItemIdentifier>`
subclasses `UITableViewDiffableDataSource` to expose closure hooks the base
class hides: `canMoveRow`/`didMoveRow` for reordering
(`SignalUI/UIKitExtensions/OWSTableViewDiffableDataSource.swift:16-37`) and
`sectionIndexTitlesProvider`/`sectionForSectionIndexTitleProvider` for the
section index sidebar (`:41-66`). This is distinct from — and complements — the
`OWSTableViewController` family documented in
[`../ViewControllers.md`](../ViewControllers.md).

## Text measurement & labels (`UIKit+Text.swift`)

**[High]** The headline capability here is **character/rect hit-testing** on
`UILabel`/`UITextView`: `NSTextContainer.characterIndex(of:…)` resolves a point
to a character index, taking care to reject points that fall in the blank space
below a short line by checking the glyph's own rect
(`SignalUI/UIKitExtensions/UIKit+Text.swift:93-125`). `boundingRects(ofCharacterRanges:…)`
(with a generic range-mapping/transform variant) returns per-line rects for
ranges, used for things like link/mention highlighting
(`:127-176`). `UILabel.characterIndex(of:)` builds a throwaway
`NSTextContainer`/`NSTextStorage`/`NSLayoutManager` to match the label's layout
(`:180-233`); a comment flags that this is imprecise for complex labels and
should eventually be replaced by `UITextView` (`:225-227`).

**[High]** Themed label factories (`title1Label`, `title2Label`,
`headlineLabel`, `subheadlineLabel`, `explanationTextLabel`) wire up
`.Signal.*` colors, clamped dynamic-type fonts, 0-line word wrapping, and
alignment (`:235-290`) — the standard Signal label presets.

**[High]** Input/keyboard utilities: `acceptAutocorrectSuggestion()` on
`UITextField`/`UITextView` (`:18-37`) and `disableAiWritingTools()` which sets
`writingToolsBehavior = .none` on iOS 18+ for `UITextView`/`UITextField`/
`UISearchBar` (`:39-61`). `NSTextAlignment.trailing` is an RTL-aware
left/right alias (`:293-298`) and `NSTextAlignment` gains a
`CustomStringConvertible` description (`:302-320`).

## Images (`UIKit+Image.swift`, `UIImage+Blur.swift`)

**[High]** `UIKit+Image.swift` provides `UIImageView` template-image factories
(`setTemplateImage(_:tintColor:)`, `withTemplateIcon(_:tintColor:constrainedTo:)`
etc., `:8-71`) and `UIImage` compositing helpers:
`withBackgroundColor(_:insets:)` (`:76`), `normalized()` (which re-draws to bake
in `imageOrientation`, `:85`), `withTitle(…)` (renders a caption under an image,
`:101`), and `withBadge(color:badgeSize:)` (draws a corner dot, `:166`).
`UIView.renderAsImage(opaque:scale:)` rasterizes a view via
`UIGraphicsImageRenderer` (`:197-213`).

**[High]** `UIImage+Blur.swift` is the Core Image Gaussian-blur pipeline. It
defines a `CompositingMode` enum mapping to CI filter names
(`SignalUI/UIKitExtensions/UIImage+Blur.swift:10-21`) and exposes
`@concurrent async` blur entry points
(`withGaussianBlurAsync(radius:resizeToMaxPixelDimension:)` /
`cgImageWithGaussianBlurAsync(…)`, `:25-37`) that assert off-main-thread
(`AssertNotOnMainThread()`), plus synchronous `withGaussianBlur(…)` overloads
(`:48-70`). The private pipeline applies `CIAffineClamp` → `CIGaussianBlur` →
optional color overlays → optional `CIVibrance`/`CIExposureAdjust`, documented
step-by-step inline (`:72-181`). The clamp-before-blur ordering is a deliberate
fix for edge artifacts (comment at `:88`).

## Typography (`UIFont+OWS.swift`, `UIFont+TextStyle.swift`)

**[High]** `UIFont+OWS.swift` defines Signal's dynamic-type ramp. Plain
`dynamicType*` accessors wrap `UIFont.preferredFont(forTextStyle:)`
(`:52-72`), while the `dynamicType*Clamped` variants cap growth at a per-style
`maxPointSizeMap` using `UIFontMetrics` so dynamic scaling is preserved
(`:78-150`). The inline comment explains *why* clamping must go through
`UIFontMetrics` rather than constructing a smaller `UIFont` — otherwise dynamic
sizing is lost (`:98-118`). Weight/style transforms (`italic()`, `medium()`,
`semibold()`, `bold()`, `monospaced()`) operate on font descriptors
(`:154-184`). This ramp is covered from the typography angle in
[`../FontsAndFormatStyles.md`](../FontsAndFormatStyles.md).

**[High]** `UIFont+TextStyle.swift` builds the **text-story / text-attachment**
fonts: `font(for: TextAttachment.TextStyle, withPointSize:)` picks a primary
font (Inter, EB Garamond, Parisienne, Barlow Condensed, …) and a per-script
`.cascadeList` fallback chain for CJK/Devanagari/Arabic coverage
(`SignalUI/UIKitExtensions/UIFont+TextStyle.swift:10-131`). `digitalClockFont(withPointSize:)`
vends the 7-segment `Hatsuishi-UPM800` face used for clock-style numerals
(`:133-142`). These depend on bundled fonts (see `SignalUI/Fonts/`). **[Medium]**

## View-controller presentation (`UIViewController+SignalUI.swift`)

**[High]** Navigation glue: `UIViewController.owsNavigationController`
down-casts to `OWSNavigationController` (`:10`), and
`findFrontmostViewController(ignoringAlerts:)` walks presented/navigation
chains, guarding against cycles with a visited-set and optionally skipping
`ActionSheetController`/`UIAlertController` (`:14-50`).

**[High]** Presentation sugar: `presentActionSheet(_:…)` (`:56`),
`presentFullScreen(_:animated:…)` (which forces `.fullScreen` to avoid the iOS
13 card style, `:64-69`), and `presentFormSheet(_:…)` which configures a
`.large()` detent sheet (`:71-83`). Completion-based
`UINavigationController.pushViewController/popViewController/popToViewController`
route through `transitionCoordinator` so the completion fires after the
animation (`:89-137`), and `awaitable*` wrappers
(`awaitablePush`/`awaitableDismiss`/`awaitablePresent`/`awaitablePresentFormSheet`)
expose `async` versions via `withCheckedContinuation`
(`:139-172`).

## Geometry (`UIGeometry+Signal.swift`)

**[High]** Small value-type helpers: `UIEdgeInsets.inverted()` (`:8-12`),
`CGSize.isLandscape`/`isPortrait` (`:16-24`), `CGPoint.offsetBy(dx:dy:)` /
`offsetBy(_:)` (`:27-35`), and the responsive-layout scalers
`CGFloat.scaleFromIPhone5To7Plus(_:_:)` / `scaleFromIPhone5(_:)` which
interpolate a value by current screen width via
`CurrentAppContext().frame.size.smallerAxis` (`:39-62`).

## Interactions with the rest of SignalUI & the app

- **PureLayout re-export** — because `UIView+AutoLayout.swift:6` uses
  `public import PureLayout`, PureLayout's API is available to every SignalUI
  consumer, and the `Views/` subsystem's manual-layout code deliberately avoids
  it for perf (see [`../Views.md`](../Views.md)). **[Medium]**
- **Appearance/theme** — button, label, and image helpers resolve colors via
  `.Signal.*` and icons via `Theme.iconImage(_:)` / `ThemeIcon`, tying this
  directory to the theming model in [`../Appearance.md`](../Appearance.md).
  **[Medium]**
- **View controllers** — the presentation/navigation helpers assume the
  `OWSNavigationController` / `OWSTableViewController` stack described in
  [`../ViewControllers.md`](../ViewControllers.md), and
  `OWSTableViewDiffableDataSource` plugs into those tables. **[Medium]**
- **App context (SSK)** — RTL flipping, screen frame, `isMainApp`, and assertion
  helpers all come from `CurrentAppContext()`/SSK, so behavior can differ
  between the main app and extensions. **[High]** (`UIKit+Animations.swift:234`,
  `UIGeometry+Signal.swift:51`, `UIKit+Text.swift:296`.)

## Notable UI / correctness considerations

- **Non-rendering views for perf:** `SpacerView`/`TransparentView` back their
  layer with `CATransformLayer` so they never draw
  (`UIView+SignalUI.swift:16`, `:94`). Don't set `backgroundColor` on them —
  it won't render.
- **Reduce-Transparency fallback:** blurred stack-view backgrounds fall back to
  a solid color when the accessibility setting is on
  (`UIStackView+SignalUI.swift:76-86`).
- **RTL correctness:** corner masks, text alignment, and bubble corners are all
  RTL-aware via `CurrentAppContext().isRTL`
  (`UIKit+Animations.swift:230`, `UIKit+Text.swift:296`).
- **Diffable data source hazard:** `recomputeRowHeights()` must not be used with
  a diffable data source (`UITableView+RowHeights.swift:19-26`).
- **Thread discipline:** Gaussian blur asserts it runs off the main thread
  (`UIImage+Blur.swift:28`, `:34`); hierarchy traversal asserts main-thread
  (`UIView+SignalUI.swift:249`, `:261`).
- **Dynamic-type preservation:** clamp fonts via `UIFontMetrics`, never by
  rebuilding a smaller `UIFont` (`UIFont+OWS.swift:98-118`).
- **iOS 26 Liquid Glass branching:** button/bar-button styling has two code
  paths (glass vs. pre-26 fallbacks) throughout
  `UIButton+SignalUI.swift` — changes must update both.

> **Intent note:** per-file comments explain the *mechanics* (relation
> inversion, clamp-before-blur, the `UIStackView` hidden-subview bug, the
> deprecation `#pragma`), but there is no directory-level design doc in source
> describing an overarching philosophy; the "subsystem responsibility" framing
> above is **inferred from the uniform shape of the files** rather than stated in
> code. **[Low]**

```mermaid
graph TD
    subgraph UIKitExtensions
        AL[UIView+AutoLayout\nPureLayout vocab]
        VU[UIView+SignalUI\nSpacerView, hit-test, a11y ids]
        AN[UIKit+Animations\nspring, setIsHidden, bezier/CA]
        BTN[UIButton+SignalUI\nConfiguration + BarButtonItem]
        OBJC[UIButton+DeprecationWorkaround.m\nObjC category]
        SV[UIStackView+SignalUI\nbackgrounds, button stacks]
        TBL[UITableView+* / OWSTableViewDiffableDataSource]
        TXT[UIKit+Text\nhit-test, label presets]
        IMG[UIKit+Image / UIImage+Blur]
        FONT[UIFont+OWS / UIFont+TextStyle]
        VC[UIViewController+SignalUI\npresentation + async]
        GEO[UIGeometry+Signal]
    end

    AL -->|public import| PureLayout
    BTN --> OBJC
    BTN --> SV
    OBJC -. re-exported by .-> UmbrellaHeader[SignalUI.h]
    BTN -. .Signal.* / ThemeIcon .-> Appearance[Appearance.md]
    VC -. OWSNavigationController .-> ViewControllers[ViewControllers.md]
    TBL -. OWSTableViewController .-> ViewControllers
    AL -. avoided by .-> ManualLayout[Views.md manual layout]
    FONT -. typography .-> Fonts[FontsAndFormatStyles.md]
    VU -. CurrentAppContext/owsFailDebug .-> SSK[SignalServiceKit]
    AN -. isRTL/CGFloat math .-> SSK
```

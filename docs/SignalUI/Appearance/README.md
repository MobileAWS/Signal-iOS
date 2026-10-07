# Appearance subsystem (`SignalUI/Appearance/`)

Source: `SignalUI/Appearance/`. This directory is the appearance layer of the
shared `SignalUI` framework. It owns four related responsibilities:

1. the **theme model** (light/dark state and its propagation) — `Theme`;
2. the **design-system color palette and tokens** — `UIColor.Signal` /
   `Color.Signal`, `ThemedColor` resolution, and the `ColorOrGradient` rendering
   bridge;
3. **conversation-rendering style** — the immutable per-render snapshot
   (`ConversationStyle`), bubble geometry (`BubbleConfiguration`), and deterministic
   per-sender group colors (`GroupNameColors`);
4. **iconography & themed SwiftUI primitives** — the bundled `SignalSymbol` font
   API, `ThemeIcon` asset lookups, and the themed `SignalList`/`SignalSection`
   SwiftUI container plus its scroll-offset machinery.

> **Scope note.** A companion doc, `docs/SignalUI/Appearance.md`, already covers
> the `Theme` singleton, the `.themeDidChange` propagation path, the
> `UIColor.Signal` palette, and `ThemedColor` in depth. This README is the
> *directory-level* overview of the whole subsystem; it summarizes the theming
> core briefly and goes deeper on the rendering/iconography/SwiftUI types that the
> companion doc only lists by signature. Read both together.

> Confidence labels: **[High]** read in full in this directory; **[Medium]**
> signature/partial read, or depends on code outside `SignalUI/Appearance/`;
> **[Low]** inferred from naming/comments. Citations are `File.swift:line`. Line
> numbers reflect the tree at authoring time and may drift — re-locate by symbol
> name. An uncited factual claim is a defect.

## Directory contents at a glance

| File | Role | Confidence |
| --- | --- | --- |
| `Theme.swift` | Light/dark state singleton, semantic color accessors, `.themeDidChange` | **[High]** (summarized here; detailed in `Appearance.md`) |
| `ThemedColor+Theme.swift` | Resolve SSK `ThemedColor` against the SignalUI theme | **[High]** |
| `Theme+Icons.swift` | `ThemeIcon` enum → asset-catalog image-name lookup | **[High]** |
| `UIColor+Signal.swift` | `UIColor.Signal` / `Color.Signal` palette + trait-aware initializers | **[High]** |
| `SignalSymbols.swift` | `SignalSymbol` custom-font glyph API (UIKit + SwiftUI) | **[High]** |
| `GroupNameColors.swift` | Deterministic per-sender color map for group threads | **[High]** |
| `ConversationStyle.swift` | Immutable per-render styling snapshot for the conversation view | **[High]** |
| `BubbleConfiguration.swift` | Message-bubble corner/stroke geometry → `UIBezierPath` | **[High]** |
| `ColorOrGradient+SignalUI.swift` | SSK `ColorOrGradientSetting` → renderable `ColorOrGradientValue` | **[High]** |
| `ColorOrGradientSwatchView.swift` | A `UIView` that renders a color/gradient/blur swatch | **[High]** |
| `SwiftUI/SignalList.swift` | Themed `List`/`Section` SwiftUI containers | **[High]** |
| `SwiftUI/ScrollOffset.swift` | Preference-key machinery to read a scroll view's offset | **[High]** |

## Theming core (summary)

**[High]** `Theme` is the single source of truth for "are we dark right now?"
via `Theme.isDarkThemeEnabled` and the semantic color accessors
(`backgroundColor`, `primaryTextColor`, …). Appearance changes are broadcast as a
single `.themeDidChange` `NSNotification`
(`SignalUI/Appearance/Theme.swift:9`), which `OWSViewController` subclasses observe
and route to an overridable `themeDidChange()` hook. See
`docs/SignalUI/Appearance.md` for the full state-resolution priority order, the
share-extension override, and the propagation sequence diagram.

**[High]** The rest of this directory consumes that one dark/light flag. Nearly
every type here either reads `Theme.isDarkThemeEnabled` directly or captures it
once per render:

- `ThemedColor.forCurrentTheme` resolves a light/dark pair with
  `Theme.isDarkThemeEnabled`, and `ThemedColor.fixed(_:)` builds a
  theme-independent color (`SignalUI/Appearance/ThemedColor+Theme.swift:10-19`).
  This is the bridge between SSK's `ThemedColor` value type and SignalUI's own
  theme flag.
- `ColorOrGradientSetting.asValue` / `asValue(themeMode:)` select dark colors
  using the same flag, unless a `ColorOrGradientThemeMode` of `.alwaysLight` /
  `.alwaysDark` forces one branch
  (`SignalUI/Appearance/ColorOrGradient+SignalUI.swift:62-76`).
- `GroupNameColors.forThread(_:)` and `ConversationStyle.init` both snapshot
  `Theme.isDarkThemeEnabled` at construction time (see below).

## Color palette & tokens — `UIColor.Signal` / `Color.Signal`

**[High]** The design-system palette is exposed as a namespaced empty enum,
declared twice — once for UIKit (`UIColor.Signal`,
`SignalUI/Appearance/UIColor+Signal.swift:64`) and once for SwiftUI
(`Color.Signal`, `SignalUI/Appearance/UIColor+Signal.swift:503`) — with parallel
token accessors (e.g. `UIColor.Signal.ultramarine`
`SignalUI/Appearance/UIColor+Signal.swift:71`, `accent` = `ultramarine`
`:125`; and `Color.Signal.ultramarine` `:510`, `accent` `:534`). Keeping the two
namespaces in lockstep lets UIKit and SwiftUI code reference the same semantic
tokens.

**[High]** Tokens are declared trait-aware through two helpers in this file:

- `UIColor.byRGBHex(...)` builds a color from a hex literal
  (`SignalUI/Appearance/UIColor+Signal.swift:26`).
- `UIColor(light:lightHighContrast:dark:darkHighContrast:)` is a custom
  convenience initializer that returns a dynamic `UIColor` resolving against the
  trait collection's `userInterfaceStyle` and `accessibilityContrast`
  (`SignalUI/Appearance/UIColor+Signal.swift:39-57`). Because the resulting
  colors are dynamic, they adapt to both dark mode and the high-contrast
  accessibility setting without any explicit theme check.

**[Medium]** The palette is widely consumed across the app (e.g. ~20 files
reference `SignalSymbol.`, and `UIColor.Signal`/`Color.Signal` tokens appear
throughout `Signal/` and `SignalUI/`). Full enumeration of consumers is out of
scope here; the point is that this file is the single declaration site for the
brand palette.

## Conversation styling

### `ConversationStyle` — the per-render snapshot

**[High]** `ConversationStyle` is a `struct` described in-source as "an immutable
snapshot of the core styling state used by CVC for a given load/render cycle"
(`SignalUI/Appearance/ConversationStyle.swift:9-10`). It is `Equatable`
(`:412`) and `CustomDebugStringConvertible` (`:435`) precisely so the conversation
view can cheaply decide whether a restyle/relayout is required.

Key design points:

- It has a lifecycle `Type` — `.initial`, `.placeholder`, `.default`,
  `.messageDetails` — where `.initial` is explicitly *not* renderable
  (`SignalUI/Appearance/ConversationStyle.swift:16-35`), guarded by `isValidStyle`.
- It **captures theme and text state at init** — `isDarkThemeEnabled` and
  `primaryTextColor` are read from `Theme` once in the initializer
  (`SignalUI/Appearance/ConversationStyle.swift:126-127`), and the dynamic-type
  body font point size is captured too (`:146-148`). This is why `Equatable`
  compares `isDarkThemeEnabled` and `dynamicBodyTypePointSize`
  (`:418-419`): a theme change or a Dynamic Type change produces a new, unequal
  style, forcing a re-render.
- It derives **layout geometry** from `viewWidth` and whether the thread is a
  group: gutters, `maxMessageWidth`, `maxMediaMessageWidth` (capped at 350pt),
  and `maxAudioMessageWidth` (capped at 244pt)
  (`SignalUI/Appearance/ConversationStyle.swift:139-181`). Group threads reserve
  width for a 28pt sender avatar (`groupMessageAvatarSizeClass`, `:65`).
- It centralizes **bubble colors**. Incoming bubbles branch on wallpaper +
  reduce-transparency + dark mode, returning a blur effect, a solid fallback, or a
  hard-coded gray (`bubbleChatColorIncoming`,
  `SignalUI/Appearance/ConversationStyle.swift:232-253`); outgoing bubbles use the
  thread's `chatColorValue` (`:263-265`); release-notes bubbles use fixed colors
  (`:267-273`). Text colors are expressed as `ThemedColor`s
  (`bubbleTextColorIncomingThemed` `:277`, `…Outgoing` `:281`) and resolved
  against the captured `isDarkThemeEnabled`.
- It owns **accessibility-sensitive effects**: `bubbleBackgroundBlurEffect`
  returns `nil` when `UIAccessibility.isReduceTransparencyEnabled`
  (`SignalUI/Appearance/ConversationStyle.swift:200-206`), and the incoming-bubble
  color falls back to a solid background in that case (`:237-238`).

**[Medium]** `ConversationStyle` is a core input to the conversation view (CVC);
~49 files under `Signal/` reference it. Its collaborators `TSThread`,
`TSMessage`, `ColorOrGradientSetting/Value`, and `ConversationAvatarView` are
defined outside this directory.

### `BubbleConfiguration` — bubble geometry

**[High]** `BubbleConfiguration` (`SignalUI/Appearance/BubbleConfiguration.swift:14`)
is a pure-geometry value type that "describes the shape of a bubble in chat",
designed to pair with `CVColorOrGradientView` / `CVWallpaperBlurView` (defined
elsewhere). It composes:

- `Corners` with three styles — `.uniform(radius:)`, `.segmented(...)` (different
  radius for a set of "sharp" corners vs. the rest), and `.capsule(maxRadius:)`
  (radius derived from the view's smaller axis)
  (`SignalUI/Appearance/BubbleConfiguration.swift:40-116`). The `segmented`
  factory normalizes degenerate inputs back to `.uniform`
  (`:62-82`).
- `Stroke` (color + width), with the width measured centered on the bubble edge
  (`SignalUI/Appearance/BubbleConfiguration.swift:150-170`).
- `bubblePath(for:)` / `strokePath(for:)` convert the configuration to a
  `UIBezierPath` so callers can drive mask/`CAShapeLayer`s
  (`SignalUI/Appearance/BubbleConfiguration.swift:175-210`).

**[High]** The *color* side of strokes lives in `ConversationStyle`, not here:
`ConversationStyle.bubbleStroke(isDarkThemeEnabled:)` builds a `BubbleConfiguration.Stroke`
with a theme-dependent color and a hairline width
(`SignalUI/Appearance/ConversationStyle.swift:390-394`), and the instance method
only returns a stroke for incoming bubbles that have a wallpaper
(`:399-404`). So `BubbleConfiguration` is the geometry, `ConversationStyle` is the
theming policy.

### `GroupNameColors` — deterministic per-sender colors

**[High]** `GroupNameColors` maps each group member's `Aci` to a stable color so a
sender's name renders in a consistent color across a conversation
(`SignalUI/Appearance/GroupNameColors.swift:15-20`). `forThread(_:)` sorts the
group's full members by ACI, then assigns colors round-robin from a fixed palette
indexed by position (`index % values.count`), capturing `Theme.isDarkThemeEnabled`
once (`SignalUI/Appearance/GroupNameColors.swift:27-42`). Non-group threads get
`defaultColors`, which falls back to `Theme.primaryTextColor`
(`:23-26`). The palette is a hand-ordered list of ~36 light/dark color pairs
"in descending order of contrast with the other values"
(`SignalUI/Appearance/GroupNameColors.swift:57-58`). Determinism comes from the
sorted-ACI + modulo scheme, so the same membership always yields the same mapping.

### `ColorOrGradient` bridge & swatch view

**[High]** `ColorOrGradient+SignalUI.swift` bridges SSK's persisted
`ColorOrGradientSetting` to a renderable `ColorOrGradientValue`
(`.transparent` / `.blur` / `.solidColor` / `.gradient`)
(`SignalUI/Appearance/ColorOrGradient+SignalUI.swift:14-41`). `asValue(themeMode:)`
picks the light or dark variant of themed settings based on a
`ColorOrGradientThemeMode` (`.auto` consults `Theme`, else forced)
(`SignalUI/Appearance/ColorOrGradient+SignalUI.swift:52-96`). It also derives a
UI tint from a bubble color via `asChatUIElementTintColor()`
(`:111-137`).

**[High]** `ColorOrGradientSwatchView` is a `ManualLayoutViewWithLayer` that
renders a setting as a circle or rectangle swatch — used for wallpaper/chat-color
previews (`SignalUI/Appearance/ColorOrGradientSwatchView.swift:20`). It:
subscribes to `.themeDidChange` and reconfigures on change
(`:76-88`); caches a `State` (size + setting) to skip redundant reconfigures
(`:98-105`); and handles solid/blur/gradient rendering, computing gradient
start/end points from an angle in unit space (`:122-210`). The file header
explains why it is distinct from `CVColorOrGradientView` (the CVC-cell variant):
this view can pin gradient bounds to a circle's edge for previews
(`SignalUI/Appearance/ColorOrGradientSwatchView.swift:9-18`). It also builds an
accessibility label naming the swatch's color(s)
(`:50-72`).

## Iconography

### `SignalSymbol` — the bundled glyph font

**[High]** `SignalSymbol` is a `Character`-backed enum where each case is a Private
Use Area code point in Signal's custom symbol font
(`SignalUI/Appearance/SignalSymbols.swift:10`). The font files themselves live
under `SignalUI/Fonts/` (outside this directory). It provides:

- **RTL-aware symbols**: `leave` and `chevronTrailing` pick an LTR/RTL glyph from
  `CurrentAppContext().isRTL`, and `chevronTrailing(for:)` detects the dominant
  language of a user string via `NLLanguageRecognizer` (results cached) to orient
  a trailing chevron correctly for mixed-direction names
  (`SignalUI/Appearance/SignalSymbols.swift:123-162`).
- **Font weights** (`Weight.light/regular/bold/medium/thin`) mapped to concrete
  font names, with static, Dynamic-Type, and clamped-Dynamic-Type variants
  (`SignalUI/Appearance/SignalSymbols.swift:166-240`).
- **Rendering**: `attributedString(...)` overloads produce `NSAttributedString`s
  for UIKit (`:242-310`), and `text(dynamicTypeBaseSize:weight:)` produces a
  SwiftUI `Text` that can be concatenated with other `Text` via `+`
  (`SignalUI/Appearance/SignalSymbols.swift:319-326`).

### `ThemeIcon` — asset-catalog lookup

**[High]** `Theme+Icons.swift` defines the `ThemeIcon` enum (a large catalog of
app icons: settings, donate, profile, group/contact info, context menus, emoji
categories, …) (`SignalUI/Appearance/Theme+Icons.swift:8`) and a `Theme` extension
that resolves each case to an asset-catalog **image name**, then to a `UIImage`
(`iconImage(_:)` / `iconName(_:)`,
`SignalUI/Appearance/Theme+Icons.swift:192-210`). Most names are theme-independent,
but several branch on `isDarkThemeEnabled` to pick a dark/light asset variant
(e.g. `.threadCompact`, `.transfer`, `.register`, `.profilePlaceholder` —
`SignalUI/Appearance/Theme+Icons.swift:371`, `:526`). A missing asset triggers
`owsFailDebug` and returns an empty image rather than crashing
(`:198-202`).

## Themed SwiftUI containers (`Appearance/SwiftUI/`)

**[High]** `SignalList` / `SignalSection` wrap SwiftUI's `List`/`Section` to apply
Signal's grouped-list theming: a `.insetGrouped` list style, iPad horizontal
padding, `Color.Signal.secondaryGroupedBackground` row backgrounds, and
`Color.Signal.groupedBackground` behind the list (hiding the default scroll
background on iOS 16+) (`SignalUI/Appearance/SwiftUI/SignalList.swift:10-57`).
`SignalSection` offers header/footer/content permutations and custom header
styling (`:62-199`). This is the SwiftUI counterpart to the UIKit
`OWSTableViewController2` grouped-list look.

**[High]** `ScrollOffset.swift` is the supporting machinery: a pair of SwiftUI
`PreferenceKey`s and view modifiers that let a scroll container report its scroll
offset. `provideScrollAnchor(correction:)` tags the top-most content element, and
`readScrollOffset()` (applied to the list) reduces those anchors through a
`GeometryReader` into a `ScrollOffsetPreferenceKey`
(`SignalUI/Appearance/SwiftUI/ScrollOffset.swift:40-92`). `SignalList` applies
`readScrollOffset()` internally and `SignalSection` applies `provideScrollAnchor`
to its content/header, so callers get scroll-offset reporting for free. The code
documents a caveat: with lazily loaded `List` content, the reported offset is that
of the highest *currently rendered* item, not the true scroll distance
(`SignalUI/Appearance/SwiftUI/ScrollOffset.swift:85-91`). **[Medium]**

## Key data flows & state

**[High]** The central piece of mutable state is the theme flag inside `Theme`;
everything else in this directory is derived from it, usually snapshotted:

```mermaid
graph TD
    Theme["Theme.isDarkThemeEnabled (+ .themeDidChange)"]
    Theme --> TC["ThemedColor.forCurrentTheme"]
    Theme --> COG["ColorOrGradientSetting.asValue(themeMode:)"]
    Theme --> GNC["GroupNameColors.forThread (snapshot)"]
    Theme --> CS["ConversationStyle.init (snapshot)"]
    Theme --> TI["ThemeIcon → iconName (dark/light variants)"]
    Palette["UIColor.Signal / Color.Signal (trait-aware)"]
    Palette --> CS
    Palette --> SwiftUIList["SignalList / SignalSection"]
    CS --> Bubbles["bubble colors + BubbleConfiguration.Stroke"]
    BubbleConfiguration["BubbleConfiguration (geometry only)"] --> Bubbles
    COG --> Swatch["ColorOrGradientSwatchView"]
    Theme -. .themeDidChange .-> Swatch
```

Two distinct propagation styles coexist:

- **Snapshot-at-render** (`ConversationStyle`, `GroupNameColors`): capture the
  theme flag once; a theme change produces a *new, unequal* value that the
  conversation view diffs to trigger a re-render. **[High]**
- **Live notification** (`ColorOrGradientSwatchView`, and `OWSViewController`
  subclasses described in `Appearance.md`): subscribe to `.themeDidChange` and
  reconfigure in place. **[High]**

## Notable UI / accessibility considerations

- **High contrast**: palette tokens accept dedicated high-contrast variants
  through the custom `UIColor(light:lightHighContrast:dark:darkHighContrast:)`
  initializer (`SignalUI/Appearance/UIColor+Signal.swift:39-57`). **[High]**
- **Reduce Transparency**: `ConversationStyle` suppresses bubble blur and uses a
  solid background when `UIAccessibility.isReduceTransparencyEnabled`
  (`SignalUI/Appearance/ConversationStyle.swift:200-238`). **[High]**
- **Dynamic Type**: `SignalSymbol` fonts come in Dynamic-Type and clamped variants
  (`SignalUI/Appearance/SignalSymbols.swift:200-240`), and `ConversationStyle`
  recomputes text insets/geometry from the current body-font point size and treats
  it as part of equality (`SignalUI/Appearance/ConversationStyle.swift:146-181`,
  `:418`). **[High]**
- **RTL**: `SignalSymbol` orients directional glyphs by app direction *and* by the
  detected direction of user-supplied strings
  (`SignalUI/Appearance/SignalSymbols.swift:123-162`). **[High]**
- **VoiceOver**: `ColorOrGradientSwatchView` is an accessibility element with a
  color-naming label, including a localized two-color label for gradients
  (`SignalUI/Appearance/ColorOrGradientSwatchView.swift:50-72`). **[High]**

## Interactions with the rest of SignalUI and the app

- **Below** (depends on): SSK value types — `ThemedColor`, `ColorOrGradientSetting`,
  `OWSColor`, `TSThread`/`TSMessage`, `Aci` (LibSignalClient) — are imported here
  and resolved/rendered into UIKit/SwiftUI. **[High]**
- **Within SignalUI**: `ConversationStyle`/`BubbleConfiguration`/`GroupNameColors`
  feed the conversation-view (CVC) rendering primitives in
  `SignalUI/ConversationView/` and `SignalUI/Views/` (e.g.
  `ConversationAvatarView`, `CVColorOrGradientView`). `OWSViewController` and
  `OWSTableViewController2` (in `SignalUI/ViewControllers/`) consume `Theme` and
  the palette; `SignalList`/`SignalSection` are the SwiftUI analogue. **[Medium]**
- **Above** (consumers): the main **Signal** app and the **SignalShareExtension**
  import all of the above to render chats, settings, wallpaper/chat-color pickers,
  and themed lists. **[Medium]**

> **Design intent note.** This directory records the *mechanics* of theming and
> styling clearly, but the broader product rationale (e.g. why group-name colors
> are ordered by mutual contrast, or the exact perceptual tuning of the palette)
> is **intent undetermined — no evidence in source** beyond the inline ordering
> comment in `GroupNameColors` (`SignalUI/Appearance/GroupNameColors.swift:57`).
> **[Low]**

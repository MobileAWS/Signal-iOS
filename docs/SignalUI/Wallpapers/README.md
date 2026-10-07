# Chat Wallpapers

Source: `SignalUI/Wallpapers/` (a single file, `Wallpaper+SignalUI.swift`). This
area turns a persisted `Wallpaper` choice into **renderable UIKit views** for the
conversation background, and provides the **frosted-glass blur** that message
bubbles sample so they read against the wallpaper. The *model* (`Wallpaper`), its
*persistence* (`WallpaperStore`), and the *custom-photo storage*
(`WallpaperImageStore`) all live in SignalServiceKit; SignalUI's job is the
view-building and blur-rendering layer on top.

> Confidence labels: **[High]** read in full; **[Medium]** signature/partial read
> or depends on code outside `SignalUI/`; **[Low]** inferred. Citations are
> `File.swift:line`. An uncited claim is a defect.

## Responsibility

**[High]** `Wallpaper+SignalUI.swift` extends SSK's `Wallpaper` enum with static
`viewBuilder(...)` factories and defines the SignalUI-only rendering types
`WallpaperViewBuilder`, `WallpaperView`, `WallpaperBlurState`,
`WallpaperBlurProvider`, and `WallpaperBlurProviderImpl`
(`SignalUI/Wallpapers/Wallpaper+SignalUI.swift:10`, `:71`, `:97`, `:179`, `:200`,
`:206`). The whole file is annotated as main-thread only via
`AssertIsOnMainThread()` in both `viewBuilder` overloads and in
`wallpaperBlurState` (`SignalUI/Wallpapers/Wallpaper+SignalUI.swift:12`, `:54`,
`:225`).

**[Medium]** The underlying `Wallpaper` enum is a `String`-backed `CaseIterable`
with solid-color cases, gradient cases, a `.photo` (custom) case, and a
`.releaseNotes` case (`SignalServiceKit/UISupport/Models/Wallpaper.swift:8-44`).
`defaultWallpapers` excludes `.photo` and `.releaseNotes`
(`SignalServiceKit/UISupport/Models/Wallpaper.swift:46`).

## Key types

### `WallpaperViewBuilder`

**[High]** A small value enum describing *what to build* without yet touching the
view hierarchy (`SignalUI/Wallpapers/Wallpaper+SignalUI.swift:71`). Cases:
`.colorOrGradient(ColorOrGradientSetting, shouldDimInDarkMode:)`,
`.customPhoto(UIImage, shouldDimInDarkMode:)`, and `.releaseNotes`
(`:72-74`). Its `build()` method instantiates the concrete `WallpaperView` on
demand (`:76`). This split exists so callers can resolve the builder inside a DB
read and then construct the actual UIKit view later on the main thread (see data
flow below).

### `WallpaperView`

**[High]** The concrete background view wrapper
(`SignalUI/Wallpapers/Wallpaper+SignalUI.swift:97`). It is built from a private
`Mode` — `.colorView(UIView)`, `.imageView(UIImage)`, or `.releaseNotesView`
(`:98-101`) — and exposes three outputs the conversation background consumes:
`contentView`, `dimmingView`, and `blurProvider`
(`:103`, `:105`, `:107`). `configure(shouldDimInDarkTheme:)` picks the content view
per mode and always attaches a `WallpaperBlurProviderImpl`
(`:129-166`). Notable rendering details, all **[High]**:

- Color/gradient wallpapers render via SSK's `ColorOrGradientSwatchView` in
  `.rectangle` shape mode, with theme mode `.auto` when dimming is allowed else
  `.alwaysLight` (`SignalUI/Wallpapers/Wallpaper+SignalUI.swift:80-90`).
- Photo wallpapers use `.scaleAspectFill`; in dark theme with dimming enabled a
  separate `dimmingView` (`.ows_blackAlpha20`) is created rather than baking the
  dim into the image (there is an inline `TODO: Bake dimming into the image.`)
  (`:141-149`).
- `.releaseNotes` renders the bundled `"official-wallpaper"` asset with
  hard-coded light/dark tint and background colors
  (`:150-162`).
- `asPreviewView()` composes `contentView` + `dimmingView` into a standalone
  preview container used by settings/preview screens
  (`:116-127`).

### `WallpaperBlurState` / `WallpaperBlurProvider` / `WallpaperBlurProviderImpl`

**[High]** `WallpaperBlurProvider` is a one-property protocol exposing an optional
`wallpaperBlurState` (`SignalUI/Wallpapers/Wallpaper+SignalUI.swift:200`).
`WallpaperBlurState` carries the pre-blurred `image`, the `referenceView` used for
coordinate conversion, a monotonically increasing `id` (from a shared-global
`AtomicUInt`), and a private de-bounce `token`
(`:179-185`). `WallpaperBlurToken` is an `Equatable` tuple of content size, dim
flag, and dark-theme flag used purely to decide whether a cached blur can be
reused (`:171-175`).

**[High]** `WallpaperBlurProviderImpl.wallpaperBlurState` is the heart of the blur:
it recomputes a token from the content view's current bounds + theme, returns the
`cachedState` if the token is unchanged, otherwise renders the content view to an
image and applies a Gaussian blur (radius 20) with hand-tuned color overlays,
vibrancy, and exposure chosen to **replicate `UIBlurEffect.Style`
`systemThinMaterialLight` / `systemThinMaterialDark`**
(`SignalUI/Wallpapers/Wallpaper+SignalUI.swift:224-272`). The overlay alpha is
reduced further when `shouldDimInDarkTheme` is set in dark mode (`:238-243`). The
result is cached until the token changes (`:267`).

**[High]** A small `CACornerMask.asUIRectCorner` helper lives at the end of the
file (`SignalUI/Wallpapers/Wallpaper+SignalUI.swift:283`); it maps layer corner
masks to `UIRectCorner` and is used by consumers masking the blur to bubble
corners.

## Resolving a wallpaper (data flow / state / persistence)

**[High]** The main entry point is
`Wallpaper.viewBuilder(for thread:tx:)`
(`SignalUI/Wallpapers/Wallpaper+SignalUI.swift:11`). Within a DB read transaction
it:

1. Short-circuits to `.releaseNotes` for the release-notes thread
   (`:16-18`).
2. Otherwise asks the SSK `WallpaperStore` for the wallpaper that should render
   for this thread (falling back to the global default), returning `nil` if none
   (`:13`, `:19-29`).
3. Resolves the **custom photo** lazily through
   `WallpaperStore.fetchResolvedValue(...)`, loading the per-thread image via
   `wallpaperImageStore.loadWallpaperImage(for:tx:)` or the global image via
   `loadGlobalThreadWallpaper(tx:)` (`:33-44`).
4. Reads the dim-in-dark-mode setting with
   `fetchDimInDarkModeForRendering(for:tx:)` and hands all of it to the second,
   pure `viewBuilder(for:customPhoto:shouldDimInDarkTheme:)` overload
   (`:45`, `:48`).

**[High]** The pure overload maps the resolved `Wallpaper` to a
`WallpaperViewBuilder`: a `.photo` with a non-nil custom image → `.customPhoto`;
anything with an `asColorOrGradientSetting` → `.colorOrGradient`; `.releaseNotes`
→ `.releaseNotes`; otherwise it `owsFailDebug`s and returns `nil`
(`SignalUI/Wallpapers/Wallpaper+SignalUI.swift:48-67`).

**[Medium]** Persistence and change-notification are owned by SSK:
`WallpaperStore` (`SignalServiceKit/UISupport/WallpaperStore.swift:9`) and
`WallpaperImageStoreImpl` (`SignalServiceKit/UISupport/WallpaperImageStoreImpl.swift:9`),
reached here through `DependenciesBridge.shared.wallpaperStore` /
`.wallpaperImageStore` (`SignalUI/Wallpapers/Wallpaper+SignalUI.swift:13`, `:36-38`).

## Interactions with the app

**[Medium]** The conversation view is the primary consumer (code lives in the
`Signal` app target, not SignalUI). `ConversationViewController+Wallpaper`
subscribes to `WallpaperStore.wallpaperDidChangeNotification`, re-resolves the
builder inside a DB read, stores it on `viewState.wallpaperViewBuilder`, and
installs `builder.build()` into the background container
(`Signal/ConversationView/ConversationViewController+Wallpaper.swift:10-35`). A
wallpaper change also re-triggers chat-color recomputation
(`ConversationViewController+Wallpaper.swift:25`).

**[Medium]** `CVBackgroundContainer` holds the live `WallpaperView`, adds its
`contentView` and `dimmingView` at fixed z-positions behind the collection view,
hides the whole thing from VoiceOver so message focus works, and *forwards*
`WallpaperBlurProvider` by returning `wallpaperView?.blurProvider?.wallpaperBlurState`
(`Signal/ConversationView/CVBackgroundContainer.swift:14-81`). Individual bubbles
sample that blur through `CVWallpaperBlurView`, which reads `provider.wallpaperBlurState`,
masks the blurred image to the bubble path, and falls back to a solid
`Theme.backgroundColor` when in preview mode or when
`UIAccessibility.isReduceTransparencyEnabled`
(`Signal/ConversationView/CellViews/CVWallpaperBlurView.swift:93-121`). **[Low]**
This reduce-transparency fallback is the accessibility counterpart to the
VoiceOver hiding in the container.

**[Medium]** Settings/preview screens — `ColorAndWallpaperSettingsViewController`,
`SetWallpaperViewController`, `PreviewWallpaperViewController`,
`ChatColorViewController`, `CustomColorViewController` (all under
`Signal/src/ViewControllers/Wallpapers/`) — build `WallpaperView`s and use
`asPreviewView()` to show choices. (Consumers located via reference search; read
by file/usage, not in full.)

## Notable UI considerations

- **[High]** Dimming is theme-aware and optional: color/gradient wallpapers dim via
  the swatch view's theme mode, while photo wallpapers get an explicit overlay
  `dimmingView` only in dark mode with dimming enabled
  (`SignalUI/Wallpapers/Wallpaper+SignalUI.swift:80-90`, `:141-149`).
- **[High]** The blur is intentionally a hand-rolled replica of system material
  styles rather than a live `UIVisualEffectView`, enabling the per-bubble
  reference-view masking and token-based caching
  (`SignalUI/Wallpapers/Wallpaper+SignalUI.swift:233-272`).
- **[Medium]** Accessibility: wallpaper background is hidden from the accessibility
  tree, and the blur is skipped under Reduce Transparency
  (`Signal/ConversationView/CVBackgroundContainer.swift:39`,
  `Signal/ConversationView/CellViews/CVWallpaperBlurView.swift:93-99`).

> **Open TODOs in source (intent noted, not resolved):** "Bake dimming into the
> image" for photo wallpapers (`SignalUI/Wallpapers/Wallpaper+SignalUI.swift:141`)
> and "Observe provider changes" in the per-bubble blur view
> (`Signal/ConversationView/CellViews/CVWallpaperBlurView.swift:90`). **[High]**

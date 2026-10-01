# Fonts & Format Styles

Sources: `SignalUI/Fonts/` (bundled font files), the font helpers in
`SignalUI/UIKitExtensions/UIFont+*.swift` and `SignalUI/Appearance/SignalSymbols.swift`,
and `SignalUI/FormatStyles/` (Swift `FormatStyle` implementations).

> Confidence: **[High]** read in full; **[Medium]** signature/partial; **[Low]**
> inferred. Citations are `File.swift:line`. An uncited claim is a defect.

## Bundled fonts & registration

**[High]** `SignalUI/Fonts/` ships the custom font resources used by the app:
`Inter-Variable.ttf`, the `SignalSymbols-*.otf` family
(Thin/Light/Regular/Medium/Bold), `fontawesome-webfont.ttf`, `EBGaramond-Regular.ttf`,
`BarlowCondensed-Medium.ttf`, `Parisienne-Regular.ttf`, and `Hatsuishi-UPM800.otf`
(enumerated from the directory listing).

**[High]** These are registered at startup, not referenced from an Info.plist
`UIAppFonts` entry: `SUIEnvironment.setUp(...)` calls `registerCustomFonts()`
(`SignalUI/AppLaunch/SUIEnvironment.swift:47`), which enumerates every `.ttf`/`.otf`
in the SignalUI bundle and registers each with Core Text via
`CTFontManagerRegisterFontsForURL(url, .process, &error)`
(`SignalUI/AppLaunch/SUIEnvironment.swift:64-80`). Registration is process-scoped
(`.process`), so both the app and the share extension register the fonts for their
own process when they bootstrap `SUIEnvironment`.

```mermaid
graph TD
    Boot["SUIEnvironment.setUp(...)"] --> Reg["registerCustomFonts()"]
    Reg -->|bundle.urls(forResourcesWithExtension:)| Files["*.ttf / *.otf in SignalUI/Fonts"]
    Files -->|CTFontManagerRegisterFontsForURL(.process)| CT["Core Text (per-process)"]
    CT --> Use["UIFont(name:…) / SignalSymbol glyphs"]
```

## `UIFont` helpers

**[High]** `UIFont+OWS` adds Signal's font accessors
(`SignalUI/UIKitExtensions/UIFont+OWS.swift`): `awesomeFont(ofSize:)` resolves the
registered `"FontAwesome"` font (`SignalUI/UIKitExtensions/UIFont+OWS.swift:14`),
and the Dynamic Type helpers like `dynamicTypeFont(ofStandardSize:weight:)`
(`SignalUI/UIKitExtensions/UIFont+OWS.swift:31`) and the named
`dynamicTypeBody`/`dynamicTypeHeadline`/… accessors
(`SignalUI/UIKitExtensions/UIFont+OWS.swift:54-66`) scale against the user's content
size category. `UIFont+TextStyle` maps text styles to the bundled `Inter`
variants (e.g. `"Inter-Regular_Medium"`
`SignalUI/UIKitExtensions/UIFont+TextStyle.swift:17`). **[Medium]**

**[High]** `SignalSymbol` is a `Character`-backed enum
(`SignalUI/Appearance/SignalSymbols.swift:10`) that selects a weight-specific
bundled symbol font — the `SignalSymbols-{Light,Regular,Medium,Bold}` faces
(`SignalUI/Appearance/SignalSymbols.swift:176-182`). This is Signal's own icon font.

## Format styles (`SignalUI/FormatStyles/`)

**[High]** `OWSByteCountFormatStyle` is a Swift `FormatStyle` for human-readable
byte counts (`SignalUI/FormatStyles/OWSByteCountFormatStyle.swift:6`). `format(_:)`
uses a `ByteCountFormatter` configured with `.useAll` units and
`countStyle = .decimal`, with optional zero-padding of fraction digits to keep a
stable string length (`SignalUI/FormatStyles/OWSByteCountFormatStyle.swift:28-57`).
A `.owsByteCount(...)` convenience is exposed via a `FormatStyle` extension
(`SignalUI/FormatStyles/OWSByteCountFormatStyle.swift:59`).

**[High]** It supports a `fudgeBase2ToBase10` option implemented by
`OWSBase2ByteCountFudger` (`SignalUI/FormatStyles/OWSByteCountFormatStyle.swift:72`),
which converts base-2 multiples (e.g. 100 GiB) to their roughly-equivalent base-10
value (100 GB). The documented use case is the Backups `storageAllowanceBytes`
value, which the server configures as a GiB value
(`SignalUI/FormatStyles/OWSByteCountFormatStyle.swift:89` and the comment at
`:84-88`).

**[High]** `OWSPercentFormatStyle` is the sibling percent `FormatStyle`
(`SignalUI/FormatStyles/OWSPercentFormatStyle.swift:6`).

**[High]** Both are covered by tests in the same directory
(`SignalUI/FormatStyles/OWSByteCountFormatStyleTest.swift:10`), confirming the
format-style behavior is intentionally pinned.

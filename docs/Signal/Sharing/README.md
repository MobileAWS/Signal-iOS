# `Signal/Sharing/` — System Share Sheet & Photo-Library Saving

Documentation of the **`Signal/Sharing/` subsystem of the first-party `Signal/` app
target**. This subsystem is a thin set of helpers that let the rest of the app hand
content — attachments, files, URLs, text, images — to the iOS system share sheet
(`UIActivityViewController`) and to the Photos library (`PHPhotoLibrary`).

Like the parent docs, this treats `SignalServiceKit/` and `SignalUI/` as
**boundaries**: it documents *how this subsystem uses them*, not their internals. For
the app-target overview, see [../README.md](../README.md).

## Scope

In scope (every file under `Signal/Sharing/`):

- `AttachmentSharing.swift` — the public entry point for presenting the system share
  sheet, plus `ShareableAttachment` (a `UIActivityItemSource`) and the
  `[ReferencedAttachmentStream] → [ShareableAttachment]` adapter.
- `AttachmentSaving.swift` — saving media attachments / images to the Photos library,
  with an opt-out confirmation action sheet.
- `ShareActivityUtil.swift` — a lower-level `UIActivityViewController` presenter that
  works around an iOS presentation bug via an invisible wrapper view controller.

Directory listing produced by listing the working tree in this session **[High]**.

## Conventions

- **Citations** are `path:line` (specific declaration) or `path` (file). Line numbers
  reflect the working tree at authoring time and may drift; relocate via the symbol
  name. Every factual claim about behavior cites a path.
- **Confidence labels** on claims about *purpose/behavior*:
  - **[High]** — directly observed (file read / declaration located in this session).
  - **[Medium]** — inferred from signatures, names, and cross-file convention; not
    every referenced file was read in full.
  - **[Low]** — educated inference from naming alone.
- Any uncited claim is a **defect**. Where the source gives no evidence of design
  intent, the text says **"intent undetermined — no evidence in source."**

## Responsibility

The subsystem has one job: **move already-materialized content out of Signal through
system facilities**. It does not fetch, download, or decrypt-on-demand network content;
it operates on `AttachmentStream`s that are already on disk and on in-memory values
(`URL`, `String`, `UIImage`). Two destinations are supported:

1. The **system share sheet** (`UIActivityViewController`) — `AttachmentSharing` and
   `ShareActivityUtil` **[High]**.
2. The **Photos library** (`PHPhotoLibrary` / `PHAssetCreationRequest`) —
   `AttachmentSaving` **[High]**.

This is strictly an *outbound* path. It is unrelated to the Share **Extension**
(inbound sharing *into* Signal from other apps), which lives in a separate target —
not in this folder **[Medium]** (no Share-Extension code is present under
`Signal/Sharing/`).

## Key types

### `AttachmentSharing` (`Signal/Sharing/AttachmentSharing.swift:9`)

A `public` uninstantiable class (private `init`) exposing overloaded
`showShareUI(for:...)` statics for the shareable value kinds **[High]**:

- `ShareableAttachment` / `[ShareableAttachment]`
  (`AttachmentSharing.swift:15`, `:27`)
- `URL` / `[URL]` (`AttachmentSharing.swift:41`, `:55`)
- `String` (`AttachmentSharing.swift:65`)
- `UIImage` — compiled only under `#if USE_DEBUG_UI` (`AttachmentSharing.swift:83`)

All overloads funnel into `showShareUIForActivityItems(_:sender:from:completion:)`
(`AttachmentSharing.swift:97`), which:

- Runs on the main thread via `DispatchMainThreadSafe` **[High]**.
- Builds a `UIActivityViewController` and logs success/error in its
  `completionWithItemsHandler`, then invokes the caller's `completion` on the main
  thread **[High]**.
- Resolves the presenting controller: the passed `viewController`, else
  `CurrentAppContext().frontmostViewController()`, then walks `presentedViewController`
  to the top of the modal stack (`AttachmentSharing.swift:118`) **[High]**.
- Configures `popoverPresentationController` for iPad from the `sender`: a
  `UIBarButtonItem`, a source `UIView`, or — if neither — a zero-size rect centered at
  the bottom of the presenter's view with no arrow. Passing an unexpected `sender` type
  trips `owsFailDebug` (`AttachmentSharing.swift:123`) **[High]**.

### `ShareableAttachment` (`Signal/Sharing/AttachmentSharing.swift:183`)

An `NSObject` conforming to `UIActivityItemSource` that adapts a decrypted
`AttachmentStream` into something the share sheet can consume **[High]**. Two internal
representations (`PreparedShareType`): `decryptedFileURL(URL)` and `image`
**[High]**.

- `shareType(_:)` (`AttachmentSharing.swift:185`) classifies a stream: WebP and
  non-animated supported image MIME types share as `.image`; everything else as
  `.decryptedFileURL` **[High]**.
- The failable/throwing `init?` writes a **decrypted temp copy** via
  `attachmentStream.makeDecryptedCopy(filename:)` for file/audio/image/video content
  types. For video it additionally checks
  `UIVideoAtPathIsCompatibleWithSavedPhotosAlbum` and returns `nil` if the video can't
  be shared **[High]**.
- `deinit` best-effort deletes the decrypted temp file via
  `OWSFileSystem.deleteFile(url:)` — so a `ShareableAttachment`'s lifetime owns its
  temp file **[High]**.
- `UIActivityItemSource` conformance: the placeholder returns `URL` or an empty
  `UIImage` by type (`AttachmentSharing.swift:266`); the real item
  (`AttachmentSharing.swift:276`) returns the file URL, or lazily calls
  `attachmentStream.decryptedImage()` for images **[High]**.

The in-source `// HACK:` comment explains the image-vs-URL distinction: handing the OS
a `UIImage` (rather than a file path) keeps multi-image saves ordered correctly in the
camera roll; mixed image/video albums still mis-order videos because videos can only be
shared as URLs — **a known, documented limitation** (`AttachmentSharing.swift:231`)
**[High]**.

### `Array<ReferencedAttachmentStream>.asShareableAttachments()` (`Signal/Sharing/AttachmentSharing.swift:154`)

The bridge callers use to turn conversation/media attachments into shareables. It
classifies each element, and **if any element must share as a file URL, forces *all*
of them to `.decryptedFileURL`** ("Once one of them are all file, they all have to be
files."). Throws if any decryption fails and drops (`compactMap`) elements whose
`ShareableAttachment` init returns `nil` **[High]**.

### `AttachmentSaving` (`Signal/Sharing/AttachmentSaving.swift:10`)

An uninstantiable `enum` of statics for saving to the Photos library **[High]**:

- `saveToPhotoLibrary(referencedAttachmentStreams:)` (`AttachmentSaving.swift:16`) —
  only `.image`/`.video` content types are saved; `.audio`/`.file` are silently
  ignored. Each saveable stream is written to a decrypted temp copy and mapped to a
  `PHAssetCreationRequestType.imageTempFile` / `.videoTempFile` **[High]**.
- `saveToPhotoLibrary(image:)` (`AttachmentSaving.swift:54`) — saves an in-memory
  `UIImage` directly **[High]**.
- `_confirmAndSaveToPhotoLibrary(...)` (`AttachmentSaving.swift:58`) — reads the
  `PreferenceStore` flag `shouldShowSaveMediaActionSheet` (default `true`); if set,
  presents an opt-out `ActionSheetController` with Save / Save-and-don't-show-again /
  Cancel, otherwise saves directly **[High]**.
- `_saveToPhotoLibrary(...)` (`AttachmentSaving.swift:113`) — a `@MainActor Task` that
  first awaits `ows_askForMediaLibraryPermissions(for: .addOnly)`, then performs the
  `PHPhotoLibrary` changes, shows a success `ToastController` or an error
  `OWSActionSheets.showErrorAlert`, and best-effort deletes temp files afterward
  **[High]**.

Supporting private types: `PHAssetCreationRequestType` (`AttachmentSaving.swift`, enum
with `tmpFileUrl` accessor) and `PreferenceStore` — a `NewKeyValueStore` wrapper in
collection `"AttachmentSaving"` keyed on `shouldShowSaveMediaActionSheet` **[High]**.

### `ShareActivityUtil` (`Signal/Sharing/ShareActivityUtil.swift:9`)

A standalone `enum` with a single `present(activityItems:from:sourceView:completion:)`
(`ShareActivityUtil.swift:10`). It presents an **invisible wrapper view controller**
(`.overCurrentContext`, clear background) and has *that* present the
`UIActivityViewController`, as a documented workaround for an iOS bug where the activity
sheet would dismiss its parent; the wrapper is dismissed twice on completion to defend
against a hang if the target app crashes mid-share **[High]**. It sets
`excludedActivityTypes` for a fixed list of social/export activities and wires the
popover's `sourceView` **[High]**.

Note it is a **separate, lower-level presenter from `AttachmentSharing`**: it takes a
required `from` controller and `sourceView` rather than resolving the frontmost
controller, and carries the invisible-wrapper hack that `AttachmentSharing` does not.
Why two presenters coexist — intent undetermined; no evidence in source.

## Interactions with the rest of the app

Callers reference `AttachmentSharing.`, `AttachmentSaving.`, and `ShareActivityUtil.`
across ~16 files **[High]** (located via text search this session). Representative
surfaces:

- **Media viewing/gallery**: `MediaPageViewController.swift`,
  `MediaTileViewController.swift` — share/save media the user is viewing **[Medium]**.
- **Conversation view**: `CVItemViewModelImpl.swift`,
  `CVComponentGenericAttachment.swift`,
  `ConversationViewController+BodyTextItems.swift`,
  `LongTextViewController.swift` — share attachments / long text from messages
  **[Medium]**.
- **Stories**: `StoryContextMenuGenerator.swift` — share/save from the story context
  menu (the heaviest single caller) **[Medium]**.
- **Settings & misc**: username-link QR sharing
  (`UsernameLinkPresentQRCodeViewController.swift`,
  `UsernameLinkShareSheetViewController.swift`), group-link QR
  (`GroupLinkQRCodeViewController.swift`, `GroupLinkViewController.swift`), donation
  receipts, data-report export, payments-transfer, proxy settings, and `DebugLogs.swift`
  — share URLs/strings/files **[Medium]**.

Dependencies this subsystem reaches *into* (boundaries):

- **`SignalServiceKit`**: `AttachmentStream` / `ReferencedAttachmentStream`,
  `makeDecryptedCopy`, `decryptedImage`, `MimeType`/`MimeTypeUtil`, `OWSFileSystem`,
  `CurrentAppContext()`, `DependenciesBridge.shared.db`, `NewKeyValueStore`,
  `Logger`, `owsFailDebug`/`owsFail`, `OWSLocalizedString` **[High]**.
- **`SignalUI`**: `ActionSheetController`/`ActionSheetAction`, `OWSActionSheets`,
  `ToastController`, `ows_askForMediaLibraryPermissions` **[High]**
  (imported in `AttachmentSaving.swift`).
- **Apple frameworks**: `UIKit` (`UIActivityViewController`) and `Photos`
  (`PHPhotoLibrary`, `PHAssetCreationRequest`) **[High]**.

## Important data flows & state

**Share flow (attachments):** caller holds `[ReferencedAttachmentStream]` →
`.asShareableAttachments()` decrypts to temp files and/or classifies as images →
`AttachmentSharing.showShareUI(for:)` → `UIActivityViewController` → on completion the
`ShareableAttachment` objects deinit and delete their temp files **[High]**.

**Save flow (Photos):** caller holds `[ReferencedAttachmentStream]` (or a `UIImage`) →
`AttachmentSaving.saveToPhotoLibrary(...)` filters to image/video and writes decrypted
temp copies → optional confirmation action sheet (gated by a persisted preference) →
media-library permission request → `PHPhotoLibrary.performChanges` → success toast or
error alert → temp files deleted **[High]**.

**Persistent state:** exactly one flag — `shouldShowSaveMediaActionSheet` in the
`NewKeyValueStore` collection `"AttachmentSaving"`, defaulting to `true` and cleared to
`false` by "Save and don't show again" (`AttachmentSaving.swift`, `PreferenceStore`)
**[High]**.

**Temp-file lifecycle:** decrypted copies are *always* temporary and best-effort
deleted — by `ShareableAttachment.deinit` on the share path, and explicitly after the
`PHPhotoLibrary` write on the save path. Both call sites comment that the files are
cleared eventually regardless **[High]**.

## Notable UI considerations

- **Main-thread presentation** is enforced on the share path
  (`DispatchMainThreadSafe`) and the save path runs its UI/permission work in a
  `@MainActor Task` **[High]**.
- **iPad popover anchoring**: `AttachmentSharing` requires a sensible `sender`
  (bar-button or view); absent one it falls back to a bottom-center zero-rect anchor
  and `owsFailDebug`s on an unexpected sender type (`AttachmentSharing.swift:123`)
  **[High]**.
- **iOS-bug workarounds**: the invisible wrapper controller in `ShareActivityUtil`
  (double-dismiss on completion) and the image-vs-URL `UIActivityItemSource` hack in
  `ShareableAttachment` are both deliberate, source-commented accommodations of OS
  behavior **[High]**.
- **Permissions & feedback**: saving explicitly requests add-only media-library
  permission and surfaces a success `ToastController` or an error alert; the opt-out
  confirmation sheet is localized and names the Photos app **[High]**.
- **Known limitation**: mixed image+video album shares can mis-order video items,
  acknowledged in-source as unfixable because videos share only as URLs
  (`AttachmentSharing.swift:231`) **[High]**.

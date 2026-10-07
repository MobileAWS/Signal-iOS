# `Signal/Attachments/` — App-target attachment glue

The **first-party app-target** helpers that bridge the UI/compose layer and the
device (pasteboard, iCloud assets, memoji) to `SignalServiceKit`'s attachment
pipeline, plus two app-owned background runners that keep attachment
download/validation work moving. This folder is thin: the heavy attachment
machinery (store, upload/download managers, validation, backfill migration) lives
in `SignalServiceKit/`; see [../../SignalServiceKit/Attachments/](../../SignalServiceKit/Attachments)
for those internals. This doc treats SSK as a **boundary** and documents *how the
app target uses it* **[High]**.

## Scope

All four files under `Signal/Attachments/` **[High]**:

- `PasteboardAttachment.swift` — build send-ready attachments from `UIPasteboard`
  (paste), stickers, and memoji glyphs.
- `SignalAttachmentCloner.swift` — re-derive a `PreviewableAttachment` from an
  already-stored `ReferencedAttachmentStream` (e.g. forwarding).
- `AttachmentDownloadRetryRunner.swift` — app-process observer that wakes queued
  attachment downloads when their retry timers elapse.
- `AttachmentValidationBackfillRunner.swift` — `BGProcessingTask` wrapper that
  revalidates attachments validated by an old validator.

## Conventions

- **Citations** are `path:line` (specific declaration) or `path` (file). Line
  numbers reflect the working tree at authoring time and may drift; relocate via
  the symbol name. Every factual claim about behavior cites a path.
- **Confidence labels**:
  - **[High]** — directly observed (file read / declaration located this session).
  - **[Medium]** — inferred from signatures, names, and cross-file convention;
    the referenced SSK/UI type was not read in full.
  - **[Low]** — educated inference from naming alone.
- Any uncited claim is a defect. Where source gives no design-intent evidence, the
  text says so.

## One-paragraph orientation

Two of these files are **synchronous builders** that convert external media into
`PreviewableAttachment` values the compose UI understands:
`PasteboardAttachment` for pasteboard/sticker/memoji sources
(`Signal/Attachments/PasteboardAttachment.swift:12`) and `SignalAttachmentCloner`
for cloning an existing stored attachment back into a previewable one
(`Signal/Attachments/SignalAttachmentCloner.swift:10`). The other two are
**background runners** wired at launch by `AppLifecycleManager`:
`AttachmentValidationBackfillRunner` registers a `BGProcessingTask`
(`AttachmentValidationBackfillRunner.swift:11`) and `AttachmentDownloadRetryRunner`
observes the download queue in-process (`AttachmentDownloadRetryRunner.swift:10`)
**[High]**.

---

## `PasteboardAttachment`

A stateless `enum` namespace of static helpers
(`PasteboardAttachment.swift:12`) that turns `UIPasteboard` contents into
`[PreviewableAttachment]` for the conversation compose flow **[High]**.

### Query helpers (cheap, synchronous)

- `hasStickerAttachment()` — true if pasteboard item 0 advertises
  `com.apple.sticker` / `com.apple.png-sticker` (`:13-30`) **[High]**.
- `mayHaveAttachments()` — `UIPasteboard.general.numberOfItems > 0` (`:32-34`).
- `hasText()` — decides whether the pasteboard is "textual." It special-cases
  `BodyRangesTextView.pasteboardType` (always text), treats `UTType.url` as text,
  and otherwise returns true only if there is a text UTI **and no** non-text media
  UTI from `SignalAttachment.mediaUTISet` — i.e. "prefer non-text contents"
  (`:36-85`) **[High]**. `ConversationInputTextView` uses the pair
  `mayHaveAttachments() && !hasText()` to decide to paste as attachment
  (`Signal/ConversationView/ConversationInputTextView.swift:151`) **[High]**.

### Builders (async where media work is involved)

- `loadPreviewableAttachments(attachmentLimits:)` — `@MainActor async throws`,
  iterates every pasteboard item (`:97-143`). The first item decides the mode: if
  it is **not** visual media (via `canEverHaveMultipleAttachments`, i.e. visual
  media that is not borderless, `:145-147`), only a single attachment is returned;
  otherwise all visual-media items are collected and non-visual items are dropped
  with a warning (`:120-141`) **[High]**.
- `loadPreviewableAttachment(atIndex:pasteboardUTIs:attachmentLimits:retrySinglePixelImages:)`
  — per-item resolution in priority order image → video → audio → generic
  (`:149-213`). Notable behaviors **[High]**:
  - Prefers PNG over JPEG when both are present (transparency / memoji stickers)
    (`:163-166`).
  - Works around a known iOS pasteboard bug that yields a single green pixel: on a
    1×1 image it sleeps 50 ms and refetches **once** (`:177-187`).
  - Images go through `PreviewableAttachment.imageAttachment(…, canBeBorderless: true)`;
    videos through `compressVideoAsMp4` (whose errors are currently swallowed —
    marked `[15M] TODO`, `:195-197`); audio/generic through their respective
    builders **[High]** (builder internals are a SignalUI boundary **[Medium]**).
- `loadPreviewableStickerAttachment()` — synchronous (must finish before `paste()`
  returns and clears the pasteboard); forces `isBorderless` and `owsFailDebug`s if
  the data was not actually a sticker (`:215-240`) **[High]**.
- `loadPreviewableMemojiAttachment(fromMemojiGlyph:)` — writes an
  `OWSAdaptiveImageGlyph`'s content to a temp `DataSourcePath` and builds a
  borderless image attachment (`:242-254`) **[High]**.

Private plumbing: `filterDynamicUTITypes` strips `dyn`-prefixed UTIs because the
attachment pipeline needs standard UTIs to map UTI↔MIME↔extension (`:87-91`);
`buildDataSource` / `dataForPasteboardItem` materialize pasteboard bytes to a temp
file (`:256-278`) **[High]**.

### App interactions

- `ConversationViewController+Delegates.swift` drives paste: stickers take the
  synchronous path, everything else runs inside a `ModalActivityIndicatorViewController`
  with the `ATTACHMENT_PASTING` string, then forwards to `didPasteAttachments`
  (`Signal/ConversationView/ConversationViewController+Delegates.swift:230-271`) **[High]**.
- `ConversationViewController+BodyRangesTextViewDelegate.swift` routes pasted
  memoji through `loadPreviewableMemojiAttachment` (`:43-45`) **[High]**.
- All call sites obtain limits via `OutgoingAttachmentLimits.currentLimits()`
  (SSK/UI type; boundary **[Medium]**).

## `SignalAttachmentCloner`

Stateless `enum` with one method,
`cloneAsSignalAttachment(attachment:attachmentLimits:)` (`SignalAttachmentCloner.swift:10-39`)
**[High]**. Given a `ReferencedAttachmentStream` already in the DB, it:

1. Resolves `dataUTI` from the stream's MIME type via `MimeTypeUtil`
   (throws `OWSAssertionError` if missing, `:15-17`).
2. Calls `attachmentStream.makeDecryptedCopy(filename:)` to get a plaintext temp
   file, wrapped in an **owned** `DataSourcePath` (`:19-24`) **[High]**.
3. Switches on `reference.renderingFlag` to pick the right builder and post-flags:
   `.default`/`.shouldLoop` → `buildAttachment` (looping sets `isLoopingVideo`),
   `.voiceMessage` → `voiceMessageAttachment`, `.borderless` → `imageAttachment`
   with `isBorderless = true` (`:25-37`) **[High]**.

Intended for re-sending/forwarding stored media (name + `ReferencedAttachmentStream`
input); exact call sites not enumerated this session **[Medium]**.

## `AttachmentValidationBackfillRunner`

`class … : BGProcessingTaskRunner`
(`AttachmentValidationBackfillRunner.swift:11`) — the app-target
`BGProcessingTask` wrapper for revalidating attachments that an older validator
version produced **[High]**. It holds `SDSDatabaseStorage`, an
`AttachmentValidationBackfillStore`, and a lazy `AttachmentValidationBackfillMigrator`
factory (`:13-27`) **[High]**.

- Conforms to the `BGProcessingTaskRunner` protocol
  (`Signal/Storage/BGProcessingTaskRunner.swift:22`) **[High]**:
  `taskIdentifier = "AttachmentValidationBackfillMigrator"`,
  `requiresNetworkConnectivity = false`, `requiresExternalPower = false`
  (`:31-35`).
- `run()` drives `runInBatches` calling `migrator().runNextBatch()` per batch
  (`:37-42`) **[High]**.
- `startCondition()` reads the store: `.asSoonAsPossible` if `needsToRun(tx:)`,
  else `.never` (`:44-52`) **[High]**.

**Wiring:** constructed in `AppLifecycleManager.didFinishLaunching` with
`migrator: { DependenciesBridge.shared.attachmentValidationBackfillMigrator }`, then
`registerBGProcessingTask(appReadiness:)` is called **synchronously** during launch
(`Signal/AppLaunch/AppLifecycleManager.swift:275-281`). The comment there notes
BGProcessingTask handlers must register synchronously in `didFinishLaunching` and
that iOS allows at most 10 registered tasks **[High]**.

## `AttachmentDownloadRetryRunner`

`public class` (`AttachmentDownloadRetryRunner.swift:10`) — an in-process engine
that re-triggers queued attachment downloads whose retry timers have elapsed. It is
a singleton: `AttachmentDownloadRetryRunner.shared` wires
`DependenciesBridge.shared.attachmentDownloadManager`,
`…attachmentDownloadStore`, and `SSKEnvironment.shared.databaseStorageRef`
(`:30-34`) **[High]**.

Structure (**[High]**):

- An inner `actor Runner` serializes work via `runIfNotRunning()` and a private
  `isRunning` flag (`:62-115`). `run()` reads `nextRetryTimestamp(tx:)`, sleeps
  until that timestamp if it is in the future, then in a write tx calls
  `updateRetryableDownloads(tx:)` and `attachmentDownloadManager.beginDownloadingIfNecessary()`,
  then recurses to wait for the next timestamp (`:80-113`).
- A `TransactionObserver` (`DownloadTableObserver`) watches GRDB for **updates** to
  `QueuedAttachmentDownloadRecord`'s `minRetryTimestamp` column only (inserts/deletes
  are ignored because downloads never *start* in the retry state)
  (`:119-170`). It records `shouldRunOnNextCommit` on `databaseDidChange` and only
  kicks the runner on `databaseDidCommit`, so work happens after the write is durable
  (`:152-168`) **[High]**.
- `beginObserving()` registers the observer on the GRDB pool, kicks the runner once,
  and subscribes to `.OWSApplicationWillEnterForeground`; on foreground it calls
  `beginDownloadingIfNecessary()` and re-runs the retry loop (`:37-60`) **[High]**.

**Wiring:** `AttachmentDownloadRetryRunner.shared.beginObserving()` is scheduled via
`appReadiness.runNowOrWhenMainAppDidBecomeReadyAsync` in `AppLifecycleManager`
(`Signal/AppLaunch/AppLifecycleManager.swift:772-774`) **[High]**.

## Data flows / state

- **Paste/clone → compose:** external bytes (`UIPasteboard`, memoji glyph) or a
  stored `ReferencedAttachmentStream` → temp `DataSourcePath` → `PreviewableAttachment`
  → conversation input toolbar / forward flow (`PasteboardAttachment.swift:97`,
  `SignalAttachmentCloner.swift:10`). State is transient temp files only **[High]**.
- **Download retry loop:** persistent state lives in SSK's
  `QueuedAttachmentDownloadRecord` table; this runner holds only the actor's
  `isRunning` flag plus the observer's `shouldRunOnNextCommit`
  (`AttachmentDownloadRetryRunner.swift:92`, `:146`). Triggers are DB commit
  observation and app-foreground notifications **[High]**.
- **Validation backfill:** persistent progress tracked by
  `AttachmentValidationBackfillStore` / migrator in SSK; this runner only reports a
  start condition and drives batches under iOS's BGProcessingTask scheduler
  (`AttachmentValidationBackfillRunner.swift:37-52`) **[High]**.

## UI considerations

- Pasting non-sticker media is async and must not block the main thread, so it runs
  behind `ModalActivityIndicatorViewController` with the localized `ATTACHMENT_PASTING`
  message; sticker pasting is forced synchronous because `UIPasteboard` is cleared
  as soon as `paste()` returns
  (`Signal/ConversationView/ConversationViewController+Delegates.swift:230-271`,
  `PasteboardAttachment.swift:215`) **[High]**.
- Pasteboard image resolution deliberately prefers PNG (transparency) and retries a
  known single-green-pixel iOS bug once, to avoid pasting a blank image
  (`PasteboardAttachment.swift:163-187`) **[High]**.
- Multi-item paste collapses to one attachment when the first item is non-visual
  media, matching the single-non-media send constraint
  (`PasteboardAttachment.swift:120-147`) **[High]**.
- Rendering flags (`voiceMessage`, `borderless`, `shouldLoop`) are preserved when
  cloning so forwarded media keeps its presentation
  (`SignalAttachmentCloner.swift:25-37`) **[High]**.
- The background runners have no UI; they operate off app-readiness/foreground
  notifications and BGProcessingTask scheduling **[High]**.

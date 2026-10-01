# SharingThreadPickerProgressSheet.swift

Source: [`SignalShareExtension/SharingThreadPickerProgressSheet.swift`](../../SignalShareExtension/SharingThreadPickerProgressSheet.swift)

## Purpose

**[High]** The modal action sheet that shows upload/send progress while the share
extension is sending. It is created and driven by
[`SharingThreadPickerViewController`](SharingThreadPickerViewController.md) and listens to
attachment-upload progress notifications to render an aggregate progress bar.

## Type

### `SharingThreadPickerProgressSheet` (`public class: ActionSheetController`)

Stored state **[High]**:
- `attachmentIds: [Attachment.IDType]` — the attachments to display progress for.
- `progressPerAttachment: [Attachment.IDType: Float]` — progress for *all* attachments
  seen via notifications; filtered down to `attachmentIds` for display (per the field
  comment).

### `init(attachmentIds:delegate:)`

**[High]** Side-effects:
- Builds subviews (`setupSubviews()`).
- Adds a cancel action (`CommonStrings.dismissButton`) whose handler calls
  `delegate?.shareViewWasCancelled()` on the weakly-held [`ShareViewDelegate`](ShareViewDelegate.md).
- Registers for `Upload.Constants.attachmentUploadProgressNotification` →
  `handleAttachmentProgressNotification(_:)`.

### `deinit`

**[High]** `NotificationCenter.default.removeObserver(self)`.

## API

- `updateSendingAttachmentIds(_:)` — replaces `attachmentIds` and re-renders. Called by the
  picker once the real prepared attachment ids are known. **[High]**

## UI

**[High]** Lazy `headerWithProgress` containing a `progressLabel`
(`SHARE_EXTENSION_SENDING_IN_PROGRESS_TITLE`) and a `UIProgressView`, installed as the
sheet's `customHeader` in `setupSubviews()`.

## Rendering

### `renderProgress()` — [`:89`](../../SignalShareExtension/SharingThreadPickerProgressSheet.swift)

**[High]**
- If `attachmentIds` is empty → label shows generic `MESSAGE_STATUS_SENDING`.
- Otherwise computes the **average** of per-attachment progress (attachments upload in
  parallel) for the bar, counts attachments at 100% to approximate "X of Y", and formats
  the label with `SHARE_EXTENSION_SENDING_IN_PROGRESS_FORMAT`
  (`min(totalCompleted + 1, count)` of `count`).

## Notifications

### `handleAttachmentProgressNotification(_:)` — [`:124`](../../SignalShareExtension/SharingThreadPickerProgressSheet.swift)

**[High]** Reads `Upload.Constants.uploadAttachmentIDKey` and
`Upload.Constants.uploadProgressKey` from `userInfo` (`owsFailDebug` if either is missing),
stores the value in `progressPerAttachment`, and re-renders.

## Side-effects / cross-process notes

- **[Medium]** Progress is sourced from the shared upload pipeline via
  `Upload.Constants.attachmentUploadProgressNotification`; all messages sharing the same
  attachment report the same progress, which is why displaying the first is safe (noted in
  the code).
- **[High]** Cancel routes through the delegate, ultimately causing
  `dismissAndCompleteExtension`/`exit(0)` (see [ShareViewController.md](ShareViewController.md)).

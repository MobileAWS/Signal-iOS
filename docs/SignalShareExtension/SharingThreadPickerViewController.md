# SharingThreadPickerViewController.swift

Source: [`SignalShareExtension/SharingThreadPickerViewController.swift`](../../SignalShareExtension/SharingThreadPickerViewController.swift)

## Purpose

**[High]** The recipient picker **and the send "brain"** of the share extension. It
subclasses the shared `ConversationPickerViewController`, drives attachment approval,
builds and enqueues outgoing messages (direct messages and stories), shows send progress,
and handles send failures. It reports lifecycle outcomes back to `ShareViewController`
through a weak [`ShareViewDelegate`](ShareViewDelegate.md).

## Type

### `SharingThreadPickerViewController` (`class: ConversationPickerViewController`)

Key stored state **[High]**:
- `weak var shareViewDelegate: ShareViewDelegate?` — callbacks for will-send / completed /
  cancelled / failed.
- `sendProgressSheet: SharingThreadPickerProgressSheet?` — the active progress sheet.
- `areAttachmentStoriesCompatPrecheck: Bool` — fast pre-check; if true, stories
  destinations are shown "forever" and incompatibility is surfaced only at send time.
- `attachmentLimits: OutgoingAttachmentLimits`.
- `typedItems: [TypedItem]` — `didSet` precondition: at most 1 item unless all are visual
  media; updates stories state + approval mode.
- `mentionCandidates: [Aci]` — group mention candidates.

### `ApprovedSend` (nested `enum`)

**[High]** `text(messageBody:, linkPreview:)`, `contact(contactShare:)`,
`other(attachments:, messageBody:)`.

### `SendFailure` (nested `struct`)

**[High]** `{ outgoingMessages: [PreparedOutgoingMessage], error: Error }`.

## Approval

- `approve()` — [`:107`](../../SignalShareExtension/SharingThreadPickerViewController.swift) —
  builds the approval VC (no cancel button) and pushes it; on throw →
  `shareViewDelegate?.shareViewFailed(error:)`. **[High]**
- `buildApprovalViewController(for thread:)` — [`:116`](../../SignalShareExtension/SharingThreadPickerViewController.swift) —
  used for a pre-selected thread; forces view load, adds the conversation to selection,
  then builds with a cancel button. Throws if the conversation is missing. **[High]**
- `buildApprovalViewController(withCancelButton:)` — [`:126`](../../SignalShareExtension/SharingThreadPickerViewController.swift) —
  switches on the first `TypedItem`:
  - `.text` → `TextApprovalViewController`
  - `.contact` → parses vCard (`SystemContact.parseVCardData`), builds `ContactShareDraft`
    in a DB read, → `ContactShareViewController`
  - `.other` → maps items to `AttachmentApprovalItem` (all remaining must be visual media
    per the `typedItems` precondition) → `AttachmentApprovalViewController`, inserting
    `.disallowViewOnce` when a story destination is selected. **[High]**

## Sending flow

### `send(_:)` — [`:196`](../../SignalShareExtension/SharingThreadPickerViewController.swift)

**[High]** Entry point from every approval delegate. Side-effects:
1. Presents an empty progress sheet (`presentOrUpdateSendProgressSheet(attachmentIds: [])`).
2. Calls `shareViewDelegate?.shareViewWillSend()` (which requests the identified chat
   connection — see [ShareViewController.md](ShareViewController.md)).
3. Runs `Task { tryToSend(...) }`; on `nil` → dismiss sheet + `shareViewWasCompleted()`;
   on `.some(failure)` → dismiss sheet then `showSendFailure(failure)`.

### `tryToSend(selectedConversations:approvedSend:)` — [`:221`](../../SignalShareExtension/SharingThreadPickerViewController.swift)

**[High]** Branches by `ApprovedSend`:
- **text** — guards non-empty body (else `SendFailure`); optionally builds a
  `LinkPreviewDataSource`; calls `sendToOutgoingMessageThreads(...)` with an
  `UnpreparedOutgoingMessage.build` message block and an `enqueueStory` closure that posts
  a text story via `StorySharing.enqueueTextStory`.
- **contact** — validates/prepares the share
  (`contactShareManager.validateAndPrepare`); on throw → `SendFailure`. Sends with default
  disappearing-message timer; `enqueueStory` returns `[]` (contacts aren't sent to stories).
- **other / media** — `AttachmentMultisend.enqueueApprovedMedia(...)` (which also adds
  threads to the profile whitelist); on throw → `SendFailure`. Updates the progress sheet
  with the prepared messages, then awaits all `sendPromise`s in a throwing task group;
  on throw → `SendFailure` carrying the prepared messages.

### `sendToOutgoingMessageThreads(...)` — [`:357`](../../SignalShareExtension/SharingThreadPickerViewController.swift)

**[High]** Shared path for text and contact sends:
1. Filters conversations to `.message` destinations and
   `AttachmentMultisend.prepareDestinations(...)`.
2. In an `awaitableWrite`: builds prepared messages via the supplied `messageBlock`,
   whitelists each destination thread
   (`ThreadUtil.addThreadToProfileWhitelistIfEmptyOrPendingRequest`, setting a default
   timer), and enqueues send promises (`ThreadUtil.enqueueMessagePromise`).
3. Runs `enqueueStory(...)` for story recipients.
4. Presents/updates the progress sheet, then awaits all promises in a task group.
   Any throw returns a `SendFailure` (DB/prepare errors carry no messages; send errors
   carry the prepared messages).

### Progress sheet management

- `presentOrUpdateSendProgressSheet(outgoingMessages:)` — reads `attachmentIdsForUpload`
  in a DB read then delegates to the id-based variant. **[High]**
- `presentOrUpdateSendProgressSheet(attachmentIds:)` — updates the existing sheet or
  creates a [`SharingThreadPickerProgressSheet`](SharingThreadPickerProgressSheet.md) and
  presents it on the navigation controller. **[High]**
- `dismissSendProgressSheet(_:)` — dismisses and clears the stored sheet. **[High]**

## Failure handling

### `showSendFailure(_:)` — [`:435`](../../SignalShareExtension/SharingThreadPickerViewController.swift)

**[High]** Builds a cancel action that, in a DB write, marks all `failure.outgoingMessages`
as `updateWithAllSendingRecipientsMarkedAsFailed` and then calls `shareViewWasCancelled()`.
Three error branches:
- **`ClockSkewError`** — a message explaining the device clock is wrong; a single dismiss
  button (no retry).
- **`UntrustedIdentityError`** — shows the recipient display name, offers *Confirm* which,
  in a DB write, moves verification to `.implicit(isAcknowledged: true)` using the captured
  identity key, then `resendMessages(...)`.
- **default** — generic failure title with a *Retry* action calling `resendMessages(...)`.

### `resendMessages(_:)` — [`:537`](../../SignalShareExtension/SharingThreadPickerViewController.swift)

**[High]** Asserts non-empty; in a DB write enqueues each message on
`messageSenderJobQueue`; shows the progress sheet; on all-fulfilled → dismiss +
`shareViewWasCompleted()`; on error → dismiss then `showSendFailure(...)` again.

## Delegate conformances

**[High]**
- `ConversationPickerDelegate` — selection change updates mention candidates; completing
  selection calls `approve()`; cancel calls `shareViewWasCancelled()`; approval mode is
  `.loading` until `typedItems` is populated.
- `TextApprovalViewControllerDelegate` — approve → `send(.text(...))`; cancel →
  `shareViewWasCancelled()`.
- `ContactShareViewControllerDelegate` — approve → `send(.contact(...))`; cancel →
  `shareViewWasCancelled()`.
- `AttachmentApprovalViewControllerDelegate` — approve → `send(.other(...))`; cancel →
  `shareViewWasCancelled()`; "add more" is an `owsFailDebug` (not supported for shares).
- `AttachmentApprovalViewControllerDataSource` — supplies recipient names and group
  mention candidates.

## Side-effects / cross-process notes

- **[High]** Performs DB reads/writes against the shared app-group database
  (`SSKEnvironment.shared.databaseStorageRef`) — the same store the main app uses.
- **[High]** Enqueues real outgoing messages / stories via `AttachmentMultisend`,
  `ThreadUtil`, and `messageSenderJobQueue`; these are shared sending machinery.
- **[High]** Mutates identity verification state when the user confirms a safety-number
  change.
- **[Medium]** Stories compatibility is only prechecked; the real check happens at send,
  consistent with the field comment on `areAttachmentStoriesCompatPrecheck`.

# `Signal/Spam/` — App-level Spam Reporting

App-target (UI/app-level) helpers for **reporting a conversation as spam** from a
message request or conversation settings. This is distinct from
[`../../SignalServiceKit/Spam`](../../SignalServiceKit) (the service-layer spam
subsystem such as `SpamChallengeResolver`): the files here are thin UI/plumbing
glue that build the "report spam" action sheet, assemble the set of messages to
report, and submit the report over the network.

> Confidence labels used throughout:
> **[High]** — directly observed (file read / declaration located in this session).
> **[Medium]** — inferred from signatures, names, and cross-file convention; the
> referenced file was not read in full.
> **[Low]** — educated inference from naming alone.
> Citations are `path:line` (declaration) or `path` (file). Line numbers reflect the
> working tree at authoring time and may drift; relocate via the symbol name.

## Documents in this folder

Mirrors the folder's file structure (one row per first-party source file):

| Source file | Primary type |
| --- | --- |
| `Signal/Spam/SpamReportingUIUtils.swift` | `enum ReportSpamUIUtils` |
| `Signal/Spam/SpamReport.swift` | `struct SpamReport` |

There are no other files in `Signal/Spam/` — no tests, mocks, generated protobufs,
or asset catalogs **[High]** (folder listing performed this session).

## Responsibility

`Signal/Spam/` owns two concerns **[High]**:

1. **Presentation** — building the confirmation action sheet the user sees when they
   choose "Report Spam" / "Block & Report Spam", and the success toast text
   (`ReportSpamUIUtils`, `Signal/Spam/SpamReportingUIUtils.swift:10`).
2. **Report assembly + submission** — selecting which recent server-GUID'd messages
   to report, recording an in-conversation info message, and POSTing the report
   requests (`ReportSpamUIUtils.insertSpamReportMessage`,
   `Signal/Spam/SpamReportingUIUtils.swift:85`; `SpamReport.submit`,
   `Signal/Spam/SpamReport.swift:15`).

Notably, this folder does **not** contain the spam-challenge/captcha UI — see
[Relationship to the spam challenge / captcha UI](#relationship-to-the-spam-challenge--captcha-ui)
below.

## Key types

### `ReportSpamUIUtils` (`SpamReportingUIUtils.swift`)

A stateless `enum` namespace of `static` helpers **[High]**.

- `showReportSpamActionSheet(from:forThread:isBlocked:onSuccess:)`
  (`Signal/Spam/SpamReportingUIUtils.swift:11`) — builds the action sheet via
  `createReportSpamActionSheet`, wires each action to
  `MessageRequestDecliner.declineMessageRequest(inThread:responseType:)` plus the
  caller's `onSuccess`, and presents it on the given view controller **[High]**.
- `createReportSpamActionSheet(forThread:isBlocked:declineMessageRequest:)`
  (`Signal/Spam/SpamReportingUIUtils.swift:28`) — pure action-sheet builder that
  returns an `ActionSheetController`; takes a `declineMessageRequest` closure so the
  decline action is injected rather than hard-wired **[High]**. It always offers a
  **Report Spam** action (`.spam`) and, only when the thread is not already blocked
  and not a terminated group, a **Block & Report Spam** action (`.blockAndSpam`),
  plus Cancel (`Signal/Spam/SpamReportingUIUtils.swift:54`) **[High]**. All titles
  come from `MESSAGE_REQUEST_*` localized strings **[High]**.
- `successfulReportText(didBlock:)` (`Signal/Spam/SpamReportingUIUtils.swift:71`) —
  returns the localized "spam reported" vs. "spam reported and blocked" string
  **[High]**.
- `insertSpamReportMessage(in:tx:)` (`Signal/Spam/SpamReportingUIUtils.swift:85`) —
  the report-assembly brain; see data flow below. Returns a `SpamReport?` **[High]**.

### `SpamReport` (`SpamReport.swift`)

A value type describing one report payload: `aci`, `serverGuids: Set<String>`, and an
optional `reportingToken` (`Signal/Spam/SpamReport.swift:10`) **[High]**.

- `submit(using networkManager:)` (`Signal/Spam/SpamReport.swift:15`) — fans the
  report out concurrently with a `withThrowingTaskGroup`, building one request per
  server GUID via `OWSRequestFactory.reportSpam(from:withServerGuid:reportingToken:)`
  and awaiting all of them (`Signal/Spam/SpamReport.swift:17`) **[High]**. It logs
  before/after and whether a reporting token was present **[High]**.

## Data flow: reporting a conversation as spam

```mermaid
flowchart TD
    SETTINGS["ConversationSettingsViewController.didTapReportSpam()<br/>(ThreadSettings)"] --> SHOW
    MREQ["ConversationViewController+MessageRequest<br/>createReportThreadActionSheet()"] --> CREATE

    SHOW["ReportSpamUIUtils.showReportSpamActionSheet"] --> CREATE["createReportSpamActionSheet"]
    CREATE -->|user taps Report / Block&Report| DECLINE["MessageRequestDecliner.declineMessageRequest(inThread:responseType:)"]

    DECLINE -->|databaseStorage.write| EFFECTS{responseType flags}
    EFFECTS -->|shouldBlockThread| BLOCK[blockingManager.addBlockedThread]
    EFFECTS -->|always| SYNC[syncManager.sendMessageRequestResponseSyncMessage]
    EFFECTS -->|shouldReportSpam| INSERT["ReportSpamUIUtils.insertSpamReportMessage(in:tx:)"]
    EFFECTS -->|shouldDeleteThread| DEL[threadDeletionManager.deleteThreads]

    INSERT -->|TSInfoMessage .reportedSpam| INFO[(info message inserted)]
    INSERT -->|collect up to 3 recent serverGuids| GUIDS[SpamReport?]
    GUIDS -->|Task { try? await }| SUBMIT["SpamReport.submit(using: networkManager)"]
    SUBMIT -->|per GUID| REQ["OWSRequestFactory.reportSpam → networkManager.asyncRequest"]
```

**Entry points (callers outside this folder).** The action sheet is surfaced from two
places in the Signal app **[High]**:

- `ConversationSettingsViewController.didTapReportSpam()`
  (`Signal/src/ViewControllers/ThreadSettings/ConversationSettingsViewController.swift:741`),
  which calls `showReportSpamActionSheet(...)` and shows a success toast via
  `successfulReportText` on success
  (`Signal/src/ViewControllers/ThreadSettings/ConversationSettingsViewController.swift:747`).
- `ConversationViewController+MessageRequest.createReportThreadActionSheet()`
  (`Signal/ConversationView/ConversationViewController+MessageRequest.swift:387`),
  which calls `createReportSpamActionSheet(...)` and injects the VC's own
  `declineMessageRequest(responseType:)`; its success path also uses
  `successfulReportText`
  (`Signal/ConversationView/ConversationViewController+MessageRequest.swift:142`).

**Decline → report.** Both action-sheet actions ultimately invoke
`MessageRequestDecliner.declineMessageRequest(inThread:responseType:)`
(`Signal/ConversationView/MessageRequestDecliner.swift:12`), which runs a single DB
write that may block, sync, leave/delete the thread, and — when
`responseType.shouldReportSpam` — calls `insertSpamReportMessage` and fires the
submission in a detached, best-effort `Task` (`try?`, not awaited)
(`Signal/ConversationView/MessageRequestDecliner.swift:42`) **[High]**.

**Report assembly (`insertSpamReportMessage`).** Within the write transaction
(`Signal/Spam/SpamReportingUIUtils.swift:85`) **[High]**:

- Resolves the reportee `Aci`: for a `TSContactThread`, the contact's `serviceId`; for
  a `TSGroupThread`, the ACI that added the local user via the invite (using
  `tsAccountManager.localIdentifiers` and `groupMembership`); otherwise `owsFailDebug`
  (`Signal/Spam/SpamReportingUIUtils.swift:86`) **[High]**.
- Inserts a `TSInfoMessage(thread:messageType: .reportedSpam)` so the conversation
  shows that spam was reported (`Signal/Spam/SpamReportingUIUtils.swift:106`) **[High]**.
- Collects **at most 3** of the most recent server-GUID'd messages
  (`maxMessagesToReport = 3`, `Signal/Spam/SpamReportingUIUtils.swift:116`): for groups
  it enumerates recent group-update info messages and keeps GUIDs whose updater ACI
  matches the reportee (`enumerateRecentGroupUpdateMessages`); for 1:1 it enumerates
  recent interactions and keeps incoming messages' `serverGuid`s
  (`enumerateInteractionsForConversationView`) **[High]**.
- Looks up a `SpamReportingToken` via
  `SpamReportingTokenRecord.reportingToken(for:database:)`
  (`Signal/Spam/SpamReportingUIUtils.swift:170`) **[High]**.
- Returns `nil` (reporting nothing) if there are no GUIDs or the ACI is missing
  (`Signal/Spam/SpamReportingUIUtils.swift:172`) **[High]**.

**Submission.** `SpamReport.submit` is called with
`SSKEnvironment.shared.networkManagerRef` and is explicitly best-effort — the caller
does not await or surface failures (`Signal/ConversationView/MessageRequestDecliner.swift:45`)
**[High]**. There is a `TODO[SPAM]` in the message-request path noting that for groups
the inviter should be fetched to add to the message
(`Signal/ConversationView/ConversationViewController+MessageRequest.swift:385`) **[High]**.

## Interactions with the rest of the module

- **SignalServiceKit** (boundary): `TSThread`/`TSContactThread`/`TSGroupThread`,
  `TSInfoMessage`, `InteractionFinder`, `SpamReportingTokenRecord`,
  `OutgoingMessageRequestResponseSyncMessage.ResponseType`, `OWSRequestFactory`,
  `NetworkManagerProtocol`, and `DependenciesBridge`/`SSKEnvironment` accessors are all
  consumed here **[High]** (imports and call sites in both source files).
- **SignalUI** (boundary): `ActionSheetController`/`ActionSheetAction` and
  `CommonStrings` are used to build the sheet (`import SignalUI`,
  `Signal/Spam/SpamReportingUIUtils.swift:8`) **[High]**.
- **LibSignalClient**: `Aci` typing for the reportee
  (`Signal/Spam/SpamReportingUIUtils.swift:6`, `Signal/Spam/SpamReport.swift:7`) **[High]**.
- **App callers**: `ConversationSettingsViewController`, `ConversationViewController`
  (+MessageRequest extension), and `MessageRequestDecliner` as described above **[High]**.

## Relationship to the spam challenge / captcha UI

The task calls out "spam challenge/captcha UI." That UI is **not** in `Signal/Spam/`
**[High]**. The server-driven rate-limit/spam challenge is handled by
`SpamChallengeResolver` (SignalServiceKit, out of scope here), and the app presents the
captcha itself from elsewhere in the Signal target **[High]**:

- `SignalApp` observes `SpamChallengeResolver.NeedsCaptchaNotification`
  (`Signal/AppLaunch/SignalApp.swift:69`) and, in `@objc spamChallenge()`
  (`Signal/AppLaunch/SignalApp.swift:87`), presents `SpamCaptchaViewController`
  (defined in `SignalUI/ViewControllers/SpamCaptchaViewController.swift` **[High]**)
  on the frontmost VC of `windowManager.captchaWindow`, then records the challenge
  date in `SupportKeyValueStore` **[High]**.
- `AppLifecycleManager` forwards a push `rateLimitChallenge` token to
  `spamChallengeResolverRef.handleIncomingPushChallengeToken`
  (`Signal/AppLaunch/AppLifecycleManager.swift:1842`) **[High]**.
- `GroupCallViewController` observes `SpamChallengeResolver.didCompleteAnyChallenge`
  (`Signal/Calls/UserInterface/GroupCallViewController.swift:327`) and
  `ConversationViewController+CVComponentDelegate` checks
  `spamChallengeResolverRef.isPausingMessages` / `retryPausedMessagesIfReady`
  (`Signal/ConversationView/ConversationViewController+CVComponentDelegate.swift:994`)
  **[High]**.

In short: the **captcha/challenge** path (dedicated window, notification-driven
presentation of `SpamCaptchaViewController`) is a separate flow from the
**report-as-spam** path that `Signal/Spam/` implements. They share the "spam" theme
and both touch `SpamReportingToken`/`SpamChallengeResolver` concepts, but they are
distinct code paths **[High]**.

## Notable UI considerations

- The report confirmation is a **destructive-style action sheet** whose "Block &
  Report Spam" option is conditionally hidden for already-blocked threads and
  terminated groups (`Signal/Spam/SpamReportingUIUtils.swift:54`) **[High]**.
- Report submission is **best-effort and silent**: there is no error UI if the network
  requests fail; the user sees the success toast and the in-conversation
  `.reportedSpam` info message regardless of request outcome
  (`Signal/ConversationView/MessageRequestDecliner.swift:42`,
  `Signal/Spam/SpamReportingUIUtils.swift:106`) **[High]**.
- Only up to **3** recent messages are reported to limit payload size
  (`Signal/Spam/SpamReportingUIUtils.swift:116`) **[High]**.

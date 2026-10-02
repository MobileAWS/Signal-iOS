# Survey of the concrete job queues

This document surveys every concrete queue in `SignalServiceKit/Jobs/`. For the
generic mechanics they build on, see [runner-model.md](runner-model.md),
[job-records.md](job-records.md), [persistence-and-finder.md](persistence-and-finder.md),
and [retry-and-backoff.md](retry-and-backoff.md).

## At a glance

| Queue | File | Engine | Concurrent | maxRetries | Early-retry triggers |
| --- | --- | --- | --- | --- | --- |
| MessageSender | `MessageSenderJobQueue.swift` | **bespoke** | per-thread, 1–2 slots | 110 (2 for PIN change) | reachability + chat-connection |
| SendGiftBadge | `SendGiftBadgeJobQueue.swift` | `JobQueueRunner` | yes | 110 | reachability |
| DonationReceiptCredentialRedemption | `DonationReceiptCredentialRedemptionJobQueue.swift` | `JobQueueRunner` | yes | unbounded transient (SEPA day-scale) | reachability |
| IncomingContactSync | `IncomingContactSyncJobQueue.swift` | `JobQueueRunner` | no (serial) | 4 | reachability |
| CallRecordDeleteAll | `CallRecordDeleteAllJobQueue.swift` | `JobQueueRunner` | no (serial) | 110 | — |
| BulkDeleteInteraction | `BulkDeleteInteractionJobQueue.swift` | `JobQueueRunner` | no (serial) | 110 | — |
| LocalUserLeaveGroup | `LocalUserLeaveGroupJob.swift` | `JobQueueRunner` | no (serial) | 110 | reachability |

<a name="container"></a>
## `SignalMessagingJobQueues` container

A tiny holder constructed with `appReadiness`, `db`, `reachabilityManager`,
owning two queues: `incomingContactSyncJobQueue` and `sendGiftBadgeJobQueue`.
**[High]** (`SignalMessagingJobQueues.swift:8-19`)

> Other queues (MessageSender, Donation, CallRecordDeleteAll,
> BulkDeleteInteraction, LocalUserLeaveGroup) are **not** held here; they are
> constructed/owned elsewhere in the dependency graph — outside this folder, so
> their ownership is **undetermined — no evidence in source** here. **[High]**

<a name="messagesenderjobqueue"></a>
## MessageSenderJobQueue (`MessageSenderJobQueue.swift:33-618`)

Durably enqueues outgoing messages and calls `MessageSender`. **Does not use
`JobQueueRunner`** — it implements its own queueing, per-thread concurrency, and
retry loop. **[High]** (`MessageSenderJobQueue.swift:33`)

### Two-tier retry design (from the file header)

- `MessageSender` retries **server-directed, immediate** errors (e.g. "missing a
  device; add it and retry") — no backoff, part of the normal flow.
- `MessageSenderJobQueue` retries **generic/transient** errors (5xx, no Internet)
  with configurable count + backoff.
- Both only retry `isRetryable` errors. **[High]**
  (`MessageSenderJobQueue.swift:11-29`)

### Enqueue API

`add(message:limitToCurrentProcessLifetime:isHighPriority:transaction:)` and a
`Promise`-returning overload. Internally:

1. `message.updateAllUnsentRecipientsAsSending(tx:)` so the UI updates immediately.
   **[High]**
2. Builds a `MessageSenderJobRecord`; on failure, marks recipients failed and
   rejects the future. **[High]**
3. If `exclusiveToCurrentProcessIdentifier` (a.k.a. `limitToCurrentProcessLifetime`),
   the record is **not** inserted into the DB (in-memory only); otherwise
   `anyInsert`. **[High]**
4. Appends to `pendingJobs` and, on `tx.addSyncCompletion`, calls
   `startPendingJobRecordsIfPossible`. **[High]**

### Loading & readiness

`setUp` (run on `appReadiness`): if `isMainApp`, loads runnable jobs via
`JobRecordFinderImpl<MessageSenderJobRecord>` (invoking `didMarkAsReady` to
re-mark recipients sending), merges with any already-pending in-memory jobs,
registers reachability + chat-connection observers, marks `isLoaded = true`, and
flushes pending jobs. **[High]**

> Comment: the notification observers are intentionally never unregistered — "all
> our JobQueues live forever." **[High]**

### Per-thread, priority-aware concurrency

- Jobs are keyed by `QueueKey(threadId, priority)` where priority ∈ `{high,
  renderableContent, low}` (high = `isHighPriority`; else renderable content vs.
  not). **[High]**
- Within a queue key, normally **one** operation runs. A **second** slot opens
  *only* when exactly one active op uses the media queue, letting a non-media
  message run concurrently so text isn't stuck behind a media upload. The
  doc-comment works through the ABCD example. **[High]**
- `TaskPriority`: high/renderableContent → `.userInitiated`; low → `.medium`.
  **[High]**

### Retry loop (`_runOperation`)

See [retry-and-backoff.md](retry-and-backoff.md#messagesender-richer-triggers) for
the full error→trigger table. Key points:

- `isFatalError` → throw immediately (fails the whole group send). **[High]**
- Prefers to surface a **retryable** error over a non-retryable one. **[High]**
- `retryDelay = max(exponential, min(cap, retryAfter), min(cap, accountChecker))`;
  cap = `14.1 * .minute`. **[High]**
- Waits with `withCooperativeTimeout(seconds: retryDelay)` racing the external
  retry trigger. **[High]**
- `maxRetries` = `110`, or `2` for PIN-change messages (used as a crude timeout).
  **[High]**
- Asserts cancellation is unsupported. **[High]**

### Completion

`runOperation` deletes the record (unless in-memory-only) and marks the message
`updateWithSendSuccess` / `updateWithAllSendingRecipientsMarkedAsFailed`, then
resolves/rejects the future. `waitUntilDone()` + a `Monitor.Condition` on
`State.isDone` let callers await full drain. **[High]**

### Edge cases

- A record that can't be restored into a `PreparedOutgoingMessage` is removed
  (unless in-memory) and its future rejected. **[High]**
- In-memory-only jobs never touch the DB, so they vanish on relaunch (that is the
  point of `limitToCurrentProcessLifetime`). **[High]**

## SendGiftBadge (`SendGiftBadgeJobQueue.swift`)

Standard `JobQueueRunner` queue, `canExecuteJobsConcurrently: true`. **[High]**
(`SendGiftBadgeJobQueue.swift:17-25`)

- `createJob(...)` builds a `SendGiftBadgeJobRecord` from a `PreparedGiftPayment`
  (Stripe or PayPal). **[High]** (`SendGiftBadgeJobQueue.swift:35-82`)
- `addJob(_:tx:)` inserts the record and, on sync completion, adds a persisted job
  with a runner carrying `chargeFuture` + `completionFuture`; returns both
  promises. **[High]** (`SendGiftBadgeJobQueue.swift:84-98`)
- `alreadyHasJob(threadId:)` → raw SQL `EXISTS` guard excluding
  `permanentlyFailed`/`obsolete`. **[High]** (`SendGiftBadgeJobQueue.swift:100-124`)

### Runner steps (`_runJobAttempt`)
1. Decode payment (Stripe vs Braintree); missing fields → thrown error. **[High]**
2. `ensureThatWeCanStillMessageRecipient` (thread exists & not blocked). **[High]**
3. Confirm payment (idempotency key = `jobRecord.uniqueId`); resolve `chargeFuture`.
   **[High]**
4. Request + generate the receipt credential presentation. **[High]**
5. Prepare optional oversize text; enqueue the gift-badge message (and optional
   text message) via `MessageSenderJobQueue`; insert a `DonationReceipt`; remove the
   job record. **[High]**

Retries via the default handler, `maxRetries = 110`. `didFinishJob` resolves
`completionFuture` on success or rejects both futures on failure. **[High]**

## DonationReceiptCredentialRedemption (`DonationReceiptCredentialRedemptionJobQueue.swift`)

The most elaborate queue. `canExecuteJobsConcurrently: true`. **[High]**

- Two save APIs: `saveBoostRedemptionJob(...)` (one-time) and
  `saveSubscriptionRedemptionJob(...)` (recurring). Both insert the record and
  return it; the caller must then call `runRedemptionJob(jobRecord:)`, which adds a
  persisted job with a `CheckedContinuation` runner and `await`s it. **[High]**
- `subscriptionJobExists(subscriberID:)` → raw SQL `EXISTS` guard. **[High]**

### Runner (`_runJobAttempt`)
- Parses the record into a `Configuration` (payment method/processor/type, receipt
  request/context, optional persisted presentation). **[High]**
- Requires registration; snapshots badges "before" for the "new badge" UI. **[High]**
- SEPA guard on restart (see [retry-and-backoff.md](retry-and-backoff.md#donationreceiptcredentialredemption-sepaideal-day-scale-retries)). **[High]**
- Loads badge + amount (cached across attempts on the runner instance). **[High]**
- Uses a persisted receipt-credential presentation if present, else requests one
  and persists the credential via `setReceiptCredential`. **[High]**
- On `ReceiptCredentialRequestError`: persists an error code and maps the code to a
  retry/terminal/success outcome (`.paymentStillProcessing`,
  `.paymentIntentRedeemed`-as-success, failures). **[High]**
- On success: clears request errors (including orphaned recurring-subscription
  errors), records redemption success, inserts a `DonationReceipt`, removes the job.
  **[High]**
- `runJobAttempt` only treats network failures as retryable (asserts so); other
  errors delete the record and finish failed. **[High]**
- `didFinishJob` posts `didSucceed`/`didFail` notifications and resumes the
  continuation. **[High]**

## IncomingContactSync (`IncomingContactSyncJobQueue.swift`)

Serial queue (`canExecuteJobsConcurrently: false`), `maxRetries = 4`. **[High]**

- `add(...)` builds an `IncomingContactSyncJobRecord` from attachment-pointer
  fields, inserts it, and schedules via an ordered `CompletionSerializer`. **[High]**
- `start(appContext:)` restarts existing jobs only in the main app. **[High]**

### Runner (`_runJob`)
1. `jobRecord.downloadInfo` → `.invalid` removes the job (logged `owsFailDebug`);
   `.transient` downloads the attachment via `attachmentDownloadManager`. **[High]**
2. `processContactSync` streams contacts in **batches of 8** to bound memory and
   avoid long write transactions. **[High]**
3. Per contact: merges/creates `SignalRecipient`, marks registered if an ACI is
   present, creates/updates the contact thread, applies disappearing-message timer.
   No identifier at all → throws `OWSAssertionError`. **[High]**
4. If `isComplete`, `pruneContacts(...)` removes `SignalAccount`s not seen in the
   complete sync (fires one identity-change notification). **[High]**
5. Posts `.incomingContactSyncDidComplete` with inserted threads. **[High]**

> The `pruneContacts` doc-comment flags that this linked-device cleanup still
> compensates for the lack of a periodic full StorageService sync, and should only
> be removed once such a sync exists. **[High]**

## CallRecordDeleteAll (`CallRecordDeleteAllJobQueue.swift`)

Serial queue, `maxRetries = 110`, `deletionBatchSize = 500`. Bulk-deletes all
`CallRecord`s before a boundary. **[High]**

- `addJob(sendDeleteAllSyncMessage:deleteAllBefore:tx:)` accepts either a
  `CallRecord` anchor (resolves a conversation id; failure logs + returns) or a raw
  timestamp. **[High]**
- Runner prefers the referenced call's `callBeganTimestamp` over the stored
  timestamp when the anchor still resolves. **[High]**
- Deletes in `TimeGatedBatch`es: thread calls go through `interactionDeleteManager`;
  call-link calls go through `callRecordDeleteManager`; **call links where we are
  admin are skipped** (deleted via Storage Service instead). **[High]**
- Advances the boundary to the earliest deleted timestamp each batch to avoid
  re-fetching skipped admin call links. **[High]**
- If `sendDeleteAllSyncMessage`, sends an `OutgoingCallLogEventSyncMessage`
  (`.cleared`) via `MessageSenderJobQueue`, then removes the job. **[High]**
- No reachability listener wired; `didFinishJob` only logs failures. **[High]**

## BulkDeleteInteraction (`BulkDeleteInteractionJobQueue.swift`)

Serial queue, `maxRetries = 110`, `deletionBatchSize = 500`. Implements
"delete-for-me" bulk deletion up to an anchor message. **[High]**

- `addJob(anchorMessageRowId:isFullThreadDelete:threadUniqueId:tx:)` records the
  anchor plus, for full-thread deletes, the current most-recent row id. **[High]**
- Runner deletes interactions `atOrBefore(anchorMessageRowId)` in `TimeGatedBatch`es,
  suppressing per-interaction thread updates and instead doing one thread update per
  transaction via `addFinalizationBlock`. **[High]**
- After deletion, waits (via `Preconditions`/`NotificationPrecondition`) until
  registration state is settled, to avoid deleting a thread while registration is
  corrupted. **[High]**
- For full-thread deletes it may additionally soft-delete the thread, but **aborts**
  if (a) addressable messages remain, or (b) the most-recent row id is newer than
  the recorded full-thread anchor (something was inserted mid-delete). **[High]**

## LocalUserLeaveGroup (`LocalUserLeaveGroupJob.swift`)

Serial queue, `maxRetries = 110`. Leaves a GV2 group (optionally assigning a
replacement admin), returning promises for the resulting send(s). **[High]**

- `addJob(groupThread:replacementAdminAci:waitForMessageProcessing:isDeletingAccount:tx:)`
  asserts the thread is GV2 (`owsFail` for GV1), inserts the record, and schedules a
  runner carrying `isDeletingAccount` + the `Future<[Promise<Void>]>`. **[High]**
- Runner: optionally waits for message fetching/processing; fetches the GV2 thread
  (missing → `OWSAssertionError`); best-effort refreshes group send endorsements
  (ignores non-network failures); calls `GroupManager.updateGroupV2` with
  `setShouldLeaveGroupDeclineInvite()` and the optional admin role change; removes
  the record; returns the send promises. **[High]**
- `didFinishJob` resolves/rejects the outer future with the send promises. **[High]**

## Common scheduling idioms

- **Ordered scheduling:** several queues use a `CompletionSerializer`
  (`jobSerializer.addOrderedSyncCompletion(tx:)`) so persisted jobs are handed to
  the runner in insertion order after the enclosing write commits. **[High]**
  (IncomingContactSync, CallRecordDeleteAll, BulkDeleteInteraction,
  LocalUserLeaveGroup, and MessageSender's own serializer)
- **Main-app gating:** `start(appContext:)` typically passes
  `appContext.isMainApp` as `shouldRestartExistingJobs`, so extensions don't drain
  the durable queue. **[High]**
- **Future/continuation hand-off:** freshly-scheduled jobs attach an in-memory
  promise/continuation via a custom runner; relaunched jobs get the default
  (no-callback) runner. **[High]** (SendGiftBadge, Donation, LocalUserLeaveGroup)

## Feature flags observed

- `TSConstants.isUsingProductionService` switches the SEPA/iDEAL retry interval
  between **1 day** (prod) and **1 minute** (non-prod) in the Donation queue.
  **[High]** (`DonationReceiptCredentialRedemptionJobQueue.swift` `Constants.sepaRetryInterval`)
- No other compile-time feature flags are referenced in these queues.
  `TESTABLE_BUILD` only toggles extra test-only initializers on some records.
  **[High]**

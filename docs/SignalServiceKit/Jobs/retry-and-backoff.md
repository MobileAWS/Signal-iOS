# Retry & backoff semantics

Primary source: `SignalServiceKit/Jobs/JobQueueRunner.swift` (`JobAttemptResult`),
plus per-queue overrides.

## Two layers of retry

The generic engine distinguishes a single **attempt** from the **job** as a whole:

- `runJobAttempt` returns a `JobAttemptResult` for one attempt.
- `_runJob` loops attempts until a terminal `JobResult`.

See the sequence diagram in [runner-model.md](runner-model.md#the-attempt-loop).

```swift
public enum JobAttemptResult<Success> {
    case finished(Result<Success, Error>)
    case retryAfter(TimeInterval, canRetryEarly: Bool = true)
}
```

**[High]** (`JobQueueRunner.swift:8-19`).

- `.finished` — success, terminal error, or "out of retries". The runner has
  already deleted the record. **[High]** (`JobQueueRunner.swift:9-11`)
- `.retryAfter(interval, canRetryEarly:)` — a transient/retryable error. If
  `canRetryEarly` (default `true`), the sleeping retry can be cancelled early by an
  external trigger (e.g. Reachability). **[High]** (`JobQueueRunner.swift:13-16`)

## The default error handler

Most queues route through:

```swift
JobAttemptResult.executeBlockWithDefaultErrorHandler(
    jobRecord:, retryLimit:, db:, block:)
```

which runs `block()`; on success returns `.finished(.success)`, and on throw calls
`performDefaultErrorHandler` inside an `awaitableWrite`. **[High]**
(`JobQueueRunner.swift:21-39`)

```swift
if jobRecord.failureCount < retryLimit, error.isRetryable {
    jobRecord.addFailure(tx: tx)
    let delay = OWSOperation.retryIntervalForExponentialBackoff(
        failureCount: jobRecord.failureCount, maxAverageBackoff: 14.1 * .minute)
    return .retryAfter(delay, canRetryEarly: true)
} else {
    jobRecord.anyRemove(transaction: tx)
    return .finished(.failure(error))
}
```

**[High]** (`JobQueueRunner.swift:41-66`). So the default policy is:

1. **Retryable error AND under the limit** → increment `failureCount`
   (persisted!), schedule exponential backoff capped at a `maxAverageBackoff` of
   **14.1 minutes**, allow early retry. **[High]**
2. **Non-retryable error, OR out of retries** → **delete the record** and finish
   with failure. **[High]**

Because `failureCount` is persisted via `addFailure` → `anyUpdate`, the retry
budget survives relaunches. **[High]** (`JobRecord.swift:170-174`)

> `error.isRetryable` is defined on `Error` outside this folder (used throughout).
> The exact classification (network vs. HTTP 5xx vs. fatal) is the caller's
> responsibility via `IsRetryableProvider`. **[Medium]** (referenced in
> `MessageSenderJobQueue.swift:30-39`; implementation outside `Jobs/`)

## Retry limits per queue

Each queue declares its own `retryLimit` / `maxRetries`:

| Queue / runner | `maxRetries` | Citation |
| --- | --- | --- |
| SendGiftBadge | 110 | `SendGiftBadgeJobQueue.swift:150-152` |
| IncomingContactSync | 4 | `IncomingContactSyncJobQueue.swift:69-71` |
| CallRecordDeleteAll | 110 | `CallRecordDeleteAllJobQueue.swift:120-123` |
| BulkDeleteInteraction | 110 | `BulkDeleteInteractionJobQueue.swift:78-81` |
| LocalUserLeaveGroup | 110 | `LocalUserLeaveGroupJob.swift:21-23` |
| MessageSender (bespoke) | 110, or **2** for PIN-change messages | `MessageSenderJobQueue.swift` `getMaxRetriesForMessageType` |
| DonationReceiptCredentialRedemption (bespoke) | unbounded transient retries (see below) | `DonationReceiptCredentialRedemptionJobQueue.swift` |

`110` is used as a crude stand-in for a ~24h timeout; MessageSender's comment says
so explicitly: "Use max-retries as a stand-in for a timeout for messages we want
sent or cancelled in less than 24 hours." **[High]**
(`MessageSenderJobQueue.swift` `getMaxRetriesForMessageType`)

## Exponential backoff

The shared primitive is `OWSOperation.retryIntervalForExponentialBackoff(
failureCount:maxAverageBackoff:)` (defined outside `Jobs/`). **[Medium]** It is
called with:

- `maxAverageBackoff: 14.1 * .minute` by the default handler and by MessageSender.
  **[High]** (`JobQueueRunner.swift:57`, `MessageSenderJobQueue.swift` `_runOperation`)
- `maxAverageBackoff: .day` by DonationReceiptCredentialRedemption's
  `incrementExponentialRetryDelay`. **[High]**
  (`DonationReceiptCredentialRedemptionJobQueue.swift` `incrementExponentialRetryDelay`)

## Early-retry triggers

### Generic engine: Reachability only

For queues on `JobQueueRunner`, the only early-retry trigger is Reachability. When
the device becomes reachable, `retryWaitingJobs` cancels all `waitingTasks`, so any
`.retryAfter(_, canRetryEarly: true)` fires immediately. `.retryAfter(_,
canRetryEarly: false)` is **not** registered and therefore waits out its full
delay. **[High]** (`JobQueueRunner.swift:261-283`, `JobQueueRunner.swift:360-371`)

Queues opt in via `listenForReachabilityChanges`:

| Queue | Listens for reachability? |
| --- | --- |
| SendGiftBadge | yes (`SendGiftBadgeJobQueue.swift:24`) |
| DonationReceiptCredentialRedemption | yes (`...:78`) |
| IncomingContactSync | yes (`IncomingContactSyncJobQueue.swift:30`) |
| LocalUserLeaveGroup | yes (`LocalUserLeaveGroupJob.swift:124`) |
| CallRecordDeleteAll | **no** (not wired) |
| BulkDeleteInteraction | **no** (not wired) |

### MessageSender: richer triggers

`MessageSenderJobQueue` has its **own** external-retry machinery with an
`ExternalRetryTriggers` option set:

```swift
static let networkBecameReachable = ExternalRetryTriggers(rawValue: 1 << 0)
static let chatConnectionOpened  = ExternalRetryTriggers(rawValue: 1 << 1)
```

**[High]** (`MessageSenderJobQueue.swift` `ExternalRetryTriggers`). Error-to-trigger
mapping during a failed send:

| Error property | Trigger inserted | Rationale (from source) |
| --- | --- | --- |
| `isNetworkFailure` | `.chatConnectionOpened` | retry as soon as we reconnect |
| `isTimeout` | `.networkBecameReachable` | same request likely fails again; back off but retry early on a better network |
| `httpResponseHeaders?.retryAfterTimeInterval` | — (sets `suggestedRetryDelay`) | obey server Retry-After |
| `SignalError.rateLimitedError(retryAfter:)` | — (sets `suggestedRetryDelay`) | obey rate limit |
| `AccountChecker.RateLimitError` | — (**sums** `accountCheckerRetryDelay`) | avoid O(n²) retry storms |
| `error.isFatalError` | — (**throws** immediately) | never retry; fails whole group send |

**[High]** (`MessageSenderJobQueue.swift` `_runOperation`). The actual wait is:

```swift
retryDelay = max(exponentialRetryDelay,
                 min(maxAverageBackoff, suggestedRetryDelay),
                 min(maxAverageBackoff, accountCheckerRetryDelay))
```

i.e. a minimum of exponential backoff is always enforced so Retry-After headers
"can't trigger tight retry loops on the client." The wait is a
`withCooperativeTimeout(seconds: retryDelay)` racing against
`waitForAnyExternalRetryTrigger(...)`. **[High]** (`MessageSenderJobQueue.swift`
`_runOperation`)

The triggers are reported from two notification observers set up in `setUp`:
`owsReachabilityDidChange` → `.networkBecameReachable`, and
`chatConnectionStateDidChange == .open` → `.chatConnectionOpened`. **[High]**
(`MessageSenderJobQueue.swift` `setUp`)

> The "clear triggers before each attempt" (`clearExternalRetryTriggers`) +
> "report after failure" pattern is explicitly designed to avoid a race where a
> trigger fires *between* starting a send and observing its failure. **[High]**
> (`MessageSenderJobQueue.swift` `ActiveOperationState` doc-comments)

### DonationReceiptCredentialRedemption: SEPA/iDEAL day-scale retries

This queue overrides `runJobAttempt` directly (not via the default handler):

- Network-failure/`isRetryable` errors → `.retryAfter(incrementExponentialRetryDelay())`
  with `maxAverageBackoff: .day` and an unbounded transient count. **[High]**
- `.paymentStillProcessing` → `.exponential` for card/PayPal/Apple Pay, or `.sepa`
  (fixed `sepaRetryInterval`, `canRetryEarly: false`) for SEPA and recurring iDEAL.
  **[High]** (`...JobQueue.swift` `retryModeIfStillProcessing`)
- `sepaRetryInterval` is **1 day** in production, **1 minute** in staging/test.
  **[High]** (`...JobQueue.swift` `Constants.sepaRetryInterval`)
- On restart, a persisted "still processing" error within the last interval causes
  the job to immediately return `.retryAfter(remainingDelay, canRetryEarly: false)`
  so it doesn't re-hit the server too soon. **[High]** (`...JobQueue.swift`
  `sepaRetryDelay`)
- `.paymentIntentRedeemed` is treated as a **success** (badge already granted by a
  sibling job). **[High]**
- `.paymentFailed`/`.localValidationFailed`/`.serverValidationFailed`/
  `.paymentNotFound` → delete + `.finished(.failure)`. **[High]**

## Timeouts

There is no first-class timeout; MessageSender's comment calls out that max-retries
is used "as a stand-in for a timeout … Eventually this will be replaced with
support for actual timeouts." **[High]** (`MessageSenderJobQueue.swift`
`getMaxRetriesForMessageType`)

## Summary of edge cases

- `failureCount` is incremented and **persisted** before each backoff, so retry
  budgets are durable. **[High]**
- Reaching `retryLimit` is treated as a terminal failure that **deletes** the
  record (it does not become `.permanentlyFailed`). **[High]**
  (`JobQueueRunner.swift:62-65`)
- `canRetryEarly: false` guarantees a *minimum* wait regardless of Reachability —
  used for server-dictated SEPA delays. **[High]**
- `TimeInterval.clampedNanoseconds` guards against overflow when converting the
  delay to `Task.sleep`. **[High]** (`JobQueueRunner.swift:360`)

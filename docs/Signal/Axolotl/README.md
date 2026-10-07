# `Signal/Axolotl/` — Sender-Key Expiration (app target)

Deep documentation of the **first-party `Signal/Axolotl/` subdirectory of the
`Signal/` app target only**. At authoring time this directory contains a single
file, `SenderKeyExpirationJob.swift`
(`Signal/Axolotl/SenderKeyExpirationJob.swift`) **[High]**.

The directory name "Axolotl" is historical — "Axolotl" was the former name of the
Signal/Double-Ratchet protocol. In this repository the actual cryptographic
machinery (the Signal protocol / Sealed Sender / Sender Keys) lives in
`SignalServiceKit/`, not here. This app-target subdirectory contains only a
maintenance job that *deletes expired Sender-Key records*; it performs **no**
cryptography itself **[High]**. The sibling store and protocol code it depends on
lives under `SignalServiceKit/Axolotl/`
(`SignalServiceKit/Axolotl/SenderKeyStore.swift`,
`SignalServiceKit/Axolotl/SenderKeyManager.swift`) and is treated here as a
**boundary** — this doc describes *how the app-target job uses it*, not its
internals **[High]**.

For the whole-repository directory map, see
[../../APPLICATION_MAP.md](../../APPLICATION_MAP.md); for the app-target overview
and dependency wiring, see [../README.md](../README.md) and
[../app-environment.md](../app-environment.md).

## Conventions

- **Citations** are `path:line` (specific declaration) or `path` (directory/file).
  Line numbers reflect the working tree at authoring time and may drift; relocate
  via the symbol name. Every factual claim about behavior cites a path.
- **Confidence labels** on claims about *purpose/behavior*:
  - **[High]** — directly observed (file read / declaration located in this session).
  - **[Medium]** — inferred from signatures, names, and cross-file convention; not
    every referenced file was read in full.
  - **[Low]** — educated inference from naming alone.
- Any uncited claim is a **defect**.
- Directory/file existence was produced by listing the working tree in this session
  **[High]**.

## Scope

In scope:

- `SenderKeyExpirationJob.swift` — the lone source file; a concrete
  `ExpirationJob` subclass that deletes aged-out `SenderKeyRecord`s
  (`Signal/Axolotl/SenderKeyExpirationJob.swift:9`) **[High]**.

Out of scope (boundaries, documented only as interactions):

- `ExpirationJob<ExpiringElement>` — abstract base
  (`SignalServiceKit/Expiration/ExpirationJob.swift:16`) **[High]**.
- `SenderKeyStore` / `SenderKeyRecord`
  (`SignalServiceKit/Axolotl/SenderKeyStore.swift:11`) **[High]**.
- `RemoteConfigProvider` / `maxSenderKeyAge`
  (`SignalServiceKit/Environment/RemoteConfigManager.swift:218`) **[High]**.

## Responsibility

`SenderKeyExpirationJob` is a long-running, in-process maintenance task whose sole
job is to delete `SenderKeyRecord` rows once they have exceeded a configured
maximum age, so stale Sender Keys don't accumulate in the local database
(`Signal/Axolotl/SenderKeyExpirationJob.swift:9-44`) **[High]**. It is a thin
specialization of the generic `ExpirationJob` scheduler: it supplies three pieces
of policy — *which* records to consider, *when* a record expires, and *how* to
delete it — and inherits all scheduling/looping logic from the base class
(`SignalServiceKit/Expiration/ExpirationJob.swift:44-58`) **[High]**.

### Key type

| Type | Where | Role |
|---|---|---|
| `SenderKeyExpirationJob` | `Signal/Axolotl/SenderKeyExpirationJob.swift:9` | Concrete `ExpirationJob<SenderKeyRecord>`; deletes aged Sender-Key records for one `DeletionType`. **[High]** |

Stored dependencies, all injected via `init`
(`Signal/Axolotl/SenderKeyExpirationJob.swift:14-32`) **[High]**:

- `deletionType: SenderKeyRecord.DeletionType` — selects which population of
  records this instance manages (`.thisDevice` vs `.otherDevice`).
- `remoteConfigProvider: any RemoteConfigProvider` — supplies `maxSenderKeyAge`.
- `senderKeyStore: SenderKeyStore` — the GRDB-backed store queried for records.
- (`dateProvider`, `db`, and a `PrefixedLogger(prefix: "[SenderKeyExpJob]")` are
  forwarded to the base `init`.)

### The three overridden policy hooks

- `nextExpiringElement(tx:)`
  (`Signal/Axolotl/SenderKeyExpirationJob.swift:31-33`) delegates to
  `senderKeyStore.fetchOldestSenderKeyRecord(deletionType:tx:)`, which returns the
  single oldest record of the configured `deletionType`, ordered by `insertedAt`
  ascending (`SignalServiceKit/Axolotl/SenderKeyStore.swift:185-193`) **[High]**.
- `expirationDate(ofElement:)`
  (`Signal/Axolotl/SenderKeyExpirationJob.swift:35-37`) computes
  `insertedAtDate + remoteConfigProvider.currentConfig().maxSenderKeyAge` **[High]**.
- `deleteExpiredElement(_:tx:)`
  (`Signal/Axolotl/SenderKeyExpirationJob.swift:39-43`) deletes the row via
  GRDB's `element.delete(tx.database)`, wrapped in `failIfThrows` **[High]**.

## Interactions with the rest of the module and the app

```
AppEnvironment.setUp
  └─ constructs SenderKeyExpirationJob (deletionType: .thisDevice)   [Signal/AppLaunch/AppEnvironment.swift:230]
        │  injects: Date.provider, DependenciesBridge.db,
        │           SSKEnvironment.remoteConfigManagerRef,
        │           DependenciesBridge.senderKeyStore
        ▼
AppLifecycleManager.runExpirationJobs()                             [Signal/AppLaunch/AppLifecycleManager.swift:1704]
  └─ taskGroup.addTask { AppEnvironment.shared.senderKeyExpirationJob.run() }
        ▼
ExpirationJob.run()  (loop)                                         [SignalServiceKit/Expiration/ExpirationJob.swift:76]
  └─ deleteExpiredElements()  →  TimeGatedBatch over the three hooks above
        └─ SenderKeyStore.fetchOldestSenderKeyRecord(.thisDevice)   [SignalServiceKit/Axolotl/SenderKeyStore.swift:185]
```

- **Construction / ownership.** The app target owns a single instance, held by
  `AppEnvironment` as `senderKeyExpirationJob`
  (`Signal/AppLaunch/AppEnvironment.swift:52`) and constructed during environment
  setup with `deletionType: .thisDevice`
  (`Signal/AppLaunch/AppEnvironment.swift:230-236`) **[High]**. All collaborators
  are pulled from `DependenciesBridge.shared` and `SSKEnvironment.shared`
  (`db`, `senderKeyStore`, `remoteConfigManagerRef`) **[High]**.
- **Invocation.** It is started as one task in the `withThrowingTaskGroup` inside
  `AppLifecycleManager.runExpirationJobs()`
  (`Signal/AppLaunch/AppLifecycleManager.swift:1694-1712`), alongside the other
  expiration jobs (disappearing messages, deleted call records, stories, pinned
  messages, decryption placeholders, group-send endorsements) **[High]**. Because
  `ExpirationJob.run()` loops forever, the task stays alive for the app session and
  is expected to terminate only via `CancellationError`
  (`SignalServiceKit/Expiration/ExpirationJob.swift:131`,
  `Signal/AppLaunch/AppLifecycleManager.swift:1705-1710`) **[High]**.
- **Only `.thisDevice` is swept here.** The app wires exactly one instance, for
  `.thisDevice` (our own send keys)
  (`Signal/AppLaunch/AppEnvironment.swift:233`); no `.otherDevice` instance is
  constructed at authoring time, so inbound/peer Sender Keys are not deleted by this
  job (searched; only this one construction site exists) **[Medium]**.
- **Boundary with `SignalServiceKit/Axolotl/`.** The record type, the deletion
  taxonomy, and the oldest-record query all live in the service kit
  (`SignalServiceKit/Axolotl/SenderKeyStore.swift:11-193`); this file only reads
  the oldest record and issues a row delete **[High]**.

## Important data flow

The base class drives an event loop (`ExpirationJob.run()`,
`SignalServiceKit/Expiration/ExpirationJob.swift:76-134`) **[High]**:

1. Call `deleteExpiredElements()`, which uses `TimeGatedBatch.processAll` to
   repeatedly: fetch the oldest record (`nextExpiringElement`), and if
   `dateProvider() >= expirationDate(ofElement:)`, delete it and continue; otherwise
   stop and return that record's expiration date
   (`SignalServiceKit/Expiration/ExpirationJob.swift:136-157`) **[High]**.
2. Sleep until the returned next-expiration date (or `.distantFuture` if the table
   is empty), subject to `minIntervalBetweenDeletes` (default `1`s)
   (`SignalServiceKit/Expiration/ExpirationJob.swift:30-37`,
   `:104-129`) **[High]**.
3. Re-run on wake, cancellation, `restart()`, or a
   `UIApplication.significantTimeChangeNotification`
   (`SignalServiceKit/Expiration/ExpirationJob.swift:89-99`) **[High]**.

Because `fetchOldestSenderKeyRecord` orders by `insertedAt` ascending and the job
walks from oldest to newest, the loop naturally stops at the first non-expired
record — all older records are already gone, all newer records are younger
(`SignalServiceKit/Axolotl/SenderKeyStore.swift:189-191`,
`SignalServiceKit/Expiration/ExpirationJob.swift:144-156`) **[High]**.

### Expiration-age source

`maxSenderKeyAge` comes from remote config
(`SignalServiceKit/Environment/RemoteConfigManager.swift:218-220`): flag
`ios.maxSenderKeyAge` (`:777`), a server-provided value in **milliseconds**
converted to seconds, defaulting to **two weeks**
(`2 * UInt64.weekInMs`) when unset **[High]**. The job reads the *current* config
on every `expirationDate(ofElement:)` call
(`Signal/Axolotl/SenderKeyExpirationJob.swift:36`), so a server-side change to the
threshold affects subsequent sweeps without a code change **[High]**. This is the
same age threshold the send path uses to decide a Sender Key is too old to reuse
(`SignalServiceKit/Axolotl/SenderKeyManager.swift:279`,
`SignalServiceKit/Messages/MessageSender+SenderKey.swift:104`) **[Medium]**.

## Notable cryptography considerations

- **No crypto in this file.** The job neither derives, rotates, nor validates keys;
  it only deletes rows. The serialized key material (`serializedRecord: Data`) is
  opaque to it (`SignalServiceKit/Axolotl/SenderKeyStore.swift:21`,
  `Signal/Axolotl/SenderKeyExpirationJob.swift:39-43`) **[High]**.
- **What a Sender Key is.** `SenderKeyRecord` is a GRDB row in table `"SenderKey"`
  holding `serializedRecord` (the libsignal `SenderKeyRecord` bytes) keyed by
  `(ownerRecipientId, ownerDeviceId, distributionId)` plus a `deletionType`
  (`SignalServiceKit/Axolotl/SenderKeyStore.swift:11-33`) **[High]**. Sender Keys
  back group/multi-recipient sends in the Signal protocol **[Medium]** — the
  underlying type comes from `LibSignalClient`
  (`SignalServiceKit/Axolotl/SenderKeyStore.swift:8`) **[High]**.
- **`DeletionType` ownership semantics.** `.thisDevice` keys are our own send keys
  that "we can delete whenever we want"; `.otherDevice` keys are peers' keys
  (comments at `SignalServiceKit/Axolotl/SenderKeyStore.swift:35-47`) **[High]**.
  This app-target job targets only `.thisDevice`
  (`Signal/AppLaunch/AppEnvironment.swift:233`) **[High]**.
- **Expiry is a hygiene/forward-secrecy aid, not security-critical deletion.**
  Expiring our own send keys forces periodic Sender-Key rotation (a fresh key must
  be distributed), bounding how long one key is reused; the deletion threshold is
  server-tunable (`maxSenderKeyAge`) rather than a hard protocol requirement
  **[Medium]** — intent beyond the code/comments undetermined; no further design
  evidence in source.
- **Deletion is unconditional on error.** `deleteExpiredElement` wraps the GRDB
  delete in `failIfThrows`, so a delete failure is a fatal/assert condition rather
  than a silently swallowed error
  (`Signal/Axolotl/SenderKeyExpirationJob.swift:39-43`) **[High]**.

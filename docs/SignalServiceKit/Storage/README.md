# SignalServiceKit `Storage/` — top-level files

This document covers the **six first-party Swift files that live directly in**
[`SignalServiceKit/Storage/`](../../../SignalServiceKit/Storage/) — i.e. the
files at the *root* of the directory, not the much larger GRDB/SDS machinery in
the `Storage/Database/` and `Storage/MediaGallery/` subdirectories. Those
subdirectories (`SDSDatabaseStorage`, `GRDBDatabaseStorageAdapter`,
`KeyValueStore`, `DatabaseRecovery`, finders, migrations, etc.) are a distinct
and far larger body of code and are **out of scope** for this file.

The six files documented here are a loosely-related grab-bag of storage-adjacent
utilities rather than a single cohesive subsystem:

| File | Primary export | One-line responsibility |
| --- | --- | --- |
| [`YDBStorage.swift`](../../../SignalServiceKit/Storage/YDBStorage.swift) | `enum YDBStorage` | Deletes the legacy YapDatabase (`Signal.sqlite`) files left over from the pre-GRDB era |
| [`TimeGatedBatch.swift`](../../../SignalServiceKit/Storage/TimeGatedBatch.swift) | `enum TimeGatedBatch` | Splits long-running work across multiple write transactions, bounded by elapsed time |
| [`SSKKeychainStorage.swift`](../../../SignalServiceKit/Storage/SSKKeychainStorage.swift) | `protocol KeychainStorage` + `KeychainStorageImpl` | Thin wrapper over the iOS Keychain (`Security.framework`) for generic-password items |
| [`RecipientIdFinder.swift`](../../../SignalServiceKit/Storage/RecipientIdFinder.swift) | `final class RecipientIdFinder` | Resolves a `ServiceId`/`SignalServiceAddress` to a `SignalRecipient`'s unique id / row id, enforcing the PNI→ACI rule |
| [`PendingReadReceiptRecord.swift`](../../../SignalServiceKit/Storage/PendingReadReceiptRecord.swift) | `struct PendingReadReceiptRecord` | GRDB record for the `pending_read_receipts` table |
| [`PendingViewedReceiptRecord.swift`](../../../SignalServiceKit/Storage/PendingViewedReceiptRecord.swift) | `struct PendingViewedReceiptRecord` | GRDB record for the `pending_viewed_receipts` table |

> **Confidence labels.** Each claim is tagged with a confidence level:
> - **[High]** — directly read from source; behavior is explicit in the code.
> - **[Medium]** — inferred from code with reasonable certainty, but depends on
>   collaborators defined outside the six files covered here.
> - **[Low]** — inferred from naming/comments; not fully verified in-tree.
>
> Where the source gives no evidence for a design question, the text says
> **"intent undetermined — no evidence in source"** rather than guessing.
>
> **Citations** are given as `File.swift:line` relative to the repository root.
> Line numbers reflect the tree at authoring time and may drift; use the cited
> symbol name to re-locate code if lines have moved.

---

## 1. `YDBStorage` — tearing down the legacy YapDatabase

`YDBStorage` is a `public enum` used purely as a namespace; it has no cases and
no instances. **[High]** (`YDBStorage.swift:6`)

Its only public entry point is `deleteYDBStorage()`, which deletes the legacy
pre-GRDB SQLite database (`Signal.sqlite`) plus its `-shm` and `-wal` sidecar
files, in **two** locations: the app document directory (the oldest location)
and the shared-container `database/` subdirectory. **[High]**
(`YDBStorage.swift:42-53`)

- The database filename is hard-coded as `"Signal.sqlite"`, with `-shm`/`-wal`
  variants derived from it. **[High]** (`YDBStorage.swift:14-16`)
- The two directory roots come from `OWSFileSystem.appDocumentDirectoryPath()`
  (legacy) and `OWSFileSystem.appSharedDataDirectoryPath()/database` (shared
  container). **[High]** (`YDBStorage.swift:7-12`)
- After deleting the individual files, it also calls
  `OWSFileSystem.deleteContents(ofDirectory:)` on the shared-data `database/`
  directory — but a source comment explicitly warns that it is **NOT** safe to
  delete the legacy app-document directory, because that is the app document dir
  itself. **[High]** (`YDBStorage.swift:50-52`)

`OWSFileSystem` is defined outside the six files covered here, so the exact
delete semantics (e.g. whether a missing file is a no-op) are **[Medium]** —
`deleteFileIfExists` strongly implies tolerance of absence by its name.

**Why it exists:** this is migration/cleanup code. Signal-iOS migrated from
YapDatabase to GRDB; `YDBStorage` reclaims the disk space held by the old store.
The GRDB side of that story lives in `Storage/Database/` (out of scope here).

---

## 2. `TimeGatedBatch` — time-bounded multi-transaction processing

`TimeGatedBatch` is a `public enum` namespace exposing static helpers for
processing work across **multiple** database write transactions, cutting a new
transaction whenever a wall-clock budget is exceeded. **[High]**
(`TimeGatedBatch.swift:9`)

The file leads with a prominent warning: *"You probably don't need this method
and shouldn't use it."* Splitting an operation across multiple transactions
requires care to preserve data integrity, because the helper will split
transactions at arbitrary points. It is intended for workloads with large,
unpredictable per-object cost. **[High]** (`TimeGatedBatch.swift:11-32`)

### `enumerateObjects(_:db:yieldTxAfter:…block:)`

Iterates a `Sequence`, calling `block(object, tx)` for each element inside a
`db.awaitableWrite { … }`. After each element it checks
`CACurrentMediaTime() - startTime`; once elapsed time reaches `yieldTxAfter`
(default `1.0` s) it returns from the write block to let a **new** transaction
open on the next outer-loop iteration. It finishes when the iterator is
exhausted (`isDone = true`). **[High]**
(`TimeGatedBatch.swift:36-70`)

Key properties:
- The time budget is a *suggestion*: the doc comment notes the actual maximum
  transaction duration is unbounded because `block` may itself be slow or never
  return. **[High]** (`TimeGatedBatch.swift:28-32`)
- Error typing is propagated via Swift typed throws (`throws(E)`), so the caller
  gets back the exact error type the block throws. **[High]**
  (`TimeGatedBatch.swift:37-44`)

### `processAll(…)` and `ProcessBatchResult`

The two `processAll` overloads drive a `processBatch` closure in a tight loop
until it returns `.done(result)`; `.more` means "keep going". The result type is
`ProcessBatchResult<T>` with cases `.more` and `.done(T)`. **[High]**
(`TimeGatedBatch.swift:74-79`, `TimeGatedBatch.swift:101-150`)

- The richer overload threads a per-transaction *context* object: `buildTxContext`
  runs once when a transaction opens, and `concludeTx` runs once just before it
  closes — useful for committing cursor/bookmark state about the batches run in
  that transaction. **[High]** (`TimeGatedBatch.swift:91-128`)
- The simpler overload is implemented in terms of the richer one by supplying a
  `DummyTxContext`. **[High]** (`TimeGatedBatch.swift:130-153`)
- `errorTxCompletion` (`.commit` default, or `.rollback`) selects between
  `db.awaitableWrite` and `db.awaitableWriteWithRollbackIfThrows` for the latest
  transaction when the block throws. Prior committed transactions are **not**
  rolled back. **[High]** (`TimeGatedBatch.swift:84-90`, `TimeGatedBatch.swift:177-192`)
- `delayTwixtTx` inserts a `Task.sleep` between transactions when more work
  remains. **[High]** (`TimeGatedBatch.swift:193-196`)
- `processBatchesInTransaction` runs batches back-to-back inside one transaction
  until the `maximumDuration` deadline passes, each batch wrapped in an
  `autoreleasepool`. **[High]** (`TimeGatedBatch.swift:208-231`)

The `DB`, `DBWriteTransaction`, and `GRDB.Database.TransactionCompletion` types
are defined outside these six files (the former in `Storage/Database/`, the
latter in the GRDB package), so the precise transaction/commit semantics are
**[Medium]** here. This helper is the "fetch-and-delete in batches" primitive
used by callers such as the durable job queues — see the
[`Jobs/`](../Jobs/README.md) documentation for consumers that process large
backlogs across relaunches.

---

## 3. `SSKKeychainStorage` — Keychain access

This file defines the error type, the protocol, and the production
implementation for storing raw `Data` blobs in the iOS Keychain. **[High]**
(`SSKKeychainStorage.swift:8-24`, `SSKKeychainStorage.swift:27-31`,
`SSKKeychainStorage.swift:35`)

### `KeychainError`

A `public enum` with cases `.notFound`, `.notAllowed`, and
`.unknownError(OSStatus)`. Its `fileprivate init(_:)` maps raw `OSStatus`
codes — `errSecItemNotFound → .notFound`, `errSecInteractionNotAllowed →
.notAllowed`, everything else → `.unknownError`. **[High]**
(`SSKKeychainStorage.swift:8-24`)

### `KeychainStorage` protocol

Three methods, all keyed by a `(service, key)` pair: `dataValue(service:key:)`,
`setDataValue(_:service:key:)`, and `removeValue(service:key:)`. **[High]**
(`SSKKeychainStorage.swift:27-31`)

### `KeychainStorageImpl`

The concrete implementation, constructed with
`isUsingProductionService: Bool`. **[High]** (`SSKKeychainStorage.swift:35-42`)

- **Staging namespacing:** when *not* using the production service,
  `normalizeService(_:)` appends `".staging"` to the service string, so staging
  and production builds never collide on the same Keychain items. **[High]**
  (`SSKKeychainStorage.swift:44-46`)
- **Base query:** every operation builds a `kSecClassGenericPassword` query with
  `kSecAttrService` (normalized) and `kSecAttrAccount` (the key). **[High]**
  (`SSKKeychainStorage.swift:48-54`)
- **Read:** `dataValue` adds `kSecMatchLimitOne` + `kSecReturnData`, calls
  `SecItemCopyMatching`, and throws `KeychainError(status)` on anything other
  than `errSecSuccess`. **[High]** (`SSKKeychainStorage.swift:56-66`)
- **Write (upsert):** `setDataValue` first attempts `SecItemAdd` with
  `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`. On `errSecSuccess` it
  returns; on `errSecDuplicateItem` it falls through to `SecItemUpdate`. The
  `ThisDeviceOnly` accessibility means these items are **not** migrated to a new
  device via encrypted backup. **[High]** (`SSKKeychainStorage.swift:68-105`)
- **Delete:** `removeValue` calls `SecItemDelete` and treats both
  `errSecSuccess` and `errSecItemNotFound` as success (idempotent delete).
  **[High]** (`SSKKeychainStorage.swift:107-117`)
- Each mutating operation logs a line (`Inserting`/`Updating`/`Removing
  <service>/<key>`). The log includes service and key names but **not** the
  stored data value. **[High]** (`SSKKeychainStorage.swift:76`,
  `SSKKeychainStorage.swift:88`, `SSKKeychainStorage.swift:108`)

The `Security.framework` constants (`kSec*`) and `Logger` live outside these six
files; the mapping of their exact runtime behavior is **[Medium]**/**[Low]**
accordingly, though the status-code handling above is read directly from source.

---

## 4. `RecipientIdFinder` — ServiceId → recipient id resolution

`RecipientIdFinder` is a `public final class` that resolves a Signal identity
(`ServiceId` or `SignalServiceAddress`) to a stored `SignalRecipient`'s
identifiers, applying one important business rule. **[High]**
(`RecipientIdFinder.swift:21`)

It is constructed from two collaborators defined elsewhere in SignalServiceKit:
`RecipientDatabaseTable` and `RecipientFetcher`. **[High]**
(`RecipientIdFinder.swift:22-32`)

### The `RecipientUniqueId` typealias and error

- `public typealias RecipientUniqueId = String` — the "unique id" returned is
  just a string. **[High]** (`RecipientIdFinder.swift:9`)
- `RecipientIdError` conforms to `IsRetryableProvider` and has a single case,
  `.mustNotUsePniBecauseAciExists`. Its `isRetryableProvider` returns `true`,
  with a comment explaining retries are allowed so the caller re-sends to the
  ACI instead of the PNI. **[High]** (`RecipientIdFinder.swift:11-18`)

### Lookup vs. ensure

The API comes in two flavors:
- **Lookups** (`recipientUniqueId(for:tx:)`, `recipientId(for:tx:)`) take a read
  transaction and return an **optional** `Result` — `nil` when no recipient
  exists for that identity. **[High]**
  (`RecipientIdFinder.swift:34-39`, `RecipientIdFinder.swift:48-53`)
- **Ensures** (`ensureRecipientUniqueId(for:tx:)`, `ensureRecipientId(for:tx:)`,
  `ensureRecipient(for:tx:)`) take a write transaction and
  `recipientFetcher.fetchOrCreate(...)`, so they always return a (non-optional)
  `Result`, creating the recipient if needed. **[High]**
  (`RecipientIdFinder.swift:41-46`, `RecipientIdFinder.swift:55-57`,
  `RecipientIdFinder.swift:59-62`)

Variants return either `RecipientUniqueId` (`.uniqueId`) or
`SignalRecipient.RowId` (`.id`) by mapping over the validated recipient.
**[High]** (`RecipientIdFinder.swift:37`, `RecipientIdFinder.swift:52`)

### The PNI→ACI rule

Every path funnels through `validateRecipient(_:for:)`. If the identity is a
`Pni` and the recipient `!canSendToPni()`, it fails with
`.mustNotUsePniBecauseAciExists`; otherwise it succeeds. This encodes the rule
that once an ACI is known for a contact, the PNI must not be used. **[High]**
(`RecipientIdFinder.swift:64-74`)

`ServiceId`/`Pni`/`Aci` come from `LibSignalClient`
(`RecipientIdFinder.swift:7`), and `SignalRecipient`, `RecipientDatabaseTable`,
and `RecipientFetcher` are defined elsewhere in SignalServiceKit, so the
behavior of `canSendToPni()` and `fetchOrCreate` is **[Medium]** from this
file's perspective. See the [`Contacts/`](../Contacts/README.md) documentation
for the recipient/identity model this finder reads from.

---

## 5 & 6. `PendingReadReceiptRecord` / `PendingViewedReceiptRecord`

These two files are near-identical GRDB record structs backing the two
"pending outbound receipt" tables. Both conform to `Codable`,
`FetchableRecord`, and `MutablePersistableRecord`. **[High]**
(`PendingReadReceiptRecord.swift:9`, `PendingViewedReceiptRecord.swift:9`)

| Record | `databaseTableName` |
| --- | --- |
| `PendingReadReceiptRecord` | `"pending_read_receipts"` **[High]** (`PendingReadReceiptRecord.swift:11`) |
| `PendingViewedReceiptRecord` | `"pending_viewed_receipts"` **[High]** (`PendingViewedReceiptRecord.swift:11`) |

Both have the identical column set: an auto-incrementing `id: Int64?`, plus
`threadId`, `messageTimestamp`, `messageUniqueId?`, `authorAciString?`, and
`authorPhoneNumber?`. **[High]**
(`PendingReadReceiptRecord.swift:13-19`, `PendingViewedReceiptRecord.swift:13-19`)

Noteworthy, identical details across both files:

- **Legacy column name:** `authorAciString` is persisted under the on-disk
  column name `"authorUuid"` via `CodingKeys`. This is a frozen legacy name;
  the Swift property was renamed but the stored column was not. **[High]**
  (`PendingReadReceiptRecord.swift:21-29`, `PendingViewedReceiptRecord.swift:21-29`)
- **ACI-xor-phone invariant:** the memberwise `init` stores the ACI's
  `serviceIdUppercaseString` and *clears* `authorPhoneNumber` whenever an ACI is
  present (`authorPhoneNumber = (authorAci == nil) ? authorPhoneNumber : nil`).
  The custom `init(from:)` decoder enforces the same rule on read: it only
  decodes `authorPhoneNumber` when `authorAciString` is `nil`. So a row never
  carries both an ACI and a phone number. **[High]**
  (`PendingReadReceiptRecord.swift:31-37`, `PendingReadReceiptRecord.swift:39-47`,
  `PendingViewedReceiptRecord.swift:31-37`, `PendingViewedReceiptRecord.swift:39-47`)
- **Row-id capture:** `didInsert(with:for:)` writes the assigned rowID back into
  `id`, as required by `MutablePersistableRecord`. **[High]**
  (`PendingReadReceiptRecord.swift:49-51`, `PendingViewedReceiptRecord.swift:49-51`)

`Aci` comes from `LibSignalClient`, and `FetchableRecord` /
`MutablePersistableRecord` from the GRDB package
(`PendingReadReceiptRecord.swift:7-8`), so the surrounding
serialization/fetching machinery is **[Medium]** from these files alone.

**Who writes these rows:** these records represent read/viewed receipts that
are queued to be *sent* to message authors but not yet delivered. The queuing,
batching, and sending logic lives outside these six files (in the receipt /
message-sending subsystems); the precise producer/consumer is **intent
undetermined — no evidence in source** within the files covered here. See
[`Messages/`](../Messages/README.md) for the receipt-sending flow and
[`DisappearingMessages/`](../DisappearingMessages/README.md) for related
read-state handling.

---

## Cross-references

- [`Storage/Database/`](../../../SignalServiceKit/Storage/Database/) — the GRDB
  adapter, `SDSDatabaseStorage`, `KeyValueStore`, migrations, finders, and
  record infrastructure. This is where `DB`/`DBWriteTransaction` (used by
  `TimeGatedBatch`) and the GRDB record protocols (used by the pending-receipt
  records) are actually wired up. Out of scope for this file, but the natural
  next read.
- [`Jobs/`](../Jobs/README.md) — durable queues that perform large, batched
  "fetch-and-delete" style work and are natural consumers of `TimeGatedBatch`.
- [`Contacts/`](../Contacts/README.md) — the `SignalRecipient` / recipient-table
  model that `RecipientIdFinder` resolves against.
- [`Messages/`](../Messages/README.md) — receipt sending, the likely consumer of
  the `PendingReadReceiptRecord` / `PendingViewedReceiptRecord` tables.

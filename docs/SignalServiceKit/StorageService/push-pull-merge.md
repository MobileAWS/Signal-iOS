# Storage Service — Push / Pull / Merge Cycle

This document traces the three network-facing flows end-to-end: **push**
(`backup`), **pull** (`restoreOrCreate`), and **manifest rotation**. Conflict
handling that these flows trigger is detailed in
[conflict-resolution.md](conflict-resolution.md). Sources:
`StorageServiceManager.swift` (orchestration) and `StorageService.swift`
(network/crypto).

---

## The server API (recap)

`StorageService.swift` exposes exactly three operations: **[High]**

| Operation | HTTP | Endpoint | Status handling |
| --- | --- | --- | --- |
| `fetchLatestManifest(ifGreaterThanVersion:…)` | GET | `v1/storage/manifest[/version/N]` | 204 `noNewerManifest`, 404 `noExistingManifest`, 200 decrypt+parse, else error (`:194-242`) |
| `fetchItems(for:manifest:…)` | PUT | `v1/storage/read` | 200 decrypt each item, else error (`:338-406`) |
| `updateManifest(_:newItems:deletedIdentifiers:deleteAllExistingRecords:…)` | PUT | `v1/storage` | 200 success, **409 → `conflictingManifest`**, else error (`:245-336`) |

`fetchItems` dedups keys and asserts `keys.count <= 1024` (the server 500s on
larger reads) (`:347-349`). The write operation bundles the encrypted manifest,
inserted items, deleted keys, and a `deleteAll` flag into one
`StorageServiceProtoWriteOperation` (`:245-304`). **[High]**

---

## Push — `backupPendingChanges`

Entry: `StorageServiceOperation.backupPendingChanges`
(`StorageServiceManager.swift:969`). **[High]**

Steps:

1. Read `State.current`, `normalizePendingMutations` (reassign a dirty *local*
   recipient to the account record, `:951`), then run `updateRecords` for every
   type (account, contact, gv1, gv2, DList, call link).
2. For each dirty local id, `updateRecord` (`:973`): **always deletes the old
   identifier** (`:1007`), clears its state, and — if the record still exists —
   builds a new record with a **fresh identifier** and appends it to
   `updatedItems`. Unknown fields on the old record are preserved into the new
   one. **[High]**
3. If nothing changed, return early (`:1085`). Pull in any `invalidIdentifiers`
   to delete alongside this write (`:1092`).
4. **Bump `manifestVersion += 1`** (`:1096`) and build the manifest from
   `allIdentifiers`.
5. `StorageService.updateManifest(...)` with `deleteAllExistingRecords: false`
   (`:1115`).
6. On success: save `State` (`clearConsecutiveConflicts: true`, `:1148`) and
   `sendFetchLatestStorageManifestSyncMessage()` to nudge linked devices
   (`:1152`). **[High]**

Error handling (`:1123-1146`): **[High]**

- `conflictingManifest` (409) → assert the conflict version ≥ our proposed
  version (`:1124`), then `mergeLocalManifest(…, mergeReason: .conflictBackingUp)`
  and **return** (the merge re-triggers the backup). See
  [conflict-resolution.md](conflict-resolution.md).
- `manifestDecryptionFailed` / `manifestProtoDeserializationFailed` **on a
  primary** → the remote manifest is unreadable garbage blocking us; overwrite
  it via `createNewManifestAndRecords(version: conflictingVersion + 1)`.

```mermaid
sequenceDiagram
    participant Op as StorageServiceOperation(.backup)
    participant DB as GRDB (State + records)
    participant SS as Storage Service

    Op->>DB: read State; build dirty records (new identifiers)
    alt no changes
        Op-->>Op: return
    end
    Op->>Op: manifestVersion += 1; buildManifestRecord
    Op->>SS: updateManifest(newItems, deletedIds, deleteAll=false)
    alt 200 OK
        SS-->>Op: ok
        Op->>DB: save State (clear conflicts)
        Op->>SS: sendFetchLatestStorageManifestSyncMessage
    else 409 conflictingManifest
        SS-->>Op: remote manifest
        Op->>Op: mergeLocalManifest(.conflictBackingUp) -> re-backup
    else manifest unreadable (primary)
        Op->>Op: createNewManifestAndRecords(conflictVersion+1)
    end
```

---

## Pull — `restoreOrCreateManifestIfNecessary`

Entry: `StorageServiceOperation.restoreOrCreateManifestIfNecessary`
(`StorageServiceManager.swift:1183`). **[High]**

1. Read `State`; choose `ifGreaterThanVersion`: normally the local
   `manifestVersion`, but **`nil` if `refetchLatestManifest`** is set (so the
   server doesn't 204 us when we specifically want the bytes again).
2. `StorageService.fetchLatestManifest(...)` and branch on the response:
   - `.noExistingManifest` (`:1201`) → `createNewManifestAndRecords(version: 1)`
     (first device to ever write).
   - `.noNewerManifest` (`:1204`) → nothing to do.
   - `.latestManifest(m)` (`:1207`) → `mergeLocalManifest(withRemoteManifest: m,
     mergeReason: .fetchedLatest)`.
3. **Decryption/deserialization recovery** (the `catch` after `:1213`):
   - primary → `createNewManifestAndRecords(version: manifestVersion + 1)`.
   - linked → `sendKeysSyncRequestMessageIfNeeded()` then rethrow.

```mermaid
sequenceDiagram
    participant Op as StorageServiceOperation(.restoreOrCreate)
    participant SS as Storage Service

    Op->>SS: fetchLatestManifest(ifGreaterThanVersion: localVersion?)
    alt 404 noExistingManifest
        Op->>Op: createNewManifestAndRecords(version: 1)
    else 204 noNewerManifest
        Op-->>Op: done
    else 200 latestManifest(m)
        Op->>Op: mergeLocalManifest(.fetchedLatest)
    else manifest unreadable
        alt primary
            Op->>Op: createNewManifestAndRecords(localVersion+1)
        else linked
            Op->>Op: sendKeysSyncRequestMessageIfNeeded(); rethrow
        end
    end
```

---

## Creating new manifests

### `createNewManifestAndRecords(version:)` (`:1284`)

Primary-only (`owsPrecondition(isPrimaryDevice)`, `:1285`). Builds a **fresh
`State`**, generates a new 32-byte `recordIkm` (`:1296`), enumerates the local
database to create a record for every recipient (account vs contact), group,
story thread (+ deleted DList tombstones), and call link, then writes the
manifest with `deleteAllExistingRecords = (version > 1)` — asking the server to
purge orphans only when we're not the very first write (an expensive server
query, so done only when necessary) (`:1379-1387`). On conflict it bumps the
version and retries once more with `deleteAll: true`; a second conflict is fatal
("Repeated conflicts … giving up", `:1412`). **[High]**

### `createNewManifestPreservingRecords(version:)` (`:1241`)

Primary-only (`:1242`). Only valid when a `recordIkm` already exists (records are
encrypted independently of the manifest). If there's no `recordIkm`, it
**pivots** to `createNewManifestAndRecords`. Otherwise it rewrites just the
manifest (no new/deleted items), and on a version conflict pivots to a full
record rotation. **[High]**

### `createNewManifestAndSaveState(...)` (`:1423`)

Shared helper used by both. Returns `nil` on success (and sends the
fetch-latest sync message `:1446` + saves `State` `:1450`), or the **conflicting
remote version** on a 409 (`:1454`) / unreadable-manifest conflict so the caller
can bump and overwrite. **[High]**

---

## Manifest rotation — `rotateManifest(mode:)`

Public `rotateManifest` (`:597`) enqueues a `pendingManifestRotation`; the
operation runs in `.rotateManifest(mode:)` mode, dispatched in
`_run` at `:830`. It is **primary-only** ("Can only rotate manifest from primary
device!", `:832`). The next version is `currentState.manifestVersion + 1`
(`:835`), dispatched to either `createNewManifestPreservingRecords` (`:839`) or
`createNewManifestAndRecords` (`:841`). **[High]**

`StorageServiceManagerManifestRotationMode` (`StorageServiceManager.swift:102-136`):
**[High]**

| Mode | Behavior | Precedence |
| --- | --- | --- |
| `.preservingRecordsIfPossible` | recreate the manifest in place, keeping record identifiers + `recordIkm`. **Only applicable if a `recordIkm` already exists**; otherwise treated as `.alsoRotatingRecords`. | 0 (lower, `:122`) |
| `.alsoRotatingRecords` | recreate the manifest *and* all records with new identifiers + a new `recordIkm`; deletes all existing records. | 1 (higher, `:123`) |

When two rotations are queued, `mergeByPrecedence` (`:129`) keeps the stronger
(`.alsoRotatingRecords` wins). The record-IKM migrator always uses
`.alsoRotatingRecords` (see [encryption-and-ikm.md](encryption-and-ikm.md)).
**[High]**

---

## Merge (pull application) — `mergeLocalManifest`

`mergeLocalManifest(withRemoteManifest:mergeReason:)`
(`StorageServiceManager.swift:1492`) is where a fetched (or conflicting) remote
manifest is applied. The detailed semantics — account-record priority, batched
fetch, deferred ACI-less contacts, orphan re-adoption, invalid-identifier
bookkeeping, and decryption-failure recovery — are documented in
[conflict-resolution.md](conflict-resolution.md). The high-level shape: **[High]**

1. `normalizePendingMutations` (`:1499`); if merging due to a backup conflict,
   bump `consecutiveConflicts` (`:1505`) and bail if over the ceiling (`:1515`).
2. Refuse to merge a **lower** manifest version than local (`:1528`).
3. Compute `newOrUpdatedItems = remoteKeys − localKeys` (`:1539`), plus any
   now-parseable previously-unknown identifiers.
4. Fetch + merge the **account record first** (`:1562`), then
   `fetchAndMergeItemsInBatches` for the rest (`:1615`).
5. Set `manifestVersion` and `recordIkm` to the remote's (`:1626`), clear
   `refetchLatestManifest`, recompute `invalidIdentifiers` (`:1671`), **re-mark
   orphaned records as `.updated`** so they get re-pushed (`:1677-1735`), and
   save.
6. If the merge was triggered by a backup conflict, re-enqueue the backup
   (`:1760`). **[High]**

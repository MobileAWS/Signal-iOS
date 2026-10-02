# Storage Service — Manifest & Record Model

This document covers the data model: the manifest, storage identifiers, storage
items, the generated proto record types, and the per-type **record updaters**
that translate between local database state and Storage Service protos. Sources:
`SignalServiceKit/StorageService/StorageService.swift` and
`StorageServiceProto+Sync.swift`.

---

## Storage identifiers

<<<<<<< ours
A `StorageService.StorageIdentifier` (`StorageService.swift:47-76`) names a
=======
A `StorageService.StorageIdentifier` (`StorageService.swift:47-81`) names a
>>>>>>> theirs
single record. It is: **[High]**

```swift
struct StorageIdentifier: Hashable, Codable {
    let data: Data   // 16 random bytes
    let type: StorageServiceProtoManifestRecordKeyType
}
```

- `generate(type:)` (`:56`) produces a fresh identifier with **16 random bytes**
  (`Randomness.generateRandomBytes(16)`). A new identifier is minted **every time
  a record changes** — identifiers are immutable names for an immutable version
  of a record. **[High]**
- `buildRecord()` (`:60`) wraps it as a `StorageServiceProtoManifestRecordKey`.
- `deduplicate(_:)` (`:65`) keeps one identifier per `data`, `owsFailDebug`-ing
  on collisions across types — guards against a corrupt manifest listing the same
  key twice. **[High]**

The `type` enum (`StorageServiceProtoManifestRecordKeyType`) has cases
`unknown`, `contact`, `groupv1`, `groupv2`, `account`, `storyDistributionList`,
`callLink`, and `UNRECOGNIZED`; it is made `Codable` and `CustomStringConvertible`
at `StorageService.swift:530-567`. **[High]**

---

## Storage items

A `StorageService.StorageItem` (`StorageService.swift:78-180`) pairs an
identifier with a decoded `StorageServiceProtoStorageRecord` (a oneof over the
concrete record types). It provides **typed accessors** that return the inner
record only when the identifier `type` matches the oneof case, `owsFailDebug`-ing
on mismatch — e.g. `contactRecord` (`:84`), `groupV2Record`, `accountRecord`,
`storyDistributionListRecord`, `callLinkRecord`. Convenience initializers build a
`StorageRecord` from each concrete record type (`:152-180`). **[High]**

---

## The manifest record

`StorageServiceProtoManifestRecord` (generated proto) carries: **[Medium]** —
fields observed in use at `StorageServiceManager.swift:1155-1181` and
`StorageService.swift`:

- `version: UInt64` — monotonically increasing.
- `keys: [ManifestRecordKey]` — every storage identifier in the account.
- `recordIkm: Data?` — optional 32-byte record IKM (see
  [encryption-and-ikm.md](encryption-and-ikm.md)).
- `sourceDevice: UInt32` — the device id that wrote this manifest.

`buildManifestRecord` (`StorageServiceManager.swift:1155`): dedups the
identifiers, sets `recordIkm` (asserting its length is
`ManifestRecordIkm.expectedLength == 32` — "Found manifest recordIkm with
unexpected length! Who generated it?"), sets the key list, and stamps
`sourceDevice` from `tsAccountManager.storedDeviceId` (or `owsFailDebug`s on an
invalid device id). **[High]**

A debug `logDescription` renders a manifest as `v[version].sourceDevice`
(`StorageServiceManager.swift:2558`). **[High]**

---

## The record updater protocol

`StorageServiceRecordUpdater` (`StorageServiceProto+Sync.swift:13-62`) is the
contract each record type implements. **[High]**

```swift
protocol StorageServiceRecordUpdater {
    associatedtype IdType
    associatedtype RecordType
    func unknownFields(for: RecordType) -> UnknownStorage?
    func buildRecord(for: IdType, unknownFields: UnknownStorage?, transaction:) -> RecordType?
    func buildStorageItem(for: RecordType) -> StorageService.StorageItem
    func mergeRecord(_: RecordType, transaction:) -> StorageServiceMergeResult<IdType>
}
```

- `buildRecord` turns a **local id** into a proto, returning `nil` if the local
  object doesn't exist or isn't eligible (callers skip `nil`). **[High]**
- `buildStorageItem` wraps a built proto with a *freshly generated* identifier
  of the right type (e.g. `.generate(type: .contact)`,
  `StorageServiceProto+Sync.swift:420`). **[High]**
- `mergeRecord` applies a remote proto to local state and returns a
  `StorageServiceMergeResult`. **[High]**

`StorageServiceMergeResult<IdType>` (`:64-76`): **[High]**

- `.invalid` — the record is malformed (usually missing its identifier); it will
  be deleted.
- `.merged(needsUpdate: Bool, IdType)` — merged; `needsUpdate == true` means
  local state diverged from the remote and we should schedule a push.

The protocol's own doc comment states the **merge philosophy**: "the latest
value on the service is always right," with quick pushes mitigating the
force-quit / simultaneous-edit windows. **[High]** — `:36-56`.

### State updaters (wiring)

Two generic wrappers in `StorageServiceManager.swift` adapt a record updater to
the single `State` struct via key paths: **[High]**

- `SingleElementStateUpdater` (`:2400`) — for the one-of-a-kind account record
  (`IdType == Void`); its change state / identifier / unknown-fields live in
  scalar `State` properties.
- `MultipleElementStateUpdater` (`:2461`) — for the keyed collections (contacts,
  groups, DLists, call links); backed by dictionaries.

`buildAccountUpdater` (`:2099`) and the sibling `buildContactUpdater` /
`buildGroupV1Updater` / `buildGroupV2Updater` /
`buildStoryDistributionListUpdater` / `buildCallLinkUpdater` (immediately
following) construct each with the key paths into `State` and the (large) set of
injected managers. **[High]**

---

## The record types

### Contact record (`StorageServiceContactRecordUpdater`, `:176`)

- **Local id:** `RecipientUniqueId`. **Proto:** `StorageServiceProtoContactRecord`.
- **`StorageServiceContact`** (`:80-174`) is the validated value type: it must
  have **at least one of ACI/PNI** (`AtLeastOneServiceId`), optionally a phone
  number and an `unregisteredAtTimestamp`. It can be built from a proto (`:118`)
  or a `SignalRecipient` (`:148`). **[High]**
- `buildRecord` (`:243`): refuses to build for any local identifier ("Can't
  create contact with any local identifier") and for contacts that
  `shouldBeInStorageService == false`. It copies ACI/PNI/e164, blocked/hidden/
  whitelisted flags, identity key + verification state, profile key + names,
  **system-contact names only when appropriate** (primary with a local address-book
  contact, or linked with a previously-synced contact), thread archived/unread/
  muted, story-hidden, a **username only when it's the best identifier available**
  (`BetterIdentifierChecker`), nickname/note, and avatar color. **[High]**
- **Validation / registration gating.** `registrationStatus` (`:108`) and
  `shouldBeInStorageService` (`:162`) classify a contact as `registered`,
  `unregisteredRecently` (within `remoteConfig.messageQueueTime`), or
  `unregisteredAWhileAgo` (dropped from Storage Service). **[High]**
- `shouldDeferMerge` (`:424`): a contact record **with no ACI** is deferred to a
  second merge pass, so ACI-bearing records (which may trigger a split) are
  processed first. **[High]**
- `mergeRecord` (`:428`): rejects invalid identifiers and local-user records
  (`.invalid`), applies a `recipientMerger.applyMergeFromStorageService`, marks
  (un)registered, optionally splits an ACI-only unregistered recipient, then
  merges profile key/names, system names, identity key + verification state, etc.
  `needsUpdate` is set whenever the merged recipient's ids differ from the
  record's (`:487-495`). **[High]**

### Group V1 record (`StorageServiceGroupV1RecordUpdater`, `:896`)

- **Local id:** `Data` (group id). GV1 is effectively deprecated; the change map
  is `@EmptyForCodable` and no longer written (see
  [architecture.md](architecture.md)). Still mergeable (`mergeRecord` `:922`) for
  backward compat. **[Medium]** — short updater; GV1 is legacy.

### Group V2 record (`StorageServiceGroupV2RecordUpdater`, `:932`)

- **Local id:** `Data` (the **group master key**). `buildRecord` (`:967`) derives
  the group id from the master key via `GroupSecretParams.deriveFromMasterKey`,
  returns `nil` if the local `GroupRecord` is gone (⇒ delete remotely), and copies
  whitelisted/blocked/story-hidden plus thread archived/unread/muted,
  mention-notify, story-send-mode, verified-name-hash — falling back to a
  *pending-restore* record's fields when the thread doesn't exist yet. **[High]**
- `mergeRecord` (`:1048`): derives the group id (`.invalid` on a bad master
  key), **inserts a `GroupRecord` immediately** (even before the group is
  restored), and reconciles story-send-mode, mention-notify, archived/unread/
  muted against the local thread. **[High]**

### Account record (`StorageServiceAccountRecordUpdater`, `:1194`)

- **Local id:** `Void` (there is exactly one). This is the largest record: it
  carries the local user's profile key, username + username link (+ QR color),
  profile given/family name, and a long tail of app settings. **[High]** —
  `buildRecord` begins at `:1301`, `mergeRecord` at `:1480`.
- The updater is injected with ~30 managers (`buildAccountUpdater` in
  `StorageServiceManager.swift:2099`) covering backup plan/subscription,
  donations, disappearing-message config, link previews, usernames, key
  transparency, payments, phone-number discoverability, pinned threads,
  preferences, receipts, typing indicators, UD, blocking, etc. — each contributes
  a slice of the account record. **[Medium]** — exhaustive field-by-field mapping
  not enumerated here; see the `buildRecord`/`mergeRecord` bodies.

### Story distribution list record (`StorageServiceStoryDistributionListRecordUpdater`, `:2088`)

- **Local id:** `Data` (distribution-list identifier / UUID). Handles both live
  private-story threads and **deleted** distribution lists (tombstones via
  `privateStoryThreadDeletionManager`); `mergeRecord` at `:2170`. **[High]** — see
  `StorageServiceManager.swift:1284` (`createNewManifestAndRecords`) for how
  deleted DLists are enumerated into records.

### Call link record (`StorageServiceCallLinkRecordUpdater`, `:2279`)

- **Local id:** `Data` (call-link root key bytes). Backed by `callLinkStore` and
  the call-record delete/store managers; `mergeRecord` at `:2334`. Admin-deleted
  links are GC'd and propagated (see [architecture.md](architecture.md)
  deleted-call-link GC). **[High]**

---

## Unknown fields and unknown record types

Two distinct forward-compatibility mechanisms: **[High]**

1. **Unknown fields** — a *known* record type whose proto contains fields this
   build doesn't recognize. The merge keeps the whole record
   (`setRecordWithUnknownFields`) so a push re-serializes those bytes verbatim
   (`StorageServiceManager.swift:2064` `mergeRecord`). Once per app version,
   `cleanUpRecordsWithUnknownFields` (`:1961`) re-merges them in case the new
   build now understands the fields. **[High]**
2. **Unknown record types** — an *unknown* `type` entirely. The identifier is
   stashed in `unknownIdentifiersTypeMap` (in `mergeItems`, `:1841`), re-listed
   in every manifest we write (`allIdentifiers`, `:2335`), and never fetched
   again until the type becomes known — at which point `cleanUpUnknownIdentifiers`
   (`:1936`) sets `refetchLatestManifest = true` to pull them. **[High]**

`isKnownKeyType` (`:1917`) is the single source of truth for which `type`s this
build understands. **[High]**

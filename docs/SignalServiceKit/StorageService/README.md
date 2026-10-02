# Signal iOS — Storage Service Subsystem

This documentation set reconstructs Signal iOS's **Storage Service** subsystem
from the first-party source in `SignalServiceKit/StorageService/` (~5.6k LOC).
Storage Service is an **encrypted, server-hosted account-sync service**: it
stores an encrypted "social graph" — your contacts, groups, pinned
conversations, story distribution lists, call links, and your own account
settings — so that your linked devices (and a freshly-registered device) all
converge on the same state without the server ever seeing plaintext.

> **Confidence labels.** Each claim is tagged with a confidence level:
> - **[High]** — directly read from source; behavior is explicit in code.
> - **[Medium]** — inferred from code with reasonable certainty, but depends on
>   collaborators defined outside this directory (notably **LibSignalClient**,
>   `AccountKeyStore`/`MasterKey`, the many record "managers" injected into the
>   record updaters, and the generated `StorageServiceProto*` types).
> - **[Low]** — inferred from naming/comments; not fully verified in-tree.
>
> Where the code does not reveal *why* something is done, this is stated as
> "intent undetermined — no evidence in source".
>
> All file+line citations refer to the state of the tree at the time of writing.
> Line numbers are approximate anchors; use the cited symbol name to locate code
> if lines have drifted.

---

## The central concept: a manifest + a bag of encrypted records

Storage Service stores two kinds of blobs on the server, both opaque ciphertext
to the server:

1. A single **manifest** (`StorageServiceProtoManifestRecord`), versioned by a
   monotonically-increasing `UInt64`. The manifest is a list of **storage
   identifiers** (`key` + `type`) naming every record that currently belongs to
   the account, plus a `recordIkm` (record input-keying-material) and the
   `sourceDevice` that wrote it. **[High]** — `StorageService.swift:194-242`
   (fetch), `StorageServiceManager.swift:1155-1181` (`buildManifestRecord`).
2. Zero or more **records** ("storage items"), each named by a 16-byte random
   identifier and a `type` enum (contact, groupv1, groupv2, account,
   storyDistributionList, callLink). Each record is independently encrypted.
<<<<<<< ours
   **[High]** — `StorageService.swift:47-186` (`StorageIdentifier` /
=======
   **[High]** — `StorageService.swift:47-205` (`StorageIdentifier` /
>>>>>>> theirs
   `StorageItem`).

The server exposes three operations: fetch-latest-manifest (conditional on a
version), read-items (by identifier), and write (an atomic
manifest+inserts+deletes operation that **fails with 409 on a version
conflict**). All three live in `StorageService.swift`. **[High]**

```mermaid
graph TD
    subgraph Server["Storage Service (server, sees only ciphertext)"]
<<<<<<< ours
        Manifest["StorageManifest<br/>version N<br/>value = enc(ManifestRecord)"]
        Items["StorageItems<br/>key -> value = enc(StorageRecord)"]
    end
    subgraph Device["This device"]
        State["StorageServiceOperation.State<br/>(local mirror: version,<br/>recordIkm, id maps, change maps)"]
        DB[(GRDB: recipients, groups,<br/>threads, call links, settings)]
=======
        Manifest["StorageManifest\nversion N\nvalue = enc(ManifestRecord)"]
        Items["StorageItems\nkey -> value = enc(StorageRecord)"]
    end
    subgraph Device["This device"]
        State["StorageServiceOperation.State\n(local mirror: version,\nrecordIkm, id maps, change maps)"]
        DB[(GRDB: recipients, groups,\nthreads, call links, settings)]
>>>>>>> theirs
    end

    DB -->|build records| State
    State -->|updateManifest PUT /v1/storage| Manifest
    State -->|insert items| Items
    Manifest -->|fetchLatestManifest GET| State
    Items -->|fetchItems PUT /v1/storage/read| State
    State -->|mergeRecord| DB

    classDef srv fill:#eef,stroke:#44a;
    class Manifest,Items srv;
```

---

## Document map

| Doc | Area covered | Primary source |
| --- | --- | --- |
| [architecture.md](architecture.md) | The actors (`StorageServiceManager`, `StorageServiceOperation`), the single-operation serialized state machine, scheduling (debounce timer, cron, app lifecycle), auth, and the local `State` mirror | `StorageServiceManager.swift` |
| [manifest-and-record-model.md](manifest-and-record-model.md) | `StorageIdentifier`, `StorageItem`, the manifest record, the per-type record updaters, `StorageServiceContact`, unknown fields / unknown identifiers | `StorageService.swift`, `StorageServiceProto+Sync.swift` |
| [encryption-and-ikm.md](encryption-and-ikm.md) | Key derivation (`MasterKey` → storage-service key → manifest/record keys), the `recordIkm` scheme, AES-256-GCM item encryption, and the `recordIkm` migrator | `StorageService.swift`, `StorageServiceRecordIkmMigrator.swift` |
| [push-pull-merge.md](push-pull-merge.md) | The backup (push), restore/create (pull), and manifest-rotation cycles end-to-end, with sequence diagrams | `StorageServiceManager.swift`, `StorageService.swift` |
| [conflict-resolution.md](conflict-resolution.md) | 409 version conflicts, merge semantics, "latest wins" philosophy, orphan re-adoption, invalid-identifier handling, decryption-failure recovery, consecutive-conflict ceiling | `StorageServiceManager.swift`, `StorageServiceProto+Sync.swift` |

---

## The four source files at a glance

| File | Role | LOC (approx) |
| --- | --- | --- |
| `StorageService.swift` | The **network/crypto layer**: HTTP requests to `/v1/storage*`, manifest/item encryption + decryption, `StorageIdentifier`/`StorageItem` value types, `ManifestRecordIkm`. **[High]** | ~560 |
| `StorageServiceManager.swift` | The **orchestration layer**: public API, the serialized `StorageServiceOperation` state machine, backup/restore/rotate/cleanup modes, the local `State` mirror, and the per-type record updaters' *wiring*. **[High]** | ~2.9k |
| `StorageServiceProto+Sync.swift` | The **record-updater layer**: for each record type, `buildRecord` (local → proto) and `mergeRecord` (proto → local), plus `StorageServiceContact` and merge-result types. **[High]** | ~2.4k |
| `StorageServiceRecordIkmMigrator.swift` | A one-shot migrator that rotates the manifest so a `recordIkm` gets generated for accounts that predate it. **[High]** | ~90 |

---

## Cross-cutting conventions observed in the code

- **Everything funnels through one serialized operation at a time.** The manager
  holds an `AtomicValue<ManagerState>` and runs at most one
  `StorageServiceOperation` concurrently; new work is queued and popped by a
  fixed precedence order (rotate → mutations → cleanup → restore →
  restore-completion → backup). **[High]** — `StorageServiceManager.swift:341-451`
  (`popNextOperation`), `:316-339` (`startNextOperationIfNeeded`).
- **A record identifier is regenerated on every change.** There is no in-place
  update of a record; a changed record gets a brand-new random
  `StorageIdentifier`, the old one is enqueued for deletion, and the manifest is
  rewritten. This is what lets a device detect "what changed" purely by set
  difference of identifiers. **[High]** —
  `StorageServiceManager.swift:973-1030` (`updateRecord`).
- **The server is the source of truth; latest-wins.** On merge, remote values
  generally overwrite local ones; local-only differences schedule a follow-up
  push (`needsUpdate`). **[High]** — `StorageServiceProto+Sync.swift:36-56`
  (protocol doc comment).
- **Unknown fields and unknown record types are preserved, never dropped**, so
  older app versions don't clobber data written by newer versions. **[High]** —
  `StorageServiceManager.swift:2335-2353` (unknown identifiers in `allIdentifiers`),
  `:1841-1875` (`mergeItems` unknown-type bucket).
- **Every operation retries up to 4 times with backoff.** **[High]** —
  `StorageServiceManager.swift:776-781` (`run` → `Retry.performWithBackoff(maxAttempts: 4)`).
- **All requests authenticate with ephemeral storage-service credentials**
  fetched via `OWSRequestFactory.storageAuthRequest`. **[High]** —
  `StorageService.swift:510-524`.

See each area document for per-type purpose, validation/business rules, error
paths, edge cases, feature flags, and diagrams.

# `Signal/Groups/` — App-level Groups support

Documentation of the **first-party `Signal/` app target's `Groups/` directory**.
This is the UI/app-level counterpart to the service-layer group machinery, which
lives in `SignalServiceKit/Groups/` and is documented separately under
[../../SignalServiceKit/Groups/](../../SignalServiceKit/Groups). This doc describes
*how the app target uses* that service layer; it does not restate its internals.

## Scope

`Signal/Groups/` currently contains a **single source file** **[High]**:

- `GroupSendEndorsementExpirationJob.swift` — a main-app expiration job that prunes
  expired group-send endorsements from the local database.

There are no view controllers, views, cell builders, or group-creation/management UI
in this directory. The large body of group-related UI (group creation, member lists,
group settings, permissions, etc.) lives elsewhere under `Signal/src/ViewControllers/`
and is out of scope for this folder-scoped doc **[Medium]** (not exhaustively verified;
inferred from the directory listing of `Signal/Groups/` containing only the one file).

## Conventions

- **Citations** are `path:line` (specific declaration) or `path` (file). Line numbers
  reflect the working tree at authoring time and may drift; relocate via the symbol
  name. Every factual claim about behavior cites a path.
- **Confidence labels**:
  - **[High]** — directly observed (file read / declaration located in this session).
  - **[Medium]** — inferred from signatures, names, and cross-file convention.
  - **[Low]** — educated inference from naming alone.

## One-paragraph orientation

The only thing in `Signal/Groups/` is a background maintenance job. Group-send
endorsements are credentials (issued by the Signal server, persisted by
`SignalServiceKit`) that let a client send to a group via sealed sender without
revealing membership; they carry an expiration date. `GroupSendEndorsementExpirationJob`
(`Signal/Groups/GroupSendEndorsementExpirationJob.swift:9`) is the app-target subclass
of the generic `ExpirationJob<ExpiringElement>` base
(`SignalServiceKit/Expiration/ExpirationJob.swift:16`) that, while the app runs,
deletes `CombinedGroupSendEndorsementRecord` rows once they expire **[High]**. It is
constructed in the main-app dependency container and started alongside the other
expiration jobs when the app activates **[High]**.

## Type

### `GroupSendEndorsementExpirationJob` (`class: ExpirationJob<CombinedGroupSendEndorsementRecord>`)

Declared at `Signal/Groups/GroupSendEndorsementExpirationJob.swift:9` **[High]**.

Stored state **[High]**:
- `groupSendEndorsementStore: GroupSendEndorsementStore` — the service-layer store that
  owns the endorsement tables.

`init(dateProvider:db:groupSendEndorsementStore:)`
(`Signal/Groups/GroupSendEndorsementExpirationJob.swift:11`) stores the store and
forwards `dateProvider`/`db` to the base `ExpirationJob` initializer with a prefixed
logger `"[GroupSendEndorsementExpJob]"` **[High]**.

It overrides the three abstract hooks of `ExpirationJob` **[High]**:

| Override | Behavior | Citation |
|---|---|---|
| `nextExpiringElement(tx:)` | Returns `groupSendEndorsementStore.fetchNextExpiringCombinedEndorsement(tx:)` — the earliest-expiring combined endorsement. | `Signal/Groups/GroupSendEndorsementExpirationJob.swift:24` |
| `expirationDate(ofElement:)` | Returns `element.expiration`. | `Signal/Groups/GroupSendEndorsementExpirationJob.swift:28` |
| `deleteExpiredElement(_:tx:)` | Calls `groupSendEndorsementStore.deleteEndorsements(groupRowId: element.groupRowId, tx:)`. | `Signal/Groups/GroupSendEndorsementExpirationJob.swift:32` |

## Interaction with the rest of the app

### Wiring (`AppEnvironment`)

The job is a stored, lazily-constructed property of the main-app dependency container:
`groupSendEndorsementExpirationJob` is declared at
`Signal/AppLaunch/AppEnvironment.swift:43` and instantiated at
`Signal/AppLaunch/AppEnvironment.swift:168` with:
- `dateProvider: Date.provider`,
- `db: DependenciesBridge.shared.db`,
- `groupSendEndorsementStore: DependenciesBridge.shared.groupSendEndorsementStore`

**[High]**. This is the standard pattern for app-target objects that depend on
`SignalServiceKit` services (see [../app-environment.md](../app-environment.md)).

### Scheduling / lifecycle (`AppLifecycleManager`)

The job is launched from `runExpirationJobs()`
(`Signal/AppLaunch/AppLifecycleManager.swift:1694`) inside a `withThrowingTaskGroup`,
as one task among the other expiration jobs:
`taskGroup.addTask { try await AppEnvironment.shared.groupSendEndorsementExpirationJob.run() }`
(`Signal/AppLaunch/AppLifecycleManager.swift:1703`) **[High]**.

`runExpirationJobs()` is itself kicked off on app **activation**: the activation path
cancels any prior `expirationJobTask`, awaits its completion, then starts a new `Task`
that calls `runExpirationJobs()` (`Signal/AppLaunch/AppLifecycleManager.swift:1629`–
`:1635`) **[High]**. `CancellationError` from the task group is treated as expected
(`Signal/AppLaunch/AppLifecycleManager.swift:1706`) **[High]**, consistent with the
jobs being torn down when the app leaves the foreground.

### Service-layer dependency (`SignalServiceKit`)

The job delegates all persistence to `GroupSendEndorsementStore`
(`SignalServiceKit/Groups/GroupSendEndorsementStore.swift:9`) and operates on
`CombinedGroupSendEndorsementRecord` (`SignalServiceKit/Groups/GroupSendEndorsementRecord.swift:9`).
Relevant store surface **[High]**:
- `fetchNextExpiringCombinedEndorsement(tx:)`
  (`SignalServiceKit/Groups/GroupSendEndorsementStore.swift:39`) returns the combined
  endorsement with the smallest `expiration` (ordered ascending, `fetchOne`).
- `deleteEndorsements(groupRowId:tx:)`
  (`SignalServiceKit/Groups/GroupSendEndorsementStore.swift:64`) deletes the combined
  record for a group by its `groupRowId` key. Because the record key is the group row
  id, deleting the combined endorsement is per-group.

Endorsements are *written* elsewhere (e.g. `saveEndorsements(...)` at
`SignalServiceKit/Groups/GroupSendEndorsementStore.swift:10`), when the client fetches
fresh endorsements from the server; that write path is part of the service layer, not
this folder **[Medium]** (observed the method exists; its callers are in
`SignalServiceKit`, documented under [../../SignalServiceKit/Groups/](../../SignalServiceKit/Groups)).

## Data flow / state

**[High]** The record model:
- `CombinedGroupSendEndorsementRecord` is a GRDB `Codable`/`FetchableRecord`/
  `PersistableRecord` backed by the `CombinedGroupSendEndorsement` table
  (`SignalServiceKit/Groups/GroupSendEndorsementRecord.swift:10`).
- Its primary key is `groupRowId` (`typealias RowId = GroupRecord.RowId`,
  `SignalServiceKit/Groups/GroupSendEndorsementRecord.swift:12`), it stores the opaque
  serialized `endorsement: Data`, and an `expiration: Date`
  (`:14`–`:16`). The `expiration` is persisted as a `UInt64` seconds-since-epoch stored
  bit-for-bit in an `Int64` column (`encode(to:)` at `:34`; `init(from:)` at `:44`).

The expiration loop (inherited from `ExpirationJob`) **[High]**:
1. `run()` (`SignalServiceKit/Expiration/ExpirationJob.swift:77`) asserts single-run,
   registers for `UIApplication.significantTimeChangeNotification` → `restart()`
   (`:88`), and loops.
2. Each iteration calls `deleteExpiredElements()`
   (`SignalServiceKit/Expiration/ExpirationJob.swift:133`), which uses
   `TimeGatedBatch.processAll` to repeatedly fetch `nextExpiringElement`, delete it if
   `dateProvider() >= expirationDate(ofElement:)`, and otherwise stop — returning the
   next element's expiration date so the loop can sleep until then.
3. The loop then waits at least `minIntervalBetweenDeletes` (default `1` second,
   `SignalServiceKit/Expiration/ExpirationJob.swift:35`) and sleeps until the next
   expiration, waking early if `restart()` bumps the `delayValidityToken` or cancels
   the delay task.
4. `restart()` (`SignalServiceKit/Expiration/ExpirationJob.swift:69`) is the signal
   that the underlying store changed; callers that save new endorsements should invoke
   it so a newly-added earlier expiration is honored. **[Medium]** This job does not
   call `restart()` itself (it only overrides the fetch/date/delete hooks); the base
   class documents that the store's writers are responsible
   (`SignalServiceKit/Expiration/ExpirationJob.swift:61`).

## UI considerations

**[High]** This directory contains **no UI**. The job runs entirely in the background
as an `async` task driven by the app lifecycle; it presents nothing, has no view
controllers, and has no user-facing surface. Its only observable effect is the eventual
deletion of expired endorsement rows from the local GRDB database.

## Notes / caveats

- **[Medium]** The job assumes exclusive `run()` per instance (the base class
  `owsPrecondition(!isRunning)` at `SignalServiceKit/Expiration/ExpirationJob.swift:80`).
  The activation path guarantees this by cancelling and awaiting the previous
  `expirationJobTask` before starting a new one
  (`Signal/AppLaunch/AppLifecycleManager.swift:1629`).
- **[Low]** The file header is `Copyright 2026`, suggesting this app-level job is a
  relatively recent addition; the backing store/record is `Copyright 2024`. Intent of
  the split (why this job lives in the app target rather than `SignalServiceKit`
  alongside its peers) — undetermined; no evidence in source. The observable fact is
  that it is wired via `AppEnvironment` and run from `AppLifecycleManager` next to the
  `DependenciesBridge`-owned expiration jobs
  (`Signal/AppLaunch/AppLifecycleManager.swift:1699`–`:1704`) **[High]**.

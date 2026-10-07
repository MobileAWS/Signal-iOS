# `Signal/Contacts/` — Contact Name Collisions & Author Column Migration

Documentation of the first-party `Signal/Contacts/` directory of the main app
target. Despite the name, this folder is **not** the general contacts/system-contacts
integration layer (that lives in `SignalServiceKit` and `SignalUI`); it is a small,
two-file subsystem focused on two unrelated contact-adjacent concerns:

1. **Name-collision detection** — surfacing cases where two different accounts share
   a display name (a profile-spoofing / message-request safety feature).
2. **Author-column background migration** — rewriting legacy phone-number author
   columns to ACIs in the background.

This doc treats `SignalServiceKit/` and `SignalUI/` as **boundaries**: it documents
*how this folder uses them* and *how the app consumes this folder*, not their
internals. For the main-app target overview, see [../README.md](../README.md); for the
whole-repository map, see [../../APPLICATION_MAP.md](../../APPLICATION_MAP.md).

## Scope

The directory contains exactly two source files (working tree, this session) **[High]**:

- `Signal/Contacts/NameCollisionFinder.swift` — the `NameCollision` model, the
  `NameCollisionFinder` protocol, and its two implementations.
- `Signal/Contacts/AuthorMergeHelperBuilder.swift` — a background builder that
  populates the author-ACI lookup table.

Related tests live outside this folder at
`Signal/test/Contacts/AuthorMergeHelperBuilderTest.swift`.

## Conventions

- **Citations** are `path:line` (specific declaration) or `path` (file). Line numbers
  reflect the working tree at authoring time and may drift; relocate via the symbol
  name. Every factual claim about behavior cites a path.
- **Confidence labels** on claims about *purpose/behavior*:
  - **[High]** — directly observed (file read / declaration located in this session).
  - **[Medium]** — inferred from signatures, names, and cross-file convention; not
    every referenced file was read in full.
  - **[Low]** — educated inference from naming alone.
- Any uncited claim is a defect. Where the source gives no evidence of design intent,
  the text says so.

## One-paragraph orientation

The two files in this folder solve independent problems. `NameCollisionFinder.swift`
defines a safety feature: it finds sets of accounts in a thread whose *display names*
collide, so the app can warn the user about possible impersonation
(`Signal/Contacts/NameCollisionFinder.swift:8`) **[High]**. Two concrete finders exist
— one for 1:1 contact threads (`ContactThreadNameCollisionFinder`,
`Signal/Contacts/NameCollisionFinder.swift:51`) and one for group threads
(`GroupMembershipNameCollisionFinder`,
`Signal/Contacts/NameCollisionFinder.swift:138`) — both conforming to the
`NameCollisionFinder` protocol (`Signal/Contacts/NameCollisionFinder.swift:39`)
**[High]**. `AuthorMergeHelperBuilder.swift` is a one-time-ish background maintenance
task that walks author tables and replaces phone-number authorship with ACIs
(`Signal/Contacts/AuthorMergeHelperBuilder.swift:10`), launched after app-ready from
`AppLifecycleManager` (`Signal/AppLaunch/AppLifecycleManager.swift:672`) **[High]**.

---

## 1. Name collision detection

### 1.1 The `NameCollision` model

`NameCollision` is an immutable value type holding **two or more** colliding elements;
its failable initializer returns `nil` for fewer than two elements, so a `NameCollision`
always represents a real collision (`Signal/Contacts/NameCollisionFinder.swift:31`)
**[High]**. Each `NameCollision.Element` carries the colliding account's
`SignalServiceAddress`, a `ComparableDisplayName`, and — when a recent profile-name
change is implicated — a `(oldestProfileName, newestProfileName)` tuple plus the
`latestUpdateTimestamp` of the triggering update
(`Signal/Contacts/NameCollisionFinder.swift:12`) **[High]**. The `Element` initializer
is `fileprivate`, so elements are only built inside the finders in this file
(`Signal/Contacts/NameCollisionFinder.swift:18`) **[High]**.

### 1.2 The `NameCollisionFinder` protocol

The protocol exposes a `thread`, `findCollisions(transaction:)` returning
`[NameCollision]`, and `markCollisionsAsResolved(transaction:)`
(`Signal/Contacts/NameCollisionFinder.swift:39`) **[High]**. "Resolving" is a
finder-specific notion of recording that the user has acknowledged the current
collisions so they are not re-shown.

### 1.3 `ContactThreadNameCollisionFinder` (1:1 threads)

Compares the contact-thread recipient against all known Signal accounts
(`Signal/Contacts/NameCollisionFinder.swift:51`) **[High]**. Key behaviors:

- Built via the static factory `makeToCheckMessageRequestNameCollisions(forContactThread:)`,
  which sets `onlySearchIfMessageRequest = true`
  (`Signal/Contacts/NameCollisionFinder.swift:62`) **[High]**. When that flag is set and
  the thread has no pending message request, `findCollisions` early-returns empty
  (`Signal/Contacts/NameCollisionFinder.swift:83`) **[High]**.
- The candidate set is the union of all whitelisted registered addresses
  (`profileManagerRef.allWhitelistedRegisteredAddresses`) and all `SignalAccount`
  recipient addresses — the latter included deliberately "to ensure we check against
  blocked system contact names" — with the target and the local address removed
  (`Signal/Contacts/NameCollisionFinder.swift:89`) **[High]**. This is the one place in
  this folder that touches **system-contact-derived data** (`SignalAccount` rows mirror
  the system address book); see §3.
- Comparison is by `ComparableDisplayName.resolvedValue()` under a `.current()`
  `DisplayName.ComparableValue.Config`; accounts without a known name are skipped
  (`Signal/Contacts/NameCollisionFinder.swift:102`) **[High]**.
- `markCollisionsAsResolved` is a no-op: contact threads always display all collisions
  (`Signal/Contacts/NameCollisionFinder.swift:132`) **[High]**.

### 1.4 `GroupMembershipNameCollisionFinder` (group threads)

Builds a `displayName → [addresses]` map over the group members and reports any name
shared by ≥2 members (`Signal/Contacts/NameCollisionFinder.swift:155`) **[High]**. It is
more stateful than the contact-thread finder:

- It caches "recent profile update messages" per address, guarded by an `UnfairLock`,
  fetched once per finder lifetime and documented as thread-safe
  (`Signal/Contacts/NameCollisionFinder.swift:145`) **[High]**.
- Only collisions that include at least one **recently changed** name are reported;
  each reported element is annotated with the oldest→newest profile name and the newest
  update timestamp, so the UI can phrase "X changed their name from … to …"
  (`Signal/Contacts/NameCollisionFinder.swift:200`) **[High]**.
- "Recent" is bounded by a per-thread `recentProfileUpdateSearchStartId`, read/written
  through `GroupMembershipNameCollisionFinderStore`
  (`Signal/Contacts/NameCollisionFinder.swift:243`) **[High]**. That store lives in
  `SignalServiceKit` (`SignalServiceKit/Contacts/GroupMembershipNameCollisionFinderStore.swift:8`)
  and persists to a `NewKeyValueStore` collection `"GroupThreadCollisionFinder"`,
  always writing `max(new, existing)` so the search window never shrinks
  (`SignalServiceKit/Contacts/GroupMembershipNameCollisionFinderStore.swift:18`) **[High]**.
  The store also conforms to `ThreadRemoverObserver` and clears its entry when a thread
  is removed (`SignalServiceKit/Contacts/GroupMembershipNameCollisionFinderStore.swift:23`)
  **[High]**.
- `markCollisionsAsResolved` advances the stored start id past the newest cached update
  message and clears the cache
  (`Signal/Contacts/NameCollisionFinder.swift:262`) **[High]**. A perf optimization,
  `setRecentProfileUpdateSearchStartIdToMax`, proactively advances the start id to the
  thread's `lastInteractionRowId` via an async write when there are no collisions
  (`Signal/Contacts/NameCollisionFinder.swift:278`) **[High]**.

### 1.5 Sorting helpers

Private extensions provide a `standardSort`: elements within a collision sort by most
recent profile update (then stably by service id / phone number), and collision arrays
sort by their first element's `ComparableDisplayName`
(`Signal/Contacts/NameCollisionFinder.swift:321`) **[High]**.

### 1.6 Data dependencies

The finders read exclusively through `SSKEnvironment.shared` and `DependenciesBridge`
accessors — `contactManagerRef.displayName(s)(for:tx:)`,
`profileManagerRef`, `databaseStorageRef`, `InteractionFinder`,
`TSContactThread`/`TSGroupThread`, and `ComparableDisplayName`
(`Signal/Contacts/NameCollisionFinder.swift:116`,
`Signal/Contacts/NameCollisionFinder.swift:165`,
`Signal/Contacts/NameCollisionFinder.swift:243`) **[High]**. All collision computation
runs inside read transactions supplied by the caller; writes (resolving) go through
`databaseStorageRef.asyncWrite` (`Signal/Contacts/NameCollisionFinder.swift:291`)
**[High]**.

### 1.7 Consumers in the app

- **Conversation banners.** `ConversationViewController+Banners.swift` constructs a
  `ContactThreadNameCollisionFinder` for message-request collision banners
  (`Signal/ConversationView/ConversationViewController+Banners.swift:768`) and a
  `GroupMembershipNameCollisionFinder` for group banners
  (`Signal/ConversationView/ConversationViewController+Banners.swift:818`) **[High]**.
  The group finder is cached on the conversation's view state as
  `CVViewState.groupNameCollisionFinder` so it persists across banner refreshes
  (`Signal/ConversationView/CVViewState.swift:29`) **[High]**.
- **Resolution screen.** `NameCollisionResolutionViewController` takes a
  `NameCollisionFinder` and renders its collisions as table cells; when the model
  collapses to ≤1 element per collision it calls `markCollisionsAsResolved` and asks its
  delegate to dismiss (`Signal/src/ViewControllers/ThreadSettings/NameCollisionResolutionViewController.swift:28`,
  `Signal/src/ViewControllers/ThreadSettings/NameCollisionResolutionViewController.swift:67`)
  **[High]**. The banner handler presents this controller
  (`Signal/ConversationView/ConversationViewController+Banners.swift:794`) **[High]**.
- **Cell rendering** uses `NameCollisionReviewCell`
  (`Signal/src/views/NameCollisionReviewCell.swift`) **[Medium]** — referenced by the
  resolution controller; file not read in full this session.

Data flow (collision path), all **[High]** unless noted:

```
ConversationViewController+Banners
  └─ make finder (ContactThread… or GroupMembership…)
       └─ finder.findCollisions(tx)            // reads contactManager / profileManager / interactions
            └─ [NameCollision] (≥2 elements each)
  └─ present NameCollisionResolutionViewController(collisionFinder:)
       └─ updateModel() → findCollisions(tx) → collisionCellModels(...)   [Medium: cellModel mapping not read]
       └─ user resolves → markCollisionsAsResolved(tx)  // group: advances stored startId; contact: no-op
```

---

## 2. `AuthorMergeHelperBuilder` — background author-column migration

`AuthorMergeHelperBuilder` is a `final class` that incrementally rewrites author
columns in legacy tables, replacing phone numbers with ACIs
(`Signal/Contacts/AuthorMergeHelperBuilder.swift:10`) **[High]**. It is dependency-injected
with an `AppContext`, an `AuthorMergeHelper`, a `DB`, a `ModelReadCaches` shim, and a
`RecipientDatabaseTable` (`Signal/Contacts/AuthorMergeHelperBuilder.swift:17`) **[High]**.

Behavior:

- **Entry point** `buildTableIfNeeded()` wraps `_buildTableIfNeeded()` and only logs
  on failure (`Signal/Contacts/AuthorMergeHelperBuilder.swift:39`) **[High]**.
- **Version gate.** It compares `authorMergeHelper.currentVersion` vs `nextVersion`
  and does nothing if already finished; otherwise it processes every
  `AuthorDatabaseTable.all` in batches, then records completion via
  `setCurrentVersion` (`Signal/Contacts/AuthorMergeHelperBuilder.swift:47`) **[High]**.
- **Batching for responsiveness.** Each batch runs only while the app is active
  (`AppActivePrecondition`) under an `OWSBackgroundTask`
  (`Signal/Contacts/AuthorMergeHelperBuilder.swift:68`) **[High]**. Batch size is
  time-bounded: it estimates read vs. write cost (`writeFactor = 50`,
  `estimatedBatchDuration = 0.5s`) and stops the current batch when the projected total
  exceeds the budget (`Signal/Contacts/AuthorMergeHelperBuilder.swift:33`,
  `Signal/Contacts/AuthorMergeHelperBuilder.swift:95`) **[High]**. Between batches it
  sleeps one second (`Signal/Contacts/AuthorMergeHelperBuilder.swift:58`) **[High]**.
- **Read-then-write discipline.** Rows are collected via a `Row.fetchCursor` SELECT,
  and updates are applied *after* the cursor is drained to avoid mutating the table
  mid-SELECT (`Signal/Contacts/AuthorMergeHelperBuilder.swift:104`,
  `Signal/Contacts/AuthorMergeHelperBuilder.swift:149`) **[High]**. Progress resumes
  from a stored `nextRowId` keyed by table name
  (`Signal/Contacts/AuthorMergeHelperBuilder.swift:125`) **[High]**.
- **Resolution logic** lives in `AuthorMergeHelperBuilderBatch.processRow`: if a row has
  no phone number it's skipped; if it already has an ACI the phone number is cleared; if
  not, it tries `RecipientDatabaseTable.fetchRecipient(phoneNumber:)` (cached) to find
  the ACI, otherwise records the phone number as missing an ACI
  (`Signal/Contacts/AuthorMergeHelperBuilder.swift:169`) **[High]**.
- After each batch it evacuates all model read caches via the shim
  (`Signal/Contacts/AuthorMergeHelperBuilder.swift:119`) **[High]**.

**Shims / testability.** `ModelReadCaches` is abstracted behind a shim/wrapper pair so
tests can substitute a mock (`Signal/Contacts/AuthorMergeHelperBuilder.swift:202`), with
a `TESTABLE_BUILD`-gated mock at the bottom of the file
(`Signal/Contacts/AuthorMergeHelperBuilder.swift:231`) **[High]**. The unit test lives at
`Signal/test/Contacts/AuthorMergeHelperBuilderTest.swift:13` and constructs the builder
with the mock caches (`Signal/test/Contacts/AuthorMergeHelperBuilderTest.swift:42`)
**[High]**.

**Wiring.** `AppLifecycleManager` kicks off the builder on a detached low-priority task
after app-ready, injecting `DependenciesBridge.shared.authorMergeHelper`, the
`ModelReadCaches` wrapper around `SSKEnvironment.shared.modelReadCachesRef`, and
`DependenciesBridge.shared.recipientDatabaseTable`
(`Signal/AppLaunch/AppLifecycleManager.swift:672`) **[High]**. The underlying
`AuthorMergeHelper` is owned by the service layer (referenced from
`SignalServiceKit/Environment/AppSetup.swift` and
`SignalServiceKit/Storage/Database/GRDBSchemaMigrator.swift`) — this builder is the
*main-app* driver that fills the table opportunistically rather than during a blocking
migration (`Signal/AppLaunch/AppLifecycleManager.swift:672`) **[Medium]**;
those service-layer files were not read in full this session.

---

## 3. System-contacts integration note

This folder does **not** implement system address-book access, permissions, or the
`CNContact` UI. It touches system-contact-derived data only indirectly: the
contact-thread finder folds `SignalAccount` rows (which mirror matched system contacts)
into its candidate set specifically so blocked **system-contact names** are considered
when detecting collisions (`Signal/Contacts/NameCollisionFinder.swift:94`) **[High]**.
The resolution UI imports `ContactsUI` and registers with
`SUIEnvironment.shared.contactsViewHelperRef`
(`Signal/src/ViewControllers/ThreadSettings/NameCollisionResolutionViewController.swift:6`,
`Signal/src/ViewControllers/ThreadSettings/NameCollisionResolutionViewController.swift:46`)
**[High]**, but that controller lives outside this directory and the general
contacts-picker machinery lives in `SignalUI` (see
[../../SignalUI/RecipientPickers.md](../../SignalUI/RecipientPickers.md)) and
`SignalServiceKit` — out of scope here. Display-name resolution itself (including whether
a name comes from a system contact, profile, or username) is delegated entirely to
`contactManagerRef`/`ComparableDisplayName` in `SignalServiceKit` **[Medium]**.

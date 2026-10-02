# `SSKEnvironment.swift` & `DependenciesBridge.swift` — The Two Global Containers

Signal iOS keeps its services in **two** process-global containers. This split
is historical: `SSKEnvironment` is the legacy ObjC-friendly container, and
`DependenciesBridge` is the newer Swift-only replacement. Both are populated in
one place — `AppSetup.initGlobals` (see
[bootstrap-appsetup.md](bootstrap-appsetup.md)).

- [1. `SSKEnvironment` — the legacy container](#1-sskenvironment--the-legacy-container)
- [2. `warmCaches` and self-repair logic](#2-warmcaches-and-self-repair-logic)
- [3. `DependenciesBridge` — the modern container](#3-dependenciesbridge--the-modern-container)
- [4. Which container holds what](#4-which-container-holds-what)
- [5. Error paths & edge cases](#5-error-paths--edge-cases)
- [6. File reference checklist](#6-file-reference-checklist)

---

## 1. `SSKEnvironment` — the legacy container

`public class SSKEnvironment: NSObject` (`SSKEnvironment.swift:8`). **[High]**

### 1.1 The singleton accessor

```swift
private static var _shared: SSKEnvironment?
public static var hasShared: Bool { _shared != nil }
@objc public static var shared: SSKEnvironment { _shared! }   // force-unwrap
public static func setShared(_ env: SSKEnvironment?, isRunningTests: Bool) {
    owsPrecondition((_shared == nil && env != nil) || isRunningTests)
    _shared = env
}
```
(`SSKEnvironment.swift:10-20`). **[High]**

- `shared` is a **force-unwrap**; accessing it before `AppSetup` installs the
  env crashes. `hasShared` is the safe probe.
- `setShared` is guarded by a precondition: the only legal transition outside
  tests is `nil → non-nil`. Under `isRunningTests` the harness may replace it
  freely. **[High]**

### 1.2 Property shape: `…Ref` suffix + ObjC bridges

Every held service is exposed as a `…Ref` property. In a `TESTABLE_BUILD`, four
of them are `private(set) var` so tests can swap them; otherwise they are `let`
(`SSKEnvironment.swift:23-32`): **[High]**

```swift
#if TESTABLE_BUILD
    public private(set) var contactManagerRef: any ContactManager
    public private(set) var networkManagerRef: NetworkManager
    public private(set) var paymentsHelperRef: PaymentsHelperSwift
    public private(set) var groupsV2Ref: GroupsV2
#else
    public let contactManagerRef: any ContactManager
    …
#endif
```

The matching `set…ForUnitTests(_:)` methods exist only under `TESTABLE_BUILD`
(`SSKEnvironment.swift:314-...`). **[High]**

Several properties also expose **downcasting / ObjC-visible aliases**: **[High]**

| Alias | Backed by | Note | Citation |
| --- | --- | --- | --- |
| `contactManagerImplRef: OWSContactsManager` | `contactManagerRef as! OWSContactsManager` | "should be deprecated"; force-cast | `SSKEnvironment.swift:33` |
| `contactManagerObjcRef: ContactsManagerProtocol` | `contactManagerRef` | `@objc` | `:34-35` |
| `profileManagerImplRef: OWSProfileManager` | `profileManagerRef as! OWSProfileManager` | "should be deprecated"; force-cast | `:40` |

> **[High]** The `as!` force-casts assume the real implementation types are
> installed. In tests that inject a different mock these aliases would crash;
> that's the documented tradeoff of "should be deprecated".

### 1.3 The ~60 held services

`SSKEnvironment` holds ~60 services (full list `SSKEnvironment.swift:38-96`),
including: `contactManagerRef`, `networkManagerRef`, `paymentsHelperRef`,
`groupsV2Ref`, `profileManagerRef`, `messageReceiverRef`, `blockingManagerRef`,
`remoteConfigManagerRef`, `udManagerRef`, `messageDecrypterRef`,
`ows2FAManagerRef`, `receiptManagerRef`/`receiptSenderRef`,
`reachabilityManagerRef`, `syncManagerRef`, `typingIndicatorsRef`,
`stickerManagerRef`, `databaseStorageRef`, `signalServiceAddressCacheRef`,
`signalServiceRef`, `storageServiceManagerRef`, `sskPreferencesRef`,
`groupV2UpdatesRef`, `messageFetcherJobRef`, `versionedProfilesRef`,
`modelReadCachesRef`, `earlyMessageManagerRef`, `messagePipelineSupervisorRef`,
`messageProcessorRef`, the payments cluster, `phoneNumberUtilRef`,
`webSocketFactoryRef`, `systemStoryManagerRef`, `contactDiscoveryManagerRef`,
`notificationPresenterRef`, `messageSendLogRef`, `preferencesRef`,
`proximityMonitoringManagerRef`, `avatarBuilderRef`, `smJobQueuesRef`,
`groupCallManagerRef`, `profileFetcherRef`, plus five job queues and three
`private` fields (`appExpiryRef`, `aciSignalProtocolStoreRef`,
`pniSignalProtocolStoreRef`). **[High]**

### 1.4 `signalProtocolStoreRef(for:)`

Dispatches the ACI vs PNI protocol store (`SSKEnvironment.swift:217-224`): **[High]**

```swift
public func signalProtocolStoreRef(for identity: OWSIdentity) -> SignalProtocolStore {
    switch identity {
    case .aci: return aciSignalProtocolStoreRef
    case .pni: return pniSignalProtocolStoreRef
    }
}
```

---

## 2. `warmCaches` and self-repair logic

`@MainActor func warmCaches(appReadiness:dependenciesBridge:)`
(`SSKEnvironment.swift:232-246`). Called by `AppSetup.FinalContinuation`. The
header comment stresses **"All of these methods must be safe to invoke
repeatedly"** — re-warming keeps the NSE in sync with the main app. **[High]**

Sequence: **[High]**

1. `verifyPniAndPniIdentityKey(dependenciesBridge:)`
2. `fixLocalRecipientIfNeeded(dependenciesBridge:)`
3. `SignalProxy.warmCaches(appReadiness:)`
4. `signalServiceRef.warmCaches()`
5. `profileManagerRef.warmCaches()`
6. `typingIndicatorsRef.warmCaches()`
7. `paymentsHelperRef.warmCaches()`
8. `paymentsCurrenciesRef.warmCaches()`
9. `StoryManager.setup(appReadiness:)`
10. `db.read { appExpiryRef.warmCaches(with: tx) }`

### 2.1 `verifyPniAndPniIdentityKey` — a deregistration self-repair

(`SSKEnvironment.swift:248-280`). **[High]**

- Only runs for a **registered** account that has a phone number
  (`registeredStateWithMaybeSneakyTransaction()` with a non-nil `phoneNumber`);
  otherwise returns.
- Computes `hasPni` (from `LocalIdentifiers`) and `hasPniIdentityKey` (from
  `identityManager.identityKeyPair(for: .pni, tx:)`).
- **If either is missing**, it logs a warning and **force-deregisters** the
  device: `registrationStateChangeManager.setIsDeregisteredOrDelinked(true,
  notify: true, tx:)` in a write tx. **[High]**

> **[High] Edge case / destructive behavior.** This is a self-protective
> *local* deregistration: a registered account lacking its PNI identity key is
> considered corrupt, and the client takes itself out of the registered state
> rather than operate with inconsistent key material. This runs on **every**
> cache warm, so it is a continuously-enforced invariant.

### 2.2 `fixLocalRecipientIfNeeded` — one-time local-recipient migration

(`SSKEnvironment.swift:282-310`). **[High]**

- No-op if not registered (`localIdentifiers(tx:) == nil`), or if registered
  with an invalid phone number (`E164(phoneNumber) == nil`).
- Otherwise calls `recipientMerger.applyMergeForLocalAccount(aci:phoneNumber:pni:
  shouldUpdateStorageService: true, tx:)` to ensure the local `SignalRecipient`
  carries its own PNI, then `blockedRecipientStore.setBlocked(false,
  recipientId:, tx:)` (you can never block yourself). **[High]**

---

## 3. `DependenciesBridge` — the modern container

`public class DependenciesBridge` (`DependenciesBridge.swift:26`). The 18-line
doc comment (`DependenciesBridge.swift:8-25`) is explicit about intent: **[High]**

> *"Temporary bridge between [legacy code that uses global accessors] and [new
> code that expects references passed around]. … It is preferred **NOT** to use
> this class, and to take dependencies on init instead, but it is better to use
> this class than to use `Dependencies`."*

Advantages it lists over the old `Dependencies` protocol/extension pattern:
(1) explicit `.shared` access rather than ambient members; (2) Swift-only, no
`@objc`; (3) members are themselves expected to be protocolized, inject their
own deps, and avoid global state. **[High]**

### 3.1 Singleton accessor

```swift
public static var shared: DependenciesBridge {
    guard let _shared else { owsFail("DependenciesBridge has not yet been set up!") }
    return _shared
}
private static var _shared: DependenciesBridge?
static func setShared(_ dependenciesBridge: DependenciesBridge?, isRunningTests: Bool) {
    owsPrecondition((_shared == nil && dependenciesBridge != nil) || isRunningTests)
    Self._shared = dependenciesBridge
}
```
(`DependenciesBridge.swift:28-48`). Same install contract as `SSKEnvironment`
(`nil → non-nil`, test escape hatch), but `shared` fails with an explanatory
`owsFail` message rather than a bare `!`. A `hasShared` probe exists only under
`TESTABLE_BUILD` (`:39-43`). **[High]**

### 3.2 Members: all `let`, mostly `public`

Unlike `SSKEnvironment`, there is **no** `TESTABLE_BUILD`-conditional mutability;
all ~135 stored properties are `let` (`DependenciesBridge.swift:49-...`).
A handful are package-internal (no `public`): `blockedRecipientStore`,
`deletedCallRecordStore`, `deleteForMeIncomingSyncMessageManager`,
`incomingCallEventSyncMessageManager`, `incomingCallLogEventSyncMessageManager`,
`localProfileChecker`, `pinnedThreadMerger`, `pinnedThreadStore`,
`senderKeySendingManager`. **[High]**

### 3.3 The `pinnedThreadManager` double-binding

The initializer takes a single `pinnedThreadManager: any PinnedThreadManager &
PinnedThreadMerger` and assigns it to **two** properties — `pinnedThreadManager`
and `pinnedThreadMerger` (`DependenciesBridge.swift:...`): **[High]**

```swift
self.pinnedThreadManager = pinnedThreadManager
self.pinnedThreadMerger = pinnedThreadManager
```

So the same object is reachable under both protocol lenses.

### 3.4 Notable members

- `db: any DB` and `databaseChangeObserver` — the DB abstraction used throughout
  SSK (distinct from `SSKEnvironment.databaseStorageRef`, which is the concrete
  `SDSDatabaseStorage`). **[High]**
- `libsignalNet: LibSignalClient.Net` — the shared LibSignal networking handle
  (also held transitively by many services). **[High]**
- `deviceSleepManager: (any DeviceSleepManager)?` — one of the few **optional**
  members (nil in contexts that can't sleep-manage, e.g. extensions). **[Medium]**
- A large set of `…ExpirationJob` / `…JobRunner` / store types — see the
  checklist in [bootstrap-appsetup.md](bootstrap-appsetup.md) for how each is
  constructed.

---

## 4. Which container holds what

Both containers are populated from the same `initGlobals` locals, but the
division is roughly: **[Medium]**

| Container | Tends to hold | Why |
| --- | --- | --- |
| `SSKEnvironment` | Older ObjC-era singletons, `@objc`-exposed managers, concrete `SDSDatabaseStorage` | Legacy call sites and ObjC interop | 
| `DependenciesBridge` | Newer Swift-only, protocolized managers/stores/jobs, the `any DB` abstraction, `libsignalNet` | Modern init-injection style |

Some services appear in **both** (e.g. `appExpiry` is a private field in
`SSKEnvironment` *and* a public member of `DependenciesBridge`; the two protocol
stores are in `SSKEnvironment` while `signalProtocolStoreManager` is in
`DependenciesBridge`). **[High]**

> *Intent of the exact per-service placement is undetermined — no evidence in
> source beyond the general "legacy vs modern" framing in the
> `DependenciesBridge` doc comment.*

---

## 5. Error paths & edge cases

| Situation | Handling | Citation |
| --- | --- | --- |
| Access `SSKEnvironment.shared` before setup | force-unwrap crash | `SSKEnvironment.swift:14` |
| Access `DependenciesBridge.shared` before setup | `owsFail` with message | `DependenciesBridge.swift:29-33` |
| `setShared` second non-test call | `owsPrecondition` crash | `SSKEnvironment.swift:17`, `DependenciesBridge.swift:45` |
| `contactManagerImplRef`/`profileManagerImplRef` when a non-impl mock is installed | `as!` force-cast crash | `SSKEnvironment.swift:33,40` |
| Registered account missing PNI / PNI identity key | **self-deregisters** on every warmCaches | `SSKEnvironment.swift:264-273` |
| Local account with invalid persisted phone number | `fixLocalRecipientIfNeeded` returns early | `SSKEnvironment.swift:293` |

---

## 6. File reference checklist

| Symbol | File:line |
| --- | --- |
| `SSKEnvironment` class | `SSKEnvironment.swift:8` |
| `shared` / `hasShared` / `setShared` | `SSKEnvironment.swift:10-20` |
| `…Ref` properties (TESTABLE_BUILD vars) | `SSKEnvironment.swift:23-96` |
| `signalProtocolStoreRef(for:)` | `SSKEnvironment.swift:217-224` |
| `warmCaches(appReadiness:dependenciesBridge:)` | `SSKEnvironment.swift:232-246` |
| `verifyPniAndPniIdentityKey(...)` | `SSKEnvironment.swift:248-280` |
| `fixLocalRecipientIfNeeded(...)` | `SSKEnvironment.swift:282-310` |
| `set…ForUnitTests(_:)` | `SSKEnvironment.swift:314-...` |
| `DependenciesBridge` class + doc | `DependenciesBridge.swift:8-26` |
| `shared` / `setShared` / `hasShared` | `DependenciesBridge.swift:28-48` |
| ~135 `let` members | `DependenciesBridge.swift:49-...` |
| `pinnedThreadManager` double-binding | `DependenciesBridge.swift:...` (assignment block) |

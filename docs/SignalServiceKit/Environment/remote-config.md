# `RemoteConfigManager.swift` — Remote Config & Feature Flags

This file implements Signal iOS's **server-driven configuration**: a dictionary
of string-keyed flags fetched from the service, persisted locally, merged with
hot-swap rules, cached in memory, and exposed as strongly-typed accessors on
`RemoteConfig`. It also decodes the **client-expiration** policy and pushes a
subset into LibSignal.

- [1. Three layers: `RemoteConfig`, provider, manager](#1-three-layers-remoteconfig-provider-manager)
- [2. Flag categories & the `FlagType` protocol](#2-flag-categories--the-flagtype-protocol)
- [3. Hot-swappable vs. non-hot-swappable merge semantics](#3-hot-swappable-vs-non-hot-swappable-merge-semantics)
- [4. Fetch / refresh lifecycle](#4-fetch--refresh-lifecycle)
- [5. Client expiration](#5-client-expiration)
- [6. Persistence (`RemoteConfigStore` + KeyValueStore)](#6-persistence-remoteconfigstore--keyvaluestore)
- [7. Country-code bucketing](#7-country-code-bucketing)
- [8. Enumerated flags & remote-config keys](#8-enumerated-flags--remote-config-keys)
- [9. Error paths & edge cases](#9-error-paths--edge-cases)
- [10. File reference checklist](#10-file-reference-checklist)

---

## 1. Three layers: `RemoteConfig`, provider, manager

```mermaid
graph TD
    server[(Signal service<br/>GET v2/config)]
    mgr[RemoteConfigManagerImpl<br/>:928 — fetch loop, HTTP, expiration]
    prov[RemoteConfigProviderImpl<br/>:889 — cache + DB warm]
    cfg[RemoteConfig<br/>:10 — typed accessors over valueFlags]
    kv[(KeyValueStore<br/>collection RemoteConfigManager)]
    net[libsignalNet.setRemoteConfig]

    server --> mgr
    mgr -->|awaitableWrite| kv
    mgr -->|updateCachedConfig| prov
    prov --> cfg
    kv -->|warmCaches/loadValueFlags| prov
    mgr -->|netConfig| net
    cfg -. RemoteConfig.current .-> callers[app code]
```

| Type | Responsibility | Citation |
| --- | --- | --- |
| `RemoteConfig` | Immutable snapshot wrapping `valueFlags: [String:String]` + `lastKnownClockSkew`; exposes ~50 typed computed properties | `RemoteConfigManager.swift:10-54` |
| `RemoteConfigProvider` (protocol) | `currentConfig()` + `warmCaches(tx:)` | `:882-885` |
| `RemoteConfigProviderImpl` | Owns the atomic in-memory `cachedConfig`; warms from DB | `:889-...` |
| `RemoteConfigManager` (protocol) | adds `refreshIfNeeded()` / `forceRefresh()` async | `:...` |
| `RemoteConfigManagerImpl` | HTTP fetch, backoff loop, persistence, LibSignal push, client expiration | `:928-...` |

`RemoteConfig.current` is a static convenience reaching through
`SSKEnvironment.shared.remoteConfigManagerRef.currentConfig()`
(`RemoteConfigManager.swift:12-14`). **[High]**

The `emptyConfig` fallback (`clockSkew: 0, valueFlags: [:]`) is returned when the
cache hasn't been warmed yet (`:56-58`, `currentConfig()` at `:...`). **[High]**

---

## 2. Flag categories & the `FlagType` protocol

All flag enums conform to `private protocol FlagType: CaseIterable { var
isHotSwappable: Bool { get } }` (`RemoteConfigManager.swift:873-879`). There are
three categories, each a `String`-raw-value enum whose raw value **is the
server key**: **[High]**

| Enum | Semantics | Citation |
| --- | --- | --- |
| `IsEnabledFlag` | Boolean flags; `isEnabled` parses `"1"/"true"/"TRUE"` as true, everything else false | `:701-754` |
| `ValueFlag` | Arbitrary string values parsed into Int/UInt/Double/regions/etc. | `:756-857` |
| `TimeGatedFlag` | Value is an epoch; flag "becomes true" once (clock-skew-corrected) now ≥ threshold. Currently only `.__none` (no real time-gated flags) | `:859-871` |

The boolean reader (`:658-667`), the time-gated reader (`:669-681`), and the raw
value reader (`:683-685`) all live on `RemoteConfig`. **[High]**

> **[High]** `IsEnabledFlag` default-false means a **missing** key reads as
> disabled; many public accessors invert this to express "kill switches" (e.g.
> `shouldCheckForServiceExtensionFailures = !isEnabled(.serviceExtensionFailureKillSwitch)`,
> `:308-310`).

---

## 3. Hot-swappable vs. non-hot-swappable merge semantics

Each flag declares `isHotSwappable`. The distinction (`FlagType` comment,
`:873-878`): **[High]**

- **Hot-swappable** — updates while the app is running, as soon as a fresh
  config is fetched; no restart needed.
- **Non-hot-swappable** — the new value is persisted but the *in-memory* value
  is **frozen at launch** until the next app restart.

`RemoteConfig.merging(newValueFlags:newClockSkew:)` (`:66-85`) implements this:
**[High]**

```swift
if var newValueFlags {
    for flag in IsEnabledFlag.allCases where !flag.isHotSwappable {
        newValueFlags[flag.rawValue] = self.valueFlags[flag.rawValue]   // keep OLD value
    }
    // …same for ValueFlag and TimeGatedFlag…
    return RemoteConfig(clockSkew: newClockSkew, valueFlags: newValueFlags)
} else {
    // nil newValueFlags (HTTP 304): keep all old flags, update only clock skew
    return RemoteConfig(clockSkew: newClockSkew, valueFlags: self.valueFlags)
}
```

So merging **replaces hot-swappable keys with fresh values** and **restores
non-hot-swappable keys from the previous snapshot**. The clock skew is *always*
taken from the new value (even on 304). **[High]**

`TimeGatedFlag.isHotSwappable` is hard-coded `true` (`:862-868`) — the comment
explains time-gated flags can flip false→true just by the clock crossing the
threshold while the app is in memory, so they must always be re-evaluated.
**[High]**

### Warm vs. merge at launch

`RemoteConfigProviderImpl.warmCaches(tx:)` (`:...`) distinguishes the **first**
warm (sets both hot- and non-hot-swappable from DB) from **subsequent** warms
(only merges hot-swappable). This is how the NSE and main app converge. **[High]**

---

## 4. Fetch / refresh lifecycle

`RemoteConfigManagerImpl` runs a perpetual refresh loop. **[High]**

- `refreshInterval = 2 * .hour` (`:...` `private static let refreshInterval`).
- On init, after `runNowOrWhenMainAppDidBecomeReadyAsync`, it starts
  `refreshRepeatedlyIfNeeded(forceInitialRefreshImmediately: false)` and
  observes `.registrationStateDidChange` (`:...`). **[High]**
- `registrationStateDidChange` forces an immediate refresh
  (`forceInitialRefreshImmediately: true`). **[High]**
- `refreshRepeatedlyIfNeeded` cancels any prior `refreshTask`, bails if **not
  registered**, else spawns a `Task` running `refreshRepeatedly`. **[High]**
- `refreshRepeatedly` loops forever: computes `nextFetchDate` (last fetch +
  interval), sleeps until then (unless forced), then runs `refreshIfNeeded`
  wrapped in `Retry.performWithBackoff(maxAttempts: .max, maxAverageBackoff:
  14.1 * .minute)`, treating **all** errors as retryable (`OWSRetryableError`).
  **[High]**
- `refreshIfNeeded` / `forceRefresh` both funnel through a
  `ConcurrentTaskQueue(concurrentLimit: 1)` so only one `_refresh` runs at a
  time. `refreshIfNeeded` is a no-op if `now <= nextFetchDate`, and asserts the
  fetch date advanced afterward (`owsPrecondition`). **[High]**

### `_refresh()` steps (`:...`) — the authoritative sequence **[High]**

1. `fetchRemoteConfig()` → `(valueFlags?, headers)`.
2. If `valueFlags == nil`, log "values haven't changed" (HTTP 304).
3. Parse `x-signal-timestamp` header → `serverEpochTimeMs` (asserted present);
   compute `clockSkew = serverDate - now`.
4. In one `awaitableWrite`: persist clock skew; if new flags present, remove the
   legacy IsEnabled & TimeGated dictionaries, set the value-flags dictionary,
   store the ETag; always set `lastFetched = now`.
5. `updateCachedConfig { (old ?? .emptyConfig).merging(newValueFlags:, newClockSkew:) }`.
6. `await checkClientExpiration(valueFlag: mergedConfig.value(.clientExpiration))`.
7. `net.setRemoteConfig(mergedConfig.netConfig(), buildVariant: BuildFlags.netBuildVariant)`.
8. `mergedConfig.logFlags()`.

### `fetchRemoteConfig()` (`:...`)

Builds `OWSRequestFactory.getRemoteConfigRequest(eTag:)` with the stored ETag,
decodes `{ "config": {String:String} }`. A **304 Not Modified**
(`OWSHTTPError.serviceResponse` where status == 304) returns `(nil, headers)`,
driving the "unchanged" merge path. **[High]**

### `netConfig()` (`:88-104`)

Projects `valueFlags` into the subset LibSignal consumes: drops any `"false"`
value (TODO notes the server should omit these), keeps only keys prefixed
`ios.libsignal.`, and strips that prefix. **[High]**

---

## 5. Client expiration

`checkClientExpiration(valueFlag:)` (`:...`) decodes the `ios.clientExpiration`
value and tells `AppExpiry` the enforcement date for the current version.
**[High]**

- `MinimumVersion` decodes `{ "minVersion": AppVersionNumber4, "iso8601": Date }`
  with `.iso8601` date strategy (`:...`).
- `parseClientExpiration`: empty/nil → `[]` (no expirations); invalid JSON →
  `owsFailDebug` and returns **nil** so the caller leaves `AppExpiry` untouched
  (*"err on the safe side"*). **[High]**
- `remoteExpirationDate`: among `MinimumVersion`s the current version does **not**
  yet satisfy, takes the **earliest** `enforcementDate`. **[High]**

> **[High]** The empty-string vs. parse-failure distinction matters: an empty
> flag clears expirations; a *malformed* flag is treated as "don't touch", so a
> server typo can't accidentally brick clients.

---

## 6. Persistence (`RemoteConfigStore` + KeyValueStore)

- `RemoteConfigStore.loadValueFlags(tx:)` (`:...`) reads the value-flags
  dictionary and **back-fills** from two legacy dictionaries
  (`IsEnabled` → `"true"/"false"`, `TimeGated` → epoch strings) with a TODO to
  remove the fallbacks. **[High]**
- The `KeyValueStore` extension (`:...`) defines the storage keys:
  `remoteConfigValueFlags`, legacy `remoteConfigKey` (IsEnabled) and
  `remoteConfigTimeGatedFlags`, `lastFetchedKey`, `clockSkewKey`, `eTag`.
  **[High]**
- `RemoteConfigProviderImpl.warmCaches(tx:)` only loads flags if the account
  `isRegistered`; otherwise `(nil, nil)` and the config stays empty. **[High]**

Test doubles exist under `TESTABLE_BUILD`: `MockRemoteConfigProvider` and
`StubbableRemoteConfigManager` (`:...`), plus `test.hotSwappable.*` /
`test.nonSwappable.*` flags on both enums. **[High]**

---

## 7. Country-code bucketing

Several flags are **per-region** or **percentage-rolled-out**: **[High]**

- `parsePhoneNumberRegions(valueFlags:flag:defaultValue:)` → `PhoneNumberRegions`
  (`:...`). Used for the six donation/payment region flags set in `init`
  (`:40-52`).
- `countryCodeValue(csvString:callingCode:)` parses `cc:value,cc:value,*:value`
  CSV, resolving the user's calling code and falling back to the `*` wildcard
  (`:...`). Malformed pairs → `owsFailDebug`, skipped. **[High]**
- `isCountryCodeBucketEnabled(csvString:key:localIdentifiers:)` + `bucket(key:
  aci:bucketSize:)` implement deterministic per-ACI rollout:
  `UINT64_FROM_FIRST_8_BYTES_BIG_ENDIAN(SHA256(key + "." + aciBytes)) % bucketSize`,
  enabled when `countEnabled > bucket` (`:...`). **[High]**

`callQualitySurveyPPM(localIdentifiers:)` (`:...`) and
`standardMediaQualityLevel(callingCode:)` (`:...`) are the public consumers.
**[High]**

---

## 8. Enumerated flags & remote-config keys

### 8.1 `IsEnabledFlag` (boolean) — `RemoteConfigManager.swift:701-754`

| Case | Server key | Hot-swap | Public accessor(s) |
| --- | --- | --- | --- |
| `applePayGiftDonationKillSwitch` | `ios.applePayGiftDonationKillSwitch` | ❌ | `canDonateGiftWithApplePay` (negated) |
| `applePayMonthlyDonationKillSwitch` | `ios.applePayMonthlyDonationKillSwitch` | ❌ | `canDonateMonthlyWithApplePay` |
| `applePayOneTimeDonationKillSwitch` | `ios.applePayOneTimeDonationKillSwitch` | ❌ | `canDonateOneTimeWithApplePay` |
| `automaticSessionResetKillSwitch` | `ios.automaticSessionResetKillSwitch` | ❌ | `automaticSessionResetKillSwitch` |
| `cardGiftDonationKillSwitch` | `ios.cardGiftDonationKillSwitch` | ❌ | `canDonateGiftWithCreditOrDebitCard` |
| `cardMonthlyDonationKillSwitch` | `ios.cardMonthlyDonationKillSwitch` | ❌ | `canDonateMonthlyWithCreditOrDebitCard` |
| `cardOneTimeDonationKillSwitch` | `ios.cardOneTimeDonationKillSwitch` | ❌ | `canDonateOneTimeWithCreditOrDebitCard` |
| `enableAutoAPNSRotation` | `ios.enableAutoAPNSRotation` | ❌ | `enableAutoAPNSRotation` (default false) |
| `enableGifSearch` | `global.gifSearch` | ❌ | `enableGifSearch` (default **true**) |
| `messageResendKillSwitch` | `ios.messageResendKillSwitch` | ❌ | `messageResendKillSwitch` |
| `paymentsResetKillSwitch` | `ios.paymentsResetKillSwitch` | ❌ | `paymentsResetKillSwitch` |
| `paypalGiftDonationKillSwitch` | `ios.paypalGiftDonationKillSwitch` | ❌ | `canDonateGiftWithPayPal` |
| `paypalMonthlyDonationKillSwitch` | `ios.paypalMonthlyDonationKillSwitch` | ❌ | `canDonateMonthlyWithPaypal` |
| `paypalOneTimeDonationKillSwitch` | `ios.paypalOneTimeDonationKillSwitch` | ❌ | `canDonateOneTimeWithPaypal` |
| `ringrtcNwPathMonitorTrialKillSwitch` | `ios.ringrtcNwPathMonitorTrialKillSwitch` | ✅ (comment: "cached during launch, so not hot-swapped in practice") | `ringrtcNwPathMonitorTrial` |
| `ringrtcSvcEnabled` | `ios.ringrtcSvcEnabled` | ❌ | `ringrtcSvcEnabled` |
| `ringrtcVp9Enabled` | `ios.ringrtcVp9Enabled.2` | ✅ | `ringrtcVp9Enabled` |
| `serviceExtensionFailureKillSwitch` | `ios.serviceExtensionFailureKillSwitch` | ✅ | `shouldCheckForServiceExtensionFailures` |
| `wifiAwareDeviceTransferKillSwitch` | `ios.wifiAwareDeviceTransferKillSwitch` | ✅ | `wifiAwareDeviceTransferEnabled` (also gated by `BuildFlags.wifiAwareDeviceTransfer`) |
| `hotSwappable` *(TESTABLE_BUILD)* | `test.hotSwappable.enabled` | ✅ | — |
| `nonSwappable` *(TESTABLE_BUILD)* | `test.nonSwappable.enabled` | ❌ | — |

### 8.2 `ValueFlag` (typed values) — `RemoteConfigManager.swift:756-857`

| Case | Server key | Hot-swap | Default / accessor |
| --- | --- | --- | --- |
| `adminDeleteMaxAgeInSeconds` | `global.adminDeleteMaxAgeInSeconds` | ✅ | 1 day (ms→s); `adminDeleteMaxAgeInSeconds` |
| `applePayDisabledRegions` | `global.donations.apayDisabledRegions` | ✅ | regions; `applePayDisabledRegions` |
| `attachmentMaxEncryptedBytes` | `global.attachments.maxBytes` | ✅ | 100 MiB, capped at `attachmentHardLimit` (1,610,612,736) |
| `attachmentMaxEncryptedReceiveBytes` | `global.attachments.maxReceiveBytes` | ✅ | derived from max + fudge |
| `automaticSessionResetAttemptInterval` | `ios.automaticSessionResetAttemptInterval` | ✅ | 1 hour |
| `backgroundRefreshInterval` | `ios.backgroundRefreshInterval` | ✅ | 1 day |
| `backupAttachmentMaxEncryptedBytes` | `ios.backupAttachments.maxBytes` | ✅ | = attachmentMaxEncryptedBytes |
| `backupListMediaDefaultRefreshIntervalMs` | `ios.backupListMediaDefaultRefreshIntervalMs` | ✅ | day or week (per `BuildFlags.Backups.useLowerDefaultListMediaRefreshInterval`) |
| `backupListMediaOutOfQuotaRefreshIntervalMs` | `ios.backupListMediaOutOfQuotaRefreshIntervalMs` | ✅ | 1 day |
| `callQualitySurveyPPM` | `ios.callQualitySurveyPPM` | ✅ | `"*:10000"`; per-region PPM |
| `cdsSyncInterval` | `cds.syncInterval.seconds` | ❌ | 2 days; `cdsSyncInterval` |
| `clientExpiration` | `ios.clientExpiration` | ✅ | JSON; see §5 |
| `creditAndDebitCardDisabledRegions` | `global.donations.ccDisabledRegions` | ✅ | regions |
| `idealEnabledRegions` | `global.donations.idealEnabledRegions` | ✅ | regions |
| `maxGroupCallRingSize` | `global.calling.maxGroupCallRingSize` | ✅ | 16 |
| `maxGroupSizeHardLimit` | `global.groupsv2.groupSizeHardLimit` | ✅ | 1001 |
| `maxGroupSizeRecommended` | `global.groupsv2.maxGroupSize` | ✅ | 151 |
| `maxNicknameLength` | `global.nicknames.max` | ❌ | 32 |
| `maxPollOptionReceiveCount` | `ios.polls.maxReceiveOptionCount` | ✅ | 10 |
| `maxPollOptionSendCount` | `ios.polls.maxSendOptionCount` | ✅ | 10 |
| `maxSenderKeyAge` | `ios.maxSenderKeyAge` | ✅ | 2 weeks (ms) |
| `maxThumbnailFileSizeBytes` | `global.backups.maxThumbnailFileSizeBytes` | ✅ | `backupThumbnailMaxSizeBytes` |
| `mediaTierFallbackCdnNumber` | `global.backups.mediaTierFallbackCdnNumber` | ✅ | 3 |
| `messageQueueTimeInSeconds` | `global.messageQueueTimeInSeconds` | ❌ | 45 days |
| `messageSendLogEntryLifetime` | `ios.messageSendLogEntryLifetime` | ❌ | 2 weeks |
| `minNicknameLength` | `global.nicknames.min` | ❌ | 3 |
| `normalDeleteMaxAgeInSeconds` | `global.normalDeleteMaxAgeInSeconds` | ✅ | 1 day (ms→s) |
| `paymentsDisabledRegions` | `global.payments.disabledRegions` | ✅ | default `"98,963,53,850,7"` |
| `paypalDisabledRegions` | `global.donations.paypalDisabledRegions` | ✅ | regions |
| `pinnedMessageLimit` | `global.pinnedMessageLimit` | ✅ | 3 |
| `pinnedThreadLimit` | `global.pinnedChatLimit` | ✅ | 4 |
| `postRegistrationWaitingPeriodSeconds` | `global.changeNumber.postRegistrationWaitingPeriodSeconds` | ✅ | 1 hour (ms→s) |
| `reactiveProfileKeyAttemptInterval` | `ios.reactiveProfileKeyAttemptInterval` | ✅ | 1 hour |
| `replaceableInteractionExpiration` | `ios.replaceableInteractionExpiration` | ❌ | 1 hour |
| `ringrtcDredDuration` | `ios.ringrtcDredDuration` | ✅ | 0 |
| `ringrtcSvcMaxBitrateBps` | `ios.ringrtcSvcMaxBitrateBps` | ✅ | 0 |
| `ringrtcSvcMode` | `ios.ringrtcSvcMode` | ✅ | `"L3T3_KEY"` |
| `ringrtcSvcModeForScreenshare` | `ios.ringrtcSvcModeForScreenshare` | ✅ | `"L1T3"` |
| `ringrtcVp9DeviceModelDecodeDenylist` | `ios.ringrtcVp9DeviceModelDecodeDenylist` | ✅ | `.`-split list |
| `ringrtcVp9DeviceModelDenylist` | `ios.ringrtcVp9DeviceModelDenylist` | ✅ | `.`-split list (encode) |
| `sepaEnabledRegions` | `global.donations.sepaEnabledRegions` | ✅ | regions |
| `standardMediaQualityLevel` | `ios.standardMediaQualityLevel` | ✅ | per-region `ImageQualityLevel` |
| `videoAttachmentMaxEncryptedBytes` | `ios.videoAttachments.maxBytes` | ✅ | = attachmentMaxEncryptedBytes |
| `hotSwappable` *(TESTABLE_BUILD)* | `test.hotSwappable.value` | ✅ | — |
| `nonSwappable` *(TESTABLE_BUILD)* | `test.nonSwappable.value` | ❌ | — |

### 8.3 `TimeGatedFlag` — `RemoteConfigManager.swift:859-871`

Only `.__none` exists; there are currently **no real time-gated flags**, but the
machinery (always hot-swappable, clock-skew-corrected threshold compare) is in
place. **[High]**

### 8.4 Other non-flag-derived accessors

`maxGroupSizeBannedMembers` aliases `maxGroupSizeHardLimit` (`:112-114`);
`messageQueueTimeMs` derives from `messageQueueTime` (`:...`). **[High]**

---

## 9. Error paths & edge cases

| Situation | Handling | Citation |
| --- | --- | --- |
| HTTP 304 (unchanged) | merge path keeps old flags, updates clock skew only | `fetchRemoteConfig` + `merging` |
| Missing `x-signal-timestamp` | `owsAssertDebug`, clock skew → 0 | `_refresh` |
| Any refresh error | treated as retryable; infinite backoff (`maxAverageBackoff 14.1 min`) | `refreshRepeatedly` |
| Not registered | refresh loop no-ops; warmCaches loads nothing | `refreshRepeatedlyIfNeeded`, provider `warmCaches` |
| `getStringConvertibleValue` parse failure | `owsFailDebug`, returns default | `:...` |
| Malformed `cc:value` CSV entry | `owsFailDebug`, entry skipped | `countryCodeValue` |
| Malformed `clientExpiration` JSON | `owsFailDebug`, returns nil → leave AppExpiry untouched | `parseClientExpiration` |
| Non-hot-swappable flag changed on server | persisted now, applied only after app restart | `merging` |
| `attachmentMax*Bytes` server value too large | clamped to `attachmentHardLimit` (1.6 GiB) | `:235-237` etc. |
| `fetchNextFetchDate` didn't advance after refresh | `owsPrecondition` crash | `refreshIfNeeded` |

---

## 10. File reference checklist

| Symbol | File:line |
| --- | --- |
| `RemoteConfig` + `RemoteConfig.current` | `RemoteConfigManager.swift:10-14` |
| region-flag parsing in `init` | `:36-53` |
| `emptyConfig` | `:56-58` |
| `merging(newValueFlags:newClockSkew:)` | `:66-85` |
| `netConfig()` | `:88-104` |
| typed accessors (group size, donations, attachments, ringrtc, polls, …) | `:106-520` |
| `bucket(key:aci:bucketSize:)` | `:...` |
| `isEnabled` / time-gated `isEnabled` / `value` readers | `:658-685` |
| `logFlags()` / `debugDescriptions()` | `:...` |
| `IsEnabledFlag` | `:701-754` |
| `ValueFlag` | `:756-857` |
| `TimeGatedFlag` | `:859-871` |
| `FlagType` protocol | `:873-879` |
| `RemoteConfigProvider` / `…Impl` | `:882-...` |
| `RemoteConfigManager` / `…Impl` | `:...-928-...` |
| `_refresh()` / `fetchRemoteConfig()` | `:...` |
| client-expiration decoding | `:...` |
| `RemoteConfigStore` | `:...` |
| `KeyValueStore` keys extension | `:...` |

# `TSConstants.swift` — Environment Constants (Prod / Staging)

`TSConstants` selects the server **environment** (production vs. staging) at
launch and exposes every endpoint URL, censorship-circumvention host, SVR2 SGX
enclave, and ZK server public-params blob the client needs. The environment is
chosen once and frozen for the process lifetime.

- [1. Environment selection](#1-environment-selection)
- [2. Static accessors & the `shared` indirection](#2-static-accessors--the-shared-indirection)
- [3. `TSConstantsProtocol` — the full surface](#3-tsconstantsprotocol--the-full-surface)
- [4. `MrEnclave` (SVR2 SGX enclaves)](#4-mrenclave-svr2-sgx-enclaves)
- [5. Production vs. Staging values](#5-production-vs-staging-values)
- [6. Edge cases](#6-edge-cases)
- [7. File reference checklist](#7-file-reference-checklist)

---

## 1. Environment selection

```swift
private enum Environment { case production; case staging }

private static let environment: Environment = {
#if DEBUG
    if ProcessInfo.processInfo.environment["USE_STAGING"] == "1" { return .staging }
#endif
    return .production
}()
```
(`TSConstants.swift:13-32`). **[High]**

- In **DEBUG** builds you can point at staging by setting the `USE_STAGING=1`
  environment variable in your Xcode scheme (comment explains this avoids
  committing an environment change). **[High]**
- In non-DEBUG builds the `USE_STAGING` check is compiled out, so the default is
  **always `.production`**; the comment notes you'd edit the fallback return to
  hard-wire staging. **[High]**
- `isUsingProductionService: Bool { environment == .production }`
  (`TSConstants.swift:34-36`) is the public probe used by `AppSetup` to pick
  LibSignal's `Net` environment. **[High]**

---

## 2. Static accessors & the `shared` indirection

`TSConstants` is a non-instantiable class (`private init()`,
`TSConstants.swift:40`). It exposes `static var` passthroughs (e.g.
`mainServiceURL`, `textSecureCDN0ServerURL`, … `svr2Enclaves`,
`applicationGroup`, `serverPublicParams`) that forward to a single `shared`
instance (`TSConstants.swift:43-76`). **[High]**

```swift
public static let shared: TSConstantsProtocol = {
    switch environment {
    case .production: return TSConstantsProduction()
    case .staging:    return TSConstantsStaging()
    }
}()

public static let libSignalEnv: Net.Environment = {
    switch environment {
    case .production: return .production
    case .staging:    return .staging
    }
}()
```
(`TSConstants.swift:78-95`). So the static facade reads from whichever concrete
implementation matches the selected environment, and `libSignalEnv` mirrors the
choice for LibSignal. **[High]**

A few constants are environment-independent plain `static let`s:
`legalTermsUrl` (`https://signal.org/legal/`), `donateUrl`
(`https://signal.org/donate/`), `appStoreUrl` (`TSConstants.swift:42-44`).
**[High]**

---

## 3. `TSConstantsProtocol` — the full surface

`public protocol TSConstantsProtocol: AnyObject`
(`TSConstants.swift:99-131`) declares every per-environment value: **[High]**

| Member | Meaning |
| --- | --- |
| `mainServiceURL` | chat/service API base |
| `textSecureCDN0ServerURL` / `…CDN2…` / `…CDN3…` | attachment CDNs |
| `storageServiceURL` | Storage Service |
| `sfuURL` / `sfuTestURL` | group-call SFU (+ test SFU) |
| `svr2URL` | Secure Value Recovery v2 websocket |
| `registrationCaptchaURL` / `challengeCaptchaURL` | captcha pages |
| `kUDTrustRoots` | sealed-sender trust roots (array of base64 keys) |
| `updatesURL` / `updates2URL` | software-update endpoints |
| `censorshipFReflectorHost` / `censorshipGReflectorHost` | domain-fronting reflector hosts |
| `serviceCensorshipPrefix` / `cdn0…` / `cdn2…` / `cdn3…` / `storageService…` / `svr2CensorshipPrefix` | censorship-circumvention path prefixes |
| `svr2Enclaves: [MrEnclave]` | ordered SGX enclaves (newest→oldest) |
| `activeSvr2EnclaveCount: Int` | how many leading enclaves to back up to |
| `applicationGroup` | iOS app-group container id |
| `serverPublicParams` / `callLinkPublicParams` / `backupServerPublicParams` | ZK/libsignal server public params (`Data`) |

---

## 4. `MrEnclave` (SVR2 SGX enclaves)

```swift
public struct MrEnclave: Equatable {
    public let dataValue: Data
    public let stringValue: String
    init(_ stringValue: StaticString) {
        self.stringValue = String(describing: stringValue)
        self.dataValue = Data.data(fromHex: self.stringValue)!   // constant; never fails
        owsPrecondition(self.dataValue.count == 32)              // all MrEnclaves are 32 bytes
    }
    public static func ==(lhs:rhs:) -> Bool { lhs.dataValue == rhs.dataValue }
}
```
(`TSConstants.swift:134-150`). **[High]**

- Takes a compile-time `StaticString` hex value, decodes to 32 bytes, and
  **crashes** (`owsPrecondition`) if the length is wrong — a build-time-only
  constant, so this can only trip during development. **[High]**
- `svr2Enclaves` is documented as **newest→oldest**: during registration the
  client tries each enclave in order to **restore** key material, and when
  **backing up** it writes to the first `activeSvr2EnclaveCount` enclaves
  (typically 2 briefly after adding a new enclave) (`TSConstants.swift:181-192`).
  **[High]**

---

## 5. Production vs. Staging values

Side-by-side (`TSConstantsProduction` at `TSConstants.swift:153-...`;
`TSConstantsStaging` at `:209-...`). **[High]**

| Member | Production | Staging |
| --- | --- | --- |
| `mainServiceURL` | `https://chat.signal.org` | `https://chat.staging.signal.org` |
| `textSecureCDN0ServerURL` | `https://cdn.signal.org` | `https://cdn-staging.signal.org` |
| `textSecureCDN2ServerURL` | `https://cdn2.signal.org` | `https://cdn2-staging.signal.org` |
| `textSecureCDN3ServerURL` | `https://cdn3.signal.org` | `https://cdn3-staging.signal.org` |
| `storageServiceURL` | `https://storage.signal.org` | `https://storage-staging.signal.org` |
| `sfuURL` | `https://sfu.voip.signal.org` | `https://sfu.staging.voip.signal.org` |
| `sfuTestURL` | `https://sfu.test.voip.signal.org` | same (no separate staging test SFU) |
| `svr2URL` | `wss://svr2.signal.org` | `wss://svr2.staging.signal.org` |
| `registrationCaptchaURL` | `.../registration/generate.html` | `.../staging/registration/generate.html` |
| `challengeCaptchaURL` | `.../challenge/generate.html` | `.../staging/challenge/generate.html` |
| `updatesURL` / `updates2URL` | `https://updates.signal.org` / `https://updates2.signal.org` | same (no separate staging endpoint) |
| `censorshipFReflectorHost` | `reflector-signal.global.ssl.fastly.net` | `reflector-staging-signal.global.ssl.fastly.net` |
| `censorshipGReflectorHost` | `reflector-nrgwuv7kwq-uc.a.run.app` | same |
| censorship prefixes | `service`, `cdn`, `cdn2`, `cdn3`, `storage`, `svr2` | each with `-staging` suffix |
| `svr2Enclaves` count | 2 enclaves (newest first) | 3 enclaves |
| `activeSvr2EnclaveCount` | 1 | 1 |
| `applicationGroup` | `group.<prefix>.signal.group` | `group.<prefix>.signal.group.staging` |
| `kUDTrustRoots` | 2 prod roots | 2 staging roots |

`serverPublicParams`, `callLinkPublicParams`, and `backupServerPublicParams` are
**different base64 blobs** per environment. Each prod value carries the comment
that changing it may require clearing credentials / a `ZkParamsMigrator`
migration (`TSConstants.swift:198-205`). **[High]**

Production SVR2 enclaves (`TSConstants.swift:188-191`): **[High]**
```
ced8217b26228e4b210c985786999d095c4958a94faf37b14acaf25c4cbb02a4   (newest)
1240acbd4aa26974184844c8a46b1022d3957ac8a76c1fd8f5b1a15141ee0708
```

### 5.1 `TSConstantsMock` (TESTABLE_BUILD)

Under `TESTABLE_BUILD`, `TSConstantsMock` (`TSConstants.swift:262-...`) implements
the protocol with `lazy var`s each defaulting to the corresponding
`TSConstantsProduction()` value, so tests can override individual endpoints
while inheriting the rest. **[High]**

---

## 6. Edge cases

| Situation | Behavior | Citation |
| --- | --- | --- |
| `USE_STAGING=1` in a RELEASE build | ignored — the check is `#if DEBUG` only | `TSConstants.swift:21-27` |
| `MrEnclave` hex not 32 bytes | `owsPrecondition` crash (build-time constant) | `TSConstants.swift:143-146` |
| Changing `serverPublicParams` | may require credential clear / `ZkParamsMigrator` | `TSConstants.swift:198-205` |
| Staging reuses prod SFU-test / updates endpoints | intentional (comments note "no separate …") | `TSConstants.swift:216-224` |
| `applicationGroup` depends on `Bundle.main.bundleIdPrefix` | computed at runtime, per-target | `TSConstants.swift:195`, `:247` |

---

## 7. File reference checklist

| Symbol | File:line |
| --- | --- |
| `Environment` enum + selection | `TSConstants.swift:13-32` |
| `isUsingProductionService` | `TSConstants.swift:34-36` |
| static accessors | `TSConstants.swift:42-76` |
| `shared` factory | `TSConstants.swift:78-85` |
| `libSignalEnv` | `TSConstants.swift:87-94` |
| `TSConstantsProtocol` | `TSConstants.swift:99-131` |
| `MrEnclave` | `TSConstants.swift:134-150` |
| `TSConstantsProduction` | `TSConstants.swift:153-206` |
| prod `svr2Enclaves` | `TSConstants.swift:188-191` |
| prod `activeSvr2EnclaveCount` / `applicationGroup` | `TSConstants.swift:193-195` |
| `TSConstantsStaging` | `TSConstants.swift:209-...` |
| staging enclaves / app group | `TSConstants.swift:239-247` |
| `TSConstantsMock` | `TSConstants.swift:262-...` (TESTABLE_BUILD) |

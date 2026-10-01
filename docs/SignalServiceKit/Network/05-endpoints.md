# 05 · Endpoint Catalog

Every service endpoint produced by the request factories in `API/Requests/`,
with method, purpose, request/response shape, auth, and retry/backoff notes.

## Conventions

- **Auth** values map to `TSRequest.Auth` (`TSRequest.swift:60-95`):
  `identified` (→ identified socket), `anonymous` / `sealedSender` / `backup`
  (→ unidentified socket), `registration` (→ REST only; the socket path throws).
- **Retry/backoff** is *not* set per endpoint. It is decided by the caller's
  `NetworkManager.RetryPolicy` (default `.dont` = 1 attempt;
  `.hopefullyRecoverable` = 3 attempts on 5xx + network failure). See
  [04-api-layer.md](04-api-layer.md). Unless noted, "default" = `.dont`,
  "timeout" = `textSecureHTTPTimeOut` (**10s**, `OWSRequestFactory.swift:19`).
  (Confidence: **High** for shapes/auth; **Medium** for retry, since it is
  caller-chosen and callers live outside this directory.)
- Base URL/host is resolved later by `OWSSignalService` / the chat socket, so
  URLs below are the relative paths exactly as written in the factories.

## `OWSRequestFactory.swift`

| Method · Path | Purpose | Request body | Response | Auth |
| --- | --- | --- | --- | --- |
| `GET v1/payments/conversions` | currency conversion rates | — | JSON | identified (default) |
| `GET v2/config/` | remote config (sends `If-None-Match` eTag) | — | JSON (or 304) | identified |
| `GET v2/calling/relays` | TURN/calling relay credentials | — | JSON | identified |
| `GET v1/certificate/auth/group?redemptionStartSeconds=..&redemptionEndSeconds=..&v101=true` | group auth credentials | — | JSON | identified |
| `GET v1/payments/auth` | payments auth credential | — | JSON | identified |
| `GET v2/directory/auth` | CDSI remote-attestation auth | — | `{username,password}` | identified |
| `GET v2/svr/auth` | SVR2 remote-attestation auth | — | `{username,password}` | identified |
| `GET v1/storage/auth` | storage service auth | — | JSON | identified (explicit) |
| `POST v1/challenge/push` | request a push challenge | `{}` | — | identified |
| `PUT v1/challenge` (`rateLimitPushChallenge`) | fulfill push challenge | `{type,challenge}` | — | identified |
| `PUT v1/challenge` (`captcha`) | fulfill captcha challenge | `{type,token,captcha}` | — | identified |
| `GET v1/certificate/delivery[?includeE164=false]` | UD sender certificate | — | JSON | identified |
| `HEAD v1/accounts/account/{serviceId}` | check if an account exists | — | status only | **anonymous** |
| `PUT v1/accounts/registration_lock` | enable reglock v2 | `{registrationLock}` | — | identified |
| `DELETE v1/accounts/registration_lock` | disable reglock v2 | `{}` | — | identified |
| `DELETE v1/accounts/me` | unregister account | `{}` | — | identified |
| `POST v1/profile/identity_check/batch` | batch identity check (≤1000) | `{elements}` | JSON | **anonymous** |
| `GET v2/keys[?identity=pni]` | available prekey count | — | JSON | identified |
| `GET v2/keys/{serviceId}/{deviceId}` | fetch recipient prekeys | — | JSON | sealedSender (optional) |
| `PUT v2/keys[?identity=pni]` | upload prekeys (**timeout 45s**) | `SetKeysRequest` (encodable) | — | identified |
| `GET v1/profile/{serviceId}` | unversioned profile | — | JSON | caller-provided |
| `GET v1/profile/{aci}/{ver}[/{credReq}?credentialType=expiringProfileKey]` | versioned profile | — | JSON | caller-provided |
| `PUT v1/profile/` | set versioned profile | profile params | — | identified |

Citations: `OWSRequestFactory.swift` lines `24-40` (config/relays/conversions),
`44-74` (auth), `69-78` (challenges), `84-95` (UD cert / account HEAD),
`128-168` (reglock / unregister / identity check), `172-186` (device provisioning;
see [04-api-layer.md](04-api-layer.md)), `~430-520` (keys), `~525-610` (profiles).
Donations endpoints below are also in this file. (Confidence: **High**.)

### Donations (in `OWSRequestFactory.swift`)

All **anonymous**; many apply `RedactionStrategy.redactURL` to hide the
subscriber ID / payment IDs in logs (`TSRequest.swift:177-213`).

| Method · Path | Purpose |
| --- | --- |
| `PUT v1/subscription/{subscriberId}` | create/extend subscriber (optional `Donation-Permit` header) |
| `DELETE v1/subscription/{subscriberId}` | delete subscriber |
| `POST v1/subscription/{id}/default_payment_method/{processor}/{paymentMethodId}` | set default payment method |
| `POST v1/subscription/{id}/default_payment_method_for_ideal/{setupIntentId}` | set default iDEAL method |
| `POST v1/subscription/{id}/create_payment_method` | create Stripe payment method (`Donation-Permit`) |
| `POST v1/subscription/{id}/create_payment_method/paypal` | create PayPal payment method |
| `PUT v1/subscription/{id}/level/{level}/{currency}/{idempotencyKey}` | set subscription level |
| `POST v1/subscription/{id}/receipt_credentials` | subscription receipt credentials |
| `POST v1/donation/redeem-receipt` | redeem receipt credential (identified) |
| `POST v1/subscription/boost/receipt_credentials` | boost receipt credentials |
| `GET v1/subscription/bank_mandate/{type}` | bank mandate text (sets `Accept-Language`) |

Citations: `OWSRequestFactory.swift:~190-410`. (Confidence: **High**.)

## `OWSRequestFactory+Usernames.swift`

| Method · Path | Purpose | Auth |
| --- | --- | --- |
| `PUT v1/accounts/username_hash/reserve` | reserve username hashes (valid 5 min) | identified |
| `PUT v1/accounts/username_hash/confirm` | confirm reserved username (`usernameHash,zkProof,encryptedUsername`) | identified |
| `GET v1/accounts/username_hash/{hash}` | ACI lookup by username hash | **anonymous** |
| `PUT v1/accounts/username_link` | set encrypted username link | identified |

Citations: `OWSRequestFactory+Usernames.swift:11-116`. (Confidence: **High**.)

## `OWSRequestFactory+Spam.swift`

| Method · Path | Purpose | Auth |
| --- | --- | --- |
| `POST v1/messages/report/{senderAci}/{serverGuid}` | report spam (optional `{token}`) | identified (default) |

Citation: `OWSRequestFactory+Spam.swift:9-30`. (Confidence: **High**.)

## `OWSRequestFactory+BoostPayments.swift`

All **anonymous**.

| Method · Path | Purpose |
| --- | --- |
| `POST v1/subscription/boost/create` | create Stripe payment intent for a boost (`Donation-Permit`) |
| `POST v1/subscription/boost/paypal/create` | create PayPal boost payment |
| `POST v1/subscription/boost/paypal/confirm` | confirm one-time PayPal payment |

Citations: `OWSRequestFactory+BoostPayments.swift:25-108`. (Confidence: **High**.)

## `OWSRequestFactory+Backups.swift`

All use **backup** auth (`BackupServiceAuth`) except
`backupAuthenticationCredentialRequest` which is **identified**.

| Method · Path | Purpose |
| --- | --- |
| `GET v1/archives/auth?redemptionStartSeconds=..&redemptionEndSeconds=..` | backup auth credentials (identified) |
| `GET v1/archives` | backup info |
| `PUT v1/archives` | refresh backup info |
| `GET v1/archives/auth/read?cdn={cdn}` | backup CDN read credentials |
| `PUT v1/archives/media` | copy an item to media tier |
| `PUT v1/archives/media/batch` | archive a batch of media items |
| `GET v1/archives/media[?limit=&cursor=]` | list media |
| `POST v1/archives/media/delete` | delete media (`mediaToDelete`) |
| `POST v1/archives/redeem-receipt` | redeem backup receipt credential |
| `GET v1/archives/auth/svrb` | fetch SVRB auth credential |

Citations: `OWSRequestFactory+Backups.swift:11-205`. (Confidence: **High**.)

## `WhoAmI/WhoAmIManager.swift`

| Method · Path | Purpose | Response | Auth |
| --- | --- | --- | --- |
| `GET v1/accounts/whoami` | fetch own account identity | `WhoAmI { aci(uuid), pni, e164(number), usernameHash?, entitlements{backup,badges} }` | identified |

`makeWhoAmIRequest()` requires HTTP 200 then JSON-decodes.
Citations: `WhoAmIManager.swift:26-45` (manager), `58-180` (response + factory).
(Confidence: **High**.)

## `RemoteAttestationAuthFetcher.swift`

Fetches `{username,password}` for `cdsi` (`GET v2/directory/auth`) or `svr2`
(`GET v2/svr/auth`); uses **identified** auth
(`RemoteAttestationAuthFetcher.swift:40-72`). (Confidence: **High**.)

## `AccountDataReportRequestFactory.swift`

`GET v2/accounts/data_report` (identified, default; parsed by `AccountDataReport`)
(`AccountDataReportRequestFactory.swift:9-13`). (Confidence: **High**.)

## `Provisioning/ProvisioningRequestFactory.swift`

| Method · Path | Purpose | Request | Auth |
| --- | --- | --- | --- |
| `PUT v1/devices/link` | verify + link a secondary device | `LinkDeviceRequest { verificationCode, accountAttributes, aci/pni signed+kyber prekeys, apnToken? }` | `registration((aci, authPassword))` |

Response codes (`ProvisioningServiceResponses.swift:9-30`):
`200` success (`VerifySecondaryDeviceResponse { pni, deviceId }`),
`409` obsolete linked device, `411` device limit exceeded, `-1` unexpected.
Citations: `ProvisioningRequestFactory.swift:11-55`. (Confidence: **High**.)

## `Registration/RegistrationRequestFactory.swift`

All registration requests use **`registration`** auth (socket path throws; these
go over REST). Session IDs are redacted from logs
(`RegistrationRequestFactory.swift:~335`).

| Method · Path | Purpose | Notable params |
| --- | --- | --- |
| `POST v1/verification/session` | begin a verification session | `number`, optional `pushToken`/`pushTokenType=apn`, `mcc`, `mnc` |
| `GET v1/verification/session/{id}` | fetch session state | — |
| `PATCH v1/verification/session/{id}` | fulfill captcha/push challenge | `captcha` / `pushChallenge` |
| `POST v1/verification/session/{id}/code` | request SMS/voice code | `transport`, `client=ios`, `Accept-Language` |
| `PUT v1/verification/session/{id}/code` | submit verification code | `code` |
| `POST v2/svr/auth/check` | check SVR2 credentials | `number`, `passwords` |
| `POST v1/registration` | create/re-register account | `RegistrationRequest` (attributes, prekeys, `sessionId`/`recoveryPassword`); `X-Signal-Agent: OWI` |
| `PUT v2/accounts/number` | change number | `ChangeNumberRequest` (new number, reglock?, PNI keys/messages) |

Response code enums live in
[`RegistrationServiceResponses.swift`](../../../SignalServiceKit/Network/API/Requests/Registration/RegistrationServiceResponses.swift)
(`class RegistrationServiceResponses`, lines `9-335`) — per-endpoint response-code
enums for the session API (begin/fetch/fulfill/request-code/submit-code) and
account creation. Citations: `RegistrationRequestFactory.swift:16-330`;
`RegistrationServiceResponses.swift:9-335`. (Confidence: **High** for request
shapes; **Medium** for the full response-code enum contents, which were
summarized from the file's structure rather than enumerated line-by-line.)

## `AccountAttributes/AccountAttributesRequestFactory.swift`

| Method · Path | Purpose | Request | Auth |
| --- | --- | --- | --- |
| `PUT v1/accounts/attributes` | update primary device attributes (`X-Signal-Agent: OWI`) | `AccountAttributes` (encodable) | identified |
| `PUT v1/devices/capabilities` | update linked device capabilities | `AccountAttributes.Capabilities` (encodable) | identified |

Driven by `AccountAttributesUpdaterImpl` (see [04-api-layer.md](04-api-layer.md)).
Citations: `AccountAttributesRequestFactory.swift:13-60`. (Confidence: **High**.)

## Device provisioning (code) — in `OWSRequestFactory.swift`

| Method · Path | Purpose | Auth |
| --- | --- | --- |
| `GET v1/devices/provisioning/code` | request a provisioning code | identified |
| `PUT v1/provisioning/{ephemeralDeviceId}` | send provisioning message (`{body: base64}`) | identified |

Wrapped by `DeviceProvisioningService`
(`DeviceProvisioningService.swift:40-73`; factory `OWSRequestFactory.swift:172-186`).
(Confidence: **High**.)

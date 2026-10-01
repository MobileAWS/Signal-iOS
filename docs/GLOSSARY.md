# Glossary — Signal iOS domain & Signal-specific terms

Concise definitions of recurring domain terms, each tied to at least one source
location (file path + line number) where the term is **defined** or **used**.
Where an existing doc under `docs/` covers a term in depth, it is cross-linked.

> **Confidence labels** (matching the convention used in the existing
> `docs/SignalServiceKit/**` docs):
> - **[High]** — definition is explicit in first-party source.
> - **[Medium]** — inferred, or depends on out-of-tree collaborators (e.g.
>   LibSignalClient) whose implementation is not in this tree.
> - **[Low]** — inferred from naming/comments only.
>
> Where a term's purpose cannot be established from first-party source, it is
> marked **"intent undetermined — no evidence in source"**. Line numbers are
> anchors against the tree at authoring time and may drift.

---

## ACI (Account Identifier)

The stable UUID that represents **the user/account** itself, independent of
their phone number. One of the two `ServiceId` kinds an account owns. The
`OWSIdentity` enum documents it directly: *"The ACI ('account identifier')
represents the user in question"* and the `case aci = 0` enumerator.
**[High]**

- Definition: `SignalServiceKit/Account/OWSIdentity.swift:6-18` (`case aci = 0`;
  doc comment defining ACI).
- Parsing/wrappers: `SignalServiceKit/Account/ServiceId.swift:10-22`
  (`Aci.parseFrom(aciString:)`).
- In-depth cross-link:
  [Account/identity-and-registration.md](SignalServiceKit/Account/identity-and-registration.md)
  (§1 "Identity primitives"),
  [Cryptography/identity-and-keys.md](SignalServiceKit/Cryptography/identity-and-keys.md).

## PNI (Phone Number Identifier)

The UUID that represents **the user's phone number (E164)** rather than the
account. The second `ServiceId` kind. Per `OWSIdentity`: *"the PNI ('phone
number identifier') represents the user's phone number (e164)"*. On the local
user, `pni` is optional because a primary may need to fetch it and a linked
device may be waiting to learn it from the primary. **[High]**

- Definition: `SignalServiceKit/Account/OWSIdentity.swift:6-18` (`case pni = 1`).
- Parsing: `SignalServiceKit/Account/ServiceId.swift:25-41`
  (`Pni.parseFrom(pniString:)` / `parseFrom(ambiguousString:)`).
- Optionality rationale: `SignalServiceKit/Account/LocalIdentifiers.swift:28-37`.
- In-depth cross-link:
  [Account/identity-and-registration.md](SignalServiceKit/Account/identity-and-registration.md)
  (§6 "PNI handling"),
  [ChangePhoneNumber/change-phone-number.md](SignalServiceKit/ChangePhoneNumber/change-phone-number.md).

## E164

A phone number in canonical E.164 format (`+<country><number>`). Used as the
identifier a registration session is created for and as the human-facing
address paired with a PNI. **[High]**

- Used as a typed field: `SignalServiceKit/Registration/RegistrationSession.swift:19-20`
  (`public let e164: E164`).
- `PhoneNumber { e164: E164; pni: Pni }` pairing:
  `SignalServiceKit/Account/LocalIdentifiers.swift:11-25`.

## ServiceId

LibSignal type that is either an `Aci` or a `Pni`; the first-party `ServiceId`
extensions add parsing, Codable, and DB-address plumbing. The concrete kind is
selectable via `ConcreteType` (`.aci` / `.pni`). **[High]**

- `ServiceId.ConcreteType` / `concreteType`: `SignalServiceKit/Account/ServiceId.swift:43-57`.
- `ServiceId.parseFrom(serviceIdBinary:serviceIdString:)`: `SignalServiceKit/Account/ServiceId.swift:59-69`.
- The underlying `Aci`/`Pni`/`ServiceId` value types themselves live in
  **LibSignalClient** (out of tree). **[Medium]**

## Prekey (pre-key)

A pre-published key the server hands out so a sender can start an encrypted
session without the recipient being online. The on-disk `PreKeyRecord`
distinguishes **one-time** vs **signed/last-resort** keys per identity and
namespace; `keyIds must not be reused within a namespace for a particular
identity`. The lifecycle (create for registration/provisioning, rotate signed
keys, refresh one-time keys) is managed by `PreKeyManager`. **[High]**

- On-disk model: `SignalServiceKit/Axolotl/PreKeyRecord.swift:9-26`
  (table `PreKey`; `isOneTime`, `namespace`, `keyId`).
- Lifecycle API: `SignalServiceKit/Axolotl/PreKeyManager.swift:8-45`
  (`createPreKeysForRegistration`, `rotateSignedPreKeysIfNeeded`,
  `refreshOneTimePreKeys`).
- In-depth cross-link: [Cryptography/prekeys.md](SignalServiceKit/Cryptography/prekeys.md).

## Sealed sender / Unidentified delivery (UD)

A delivery mode in which the sender's identity is hidden from the server, using
an **Unidentified Access Key (UAK)**. The client tracks a per-recipient
`UnidentifiedAccessMode` (`unknown` / `enabled` / `disabled` / `unrestricted`)
and constructs an `OWSUDAccess` (a UAK the client believes may be valid). The
envelope encryption and sealed-sender *certificate* handling themselves live in
LibSignalClient; the certificate path is **not** present in these first-party
files. **[High]** for the mode enum / access type; **[Medium]** for the crypto
that lives out of tree.

- `UnidentifiedAccessMode` enum: `SignalServiceKit/Messages/UD/OWSUDManager.swift:16-20`.
- `OWSUDAccess` (a UAK believed valid): `SignalServiceKit/Messages/UD/OWSUDManager.swift:41-57`.
- Certificate handling: intent undetermined — no evidence in first-party source
  (lives in LibSignalClient). **[Medium]**
- In-depth cross-link: [Cryptography/sealed-sender.md](SignalServiceKit/Cryptography/sealed-sender.md).

## Sender key

Group-messaging optimization: a sender encrypts a message **once** and sends the
same ciphertext to many recipients, after distributing the sender key to each
recipient device via a **Sender Key Distribution Message (SKDM)**. The
first-party `SenderKeyStore` / `SenderKeyManager` manage which sender key to use
per group, who has already received an SKDM, and persistence of opaque records;
the ratchet and SKDM crypto are in LibSignalClient. **[High]** for the store;
**[Medium]** for the ratchet (out of tree).

- `SenderKeyManager`: `SignalServiceKit/Axolotl/SenderKeyManager.swift:9`.
- `SenderKeyRecord` on-disk model (table `SenderKey`):
  `SignalServiceKit/Axolotl/SenderKeyStore.swift:11-105`.
- In-depth cross-link: [Cryptography/sealed-sender.md](SignalServiceKit/Cryptography/sealed-sender.md)
  (covers Sender Keys + SKDM + UD together).

## SKDM (Sender Key Distribution Message)

The message that distributes a sender key to a recipient device so it can later
decrypt sender-key-encrypted group messages. First-party code tracks *who* has
already received a copy; the message's crypto is in LibSignalClient. **[Medium]**
(inferred from the SKDM-tracking comment).

- SKDM delivery tracking: `SignalServiceKit/Axolotl/SenderKeyStore.swift:95-107`.
- In-depth cross-link: [Cryptography/sealed-sender.md](SignalServiceKit/Cryptography/sealed-sender.md).

## GV2 (Groups V2)

The second-generation group model (server-enforced membership via zero-knowledge
group credentials), as opposed to legacy V1 groups. The `GroupsVersion` enum
defines `GroupsVersionV1 = 0` and `GroupsVersionV2`; modern groups report
`.V2`. **[High]**

- Enum definition: `SignalServiceKit/Groups/TSGroupModel.h:19-21`
  (`GroupsVersionV1 = 0`, `GroupsVersionV2`).
- Modern groups return `.V2`: `SignalServiceKit/Groups/TSGroupModel.swift:200-201`.
- Version checks in use: `SignalServiceKit/Threads/TSThread+OWS.swift:27-34`
  (`== .V1` / `== .V2`); `SignalServiceKit/Groups/GroupManager.swift:772`.
- Implementation: `SignalServiceKit/Groups/GroupsV2Impl.swift`.

## SVR (Secure Value Recovery)

Server-side, enclave-backed recovery of the user's **master key** material via a
PIN, so a user can restore after reinstall. The `SVR` enum defines the auth
methods used to talk to the SVR server and the restore-result cases
(`success(MasterKey)`, `invalidPin(remainingAttempts:)`, `backupMissing`, …).
PIN normalization/hashing is in `SVRUtil`. **[High]**

- `SVR` enum (auth methods, restore results, `maximumKeyAttempts`):
  `SignalServiceKit/SecureValueRecovery/SecureValueRecovery.swift:9-40`.
- PIN normalization/hashing: `SignalServiceKit/SecureValueRecovery/SVRUtil.swift:8-40`.
- SVR2 implementation: `SignalServiceKit/SecureValueRecovery/SecureValueRecovery2Impl.swift`.
- In-depth cross-link:
  [SecureValueRecovery/secure-value-recovery.md](SignalServiceKit/SecureValueRecovery/secure-value-recovery.md).

## Storage service

An encrypted, server-hosted store of account/contact/group **manifest + records**
that syncs across a user's devices. Encryption is keyed by material derived from
the master key (see SSK). `StorageService` defines the manifest/record error
types and the `MasterKeySource` used when restoring/creating a manifest. **[High]**

- `StorageService` struct + errors + `MasterKeySource`:
  `SignalServiceKit/StorageService/StorageService.swift:7-35`.
- Restore/create entry point:
  `SignalServiceKit/StorageService/StorageServiceManager.swift:40`
  (`restoreOrCreateManifestIfNecessary`).

## SSK (Storage Service Key)

The key used to encrypt storage-service data, **derived from the master key** via
HKDF-style HMAC with info string `"Storage Service Encryption"`. From it are
further derived per-version manifest keys and legacy record keys.

> Note: in this codebase `SSK` as a *module/prefix* also commonly abbreviates
> **SignalServiceKit** (e.g. `SSKEnvironment`,
> `SignalServiceKit/Environment/SSKEnvironment.swift:1`). The glossary entry here
> is the **Storage Service Key** sense, which is the key-material meaning. **[High]**
> for the key derivation; **[Medium]** for which `SSK` sense is intended in any
> given mention — intent of the acronym is context-dependent.

- `StorageServiceKey` struct + `deriveFrom(masterKey:)` (info string
  `"Storage Service Encryption"`): `SignalServiceKit/Account/MasterKey.swift:171-190`.
- `MasterKey.deriveStorageServiceKey()`: `SignalServiceKit/Account/MasterKey.swift:59-61`.
- `SSKEnvironment` (the SignalServiceKit-prefix sense):
  `SignalServiceKit/Environment/SSKEnvironment.swift:1`.

## Interaction (`TSInteraction`)

The base model for anything that appears in a conversation timeline — incoming/
outgoing messages, calls, errors, info messages, typing indicators, etc. The
`OWSInteractionType` enum enumerates the kinds. Each interaction belongs to a
thread via `threadUniqueId`. **[High]**

- Base class: `SignalServiceKit/Messages/Interactions/TSInteraction.h:40`
  (`@interface TSInteraction : BaseModel`).
- Interaction-type enum: `SignalServiceKit/Messages/Interactions/TSInteraction.h:12-26`
  (`OWSInteractionType`).
- Belongs to a thread: `SignalServiceKit/Messages/Interactions/TSInteraction.h:51-54`
  (`threadUniqueId:` designated initializer).

## Thread (`TSThread`)

A conversation: 1:1 contact (`TSContactThread`), group (`TSGroupThread`), private
story, or release notes. `TSThreadType` enumerates concrete record types; a
thread owns its interactions. **[High]**

- Base class + record types: `SignalServiceKit/Contacts/TSThread.swift:16-37`
  (`TSThreadType`, `TSThread`, `concreteType(forRecordType:)`).

## DM timer (disappearing-messages timer)

Per-conversation setting controlling after how long messages auto-delete, stored
as a `DisappearingMessagesConfigurationRecord` and set via a versioned token.
Versions exist so linked devices can stay in sync; `resetAllDMTimerVersions`
resets them when devices may be out of sync. **[High]**

- Store protocol (`set(token:…)`, `resetAllDMTimerVersions`, "Keep all DM timers"
  comment): `SignalServiceKit/DisappearingMessages/DisappearingMessagesConfigurationStore.swift:8-31`.
- Record model: `SignalServiceKit/Contacts/DisappearingMessagesConfigurationRecord.swift`.
- Timeline marker interaction type `OWSInteractionType_DefaultDisappearingMessageTimer`:
  `SignalServiceKit/Messages/Interactions/TSInteraction.h:23`.
- In-depth cross-link:
  [DisappearingMessages/README.md](SignalServiceKit/DisappearingMessages/README.md).

## CDS (Contact Discovery Service)

The server service used to discover which of the user's phone-number contacts
are Signal users, without revealing the full contact list. The
`ContactDiscoveryManager` coordinates CDS lookups: serializing stateful requests,
respecting rate limits, and avoiding repeated lookups of the same numbers.
**[High]**

- `ContactDiscoveryManager` protocol + CDS responsibilities doc comment:
  `SignalServiceKit/Contacts/Discovery/ContactDiscoveryManager.swift:8-48`
  (`lookUp(phoneNumbers:mode:)`).
- `ContactDiscoveryMode`: `SignalServiceKit/Contacts/Discovery/ContactDiscoveryManager.swift:60+`.
- CDSv2 operation / errors: `SignalServiceKit/Contacts/Discovery/ContactDiscoveryV2Operation.swift`,
  `SignalServiceKit/Contacts/Discovery/ContactDiscoveryError.swift`.

## Registration session (verification session)

The server-side phone-number verification session half of registration: an
opaque session id tied to an E164, used to satisfy anti-abuse challenges, request
SMS/voice codes, and submit a code. It does **not** itself create the account —
a verified session is handed to the registration coordinator. **[High]**

- Persisted model (`id`, `e164`, `receivedDate`, `nextSMS`, `nextCall`):
  `SignalServiceKit/Registration/RegistrationSession.swift:14-40`.
- Manager API: `SignalServiceKit/Registration/RegistrationSessionManager.swift:8-30`
  (`beginOrRestoreSession`, `requestVerificationCode`, `submitVerificationCode`).
- In-depth cross-link:
  [Registration/registration-sessions.md](SignalServiceKit/Registration/registration-sessions.md).

## Device linking / provisioning

Adding a secondary ("linked") device to an existing account by provisioning it
with identity keys from the primary. The primary device is always device ID 1;
a `DeviceType` is `.primary` or `.linked`. The `ProvisioningManager.provision`
flow and `DeviceProvisioningService.provisionDevice` drive it. **[High]**

- Device identity: `SignalServiceKit/Account/DeviceId.swift:9-14`
  (`DeviceId.primary` = 1); `DeviceType` enum
  (`SignalServiceKit/Account/DeviceType.swift:8-11`).
- Provisioning entry points: `Signal/Provisioning/ProvisioningManager.swift:10-44`
  (`class ProvisioningManager`, `func provision`);
  `SignalServiceKit/Network/API/DeviceProvisioningService.swift:29-56`
  (`provisionDevice(messageBody:ephemeralDeviceId:)`).
- Prekeys for provisioning: `SignalServiceKit/Axolotl/PreKeyManager.swift:20-27`
  (`createPreKeysForProvisioning`).
- In-depth cross-link:
  [Devices/device-linking-and-sync.md](SignalServiceKit/Devices/device-linking-and-sync.md).

## Message Send Log (MSL)

A short-lived log of sent-message payloads kept so the client can **resend** a
specific payload if the server reports a delivery issue (e.g. a device needs the
message again). Entries expire after a remote-config lifetime and are periodically
cleaned up; they are keyed per interaction and deleted when the interaction is
deleted. **[High]**

- `MessageSendLog` class, payload lifetime/cleanup constants, `Payload` record
  (table `MessageSendLog_Payload`):
  `SignalServiceKit/Messages/MessageSendLog.swift:17-40`.
- Record a payload: `SignalServiceKit/Messages/MessageSendLog.swift:87`
  (`recordPayload(_:for:tx:)`).
- Delete on interaction deletion: `SignalServiceKit/Messages/MessageSendLog.swift:11-14`
  (`deleteAllPayloads(forInteraction:tx:)`).
- Purpose ("resend"): **[Medium]** — inferred from the record/delete-per-interaction
  structure and the per-payload lifetime; the exact resend trigger is handled by
  message-sending code outside this file.

---

## Related in-depth docs

- Identity / account / keys:
  [Account/identity-and-registration.md](SignalServiceKit/Account/identity-and-registration.md),
  [Cryptography/identity-and-keys.md](SignalServiceKit/Cryptography/identity-and-keys.md)
- Crypto primitives:
  [Cryptography/prekeys.md](SignalServiceKit/Cryptography/prekeys.md),
  [Cryptography/sealed-sender.md](SignalServiceKit/Cryptography/sealed-sender.md),
  [Cryptography/sessions-and-ratchet.md](SignalServiceKit/Cryptography/sessions-and-ratchet.md),
  [Cryptography/key-transparency.md](SignalServiceKit/Cryptography/key-transparency.md)
- Recovery / sync:
  [SecureValueRecovery/secure-value-recovery.md](SignalServiceKit/SecureValueRecovery/secure-value-recovery.md),
  [Devices/device-linking-and-sync.md](SignalServiceKit/Devices/device-linking-and-sync.md)
- Registration & phone number:
  [Registration/registration-sessions.md](SignalServiceKit/Registration/registration-sessions.md),
  [ChangePhoneNumber/change-phone-number.md](SignalServiceKit/ChangePhoneNumber/change-phone-number.md)
- Disappearing messages:
  [DisappearingMessages/README.md](SignalServiceKit/DisappearingMessages/README.md)

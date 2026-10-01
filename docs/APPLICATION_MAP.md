# Application Map

A directory/target map of the whole Signal-iOS repository: what lives where, which
Xcode targets the top-level directories feed, and — for the major source targets —
their principal subdirectories, what each contains, and representative file paths.

This document is a **navigation aid**. The authoritative classification of every path
into a module/category is the module boundary map in
[INVENTORY.md](INVENTORY.md); where this map and the inventory disagree, the inventory
wins. (Note: [INVENTORY.md](INVENTORY.md) currently contains unresolved Git merge
conflict markers — `<<<<<<< ours` / `=======` / `>>>>>>> theirs` — at the time of
writing; **[High]**, directly observed in the file.)

## Conventions

- **Citations** are given as `path` or `path:line`. A path points at a directory or
  representative file; a `path:line` points at a specific declaration. Line numbers
  reflect the tree at authoring time and may drift; use the symbol name to relocate.
- **Confidence labels** on claims about *purpose/behavior*:
  - **[High]** — directly observed (file read, declaration located, or directory
    listed by tooling in this session).
  - **[Medium]** — inferred from file/directory names, signatures, and cross-file
    convention; not every file in the area was read.
  - **[Low]** — educated inference from naming alone.
- Any uncited claim is a **defect**. Where the source gives no evidence of intent, the
  text says **"intent undetermined — no evidence in source."**
- Directory existence and file sizes/paths in this document were produced by listing
  the working tree in this session **[High]**.

---

## 1. Top-level layout

The repository root contains five first-party module directories, build/CI/config
directories, vendored dependencies, and repo-level docs/legal files. The following
were observed by listing the repository root **[High]** (`/`):

| Top-level dir | Role | Xcode target(s) it primarily feeds | Doc |
|---|---|---|---|
| `Signal/` | Main iOS application (UI, app lifecycle, feature VCs) | `Signal` app | §2 below; file-level docs under [docs/](.) are not yet present for this module |
| `SignalServiceKit/` | Core service/data layer (networking, storage, crypto, messaging, groups, accounts) | `SignalServiceKit` framework | §3 below; [SignalServiceKit/README.md](SignalServiceKit/README.md) |
| `SignalUI/` | Shared UI layer used by the app + extensions | `SignalUI` framework | §4 below |
| `SignalShareExtension/` | Share-sheet app extension (SAE) | `SignalShareExtension` extension | §5 below; [SignalShareExtension/README.md](SignalShareExtension/README.md) |
| `SignalNSE/` | Notification Service Extension (NSE) | `SignalNSE` extension | §6 below |
| `Scripts/` | Build/codegen/lint/translation tooling | build-time only | §7 below; [BUILD_ENV.md](BUILD_ENV.md) |
| `Config/` | `.xcconfig` build settings | all targets (build settings) | §7 below; [BUILD_ENV.md](BUILD_ENV.md) |
| `ci_scripts/` | Xcode Cloud CI hooks | CI | §7 below; [BUILD_ENV.md](BUILD_ENV.md) |
| `fastlane/` | Release automation + App Store metadata | release/CI | §7 below |
| `ThirdParty/` | Vendored podspecs (non-submodule) | dependency resolution | §8 below |
| `Pods/` | CocoaPods dependencies (git submodule) | all targets (link) | §8 below |

Supporting root files observed **[High]**: `Podfile`, `Podfile.lock`, `Gemfile`,
`Gemfile.lock`, `Makefile`, `Signal.xcodeproj/` (`project.pbxproj` ≈ 2.4 MB),
`Signal.xcworkspace/`, `.swiftformat`, `.clang-format`, `.xcode-version`,
`.ruby-version`, `.gitmodules`, and legal/meta files `README.md`, `BUILDING.md`,
`CONTRIBUTING.md`, `MAINTAINING.md`, `SECURITY.md`, `LICENSE`.

**Submodules** (from `.gitmodules`, read in full **[High]**):
- `Pods` → `https://github.com/signalapp/Signal-Pods.git`
- `SignalServiceKit/tests/MessageBackup/Signal-Message-Backup-Tests` →
  `https://github.com/signalapp/Signal-Message-Backup-Tests`

Both are typically **not checked out** in a bare working tree (see
[INVENTORY.md](INVENTORY.md) "Vendored dependencies"); `Pods/` was observed empty in
this session **[High]**.

---

## 2. `Signal/` — main application target

The main app. Contains app lifecycle, the conversation UI, settings, registration UI,
calling UI, and numerous feature areas. Mixed source, assets, and localization.
Principal subdirectories (listed from the tree **[High]**; purpose **[Medium]** unless a
file was read):

### App lifecycle & environment
- `Signal/AppLaunch/` — process entry and app wiring. `AppDelegate.swift:14`
  (`final class AppDelegate`) **[High]**; `SignalApp.swift:16` (`public class SignalApp`)
  **[High]**; also `SceneDelegate.swift`, `MainAppContext.swift`, `AppEnvironment.swift`,
  `AppLifecycleManager.swift` (~91 KB), `WindowManager.swift`, `LoadingViewController.swift`.
- `Signal/Preconditions/`, `Signal/ClockSkew/`, `Signal/LowDiskSpace/`,
  `Signal/Expiration/` — small app-readiness/monitoring helpers (e.g.
  `OsExpiry.swift`, `ClockSkewMonitoringManager.swift`) **[Medium]**.

### Conversation UI
- `Signal/ConversationView/` — the conversation screen and input toolbar.
  `ConversationViewController.swift:37` (`public final class ConversationViewController`)
  **[High]**, plus many `ConversationViewController+*.swift` extensions,
  `ConversationInputToolbar.swift` (~135 KB), `ConversationViewLayout.swift`,
  and `CVViewState.swift` / `CVCell.swift`. Subfolders: `Components/`, `CellViews/`,
  `Reactions/`, `Loading/`, `DynamicInteractions/`, `DoubleTapToEdit/`, `VoiceMessage/`.
- `Signal/src/ViewControllers/` — the bulk of feature view controllers **[High]**.
  Notable subfolders: `HomeView/` (chat list, `ConversationSplitViewController.swift`,
  `HomeTabBarController.swift`, `Stories/`, `Chat List/`), `AppSettings/`
  (`Account/`, `Privacy/`, `Notifications/`, `Appearance/`, `Data Usage/`, `Donations/`,
  `Linked Devices/`, `Internal/`, `Payments/`, `Profile/`), `ThreadSettings/`,
  `MediaGallery/` (`MediaGallery.swift` ~47 KB, `MediaTileViewController.swift` ~71 KB),
  `Photos/` (`PhotoCaptureViewController.swift` ~104 KB, `CameraCaptureSession.swift`),
  `Donations/`, `Payments/`, `Polls/`, `PinnedMessages/`, `NewGroupView/`, `GifPicker/`,
  `Stickers/`, `DebugUI/`, `ContextMenus/`, `Wallpapers/`, `RecipientPicker/`,
  `MemberLabels/`, `Avatars/`, `Attachment Keyboard/`, `Categories/`.
- `Signal/src/views/`, `Signal/src/View Supplements/` — supporting view code **[Medium]**.

### Calling (app side)
- `Signal/Calls/` — call orchestration/UI glue. `CallService.swift:19`
  (`final class CallService`, file ~69 KB) **[High]**; `IndividualCallService.swift`
  (~58 KB), `CallAudioService.swift`, `GroupThreadCall.swift`, `CallLink*` managers,
  `CallKitCallManager.swift`, and `UserInterface/`. Links to SSK call model/doc:
  [SignalServiceKit/Calls/README.md](SignalServiceKit/Calls/README.md).

### Registration / provisioning / device transfer (app side)
- `Signal/Registration/` — `RegistrationCoordinatorImpl.swift:21`
  (`public class RegistrationCoordinatorImpl`, file ~222 KB) **[High]**;
  `RegistrationCoordinatorLoader.swift`, `RegistrationStep.swift`, and `UserInterface/`.
  Related SSK doc: [SignalServiceKit/Registration/registration-sessions.md](SignalServiceKit/Registration/registration-sessions.md).
- `Signal/Provisioning/` — `ProvisioningCoordinatorImpl.swift`, `ProvisioningManager.swift`,
  `ProvisioningSocketManager.swift`, `DeviceProvisioningURL.swift` **[Medium]**.
- `Signal/QuickRestore/` — outgoing device-restore flow (`QuickRestoreManager.swift`,
  `OutgoingDeviceRestore*`) **[Medium]**. Related SSK doc:
  [SignalServiceKit/Devices/device-linking-and-sync.md](SignalServiceKit/Devices/device-linking-and-sync.md).
- `Signal/DeviceTransfer/` — peer-to-peer transfer (`OutgoingDeviceTransferTask.swift`
  ~26 KB, `IncomingDeviceTransferTask.swift`, `MultiPeerConnectivity/`, `WiFiAware/`) **[Medium]**.

### Other feature areas (app side)
- `Signal/Backups/` — backup settings/onboarding UI (`BackupSettingsViewController.swift`
  ~155 KB, `BackupEnablingManager.swift`, `RecoveryKey/`, `Onboarding/`) **[Medium]**.
  Related SSK doc: [SignalServiceKit/Attachments/Backfill.md](SignalServiceKit/Attachments/Backfill.md) (adjacent backup/attachment flows).
- `Signal/Notifications/` — `PushRegistrationManager.swift`, `NotificationActionHandler.swift`,
  `BadgeManager.swift`, `MessageFetchBGRefreshTask.swift` **[Medium]**.
- `Signal/Emoji/` — emoji picker + `Generated/` tables **[Medium]**.
- `Signal/Usernames/`, `Signal/Contacts/`, `Signal/Groups/`, `Signal/Profiles/`,
  `Signal/Spam/`, `Signal/Avatars/`, `Signal/Megaphones/`, `Signal/OrphanData/`,
  `Signal/Storage/`, `Signal/Sharing/`, `Signal/Attachments/`, `Signal/Axolotl/`,
  `Signal/Autofill/`, `Signal/Screenshots/`, `Signal/Accessibility/`, `Signal/URLs/`,
  `Signal/QRCodes/` — feature-specific app glue, each a handful of files **[Medium]**.
- `Signal/util/` — app-level utilities (`DisplayableText.swift`, `ScreenLockUI.swift`,
  `VolumeButtons.swift`, `BatchUpdate.swift`) **[High]**, listing observed.

### Non-code resources in `Signal/`
- Assets: `Images.xcassets/`, `Symbols.xcassets/`, `AppIcon.xcassets/`,
  `NSE-Images.xcassets/`, `AppIcons/*.icon`, `Lottie/` (animation JSON),
  `Sounds/`, `AudioFiles/` **[High]**.
- Localization: `Signal/translations/*.lproj` (per-locale `.lproj` dirs) **[High]**.
- Config: `Signal-Info.plist`, `Signal.entitlements`,
  `Signal-AppStore.entitlements`, `PrivacyInfo.xcprivacy`, `Signal-Prefix.pch`,
  `Settings.bundle/` **[High]**.
- `Signal/test/` — the app target's XCTest sources (`SignalBaseTest.swift`, plus per-area
  subfolders: `Backups/`, `Calls/`, `Registration/`, `ViewControllers/`, `Payments/`, …)
  **[High]**. These are **Tests** per [INVENTORY.md](INVENTORY.md), not first-party app code.

---

## 3. `SignalServiceKit/` — core service/data layer framework

The largest module. Networking, persistence, cryptography, message pipeline, groups,
accounts/identity, payments, backups, and shared utilities. The module overview doc is
[SignalServiceKit/README.md](SignalServiceKit/README.md). Principal subdirectories
(listed from the tree **[High]**; cross-links point to existing docs):

### Environment / dependency wiring
- `SignalServiceKit/Environment/` — process environment and DI. `SSKEnvironment.swift:8`
  (`public class SSKEnvironment`) **[High]**; `AppSetup.swift:11` (`public class AppSetup`,
  file ~110 KB) **[High]**; `DependenciesBridge.swift:26` (`public class DependenciesBridge`)
  **[High]**; also `RemoteConfigManager.swift` (~49 KB), `TSConstants.swift`,
  `BuildFlags.swift` / `BuildFlags+Generated.swift`.

### Networking
- `SignalServiceKit/Network/` — HTTP/websocket stack: `OWSChatConnection.swift` (~46 KB),
  `OWSUrlSession.swift` (~42 KB), `ChatConnectionManager.swift`, `OWSSignalService.swift`,
  `OWSCensorshipConfiguration.swift`, `SignalProxy/`, and `API/`. Documented in depth under
  [SignalServiceKit/Network/README.md](SignalServiceKit/Network/README.md) (01–09 files).

### Messaging pipeline
- `SignalServiceKit/Messages/` — send/receive core. `MessageReceiver.swift:62`
  (`public final class MessageReceiver`, file ~120 KB) **[High]**; in
  `MessageSender.swift` (~96 KB) `MessageSender` is a protocol and the concrete
  implementation is `MessageSender.swift:28` (`public class MessageSenderImpl: MessageSender`)
  **[High]** (see also `MessageSender+SenderKey.swift`, `MessageSender+Errors.swift`);
  `MessageProcessor.swift`, `OWSMessageDecrypter.swift`,
  `OWSIdentityManager.swift` (~45 KB), `OWSReceiptManager.swift`, `BlockingManager.swift`,
  `RecipientHidingManager.swift`, `EarlyMessageManager.swift`. Subfolders: `Interactions/`,
  `Attachments/`, `BodyRanges/`, `Edit/`, `DeviceSyncing/`, `Reactions/`, `Stickers/`,
  `Stories/`, `Payments/`, `UD/`, `InvalidKeyMessages/`, `OutgoingMessagePreparer/` **[Medium]**.

### Groups
- `SignalServiceKit/Groups/` — GroupsV2 engine. `GroupManager.swift:13`
  (`public class GroupManager`, file ~61 KB) **[High]**; `GroupsV2Impl.swift` (~96 KB),
  `GroupV2UpdatesImpl.swift` (~54 KB), `GroupMembership.swift`, `TSGroupModel.swift`,
  `GroupsV2Protos.swift` **[Medium]**.

### Accounts, identity, keys
- `SignalServiceKit/Account/` — identifiers/keys. `TSAccountManager/`, `ServiceId.swift`,
  `LocalIdentifiers.swift`, `MasterKey.swift`, `AccountEntropyPool.swift`,
  `PhoneNumberDiscoverabilityManager/`, `IdentityKeyMismatchManager.swift` **[Medium]**.
  Documented in [SignalServiceKit/Account/identity-and-registration.md](SignalServiceKit/Account/identity-and-registration.md).
- `SignalServiceKit/Axolotl/` — Signal protocol stores and prekeys (`SessionStore.swift`,
  `SenderKeyManager.swift`, `PreKeyManagerImpl.swift`, `PreKeyTaskManager.swift`) **[Medium]**.
  Documented under [SignalServiceKit/Cryptography/](SignalServiceKit/Cryptography/README.md)
  (`prekeys.md`, `sessions-and-ratchet.md`, `identity-and-keys.md`).
- `SignalServiceKit/Cryptography/` — `Cryptography.swift` (~56 KB), `CipherContext.swift`,
  `Aes256Key.swift`, `Sha256HmacSiv.swift` **[High, sizes observed]**. Doc:
  [SignalServiceKit/Cryptography/README.md](SignalServiceKit/Cryptography/README.md).
- `SignalServiceKit/SecureValueRecovery/` — SVR2 / PIN (`SecureValueRecovery2Impl.swift`
  ~29 KB, `SgxWebsocketConnection.swift`, `SVR2PinHash.swift`) **[Medium]**. Doc:
  [SignalServiceKit/SecureValueRecovery/secure-value-recovery.md](SignalServiceKit/SecureValueRecovery/secure-value-recovery.md).
- `SignalServiceKit/KeyTransparency/` — `KeyTransparencyManager.swift` (~29 KB) **[Medium]**.
  Doc: [SignalServiceKit/Cryptography/key-transparency.md](SignalServiceKit/Cryptography/key-transparency.md).
- `SignalServiceKit/ZeroKnowledge/` — auth credential managers (`AuthCredentialManager.swift`,
  `BackupAuthCredentialManager.swift`) **[Medium]**.

### Storage & persistence
- `SignalServiceKit/Storage/` — `Database/`, `MediaGallery/`, `YDBStorage.swift`,
  `SSKKeychainStorage.swift`, Obj-C `TSYapDatabaseObject.{h,m}`, `BaseModel.{h,m}` **[High]**.
- `SignalServiceKit/StorageService/` — storage-service sync: `StorageServiceManager.swift`
  (~115 KB), `StorageServiceProto+Sync.swift` (~105 KB), `StorageService.swift` **[High, sizes
  observed]**.
- `SignalServiceKit/Stores/`, `SignalServiceKit/PinnedThreadManager/`,
  `SignalServiceKit/Threads/`, `SignalServiceKit/Contacts/` — thread/recipient stores and
  the contacts manager. `Contacts/OWSContactsManager.swift:58`
  (`public class OWSContactsManager`, file ~59 KB) **[High]**; `RecipientMerger.swift` (~46 KB),
  `SignalServiceAddress.swift`, `TSThread.swift`, `Discovery/`, `SignalRecipient/`,
  `NicknameRecord/` **[Medium]**.

### Backups
- `SignalServiceKit/Backups/` — `Archiving/`, `Attachments/`, `BackupExportJob/`,
  `LocalFileBackup/`, `Settings/`, `BackupRequestManager.swift` (~22 KB),
  `MessageRootBackupKey.swift`, `MediaRootBackupKey.swift` **[Medium]**.
- `SignalServiceKit/AttachmentBackfill/` — `AttachmentBackfillManager.swift` (~27 KB) and
  sync-message records **[Medium]**. Doc:
  [SignalServiceKit/Attachments/Backfill.md](SignalServiceKit/Attachments/Backfill.md).

### Attachments / media
- `SignalServiceKit/Attachments/` — limits/padding (`AttachmentLimits.swift`,
  `PaddingBucket.swift`) **[High]**; the V2 attachment model lives under
  `SignalServiceKit/Messages/Attachments/` **[High]**. `SignalServiceKit/Upload/`
  — `AttachmentUploadManager.swift` (~51 KB), CDN2/CDN3 endpoints **[Medium]**.
  Docs: [SignalServiceKit/Attachments/README.md](SignalServiceKit/Attachments/README.md)
  (Core, V2-Model, ContentValidation, Store-and-Records, Manager, Downloads, Upload,
  Media, OrphanedAttachments).

### Calls (model/service side)
- `SignalServiceKit/Calls/` — `CallRecord/`, `Group/`, `Individual/`, `DeletedCallRecord/`,
  `GroupCallManager.swift`, `CallLinkRecord.swift`, RingRTC config
  (`RingrtcFieldTrials.swift`, `RingrtcSvcConfig.swift`, `RingrtcVp9Config.swift`) **[Medium]**.
  Doc: [SignalServiceKit/Calls/README.md](SignalServiceKit/Calls/README.md).

### Other SSK areas
- `SignalServiceKit/Jobs/` — durable job queues (`MessageSenderJobQueue.swift`,
  `JobQueueRunner.swift`, `Cron.swift`, `JobRecords/`) **[Medium]**.
- `SignalServiceKit/Notifications/` — `NotificationPresenterImpl.swift` (~76 KB),
  `UserNotificationsPresenter.swift` **[High, size observed]**.
- `SignalServiceKit/Payments/`, `SignalServiceKit/Subscriptions/` — MobileCoin payment
  models and donation/subscription flows (`PaymentsHelperImpl.swift`,
  `DonationReceiptCredentialRedemptionJobQueue.swift` in `Jobs/`) **[Medium]**.
- `SignalServiceKit/Profiles/` — `OWSProfileManager.swift` (~74 KB),
  `OWSUserProfile.swift` (~57 KB), `ProfileFetcherJob.swift` **[High, sizes observed]**.
- `SignalServiceKit/Devices/` — `LinkAndSyncManager.swift` (~33 KB), `OWSDeviceService.swift`,
  `SyncMessages/`, `SentMessageTranscriptReceiver/`, `ConversationSync/` **[Medium]**. Doc:
  [SignalServiceKit/Devices/device-linking-and-sync.md](SignalServiceKit/Devices/device-linking-and-sync.md).
- `SignalServiceKit/Registration/` — `RegistrationSessionManagerImpl.swift` (~24 KB),
  `RegistrationSession.swift` **[Medium]**. Doc:
  [SignalServiceKit/Registration/registration-sessions.md](SignalServiceKit/Registration/registration-sessions.md).
- `SignalServiceKit/ChangePhoneNumber/` — PNI change management **[Medium]**. Doc:
  [SignalServiceKit/ChangePhoneNumber/change-phone-number.md](SignalServiceKit/ChangePhoneNumber/change-phone-number.md).
- `SignalServiceKit/DisappearingMessages/` + `SignalServiceKit/Expiration/` — DM config
  and the generic expiry job **[Medium]**. Doc:
  [SignalServiceKit/DisappearingMessages/README.md](SignalServiceKit/DisappearingMessages/README.md).
- `SignalServiceKit/VoiceMessage/` — interrupted-draft store **[Medium]**. Doc:
  [SignalServiceKit/VoiceMessage/README.md](SignalServiceKit/VoiceMessage/README.md).
- `SignalServiceKit/Usernames/`, `Spam/`, `Stories/`, `Megaphones/`, `Search/`,
  `SafetyTips/`, `QRCodes/`, `UISupport/`, `Avatars/`, `Dates/`, `Concurrency/`,
  `LocalDeviceAuth/`, `LowDiskSpace/`, `ClockSkew/`, `Preconditions/` — feature-scoped
  services, each a modest file set **[Medium]**.
- `SignalServiceKit/Util/` — large shared-utility grab-bag (`MimeTypeUtil.swift` ~100 KB,
  `OWSProgress.swift`, `OWSFileSystem.swift`, `String+SSK.swift`, `AvatarBuilder`-adjacent
  helpers, `ImageMetadata/`, `StreamTransform/`) **[High, sizes observed]**.
- `SignalServiceKit/PromiseKit/` — in-tree Promise/Guarantee implementation **[High]**.
- `SignalServiceKit/Resources/Certificates/` — pinned certs **[High]**.

### Generated & vendored inside SSK (not first-party)
- `SignalServiceKit/Protos/` — protobuf outputs under `Protos/Generated/` and `Protos/Backups/`,
  plus `Protos/Specifications/` and a `Makefile`; **Generated** per
  [INVENTORY.md](INVENTORY.md) **[High, listing observed]**.
- `*+SDS.swift` files (e.g. `Messages/OWSUnknownProtocolVersionMessage+SDS.swift`) are
  SDSCodeGenerator output — **Generated**, not first-party **[High]**.
- `SignalServiceKit/tests/`, `Mocks/`, `TestUtils/` — **Tests** per the inventory; includes
  the `tests/MessageBackup/Signal-Message-Backup-Tests` submodule (**Vendored**) **[High]**.

---

## 4. `SignalUI/` — shared UI framework

Reusable UI used by `Signal`, `SignalShareExtension`, and (partially) `SignalNSE`. Entry
header `SignalUI/SignalUI.h` **[High]**. Principal subdirectories (listed **[High]**;
purpose **[Medium]**):

- `SignalUI/Appearance/` — theming. `Theme.swift:12` (`public final class Theme`) **[High]**;
  `UIColor+Signal.swift`, `ConversationStyle.swift`, `SignalSymbols.swift`, `SwiftUI/`.
- `SignalUI/ViewControllers/` — base VCs: `OWSViewController.swift`,
  `OWSTableViewController2.swift` (~44 KB), `OWSNavigationController.swift`,
  `InteractiveSheetViewController.swift`, `ScanQRCodeViewController.swift`.
- `SignalUI/Views/` — reusable views (`ConversationAvatarView.swift` ~44 KB,
  `ManualStackView.swift`, `TextAttachmentView.swift`, `Toast.swift`, `BodyRanges/`,
  `Tooltips/`).
- `SignalUI/RecipientPickers/` — `RecipientPickerViewController.swift` (~59 KB),
  `ConversationPicker.swift` (~62 KB), `ContactsViewHelper.swift`.
- `SignalUI/ImageEditor/` — image editing (`ImageEditorCanvasView.swift` ~53 KB,
  `ImageEditorCropViewController.swift`, `ImageEditorView.swift`).
- `SignalUI/AttachmentApproval/`, `SignalUI/AttachmentMultisend/`, `SignalUI/Attachments/`,
  `SignalUI/VideoEditor/`, `SignalUI/Media/` — attachment capture/approval/send pipeline UI.
- `SignalUI/Payments/` — MobileCoin UI/API glue (`MobileCoinAPI.swift`,
  `PaymentsReconciliation.swift` ~50 KB, `PaymentsImpl.swift` ~44 KB).
- `SignalUI/Stories/`, `SignalUI/Stickers/`, `SignalUI/SafetyNumbers/`,
  `SignalUI/LinkPreview/`, `SignalUI/Search/`, `SignalUI/Calls/`, `SignalUI/ContactSharing/`,
  `SignalUI/Usernames/`, `SignalUI/Sending/`, `SignalUI/Wallpapers/`,
  `SignalUI/ActionSheets/`, `SignalUI/AppBlocking/`, `SignalUI/AV/` — feature UI areas.
- `SignalUI/UIKitExtensions/`, `SignalUI/SwiftUIExtensions/`, `SignalUI/Utils/`,
  `SignalUI/FormatStyles/`, `SignalUI/AppLaunch/` — extensions/utilities.
- `SignalUI/Fonts/` — bundled font files (`Inter-Variable.ttf`, `SignalSymbols-*.otf`,
  `fontawesome-webfont.ttf`) **[High]**.

No per-file docs exist under `docs/` for this module yet — **intent undetermined — no
evidence in source** as to whether SignalUI docs are planned.

---

## 5. `SignalShareExtension/` — Share App Extension (SAE)

A separate-process app extension that appears in the system share sheet. Principal
class `ShareViewController.swift:13` (`public class ShareViewController: OWSNavigationController`)
**[High]**. Other sources (all at the target root, listed **[High]**):
`SharingThreadPickerViewController.swift` (~28 KB), `SharingThreadPickerProgressSheet.swift`,
`ShareViewDelegate.swift`, `ShareAppExtensionContext.swift`, `SAEScreenLockViewController.swift`,
`SAELoadViewController.swift`. Supporting: `Info.plist`, `SignalShareExtension.entitlements`,
`SignalShareExtension-AppStore.entitlements`, `PrivacyInfo.xcprivacy`.

Fully documented (one doc per source file) under
[SignalShareExtension/README.md](SignalShareExtension/README.md).

---

## 6. `SignalNSE/` — Notification Service Extension (NSE)

A separate-process extension that handles incoming push payloads to decrypt and present
notifications. Sources (listed **[High]**): `NotificationService.swift:33`
(`class NotificationService: UNNotificationServiceExtension`) **[High]**,
`NSEEnvironment.swift`, `NSEContext.swift`, `NSECallMessageHandler.swift`,
`NSELogger.swift`. Supporting: `Info.plist`, `SignalNSE.entitlements`,
`SignalNSE-AppStore.entitlements`, `PrivacyInfo.xcprivacy`.

No dedicated doc exists under `docs/` for this module yet — **intent undetermined — no
evidence in source**.

---

## 7. Build / CI / configuration

- `Scripts/` — build-time tooling (listed **[High]**): `precommit.py`, `build-and-test.sh`,
  `EmojiGenerator.swift` + `emoji_ranges.py` + `emoji-data.txt`, `sds_codegen/`,
  `protos/`, `lint/`, `translation/` + `translation-tool/` + `translation-validator/`,
  `git_hooks/`, `sort-Xcode-project-file`, `check_xcode_version.py`,
  `bump_build_tag.py`, `feature_flags_*.py`, `symbolicate.py`, `sqlclient`.
- `Config/` — `.xcconfig` build settings (listed **[High]**): `Project.xcconfig`,
  `Project-Debug.xcconfig`, `Project-Release.xcconfig`, `User.xcconfig.sample`.
- `ci_scripts/` — Xcode Cloud hooks (listed **[High]**): `ci_post_clone.sh`,
  `ci_pre_xcodebuild.sh`, `ci_post_xcodebuild.sh`, `send_build_notification.py`,
  `feature_flag_level.txt`, `tag_template.txt`.
- `fastlane/` — release automation (`Fastfile`, `Appfile`) and App Store `metadata/`
  (per-locale directories) **[High]**.
- Root build files: `Podfile`/`Podfile.lock`, `Gemfile`/`Gemfile.lock`, `Makefile`,
  `Signal.xcodeproj/`, `Signal.xcworkspace/` **[High]**.

The build environment is documented in detail in [BUILD_ENV.md](BUILD_ENV.md).

---

## 8. Vendored dependencies

- `Pods/` — CocoaPods dependencies, a git submodule (`.gitmodules` → `Signal-Pods.git`)
  **[High]**; observed empty (not checked out) in this session **[High]**.
- `ThirdParty/` — vendored podspecs not managed as submodules (listed **[High]**):
  `libwebp.podspec.json`, `blurhash.podspec`.

Per [INVENTORY.md](INVENTORY.md), both are classified **Vendored dependencies** and
excluded from first-party counts.

---

## 9. Where documentation lives today

The `docs/` tree mirrors module structure. Existing docs observed in this session **[High]**:

- Repo-level: [INVENTORY.md](INVENTORY.md), [BUILD_ENV.md](BUILD_ENV.md), this file.
- [SignalShareExtension/](SignalShareExtension/README.md) — complete, per-file.
- [SignalServiceKit/](SignalServiceKit/README.md) with subtrees: `Account/`,
  `Attachments/`, `Calls/`, `ChangePhoneNumber/`, `Cryptography/`, `Devices/`,
  `DisappearingMessages/`, `Network/`, `Registration/`, `SecureValueRecovery/`,
  `VoiceMessage/`.

No `docs/` coverage currently exists for `Signal/`, `SignalUI/`, or `SignalNSE/`
module-level docs — **intent undetermined — no evidence in source** as to whether these
are planned.

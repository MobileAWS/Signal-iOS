# View-Controller & App Subdirectory Map

A high-level map of the `Signal` target's UI and feature code — enough to navigate,
without restating every view controller. All claims cite `path` / `path:line`; see
[README.md](README.md) for conventions. For the whole-repo map, see
[../APPLICATION_MAP.md](../APPLICATION_MAP.md).

Directory existence/paths here were produced by listing the working tree in this session
**[High]**. Purpose of individual leaf files is **[Medium]** (inferred from name +
location) unless a file was read.

## Where the UI roots

The launch path installs one of three root view controllers (see
[launch-sequence.md](launch-sequence.md) §7): `ConversationSplitViewController` (chat
list), `RegistrationNavigationController`, or the provisioning flow
(`Signal/AppLaunch/SignalApp.swift:24-135`) **[High]**. The chat-list root lives under
`Signal/src/ViewControllers/HomeView/`.

## `Signal/src/ViewControllers/` — the feature VC layer

The bulk of the app's screens. Top-level subdirectories observed in the tree **[High]**:

| Subdir | Role |
|---|---|
| `HomeView/` | App shell + chat list + stories. See below. |
| `AppSettings/` | Settings tree. See below. |
| `MediaGallery/` | Conversation media browser (`MediaGallery/`, incl. large `MediaTileViewController`) **[Medium]**. |
| `Photos/` | In-app camera / photo capture + library picker **[Medium]**. |
| `Donations/` | Donation flows (badges, payment methods) **[Medium]**. |
| `Payments/` | MobileCoin payments UI (distinct from `AppSettings/Payments/`) **[Medium]**. |
| `Polls/` | Poll creation/results UI **[Medium]**. |
| `PinnedMessages/` | Pinned-messages UI **[Medium]**. |
| `NewGroupView/` | New-group creation flow **[Medium]**. |
| `GifPicker/` | GIF search/picker **[Medium]**. |
| `Stickers/` | Sticker management/picker **[Medium]**. |
| `DebugUI/` | Internal/debug screens **[Medium]**. |
| `ContextMenus/` | Custom context-menu infrastructure **[Medium]**. |
| `Wallpapers/` | Chat wallpaper selection **[Medium]**. |
| `RecipientPicker/` | App-side recipient-picker wrappers **[Medium]**. |
| `MemberLabels/` | Group member labels/tags UI **[Medium]**. |
| `Avatars/` | Avatar editing/selection **[Medium]**. |
| `Attachment Keyboard/` | The attachment input keyboard **[Medium]**. |
| `Categories/` | Shared VC category/extension helpers **[Medium]**. |
| `ThreadSettings/` | Per-conversation settings **[Medium]**. |

Loose top-level files include `SafetyTipsViewController.swift`,
`ForwardMessageViewController.swift`, `MessageDetailViewController.swift`,
`SendMessageFlow.swift`, `GroupInviteLinksUI.swift`, `PinSetupViewController.swift` /
`PinReminderViewController.swift` / `PinConfirmationViewController.swift`,
`DatabaseRecoveryViewController.swift` (used by the launch-failure recovery path, see
[launch-sequence.md](launch-sequence.md) §6), `LongTextViewController.swift`,
`LocationPicker.swift`, and others **[High]** (listed from the tree).

### `HomeView/` — app shell + chat list

Observed files/subdirs **[High]**:

- `ConversationSplitViewController.swift` (`~30 KB`) — the chat-list root installed by
  `SignalApp.showConversationSplitView` (`SignalApp.swift:26-33`) **[High]**; a
  `UISplitViewController`-style shell holding the chat list and the selected conversation.
- `HomeTabBarController.swift` (`~16 KB`), `HomeTabViewController.swift` — the tab shell
  (Chats / Calls / Stories) **[Medium]**.
- `Chat List/` — the chat-list screen proper: `ChatListViewController.swift` (`~69 KB`)
  plus many `ChatListViewController+*.swift` extensions (Search, Reminders, Notifications,
  Multiselect, Loading, Camera, Actions, backup progress views), the `CLV*` view-state /
  loader / render-state / data-source files, `ChatListCell.swift` (`~45 KB`), and
  filtering (`ChatListFilter*`) **[High]** (listed).
- `Stories/` — stories tab: `StoriesViewController.swift`, `MyStoriesViewController.swift`,
  `StoryListDataSource.swift`, cells/view-models, plus `Settings/`, `Context View/`,
  `Replies & Views Sheets/`, `Transitions/` **[High]** (listed).

### `AppSettings/` — the settings tree

Root `AppSettingsViewController.swift` (`~24 KB`) **[High]**, with these observed
subdirectories (**[High]**; each groups the screens its name implies, **[Medium]**):

| Subdir | Representative files |
|---|---|
| `Account/` | `AccountSettingsViewController.swift`, `DeleteAccountConfirmationViewController.swift`, `RequestAccountDataReportViewController.swift`, `AdvancedPinSettingsTableViewController.swift` |
| `Privacy/` | `PrivacySettingsViewController.swift`, `AdvancedPrivacySettingsViewController.swift`, `PhoneNumberPrivacySettingsViewController.swift`, `ProxySettingsViewController.swift`, `BlockListViewController.swift` |
| `Notifications/` | `NotificationSettingsViewController.swift` + sound/content/badge/while-muted screens |
| `Appearance/` | `AppearanceSettingsTableViewController.swift`, `ThemeSettingsTableViewController.swift`, `AppIconSettingsTableViewController.swift` |
| `Data Usage/` | `DataSettingsTableViewController.swift`, media-download/sent-quality/network-interface screens |
| `Donations/` | `DonationSettingsViewController.swift` (+`MySupport`), donation-receipt and badge-gifting screens |
| `Payments/` | `PaymentsSettingsViewController.swift` (`~56 KB`), passphrase/transfer/restore/history screens |
| `Linked Devices/` | `LinkedDevicesView.swift`, `LinkDeviceViewController.swift`, link/sync picker + restore-progress sheets |
| `Profile/` | `ProfileSettingsViewController.swift`, name/bio, badge configuration/collection |
| `Internal/` | `InternalSettingsViewController.swift`, `TestingViewController.swift`, `FlagsViewController.swift`, SQL/disk/media/backup internal tools |

Loose files include `ChatsSettingsViewController.swift`, `HelpViewController.swift`,
`ContactSupportViewController.swift`, `CurrencyPickerViewController.swift`,
`SupportKeyValueStore.swift` (whose `setLastChallengeDate` is written by
`SignalApp.spamChallenge`, `SignalApp.swift:87-100`, write at `:95`) **[High]**.

## The conversation UI (`Signal/ConversationView/`)

Not under `src/ViewControllers/` but central to the app: `Signal/ConversationView/` holds
`ConversationViewController.swift` plus a large family of `ConversationViewController+*.swift`
extensions, the input toolbar (`ConversationInputToolbar.swift`, `~135 KB`), layout
(`ConversationViewLayout.swift`), `CV*` state/cell/model files, and subfolders
`Components/`, `CellViews/`, `Reactions/`, `Loading/` (incl. `MessageLoader.swift`),
`DynamicInteractions/`, `DoubleTapToEdit/`, `VoiceMessage/` **[High]** (listed). The
conversation is presented inside `ConversationSplitViewController`
(`SignalApp.presentConversationForThread`, `SignalApp.swift:193`) **[High]**.

## Other first-party app subdirectories (not VCs)

Observed under `Signal/` **[High]**; these are app-side logic/managers rather than the VC
layer, and several are touched directly by the launch path:

| Dir | Role | Launch-path tie-in |
|---|---|---|
| `AppLaunch/` | Entry, launch, scene/window, environments | The subject of the other docs here. |
| `Calls/` | Call orchestration/UI glue (`CallService.swift` `~69 KB`, `IndividualCallService.swift`, CallKit, call links) | `CallService` built in `setUpMainAppEnvironment` (`AppLifecycleManager.swift:510-528`) **[High]**. |
| `Registration/` | Registration coordinator + UI | `.registration` launch interface (`SignalApp.swift:99-129`) **[High]**. |
| `Provisioning/` | Secondary-device provisioning | `.secondaryProvisioning` launch interface (`SignalApp.swift:131-135`) **[High]**. |
| `QuickRestore/` | Outgoing device-restore flow | `OutgoingDeviceRestorePresenter` built in `AppEnvironment.setUp` **[High]**. |
| `DeviceTransfer/` | Device-to-device transfer | `DeviceTransferRestore.launchCleanup()` runs early in launch (`AppLifecycleManager.swift:200`) **[High]**. |
| `Notifications/` | Push registration, badge, BG refresh, action handling | `PushRegistrationManager` eager in `AppEnvironment`; `MessageFetchBGRefreshTask`, `SyncPushTokensJob`, `NotificationActionHandler` on launch/activation **[High]**. |
| `Storage/` | App-side DB migrators / BG task runners | `LazyDatabaseMigratorRunner`, `BGProcessingTaskRunner` registered in `didFinishLaunching` (`AppLifecycleManager.swift:306-317`) **[High]**. |
| `Backups/` | Backup enabling/disabling/trackers + settings UI | Backup managers built in `AppEnvironment.setUp`; BG runners registered at launch **[High]**. |
| `ClockSkew/`, `LowDiskSpace/`, `Screenshots/`, `Expiration/`, `Preconditions/` | App-readiness/monitoring helpers | `ClockSkewMonitoringManager`, `LowDiskSpaceMonitoringManager`, `ScreenshotBlockingManager` started via `AppEnvironment.setUp` ready-blocks; `LowDiskSpaceManager.additionalBytesRequiredToLaunch()` is the first preflight check (`AppLifecycleManager.swift:1194`) **[High]**. |
| `URLs/` | URL parsing/opening (`UrlOpener.swift`) | `handleOpenUrl` → `UrlOpener` (`AppLifecycleManager.swift`, URL-handling section) **[High]**. |
| `Debugging/` | Debug logs + support action sheets | Used throughout launch-failure UI **[High]**. |
| `util/`, `src/views/`, `src/View Supplements/` | Shared app views/utilities (incl. `ScreenLockUI.swift`, `Launch Screen.storyboard`) | `ScreenLockUI` eager in `AppEnvironment`; `Launch Screen` storyboard is the terminal-error root (`AppLifecycleManager.swift:1508-1513`) **[High]**. |

Feature-area dirs with little or no launch-path involvement (purpose **[Medium]** from
name): `Attachments/`, `Avatars/`, `Autofill/`, `Axolotl/`, `Contacts/`, `Emoji/`,
`Groups/`, `Megaphones/`, `OrphanData/`, `Profiles/`, `QRCodes/`, `Sharing/`, `Spam/`,
`Usernames/`, `Accessibility/` **[High]** (dirs listed).

## Non-source app content

`Signal/` also contains non-source bundles observed in the tree **[High]**:
`Images.xcassets/`, `Symbols.xcassets/`, `AppIcon.xcassets/`, `AppIcons/`,
`NSE-Images.xcassets/`, `Lottie/` (animation JSON incl. `launchApp-iPhone.json` /
`launchApp-iPad.json`), `AudioFiles/`, `Sounds/`, `translations/` (localization), plus
target config files (`Signal-Info.plist`, `Signal.entitlements`,
`Signal-AppStore.entitlements`, `PrivacyInfo.xcprivacy`, `Signal-Prefix.pch`). Intent of
individual assets beyond their names — undetermined, no evidence in source.

## Tests

`Signal/test/` mirrors the app with unit tests, including `AppDelegateTest.swift`, which
exercises `AppLifecycleManager.applicationShortcutItems(isRegistered:)` for the quick-compose
shortcut (`Signal/test/AppDelegateTest.swift:12-24`) **[High]** — the same shortcut built at
`AppLifecycleManager.swift` (`applicationShortcutItems`, quick-compose type) **[High]**.

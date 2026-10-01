# Repository Inventory

This is the foundational documentation artifact for the Signal iOS repository. Every
other document in `docs/` depends on the classification and module boundary map
established here.

All line counts in this document were produced with tooling (`find` + `wc -l`) against
the working tree as checked out. Numbers are exact for the current checkout. Note that
two git submodules are **not checked out** in this tree (see
[Vendored dependencies](#vendored-dependencies)); their contents are therefore excluded
from all counts.

---

## 1. Documentation layout convention

The `docs/` directory **mirrors the module structure** of the repository. Each
first-party module gets its own documentation file (or subdirectory, for larger
modules), named after the module, so that a reader can navigate docs the same way they
navigate source:

```
docs/
├── INVENTORY.md              ← this file (foundational; everything depends on it)
├── Signal.md                 ← the main app module
├── SignalServiceKit.md       ← the core service/data layer
├── SignalUI.md               ← shared UI layer
├── SignalNSE.md              ← Notification Service Extension
└── SignalShareExtension.md   ← Share Extension
```

Conventions:

- **One doc per module.** The five first-party modules below each map to one top-level
  doc. A module whose doc grows too large may expand into a subdirectory
  (`docs/SignalServiceKit/…`) whose internal structure mirrors the module's own
  top-level subdirectories (the "directory breakdown" tables in
  [§6](#6-first-party-directory-breakdown-by-module)).
- **Module boundary map is authoritative.** The boundary map in
  [§5](#5-module-boundary-map) is the single source of truth for which paths belong to
  which module. Per-module docs mirror these boundaries exactly and must not
  reclassify files.
- **Cite path + line when quoting config.** Any doc that quotes a configuration value
  cites it as `path:line` so claims are traceable.

---

## 2. Classification categories

Every file/directory in the repository is classified into exactly one of the following
categories. The rules are listed in priority order (a file matching multiple rules is
assigned to the first matching category).

| # | Category | Matching rule |
|---|----------|---------------|
| 1 | **Vendored dependencies** | `Pods/`, `ThirdParty/`, and any path registered as a git submodule in `.gitmodules` |
| 2 | **Generated code** | `*+SDS.swift` (SDSCodeGenerator output) and protobuf outputs under `SignalServiceKit/Protos/` |
| 3 | **Tests** | `*/tests/**`, `*/test/**`, `*Tests.swift`, `*Test.swift`, `SignalServiceKit/Mocks/**`, `SignalServiceKit/TestUtils/**` |
| 4 | **Assets / localization** | `*.xcassets/**`, `Signal/translations/**`, `fastlane/metadata/**` |
| 5 | **Build / CI** | `Config/`, `Scripts/`, `ci_scripts/`, `fastlane/` (minus metadata), `.github/`, `*.xcodeproj`, `*.xcworkspace`, `Podfile`, `Makefile` |
| 6 | **First-party code** | Everything else under the five module roots (`Signal/`, `SignalServiceKit/`, `SignalUI/`, `SignalNSE/`, `SignalShareExtension/`) |
| 7 | **Documentation / Meta** | Catch-all for repository-level documentation, legal, and meta files that belong to no module and are neither build/CI config nor assets: the root docs/legal files (`README.md`, `BUILDING.md`, `CONTRIBUTING.md`, `MAINTAINING.md`, `SECURITY.md`, `LICENSE`) and the `docs/` directory itself |

Priority matters: e.g. `SignalServiceKit/Protos/` Swift files are **generated**, not
first-party; `SignalServiceKit/tests/MessageBackup/Signal-Message-Backup-Tests` is a
**vendored** submodule, not tests.

---

## 3. Category summary (line counts)

Line counts below count `.swift` lines for code categories (the overwhelming majority of
first-party and generated content) and file counts for asset/config categories where
"lines" are not meaningful.

| Category | Metric | Count |
|----------|--------|-------|
| First-party code (Swift, all modules, excl. tests & generated) | lines | **629,457** |
| First-party code (Objective-C, all modules, `.m` + `.h`) | lines (75 files) | **10,116** |
| Generated code — `*+SDS.swift` | lines (24 files) | **8,617** |
| Generated code — protobuf output under `SignalServiceKit/Protos/` | lines (22 `.swift` files) | **68,208** |
| Tests (Swift, all modules + Mocks + TestUtils) | lines | **77,075** |
| Assets — `*.xcassets` | files (4 catalogs) | **1,233** |
| Localization — `Signal/translations/` | files (47 `.lproj` dirs) | **142** |
| Localization — `fastlane/metadata/` | files (43 locales) | **86** |
| Build / CI — `Scripts/` | files | **56** |
| Build / CI — `.github/` | files | **12** |
| Build / CI — `ci_scripts/` | files | **6** |
| Vendored — `Pods/` submodule | — | **not checked out** |
| Vendored — `ThirdParty/` | files | **2** |
| Documentation / Meta (root docs/legal + `docs/`) | files | **7** |

Derivation of the first-party Swift total:

```
total Swift lines across 5 modules      =  783,357
  Signal                 303,287
  SignalServiceKit       399,191
  SignalUI                78,234
  SignalNSE                  795
  SignalShareExtension     1,850
minus generated (+SDS)                   =   -8,617
minus generated (Protos/*.swift)         =  -68,208
minus tests (Swift, all modules)         =  -77,075   (Signal 19,487; SSK 55,915; SignalUI 1,673)
                                          ----------
first-party Swift lines                  =  629,457
```

Categories are mutually exclusive (verified: no `+SDS.swift` lives under a test or
`Protos/` path, and no `Protos/*.swift` is a test file), so the subtraction has no
double-counting.

> Note: the repository contains **2,652** `.swift` files totalling **784,957** lines
> across the whole tree; the five-module figure (783,357) differs slightly because a
> small number of `.swift` files live outside module roots (e.g. tooling under
> `Scripts/`). The grand total is recorded here for completeness.

> Non-Swift first-party source: the five module roots also contain **75 Objective-C
> files** (34 `.m` + 41 `.h`) totalling **10,116 lines**. These are concentrated almost
> entirely in `SignalServiceKit` (72 files, 10,044 lines) with a small remainder in
> `SignalUI` (3 files, 72 lines); `Signal`, `SignalNSE`, and `SignalShareExtension`
> contain no `.m`/`.h` files. Derived with:
> ```sh
> find Signal SignalServiceKit SignalUI SignalNSE SignalShareExtension \
>   \( -name '*.m' -o -name '*.h' \) -type f | wc -l            # 75
> find Signal SignalServiceKit SignalUI SignalNSE SignalShareExtension \
>   \( -name '*.m' -o -name '*.h' \) -type f -exec cat {} + | wc -l   # 10,116
> ```

---

## 4. Per-category inventory

### Vendored dependencies

Declared submodules — `.gitmodules:1-6`:

```
# .gitmodules:1-3
[submodule "Pods"]
	path = Pods
	url = https://github.com/signalapp/Signal-Pods.git
# .gitmodules:4-6
[submodule "SignalServiceKit/tests/MessageBackup/Signal-Message-Backup-Tests"]
	path = SignalServiceKit/tests/MessageBackup/Signal-Message-Backup-Tests
	url = https://github.com/signalapp/Signal-Message-Backup-Tests
```

- **`Pods/`** — CocoaPods dependencies, pulled from the `Signal-Pods` submodule.
  **Empty in this checkout** (submodule not initialized). The dependency set is pinned
  in `Podfile.lock`. The lockfile declares **17 top-level dependencies** (the
  `DEPENDENCIES:` section) which resolve to **27 top-level `PODS:` entries** (including
  subspecs) and **17 resolved pods** recorded under `SPEC CHECKSUMS:`. Examples include
  `GRDB.swift/SQLCipher (5.26.0)` (`Podfile.lock:7`), `LibSignalClient (0.103.1)`
  (`Podfile.lock:12`), `SignalRingRTC (2.72.0)` (`Podfile.lock:38`), `SwiftProtobuf
  (1.38.1)` (`Podfile.lock:46`). Derived with:
  ```sh
  # 27 top-level PODS entries (lines beginning "  - " in the PODS: block)
  awk '/^PODS:/{f=1;next} /^[A-Z]/{f=0} f && /^  - /' Podfile.lock | wc -l   # 27
  # 17 declared DEPENDENCIES
  awk '/^DEPENDENCIES:/{f=1;next} /^$/{f=0} f && /^  - /' Podfile.lock | wc -l   # 17
  # 17 SPEC CHECKSUMS (resolved pods)
  awk '/^SPEC CHECKSUMS:/{f=1;next} /^$/{f=0} f' Podfile.lock | wc -l   # 17
  ```
- **`SignalServiceKit/tests/MessageBackup/Signal-Message-Backup-Tests`** — vendored
  message-backup test vectors submodule. **Empty in this checkout.**
- **`ThirdParty/`** — 2 vendored podspecs consumed locally:
  `ThirdParty/blurhash.podspec` and `ThirdParty/libwebp.podspec.json`. Referenced from
  the Podfile as `blurhash (from './ThirdParty/blurhash.podspec')` (`Podfile.lock:49`).

Submodule setup is driven by the `Makefile`:

```
# Makefile:5-6
dependencies: pod-setup backup-tests-setup fetch-ringrtc
# Makefile:9-13 (pod-setup)
	git submodule update --init --progress ${PODS}
# Makefile:16-18 (backup-tests-setup)
	git submodule update --init --progress ${BACKUP_TESTS}
```

### Generated code

| Source | Files | Lines |
|--------|------:|------:|
| `*+SDS.swift` (SDSCodeGenerator) | 24 | 8,617 |
| `SignalServiceKit/Protos/*.swift` (SwiftProtobuf output from `.proto`) | 22 | 68,208 |

- **SDS files** live beside their models, e.g. under `SignalServiceKit/Calls/Group`,
  `SignalServiceKit/Calls/Individual`, `SignalServiceKit/Messages`,
  `SignalServiceKit/Messages/Interactions`, `SignalServiceKit/Messages/Payments`.
- **`SignalServiceKit/Protos/`** holds 14 `.proto` source files, 22 generated `.swift`
  files, a build `Makefile`, a `.py` generator helper, and 2 `.md` docs. The `.proto`
  files are the source of truth; the `.swift` files are generated output and must not be
  edited by hand.

### Tests

| Module | Test files | Test lines |
|--------|-----------:|-----------:|
| Signal | 70 | 19,487 |
| SignalServiceKit | 290 | 55,915 |
| SignalUI | 10 | 1,673 |
| SignalNSE | 0 | 0 |
| SignalShareExtension | 0 | 0 |
| **Total** | **370** | **77,075** |

The SignalServiceKit figure (290 files, 55,915 lines) reconciles exactly from five
components:

| Component | Files | Lines |
|-----------|------:|------:|
| `SignalServiceKit/tests/` | 238 | 46,288 |
| Scattered `*Tests.swift` / `*Test.swift` outside `tests/`, `Mocks/`, `TestUtils/` | 22 | 5,472 |
| `SignalServiceKit/TestUtils/` | 11 | 2,973 |
| `SignalServiceKit/Mocks/` (top-level) | 17 | 897 |
| Nested `Messages/Attachments/V2/Mocks/` (`MockAttachment.swift`, `MockAttachmentReference.swift`) | 2 | 285 |
| **Total** | **290** | **55,915** |

The 22 scattered test files (5,472 lines) are `*Tests.swift`/`*Test.swift` files that
live beside the production code they exercise (e.g. `Cryptography/CryptographyTests.swift`,
`Util/StringSanitizerTests.swift`) rather than under the `tests/` tree; they are called
out explicitly here so the breakdown reconciles to the 55,915 total. The two nested
`Mocks/` files match the `*/Mocks/*` test rule but sit under `Messages/Attachments/V2/`,
not the top-level `SignalServiceKit/Mocks/` directory, so they are counted separately.
The bulk sits under `SignalServiceKit/tests/` (238 files, 46,288 lines) and
`Signal/test/` (70 files, 19,487 lines).

### Assets / localization

| Location | Files |
|----------|------:|
| `Signal/AppIcon.xcassets` | 26 |
| `Signal/Images.xcassets` | 368 |
| `Signal/NSE-Images.xcassets` | 5 |
| `Signal/Symbols.xcassets` | 834 |
| `Signal/translations/` (47 `.lproj` dirs) | 142 |
| `fastlane/metadata/` (43 App Store locales) | 86 |

Four `*.xcassets` catalogs totalling **1,233** files. Localization is split between
in-app strings (`Signal/translations/`, 47 locales) and App Store listing metadata
(`fastlane/metadata/`, 43 locales such as `ar-SA`, `zh-Hans`, `pt-BR`).

### Build / CI

| Location | Files | Notes |
|----------|------:|-------|
| `Config/` | 4 | `Project.xcconfig`, `Project-Debug.xcconfig`, `Project-Release.xcconfig`, `User.xcconfig.sample` |
| `Scripts/` | 56 | build/codegen/lint tooling |
| `ci_scripts/` | 6 | Xcode Cloud hooks (`ci_post_clone.sh`, `ci_pre_xcodebuild.sh`, `ci_post_xcodebuild.sh`, …) |
| `fastlane/` (excl. `metadata/`) | 3 | `Fastfile`, `Appfile`, `.gitignore` |
| `.github/` | 12 | GitHub workflows/config |
| `Signal.xcodeproj` | 21 | Xcode project |
| `Signal.xcworkspace` | 5 | Xcode workspace |
| `Podfile` / `Podfile.lock` | 2 | CocoaPods manifest + lockfile |
| `Makefile` | 1 | dependency bootstrap (`make dependencies`) |

Loose root-level build/CI files also include `Gemfile`, `Gemfile.lock`,
`.swiftformat`, `.clang-format`, `.ruby-version`, `.xcode-version`, `.gitmodules`,
`.gitignore`, `.gitattributes`.

### Documentation / Meta

Repository-level documentation and legal files that belong to no module and are neither
build/CI config nor assets fall into this catch-all category, which exists so that
**every** file/dir in the tree is classified:

| Path | Kind |
|------|------|
| `README.md` | Project overview / landing doc |
| `BUILDING.md` | Build & dev-environment setup instructions |
| `CONTRIBUTING.md` | Contribution guidelines |
| `MAINTAINING.md` | Maintainer notes |
| `SECURITY.md` | Security disclosure policy |
| `LICENSE` | GNU AGPLv3 license text |
| `docs/` | This documentation tree (including `docs/INVENTORY.md`) |

That is **6 root documentation/legal files + the `docs/` directory** (7 top-level
entries total). The `docs/` directory is self-referential: it is produced by this
documentation effort and classified here rather than under any module.

---

## 5. Module boundary map

This map is **authoritative**. Per-module docs mirror it exactly.

| Module | Root path | Role | Swift files | Swift lines (total, incl. tests/generated) |
|--------|-----------|------|------------:|-------------------------------------------:|
| **Signal** | `Signal/` | The iOS app target: top-level UI, view controllers, app lifecycle, feature flows (registration, calls, conversation view). | 893 | 303,287 |
| **SignalServiceKit** | `SignalServiceKit/` | Core service & data layer: networking, storage (GRDB), messaging protocol, backups, protobufs, crypto. The largest module. | 1,464 | 399,191 |
| **SignalUI** | `SignalUI/` | Shared UI layer reused across targets: views, pickers, attachment/image editing, appearance. | 271 | 78,234 |
| **SignalNSE** | `SignalNSE/` | Notification Service Extension: decrypts/handles push notifications out-of-process. | 5 | 795 |
| **SignalShareExtension** | `SignalShareExtension/` | Share Extension: lets other apps share content into Signal. | 7 | 1,850 |

Dependency direction (coarse): `SignalNSE`, `SignalShareExtension`, and `Signal` all
depend on `SignalUI` and `SignalServiceKit`; `SignalUI` depends on `SignalServiceKit`;
`SignalServiceKit` is the foundation and depends only on vendored pods. The two
extensions are thin (795 + 1,850 lines) and delegate heavily into the lower layers.

---

## 6. First-party directory breakdown by module

Line counts are `.swift` lines per top-level subdirectory. Generated (`+SDS.swift`,
`Protos/`) and test (`tests/`, `test/`, `Mocks/`, `TestUtils/`) subdirectories are
flagged inline rather than removed, so the tables reconcile with the module totals in
[§5](#5-module-boundary-map).

### Signal/ (303,287 Swift lines)

| Subdirectory | Files | Lines | Category |
|--------------|------:|------:|----------|
| `src` | 361 | 124,992 | first-party |
| `ConversationView` | 139 | 57,370 | first-party |
| `Calls` | 85 | 29,376 | first-party |
| `test` | 70 | 19,487 | **tests** |
| `Registration` | 39 | 16,019 | first-party |
| `Emoji` | 12 | 14,804 | first-party |
| `Backups` | 38 | 11,125 | first-party |
| `AppLaunch` | 10 | 4,500 | first-party |
| `DeviceTransfer` | 20 | 4,338 | first-party |
| `Provisioning` | 18 | 3,648 | first-party |
| `Usernames` | 13 | 3,563 | first-party |
| `Megaphones` | 21 | 2,999 | first-party |
| `util` | 14 | 2,823 | first-party |
| `QuickRestore` | 6 | 1,271 | first-party |
| `Notifications` | 6 | 1,152 | first-party |
| `QRCodes` | 7 | 979 | first-party |
| `Debugging` | 3 | 744 | first-party |
| `Contacts` | 2 | 561 | first-party |
| `Sharing` | 3 | 553 | first-party |
| `Storage` | 4 | 527 | first-party |
| `Attachments` | 4 | 522 | first-party |
| `OrphanData` | 1 | 453 | first-party |
| `URLs` | 2 | 351 | first-party |
| `Avatars` | 1 | 227 | first-party |
| `Spam` | 2 | 211 | first-party |
| `Screenshots` | 2 | 175 | first-party |
| `Expiration` | 2 | 99 | first-party |
| `Profiles` | 1 | 95 | first-party |
| `Autofill` | 1 | 66 | first-party |
| `LowDiskSpace` | 1 | 54 | first-party |
| `Accessibility` | 1 | 53 | first-party |
| `ClockSkew` | 1 | 47 | first-party |
| `Axolotl` | 1 | 44 | first-party |
| `Groups` | 1 | 36 | first-party |
| `Preconditions` | 1 | 23 | first-party |
| asset dirs (`AppIcon.xcassets`, `Images.xcassets`, `NSE-Images.xcassets`, `Symbols.xcassets`, `AppIcons`, `AudioFiles`, `Lottie`, `Settings.bundle`, `Sounds`, `translations`) | 0 Swift | 0 | **assets / localization** |

### SignalServiceKit/ (399,191 Swift lines)

| Subdirectory | Files | Lines | Category |
|--------------|------:|------:|----------|
| `Messages` | 295 | 77,749 | first-party (contains some `+SDS.swift`) |
| `Protos` | 22 | 68,208 | **generated** |
| `tests` | 238 | 46,288 | **tests** |
| `Backups` | 122 | 39,775 | first-party |
| `Util` | 138 | 23,233 | first-party |
| `Storage` | 52 | 19,149 | first-party |
| `Groups` | 38 | 12,871 | first-party |
| `Network` | 58 | 11,724 | first-party |
| `Contacts` | 53 | 11,435 | first-party |
| `Calls` | 50 | 8,382 | first-party (contains some `+SDS.swift`) |
| `Subscriptions` | 39 | 7,474 | first-party |
| `Profiles` | 14 | 5,884 | first-party |
| `StorageService` | 4 | 5,610 | first-party |
| `Environment` | 7 | 5,256 | first-party |
| `Jobs` | 21 | 5,098 | first-party |
| `Account` | 32 | 4,044 | first-party |
| `Axolotl` | 23 | 3,829 | first-party |
| `Devices` | 27 | 3,828 | first-party |
| `Payments` | 14 | 3,764 | first-party |
| `Notifications` | 7 | 3,189 | first-party |
| `TestUtils` | 11 | 2,973 | **tests** |
| `Upload` | 10 | 2,592 | first-party |
| `Threads` | 12 | 2,467 | first-party |
| `Cryptography` | 10 | 2,186 | first-party |
| `Usernames` | 16 | 2,112 | first-party |
| `Megaphones` | 8 | 2,037 | first-party |
| `Avatars` | 4 | 2,035 | first-party |
| `SecureValueRecovery` | 15 | 1,652 | first-party |
| `PromiseKit` | 17 | 1,542 | first-party |
| `UISupport` | 10 | 1,464 | first-party |
| `Concurrency` | 16 | 1,043 | first-party |
| `Debugging` | 8 | 1,034 | first-party |
| `ZeroKnowledge` | 5 | 1,020 | first-party |
| `AttachmentBackfill` | 4 | 921 | first-party |
| `Mocks` | 17 | 897 | **tests** |
| `Registration` | 4 | 881 | first-party |
| `Spam` | 6 | 871 | first-party |
| `KeyTransparency` | 2 | 865 | first-party |
| `Search` | 2 | 685 | first-party |
| `PinnedThreadManager` | 5 | 583 | first-party |
| `DisappearingMessages` | 4 | 442 | first-party |
| `Attachments` | 3 | 327 | first-party |
| `Devices`…(smaller dirs: `ChangePhoneNumber` 275, `Stories` 256, `Expiration` 157, `Preconditions` 154, `ClockSkew` 163, `Dates` 127, `LowDiskSpace` 130, `LocalDeviceAuth` 117, `QRCodes` 106, `Stores` 102, `VoiceMessage` 110, `SafetyTips` 75) | — | <300 each | first-party |

### SignalUI/ (78,234 Swift lines)

| Subdirectory | Files | Lines | Category |
|--------------|------:|------:|----------|
| `Views` | 52 | 14,059 | first-party |
| `RecipientPickers` | 28 | 9,084 | first-party |
| `ImageEditor` | 25 | 7,763 | first-party |
| `ViewControllers` | 19 | 6,963 | first-party |
| `Payments` | 10 | 5,543 | first-party |
| `Appearance` | 12 | 3,475 | first-party |
| `AttachmentApproval` | 8 | 3,400 | first-party |
| `Stickers` | 10 | 3,280 | first-party |
| `UIKitExtensions` | 17 | 3,234 | first-party |
| `SafetyNumbers` | 4 | 2,192 | first-party |
| `LinkPreview` | 8 | 2,208 | first-party |
| `ConversationView` | 6 | 1,984 | first-party |
| `Utils` | 10 | 1,903 | first-party |
| `Attachments` | 5 | 1,828 | first-party |
| `ActionSheets` | 4 | 1,819 | first-party |
| `VideoEditor` | 4 | 1,423 | first-party |
| `Stories` | 11 | 1,400 | first-party |
| `AV` | 4 | 975 | first-party |
| `Sending` | 3 | 911 | first-party |
| `ContactSharing` | 5 | 904 | first-party |
| `Search` | 2 | 881 | first-party |
| `AttachmentMultisend` | 2 | 802 | first-party |
| `SwiftUIExtensions` | 8 | 565 | first-party |
| `Usernames` | 2 | 503 | first-party |
| `Wallpapers` | 1 | 300 | first-party |
| `AppBlocking` | 2 | 234 | first-party |
| `Calls` | 3 | 170 | first-party |
| `FormatStyles` | 3 | 169 | first-party |
| `Media` | 1 | 158 | first-party |
| `AppLaunch` | 2 | 104 | first-party |
| `Fonts` | 0 Swift | 0 | assets |

SignalUI tests (10 files, 1,673 lines) live in `*Tests.swift` files within the above
subdirectories rather than a dedicated `tests/` tree.

### SignalNSE/ (795 Swift lines)

Flat module — 5 Swift files, no subdirectories: `NotificationService.swift`,
`NSEEnvironment.swift`, `NSEContext.swift`, `NSECallMessageHandler.swift`,
`NSELogger.swift`. Plus non-Swift target files (`Info.plist`,
`PrivacyInfo.xcprivacy`, `SignalNSE.entitlements`,
`SignalNSE-AppStore.entitlements`).

### SignalShareExtension/ (1,850 Swift lines)

Flat module — 7 Swift files, no subdirectories: `ShareViewController.swift`,
`SharingThreadPickerViewController.swift`, `SharingThreadPickerProgressSheet.swift`,
`SAELoadViewController.swift`, `SAEScreenLockViewController.swift`,
`ShareAppExtensionContext.swift`, `ShareViewDelegate.swift`. Plus non-Swift target
files (`Info.plist`, `PrivacyInfo.xcprivacy`,
`SignalShareExtension-AppStore.entitlements`).

---

## 7. Methodology

Counts were produced with the following tooling against the checkout:

```sh
# Swift files/lines per module
find "$module" -name '*.swift' -type f | wc -l
find "$module" -name '*.swift' -type f -exec cat {} + | wc -l

# Generated SDS code
find . -name '*+SDS.swift' -type f | wc -l
find . -name '*+SDS.swift' -type f -exec cat {} + | wc -l

# Generated protobuf output
find SignalServiceKit/Protos -name '*.swift' -exec cat {} + | wc -l

# Tests (per module)
find "$module" \( -path '*/tests/*' -o -path '*/test/*' \
  -o -name '*Tests.swift' -o -name '*Test.swift' \
  -o -path '*/Mocks/*' -o -path '*/TestUtils/*' \) -name '*.swift' -type f

# Assets / localization / build-CI
find . -name '*.xcassets' -type d
find Signal/translations -type f
find fastlane/metadata -type f
```

Caveats:
- `.swift` is the dominant source language; Objective-C / C and resource files are not
  line-counted here (they are inventoried by file count where relevant).
- The two uninitialized submodules (`Pods/`, `Signal-Message-Backup-Tests`) are
  excluded from counts because they are empty in this checkout.

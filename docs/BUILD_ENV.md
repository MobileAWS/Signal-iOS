# Build & Platform Configuration

This document describes how Signal iOS is built and configured at the project,
target, and platform level. It is derived entirely from build files, Xcode
project/scheme definitions, entitlements, `Info.plist`s, privacy manifests, and
CI/CD scripts in the working tree. It does **not** depend on the per-module
source docs; it complements [`INVENTORY.md`](./INVENTORY.md), which establishes
the module boundary map referenced below.

Every claim cites `file:line` (or `file` for whole-file facts). Line numbers are
for the current checkout.

---

## 1. Xcode project, workspace & schemes

### 1.1 Workspace and project

Signal is built from a CocoaPods workspace that references the app project plus
the generated Pods project:

- `Signal.xcworkspace/contents.xcworkspacedata:1-10` — the workspace references
  two file groups: `group:Signal.xcodeproj` and `group:Pods/Pods.xcodeproj`.
- `Signal.xcodeproj/project.pbxproj` — the single app project (~2.4 MB) that
  defines all first-party targets.
- `BUILDING.md:38` / `BUILDING.md:41` — the canonical entry point is to open
  `Signal.xcworkspace` (`open Signal.xcworkspace`).

Workspace shared data:

- `Signal.xcworkspace/xcshareddata/WorkspaceSettings.xcsettings`
- `Signal.xcworkspace/xcshareddata/IDEWorkspaceChecks.plist`
- `Signal.xcworkspace/xcshareddata/IDETemplateMacros.plist`
- `Signal.xcworkspace/xcshareddata/Signal.xcscmblueprint`

### 1.2 Targets

The five first-party targets (matching the module boundary map in
`INVENTORY.md`) are:

| Target | Product | Role |
| --- | --- | --- |
| `Signal` | `Signal.app` | Main application |
| `SignalNSE` | `SignalNSE.appex` | Notification Service Extension |
| `SignalShareExtension` | `SignalShareExtension.appex` | Share Extension (app extension) |
| `SignalServiceKit` | `SignalServiceKit.framework` | Core service/data layer |
| `SignalUI` | `SignalUI` framework | Shared UI layer |

Each also has a test target: `SignalTests`, `SignalUITests`,
`SignalServiceKitTests` (declared in the Podfile at
`Podfile:59`, `Podfile:73`, `Podfile:81`). The product/blueprint names come from
the shared schemes (e.g. `Signal.app` at
`Signal.xcodeproj/xcshareddata/xcschemes/Signal.xcscheme:18`,
`SignalNSE.appex` at `.../SignalNSE.xcscheme:19`,
`SignalShareExtension.appex` at `.../SignalShareExtension.xcscheme:19`,
`SignalServiceKit.framework` at `.../SignalServiceKit.xcscheme:19`). The
`SignalShareExtension.appex` product is declared as a PBXFileReference with
`explicitFileType = "wrapper.app-extension"` at `project.pbxproj:4872`.

Bundle identifiers (all derived from `SIGNAL_BUNDLEID_PREFIX`):

- Main app: `$(SIGNAL_BUNDLEID_PREFIX).signal` — `project.pbxproj:22010`
- NSE: `$(SIGNAL_BUNDLEID_PREFIX).signal.SignalNSE` — `project.pbxproj:21149`
- Share Extension: `$(SIGNAL_BUNDLEID_PREFIX).signal.shareextension` —
  `project.pbxproj:21806`
- SignalUI: `org.signal.SignalUI` — `project.pbxproj:21358`
- `SIGNAL_BUNDLEID_PREFIX = org.whispersystems` and
  `SIGNAL_MERCHANTID = org.signalfoundation` are set project-wide at
  `project.pbxproj:21951-21952`.

### 1.3 Schemes

Shared schemes live in
`Signal.xcodeproj/xcshareddata/xcschemes/`:

- `Signal.xcscheme` — primary scheme. Tests run against `SignalTests`,
  `SignalUITests`, `SignalServiceKitTests`
  (`Signal.xcscheme:65-99`); code coverage is enabled and limited to
  `Signal.app` + `SignalServiceKit.framework`
  (`Signal.xcscheme:48-63`). `MessageSendLogTests/testPlaintextMismatchFails()`
  is skipped (`Signal.xcscheme:94-98`). Archive uses the
  `App Store Release` configuration (`Signal.xcscheme:192-195`).
- `Signal-Staging.xcscheme` — staging variant.
- `SignalNSE.xcscheme` — builds `SignalNSE.appex` + `Signal.app`, marked
  `wasCreatedForAppExtension = "YES"` (`SignalNSE.xcscheme:4`).
- `SignalServiceKit.xcscheme` — framework scheme with autocreated test plan
  (`SignalServiceKit.xcscheme:26-32`, `shouldAutocreateTestPlan = "YES"` at
  `SignalServiceKit.xcscheme:31`).
- `SignalShareExtension.xcscheme` — marked
  `wasCreatedForAppExtension = "YES"` (`SignalShareExtension.xcscheme:4`).

The `Signal` launch action defines debugging env vars and optional feature-flag
launch arguments (`Signal.xcscheme:102-171`), including `USE_STAGING`,
`UBSAN_OPTIONS=suppressions=SignalUBSan.supp`,
`TSAN_OPTIONS=suppressions=SignalTSan.supp` (sanitizer suppression files at
`Signal/SignalUBSan.supp`, `Signal/SignalTSan.supp`).

### 1.4 Build configurations

Four build configurations are defined (named in `project.pbxproj`):

- `Debug` (e.g. `project.pbxproj:21155`)
- `Testable Release` (`project.pbxproj:21323`)
- `Profiling` (`project.pbxproj:21267`)
- `App Store Release` (`project.pbxproj:21211`) — used for Archive.

The Podfile mirrors these in `configure_testable_build`, which enables
`ONLY_ACTIVE_ARCH` and `ENABLE_TESTABILITY` for `Testable Release`, `Debug`, and
`Profiling` (`Podfile:144-151`).

### 1.5 Shared build settings (`Config/*.xcconfig`)

The `Config/` directory holds the xcconfig layering, included via the project:

- `Config/Project.xcconfig` — treats warnings as errors across C/Swift/Metal:
  `GCC_TREAT_WARNINGS_AS_ERRORS = YES`,
  `OTHER_SWIFT_FLAGS = $(inherited) -warnings-as-errors`
  (`Config/Project.xcconfig:6-9`). It `#include?`s `User.xcconfig` for
  per-developer overrides (`Config/Project.xcconfig:12`).
- `Config/Project-Debug.xcconfig` — includes `Project.xcconfig` and
  `User-Debug.xcconfig` (`Config/Project-Debug.xcconfig:6`,
  `Config/Project-Debug.xcconfig:9`).
- `Config/Project-Release.xcconfig` — includes `Project.xcconfig` and
  `User-Release.xcconfig` (`Config/Project-Release.xcconfig:6`,
  `Config/Project-Release.xcconfig:9`).
- `Config/User.xcconfig.sample` — template that disables
  warnings-as-errors (`OTHER_SWIFT_FLAGS = $(inherited) -no-warnings-as-errors`,
  `Config/User.xcconfig.sample:6`) and documents the optional file-based camera
  input path (`Config/User.xcconfig.sample:8-11`).

Other notable project-wide build settings (`project.pbxproj`, Testable Release
block around lines 21943-21976):

- `SWIFT_VERSION = 5.0` (`project.pbxproj:21965`)
- `SWIFT_COMPILATION_MODE = wholemodule` (`project.pbxproj:21953`)
- `SWIFT_OPTIMIZATION_LEVEL = -O` (`project.pbxproj:21955`)
- A large set of `SWIFT_UPCOMING_FEATURE_*` flags enabled
  (`project.pbxproj:21956-21964`), e.g.
  `SWIFT_UPCOMING_FEATURE_DEPRECATE_APPLICATION_MAIN`,
  `SWIFT_UPCOMING_FEATURE_INTERNAL_IMPORTS_BY_DEFAULT`.
- `WARNING_CFLAGS` promotes several Objective-C warnings to errors
  (`project.pbxproj:21967-21976`) — this list is intentionally kept in sync with
  the Podfile's `configure_warning_flags` (`Podfile:130-141`).
- `SDKROOT = iphoneos` (`project.pbxproj:21950`).

---

## 2. iOS deployment target & notable deprecated APIs

### 2.1 Minimum deployment target

- `IPHONEOS_DEPLOYMENT_TARGET = 15.0` project-wide
  (`project.pbxproj:21943`, also `project.pbxproj:22207`).
- `Podfile:1` — `platform :ios, '15.0'`.
- The Podfile's `promote_minimum_supported_version` bumps every pod whose
  deployment target is lower than the project min up to 15.0, suppressing
  warnings (`Podfile:157-171`).

### 2.2 Notable deprecated / legacy API surface

Derived from build configuration (not from source docs):

- `UIRequiredDeviceCapabilities = [armv7]` in the app `Info.plist`
  (`Signal/Signal-Info.plist`, `UIRequiredDeviceCapabilities` key) is a
  legacy declaration; the build itself **excludes `armv7`** via the Podfile's
  `disable_armv7`, which sets `EXCLUDED_ARCHS = armv7`
  (`Podfile:181-187`). Modern builds are arm64-only.
- `ENABLE_BITCODE = NO` is forced for all pods (`Podfile:173-179`); Bitcode is
  deprecated by Apple.
- The AppStore entitlement `com.apple.developer.pushkit.unrestricted-voip`
  (`Signal/Signal-AppStore.entitlements`) reflects the legacy PushKit VoIP
  path; the development entitlements omit it.
- `strip_valid_archs` (`Podfile:189-196`) rewrites generated pod xcconfigs to
  remove hard-coded `VALID_ARCHS` entries that CocoaPods still emits.
- `update_frameworks_script` (`Podfile:202-213`) patches
  `Pods-Signal-frameworks.sh` to work around a CocoaPods XCFramework→framework
  symlink bug (CocoaPods issue #7587).

---

## 3. Entitlements, app groups & permissions

### 3.1 App groups (shared storage)

All targets share the same App Group identifiers:

- `group.$(SIGNAL_BUNDLEID_PREFIX).signal.group` and `.signal.group.staging`
  declared in every target's `*.entitlements` under
  `com.apple.security.application-groups`:
  - `Signal/Signal.entitlements`
  - `Signal/Signal-AppStore.entitlements`
  - `SignalNSE/SignalNSE.entitlements`, `SignalNSE/SignalNSE-AppStore.entitlements`
  - `SignalShareExtension/SignalShareExtension.entitlements`,
    `SignalShareExtension/SignalShareExtension-AppStore.entitlements`

**Code paths that rely on the app group:**

- `SignalServiceKit/Environment/TSConstants.swift:195` —
  `applicationGroup = "group." + Bundle.main.bundleIdPrefix + ".signal.group"`
  (production); staging at `TSConstants.swift:247`
  (`.signal.group.staging`). Exposed via
  `TSConstants.applicationGroup` (`TSConstants.swift:72`, protocol at `:127`).
- `Signal/AppLaunch/MainAppContext.swift:171` — the shared container path is
  `FileManager.default.containerURL(forSecurityApplicationGroupIdentifier: TSConstants.applicationGroup)`.
- `Signal/AppLaunch/MainAppContext.swift:176` — shared defaults via
  `UserDefaults(suiteName: TSConstants.applicationGroup)`.

### 3.2 Keychain access group

- `keychain-access-groups = [$(AppIdentifierPrefix)$(SIGNAL_BUNDLEID_PREFIX).signal]`
  in all target entitlements (e.g. `Signal/Signal.entitlements`).

### 3.3 Data protection & hardened process

Across all target entitlements:

- `com.apple.developer.default-data-protection = NSFileProtectionComplete`
  (e.g. `Signal/Signal.entitlements`).
- A full `com.apple.security.hardened-process*` family (checked-allocations,
  dyld-ro, hardened-heap, enhanced-security version string, platform
  restrictions) is enabled in every target's entitlements.

### 3.4 App-specific entitlements (main app)

From `Signal/Signal.entitlements` / `Signal/Signal-AppStore.entitlements`:

- `aps-environment = development` (push notifications).
- `com.apple.developer.associated-domains` — applinks for `signal.art`,
  `signal.tube`, `signal.group`, `signal.me`, `signaldonations.org`,
  `signal.link`, plus `webcredentials:signal.org`.
- `com.apple.developer.in-app-payments = [merchant.$(SIGNAL_MERCHANTID)]`
  (Apple Pay / donations).
- `com.apple.developer.usernotifications.communication = true`
  (Communication Notifications).
- `com.apple.developer.networking.carrier-constrained.*` (carrier-constrained
  networking, category `messaging-8001`).
- `com.apple.developer.wifi-aware = [Publish, Subscribe]` and
  `com.apple.developer.ubiquity-kvstore-identifier` (iCloud KVS).
- AppStore-only: `com.apple.developer.pushkit.unrestricted-voip = true`.

NSE-only entitlement:
`com.apple.developer.usernotifications.filtering = true`
(`SignalNSE/SignalNSE-AppStore.entitlements`).

### 3.5 Usage-description / permission strings (`Signal/Signal-Info.plist`)

The main app `Info.plist` declares usage descriptions for camera, contacts,
Face ID, local network, location (when-in-use), microphone, photo-library (add
and read), and Apple Music (`NS*UsageDescription` keys). Other notable keys:

- `UIBackgroundModes = [audio, fetch, processing, remote-notification, voip]`.
- `BGTaskSchedulerPermittedIdentifiers` — five background task identifiers
  (`LocalFileBackupBGProcessingTaskRunner`, `AttachmentValidationBackfillMigrator`,
  `BackupBGProcessingTaskRunner`, `MessageFetchBGRefreshTask`,
  `LazyDatabaseMigratorTask`).
- `CFBundleURLSchemes = [sgnl]` and `LSApplicationQueriesSchemes = [maps,
  comgooglemaps, mailto]`.
- `NSUserActivityTypes = [INSendMessageIntent, INStartCallIntent]`.
- `NSBonjourServices = [_sgnl-new-device._tcp]` and
  `WiFiAwareServices = {_sgnl-transfer._tcp}` for device transfer.
- `NSAppTransportSecurity` grants an insecure-HTTP exception for `signal.org`
  subdomains.
- `ITSAppUsesNonExemptEncryption = false`;
  `LSApplicationCategoryType = public.app-category.social-networking`.
- `CFBundleShortVersionString = 8.31`, `CFBundleVersion = 0` (both the app and
  the extensions carry `8.31` — see `SignalNSE/Info.plist`,
  `SignalShareExtension/Info.plist`).
- The `BuildDetails` dict is populated at build time by
  `Scripts/update_plist_info.sh` (added at `Scripts/update_plist_info.sh:18`,
  with the `App Store Release` detail fields at
  `Scripts/update_plist_info.sh:21-30`).

Extension `Info.plist` extension points:

- NSE: `NSExtensionPointIdentifier = com.apple.usernotifications.service`,
  principal class `$(PRODUCT_MODULE_NAME).NotificationService`
  (`SignalNSE/Info.plist`).
- Share Extension: `NSExtensionPointIdentifier = com.apple.share-services`,
  principal class `$(PRODUCT_MODULE_NAME).ShareViewController`, with an
  `NSExtensionActivationRule` SUBQUERY accepting `public.data`, `public.url`,
  and `com.apple.pkpass` attachments; supports `INSendMessageIntent`
  (`SignalShareExtension/Info.plist`).

### 3.6 Privacy manifests (`PrivacyInfo.xcprivacy`)

Each target ships a privacy manifest; all set `NSPrivacyTracking = false` with
no tracking domains:

- `Signal/PrivacyInfo.xcprivacy` — accessed-API reasons for FileTimestamp
  (`C617.1`, `3B52.1`), DiskSpace (`E174.1`), UserDefaults (`CA92.1`, `1C8F.1`);
  collects `NSPrivacyCollectedDataTypePhoneNumber` for app functionality
  (not linked, not for tracking).
- `SignalNSE/PrivacyInfo.xcprivacy` — FileTimestamp (`C617.1`) + UserDefaults;
  no collected data types.
- `SignalShareExtension/PrivacyInfo.xcprivacy` — FileTimestamp (`C617.1`,
  `3B52.1`) + UserDefaults; no collected data types.

---

## 4. Dependency management

### 4.1 CocoaPods (`Podfile` / `Podfile.lock`)

- `use_frameworks!` (`Podfile:3`) with source `https://cdn.cocoapods.org/`
  (`Podfile:9`). Pinned CocoaPods version: `COCOAPODS: 1.15.2`
  (`Podfile.lock`, final line).
- Shared "ui_pods" group (`Podfile:44-52`): `BonMot`, `PureLayout`,
  `lottie-ios`, plus MobileCoin stack (`LibMobileCoin/CoreHTTP`,
  `MobileCoin/CoreHTTP`).
- Signal-maintained forks / prebuilt binaries:
  - `LibSignalClient` from `signalapp/libsignal` tag `v0.103.1`
    (`Podfile:15`), with a prebuild checksum env
    `LIBSIGNAL_FFI_PREBUILD_CHECKSUM` (`Podfile:14`).
  - `SignalRingRTC` from `signalapp/ringrtc` tag `v2.72.0` (`Podfile:20`), with
    `RINGRTC_PREBUILD_CHECKSUM` (`Podfile:18`).
  - `SQLCipher` from `signalapp/sqlcipher` tag `v4.6.1-f_barrierfsync`
    (`Podfile:26`).
  - `libPhoneNumber-iOS` from `signalapp/libPhoneNumber-iOS` branch
    `signal-master` (`Podfile:33`).
- `GRDB.swift/SQLCipher` (`Podfile:23`), `SDWebImage` (`Podfile:36`),
  `SDWebImageWebPCoder` (`Podfile:37`), `libwebp` (`Podfile:38`).
- `SwiftProtobuf` pinned to `1.38.1` (`Podfile:12`); `CocoaLumberjack` only in
  `SignalServiceKit` (`Podfile:79`).

Resolved versions / integrity are locked in `Podfile.lock` (`PODS:`,
`SPEC CHECKSUMS:`, and `CHECKOUT OPTIONS:` sections), e.g. GRDB.swift 5.26.0,
LibSignalClient 0.103.1, SignalRingRTC 2.72.0, SQLCipher 4.6.1, SwiftProtobuf
1.38.1. `PODFILE CHECKSUM: b8de764822afba0fea924066e4313b8885b3e87d`.

**`post_install` hooks** (`Podfile:89-103`) run a pipeline that: strips installed
products, enables PureLayout for app extensions
(`PURELAYOUT_APP_EXTENSIONS=1`, `Podfile:114-123`, with the flag set at
`Podfile:119`), syncs warning flags, configures testability, promotes min iOS
version, disables Bitcode, disables armv7, strips `VALID_ARCHS`, patches the
frameworks script, suppresses warnings on non-development pods, fixes the RingRTC
symlink, fetches RingRTC, and copies third-party acknowledgements into
`Signal/Settings.bundle/Acknowledgements.plist` (`copy_acknowledgements`,
`Podfile:250-375`).

### 4.2 Git submodules (`.gitmodules`)

Two submodules (`.gitmodules`):

- `Pods` → `https://github.com/signalapp/Signal-Pods.git` — the prebuilt
  CocoaPods checkout (`.gitmodules:1-3`).
- `SignalServiceKit/tests/MessageBackup/Signal-Message-Backup-Tests` →
  `https://github.com/signalapp/Signal-Message-Backup-Tests` (`.gitmodules:4-6`).

Per `INVENTORY.md`, both submodules are **not checked out** in this tree, so
their contents are excluded from inventory counts. `BUILDING.md:10`
emphasizes cloning `--recurse-submodules`.

### 4.3 Vendored podspecs (`ThirdParty/`)

- `ThirdParty/blurhash.podspec` — pins `woltapp/blurhash` to commit
  `0a1f97898d…`, MIT (`ThirdParty/blurhash.podspec`). Referenced from
  `Podfile:11`.
- `ThirdParty/libwebp.podspec.json` — a vendored libwebp spec (declares
  `v1.3.2`; the resolved pod in `Podfile.lock` is `libwebp 1.5.0` from trunk).

### 4.4 Ruby toolchain (`Gemfile` / `Gemfile.lock`)

- `Gemfile` requires `cocoapods`, `fastlane`, `xcode-install`.
- `.ruby-version` pins the Ruby used by CI (`ruby/setup-ruby` reads it,
  `.github/workflows/main.yml` Setup Ruby step).

### 4.5 Dependency bootstrap (`Makefile`)

`make dependencies` (`Makefile:5-6`) runs three phony targets:

- `pod-setup` — cleans/resets the `Pods` submodule, runs
  `Scripts/setup_private_pods`, and `git submodule update --init` for `Pods`
  (`Makefile:8-13`).
- `backup-tests-setup` — inits the Message Backup tests submodule
  (`Makefile:15-17`).
- `fetch-ringrtc` — runs
  `Pods/SignalRingRTC/bin/set-up-for-cocoapods` (`Makefile:19-21`).

---

## 5. CI / CD

### 5.1 Toolchain pinning

- `.xcode-version` → `Xcode 26.6` (the expected Xcode version that
  `Scripts/check_xcode_version.py` compares against).
- `Scripts/check_xcode_version.py` compares `xcodebuild -version` against
  `.xcode-version`, with a `--relaxed` mode that ignores the patch version
  (`Scripts/check_xcode_version.py:22-39`; the `--relaxed` flag is defined at
  `Scripts/check_xcode_version.py:24-26` and applied at
  `Scripts/check_xcode_version.py:31-33`).
- GitHub Actions pin `DEVELOPER_DIR = /Applications/Xcode_26.6.app`
  (`.github/workflows/main.yml:21`; the matrix build re-exports it via
  `.github/workflows/main.yml:39`), consistent with the `.xcode-version` value
  above.

### 5.2 GitHub Actions (`.github/workflows/`)

- `main.yml` — **CI** on PRs and pushes to `main` / `release/*`. The
  `build_and_test` job runs on `macos-26-xlarge`, sets the Xcode version,
  downloads the Metal toolchain and iOS runtime, runs the composite
  `clone-everything` action, sets up Ruby via `.ruby-version`, then runs
  `Scripts/build-and-test.sh`. On failure it uploads
  `~/Library/Logs/Signal-CI`; it always uploads the normalized DB schema from
  `~/Library/Signal-iOS-Schema` (`main.yml:24-82`). A second job,
  `check_autogenstrings`, runs `Scripts/translation/auto-genstrings` and fails
  if `git diff` is non-empty (`main.yml:84-100`).
- `precommit.yml` — builds pinned SwiftFormat (`v0.60.1`, ref
  `c8e50ff…`) and clang-format, then runs `Scripts/precommit.py --ref
  origin/<base>` on changed files, failing on any diff
  (`precommit.yml:60-93`).
- `protobuf-check.yml` — builds pinned `apple/swift-protobuf` (`v1.38.1`),
  regenerates protos (`SignalServiceKit/Protos` `make`), and fails on any diff
  (`protobuf-check.yml:46-90`).
- `translation-check.yml`, `translation-tool.yml`,
  `translation-validator.yml` — localization checks.
- `stale.yml` — stale issue/PR management.

### 5.3 Composite action (`.github/actions/`)

- `clone-everything/action.yml` — checks out the repo with
  `submodules: recursive`; if the public submodule checkout fails (private Pods
  commit), it falls back to checking out `signalapp/Signal-Pods-Private` and
  `signalapp/Signal-Message-Backup-Tests` at the pinned refs using an access
  token, then runs `make fetch-ringrtc`
  (`.github/actions/clone-everything/action.yml`).

### 5.4 Local build/test script (`Scripts/`)

- `Scripts/build-and-test.sh` — selects the latest iOS Simulator runtime via
  `xcrun simctl`, then runs `xcodebuild -workspace Signal.xcworkspace -scheme
  Signal ... build test` with test-timeout settings, pipes output through
  `xcbeautify`, and dumps results to `~/Library/Logs/Signal-CI`. It then runs
  `xcresulttool` + `Scripts/parse-xcresult.py` and exits with the xcodebuild
  result code (`Scripts/build-and-test.sh`).
- Other build-relevant scripts: `Scripts/update_plist_info.sh` (injects
  `BuildDetails` for App Store Release), `Scripts/precommit.py`,
  `Scripts/sort-Xcode-project-file`, `Scripts/bump_build_tag.py`,
  `Scripts/check_xcode_version.py`, `Scripts/upload_metadata`, and the
  `Scripts/feature_flags_*.py` family.

### 5.5 Xcode Cloud (`ci_scripts/`, `Signal.xcodeproj/.../xcodecloud/`)

Xcode Cloud looks for `ci_scripts/` at the repo root:

- `ci_scripts/ci_post_clone.sh` — sends a "started" build notification, skips
  the version check for the `Nightly (Xcode 26)` workflow (writing a
  `TestFlight/WhatToTest.en-US.txt`), otherwise runs
  `Scripts/check_xcode_version.py`, then `make dependencies`
  (`ci_scripts/ci_post_clone.sh`).
- `ci_scripts/ci_pre_xcodebuild.sh` — minimal (`set -eux`).
- `ci_scripts/ci_post_xcodebuild.sh` — sends `finished`/`failed` notification
  based on `CI_XCODEBUILD_EXIT_CODE` (`ci_scripts/ci_post_xcodebuild.sh`).
- `ci_scripts/send_build_notification.py` — posts a build status message (reads
  marketing version from `Signal/Signal-Info.plist`, Xcode version from
  `xcodebuild -version`; only runs for `Nightly`/`Release` workflows)
  (`ci_scripts/send_build_notification.py`).
- `ci_scripts/feature_flag_level.txt`, `ci_scripts/tag_template.txt` — build
  inputs.
- `Signal.xcodeproj/xcshareddata/xcodecloud/manifest.json` — Xcode Cloud target
  manifest listing the `Signal` target.

### 5.6 Fastlane (`fastlane/`)

- `fastlane/Fastfile` — `default_platform(:ios)`; otherwise minimal/templated.
- `fastlane/Appfile` — commented template for `app_identifier` / `apple_id`.
- `fastlane/metadata/` — App Store metadata per locale (consumed by
  `Scripts/upload_metadata`).

---

## 6. Performance baselines

`Signal.xcodeproj/xcshareddata/xcbaselines/` holds XCTest performance baselines
(`.xcbaseline` bundles keyed by target UUIDs
`D221A0A9169C9E5F00537ABF` and `4C10B17F23176D250099396B`), used by the test
targets to detect performance regressions.

---

## Source file index

| Area | Files |
| --- | --- |
| Project/workspace | `Signal.xcodeproj/project.pbxproj`, `Signal.xcworkspace/contents.xcworkspacedata`, `Signal.xcodeproj/xcshareddata/xcschemes/*.xcscheme` |
| Build config | `Config/Project.xcconfig`, `Config/Project-Debug.xcconfig`, `Config/Project-Release.xcconfig`, `Config/User.xcconfig.sample` |
| Entitlements | `Signal/Signal.entitlements`, `Signal/Signal-AppStore.entitlements`, `SignalNSE/SignalNSE*.entitlements`, `SignalShareExtension/SignalShareExtension*.entitlements` |
| Info.plist | `Signal/Signal-Info.plist`, `SignalNSE/Info.plist`, `SignalShareExtension/Info.plist` |
| Privacy | `Signal/PrivacyInfo.xcprivacy`, `SignalNSE/PrivacyInfo.xcprivacy`, `SignalShareExtension/PrivacyInfo.xcprivacy` |
| Dependencies | `Podfile`, `Podfile.lock`, `.gitmodules`, `ThirdParty/blurhash.podspec`, `ThirdParty/libwebp.podspec.json`, `Gemfile`, `Gemfile.lock`, `Makefile` |
| App-group code paths | `SignalServiceKit/Environment/TSConstants.swift`, `Signal/AppLaunch/MainAppContext.swift` |
| CI/CD | `.github/workflows/*.yml`, `.github/actions/clone-everything/action.yml`, `ci_scripts/*`, `fastlane/*`, `Scripts/build-and-test.sh`, `Scripts/check_xcode_version.py`, `Scripts/update_plist_info.sh`, `.xcode-version`, `.ruby-version` |

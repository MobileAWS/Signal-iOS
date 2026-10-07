# `Signal/URLs/` — Deep Link & External URL Handling

Documentation of the **first-party `Signal/URLs/` subsystem** of the main app
target: the component that parses externally-delivered URLs (deep links,
universal links, custom `sgnl://` scheme links) into a known set of actions and
then opens the matching UI.

This set treats `SignalServiceKit/` and `SignalUI/` as **boundaries**: it
documents *how this subsystem uses them*, not their internals. For the app
target overview see [../README.md](../README.md); for the launch state machine
that invokes this subsystem see [../launch-sequence.md](../launch-sequence.md)
and [../scene-window-management.md](../scene-window-management.md).

## Scope

In scope (the entire contents of `Signal/URLs/`):

- `UrlOpener.swift` — URL parser (`UrlOpener.parseUrl`) and opener
  (`UrlOpener.openUrl`) covering every supported deep-link type.
- `SignalDotMePhoneNumberLink.swift` — namespace for `signal.me` phone-number
  links (`{https,sgnl}://signal.me/#p/<e164>`).

## Conventions

- **Citations** are `path:line` (specific declaration) or `path`
  (directory/file). Line numbers reflect the working tree at authoring time and
  may drift; relocate via the symbol name. Every factual claim about behavior
  cites a path.
- **Confidence labels** on claims about *purpose/behavior*:
  - **[High]** — directly observed (file read / declaration located in this
    session).
  - **[Medium]** — inferred from signatures, names, and cross-file convention;
    not every referenced file was read in full.
  - **[Low]** — educated inference from naming alone.
- Any uncited claim is a **defect**.

## One-paragraph orientation

`Signal/URLs/` turns an inbound `URL` into an in-app action in two phases.
First, `UrlOpener.parseUrl` classifies the URL into one of nine
`OpenableUrl` cases (`Signal/URLs/UrlOpener.swift:9`, parsing at
`Signal/URLs/UrlOpener.swift:61`), returning an opaque `ParsedUrl`
(`Signal/URLs/UrlOpener.swift:50`) or `nil` if nothing matches **[High]**.
Second, `UrlOpener.openUrl(_:in:)` runs on the main actor, dismisses any
presented view controller when appropriate, and dispatches each case to the
owning feature's UI — presenting conversations, sticker packs, group-invite
sheets, proxy sheets, device-linking/quick-restore action sheets, iDEAL
donation continuation, or a call-link lobby (`Signal/URLs/UrlOpener.swift:130`
onward) **[High]**. `SignalDotMePhoneNumberLink` is a self-contained helper for
the `signal.me/#p/<e164>` phone-number case, doing a contact-discovery lookup
before opening a chat (`Signal/URLs/SignalDotMePhoneNumberLink.swift:11`)
**[High]**.

## Responsibility

The subsystem is the main app's **single parse-and-dispatch point for external
URLs**. It owns:

- Recognizing which Signal feature a URL belongs to **[High]**
  (`Signal/URLs/UrlOpener.swift:61`).
- Enforcing registration/primary-device preconditions before acting **[High]**
  (`Signal/URLs/UrlOpener.swift:164`, `:199`, `:226`).
- Deciding presentation hygiene — whether to dismiss an existing modal before
  presenting the destination **[High]** (`Signal/URLs/UrlOpener.swift:144`).

It explicitly does **not** own the UI it launches: each case hands off to a
type defined elsewhere (sticker pack VC, group-invite UI, proxy sheet, call
lobby, donation utilities), so the destination behavior lives in those features,
not here **[High]** (`Signal/URLs/UrlOpener.swift:161`–`:293`).

## Key types

### `UrlOpener` (class)
`Signal/URLs/UrlOpener.swift:22`. Holds five service dependencies injected at
construction — `SDSDatabaseStorage`, `DonationSubscriptionManager`,
`PendingIDEALDonationStore`, `ProfileBadgeManager`, `TSAccountManager`
(`Signal/URLs/UrlOpener.swift:23`–`:40`) **[High]**. The dependencies exist only
to support the iDEAL-donation and registration-check paths; parsing itself is
all static and dependency-free **[High]**
(`Signal/URLs/UrlOpener.swift:55`).

- **`Constants.sgnlPrefix = "sgnl"`** (`Signal/URLs/UrlOpener.swift:46`) — the
  custom URL scheme string. Reused outside this subsystem for device-transfer
  URL construction (`Signal/DeviceTransfer/MultiPeerConnectivity/MPCDeviceTransferAdvertiser.swift:94`,
  `Signal/DeviceTransfer/WiFiAware/WADeviceTransferIncomingConnection.swift:59`)
  and by `SignalDotMePhoneNumberLink`'s regex
  (`Signal/URLs/SignalDotMePhoneNumberLink.swift:12`) **[High]**.
- **`ParsedUrl`** (`Signal/URLs/UrlOpener.swift:50`) — an opaque wrapper around
  the private `OpenableUrl`. Because its single field is `fileprivate`, callers
  can hold the result of `parseUrl` but cannot inspect or construct it, keeping
  the case set internal **[High]**.
- **`static parseUrl(_:) -> ParsedUrl?`** (`Signal/URLs/UrlOpener.swift:55`) —
  public entry; delegates to the private `parseOpenableUrl` **[High]**.
- **`openUrl(_:in:)`** (`@MainActor`, `Signal/URLs/UrlOpener.swift:130`) —
  requires the window to have a `rootViewController`, otherwise logs and bails
  via `owsFailDebug` **[High]**.

### `OpenableUrl` (private enum)
`Signal/URLs/UrlOpener.swift:9`. The closed set of recognized actions:
`phoneNumberLink`, `usernameLink`, `stickerPack`, `groupInvite`, `signalProxy`,
`linkDevice`, `completeIDEALDonation`, `callLink`, `quickRestore` **[High]**.
Being `private` means the universe of supported deep links is defined entirely
within this file **[High]**.

### `SignalDotMePhoneNumberLink` (class / namespace)
`Signal/URLs/SignalDotMePhoneNumberLink.swift:11`. Handles
`{https,sgnl}://signal.me/#p/<e164>` links via a precompiled
`NSRegularExpression` (`Signal/URLs/SignalDotMePhoneNumberLink.swift:12`)
**[High]**.

- **`isPossibleUrl(_:) -> Bool`** (`Signal/URLs/SignalDotMePhoneNumberLink.swift:14`)
  — lowercases the absolute string and tests the pattern **[High]**.
- **`openChat(url:fromViewController:)`** (`@MainActor`,
  `Signal/URLs/SignalDotMePhoneNumberLink.swift:18`) — resolves the phone number
  then presents the conversation in `.compose` action via
  `SignalApp.shared.presentConversationForAddress` **[High]**.
- **`open(...)`** (private, `Signal/URLs/SignalDotMePhoneNumberLink.swift:25`) —
  presents a cancelable `ModalActivityIndicatorViewController`, performs a
  contact-discovery lookup through
  `SSKEnvironment.shared.contactDiscoveryManagerRef.lookUp(...)`, and either
  invokes the completion block with the resolved recipient, offers an SMS
  invitation sheet when no recipient is found, or shows an error alert on
  failure **[High]**.

## Parsing flow (`parseOpenableUrl`)

`Signal/URLs/UrlOpener.swift:61`. The parser tries matchers **in order**, so
precedence is positional; the first match wins **[High]**:

1. `SignalDotMePhoneNumberLink.isPossibleUrl` → `.phoneNumberLink` (`:63`).
2. `Usernames.UsernameLink(usernameLinkUrl:)` → `.usernameLink` (`:66`).
3. `StickerPackInfo.isStickerPackShare` + `parseStickerPackShare` →
   `.stickerPack` (`:69`).
4. `parseSgnlAddStickersUrl` (local, `:97`) → `.stickerPack` (`:72`).
5. `PossibleGroupInviteLinkUrl.parseFrom` → `.groupInvite` (`:75`).
6. `SignalProxy.isValidProxyLink` → `.signalProxy` (`:78`).
7. `DeviceProvisioningURL(urlString:)` (via `isSgnlLinkDeviceUrl`, `:124`) →
   `.linkDevice` or `.quickRestore` depending on `linkType` (`:81`).
8. `Stripe.parseStripeIDEALCallback` → `.completeIDEALDonation` (`:87`).
9. `CallLink(url:)` → `.callLink` (`:90`).

No match → logs `owsFailDebug("Couldn't parse URL")` and returns `nil` (`:93`)
**[High]**. Most matchers are owned by `SignalServiceKit`/`SignalUI`; this
subsystem only orchestrates them **[Medium]** (type names observed at call
sites; their definitions were not read in this session).

`parseSgnlAddStickersUrl` (`Signal/URLs/UrlOpener.swift:97`) is the only matcher
implemented inline: it requires scheme `sgnl` and host prefix `addstickers`,
reads `pack_id`/`pack_key` query items (asserting each appears once), warns on
unknown items, and defers construction to `StickerPackInfo.parse` **[High]**.

## Opening flow (`openUrl` → `_openUrlAfterDismissing`)

`Signal/URLs/UrlOpener.swift:130`. Behavior per case **[High]**:

- **Dismiss decision** — `shouldDismiss` (`:144`) returns `false` only for
  `.completeIDEALDonation`; every other case dismisses an existing presented
  modal first so the destination presents cleanly (`:135`).
- **Registration gate** — all cases except `.signalProxy` call
  `tsAccountManager.registeredStateWithMaybeSneakyTransaction()`; the throwing
  variant (`throws(NotRegisteredError)`) is caught at `:156` and the URL is
  silently ignored with a log when unregistered **[High]**.
- **`.phoneNumberLink`** → `SignalDotMePhoneNumberLink.openChat` (`:163`).
- **`.usernameLink`** → async `UsernameQuerier().queryForUsernameLink`, then
  `SignalApp.shared.presentConversationForAddress` (`:167`).
- **`.stickerPack`** → presents `StickerPackViewController` (`:185`).
- **`.groupInvite`** → `GroupInviteLinksUI.openGroupInviteLink` (`:190`).
- **`.signalProxy`** → presents `ProxyLinkSheetViewController` (no registration
  gate) (`:194`).
- **`.linkDevice`** → requires `registeredState.isPrimary`, else logs and
  returns; otherwise shows an action sheet routing to
  `SignalApp.shared.showAppSettings(mode: .linkedDevices)` (`:196`).
- **`.quickRestore`** → also primary-only; shows an action sheet that opens the
  camera capture view and a `HeroSheetViewController` explaining QR scanning
  (`:224`).
- **`.completeIDEALDonation`** → async; tries
  `DonationViewsUtil.attemptToContinueActiveIDEALDonation` then falls back to
  `restartAndCompleteInterruptedIDEALDonation`, treating
  `Signal.DonationJobError.timeout` as an expected non-error (`:262`).
- **`.callLink`** → `GroupCallViewController.presentLobby(for:)` (`:291`).

## Interactions with the rest of the app

### Entry points (who calls this subsystem)
`UrlOpener` is driven exclusively by `AppLifecycleManager.handleOpenUrl`
(`Signal/AppLaunch/AppLifecycleManager.swift:2090`): it bails if launch failed,
calls `UrlOpener.parseUrl` (returning `false` if unparseable), and defers the
open to `appReadiness.runNowOrWhenUIDidBecomeReadySync`, constructing the
`UrlOpener` with dependencies pulled from `SSKEnvironment` and
`DependenciesBridge` and opening into `self.window!`
(`Signal/AppLaunch/AppLifecycleManager.swift:2097`–`:2109`) **[High]**.

`handleOpenUrl` has two upstream sources **[High]**:
- **Custom-scheme / direct URL opens** from `SceneDelegate` — both the
  cold-start `connectionOptions.urlContexts` path
  (`Signal/AppLaunch/SceneDelegate.swift:41`) and the live
  `scene(_:openURLContexts:)` callback
  (`Signal/AppLaunch/SceneDelegate.swift:98`).
- **Universal links** — `NSUserActivityTypeBrowsingWeb` activities forward their
  `webpageURL` into `handleOpenUrl`
  (`Signal/AppLaunch/AppLifecycleManager.swift:1938`).

### Sibling in-app path (not routed through `UrlOpener`)
Tapping a link inside a conversation does **not** go through `UrlOpener`;
`ConversationViewController` re-implements per-type dispatch directly, including
reusing `SignalDotMePhoneNumberLink.isPossibleUrl` / `openChat`
(`Signal/ConversationView/ConversationViewController+CVComponentDelegate.swift:651`,
`:728`) **[High]**. This means the recognized-URL logic is partially duplicated
between the two call sites rather than centralized **[Medium]** (both sites read;
whether this duplication is intentional — intent undetermined, no evidence in
source).

### Shared constant reuse
`UrlOpener.Constants.sgnlPrefix` is referenced by the device-transfer code when
building outgoing `sgnl://` URLs
(`Signal/DeviceTransfer/MultiPeerConnectivity/MPCDeviceTransferAdvertiser.swift:94`,
`Signal/DeviceTransfer/WiFiAware/WADeviceTransferIncomingConnection.swift:59`),
so this subsystem is the canonical home of the scheme string **[High]**.

### Dependencies on SignalServiceKit / SignalUI (boundary)
Parsing and opening lean on service-layer and UI types owned outside this
subsystem — e.g. `Usernames.UsernameLink`, `StickerPackInfo`,
`PossibleGroupInviteLinkUrl`, `SignalProxy`, `DeviceProvisioningURL`,
`Stripe`/`DonationViewsUtil`, `CallLink`, `TSAccountManager`,
`contactDiscoveryManagerRef`, and presenters like
`StickerPackViewController`, `ProxyLinkSheetViewController`,
`GroupCallViewController`, and `SignalApp` **[Medium]** (observed at call sites;
definitions not read here).

## Data flow / state

The subsystem is **stateless across calls**: `parseUrl` is pure static parsing,
and each `openUrl` constructs no persistent state of its own — the per-feature
destinations own any resulting state **[High]**. The only observable state
dependencies are reads of registration/primary-device status via
`TSAccountManager` and, in the iDEAL path, reads/writes through the injected
`PendingIDEALDonationStore` and donation managers **[High]**
(`Signal/URLs/UrlOpener.swift:262`). Concurrency: opening is `@MainActor`
throughout; the username-link, iDEAL, and `signal.me` contact-discovery paths
spin off `Task`/async work and present results back on the main actor **[High]**.

## Tests

Behavior is exercised by `Signal/test/URLs/UrlOpenerTest.swift`
(`UrlOpener.parseUrl` recognition, `:25`) and
`Signal/test/util/SignalMeTest.swift`
(`SignalDotMePhoneNumberLink.isPossibleUrl` positive/negative cases, `:19`,
`:51`) **[High]**. Test internals were not read in depth in this session
**[Medium]**.

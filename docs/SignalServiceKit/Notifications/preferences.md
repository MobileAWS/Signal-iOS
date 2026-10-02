# Notification Preferences

Covers `SignalServiceKit/Notifications/NotificationPreferencesManager.swift`.

> Confidence labels: **[High]** read from source · **[Medium]** inferred ·
> **[Low]** plausible but unverified. Citations are `File.swift:line` within
> `SignalServiceKit/Notifications/`.

---

## 1. Overview

[High] `NotificationPreferencesManager` is a thin, value-type façade
(`struct`, `NotificationPreferencesManager.swift:43`) over a single
key-value store `NewKeyValueStore(collection: "NotificationPreferences")`
(`:72`). It persists global notification preferences and resolves per-thread
"while-muted" overrides. It is injected via `DependenciesBridge.shared.notificationPreferencesManager`
(used by `NotificationPresenterImpl` and `BadgeCountFetcher`). [High]

Every getter takes a `DBReadTransaction` and returns the stored value or a
default; every setter takes a `DBWriteTransaction`. [High]

---

## 2. Enums

### 2.1 `NotificationType` (preview type)

[High] `NotificationPreferencesManager.swift:6-21`. `UInt`-raw:
- `noNameNoPreview = 0`
- `nameNoPreview = 1`
- `namePreview = 2`

Has a localized `displayName`. These values flow into `NotificationPresenterImpl`
to control title/body/action rendering (see
[notification-presenter.md §3](./notification-presenter.md#31-title--body-preview-type)). [High]

### 2.2 `BadgeCountType`

[High] `NotificationPreferencesManager.swift:23-41`. `Int64`-raw, `CaseIterable`:
- `unreadMessages = 0`
- `unreadChats = 1`

Each has a localized `title`. Consumed by `BadgeCountFetcher` (see
[badge-count.md](./badge-count.md)). [High]

---

## 3. Defaults & storage keys

[High] `Defaults` enum (`NotificationPreferencesManager.swift:44-56`):

| Default | Value |
| --- | --- |
| `globalNotificationSound` | `Sound.standard(.note)` |
| `previewType` | `.namePreview` |
| `playSoundInForeground` | `true` |
| `messageSentSound` | `true` |
| `shouldNotifyOfNewAccounts` | `false` |
| `includeMutedThreadsInBadgeCount` | `false` |
| `badgeCountType` | `.unreadMessages` |
| `shouldNotifyForMentionsWhenMuted` | `true` |
| `notifyForCallsWhenMuted` | `false` |
| `notifyForRepliesWhenMuted` | `true` |
| `areReactionNotificationsEnabled` | `true` |

[High] `Key` enum (`NotificationPreferencesManager.swift:58-70`) maps each
preference to a stable string key (e.g. `"PreviewType"`, `"GlobalNotificationSound"`,
`"BadgeCountType"`, `"NotifyForCallsWhenMuted"`). These keys are the on-disk
contract and must remain stable. [Medium — inferred from the KV-store pattern;
no explicit "stable" comment here]

---

## 4. Global preference accessors

[High] Straightforward get/set pairs backed by `kvStore`:

| Preference | Getter → default | Citation |
| --- | --- | --- |
| Preview type | `previewType(tx:)` → `.namePreview` | `:78-85` |
| Play sound in foreground | `playSoundInForeground(tx:)` → `true` | `:89-95` |
| Message-sent sound | `isMessageSentSoundEnabled(tx:)` → `true` | `:97-103` |
| Reaction notifications | `areReactionNotificationsEnabled(tx:)` → `true` | `:107-113` |
| Notify of new accounts | `shouldNotifyOfNewAccounts(tx:)` → `false` | `:117-123` |
| Include muted in badge | `includeMutedThreadsInBadgeCount(tx:)` → `false` | `:127-133` |
| Badge count type | `badgeCountType(tx:)` → `.unreadMessages` | `:135-141` |
| Global notification sound | `globalNotificationSound(tx:)` → `Sound.standard(.note)` | `:146-150` |

### 4.1 `globalNotificationSound` validation

[High] `setGlobalNotificationSound(_:tx:)` logs the chosen sound, then **guards
on `Sounds.writeFallbackNotificationSoundFile(for:)`** — if writing the fallback
sound file fails, it returns **without** persisting the preference
(`NotificationPreferencesManager.swift:152-162`). The getter
resolves the stored `soundId` via `Sounds.soundForId`, falling back to the
default when unset. [High]

---

## 5. Per-thread "while muted" overrides

[High] Three preferences each have (a) a global default accessor and (b) a
per-thread resolver that falls back to the default when the thread has no
explicit value:

| Preference | Default accessor | Per-thread resolver | Setter (nil ⇒ inherit) |
| --- | --- | --- | --- |
| Calls when muted | `defaultNotifyForCallsWhenMuted(tx:)` → `false` | `notifyForCallsWhenMuted(thread:tx:)` | `setNotifyForCallsWhenMuted(_:thread:tx:)` |
| Mentions when muted | `defaultNotifyForMentionsWhenMuted(tx:)` → `true` | `notifyForMentionsWhenMuted(thread:tx:)` | `setNotifyForMentionsWhenMuted(_:thread:tx:)` |
| Replies when muted | `defaultNotifyForRepliesWhenMuted(tx:)` → `true` | `notifyForRepliesWhenMuted(thread:tx:)` | `setNotifyForRepliesWhenMuted(_:thread:tx:)` |

[High] The per-thread resolvers read `thread.shouldNotifyForCallsWhenMuted` /
`...ForMentionsWhenMuted` / `...ForRepliesWhenMuted` and use `?? default…`
(`NotificationPreferencesManager.swift:164-237`, the "While muted" section). The setters
write the override onto the thread (`updateWith…`), where `nil` clears the
override to inherit the default. [High]

> **Known gap:** each per-thread setter carries the comment
> `// [Notifications] TODO: Storage Service sync` — these overrides are **not yet
> synced** to Storage Service. This matches the `improvedNotifications` build flag
> comment *"Don't enable until Storage Service is integrated."* [High]

### 5.1 `whileMutedEnabledString(thread:tx:)`

[High] Produces a human-readable summary of which "while muted" options are on.
It evaluates calls/mentions/replies (per-thread if a thread is given, else the
defaults), appends the localized titles, and — for group threads (or when no
thread is given) — includes mentions/replies. Returns `CommonStrings.switchOff`
when none are enabled, otherwise an `.and`-joined narrow list
(`NotificationPreferencesManager.swift:239-276` `whileMutedEnabledString`). [High]
The three localized titles (`whileMutedCallsTitle`, `whileMutedMentionsTitle`,
`whileMutedRepliesTitle`) are static constants (`:224-237`). [High]

---

## 6. Reset

[High] `resetAll(tx:)` (`NotificationPreferencesManager.swift:278-283`):
1. `kvStore.removeAll(tx:)` — clears all global prefs.
2. `Sounds.resetThreadNotificationSounds(tx:)`.
3. Re-writes the default global notification sound.
4. `resetPerChatNotificationPreferences(tx:)`.

`resetPerChatNotificationPreferences(tx:)` (`:285-318`) enumerates **non-story** threads via
`ThreadFinder().enumerateNonStoryThreads`, collects threads that deviate from
defaults (to avoid mutating while the cursor is open), then for each resets the
legacy mentions flag and clears any per-thread calls/mentions/replies overrides
(setting them to `nil`). [High]

---

## 7. Interaction with `improvedNotifications`

[High] The *manager itself* always exposes the new per-thread prefs. Whether they
are **used** is gated at the call sites by `BuildFlags.improvedNotifications`
(`build <= .dev`, `SignalServiceKit/Environment/BuildFlags.swift:84`):
- `NotificationPresenterImpl.canNotify(...)` chooses between
  `notifyForMentionsWhenMuted`/`notifyForRepliesWhenMuted` and the legacy
  `shouldNotifyForMentionsWhenMutedLegacy` (`NotificationPresenterImpl.swift:578`
  region). [High]
- `NotificationPresenterImpl.notifyUserOfMissedCall(...)` only consults
  `notifyForCallsWhenMuted` under the flag (`NotificationPresenterImpl.swift:398`). [High]
- `BadgeCountFetcher` only consults `badgeCountType` under the flag
  (`BadgeCountFetcher.swift:29`). [High]

So on production builds these preferences are stored but largely bypassed. [High]

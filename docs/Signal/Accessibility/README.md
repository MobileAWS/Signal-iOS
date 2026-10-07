# Accessibility

Source: `Signal/Accessibility/` — a single file, `SpeechManager.swift`
(`Signal/Accessibility/SpeechManager.swift:9`). This subsystem is the app's
"speak selection" plumbing: it drives iOS text-to-speech for message bodies in a
conversation.

> Confidence: **[High]** read in full; **[Medium]** signature/partial; **[Low]**
> inferred. Citations are `File.swift:line`. An uncited claim is a defect.

## Responsibility

**[High]** `SpeechManager` is a thin wrapper around a single
`AVSpeechSynthesizer` (`Signal/Accessibility/SpeechManager.swift:10`) that speaks
an `AVSpeechUtterance` aloud and can stop playback. It exists so the app has one
long-lived synthesizer whose lifecycle (start/stop) and background behavior are
managed centrally, rather than each call site creating its own synthesizer.

The class conforms to `AVSpeechSynthesizerDelegate`
(`Signal/Accessibility/SpeechManager.swift:9`) and is an `NSObject` so it can be a
delegate and an `@objc` notification observer target.

## Key type

**[High]** `SpeechManager` (`Signal/Accessibility/SpeechManager.swift:9`) exposes:

- `speechSynthesizer` — a private, `nonisolated(unsafe)` `AVSpeechSynthesizer`
  retained for the lifetime of the manager
  (`Signal/Accessibility/SpeechManager.swift:10`). The `init` sets `self` as its
  delegate (`Signal/Accessibility/SpeechManager.swift:12-15`).
- `isSpeaking: Bool` — forwards `speechSynthesizer.isSpeaking`
  (`Signal/Accessibility/SpeechManager.swift:19-21`). Used by the UI to decide
  whether to offer "speak" or "stop speaking".
- `speak(_ utterance:)` — calls `stop()` first (so a new utterance preempts any
  in-progress one), then `speechSynthesizer.speak(utterance)`
  (`Signal/Accessibility/SpeechManager.swift:25-28`).
- `stop()` — `@objc` so it can be a notification `selector`; stops speaking
  `.immediate` (`Signal/Accessibility/SpeechManager.swift:30-33`).
- The five `AVSpeechSynthesizerDelegate` callbacks
  (`Signal/Accessibility/SpeechManager.swift:35-39`), which toggle background
  observation (see below).

## Ownership and wiring

**[High]** There is exactly one `SpeechManager`, owned by `AppEnvironment`:
`speechManagerRef` is declared `let speechManagerRef: SpeechManager`
(`Signal/AppLaunch/AppEnvironment.swift:29`) and constructed in `AppEnvironment`'s
`init` as `SpeechManager()` (`Signal/AppLaunch/AppEnvironment.swift:59`). Callers
reach it through the shared environment singleton,
`AppEnvironment.shared.speechManagerRef`. This matches the broader
`AppEnvironment`-owned object model documented in
`docs/Signal/app-environment.md:81`.

**[Medium]** The manager lives in the Signal app target only (it is listed under
the main app's build sources in
`Signal.xcodeproj/project.pbxproj:3724,19199`), not in `SignalUI` or
`SignalServiceKit`. The only shared-framework coupling is the `AVSpeechUtterance`
type and the `OWSApplicationDidEnterBackground` notification, both of which cross
module boundaries.

## Data flow: speaking a message

**[High]** The feature is surfaced as a message long-press / context-menu action
in the conversation view.

1. When building the per-message action list, `MessageActions` checks state and
   conditionally offers the speak/stop actions
   (`Signal/ConversationView/MessageActions.swift:278-287`): if the manager is
   already speaking (`AppEnvironment.shared.speechManagerRef.isSpeaking`,
   `Signal/ConversationView/MessageActions.swift:281`) it offers "Stop speaking";
   otherwise, if the OS "Speak Selection" setting is on
   (`UIAccessibility.isSpeakSelectionEnabled`,
   `Signal/ConversationView/MessageActions.swift:283`) it offers "Speak". The
   inline comment notes the deliberate ordering so a user can still stop playback
   even after turning the OS setting off mid-speech
   (`Signal/ConversationView/MessageActions.swift:279-280`).
2. Those actions are built by `MessageActionBuilder.speakMessage(...)`
   (`Signal/ConversationView/MessageActions.swift:160`) and
   `stopSpeakingMessage(...)` (`Signal/ConversationView/MessageActions.swift:173`),
   whose blocks call back into the delegate's `messageActionsSpeakItem` /
   `messageActionsStopSpeakingItem` (`Signal/ConversationView/MessageActions.swift:168,180`;
   protocol declared at `Signal/ConversationView/MessageActions.swift:15-16`).
3. `ConversationViewController+MessageActionsDelegate` implements those:
   `messageActionsSpeakItem` extracts the message's displayable body text, builds
   an `AVSpeechUtterance` from the `.text` / `.attributedText` / `.messageBody`
   variant (`Signal/ConversationView/ConversationViewController+MessageActionsDelegate.swift:231-245`),
   and calls `AppEnvironment.shared.speechManagerRef.speak(utterance)`
   (`Signal/ConversationView/ConversationViewController+MessageActionsDelegate.swift:247`).
   `messageActionsStopSpeakingItem` calls `...speechManagerRef.stop()`
   (`Signal/ConversationView/ConversationViewController+MessageActionsDelegate.swift:250-252`).
4. For the `.messageBody` case the utterance comes from
   `HydratedMessageBody.utterance`, which is `AVSpeechUtterance(string: hydratedText)`
   (`SignalServiceKit/Messages/BodyRanges/HydratedMessageBody.swift:821`). This is
   the fully hydrated (mentions resolved) plain text, so spoken output reads the
   rendered message rather than raw markup/placeholders.

```mermaid
graph TD
    MA["MessageActions.buildActions\n(checks isSpeaking / isSpeakSelectionEnabled)"]
    MA -->|speak action| D1["ConversationVC delegate\nmessageActionsSpeakItem"]
    MA -->|stop action| D2["ConversationVC delegate\nmessageActionsStopSpeakingItem"]
    D1 -->|build AVSpeechUtterance\n(HydratedMessageBody.utterance)| SM["AppEnvironment.shared.speechManagerRef"]
    D2 --> SM
    SM -->|speak(_:) / stop()| Synth["AVSpeechSynthesizer"]
    Synth -->|delegate callbacks| BG["background-observer toggle"]
```

## State: background handling

**[High]** The only persistent state is the synthesizer and whether the manager
is currently observing the background notification. On
`didStart` / `didContinue` the manager registers for
`.OWSApplicationDidEnterBackground`
(`Signal/Accessibility/SpeechManager.swift:35,37`,
`listenToApplicationDidEnterBackgroundEvent()` at
`Signal/Accessibility/SpeechManager.swift:42-49`); on
`didPause` / `didFinish` / `didCancel` and in `deinit` it removes the observer
(`Signal/Accessibility/SpeechManager.swift:36,38,39,17`,
`stopListeningToEvents()` at `Signal/Accessibility/SpeechManager.swift:51-53`).
When speech is active and the app backgrounds, the posted notification invokes
`stop()`, immediately halting playback
(`Signal/Accessibility/SpeechManager.swift:44-46`). The notification name itself
is defined in the shared `AppContext`
(`SignalServiceKit/Util/AppContext.swift:152`).

**[Medium]** Observation is scoped to active-speech windows (only registered while
speaking) so the manager isn't permanently subscribed. Because
`removeObserver(self)` is used without a name/object
(`Signal/Accessibility/SpeechManager.swift:52`), it removes all of this object's
observers — fine here since the background notification is the only one it ever
registers.

**[Low]** `speechSynthesizer` is marked `nonisolated(unsafe)`
(`Signal/Accessibility/SpeechManager.swift:10`), signaling the author's intent to
opt out of actor-isolation checking for the synthesizer; call sites (the
conversation delegate and action builder) run on the main actor, so in practice
access is main-thread.

## UI / accessibility considerations

- **[High]** The speak action is gated on the OS-level "Speak Selection"
  accessibility setting (`UIAccessibility.isSpeakSelectionEnabled`,
  `Signal/ConversationView/MessageActions.swift:283`), so Signal only exposes it
  to users who have enabled spoken content system-wide.
- **[High]** Spoken text is the hydrated body
  (`SignalServiceKit/Messages/BodyRanges/HydratedMessageBody.swift:821`),
  ensuring mentions and ranges are read as displayed rather than as raw markup.
- **[High]** The speak/stop message actions carry localized `accessibilityLabel`s
  and stable `accessibilityIdentifier`s
  (`Signal/ConversationView/MessageActions.swift:162-164,175-177`), supporting
  VoiceOver and UI tests.
- **[High]** Backgrounding the app stops playback
  (`Signal/Accessibility/SpeechManager.swift:44-46`), avoiding speech continuing
  when the conversation is no longer on screen.

## Notes

- **[High]** This folder is production code (not tests/mocks/protobufs/assets);
  it contains only `SpeechManager.swift`.
- **[Medium]** Despite the folder name "Accessibility", the subsystem's scope is
  narrow: text-to-speech for messages. Other accessibility concerns (Dynamic
  Type, VoiceOver labeling, etc.) live with the relevant UI code and the
  `SignalUI` font/appearance layers, not here.

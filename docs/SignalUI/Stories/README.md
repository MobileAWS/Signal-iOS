# Stories (SignalUI)

Source: `SignalUI/Stories/`. This is the **shared UI** for *configuring who sees
your stories* and *creating new story destinations* (custom "private" stories and
group stories), plus a couple of small story-related value/helpers used by the
rest of SignalUI. It is distinct from `SignalServiceKit/Stories` (the durable
model / storage / sending layer): everything here is view controllers, views, and
UI-side glue that *drives* the SSK story types rather than defining them.

> Confidence: **[High]** read in full; **[Medium]** signature/partial; **[Low]**
> inferred. Citations are `File.swift:line`. An uncited claim is a defect.

## Responsibility & scope

**[High]** The directory contains eleven Swift files and no tests, mocks, protobufs,
or asset catalogs (enumerated from the tree). Their collective responsibility is the
**story-settings and story-creation UI surface**:

- Pick/edit the audience for "My Story" and private stories
  (`SelectMyStoryRecipientsViewController`, `MyStorySettingsViewController`).
- Create new private stories and group stories
  (`NewPrivateStoryRecipientsViewController` → `NewPrivateStoryConfirmViewController`,
  `NewGroupStoryViewController`, and the `NewStoryHeaderView` entry point).
- Educate the user about "Signal Connections" and list them
  (`ConnectionsEducationSheetViewController`, `AllSignalConnectionsViewController`).
- Provide small story value/helpers consumed elsewhere in SignalUI
  (`StorySharing`, `StoryMessage+SignalUI`, `StoryContextViewState`).

**[High]** Every file begins with `import SignalServiceKit` (or
`public import`), consistent with the framework-wide dependency on SSK described in
[../README.md](../README.md). The audience/creation screens build on SignalUI's own
base classes and recipient-picking machinery (see
[../ViewControllers.md](../ViewControllers.md) and
[../RecipientPickers.md](../RecipientPickers.md)) rather than defining new UI
primitives.

```mermaid
graph TD
    subgraph "Story settings"
        MSS[MyStorySettingsViewController / Sheet]
        SMSR[SelectMyStoryRecipientsViewController]
        ASC[AllSignalConnectionsViewController]
        CES[ConnectionsEducationSheetViewController]
    end
    subgraph "Story creation"
        NSH[NewStoryHeaderView]
        NPSR[NewPrivateStoryRecipientsViewController]
        NPSC[NewPrivateStoryConfirmViewController]
        NGS[NewGroupStoryViewController]
    end
    MSS -->|"Only share with" / "except"| SMSR
    MSS -->|"All connections → View"| ASC
    MSS -->|Learn more| CES
    NSH -->|custom| NPSR --> NPSC
    NSH -->|group| NGS
    NPSC -->|creates| TSPST[(TSPrivateStoryThread)]
    SMSR -->|updates view mode / recipients| TSPST
    NGS -->|enables story send| TSGT[(TSGroupThread)]
```

## Key types

### Story-settings screens

**[High]** `MyStorySettingsViewController` is an `OWSTableViewController2` that
renders the "My Story" privacy settings
(`SignalUI/Stories/MyStorySettingsViewController.swift:11`), and
`MyStorySettingsSheetViewController` is the `OWSTableSheetViewController` variant
(`SignalUI/Stories/MyStorySettingsViewController.swift:28`). Both delegate all table
construction to a private `MyStorySettingsDataSource`
(`SignalUI/Stories/MyStorySettingsViewController.swift:58`) whose
`generateTableContents(style:)` is parameterized by a `Style` enum —
`.sheet` vs `.fullscreen` — that controls whether section headers, the
replies/reactions toggle, and the "learn more" footer are shown
(`SignalUI/Stories/MyStorySettingsViewController.swift:67`,
`SignalUI/Stories/MyStorySettingsViewController.swift:74`). The three visibility
options it builds — "All Signal Connections", "All except…", and "Only share
with…" — map to SSK's `TSThreadStoryViewMode` (`.blockList` with/without recipients,
and `.explicit`) and persist via `TSPrivateStoryThread.updateWithStoryViewMode(...)`
inside a `databaseStorage` write. **[Medium]** (the `TSThreadStoryViewMode` /
`updateWithStoryViewMode` / `TSPrivateStoryThread.getMyStory` symbols are SSK
types read by signature.)

**[High]** `SelectMyStoryRecipientsViewController` extends
`BaseMemberViewController` (the recipient-member picker base from
[../RecipientPickers.md](../RecipientPickers.md)) and is the screen for choosing the
allow-list or block-list for a private story
(`SignalUI/Stories/SelectMyStoryRecipientsViewController.swift:9`). It is created via
a static `load(for:mode:tx:completionBlock:)` that seeds the recipient set from
`DependenciesBridge.shared.storyRecipientManager`
(`SignalUI/Stories/SelectMyStoryRecipientsViewController.swift:19`), tracks unsaved
changes by comparing against the original set
(`SignalUI/Stories/SelectMyStoryRecipientsViewController.swift:15`), and on save
resolves addresses to recipient IDs and calls
`thread.updateWithStoryViewMode(mode, storyRecipientIds: .setTo(...), updateStorageService: true, ...)`
(`SignalUI/Stories/SelectMyStoryRecipientsViewController.swift:107`). It conforms to
`MemberViewDelegate`, and in `.blockList` mode shows a red "x-circle-fill" indicator
and allows selecting blocked recipients. **[Medium]** (`TSThreadStoryViewMode`,
`storyRecipientManager`, `recipientFetcher` are SSK collaborators.)

**[High]** `AllSignalConnectionsViewController` is an `OWSTableViewController2` that
lists all whitelisted/registered addresses (minus the local user), grouped by
`UILocalizedIndexedCollation`
(`SignalUI/Stories/AllSignalConnectionsViewController.swift:9`). It renders rows with
`ContactTableViewCell` (a SignalUI recipient cell). It is read-only — a "View"
destination from the settings screen.

**[High]** `ConnectionsEducationSheetViewController` is a `HeroSheetViewController`
explainer sheet (image + title + bullet list) describing Signal Connections
(`SignalUI/Stories/ConnectionsEducationSheetViewController.swift:8`). It is presented
when the user taps the "Learn more" link in the My Story settings footer; the data
source overrides the text-view link tap to present this sheet instead of opening the
URL (`SignalUI/Stories/MyStorySettingsViewController.swift:58` data source +
`UITextViewDelegate`). It ships a DEBUG `#Preview`.

### Story-creation screens

**[High]** `NewStoryHeaderView` is a `UIStackView` table-section header with an
"add new story" button whose menu offers "custom story" and "group story"
(`SignalUI/Stories/NewStoryHeaderView.swift:12`). Its `NewStoryHeaderDelegate`
protocol refines `OWSTableViewController2`
(`SignalUI/Stories/NewStoryHeaderView.swift:8`) and is notified via
`newStoryHeaderView(_:didCreateNewStoryItems:)` with the resulting
`StoryConversationItem`s. Tapping "custom" presents
`NewPrivateStoryRecipientsViewController`
(`SignalUI/Stories/NewStoryHeaderView.swift:95`); tapping "group" presents
`NewGroupStoryViewController` (`SignalUI/Stories/NewStoryHeaderView.swift:103`), each
wrapped in an `OWSNavigationController` form sheet.

**[High]** `NewPrivateStoryRecipientsViewController` (another
`BaseMemberViewController`) is step 1 of custom-story creation: it collects the
viewer set (`SignalUI/Stories/NewPrivateStoryRecipientsViewController.swift:9`) and on
"Next" pushes `NewPrivateStoryConfirmViewController`
(`SignalUI/Stories/NewPrivateStoryRecipientsViewController.swift:57`). It carries an
optional `selectItemsInParent` callback so the originating screen can auto-select the
newly created story.

**[High]** `NewPrivateStoryConfirmViewController` is step 2: an
`OWSTableViewController2` for naming the story, toggling replies/reactions, and
listing viewers (`SignalUI/Stories/NewPrivateStoryConfirmViewController.swift:10`).
`didTapCreate()` validates the name, then does an awaitable DB write that inserts a
`TSPrivateStoryThread(name:allowsReplies:viewMode:.explicit)`, resolves recipient
IDs, and sets them via `storyRecipientManager`
(`SignalUI/Stories/NewPrivateStoryConfirmViewController.swift:157`). On completion it
dismisses and invokes `selectItemsInParent` with a `StoryConversationItem` wrapping
the new `.privateStory` backing item. **[Medium]** (`TSPrivateStoryThread`,
`storyRecipientManager`, `recipientFetcher` are SSK types.)

**[High]** `NewGroupStoryViewController` extends
`ConversationPickerViewController` (see [../RecipientPickers.md](../RecipientPickers.md))
filtered to groups whose story-send is not yet explicitly enabled and that can
receive chat messages (`SignalUI/Stories/NewGroupStoryViewController.swift:8`,
`threadFilter` at `SignalUI/Stories/NewGroupStoryViewController.swift:18`). On
selection completion it flips `TSGroupThread.updateWithStorySendEnabled(true, ...)`
for each chosen group and reports back `.groupStory` `StoryConversationItem`s via
`selectItemsInParent`. **[Medium]** (`TSGroupThread`, `GroupConversationItem` are
defined outside this directory.)

### Small value types & helpers

**[High]** `StorySharing` is a stateless enum that turns a composed text message
into a *text story* and enqueues it to the story-type conversations in a selection
(`SignalUI/Stories/StorySharing.swift:8`). `enqueueTextStory(with:linkPreviewDraft:to:)`
filters the passed `ConversationItem`s to those whose `outgoingMessageType ==
.storyMessage` and hands a built `UnsentTextAttachment` to
`AttachmentMultisend.enqueueTextAttachment(...)`
(`SignalUI/Stories/StorySharing.swift:9`; see
[../AttachmentFlows.md](../AttachmentFlows.md)). The default text-story styling (white
text on a `0x688BD4` background, `.regular` style) is defined in `buildTextAttachment`.
`text(for:with:)` hydrates mentions and then strips the link-preview URL from the body
when it is the whole message, a leading, or a trailing token — otherwise leaving the
body unchanged (`SignalUI/Stories/StorySharing.swift:37`).

**[High]** `StoryMessage+SignalUI` extends SSK's `StoryMessage` with
`quotedBody(transaction:)`, which derives the `MessageBody` to show when a story is
quoted/replied-to: for `.text` stories it returns the styled body (or the link-preview
URL when empty); for `.media` stories it fetches the attachment's
`storyMediaCaption` and reconstructs a `MessageBody` with its collapsed styles
(`SignalUI/Stories/StoryMessage+SignalUI.swift:10`). **[Medium]** (`StoryMessage`,
`attachmentStore`, `MessageBody*` are SSK types; the SignalUI role here is the
display-body derivation.)

**[High]** `StoryContextViewState` is a tiny `Equatable` enum —
`unviewed` / `viewed` / `noStories` — with a `hasStoriesToDisplay` convenience
(`SignalUI/Stories/StoryContextViewState.swift:8`). It encodes the per-context story
badge/viewed state used by consumers of story UI. **[Low]** (its consumers live
outside this directory; its purpose is inferred from the case names and the
`hasStoriesToDisplay` helper.)

## Data flows & state

**[High]** The settings/creation screens are **thin UI over SSK persistence**:
reads come from `SSKEnvironment.shared.databaseStorageRef.read` and
`DependenciesBridge.shared` managers (`storyRecipientManager`, `storyRecipientStore`,
`recipientFetcher`), and every mutation is a `databaseStorage` write that calls an
SSK `TSPrivateStoryThread`/`TSGroupThread` update with
`updateStorageService: true` so the change syncs
(`SignalUI/Stories/SelectMyStoryRecipientsViewController.swift:107`,
`SignalUI/Stories/NewPrivateStoryConfirmViewController.swift:157`,
`SignalUI/Stories/MyStorySettingsViewController.swift:74`). **[Medium]** (storage
service / sync behavior lives in SSK.)

**[High]** Creation screens communicate results **upward via closures** rather than
delegates/notifications: `selectItemsInParent: (([StoryConversationItem]) -> Void)?`
threads from `NewStoryHeaderView` through the recipients/confirm/group controllers so
the originating list can insert and pre-select the new story. The in-progress
audience is held as an `OrderedSet<PickedRecipient>` (the recipient-picker value type
from [../RecipientPickers.md](../RecipientPickers.md)), and "unsaved changes" state
gates the save/next button (`…RecipientsViewController.swift` `hasUnsavedChanges`).

## Notable UI considerations

- **[High]** The My Story settings UI is rendered **twice** from one data source via
  the `Style` enum (full-screen vs. bottom sheet), with the sheet omitting headers,
  the replies toggle, and subtitles
  (`SignalUI/Stories/MyStorySettingsViewController.swift:67`). This keeps the two
  presentations in sync.
- **[High]** The "Learn more" link uses a `LinkingTextView` whose URL tap is
  intercepted to present `ConnectionsEducationSheetViewController` instead of opening
  the (placeholder) support URL — the code comments explicitly say the link target
  "doesn't matter" (`SignalUI/Stories/MyStorySettingsViewController.swift` `Constants`
  + `UITextViewDelegate`).
- **[High]** Titles are pluralization-aware, using `PluralAware`-tabled
  `OWSLocalizedString` format strings that embed the selected/excluded count
  (e.g. `SignalUI/Stories/SelectMyStoryRecipientsViewController.swift` title logic),
  and all user-facing text is localized.
- **[High]** New-story flows are presented as `OWSNavigationController` **form
  sheets** from the header view (`SignalUI/Stories/NewStoryHeaderView.swift:95`),
  and screens use SignalUI theming (`UIColor.Signal.*`, Dynamic Type `dynamicType…`
  fonts) consistent with [../Appearance.md](../Appearance.md) and
  [../FontsAndFormatStyles.md](../FontsAndFormatStyles.md).
- **[Medium]** `NewGroupStoryViewController` filters out groups that already have
  story-send explicitly enabled, so the creation list only offers groups that would
  actually be *turned into* story destinations
  (`SignalUI/Stories/NewGroupStoryViewController.swift:18`).

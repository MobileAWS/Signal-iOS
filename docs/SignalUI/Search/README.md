# Search (SignalUI)

Source: `SignalUI/Search/`. The **UI-facing full-text / name search** layer. It
turns a user's search string into the typed, sorted result sets that SignalUI's
search surfaces render — the home-screen (chat list) search, the recipient picker
search, and in-conversation message search. It is a thin orchestration layer: it
assembles results on top of SignalServiceKit's indexes (`SearchableNameIndexer`,
`FullTextSearchIndexer`, `MentionFinder`) rather than owning any index itself.

> **Not to be confused with** `SignalServiceKit/.../Search` (the underlying
> indexing/storage). This doc covers only `SignalUI/Search/`, which depends on and
> drives those SSK indexers.

> Confidence: **[High]** read in full; **[Medium]** signature/partial or depends on
> collaborators defined outside `SignalUI/`; **[Low]** inferred. Citations are
> `File.swift:line`. An uncited claim is a defect.

## Contents

**[High]** The directory contains exactly two Swift files (confirmed from the
tree): `FullTextSearcher.swift` and `SearchableNameFinder.swift`. There are no
tests, mocks, protobufs, or asset catalogs here. Both `public import
SignalServiceKit` (`SignalUI/Search/FullTextSearcher.swift:9`,
`SignalUI/Search/SearchableNameFinder.swift:8`), consistent with the framework-wide
convention described in [../README.md](../README.md).

## Responsibility

**[High]** `FullTextSearcher` is the public entry point for all three search
surfaces (`SignalUI/Search/FullTextSearcher.swift:236`), exposed as a shared
singleton `FullTextSearcher.shared`
(`SignalUI/Search/FullTextSearcher.swift:240`) with a default result cap
`kDefaultMaxResults = 500` (`SignalUI/Search/FullTextSearcher.swift:238`). It
provides three query methods, each returning a different typed result set:

- `searchForHomeScreen(searchText:maxResults:tx:)`
  (`SignalUI/Search/FullTextSearcher.swift:343`) → `HomeScreenSearchResultSet`.
- `searchForRecipients(searchText:includeLocalUser:includeStories:maxResults:tx:)`
  (`SignalUI/Search/FullTextSearcher.swift:242`) → `RecipientSearchResultSet`.
- `searchWithinConversation(threadUniqueId:isGroupThread:searchText:maxResults:transaction:)`
  (`SignalUI/Search/FullTextSearcher.swift:642`) → `ConversationScreenSearchResultSet`.

**[High]** `SearchableNameFinder` is the companion that resolves a search string
to matching **contact/thread names** by driving SSK's `SearchableNameIndexer`
(`SignalUI/Search/SearchableNameFinder.swift:10`,
`SignalUI/Search/SearchableNameFinder.swift:28`). `FullTextSearcher` constructs a
fresh `SearchableNameFinder` for each query from `DependenciesBridge.shared` /
`SSKEnvironment.shared` services (e.g.
`SignalUI/Search/FullTextSearcher.swift:257-259`).

```mermaid
graph TD
    subgraph SignalUI/Search
        FTS[FullTextSearcher.shared]
        SNF[SearchableNameFinder]
    end
    HS[Chat-list search UI] --> FTS
    RP[Recipient picker search] --> FTS
    CV[In-conversation search] --> FTS
    FTS --> SNF
    SNF -->|name index| SNI[SearchableNameIndexer SSK]
    FTS -->|message FTS| FTSI[FullTextSearchIndexer SSK]
    FTS -->|@mentions| MF[MentionFinder SSK]
    FTS --> RS[(Typed result sets)]
    classDef sui fill:#e8f0fe,stroke:#4285f4;
    class FTS,SNF sui;
```

## `SearchableNameFinder` — name → addresses

**[High]** `searchNames(for:maxResults:localIdentifiers:tx:addGroupThread:addStoryThread:)`
(`SignalUI/Search/SearchableNameFinder.swift:28`) calls
`searchableNameIndexer.search(...)` and switches on the heterogeneous
`indexableName` matches it yields: `SignalAccount`, `OWSUserProfile`,
`SignalRecipient`, `UsernameLookupRecord`, `NicknameRecord` are folded into an
internal `ContactMatches` accumulator, while `TSGroupThread` and
`TSPrivateStoryThread` matches are forwarded to the caller-supplied
`addGroupThread` / `addStoryThread` closures
(`SignalUI/Search/SearchableNameFinder.swift:70-74`). A `TSContactThread` match is
ignored (`SignalUI/Search/SearchableNameFinder.swift:76`), and any other type trips
`owsFailDebug` (`SignalUI/Search/SearchableNameFinder.swift:80`).

**[High]** The function is **cancellation-aware**: it is typed
`throws(CancellationError)` and checks `Task.isCancelled` inside the match callback
(`SignalUI/Search/SearchableNameFinder.swift:42-44`), throwing so a superseded
search aborts promptly.

**[High]** `ContactMatches` (`SignalUI/Search/SearchableNameFinder.swift:87`)
keys partial matches by `SignalServiceAddress`, merging the several record types
that can describe the same address into one `ContactMatch`. Two filtering rules are
notable:

- **Phone-number visibility & registration.** A `SignalRecipient` match is kept
  only if it `isRegistered` **and** `phoneNumberVisibilityFetcher.isPhoneNumberVisible`
  returns true (`SignalUI/Search/SearchableNameFinder.swift:131-134`) — i.e. you
  cannot find a user by a phone number they've hidden.
- **Display-name consistency.** `matchedAddresses(contactManager:tx:)`
  (`SignalUI/Search/SearchableNameFinder.swift:154`) resolves each address'
  `displayName` and only returns the address if the *kind* of name actually shown
  (nickname / system contact / profile / username) corresponds to a record that
  matched (`SignalUI/Search/SearchableNameFinder.swift:159-171`). A bare
  `signalRecipient` match is also admitted. **[Medium]** The intent reads as "don't
  surface a result unless the string the user typed matches the name the user will
  actually see" — inferred from the switch structure.

## Typed result sets & sort keys

**[High]** `FullTextSearcher.swift` defines a family of `Comparable` result value
types so the UI can sort deterministically:

- `ConversationSortKey` (`SignalUI/Search/FullTextSearcher.swift:13`) — orders
  contact threads first, then by `lastInteractionRowId`, then creation date.
- `ConversationSearchResult<SortKey>` (`SignalUI/Search/FullTextSearcher.swift:36`)
  — wraps a `ThreadViewModel` plus optional message id/date and a `CVTextValue`
  snippet.
- `ContactSearchResult` (`SignalUI/Search/FullTextSearcher.swift:66`) — sorts by
  most-recent chat, falling back to `ComparableDisplayName`.
- `GroupSearchResult` (`SignalUI/Search/FullTextSearcher.swift:146`), including the
  `withMatchedMembersSnippet(...)` factory that builds the "matched members"
  subtitle via `groupThread.sortedMemberNames(...)`
  (`SignalUI/Search/FullTextSearcher.swift:152`).
- `StorySearchResult` (`SignalUI/Search/FullTextSearcher.swift:104`),
  `MessageSearchResult` (`SignalUI/Search/FullTextSearcher.swift:203`).

**[High]** The surface-specific aggregate result sets are
`HomeScreenSearchResultSet` (`SignalUI/Search/FullTextSearcher.swift:119`, with an
`.empty` at `:126` and `isEmpty` at `:136`), `RecipientSearchResultSet`
(`SignalUI/Search/FullTextSearcher.swift:188`, with `groupThreads` at `:194`),
and `ConversationScreenSearchResultSet`
(`SignalUI/Search/FullTextSearcher.swift:214`, which lazily derives
`messageSortIds` for highlighting — `SignalUI/Search/FullTextSearcher.swift:218`).

## Data flow — home-screen search

**[High]** `_searchForHomeScreen(...)`
(`SignalUI/Search/FullTextSearcher.swift:355`) is the richest flow and encodes the
subsystem's ordering policy. It:

1. Builds a `NameResolverImpl`
   (`SignalUI/Search/FullTextSearcher.swift:366`) and sets up local **caches** for
   threads and `ThreadViewModel`s so repeated lookups during one search are cheap
   (`SignalUI/Search/FullTextSearcher.swift:369`,
   `SignalUI/Search/FullTextSearcher.swift:379`).
2. Runs name search first (chats/contacts), then group membership, then message
   full-text — explicitly so a query like "Matthew" surfaces the *chat* with that
   contact above *messages* mentioning the name (comment at
   `SignalUI/Search/FullTextSearcher.swift:501`).
3. Removes the local ACI address and re-adds Note-to-Self only when the query
   actually matches the local name/number or the localized "Note to Self" string,
   via `noteToSelfMatch(...)`
   (`SignalUI/Search/FullTextSearcher.swift:328`,
   `SignalUI/Search/FullTextSearcher.swift:538-542`).
4. For each remaining address, `appendAddress(...)`
   (`SignalUI/Search/FullTextSearcher.swift:456`) adds a contact-thread result, a
   bare contact result (if whitelisted), group results, and mention results; it is
   invoked for every address at `SignalUI/Search/FullTextSearcher.swift:551`.
5. Runs `FullTextSearchIndexer.search(...)`
   (`SignalUI/Search/FullTextSearcher.swift:564`) for message bodies and builds the
   highlighted snippet (see below), then sorts each bucket before returning
   (`SignalUI/Search/FullTextSearcher.swift:635-638`).

**[High]** `remainingResultCount()`
(`SignalUI/Search/FullTextSearcher.swift:494`) caps total work across all buckets
against `maxResults`, and the message loop stops once the budget is exhausted
(`SignalUI/Search/FullTextSearcher.swift:569-570`).

**[High]** **Cancellation is checked repeatedly** between phases
(`Task.isCancelled` at `SignalUI/Search/FullTextSearcher.swift:507`, `:533`, `:547`,
`:560`, `:610`), including before the first DB query because the search may already
have been superseded while waiting for the database (comment at
`SignalUI/Search/FullTextSearcher.swift:504`).

## Snippet highlighting (notable UI consideration)

**[High]** For message results, the raw snippet string from the SSK indexer carries
match ranges via `FullTextSearchIndexer.matchTag`; the searcher converts those
XML-tagged ranges into **bold** `MessageBodyRanges.Style` runs using a BonMot
`StringStyle` and a private `OWSSearchMatch` attribute key
(`SignalUI/Search/FullTextSearcher.swift:577-580`), then merges the styling back
into the message body (`SignalUI/Search/FullTextSearcher.swift:592`) and hydrates
mentions (`SignalUI/Search/FullTextSearcher.swift:604`). The result is a
`CVTextValue.messageBody` so the match can be rendered highlighted by the
conversation-view text primitives (see [../ConversationView.md](../ConversationView.md)).
**[Medium]** The mechanics are explicit in-source; the downstream rendering lives in
ConversationView.

## Data flow — recipients & in-conversation

**[High]** `searchForRecipients(...)`
(`SignalUI/Search/FullTextSearcher.swift:242`) reuses `SearchableNameFinder`
(`SignalUI/Search/FullTextSearcher.swift:257`), honoring `includeLocalUser`
(Note-to-Self) and `includeStories` (adds `StorySearchResult`s, skipping
`.disabled` private story threads queued for deletion —
`SignalUI/Search/FullTextSearcher.swift:289`), and only returns contacts that are in
the profile whitelist (`SignalUI/Search/FullTextSearcher.swift:309`). Buckets are
sorted on the way out (`SignalUI/Search/FullTextSearcher.swift:317-318`).

**[High]** `searchWithinConversation(...)`
(`SignalUI/Search/FullTextSearcher.swift:642`) runs the message full-text indexer
(`SignalUI/Search/FullTextSearcher.swift:657`) filtered to one `threadUniqueId`,
and — **only for group threads** (`canSearchForMentions`,
`SignalUI/Search/FullTextSearcher.swift:671-672`) — also includes messages that
@-mention any name-matched member via
`MentionFinder.messagesMentioning(aci:in:tx:)`
(`SignalUI/Search/FullTextSearcher.swift:694`). Results are returned
most-recent-first (`SignalUI/Search/FullTextSearcher.swift:700`).

## Interactions with the rest of SignalUI / the app

- **[High]** Results are expressed in SignalUI/SSK view-model terms —
  `ThreadViewModel` (used throughout, e.g.
  `SignalUI/Search/FullTextSearcher.swift:379`) and `CVTextValue` snippets — so the
  consuming UI (chat-list and recipient-picker screens) can render rows and
  highlighted previews directly. `ThreadViewModel` is the same type referenced in
  [../RecipientPickers.md](../RecipientPickers.md).
- **[Medium]** All collaborators that do the heavy lifting are defined **outside**
  `SignalUI/Search/`: `SearchableNameIndexer`, `FullTextSearchIndexer`,
  `MentionFinder`, `ContactManager`, `PhoneNumberVisibilityFetcher`,
  `RecipientDatabaseTable`, and the profile/account managers are SSK types reached
  through `DependenciesBridge.shared` / `SSKEnvironment.shared`
  (`SignalUI/Search/FullTextSearcher.swift:257-259`). This subsystem is pure
  orchestration + presentation shaping.
- **[Medium]** All public query methods are synchronous but
  `throws(CancellationError)` and read-transaction based (`tx: DBReadTransaction`);
  the surrounding UI is expected to run them off the main thread inside a
  cancellable `Task` and discard stale results — the pervasive `Task.isCancelled`
  checks only matter under that usage. (Caller code lives outside this directory.)

## Notable considerations

- **Determinism:** every result type is `Comparable` with an explicit ordering;
  buckets are sorted before return so the UI never has to re-sort
  (`SignalUI/Search/FullTextSearcher.swift:635-638`).
- **Privacy filtering:** hidden phone numbers, unregistered recipients, and
  non-whitelisted profiles are filtered out at the search layer, not the view
  layer (`SignalUI/Search/SearchableNameFinder.swift:131-134`,
  `SignalUI/Search/FullTextSearcher.swift:309`).
- **Debug affordance:** when internal logging is on, a 13-digit numeric query is
  treated as a message `timestamp` lookup
  (`SignalUI/Search/FullTextSearcher.swift:615-623`). **[Medium]** Clearly a
  developer convenience; gated on `DebugFlags.internalLogging`.

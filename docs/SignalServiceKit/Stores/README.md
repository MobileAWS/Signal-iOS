# SignalServiceKit — Stores module

This documentation set covers the small **settings-store** module in
[`SignalServiceKit/Stores/`](../../../SignalServiceKit/Stores/). The directory
contains exactly two Swift files, each a thin, typed façade over a
`KeyValueStore`-backed collection that persists a single feature's user
preference:

| Source file | Store type | Backing collection |
| --- | --- | --- |
| [`CallServiceSettingsStore.swift`](../../../SignalServiceKit/Stores/CallServiceSettingsStore.swift) | `CallServiceSettingsStore` | `"CallService"` |
| [`ThemeDataStore.swift`](../../../SignalServiceKit/Stores/ThemeDataStore.swift) | `ThemeDataStore` | `"ThemeCollection"` |

Both types are small `public struct`s with a no-argument initializer, each
owning a private `KeyValueStore` instance and exposing read/write accessors that
take a `DBReadTransaction` / `DBWriteTransaction`. **[High]**
(`CallServiceSettingsStore.swift:10-17`, `ThemeDataStore.swift:6-19`)

> **Confidence labels.** Each claim is tagged with a confidence level:
> - **[High]** — directly read from source; behavior is explicit in the code.
> - **[Medium]** — inferred from code with reasonable certainty, but depends on
>   collaborators defined outside `SignalServiceKit/Stores/`.
> - **[Low]** — inferred from naming/comments; not fully verified in-tree.
>
> Where the source gives no evidence for a design question, the text says
> **"intent undetermined — no evidence in source"** rather than guessing.
>
> **Citations** are given as `File.swift:line`, relative to the repository root,
> pointing at the definition being described. Line numbers reflect the state of
> the tree at authoring time and may drift as code changes; use the cited symbol
> name to re-locate code if lines have moved.

## Responsibility

The module's single responsibility is **durable persistence of two
user-facing preferences** behind narrow, strongly-typed interfaces, so callers
never touch raw `KeyValueStore` keys or raw integer encodings directly:

- **Call data-usage preference** — which network interfaces (Wi-Fi, cellular, or
  both) are permitted to use "high data" during calls. **[High]**
  (`CallServiceSettingsStore.swift:19-48`)
- **Theme / appearance preference** — whether the app follows the system
  appearance or is forced to light/dark, with migration from a legacy boolean
  setting. **[High]** (`ThemeDataStore.swift:21-53`)

Each store encapsulates its own `KeyValueStore` collection name and key strings
as private members, so the on-disk schema is an implementation detail that
cannot leak to callers. **[High]** (`CallServiceSettingsStore.swift:11-15`,
`ThemeDataStore.swift:14-18`)

## `CallServiceSettingsStore`

Defined at `CallServiceSettingsStore.swift:10`, this store holds a single
preference keyed by `"HighBandwidthPreferenceKey"`. A source comment notes the
key name is historical: it "used to be called 'high bandwidth', but 'data' is
more accurate" — the on-disk key string is deliberately **not** renamed to
preserve compatibility with existing installs. **[High]**
(`CallServiceSettingsStore.swift:11-14`)

### API

- `setHighDataInterfaces(_:tx:)` persists a `NetworkInterfaceSet` by storing its
  `rawValue` as a `UInt` via `keyValueStore.setUInt(...)`, logs the change, and
  schedules a notification on write-commit. **[High]**
  (`CallServiceSettingsStore.swift:19-34`)
- `highDataNetworkInterfaces(tx:)` reads the stored `UInt`; when **no** value has
  been persisted it falls back to `.wifiAndCellular` (i.e. both interfaces
  allowed by default), otherwise it reconstructs the set from the stored
  `rawValue`. **[High]** (`CallServiceSettingsStore.swift:36-47`)

### Change notification

On a successful write, the store uses `tx.addSyncCompletion { … }` to post the
`callServicePreferencesDidChange` notification on the main thread. The write is
therefore observable only **after** the transaction's sync completion runs, not
mid-transaction. **[High]** (`CallServiceSettingsStore.swift:29-33`)

The notification name is declared as a `public static` extension on
`Notification.Name` at the top of the file, with the underlying string
`"CallServicePreferencesDidChange"`. **[High]**
(`CallServiceSettingsStore.swift:6-8`)

### `NetworkInterfaceSet` dependency

The stored/returned type `NetworkInterfaceSet` is an `OptionSet` defined in the
sibling networking directory at
`SignalServiceKit/Network/NetworkInterfaceSet.swift:17`. The default returned by
`highDataNetworkInterfaces(tx:)`, `.wifiAndCellular`, is itself defined there as
`[.cellular, .wifi]`. **[High]**
(`NetworkInterfaceSet.swift:17-26`, `CallServiceSettingsStore.swift:42-46`)
Because this type lives outside `SignalServiceKit/Stores/`, see the networking
layer docs for its full semantics: [`../Network/README.md`](../Network/README.md).

## `ThemeDataStore`

Defined at `ThemeDataStore.swift:6`, this store persists the app's appearance
preference as a `UInt` under key `"ThemeKeyCurrentMode"`, in collection
`"ThemeCollection"`. **[High]** (`ThemeDataStore.swift:14-18`)

### `Appearance` enum

The store defines a nested `public enum Appearance: UInt` with three cases —
`system` (raw 0), `light` (raw 1), `dark` (raw 2), in declaration order. The raw
values are what get written to disk, so case ordering is effectively a persisted
contract. **[High]** (`ThemeDataStore.swift:8-12`)

### API

- `getCurrentMode(tx:)` returns an `Appearance`, defaulting to `.system`. **[High]**
  (`ThemeDataStore.swift:21-48`)
- `setCurrentMode(_:tx:)` writes `mode.rawValue` as a `UInt` under
  `Keys.currentMode`. **[High]** (`ThemeDataStore.swift:50-52`)

### Legacy-setting migration

`getCurrentMode(tx:)` implements a one-way migration from an older boolean
setting:

1. If `Keys.currentMode` (`"ThemeKeyCurrentMode"`) has a value, decode it as an
   `Appearance`; an unrecognized raw value leaves the mode at `.system`. **[High]**
   (`ThemeDataStore.swift:23-33`)
2. Otherwise, if the legacy key `Keys.legacyThemeEnabled`
   (`"ThemeKeyThemeEnabled"`) exists, treat its boolean value as "dark enabled":
   `true` → `.dark`, `false` → `.light`. **[High]**
   (`ThemeDataStore.swift:34-45`)
3. If neither key is present, default to `.system`. **[High]**
   (`ThemeDataStore.swift:22-47`)

Note that `getCurrentMode(tx:)` only **reads** the legacy key for interpretation;
it does not rewrite the preference under the new key within this method, so the
legacy value is re-interpreted on each read until `setCurrentMode(_:tx:)` is
called to persist a modern `currentMode`. **[High]**
(`ThemeDataStore.swift:21-52`)

## Shared backing dependency: `KeyValueStore`

Both stores are built on `KeyValueStore`, constructed with a per-feature
`collection:` string. **[High]** (`CallServiceSettingsStore.swift:15`,
`ThemeDataStore.swift:17`) `KeyValueStore` itself is defined outside this module
at `SignalServiceKit/Storage/Database/KeyValueStore.swift:8`, and provides the
`getUInt` / `setUInt` / `getBool` / `hasValue` primitives these stores call.
**[High]** (`KeyValueStore.swift:8`) The transaction types `DBReadTransaction`
and `DBWriteTransaction` threaded through every accessor are likewise defined
elsewhere in the storage layer; their lifecycle (and `addSyncCompletion`
semantics used by `CallServiceSettingsStore`) are owned by that layer. **[Medium]**
(`CallServiceSettingsStore.swift:29`)

## Design observations

- **Uniform shape.** Both files follow the same pattern — a `public struct` with
  a private `Keys` enum of string constants, a private `KeyValueStore`, an empty
  `public init()`, and transaction-scoped accessors — making this directory a
  cohesive "typed preference façade" module rather than a general store
  framework. **[High]** (`CallServiceSettingsStore.swift:10-17`,
  `ThemeDataStore.swift:6-19`)
- **Backward-compatible key naming.** Both files intentionally retain legacy key
  strings/encodings (`"HighBandwidthPreferenceKey"`, `"ThemeKeyThemeEnabled"`)
  for on-disk continuity. **[High]** (`CallServiceSettingsStore.swift:12-13`,
  `ThemeDataStore.swift:16`)
- **Only `CallServiceSettingsStore` broadcasts changes.** `ThemeDataStore` has no
  notification mechanism; any UI refresh on theme change must be driven by its
  callers. Whether such a mechanism exists is **intent undetermined — no evidence
  in source** within `SignalServiceKit/Stores/`. **[High]**
  (`ThemeDataStore.swift:50-52`)

## Related modules

- [`../Network/README.md`](../Network/README.md) — defines `NetworkInterfaceSet`,
  the type persisted by `CallServiceSettingsStore`. **[High]**
  (`NetworkInterfaceSet.swift:17`)
- `SignalServiceKit/Storage/Database/KeyValueStore.swift` — the key/value
  persistence primitive both stores wrap. **[High]** (`KeyValueStore.swift:8`)
- [`../Jobs/README.md`](../Jobs/README.md) — another `SignalServiceKit`
  subsystem; its `Cron` scheduler similarly uses a `KeyValueStore`-backed
  collection to persist small amounts of state, illustrating the same
  key/value-store pattern applied for a different purpose. **[Medium]**

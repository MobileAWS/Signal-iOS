# `Signal/` — Main Application Target

Deep documentation of the **first-party `Signal/` app target only** — process
entry, app launch, scene/window management, the app environment/dependency wiring,
and a high-level map of the application's view-controller layer.

This set deliberately excludes third-party pods (`Pods/`, `ThirdParty/`). It also
treats `SignalServiceKit/` and `SignalUI/` as **boundaries**: it documents *how the
app target uses them*, not their internals (those have their own docs under
[../SignalServiceKit/](../SignalServiceKit)). For the whole-repository directory map,
see [../APPLICATION_MAP.md](../APPLICATION_MAP.md).

## Scope

In scope (the files the task names, all under `Signal/AppLaunch/` unless noted):

- `AppDelegate.swift` — UIKit app-wide lifecycle entry.
- `SceneDelegate.swift` — UIKit scene/UI lifecycle entry.
- `SignalApp.swift` — top-level UI presentation controller / launch-interface switch.
- `AppEnvironment.swift` — main-app dependency container + launch jobs/cron wiring.
- `AppLifecycleManager.swift` — the launch state machine and lifecycle handler (~91 KB).
- `MainAppContext.swift` — the main-app `AppContext` implementation.
- `WindowManager.swift` — UIWindow stack management (root/call/screen-block/clock-skew).
- `LoadingViewController.swift` — the launch/loading screen.
- `LaunchJobs.swift` — one-shot pre-chat-list database jobs.
- `ViewControllerContext.swift` — a scoped dependency bundle for view controllers.

Plus a high-level map of `Signal/src/ViewControllers/` and sibling app subdirectories.

## Documents

| Doc | Covers |
|---|---|
| [launch-sequence.md](launch-sequence.md) | End-to-end launch: `AppDelegate` → `AppLifecycleManager.didFinishLaunching` → environment setup → database migration → launch-interface selection → UI presentation. Narrative + Mermaid. All launch-path error cases, edge cases, and feature flags. |
| [scene-window-management.md](scene-window-management.md) | `SceneDelegate`, scene connection/reconnection, `WindowManager`'s window stack and state machine, `LoadingViewController`. |
| [app-environment.md](app-environment.md) | `AppEnvironment`, `MainAppContext`, `ViewControllerContext`, `LaunchJobs`, and how the app target wires into `SignalServiceKit` (`SSKEnvironment`/`DependenciesBridge`/`AppSetup`) and `SignalUI` (`SUIEnvironment`). |
| [view-controllers-map.md](view-controllers-map.md) | High-level map of `Signal/src/ViewControllers/` (HomeView, AppSettings, ConversationView, etc.) and other first-party app subdirectories, without restating every view controller. |

## Conventions

- **Citations** are `path:line` (specific declaration) or `path` (directory/file).
  Line numbers reflect the working tree at authoring time and may drift; relocate via
  the symbol name. Every factual claim about behavior cites a path.
- **Confidence labels** on claims about *purpose/behavior*:
  - **[High]** — directly observed (file read / declaration located in this session).
  - **[Medium]** — inferred from signatures, names, and cross-file convention; not
    every referenced file was read in full.
  - **[Low]** — educated inference from naming alone.
- Any uncited claim is a **defect**.
- Where the source gives no evidence of design intent, the text says
  **"intent undetermined — no evidence in source."**
- Directory/file existence and sizes were produced by listing the working tree in this
  session **[High]**.

## One-paragraph orientation

The `Signal` target is a scene-based UIKit app. `AppDelegate` is the `@main` entry
(`Signal/AppLaunch/AppDelegate.swift:14`) and `SceneDelegate` handles UI-adjacent
events (`Signal/AppLaunch/SceneDelegate.swift:14`); both are thin and forward every
callback to the singleton `AppLifecycleManager.shared`
(`Signal/AppLaunch/AppLifecycleManager.swift:41`), which owns the launch state machine
and all lifecycle behavior **[High]**. The launch pipeline sets up two dependency
containers — `SSKEnvironment`/`DependenciesBridge` (service layer, via `AppSetup`) and
`AppEnvironment` (main-app objects, `Signal/AppLaunch/AppEnvironment.swift:9`) — plus
`SUIEnvironment` from `SignalUI`, then chooses a `LaunchInterface`
(`Signal/AppLaunch/SignalApp.swift:10`) and asks `SignalApp`
(`Signal/AppLaunch/SignalApp.swift:16`) to present it in a window managed by
`WindowManager` (`Signal/AppLaunch/WindowManager.swift:30`) **[High]**.

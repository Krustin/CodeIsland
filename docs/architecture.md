## Scope
System-level overview of CodeIsland: targets, processes, data flow, trust boundaries, threading, and persistence.

## Targets
Defined in `Package.swift` (~L13–L48):

- **CodeIslandCore** (`Sources/CodeIslandCore/`) — pure-Swift library shared by app and bridge. Holds models (`Models.swift`, `SessionSnapshot.swift`), wire helpers (`SocketPath.swift`, `EventNormalizer.swift`), the transcript watcher (`JSONLTailer.swift`), the ESP32 BLE protocol (`ESP32Protocol.swift`), the Warp pane mapping (`WarpPaneResolver.swift`), the Codex JSON-RPC client (`CodexAppServerClient.swift`), and `ChatMessageTextFormatter.swift`. No AppKit, no SwiftUI, no Yams/Sparkle.
- **CodeIsland** (`Sources/CodeIsland/`) — main app executable, ~50 files. Hosts the menu-bar item, the floating panel, all SwiftUI views, the [HookServer](hook-pipeline.md), [AppState](app-state.md), the [ConfigInstaller](hook-pipeline.md), Sparkle integration, and BLE bridging.
- **codeisland-bridge** (`Sources/CodeIslandBridge/main.swift`) — single-file executable shipped at `Contents/Helpers/codeisland-bridge`, installed at runtime to `~/.codeisland/codeisland-bridge`. Reads agent hook JSON on stdin, sends an enriched envelope over the Unix socket, prints the response on stdout for blocking events.

## Process model

Two cooperating executables:

1. **CodeIsland.app** — long-lived menu-bar GUI. Owns the SwiftUI panel, [AppState](app-state.md), and the listening end of the Unix socket via `HookServer` (`Sources/CodeIsland/HookServer.swift`).
2. **codeisland-bridge** — short-lived child invoked by the agent's hook command. One process per fired hook. Exits within milliseconds for non-blocking events; can stay alive for hours waiting on user approval for blocking events.

Why two processes (not in-process):
- Hooks must work even when the user has the app quit (the bridge silently no-ops if the socket is missing — `Sources/CodeIslandBridge/main.swift`, ~L226).
- The agent's hook contract is "exec a command, read stdout" — the bridge is the smallest thing that satisfies that contract while still talking native Swift to the app.
- Crashes in hook collection (bad JSON from a beta CLI) take down the bridge child, never the main app.

## Data flow

```
                 ┌────────────────────┐
                 │ AI agent (Claude,  │
                 │ Cursor, Codex, …)  │  fires lifecycle hook
                 └─────────┬──────────┘
                           │ exec ~/.codeisland/codeisland-hook.sh
                           ▼
                 ┌────────────────────┐
                 │ codeisland-hook.sh │  shell dispatcher
                 └─────────┬──────────┘
                           │ exec ~/.codeisland/codeisland-bridge
                           ▼
                 ┌────────────────────┐
                 │ codeisland-bridge  │  enriches JSON: _source, _ppid,
                 │  (short-lived)     │  _via_plugin, terminal env, ancestry
                 └─────────┬──────────┘
                           │ AF_UNIX SOCK_STREAM, JSON, shutdown(SHUT_WR)
                           │ /tmp/codeisland-<uid>.sock
                           ▼
        ┌──────────────────────────────────────┐
        │ HookServer (NWListener, @MainActor)  │
        │  routeKind → permission/question/event│
        └──────────────────┬───────────────────┘
                           │ async continuation (blocking events)
                           ▼
        ┌──────────────────────────────────────┐
        │ AppState (@MainActor @Observable)    │
        │  sessions, queues, surface           │
        └──────────────────┬───────────────────┘
                           │ @Observable triggers SwiftUI invalidation
                           ▼
        ┌──────────────────────────────────────┐
        │ NotchPanelView in non-activating     │
        │ NSPanel (PanelWindowController)      │
        └──────────────────────────────────────┘

   Optional fan-outs:
     AppState ──▶ ESP32BridgeManager ──▶ BLE ──▶ Buddy device
     AppState ──▶ SSHForwarder ──▶ remote host (tunneled socket)
     HookServer ──▶ user webhook (fire-and-forget HTTP POST)
```

## Trust boundaries

- **Unix socket permissions**: `HookServer.start` calls `umask(0o077)` *before* `NWListener` creates the socket file, then `chmod(0o700)` belt-and-suspenders once `.ready` fires (`Sources/CodeIsland/HookServer.swift`, ~L30 + ~L56). This closes the TOCTOU window where another local user could connect.
- **Payload size cap**: `HookServer.maxPayloadSize = 1_048_576` (1 MB) — oversized streams are dropped, not parsed (`Sources/CodeIsland/HookServer.swift`, ~L85, ~L103).
- **Sparkle updates**: pinned to `2.6.0+` for EdDSA (ed25519) signature verification (`Package.swift`, ~L8–L10). Public key is configured in `Info.plist`.
- **AppleEvents entitlement**: `com.apple.security.automation.apple-events` is granted (`CodeIsland.entitlements`) so the app can drive Terminal/iTerm/Warp via AppleScript when raising session windows. No other entitlements except Bluetooth for Buddy.
- **Not sandboxed**: deliberate — the app reads/writes agent config files in `$HOME` and opens a socket in `/tmp`.

## Threading model

- **Main actor**: all UI, all state mutation, all hook routing. `AppState`, `HookServer`, `PanelWindowController` are `@MainActor`.
- **DispatchSource (off-main, hops to main)**:
  - File watching: `JSONLTailer` (`Sources/CodeIslandCore/JSONLTailer.swift`, ~L114) uses `DispatchSource.makeFileSystemObjectSource` on a background queue then dispatches deltas via the constructor closure.
  - Process exit: `AppState` uses `DispatchSource.makeProcessSource(identifier:eventMask:.exit, queue:.main)` (`Sources/CodeIsland/AppState.swift`, ~L410).
  - FSEventStream: session discovery uses `FSEventStreamSetDispatchQueue(stream, .main)` (~L1992) so handlers run on main without a hop.
- **NWListener / NWConnection**: `HookServer` calls `listener?.start(queue: .main)` and `connection.start(queue: .main)` (`Sources/CodeIsland/HookServer.swift`, ~L66, ~L81). All connection callbacks are wrapped in `Task { @MainActor in ... }` for actor isolation.
- **Continuations**: blocking hooks (`PermissionRequest`, `Notification`-question) suspend on `withCheckedContinuation` — the bridge's socket waits on `recvAll` until `AppState` resumes the continuation with the user's decision (`Sources/CodeIsland/HookServer.swift`, ~L335–L348).

## Persistence

| Location                                  | Owner                | Contents |
|-------------------------------------------|----------------------|----------|
| `~/.codeisland/codeisland-bridge`         | `ConfigInstaller`    | Helper executable, copied from app bundle on launch |
| `~/.codeisland/codeisland-hook.sh`        | `ConfigInstaller`    | Shell dispatcher that execs the bridge |
| `~/.claude/settings.json`                 | `ConfigInstaller`    | Claude Code hooks block, JSON-merged |
| `$CODEX_HOME/hooks.json` (or `~/.codex/`) | `ConfigInstaller`    | Codex hooks (path resolved via `ConfigInstaller.codexHome()`, ~L123) |
| `~/.gemini/settings.json`                 | `ConfigInstaller`    | Gemini hooks |
| `~/.cursor/hooks.json`                    | `ConfigInstaller`    | Cursor hooks |
| `~/.trae/traecli.yaml`                    | `ConfigInstaller`    | Trae YAML hooks (uses Yams) |
| Other agents (Copilot, Qwen, Kimi, Qoder, Hermes, OpenCode, StepFun, Dex, Droid, Warp) | `ConfigInstaller` | See `builtInCLIs` table (~L142) and `customCLIConfigs` |
| `UserDefaults` (`SettingsKey.*`)          | `SettingsManager`    | User preferences: webhook URL, plugin session mode, panel sizing, mascot choice, auto-approve tools |
| `~/Library/Caches/com.codeisland/`        | macOS                | Sparkle download cache |
| `/tmp/codeisland-<uid>.sock`              | `HookServer`         | Per-user Unix socket (cleaned at start, removed 1s after stop) |
| `/tmp/codeisland-bridge.log`              | bridge               | Debug log, only when `CODEISLAND_DEBUG` is set |

The full per-agent install matrix lives in `ConfigInstaller.builtInCLIs` (`Sources/CodeIsland/ConfigInstaller.swift`, ~L142+).

## See also
- [glossary](glossary.md)
- [conventions](conventions.md)
- [hook pipeline](hook-pipeline.md)
- [app state](app-state.md)
- [ui panel](ui-panel.md)

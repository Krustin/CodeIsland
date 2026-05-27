# CodeIsland Docs

Agent-first reference for the CodeIsland macOS app. Optimized to be scanned by AI coding agents and humans alike: small focused files, real file:line references, no fluff.

If you're new here, read [architecture.md](architecture.md) first, then [glossary.md](glossary.md).

## Task → doc routing

| If you want to…                                            | Read                                                          |
| ---------------------------------------------------------- | ------------------------------------------------------------- |
| Understand the whole system end-to-end                     | [architecture.md](architecture.md)                            |
| Look up an unfamiliar term                                 | [glossary.md](glossary.md)                                    |
| Follow how an agent event becomes a UI surface             | [hook-pipeline.md](hook-pipeline.md)                          |
| Add a new agent (Claude/Cursor/…)                          | [agents.md](agents.md) + [hook-pipeline.md](hook-pipeline.md) |
| Change panel UI or add a new surface state                 | [app-state.md](app-state.md) + [ui-panel.md](ui-panel.md)     |
| Debug a missing hook event / permission flow               | [hook-pipeline.md](hook-pipeline.md) + [app-state.md](app-state.md) |
| Touch the BLE companion ("Buddy")                          | [esp32-buddy.md](esp32-buddy.md)                              |
| Add SSH support for a new remote scenario                  | [remote-sessions.md](remote-sessions.md)                      |
| Change how clicking a session focuses its terminal         | [terminal-jump.md](terminal-jump.md)                          |
| Build, sign, notarize, ship a release                      | [build-and-run.md](build-and-run.md)                          |
| Write or run tests                                         | [testing.md](testing.md)                                      |
| Follow the doc style / reference convention                | [conventions.md](conventions.md)                              |

## Repo layout

```
Package.swift             3 SPM targets (CodeIslandCore, CodeIsland, codeisland-bridge)
Sources/
  CodeIslandCore/         shared models, JSONLTailer, EventNormalizer,
                          ESP32Protocol, SocketPath, WarpPaneResolver,
                          CodexAppServerClient
  CodeIsland/             main app — AppState, HookServer, ConfigInstaller,
                          PanelWindowController, per-agent views, ESP32*,
                          SSHForwarder, RemoteManager, TerminalActivator
  CodeIslandBridge/       helper executable (Contents/Helpers/codeisland-bridge)
Tests/                    CodeIslandCoreTests + CodeIslandTests
android-watch/            Android Wear companion (Gradle, separate build)
hardware/                 ESP32 firmware
scripts/                  build helpers
build.sh                  universal swift build → app bundle → sign → (notarize)
CodeIsland.entitlements   AppleEvents + Bluetooth (app is NOT sandboxed)
Info.plist                bundle metadata + Sparkle EdDSA feed config
appcast.xml               Sparkle update feed
```

## Doc index

- [architecture.md](architecture.md) — targets, process model, data flow, trust boundaries, threading, persistence.
- [glossary.md](glossary.md) — Surface, Envelope, Bridge, Hook, Tailer, Pane, Buddy, Mascot, Session, Resolver.
- [conventions.md](conventions.md) — doc reference style + codebase conventions.
- [hook-pipeline.md](hook-pipeline.md) — agent config → `codeisland-hook.sh` → bridge → socket → `HookServer` → `AppState`.
- [app-state.md](app-state.md) — `AppState` (`@MainActor @Observable`), extension split, `IslandSurface` state machine, permission async flow, ToolUseCache, Codex Desktop integration.
- [ui-panel.md](ui-panel.md) — non-activating `NSPanel`, `NotchHostingView` re-entrancy workaround, notch detection, per-agent mascot views, status item.
- [agents.md](agents.md) — per-agent matrix (config path, transcript path, view file, quirks) + "add a new agent" checklist.
- [remote-sessions.md](remote-sessions.md) — SSH `-L` tunnel of `/tmp/codeisland.sock`, remote hook installer, exponential-backoff reconnect.
- [esp32-buddy.md](esp32-buddy.md) — BLE service/characteristic UUIDs, 20-byte frame format, marker bytes, uplink opcodes, pairing.
- [terminal-jump.md](terminal-jump.md) — bundle-ID dispatch, per-terminal AppleScript, `WarpPaneResolver` SQLite read-only.
- [build-and-run.md](build-and-run.md) — `build.sh` walkthrough, code-signing order, Sparkle config, notarization, troubleshooting.
- [testing.md](testing.md) — test targets, what's covered, what isn't, `swift test`.

## Doc conventions in one line

References use `Path/To/File.swift::symbolName (~Lline)` — path + symbol are authoritative, line is a hint that drifts. Full rules in [conventions.md](conventions.md).

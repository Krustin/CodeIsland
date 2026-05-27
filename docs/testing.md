# Testing

## Scope
What tests exist, how to run them, and what's deliberately not covered.

## Test targets
Declared in `Package.swift (~L35)`:

- `CodeIslandCoreTests` — depends on `CodeIslandCore` only. Pure-logic tests (no AppKit, no `@MainActor`).
- `CodeIslandTests` — depends on `CodeIsland` (the executable target) + `Yams`. Many tests are `@MainActor` because they exercise `AppState`.

Both targets live under `Tests/` and are run by `swift test`.

## Test files

### `Tests/CodeIslandCoreTests/`
- `CLIProcessResolverTests.swift` — `CLIProcessResolver.resolvedSessionPID` correctness, especially the Cursor sub-agent collapse (`#148`) where multiple parallel sub-agents must fold onto the same root PID.
- `ChatMessageTextFormatterTests.swift` — `ChatMessageTextFormatter.displayText` — verifies user messages stay literal (no markdown rendering) and assistant messages get formatted.
- `CodexAppServerClientTests.swift` — `CodexAppServerClient.drainMessages` framed-buffer parser for Codex's app-server JSON stream.
- `DerivedSessionStateTests.swift` — `AppState`-level derived state on `SessionSnapshot` (primary-source resolution when all sessions are idle, etc.).
- `ESP32ProtocolTests.swift` — Buddy wire contract: `MascotID` source folding (aliases like `traecli`, `factory`, `codybuddycn`), frame encoding, status mapping.
- `HookEventToolUseIdTests.swift` — JSON decoding of the `tool_use_id` field across flat snake_case, camelCase, and nested-tool-input shapes.
- `JSONLTailerTests.swift` — `JSONLTailer.scanLines` line splitter and trailing-fragment handling against synthetic and realistic Claude transcript fixtures (inline JSON in the test file).
- `PerformanceBenchmarks.swift` — micro-benchmarks with generous ceilings; guards against O(n²) regressions in `JSONLTailer.scanLines` and adjacent hot paths.
- `SessionSnapshotTitleTests.swift` — `SessionSnapshot.displayTitle` preference order (provider title → session id → fallback).
- `WarpPaneResolverTests.swift` — pure-logic `cwdVariants` (firmlink + trailing slash) and end-to-end SQLite query against an in-test sqlite fixture DB built with `SQLite3` directly.

### `Tests/CodeIslandTests/`
- `AppStateCodexAppServerTests.swift` — `AppState.applyCodexThreadStatus` flag → `AgentStatus` mapping (e.g. `waitingOnApproval` → `.waitingApproval`).
- `AppStateCodexTranscriptTests.swift` — extracting the latest terminal turn timestamp from a Codex JSONL transcript.
- `AppStatePermissionFlowTests.swift` — async permission request/response: approve, deny, dismiss, multi-session ordering.
- `AppStatePrimarySourceTests.swift` — `#149` regression: when all sessions are idle, primary source must honor user default rather than echoing the last speaker.
- `AppStateQuestionFlowTests.swift` — `AskUserQuestion` multi-question answer routing and respond paths.
- `AppStateToolUseCacheTests.swift` — pre/post-tool-use record caching lifecycle in `AppState`.
- `CodexHomeTests.swift` — `$CODEX_HOME` env-var resolution with set/unset/saved/restored fixtures.
- `ConfigInstallerTests.swift` — YAML + JSON config merging logic in `ConfigInstaller` (uses `Yams` for round-trips); operates on in-memory strings, not the real filesystem.
- `HookServerCwdFilterTests.swift` — `HookServer.cwdMatchesAnyPattern` substring blocklist matching for Settings → Behavior → "Ignore Hooks From Paths" (`#125`).
- `JSONMinimalEditorTests.swift` — `setTopLevelValue` minimal-edit invariants: preserves unrelated keys, ordering, comments, slashes.
- `KiroSupportTests.swift` — Kiro CLI wire-level integration (`#127`): `EventNormalizer` mappings, default event list, supported-source recognition, `.kiroAgent` `HookFormat`.
- `L10nTests.swift` — locks the per-language key sets (Turkish vs English etc.) to keep translations exhaustive.
- `NotchPanelViewTests.swift` — pure helpers extracted from the view: `shouldTriggerJumpFailureFeedback`, `JumpAnimationHelper.shakeSequence`.
- `PanelWindowControllerTests.swift` — pure helpers extracted from the controller: `screenHopMotion` timing constants.
- `RemoteManagerTests.swift` — `RemoteManager.reconnectDelay(attempt:)` backoff table (5, 15, 45, 120, 300, clamped at 300).
- `ScreenDetectorTests.swift` — `ScreenDetector.autoPreferredIndex` ordering across notch / non-notch / external displays.
- `SessionPersistenceTests.swift` — codable round-trips for `PersistedSession`, including backward compatibility (missing `cliStartTime` etc.).
- `SessionTitleStoreTests.swift` — `SessionTitleStore.codexThreadName` JSONL index lookup, picking the latest matching `updated_at`.

## Running tests
```sh
swift test                                 # all targets
swift test --filter WarpPaneResolverTests  # one test class
swift test --filter testReconnectDelay     # one test method (substring)
```

Filters use `<TestClass>` or `<TestClass>/<testMethod>` substring matching across both targets.

## Coverage gaps
Deliberately **not** covered by automated tests — don't waste time looking for them:

- SwiftUI views (NotchPanelView, SettingsView, etc.) beyond pure helper extraction.
- `NSPanel` / `NSWindow` placement code beyond extracted timing constants.
- Buddy / ESP32 BLE hardware path (`ESP32BridgeManager`, `ESP32StatePublisher`) — CoreBluetooth has no test double here. Only the on-wire protocol (`ESP32ProtocolTests`) is locked down.
- Sparkle auto-update flow.
- `ConfigInstaller` filesystem writes — only the YAML/JSON manipulation logic is tested, not the actual on-disk install.
- SSH forwarding / remote-session transport (see `docs/remote-sessions.md`). `RemoteManagerTests` covers only the backoff table.
- AppleScript / `osascript` execution in `TerminalActivator` and `TerminalVisibilityDetector` — they spawn `/usr/bin/osascript` and are not exercised by `swift test`.
- Real Warp / iTerm2 / Ghostty / kitty IPC.

## Fixture conventions
- No `fixtures/` directories on disk. All fixtures are **inline** in the test files — triple-quoted JSONL strings, `Data()` literals, hand-built `SessionSnapshot` values.
- `WarpPaneResolverTests.swift` builds a temporary on-disk SQLite database in `setUp` using the system `SQLite3` C API, then points `WarpPaneResolver(sqlitePath:)` at it.
- `JSONLTailerTests.swift` and `PerformanceBenchmarks.swift` synthesize realistic Claude transcript line mixes inline (`assistantLine(text:)` helpers etc.).
- `CodexHomeTests.swift` saves/restores the real `$CODEX_HOME` env var in `setUp`/`tearDown`.

## See also
- [architecture.md](./architecture.md)
- [conventions.md](./conventions.md)

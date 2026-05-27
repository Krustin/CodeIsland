## Scope
How these docs are written and the codebase conventions an agent needs to read/edit CodeIsland safely.

## Doc conventions

### Code references
- Authoritative form: `Sources/<target>/<File>.swift::SymbolName (~Lline)`.
- Path + symbol name are the durable identifier; the line number is a hint that drifts. Always re-locate by symbol name when editing.
- For nested or extension symbols use `Type.method`, e.g. `AppState.handlePermissionRequest`.
- Bare paths like `Sources/CodeIsland/HookServer.swift` are fine when referring to the whole file.

### When to embed code
Embed only:
- Wire formats (the JSON envelope schema, response shapes).
- Regex / magic numbers (`0o077`, `1_048_576`, alarm seconds).
- Short enum or state-machine declarations that are the doc's subject.

Otherwise link with `[symbol](path-to-doc.md#anchor)` or a path reference and trust the reader to open it.

### Doc structure
Every doc in `docs/` starts with:

```
## Scope
<one sentence>
```

…and ends with:

```
## See also
- [other doc](other.md)
```

Keep files in the 100–400 line range. Headings + bullets beat prose paragraphs.

## Codebase conventions

### Concurrency
- All UI- and state-mutating code is `@MainActor`. `AppState` is declared `@MainActor @Observable final class` (`Sources/CodeIsland/AppState.swift::AppState`, ~L15). `HookServer` is `@MainActor` (`Sources/CodeIsland/HookServer.swift::HookServer`, ~L8). `PanelWindowController` is `@MainActor` (`Sources/CodeIsland/PanelWindowController.swift::PanelWindowController`, ~L84).
- Background work (filesystem watching, process exit signals) runs on `DispatchSource` and hops to main with `Task { @MainActor in ... }` before touching state. See `JSONLTailer` (`Sources/CodeIslandCore/JSONLTailer.swift`, ~L33) and `AppState.processMonitors` using `DispatchSource.makeProcessSource(...)` (~L410).
- Async hook handlers use `withCheckedContinuation` to bridge NWConnection callbacks to `await`-style code in `HookServer.processRequest` (~L335–L348).

### Filesystem watching
- Use `DispatchSource.makeFileSystemObjectSource` for file deltas (transcripts).
- Use `FSEventStreamCreate` for directory roots (session discovery — `AppState`, ~L1977).
- Both are dispatched to `.main` so handlers can mutate `@MainActor` state without an explicit hop.

### Dependencies
Only two third-party packages in `Package.swift` (~L7–L12):
- Sparkle 2.6+ — auto-updater with EdDSA signature verification.
- Yams 5.0+ — YAML parsing for agents that ship YAML configs (e.g. Trae).

No other SPM dependencies. Avoid adding any.

### Logging
- Use `os.Logger` for app-side code: `private let log = Logger(subsystem: "com.codeisland", category: "AppState")` is the standard pattern (`Sources/CodeIsland/AppState.swift`, ~L8; `Sources/CodeIsland/HookServer.swift`, ~L6; `Sources/CodeIsland/PanelWindowController.swift`, ~L5).
- The bridge uses a hand-rolled `debugLog` that appends to `/tmp/codeisland-bridge.log` only when `CODEISLAND_DEBUG` is set (`Sources/CodeIslandBridge/main.swift::debugLog`, ~L101). It avoids `os.Logger` because the bridge runs as a short-lived hook child where unified-logging setup cost matters.
- Plain `print` is used in a handful of non-hot paths (debug harness, settings) — prefer `Logger` for new code.

### Targets
Three SPM targets (`Package.swift`, ~L13–L34):
- `CodeIslandCore` — pure-Swift, no AppKit/SwiftUI. Models, normalizers, tailers, resolvers, socket-path constant. Linkable from both app and bridge.
- `CodeIsland` — the executable app. SwiftUI + AppKit. Imports `CodeIslandCore`, `Sparkle`, `Yams`.
- `codeisland-bridge` — single-file executable. Imports `CodeIslandCore` only.

### File-level patterns
- `AppState.swift` is intentionally split into extensions in sibling files (`AppState+CodexAppServer.swift`, `AppState+ToolUseCache.swift`, `AppState+TranscriptTailer.swift`). New domain logic that mutates `AppState` should be added as another `AppState+*.swift` extension rather than swelling the main file.
- Per-agent SwiftUI mascots follow the shape `struct <Agent>View: View { let status: AgentStatus; var size: CGFloat = 27 }`. See [ui panel](ui-panel.md).

### Sandboxing
The app is **not** sandboxed (see `CodeIsland.entitlements` — only `com.apple.security.automation.apple-events` and `com.apple.security.device.bluetooth` are present, no `com.apple.security.app-sandbox`). This is required because the app installs files into agent config directories under `$HOME` and opens a Unix socket in `/tmp`.

## See also
- [glossary](glossary.md)
- [architecture](architecture.md)

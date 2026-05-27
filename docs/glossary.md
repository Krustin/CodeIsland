## Scope
One-paragraph definitions for terms that appear across the CodeIsland codebase. Cross-link to the canonical definition site for each term.

## Terms

### Surface
The single visible "face" of the floating panel at any moment. Modeled by the `IslandSurface` enum (`Sources/CodeIsland/IslandSurface.swift::IslandSurface`) with cases `collapsed`, `sessionList`, `approvalCard(sessionId:)`, `questionCard(sessionId:)`, `completionCard(sessionId:)`. The surface is mutually exclusive — only one card is shown at a time. The current surface lives on `AppState.surface` (`Sources/CodeIsland/AppState.swift::AppState.surface`, ~L101) and drives the switch in `NotchPanelView` (~L110).

### Envelope
The JSON payload sent over the Unix socket from the bridge to `HookServer`. It is the agent's raw hook JSON, augmented in-place with bridge-injected fields prefixed by underscore: `_source` (agent name, e.g. `claude`/`cursor`/`codex`), `_ppid` (resolved CLI process PID), `_hook_ppid` (the immediate hook caller when different), `_via_plugin` (true when the source was inferred from ancestry rather than `--source`), `_term_app`, `_term_bundle`, `_iterm_session`, `_kitty_window`, `_tmux`, `_tmux_pane`, `_tmux_client_tty`, `_tty`, `_cmux_surface_id`, `_cmux_workspace_id`. Built in `Sources/CodeIslandBridge/main.swift` (~L286–L412).

### Bridge
The `codeisland-bridge` helper executable (separate SPM target, `Sources/CodeIslandBridge/main.swift`). Shipped inside the app bundle at `Contents/Helpers/`, installed to `~/.codeisland/codeisland-bridge` (`Sources/CodeIsland/ConfigInstaller.swift::ConfigInstaller.bridgePath`, ~L107). It reads a hook event on stdin, enriches it with environment + ancestry data, sends the [envelope](#envelope) to the [HookServer](architecture.md) over a Unix socket, then forwards the server's response on stdout (for blocking events).

### Hook
A shell command installed into a coding agent's per-tool config so that the agent invokes it on lifecycle events (`PreToolUse`, `PostToolUse`, `PermissionRequest`, `Stop`, etc.). The installed command is `~/.codeisland/codeisland-hook.sh` (`Sources/CodeIsland/ConfigInstaller.swift::ConfigInstaller.hookCommand`, ~L109), which is a thin dispatcher that execs the bridge. Per-agent installation lives in `ConfigInstaller` (e.g. `builtInCLIs` table, ~L142).

### Tailer
`JSONLTailer` (`Sources/CodeIslandCore/JSONLTailer.swift::JSONLTailer`, ~L33). A `DispatchSourceFileSystemObject`-based watcher that tails JSONL transcript files (e.g. Claude Code session transcripts), emitting only the newly appended lines as `TranscriptDelta` values. AppState uses one shared instance via `AppState.transcriptTailer` (`Sources/CodeIsland/AppState.swift`, ~L82) and routes deltas through the `+TranscriptTailer` extension.

### Pane
A terminal pane — the unit of "where this agent process is running." For Warp specifically, `WarpPaneResolver` (`Sources/CodeIslandCore/WarpPaneResolver.swift::WarpPaneResolver`) maps an agent PID to the Warp tab/pane that hosts it so we can raise the right window on click. Pane identity is also carried in the [envelope](#envelope) for tmux (`_tmux_pane`), iTerm (`_iterm_session`), Kitty (`_kitty_window`), and cmux (`_cmux_surface_id`).

### Buddy
The optional ESP32 BLE companion device. Code lives in `Sources/CodeIsland/ESP32BridgeManager.swift`, `ESP32FocusCoordinator.swift`, `ESP32StatePublisher.swift` and `Sources/CodeIslandCore/ESP32Protocol.swift`. The Buddy displays mascot state on a tiny external screen and can fire approve/deny via physical buttons (see `AppState.handleBuddyControlCommand`, `Sources/CodeIsland/AppState.swift` ~L1113).

### Mascot
The per-agent character icon rendered in the notch. Each supported agent has a `*View.swift` in `Sources/CodeIsland/` (`CursorView.swift`, `CopilotView.swift`, `GeminiView.swift`, `KimiView.swift`, `QwenView.swift`, `TraeView.swift`, `QoderView.swift`, `HermesView.swift`, `OpenCodeView.swift`, `StepFunView.swift`, `DexView.swift`, `DroidView.swift`). They share the shape `struct XView: View { let status: AgentStatus; var size: CGFloat = 27 }`. The neutral default mascot lives in `MascotView.swift`.

### Session
One agent conversation, keyed by `sessionId`. Stored in `AppState.sessions: [String: SessionSnapshot]` (`Sources/CodeIsland/AppState.swift`, ~L38). The `SessionSnapshot` model is defined in `Sources/CodeIslandCore/SessionSnapshot.swift`. The bridge guarantees every event carries a non-empty `session_id`, falling back to a synthesized `<source>-ppid-<rootPid>` when the agent doesn't expose one (`Sources/CodeIslandBridge/main.swift`, ~L327–L342).

### Resolver
A small helper class that maps from an opaque external identifier to a UI-actionable target. `WarpPaneResolver` (`Sources/CodeIslandCore/WarpPaneResolver.swift`) maps PID → Warp pane. `CLIProcessResolver` (referenced in the bridge, ~L299–L321; defined in `Sources/CodeIslandCore/`) maps process ancestry → canonical agent source + tracked PID + session PID. `EventNormalizer` (`Sources/CodeIslandCore/EventNormalizer.swift`) maps per-agent event names to a normalized vocabulary.

## See also
- [conventions](conventions.md)
- [architecture](architecture.md)
- [hook pipeline](hook-pipeline.md)
- [app state](app-state.md)
- [ui panel](ui-panel.md)

# Per-agent integrations

The matrix that says, for every supported AI coding agent, **where its config lives, where its transcript lives, and which Swift file draws its mascot**. If you are adding or fixing a single agent, start here.

## Scope

- One row per agent CLI/IDE that CodeIsland recognizes today.
- Pointers into source — not a tutorial. See [hook-pipeline.md](hook-pipeline.md) for how config patches turn into events, and [ui-panel.md](ui-panel.md) for how the per-agent View is mounted.

## Quick orientation

- Source-of-truth for the agent list is `builtInCLIs` in `Sources/CodeIsland/ConfigInstaller.swift:142–345`.
- Source-of-truth for mascot routing is `MascotView` at `Sources/CodeIsland/MascotView.swift:24–62`.
- Source-of-truth for the Buddy mascot slot mapping is `MascotID` at `Sources/CodeIslandCore/ESP32Protocol.swift:131–199`.
- Per-agent transcript discovery sits in `AppState.swift` (search for `~/.<agent>/`); see [hook-pipeline.md](hook-pipeline.md) for the upstream events and [glossary.md](glossary.md) for terminology.

## Hook formats

Agents differ in how `hooks` get serialized — encoded as the `HookFormat` enum used by `CLIConfig` (see `ConfigInstaller.swift:1–110` for the enum). Same field name, different shape:

| Format       | Where                                  | Shape                                  |
|--------------|----------------------------------------|----------------------------------------|
| `.claude`    | Claude Code, all forks, Qwen           | nested with `matcher`/`hooks` arrays   |
| `.nested`    | Codex, Gemini                           | event → array of entries with `hooks`  |
| `.flat`      | Cursor, Trae, Trae CN                   | event → array of `{type,command}`      |
| `.traecli`   | TraeCli (`~/.trae/traecli.yaml`)        | YAML list with `matchers:`             |
| `.copilot`   | GitHub Copilot CLI                      | flat-but-special; written to a sidecar file |
| `.kimi`      | Kimi Code CLI (`~/.kimi/config.toml`)   | TOML `[[hooks]]` arrays                |
| `.kiroAgent` | Kiro CLI (`~/.kiro/agents/codeisland.json`) | per-agent JSON with `timeout_ms`   |

## Built-in agent matrix

| Agent | Config file | Transcript path / glob | Per-agent mascot view | Notes |
|---|---|---|---|---|
| Claude Code | `~/.claude/settings.json` | `~/.claude/projects/<encoded-cwd>/<sessionId>.jsonl` (`AppState.swift:1540`, `:1909`) | default `ClawdView` (`MascotView.swift:60`) | hook script + bridge dispatcher; legacy `~/.claude/hooks/*` cleaned at install (`ConfigInstaller.swift:570`). `PostToolUseFailure` requires Claude ≥ 2.1.89 (`:163`). |
| Codex | `$CODEX_HOME/hooks.json` (default `~/.codex/hooks.json`) | `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl` (`AppState.swift:1618`, `:3380–3492`) | `DexView.swift` | Requires `[features] codex_hooks = true` in `config.toml` (`ConfigInstaller.swift:1576–1614`). `$CODEX_HOME` resolution at `:123–138`. Also supports the Codex APP server — see `AppState+CodexAppServer.swift` and `CodexAppServerClient.swift`. |
| Gemini | `~/.gemini/settings.json` | `~/.gemini/tmp/<id>/...` resolved via `~/.gemini/projects.json` (`AppState.swift:1748–1752`, `:2823–2826`) | `GeminiView.swift` | Timeouts in **milliseconds**, not seconds (`ConfigInstaller.swift:192–199`). Event names PascalCase `BeforeTool`/`AfterTool` etc. (normalized in `EventNormalizer.swift:18–22`). |
| Cursor | `~/.cursor/hooks.json` | `~/.cursor/projects/<encoded-cwd>/agent-transcripts` (`AppState.swift:1768`, `:2985`) | `CursorView.swift` | camelCase event names — `EventNormalizer.swift:7–17`. |
| Copilot (GitHub CLI) | `~/.copilot/hooks/codeisland.json` | `~/.copilot/session-state` (`AppState.swift:1779`, `:3043`) | `CopilotView.swift` | Sidecar file (root `~/.copilot` must already exist; not auto-created — `ConfigInstaller.swift:961–963`). Event names from `EventNormalizer.swift:23–29`. |
| Qwen Code | `~/.qwen/settings.json` | n/a (hook-driven only) | `QwenView.swift` | Claude-format but **timeouts in ms** (`ConfigInstaller.swift:296–315`). |
| Kimi Code CLI | `~/.kimi/config.toml` | n/a | `KimiView.swift` | TOML format. Max timeout 600s. **No** `PermissionRequest` event. Install logic at `ConfigInstaller.swift:1615–1737`. Root only created if it already exists (`:945–947`). |
| Trae | `~/.trae/hooks.json` | n/a | `TraeView.swift` | `.flat` format. |
| Trae CN | `~/.trae-cn/hooks.json` | n/a | `TraeView.swift` (shared) | `.flat` format. |
| TraeCli | `~/.trae/traecli.yaml` | n/a | `TraeView.swift` (shared) | YAML managed-block merge (`ConfigInstaller.swift:1057–1568`). The merger is intentionally indentation-tolerant; see the long YAML normalizer in `RemoteInstaller.swift:164–410` (mirrored Python implementation). |
| Qoder | `~/.qoder/settings.json` | n/a | `QoderView.swift` | Claude Code fork — Claude format. |
| Factory (Droid) | `~/.factory/settings.json` | n/a | `DroidView.swift` | Source identifier is **`droid`** (not `factory`) — `ConfigInstaller.swift:247–253`. The desktop app bundle id `com.factory.app` is in `TerminalActivator.sourceToNativeAppBundleId` at `:31`. |
| CodeBuddy | `~/.codebuddy/settings.json` | n/a | `BuddyView.swift` | Claude Code fork. |
| CodyBuddyCN | `~/.codybuddycn/settings.json` | n/a | `BuddyView.swift` (shared) | China variant. |
| StepFun | `~/.stepfun/settings.json` | n/a | `StepFunView.swift` | Claude Code fork. |
| AntiGravity | `~/.antigravity/settings.json` | n/a | `AntiGravityView.swift` | Claude Code fork. |
| WorkBuddy | `~/.workbuddy/settings.json` | n/a | `WorkBuddyView.swift` | Claude Code fork. |
| Hermes | `~/.hermes/settings.json` | n/a | `HermesView.swift` | Claude Code fork. |
| OpenCode | `~/.config/opencode/opencode.jsonc` (preferred) or `opencode.json` | n/a | `OpenCodeView.swift` | **Plugin model**, not hooks: `codeisland-opencode.js` is dropped into `~/.codeisland/` and referenced from the user config (`ConfigInstaller.swift:1907–2098`). `.jsonc` wins when both exist (issue #132). |
| Kiro | `~/.kiro/agents/codeisland.json` | n/a | (no dedicated view; default `ClawdView`) | Hooks only fire when launched as `kiro --agent codeisland` (#127). Format `.kiroAgent`; locked-in by `Tests/CodeIslandTests/KiroSupportTests.swift`. |
| Warp | n/a (no hooks) | n/a | n/a | Click-to-focus only — pane resolved from Warp's SQLite. See [terminal-jump.md](terminal-jump.md). |
| Dex | n/a | n/a | `DexView.swift` is **shared with Codex**, not a separate agent | "Dex" is internal naming for the Codex mascot view file. |

### Custom CLIs

User-defined entries are stored as `CustomCLIConfig` in `UserDefaults` and merged at runtime by `customCLIs()` (`ConfigInstaller.swift:446–471`). They share the built-in install/uninstall plumbing — only built-in agents get a per-agent `*View.swift`.

## Buddy mascot slot table

The ESP32 firmware only has 16 slots. Source aliases collapse to one slot via `MascotID(sourceName:)` at `Sources/CodeIslandCore/ESP32Protocol.swift:175–199`. Notable folds:

- `cursor` and `cursor-cli` → `.cursor`
- `trae`, `traecn`, `traecli` → `.trae`
- `qoder`, `qoder-cli` → `.qoder`
- `codebuddy`, `codybuddycn` → `.codebuddy`
- `droid` → `.droid` (Factory). `factory` and `ag` are accepted as aliases via `SessionSnapshot.normalizedSupportedSource`.
- Unrecognized sources have **no Buddy slot** — they show in the panel but not on Buddy.

## Adding a new agent — checklist

For a hook-driven CLI roughly modeled on Claude Code:

1. **Register in ConfigInstaller** — append a `CLIConfig` to `builtInCLIs` (`Sources/CodeIsland/ConfigInstaller.swift:142–345`). Pick the right `HookFormat`. Provide an `events` list (or use `defaultEvents(for:)`).
2. **Per-agent view** — add `Sources/CodeIsland/<Name>View.swift` modeled on a sibling (e.g. `QoderView.swift`). Must accept `(status: AgentStatus, size: CGFloat)`.
3. **Register the view** — add a `case "<source>":` to the switch in `Sources/CodeIsland/MascotView.swift:24–62`.
4. **Mascot icon** — drop `<source>.png` in `Sources/CodeIsland/Resources/cli-icons/`. The set today is `{antigravity, claude, codebuddy, codex, copilot, cursor, factory, gemini, hermes, kimi, opencode, pi, qoder, qwen, stepfun, trae, workbuddy}.png`.
5. **Buddy slot** (optional) — only if you want LCD support: claim a free slot in `MascotID` (`Sources/CodeIslandCore/ESP32Protocol.swift:131–148`), update `sourceName` (`:151–170`), wire alias folds in `init?(sourceName:)` (`:175–199`), and add a `mascot_<name>.h` file in `hardware/`. Slots 0–15 are full today.
6. **Event name normalization** — if the CLI emits non-canonical event names, add a case to `EventNormalizer.normalize` (`Sources/CodeIslandCore/EventNormalizer.swift:5–48`).
7. **Transcript discovery** (optional) — if you want chat history / titles, follow the existing patterns in `AppState.swift` (search `~/.<agent>/`); typical entry points are the source-keyed dictionaries at `:1909–1916`.
8. **Native app jump** (optional) — if there's a desktop app counterpart, add bundle ids to `TerminalActivator.sourceToNativeAppBundleId` and `nativeAppBundles` in `Sources/CodeIsland/TerminalActivator.swift:25–53`.
9. **Remote install** (optional) — if you also want this agent installed over SSH, add an `install_<name>()` to the Python script in `RemoteInstaller.swift:79–522` and append it to the `parts = […]` list at `:520`.
10. **Tests** — add a fixture to `Tests/CodeIslandTests/ConfigInstallerTests.swift` proving install/uninstall round-trips. See [testing.md](testing.md).

## Cross-references

- The hook → JSON envelope → AppState path: [hook-pipeline.md](hook-pipeline.md).
- AppState session shape, status derivation: [app-state.md](app-state.md).
- Click-to-focus dispatch (per-source bundle ids, AppleScript activators): [terminal-jump.md](terminal-jump.md).
- Buddy wire format: [esp32-buddy.md](esp32-buddy.md).
- Terminology (`source`, `MascotID`, `transcriptPath`): [glossary.md](glossary.md).
- Doc style: [conventions.md](conventions.md).

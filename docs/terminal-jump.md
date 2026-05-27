# Terminal Jump

## Scope
Clicking a session card in the notch panel focuses the exact terminal window/tab/pane (or IDE window) where that agent is running.

## Dispatch order
Entry point: `Sources/CodeIsland/TerminalActivator.swift::TerminalActivator.activate (~L55)`.

Routing is resolved in this order — earlier matches short-circuit later ones:

1. **Remote sessions** — skipped entirely (`session.isRemote` guard).
2. **Native desktop app by bundle ID** — `session.termBundleId` ∈ `nativeAppBundles` (Codex APP, Cursor, Trae, Qoder, Factory, CodeBuddy, CodyBuddyCN, StepFun, OpenCode, WorkBuddy). Goes directly to `activateByBundleId (~L802)`.
3. **IDE integrated terminal** — `session.isIDETerminal` true: `activateIDEWindow (~L711)` picks the window whose title contains the project folder name via System Events.
4. **Native app fallback when `termBundleId == nil`** — if the source has a known desktop app (`sourceToNativeAppBundleId (~L25)`) that's running, prefer it over the inherited `TERM_PROGRAM` env (avoids OpenCode-in-Ghostty jumping to the wrong app).
5. **Terminal resolution** — bundle ID (most accurate) → `TERM_PROGRAM` → `detectRunningTerminal()` scan.
6. **tmux pre-focus** — if `session.tmuxPane` set, `activateTmux (~L698)` runs `tmux select-window` then `select-pane`. The activator then continues with the outer terminal (using `tmuxClientTty` instead of the inner pty for tab matching).
7. **Per-terminal activation** — see next section.

## Supported terminals
From `knownTerminals (~L8)`:

| Name | Bundle ID | Activator (in `TerminalActivator.swift`) |
|------|-----------|------------------------------------------|
| cmux | `com.cmuxterm.app` | `activateCmux (~L927)` — `cmux focus-panel --panel <surface>` (+ `--workspace`). |
| Ghostty | `com.mitchellh.ghostty` | `activateGhostty (~L197)` — AppleScript: tmux title → session-ID-in-title → CWD → folder/tilde fallback → System Events de-miniaturize. |
| iTerm2 | `com.googlecode.iterm2` | `activateITerm (~L491)` by `unique ID`, else `activateITermByTtyOrCwd (~L430)`. |
| WezTerm | `com.github.wez.wezterm` | `activateWezTerm (~L647)` — `wezterm cli list --format json` then `cli activate-tab --tab-id`. |
| kitty | `net.kovidgoyal.kitty` | `activateKitty (~L675)` — `kitten @ focus-window --match id:` or `focus-tab --match cwd:`/`title:`. |
| Alacritty | `org.alacritty` | Falls to `activateTerminalWindow (~L769)` — System Events AXRaise by folder-in-title. |
| Warp | `dev.warp.Warp-Stable` | `activateWarp (~L959)` — see SQLite section below. |
| Terminal.app | `com.apple.Terminal` | `activateTerminalApp (~L524)` — AppleScript: tty → tab-name folder → custom-title folder → unminimize fallback. |
| Hyper / Tabby / Rio / other | (varies) | Generic `activateTerminalWindow` or `bringToFront (~L817)`. |

Tab-level precision (per `TerminalVisibilityDetector`'s comment): iTerm2, Ghostty, Terminal.app, WezTerm, kitty. App-level only: Alacritty, Warp (tab keystroke after SQLite resolve), Hyper, Tabby, Rio.

## Warp pane resolver
`Sources/CodeIslandCore/WarpPaneResolver.swift::WarpPaneResolver`.

- **DB path** (`defaultSQLitePath (~L59)`): `~/Library/Group Containers/2BBY89MBSN.dev.warp/Library/Application Support/dev.warp.Warp-Stable/warp.sqlite`.
- **Open mode** (`fileURI (~L199)` + `performQuery (~L106)`): `file://<percent-encoded-path>?mode=ro&nolock=1`, opened with `SQLITE_OPEN_READONLY | SQLITE_OPEN_URI | SQLITE_OPEN_NOMUTEX`. `sqlite3_busy_timeout = 150ms` as legacy insurance. The `nolock=1` URI param means CodeIsland never contends with Warp's own writer.
- **Firmlink normalization** (`cwdVariants (~L81)`): for every input cwd, the resolver derives `{trimmed, with/without trailing slash}` and then for each seed adds the firmlink counterpart — `/private/<x>` ↔ `<x>` for `/tmp`, `/var`, `/etc`. All variants are passed as bound parameters in one `IN (?, ?, ...)` clause.
- **Ranking SQL** (`performQuery (~L123)`):
  ```
  ORDER BY tp.is_active DESC, focused DESC, tp.id DESC
  ```
  i.e. pane currently active → pane currently focused → newest pane by row id.
- **Hierarchy join**: `terminal_panes` ⨝ `pane_leaves` (`is_focused`) ⨝ `pane_nodes` (`tab_id`) ⨝ `tabs` (`window_id`) ⨝ `windows` (`active_tab_index`), plus a subquery counting prior tabs in the window to produce `tabIndexInWindow`. `isActiveTab = (window.active_tab_index == tab_idx)`.

`activateWarp (~L959)` consumes the best `WarpPaneMatch`. If `!isActiveTab` and `tabIndexInWindow + 1 ∈ 1...9`, it posts `Cmd+<digit>` via `sendWarpGoToTab (~L997)` using `CGEvent` (requires Accessibility permission; silently degrades to plain app activation otherwise). Tabs ≥ 10 are not supported.

## Terminal visibility detection
`Sources/CodeIsland/TerminalVisibilityDetector.swift::TerminalVisibilityDetector` powers smart-suppress (don't notify when the user is already looking at the session).

- `isTerminalFrontmostForSession (~L27)`: cheap, main-thread safe, bundle-ID-first match against `NSWorkspace.frontmostApplication`. Falls back to `TERM_PROGRAM` substring match only when bundle ID is absent (avoids Warp's `TERM_PROGRAM=Apple_Terminal` false positive).
- `isSessionTabVisible (~L55)`: full tab-precision check — **background thread only** (AppleScript / CLI may block 50–200ms). Routes by bundle ID:
  - Native app mode → frontmost is sufficient.
  - IDE terminal → frontmost is sufficient (can't query embedded tool-window focus without Accessibility).
  - tmux pane → `isTmuxPaneActive (~L244)` via `tmux display-message`.
  - iTerm2 → `unique ID` of `current session of current tab of current window`.
  - Ghostty → System Events front-window title contains both folder name AND source keyword.
  - Terminal.app → `tty of selected tab of front window`.
  - WezTerm → `wezterm cli list` JSON, find `is_active == true` pane, match by `tty_name` or `cwd`.
  - kitty → `kitten @ ls`, walk `is_focused` chain and match by window id.
  - Other → returns false (prefer showing notification).

Called by `NotchPanelView` / `PanelWindowController` with a 5s `ProcessRunner` cap to keep the UI from stuttering.

## See also
- [agents.md](./agents.md)
- [architecture.md](./architecture.md)
- [glossary.md](./glossary.md)

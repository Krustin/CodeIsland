## Scope
End-to-end trace of one hook event: from the agent firing a lifecycle hook to [AppState](app-state.md) routing it into the UI.

## Stage 1 — ConfigInstaller patches the agent's config

`Sources/CodeIsland/ConfigInstaller.swift::ConfigInstaller` (~L105) maintains a `builtInCLIs: [CLIConfig]` table (~L142+) listing every supported agent: name, source tag, config path (relative to `$HOME`), config-file format (`.claude` / `.nested` / `.flat`), and the list of `(eventName, timeout, captureOutput)` tuples.

For each enabled agent, the installer merges a hook entry of the form:

```json
{
  "type": "command",
  "command": "~/.codeisland/codeisland-hook.sh",
  "timeout": 5
}
```

…into the agent's config under the agent-specific key (e.g. `hooks.PreToolUse[].hooks[]` for Claude). The exact insertion code is `Sources/CodeIsland/ConfigInstaller.swift` ~L894–L910.

Every agent points at the **same** `~/.codeisland/codeisland-hook.sh`. Per-agent disambiguation happens later through `--source <name>` arguments and process-ancestry inference in the bridge.

## Stage 2 — codeisland-hook.sh dispatches to the bridge

`~/.codeisland/codeisland-hook.sh` (`ConfigInstaller.hookScriptPath`, ~L108) is a tiny shell wrapper that:

1. Forwards stdin unchanged.
2. Execs `~/.codeisland/codeisland-bridge` with any agent-specific `--source` / `--event` flags appended.

It exists so we can ship updates to the dispatcher (PATH ordering, debug toggles) without rewriting every agent's config.

## Stage 3 — codeisland-bridge enriches and forwards

Source: `Sources/CodeIslandBridge/main.swift` (453 lines).

### Envelope schema

The bridge takes the agent's raw hook JSON on stdin and adds:

| Field                  | Source                                                | Notes |
|------------------------|-------------------------------------------------------|-------|
| `_source`              | `--source` arg, else inferred from process ancestry   | `claude`, `cursor`, `cursor-cli`, `codex`, etc. (~L299–L307) |
| `_ppid`                | Resolved CLI process PID                              | Walks ancestry past transient `sh -c` shells (~L316–L323) |
| `_hook_ppid`           | Immediate `getppid()` when different from `_ppid`     | Diagnostic |
| `_via_plugin`          | `true` when `--source` was absent but ancestry inferred one | Routes through plugin-session-mode (~L312) |
| `session_id`           | Existing field, or normalized from `sessionId`/payload, or fallback `<source>-ppid-<rootPid>` | (~L252–L342) |
| `hook_event_name`      | Existing, or normalized from `hookEventName`/`eventName`/`event`/`--event` arg | (~L241–L251) |
| `_term_app`            | `TERM_PROGRAM` env                                    | (~L364) |
| `_term_bundle`         | `__CFBundleIdentifier` env                            | (~L367) |
| `_iterm_session`       | `ITERM_SESSION_ID` env, GUID after `:`                | (~L372) |
| `_kitty_window`        | `KITTY_WINDOW_ID` env                                 | (~L381) |
| `_tmux`                | `TMUX` env                                            | (~L386) |
| `_tmux_pane`           | `TMUX_PANE` env                                       | (~L388) |
| `_tmux_client_tty`     | Output of `tmux display-message ... #{client_tty}`    | Uses absolute homebrew path (~L391) |
| `_tty`                 | `ttyname(open("/dev/tty"))`                           | (~L399) |
| `_cmux_surface_id`     | `CMUX_SURFACE_ID` env                                 | (~L407) |
| `_cmux_workspace_id`   | `CMUX_WORKSPACE_ID` env                               | (~L410) |

Copilot has special handling: its stdin payload uses camelCase and lacks `session_id` / `hook_event_name`, so the bridge promotes `sessionId` and the `--event` arg, and maps `toolName`/`toolArgs` → `tool_name`/`tool_input` (~L266–L282).

### Signal handling

- `signal(SIGPIPE, SIG_IGN)` (~L20) — broken pipe never kills the bridge mid-write.
- `signal(SIGALRM)` (~L26) — handler is `_exit(0)`, no cleanup. The bridge bails immediately if a deadline fires.
- The alarm is armed in two phases:
  - `alarm(5)` before `readDataToEndOfFile` on stdin (~L231) — protects against agents that spawn the hook but forget to close their pipe.
  - `alarm(isBlocking ? 8 : 4)` before env collection + connect + send (~L360).
  - For blocking events (permission / question), `alarm(0)` is called *after* `shutdown(SHUT_WR)` because the user can take hours to answer (~L439).

### Socket protocol

- Path: `SocketPath.path` = `$CODEISLAND_SOCKET_PATH` if set, else `/tmp/codeisland-<uid>.sock` (`Sources/CodeIslandCore/SocketPath.swift`, ~L4–L11).
- `connectSocket` does a non-blocking `connect()` + `poll()` with timeout 1 s (non-blocking event) or 3 s (blocking event) (~L121, ~L418).
- After sending the JSON, the bridge calls `shutdown(sock, SHUT_WR)` (~L435) — the server uses this half-close as EOF to know the request is complete. This was a deliberate choice, see [hook-pipeline §EOF semantics](#eof-semantics) below.
- The bridge then `recvAll`s the response (~L445); for blocking events, `SO_RCVTIMEO` is set to 86400 s.
- For blocking events, the response is forwarded to stdout (~L448–L450), which Claude Code / friends parse as the hook's decision payload.

## Stage 4 — HookServer accepts and routes

Source: `Sources/CodeIsland/HookServer.swift` (429 lines).

### Listener setup
`HookServer.start` (~L24):
1. `unlink` any stale socket file.
2. `umask(0o077)` *before* `NWListener` creates the socket — closes the brief window where the file would otherwise be world-readable.
3. Build `NWParameters` with `NWEndpoint.unix(path:)`, set TCP transport options (NWListener requires a transport even for AF_UNIX).
4. On `.ready`, restore previous umask and `chmod(socketPath, 0o700)` belt-and-suspenders.
5. On `stop`, the socket file is deleted **1 second after** `listener.cancel()` so in-flight bridge writes can finish (~L74). This fixed `#45`.

### Receive loop
`receiveAll` (~L88) reads in 64 KB chunks until `isComplete` (the EOF from the bridge's `SHUT_WR`) or error. Cap at `maxPayloadSize = 1_048_576` (~L85, ~L103) — oversize → drop.

### Three event lanes
After parsing into `HookEvent`, `HookServer.routeKind(for:)` (~L216) returns one of:

- `.permission` — `EventNormalizer.normalize(eventName) == "PermissionRequest"`. UI gates on user decision; response is `{"hookSpecificOutput":{"hookEventName":"PermissionRequest","decision":{"behavior":"allow"|"deny"}}}`. Auto-approved for `autoApproveTools` set (~L120, ~L326).
- `.question` — `Notification` event with a `QuestionPayload`. Routed to QuestionBar.
- `.event` — everything else. `appState.handleEvent(event)`, response is `{}`.

`AskUserQuestion` arrives as a permission event but is rerouted to the question lane (~L333–L342).

### Plugin pre-filter
Before parsing, `processRequest` (~L229) does a cheap byte probe for `_via_plugin` to decide whether to merge the event into a parent session, hide it (auto-allow), or keep it separate per `SettingsKey.pluginSessionMode` (~L242–L262).

### Webhook forwarding
`forwardEventToWebhook` (~L146) fire-and-forgets a POST to a user-configured URL with a stable envelope (`event`, `raw_event`, `session_id`, `source`, `cwd`, `tool_name`, `timestamp`, `raw`). 5 s timeout, failures swallowed.

### EOF semantics
The bridge's `shutdown(SHUT_WR)` would naturally trigger `connection.receive` to deliver `isComplete = true`. Earlier code used `connection.receive(min:1, max:1)` as a "peer disconnect" probe — but because the bridge always half-closes, that fired immediately and auto-denied every PermissionRequest. Current code uses `stateUpdateHandler` transitioning to `.cancelled`/`.failed` for real teardown only (`Sources/CodeIsland/HookServer.swift::monitorPeerDisconnect`, ~L386).

## Stage 5 — Routing into AppState

For `.event`: `appState.handleEvent(event)` (`Sources/CodeIsland/AppState.swift`, ~L890), then immediate `{}` response.

For `.permission` and `.question`: a `withCheckedContinuation` is created; `appState.handlePermissionRequest(event, continuation:)` (~L1033) or `handleQuestion`/`handleAskUserQuestion` enqueues the request. The continuation resumes only when the user clicks Approve/Deny/Answer in the UI, at which point the resumed `Data` is sent back to the bridge.

See [app state](app-state.md) for the queue + surface transitions.

## Failure modes

| Failure | What happens | Where |
|---|---|---|
| App not running, socket missing | `stat` check fails, bridge `exit(0)` | bridge ~L226 |
| Bridge stuck on stdin (caller didn't close pipe) | `alarm(5)` fires, `_exit(0)` | bridge ~L26, ~L231 |
| `connect()` times out (server stuck) | `poll()` returns 0, `close(sock)`, `exit(0)` | bridge ~L154 |
| Bridge writes fail mid-send | `SIGPIPE` ignored, `send` returns -1, loop breaks | bridge ~L20, ~L173 |
| Malformed JSON arrives | `HookEvent(from:)` returns nil → `{"error":"parse_failed"}` | HookServer ~L266 |
| Payload > 1 MB | Connection cancelled, no response | HookServer ~L103 |
| User never decides on permission | Continuation held in `permissionQueue`; bridge `recv` waits up to 86400 s | AppState ~L1064 |
| Bridge process killed (Ctrl-C) while user deciding | `NWConnection` transitions to `.cancelled`/`.failed`, `handlePeerDisconnect(sessionId:)` cleans up | HookServer ~L386, AppState ~L1440 |

## See also
- [glossary](glossary.md) (Envelope, Bridge, Hook)
- [architecture](architecture.md)
- [app state](app-state.md)

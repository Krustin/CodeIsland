# Remote sessions over SSH

How CodeIsland watches AI agents running on a *different* machine. There is no daemon on the remote box — instead, an `ssh -R` tunnel reverse-forwards `/tmp/codeisland.sock` from the remote host into the local `HookServer` socket, and a small Python hook script on the remote side speaks the same JSON envelope shape as `codeisland-bridge`.

## Scope

- The four files involved: `Sources/CodeIsland/RemoteHost.swift`, `RemoteManager.swift`, `RemoteInstaller.swift`, `SSHForwarder.swift`.
- The remote payload (`Sources/CodeIsland/Resources/codeisland-remote-hook.py`).
- See [hook-pipeline.md](hook-pipeline.md) for the local socket protocol and [agents.md](agents.md) for which agents are remote-installable today.

## Component map

```
   Mac (CodeIsland)                                Remote host
   ─────────────────────                           ─────────────────────
   AppState  ◀── HookServer ◀── /tmp/codeisland.sock ─┐
                                                     │ ssh -R (StreamLocal)
   RemoteManager ──▶ SSHForwarder ──▶ /usr/bin/ssh ──┘
                  ──▶ RemoteInstaller ──▶ ssh ───▶ python3 codeisland-remote-hook.py
                                                     ▲  (fired by patched
                                                     │   ~/.claude, ~/.codex,
                                                     │   ~/.codebuddy,
                                                     │   ~/.trae configs)
```

## RemoteHost — the model

`Sources/CodeIsland/RemoteHost.swift:3–65`. A `Codable` struct persisted to `UserDefaults` (key `remoteHosts`, see `RemoteManager.swift:16`).

Fields:

- `id`, `name` — UI identity.
- `host`, `user`, `port` — SSH target. `sshTarget` returns `user@host` (or just `host` if user is empty).
- `identityFile` — optional `-i <path>`.
- `autoConnect` — drives both startup auto-connect and the auto-reconnect policy (see below).
- `authSocket` — optional `SSH_AUTH_SOCK` override merged into the spawn env so password-manager-backed agents (1Password, Bitwarden) can sign the handshake when the GUI app didn't inherit the env (issue #81). Backward-compatible decode at `:36–47`.
- `remoteSocketPath` — hard-coded `/tmp/codeisland.sock`.

## SSHForwarder — the tunnel

`Sources/CodeIsland/SSHForwarder.swift`. One `Process` running `/usr/bin/ssh`. Single tunnel per host id; replacing it bumps a `generation` counter so stale termination handlers no-op (`:34`, `:52`).

SSH arguments (`buildArguments` at `:102–126`):

```
-N -T
-o BatchMode=yes
-o ExitOnForwardFailure=yes
-o ServerAliveInterval=15
-o ServerAliveCountMax=2
-o StreamLocalBindUnlink=yes
-o StreamLocalBindMask=0000
[-p <port>] [-i <identity>]
-R <remoteSocketPath>:<localSocketPath>
<sshTarget>
```

Status transitions (`Status` enum at `:5–10`):

- `disconnected` → `connecting` → `connected` once `process.isRunning` survives a 350ms grace window (`:70–79`).
- Any stderr text seen while still in `.connecting` flips to `.failed(<text>)` (`:142–157`).
- Termination handler maps non-zero exit to `.failed("ssh exited (<code>)")`.
- The stderr `readabilityHandler` is explicitly nil'd on exit because a closed FD with a live handler pegs CPU at 100% (`:54–57`).

Env injection: `buildEnvironment` (`:132–140`) tilde-expands `host.authSocket` and writes it as `SSH_AUTH_SOCK`.

## RemoteInstaller — pushing the hook

`Sources/CodeIsland/RemoteInstaller.swift`. Two-step Python upload over `ssh`:

1. **Upload** (`uploadRemoteHook` at `:54–66`): base64-encodes the bundled `codeisland-remote-hook.py` and pipes a tiny Python one-liner that decodes it to `~/.codeisland/codeisland-remote-hook.py` and `chmod 0755`. Timeout 25s.
2. **Configure** (`configureRemoteHooks` at `:68–77`): runs the long Python `configureRemoteHooksScript` under `"$SHELL" -lc` so login-shell rc files (`.zprofile`, `.bash_profile`) are sourced — necessary for `$CODEX_HOME` and similar env vars to be visible to a non-interactive ssh session. Script body is base64-encoded to survive double-shell quoting. Timeout 30s.

What the Python script patches on the remote (`configureRemoteHooksScript` at `:79–522`):

- `~/.claude/settings.json` — Claude format with `UserPromptSubmit/PermissionRequest/Notification/Stop/SessionStart/SessionEnd/PreCompact`. `install_claude` at `:412–439`.
- `$CODEX_HOME/hooks.json` (default `~/.codex/hooks.json`) plus `codex_hooks = true` in `config.toml`. `install_codex` at `:456–474`.
- `~/.codebuddy/settings.json` — same shape as Claude. `install_codebuddy` at `:476–503`.
- `~/.trae/traecli.yaml` — YAML managed-block merge. `install_traecli` at `:505–518`. Indentation normalizer at `:164–240` makes the merger tolerant of preexisting hand-edited YAML.

Each install function returns a one-line status (`"Claude ok"` / `"Codex skipped"` / etc.). Final stdout is `" · ".join(parts)` (`:521`) which `RemoteManager` surfaces as `lastMessage`.

The hook command injected for every event is built by `command_for(source)` at `:120`:

```
CODEISLAND_SOCKET_PATH=/tmp/codeisland.sock \
CODEISLAND_REMOTE_HOST_ID=<json id> \
CODEISLAND_REMOTE_HOST_NAME=<json name> \
CODEISLAND_SOURCE=<source> \
python3 ~/.codeisland/codeisland-remote-hook.py
```

The host id/name end up tagged on every envelope so the local `AppState` can attribute the session to the correct host. The remote hook version is `0.1.1` (`:17`).

`RemoteInstaller.cleanupRemoteSocket(host:)` at `:38–40` runs `rm -f /tmp/codeisland.sock` over SSH — called *before* dialing the tunnel so a stale socket from a crashed previous session doesn't make `ExitOnForwardFailure=yes` blow up the connect.

## RemoteManager — orchestration + auto-reconnect

`Sources/CodeIsland/RemoteManager.swift`. `@MainActor` singleton, observable.

State:

- `hosts: [RemoteHost]` — persisted via `JSONEncoder` to `UserDefaults` at key `remoteHosts` (`:14–17`, `:193–205`).
- `connectionStatus[id]` — mirrors `SSHForwarder.Status`.
- `installRunning[id]`, `lastMessage[id]` — per-host UI state.

Lifecycle:

- `startup()` — connects every host with `autoConnect = true` (`:39–43`).
- `shutdown()` — disconnects all.
- `connect(id:)` — clears the reconnect timer/counter and calls `connectInternal` which cleans the remote socket, then dials.
- On `.connected`, `installHooks(for:)` at `:183–191` runs the install script. If it fails, the host transitions to `.failed(<message>)`.

### Exponential-backoff reconnect

Implemented at `:21–33`, `:149–181`. Verified table:

```swift
private static let reconnectBackoffSeconds: [Int] = [5, 15, 45, 120, 300]
private static let reconnectMaxAttempts = 10
```

Behavior (`scheduleReconnect` at `:149–176`):

- Only triggers for hosts with `autoConnect == true`. Manual hosts that fail are left alone — otherwise a typo'd address would retry forever.
- Each `.failed` increments the attempt counter and arms a `Task` that sleeps `reconnectDelay(attempt:)` seconds.
- `reconnectDelay` clamps the index to the last entry (`:30–33`) — i.e. attempts 5..10 all wait 300s.
- After 10 failed attempts, sets `lastMessage` to `"Gave up after 10 reconnect attempts"` and stops.
- A successful `.connected` clears `reconnectAttempts[id]` (`:131`).
- `disconnect(id:)` and a new `connect(id:)` both call `cancelScheduledReconnect`.

Tested by `Tests/CodeIslandTests/RemoteManagerTests.swift` (the public `reconnectDelay(attempt:)` is exposed for that purpose).

## Failure modes

- **Stale remote socket** — `ExitOnForwardFailure=yes` causes `ssh` to exit immediately if the remote socket can't be bound. `cleanupRemoteSocket` is called every reconnect to prevent this.
- **Login shell didn't source rc files** — `$CODEX_HOME` not seen → Codex hooks land in `~/.codex` instead of the user's actual `CODEX_HOME`. Mitigated by running configure under `"$SHELL" -lc` (`RemoteInstaller.swift:75`).
- **Password-manager SSH agent invisible to GUI apps** — set `authSocket` on the host (issue #81).
- **Python missing on remote** — uploads run `python3`. Hosts without `python3` fail at the upload stage with `Upload failed: …`.
- **macOS sleep / network blip** — auto-reconnect handles it for `autoConnect` hosts; for others the user re-clicks Connect.
- **Multiple hosts pushing through the same `/tmp/codeisland.sock`** — by design. The envelope's `CODEISLAND_REMOTE_HOST_ID` disambiguates downstream.
- **stderr handler CPU spin** — fixed in place at `SSHForwarder.swift:54–57`. If you refactor `terminationHandler`, keep the explicit `readabilityHandler = nil`.

## See also

- Local equivalent of the remote hook: [hook-pipeline.md](hook-pipeline.md).
- Routing of remote-tagged sessions inside AppState: [app-state.md](app-state.md).
- Doc style: [conventions.md](conventions.md).

## Scope

`AppState` is the single `@MainActor`/`@Observable` brain holding all session, surface, permission, and question state for the app; every UI view reads it and every hook/event/timer eventually mutates it.

## Class layout

- Main type: `Sources/CodeIsland/AppState.swift::AppState (~L17)` — `@MainActor @Observable final class`. Owns ~3950 lines: session dict, surface, queues, FSEvent discovery, process monitoring, cleanup timer, permission/question handlers, transcript/model backfill, and rotation.
- `Sources/CodeIsland/AppState+CodexAppServer.swift` — JSON-RPC client lifecycle for Codex Desktop (`com.openai.codex`); maintains `codexAppServerClient` plus NSWorkspace launch/terminate observers.
- `Sources/CodeIsland/AppState+ToolUseCache.swift` — `pendingToolUses` cache of `PreToolUseRecord`, keyed by `tool_use_id`, used for permission correlation, dedup, and stale-queue draining.
- `Sources/CodeIsland/AppState+TranscriptTailer.swift` — attaches/detaches the lazy `JSONLTailer` per session and applies `ConversationTailDelta` on the main actor.

Cross-cutting state structs live in `Sources/CodeIslandCore/Models.swift` (`HookEvent`, `SessionSnapshot`, `AgentStatus`, `ChatMessage`, …) and `Sources/CodeIsland/Models.swift` (UI-only types like `PermissionRequest`, `QuestionRequest`, `QuestionPayload`).

## Key @Published / observed surfaces

`AppState` uses Swift `@Observable`, so any non-`@ObservationIgnored` stored property is reactive. The minimal contract UI depends on:

- `sessions: [String: SessionSnapshot]` — `AppState.swift::sessions (~L38)`. Source of truth for everything that happens per-session.
- `activeSessionId: String?` — `AppState.swift::activeSessionId (~L39)`. Drives "focused" UI (status item, rotation hand-off).
- `permissionQueue: [PermissionRequest]` — `AppState.swift::permissionQueue (~L40)`. FIFO of pending approval cards; head is shown.
- `questionQueue: [QuestionRequest]` — `AppState.swift::questionQueue (~L41)`. Same shape, for ask-user-question style prompts.
- `surface: IslandSurface` — `AppState.swift::surface (~L101)`. Single source of truth for which face the notch is showing.
- `status: AgentStatus` (derived/refreshed) — `AppState.swift::status (~L801)`. Aggregate "what's the most interesting agent doing right now" used by the compact bar.
- `primarySource`, `activeSessionCount`, `totalSessionCount` — `AppState.swift (~L802–804)`. Header/mascot inputs.
- `rotatingSessionId` / `rotatingSession` — `AppState.swift::rotatingSession (~L145)`. Drives the compact-bar carousel when multiple sessions are active.

`@ObservationIgnored` (internal bookkeeping, not for UI reads): `pendingToolUses`, `attachedTranscriptPaths`, `transcriptTailer`, `codexAppServerClient`, `codexAppServerObservers`, `recentHookEvents`.

Computed convenience accessors used heavily by views: `pendingPermission` (`~L96`), `pendingQuestion` (`~L98`), `justCompletedSessionId` (`~L103`).

## IslandSurface state machine

Defined in `Sources/CodeIsland/IslandSurface.swift::IslandSurface (~L2)`. Cases (verified):

- `collapsed` — only the compact bar / notch shell visible.
- `sessionList` — user-driven expansion showing all sessions.
- `approvalCard(sessionId:)` — permission request awaiting approve/deny.
- `questionCard(sessionId:)` — ask-user-question prompt awaiting an answer.
- `completionCard(sessionId:)` — auto-expanded "this session just finished" notification.

`isExpanded` returns `true` for any case other than `.collapsed`. `sessionId` is `nil` for `.collapsed` / `.sessionList` and the associated id otherwise.

Transitions (each labelled with the call site that performs the write):

- `collapsed → sessionList` (user tap on idle indicator / compact bar) — `Sources/CodeIsland/NotchPanelView.swift (~L276, ~L1651)` and `Sources/CodeIsland/AppDelegate.swift (~L100, ~L121, ~L190)`.
- `* → approvalCard(sid)` (new permission arrives and queue was empty) — `AppState.swift::handlePermissionRequest (~L1072)`. Suppressed if `surface == .sessionList` so user browsing isn't yanked.
- `* → questionCard(sid)` (new Notification-style question or AskUserQuestion arrives, queue was empty) — `AppState.swift::handleQuestion (~L1217)` and `handleAskUserQuestion (~L1318)`. Wrapped in `withAnimation(NotchAnimation.open)`.
- `idle → completionCard(sid)` (session ended; auto-expand) — `AppState.swift::doShowCompletion (~L769)`. Gated by `enqueueCompletion` / `shouldSuppressAppLevel` so interactive surfaces are not stomped (`AppState.swift::isShowingInteractive (~L128)`).
- `completionCard → collapsed` (auto-collapse timer fires, mouse outside) — `AppState.swift::autoCollapseTask (~L774)` + `showNextCompletionOrCollapse (~L787)` → `surface = .collapsed` (~L796). Deferred until mouse-leave when `deferCollapseOnMouseLeave` is set.
- `approvalCard → next` (approve/deny/dismiss) — `approvePermission (~L1079)` / `denyPermission (~L1157)` / `dismissPermissionPrompt (~L1176)` all funnel through `showNextPending (~L1476)`, which picks the next visible permission, then question, else collapses (`~L1502/L1504`).
- `approvalCard → collapsed` (last dismissable card hidden) — `dismissPermissionPrompt (~L1187)` wrapped in `withAnimation(NotchAnimation.close)`.
- `questionCard → next` (answer / skip) — `answerQuestion (~L1325)`, `answerQuestionMulti (~L1364)`, `skipQuestion (~L1410)` → `showNextPending`.

Invariant: surface transitions are always assigned synchronously on the main actor; animation wrappers use `NotchAnimation` constants from `Sources/CodeIsland/NotchAnimation.swift`.

## Session lifecycle

1. **Creation.** A `SessionSnapshot` is inserted into `sessions` on first hook event for an id (`AppState.swift::handleEvent (~L890, ~L906)`) or via discovery scan (`~L1953` `startSessionDiscovery`). Metadata (`cwd`, `source`, `cliPid`, `terminal`) is filled by `extractMetadata`.
2. **Process monitor attach.** `tryMonitorSession (~L621)` resolves a live `ProcessIdentity` via `liveProcessIdentity` / `trackedProcessIdentity` (`~L331, ~L342`) and arms a `DispatchSourceProcess` exit handler in `monitorProcess (~L408)`. Codex Desktop sessions skip this — they're tracked through the JSON-RPC channel instead.
3. **Transcript tailing.** `attachTranscriptTailerIfNeeded (AppState+TranscriptTailer.swift::~L8)` attaches the shared `JSONLTailer` to `sessions[id].transcriptPath`, idempotent on `attachedTranscriptPaths`.
4. **Discovery rescan.** An FSEventStream on session-store roots (`~L1965 startProjectsWatcher`) fires `handleProjectsDirChange (~L1998)` with a 2-second coalesce + 3-second debounce, enqueuing `requestDiscoveryScan`.
5. **Cleanup reaper.** `startCleanupTimer (~L151)` schedules a `Timer.scheduledTimer(withTimeInterval: 3, repeats: true)` invoking `cleanupIdleSessions (~L160)`:
   - Verifies each monitored PID is still alive via `proc_pidinfo(PROC_PIDTBSDINFO)` and synthesizes a `handleProcessExit` when the `DispatchSourceProcess` silently missed it.
   - Sends `SIGTERM` to orphaned children (`ppid <= 1`) via `shouldTerminateOrphanedProcess (~L324)`.
   - Force-resets stuck `waitingApproval`/`waitingQuestion`/processing sessions after 180–300s of no activity *only* when no process monitor is attached (remote sessions and live local monitors are exempt).
   - Also calls `prunePendingToolUses` on the same tick to age out `PreToolUseRecord` entries past `pendingToolUseTTL = 900` (`AppState+ToolUseCache.swift::~L25`).
6. **Removal.** `handleProcessExit (~L439)` adds the session to `exitingSessions` for a 5-second grace, then `removeSession (~L513)` strips it, detaches the tailer, and stops the monitor.

## Permission request async flow

End-to-end: a hook server request is suspended on a Swift continuation, queued into AppState, and resumed only when the user clicks a button.

1. `HookServer` receives a `PermissionRequest` HTTP/socket call and packages it into a `HookEvent`.
2. It calls `withCheckedContinuation { cont in appState.handlePermissionRequest(event, continuation: cont) }`, parking the request handler.
3. `handlePermissionRequest (AppState.swift::~L1033)`:
   - Ensures the `SessionSnapshot` exists and extracts metadata.
   - Calls `enrichPermissionRequestFromCache (~L1053)` to backfill missing `toolName` / `toolDescription` from `pendingToolUses` (see next section).
   - Wraps event + continuation into a `PermissionRequest` (`Models.swift`) and either merges (`mergeDuplicatePermissionRequest`) or appends to `permissionQueue`.
   - If this is the head item and `surface != .sessionList`, flips `surface = .approvalCard(sessionId:)` and plays the `PermissionRequest` sound.
4. The UI (`ApprovalBar` rendered in `NotchPanelView.swift (~L114)`) calls one of:
   - `approvePermission(always:)` (`~L1079`) — resumes the continuation with `{"hookSpecificOutput":{"hookEventName":"PermissionRequest","decision":{"behavior":"allow"}}}` (or an `addRules` envelope when `always == true`).
   - `denyPermission()` (`~L1157`) — resumes with `behavior:"deny"`.
   - `dismissPermissionPrompt()` (`~L1176`) — marks this session id as dismissed in `dismissedPermissionSessionIds` and advances; the continuation stays parked until the next decision or until the tool resolves itself.
5. Each terminal path calls `showNextPending (~L1476)`, which surfaces the next queued permission or question, otherwise collapses.

Stale waiters are also drained surgically by `resolveToolUseIfCompleted` (see next section) when a `PostToolUse` / `PostToolUseFailure` / `PermissionDenied` event arrives for the same `tool_use_id`.

Wire format constants (deny body used in multiple paths):

```
{"hookSpecificOutput":{"hookEventName":"PermissionRequest","decision":{"behavior":"deny"}}}
```

## ToolUseCache role

`PreToolUseRecord` (`AppState+ToolUseCache.swift::~L14`) snapshots `{sessionId, toolName, toolDescription, toolInput, receivedAt}` keyed by `tool_use_id` for up to 15 minutes (`pendingToolUseTTL` ~L25).

- **Cache write.** `cachePreToolUseIfApplicable (~L29)` is called from `handleEvent` (`AppState.swift::~L916`) for every `PreToolUse` carrying a `toolUseId`.
- **Permission enrichment.** When a thin `PermissionRequest` arrives missing tool fields, `enrichPermissionRequestFromCache (~L125)` patches `sessions[id].currentTool` / `toolDescription` from the cached record so the approval card knows what's being requested. This is the primary reason the cache exists.
- **Replay dedup.** `mergeDuplicatePermissionRequest (~L94)` swaps a duplicate-`tool_use_id` queue entry in place, denying the older continuation so Claude's prior waiter doesn't hang while keeping the visible card stable.
- **Stale drain.** `resolveToolUseIfCompleted (~L45)` drops the cache entry on `PostToolUse` / `PostToolUseFailure` / `PermissionDenied` and, if a `PermissionRequest` for the same id is still queued (agent moved on locally), denies it surgically. Some providers (`trae*`) opt out via `shouldKeepQueuedPermissionForCompletedEvent (~L107)` because they re-emit completion before approval.
- **TTL prune.** `prunePendingToolUses (~L82)` is called from the cleanup timer.

## CodexAppServer

`Sources/CodeIsland/AppState+CodexAppServer.swift` integrates with Codex Desktop (`com.openai.codex`):

- `startCodexAppServerWatcher (~L21)` registers NSWorkspace `didLaunch` / `didTerminate` observers on the bundle id, and catches up on app boot if Codex is already running.
- When Codex launches, `startCodexAppServerClientIfPossible (~L66)` spawns `codex app-server` (path from `CodexAppServerClient.defaultExecutablePath`), wires `onMessage` / `onExit` callbacks back to the main actor, and runs the JSON-RPC `initializeHandshake`.
- Inbound JSON-RPC messages are dispatched to `handleCodexAppServerMessage` (rest of the extension), which creates / updates sessions keyed under the `codexapp:` prefix (`codexAppSessionPrefix ~L10`). The prefix keeps Codex Desktop sessions disjoint from Codex CLI sessions discovered through the rollout-file path.
- On Codex Desktop quit (or client crash via `onExit`), `removeCodexAppServerSessions (~L99)` strips any `codexapp:`-prefixed session.

These sessions never get a local `DispatchSourceProcess` monitor; their lifecycle is purely driven by the JSON-RPC stream.

## See also

- [glossary.md](glossary.md) — definitions for `HookEvent`, `SessionSnapshot`, `AgentStatus`, sources.
- [hook-pipeline.md](hook-pipeline.md) — how `HookServer` turns sockets into the `handlePermissionRequest` / `handleQuestion` / `handleEvent` calls described above.
- [ui-panel.md](ui-panel.md) — how `surface` and the queues are rendered.
- [agents.md](agents.md) — per-source quirks referenced by `mergeDuplicatePermissionRequest` / `shouldKeepQueuedPermissionForCompletedEvent`.
- [remote-sessions.md](remote-sessions.md) — why the cleanup reaper skips remote sessions.
- [architecture.md](architecture.md) — module overview.

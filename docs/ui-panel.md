## Scope

The floating notch panel — an `NSPanel` hosting a SwiftUI tree (`NotchPanelView`) that renders the bar, the per-surface card, and per-agent mascots, positioned under (or simulating) the menu-bar notch on the active display.

## Window plumbing

Owner: `Sources/CodeIsland/PanelWindowController.swift::PanelWindowController (~L85)`, an `@MainActor NSObject : NSWindowDelegate`.

`showPanel (~L153)` constructs a private `KeyablePanel : NSPanel` (`~L7`) with the following choices (verified):

- `styleMask = [.borderless, .nonactivatingPanel]` (`~L161`).
- `backing: .buffered`, `defer: false`.
- `isFloatingPanel = true` (`~L165`).
- `level = mainMenuWindow + 2` via `CGWindowLevelForKey(.mainMenuWindow)` (`~L167`) — sits above the menu bar.
- `backgroundColor = .clear`, `isOpaque = false`, `hasShadow = false` (`~L168–170`).
- `isMovableByWindowBackground = false`, `hidesOnDeactivate = false` (`~L171–172`).
- `collectionBehavior = [.canJoinAllSpaces, .fullScreenAuxiliary, .stationary, .ignoresCycle]` (`~L173`).
- `sharingType = .readOnly`, `acceptsMouseMovedEvents = true` (`~L166, L174`).

Why `.nonactivatingPanel` matters: clicking the panel must not steal focus from the terminal/IDE the user is actually working in. The trade-off is that AppKit consumes the first click for activation handling; `KeyablePanel.canBecomeKey = true` plus `NotchHostingView.mouseDown` (which calls `window?.makeKey()` before forwarding, `~L20`) and `acceptsFirstMouse = true` (`~L25`) ensure the *first* click still drives SwiftUI button actions instead of being swallowed.

Display ordering uses `orderFrontRegardless()` (`~L183`); visibility is gated by `updateVisibility (~L541)` based on fullscreen detection (`isActiveSpaceFullscreen ~L559`) and the `hideWhenNoSession` setting.

## NotchHostingView re-entrancy workaround

`NotchHostingView<Content>: NSHostingView<Content>` (`PanelWindowController.swift::~L16`) guards against an AppKit crash where, during `updateConstraints()` / layout, SwiftUI re-invalidates the view graph and synchronously calls `setNeedsUpdateConstraints` again — AppKit forbids this (`_postWindowNeedsUpdateConstraints` throws).

Mechanism:

- Override `needsUpdateConstraints` and `needsLayout` setters (`~L32–62`).
- Instead of forwarding to `super` synchronously, the new value is captured and re-applied on the next runloop turn via `DispatchQueue.main.async`.
- A re-entrancy flag `applyingDeferred` (`~L18`) lets the deferred handler set `super` without re-deferring.
- Net effect: one-tick delay on constraint/layout invalidation, imperceptible visually, no crash.

## Screen / notch detection

`Sources/CodeIsland/ScreenDetector.swift::ScreenDetector (~L3)` — pure struct, no state.

- `screenHasNotch(_:) (~L79)` — true MacBook notch detection from `NSScreen.safeAreaInsets.top`.
- `topBarHeight(for:) (~L87)` / `notchWidth(for:) (~L136)` — real geometry on notch displays, simulated values otherwise (`fakeNotchWidth ~L12` clamps `screenW * 0.14` between 160 and 240).
- `autoPreferredIndex(candidates:activeWindowBounds:) (~L17)` — picks the screen containing the frontmost app window, falls back to greatest overlap, then to the notch-bearing screen, then main.
- `signature(for:) (~L147)` — stable identifier used to detect a meaningful change before rebuilding.

Multi-display behavior in `PanelWindowController`:

- `chosenScreen (~L510)` consults `ScreenDetector.preferredScreen` (or auto via `autoPreferredIndex` + `frontmostApplicationWindowBounds`).
- `configureAutoScreenPolling (~L419)` runs a low-frequency timer that re-evaluates the active window; if its `signature` changes, `refreshCurrentScreen (~L307)` is invoked.
- Screen-hop animation: `animateScreenHop (~L323)` fades the panel out on the old screen (`fadeOutDuration 0.14s`), rebuilds `NotchPanelView` against the new screen geometry via `rebuildForCurrentScreen (~L298)`, then fades in (`fadeInDuration 0.34s`) with a `incomingPauseDuration 0.06s` gap. Skipped when `accessibilityDisplayShouldReduceMotion` is set — panel jumps directly.
- A horizontal drag monitor (`setupHorizontalDragMonitor ~L463`) lets the user shove the panel sideways within the chosen screen, clamped by `clampedX (~L459)`.

## NotchPanelView root

`Sources/CodeIsland/NotchPanelView.swift::NotchPanelView (~L4)` — top-level SwiftUI view, takes `appState`, `hasNotch`, `notchHeight`, `notchW`, `screenWidth`.

Layout (`body ~L70`):

- Outer `VStack` containing the compact bar (`CompactLeftWing` / `CompactToolStatus` / `CompactRightWing` ~L75–86) sized to `notchHeight`.
- Below-notch expanded content rendered when `shouldShowExpanded` is true (`~L39, L104`): a dashed separator (`Line`) plus a switch on `appState.surface` (see next section).
- Background shape `NotchPanelShape` (`~L176`) draws the rounded notch silhouette with different top extension and bottom radius depending on expanded state.

Hover handling:

- `hoverTimer: Timer?` (`~L21`) plus boolean state `isHovered` / `idleHovered`.
- Idle indicator hover (`~L205`) un-hovers after 300 ms to prevent oscillation when the cursor brushes through.
- Expand-on-hover for the active state (`~L243`) uses a 500 ms timer before triggering, and a 150 ms timer for collapse (`~L287`), and respects `SettingsManager.shared.collapseOnMouseLeave` plus the `smartSuppress` setting.
- Completion-card hover (`~L226`) marks `completionHasBeenEntered` so auto-collapse waits for the user to leave the panel before dismissing.

Width is computed reactively in `panelWidth` (`~L57`) based on idle/active state, whether the surface is expanded, and the displayed tool-status chip.

## Surface to view mapping

Switch lives at `NotchPanelView.swift::body (~L110)`:

| `IslandSurface` case        | Rendered view                                              | Site            |
|-----------------------------|------------------------------------------------------------|-----------------|
| `.approvalCard(sid)`        | `ApprovalBar(...)` fed by `appState.pendingPermission`     | `~L111–128`     |
| `.questionCard(sid)`        | `QuestionBar(...)` from `pendingQuestion` or `previewQuestionPayload` | `~L129–161`     |
| `.completionCard`           | `SessionListView(onlySessionId: justCompletedSessionId)`   | `~L162–164`     |
| `.sessionList`              | `SessionListView(onlySessionId: nil)`                      | `~L165–167`     |
| `.collapsed`                | `EmptyView` (only the compact bar above remains visible)   | `~L168–170`     |

Transitions are wrapped in `.blurFade.combined(with: .scale / .move)` from `Sources/CodeIsland/NotchAnimation.swift`.

## Per-agent view shape

Per-agent mascot views are independent `struct ... : View` types — *no shared protocol*, no shared base. They are not polymorphic over a session; they are pure renderers for `(AgentStatus, size)`. The seam is hand-rolled inside `MascotView` (see below).

Inspected: `Sources/CodeIsland/CursorView.swift::CursorView (~L7)`, `Sources/CodeIsland/CopilotView.swift::CopilotView (~L7)`, `Sources/CodeIsland/GeminiView.swift::GeminiView (~L6)`. Each one follows the same convention:

- Inputs: `let status: AgentStatus`, `var size: CGFloat = 27`.
- `@State private var alive = false` and `@Environment(\.mascotSpeed) private var speed` — common across mascot files.
- Brand palette declared as `private static let` colors.
- `body` switches on `status` (`.idle → sleepScene`, `.processing/.running → workScene`, `.waitingApproval/.waitingQuestion → alertScene`).
- Private nested `struct V` that captures the SVG-to-screen transform (`ox, oy, s, y0`) with helpers `r(...)` and `pt(...)`.
- Per-state `Canvas`-based draw functions (`sleepCanvas`, `workCanvas`, `alertCanvas`) plus draw helpers (`drawShadow`, `drawLegs`, agent-specific shape draws).
- Animation reset on `onChange(of: status)` toggling `alive` to retrigger keyframes.

There is no `AgentMascot` protocol — adding a new agent means writing a new `struct FooView: View` matching this informal shape and adding a case in `MascotView`.

## Mascots

`Sources/CodeIsland/MascotView.swift::MascotView (~L18)` is the dispatcher:

- Inputs: `source: String, status: AgentStatus, size: CGFloat = 27`.
- Body is a `switch source` mapping each normalised source identifier to its concrete view (`~L26–61`). Verified routes: `codex → DexView`, `gemini → GeminiView`, `cursor → CursorView`, `trae`/`traecn`/`traecli → TraeView`, `copilot → CopilotView`, `qoder → QoderView`, `droid → DroidView`, `codebuddy`/`codybuddycn → BuddyView`, `stepfun → StepFunView`, `opencode → OpenCodeView`, `qwen → QwenView`, `antigravity → AntiGravityView`, `workbuddy → WorkBuddyView`, `hermes → HermesView`, `kimi → KimiView`, default → `ClawdView`.
- Wraps the chosen view in `.environment(\.mascotSpeed, Double(speedPct)/100.0)` so each agent's animation respects the user-tunable speed setting.

The fallback `ClawdView` is the original Claude mascot, defined in `Sources/CodeIsland/PixelCharacterView.swift::ClawdView (~L6)`, which establishes the canvas-based pixel-art pattern (`sleepCanvas ~L150`, `workCanvas ~L170`, `alertCanvas ~L286`, `floatingZs ~L125`, shared `V`-style transform `~L38`) the per-agent views follow.

Selection input (`source`) comes from `SessionSnapshot.source` normalized via `SessionSnapshot.normalizedSupportedSource`.

## Status item

`Sources/CodeIsland/StatusItemController.swift::StatusItemController (~L10)` — `@MainActor final class`, singleton (`shared`).

- `startObserving (~L17)` calls `syncVisibility` and KVO-observes `UserDefaults.standard.hideWhenNoSession`.
- `syncVisibility (~L26)` — only shows the menu-bar status item when `hideWhenNoSession` is true (because then the notch panel itself disappears and the user still needs a re-entry point).
- `showStatusItem (~L34)` creates `NSStatusBar.system.statusItem(withLength: NSStatusItem.squareLength)`, sets the bundle icon at 18×18, and attaches `menu`.
- `makeMenu (~L55)` builds a two-item `NSMenu`: `settings_ellipsis` → `openSettings (~L79)` which opens `SettingsWindowController.shared`, then a separator and `quit` → `NSApp.terminate(nil)` (`~L85`).

The status item is intentionally *not* the primary UI — it's the fallback opener when the panel is hidden.

## See also

- [app-state.md](app-state.md) — `IslandSurface`, `surface`, `permissionQueue`, `questionQueue` consumed by `NotchPanelView`.
- [glossary.md](glossary.md) — `AgentStatus`, `SessionSnapshot.source` values listed in `MascotView`.
- [agents.md](agents.md) — provider-specific source identifiers used by the mascot router.
- [conventions.md](conventions.md) — SwiftUI / animation conventions.
- [architecture.md](architecture.md) — how the panel sits relative to `HookServer` and `AppState`.

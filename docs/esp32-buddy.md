# Buddy (ESP32 BLE Companion)

## Scope
Optional BLE companion ("Buddy") that mirrors the notch's mascot, workspace, and message preview onto an external LCD and emits button-press uplinks back to CodeIsland.

## BLE identifiers
All values come from `Sources/CodeIslandCore/ESP32Protocol.swift::ESP32Protocol (~L43)`.

- Service UUID: `0000beef-0000-1000-8000-00805f9b34fb`
- Write (host→Buddy, `WRITE_NR`): `0000beef-0001-1000-8000-00805f9b34fb`
- Notify (Buddy→host): `0000beef-0002-1000-8000-00805f9b34fb`
- Advertised name prefix: `Buddy` (firmware appends `-XXXXXX` chipId — each device advertises a unique `Buddy-XXXXXX` local name).

## Downlink frame formats
Every frame is ≤ 20 bytes and written `.withoutResponse`. Encoders live in `Sources/CodeIslandCore/ESP32Protocol.swift`.

### Agent frame — `MascotFramePayload.encode (~L245)`
No leading marker (distinguished by `byte[0]` being a valid `MascotID` 0..15).

```
byte[0]   = sourceId   (MascotID, 0..15)
byte[1]   = statusId   (MascotStatusCode, 0..4)
byte[2]   = toolLen    (0..17)
byte[3..] = toolName   UTF-8, byte-truncated to maxToolNameBytes = 17
```

`MascotStatusCode`: `0 idle`, `1 processing`, `2 running`, `3 waitingApproval`, `4 waitingQuestion`.

### Workspace frame — `BuddyWorkspacePayload.encode (~L276)`
```
byte[0]   = 0xFC  (workspaceFrameMarker)
byte[1]   = workspaceLen  (0..18)
byte[2..] = workspace UTF-8, truncated to maxWorkspaceNameBytes = 18
```

### Message preview frame — `BuddyMessagePreviewPayload.encode (~L309)`
```
byte[0]   = 0xFB  (messagePreviewFrameMarker)
byte[1]   = messageIndex (0-based)
byte[2]   = messageCount
byte[3]   = flagsAndLen   (bit7 = isUser, low7 = textLen, max 16)
byte[4..] = preview UTF-8, truncated to maxMessagePreviewBytes = 16
```

### Brightness frame — `BuddyBrightnessPayload.encode (~L347)`
```
byte[0] = 0xFE  (brightnessFrameMarker)
byte[1] = brightness percent (clamped to 10..100, default 70)
```

### Screen orientation frame — `BuddyScreenOrientationPayload.encode (~L360)`
```
byte[0] = 0xFD  (orientationFrameMarker)
byte[1] = orientation (0 = up, 1 = down)
```

## Uplink opcodes
A single byte from the notify characteristic. Decoded by `BuddyUplinkEvent.init(payload:) (~L87)` in `Sources/CodeIslandCore/ESP32Protocol.swift`.

| Byte | Meaning |
|------|---------|
| `0x00..0x0F` | Focus request — `MascotID` of the displayed mascot (button press). |
| `0xF0` | `approveCurrentPermission` — approve current pending permission. |
| `0xF1` | `denyCurrentPermission` — deny current pending permission. |
| `0xF2` | `skipCurrentQuestion` — skip the current `AskUserQuestion`. |

Anything else: ignored.

## Pairing & discovery
Implementation in `Sources/CodeIsland/ESP32BridgeManager.swift`.

- The user enters discovery from Settings; `startDiscovery (~L136)` runs `scanForPeripherals` with `allowDuplicates = true` and service-UUID filter so RSSI updates flow live.
- Each peripheral becomes a `DiscoveredBuddy (~L33)` (id = `CBPeripheral.identifier`, plus name, rssi, lastSeen). Entries older than `discoveryStaleSeconds = 10s` are pruned.
- `select(buddyId:) (~L160)` persists the chosen UUID to `UserDefaults` under `SettingsKey.selectedBuddyIdentifier` (and friendly name under `SettingsKey.selectedBuddyName`). `forgetSelection (~L186)` clears both.
- On launch, `loadSelectionFromDefaults (~L250)` re-reads the UUIDs; `attemptReconnectToSelected (~L264)` first tries `retrievePeripherals(withIdentifiers:)`, then falls back to a directed scan (`beginDirectedScan (~L297)`).
- Reconnect backoff: `[1, 2, 4, 8, 16, 30]` seconds (`reconnectBackoff (~L68)`), mirroring the firmware's own backoff. State exposed via `ESP32BridgeStatus (~L8)`.

## State publisher heartbeat
`Sources/CodeIsland/ESP32StatePublisher.swift::ESP32StatePublisher`.

- Heartbeat interval is user-configurable, defaults to `5.0s` (`heartbeatInterval (~L22)`), clamped to ≥ 1s in `configure (~L43)`.
- Firmware AGENT-mode inactivity timeout is `60_000ms` (`ESP32Protocol.firmwareInactivityTimeoutMs`), so the 5s tick keeps the device alive comfortably.
- Each `flush (~L79)` push sends: the agent frame (`MascotFramePayload`), the workspace frame, and one or more message-preview frames (one per segmented preview line).
- `notifyDirty (~L74)` is called from `AppState.refreshDerivedState` so any session mutation also pushes immediately, and arms a short interactive retry burst (600ms, 1.8s) for waiting-approval / waiting-question states to survive transient drops.
- Display selection mirrors the notch: `rotatingSessionId ?? activeSessionId ?? first sorted session` — see `AppState.esp32DisplaySession (~L137)`.

## Button-press routing
1. `ESP32BridgeManager.didUpdateValueFor (~L594)` decodes the uplink byte into a `BuddyUplinkEvent`.
2. For `.focus(MascotID)`, it invokes `onFocusRequest`, wired to `ESP32FocusCoordinator.handle (~L31)` in `Sources/CodeIsland/ESP32FocusCoordinator.swift`. That coordinator filters `AppState.sessions` by `source == mascot.sourceName`, ranks by status priority (`waitingApproval > waitingQuestion > running > processing > idle`) then by `lastActivity`, and hands the winner to `TerminalActivator.activate`. If no session exists, it falls back to activating the source's desktop app via `TerminalActivator.sourceToNativeAppBundleId`.
3. For `.command(BuddyControlCommand)`, it invokes `onControlCommand`, routed to the pending permission/question respondent in `AppState`.

## Firmware
Firmware sources, render notes, and per-mascot bitmaps live under `/Users/j.loose/Documents/GitHub/CodeIsland/hardware/` (`hardware.ino`, `mascot_*.h`, `HARDWARE_NOTES.md`, `RENDER_OPTIMIZATION.md`). The Mac app only owns the wire contract above; firmware build/flash is out of scope here.

## See also
- [architecture.md](./architecture.md)
- [agents.md](./agents.md)
- [glossary.md](./glossary.md)

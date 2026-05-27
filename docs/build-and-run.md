# Build & Run

## Scope
How to build, sign, notarize, package, and run CodeIsland locally. Driven by `build.sh` at the repo root.

## Prerequisites
- Xcode / Swift 5.9+ toolchain (Package.swift `swift-tools-version: 5.9`, platforms `.macOS(.v14)` — `Package.swift (~L1)`).
- macOS 14+ deployment target (also encoded in `actool --minimum-deployment-target 14.0`, `build.sh (~L111)`).
- `swift` available on `PATH` (drives both arm64 and x86_64 builds).
- Optional, only for `--notarize`:
  - A keychain notarization profile named `CodeIsland` (created via `xcrun notarytool store-credentials CodeIsland`).
  - `create-dmg` on `PATH`.
  - A `Developer ID Application` signing identity in the login keychain.

## Build flags
From `build.sh::usage (~L18)`:

- `./build.sh` — default: build macOS app only.
- `./build.sh --watch` — Android watch app **only** (skips the macOS build).
- `./build.sh --with-watch` — macOS app **and** Android watch app.
- `./build.sh --notarize` — after signing, submit the app (and DMG) to Apple notarization. Only runs the notarize block when `SIGN_ID` contains `Developer ID` (`build.sh (~L160)`).
- `./build.sh --help` / `-h`.

`SIGN_ID` env var, if set, short-circuits the auto-detect.

## macOS build walkthrough
`build.sh::build_mac (~L66)`.

1. **Universal build** — two passes: `swift build -c release --arch arm64` then `--arch x86_64`. Outputs in `.build/arm64-apple-macosx/release/` and `.build/x86_64-apple-macosx/release/`.
2. **Bundle skeleton** — `rm -rf .build/release/CodeIsland.app` then `mkdir -p` `Contents/{MacOS,Helpers,Resources,Frameworks}`.
3. **`lipo`** — fat-combine `CodeIsland` into `Contents/MacOS/CodeIsland`, and `codeisland-bridge` into `Contents/Helpers/codeisland-bridge`.
4. **Info.plist** — straight `cp Info.plist Contents/Info.plist`.
5. **Sparkle.framework** — `ditto` copy from `.build/artifacts/sparkle/Sparkle/Sparkle.xcframework/macos-arm64_x86_64/Sparkle.framework` into `Contents/Frameworks/` (ditto is required to preserve the framework's `Versions/` symlinks).
6. **rpath** — `install_name_tool -add_rpath @executable_path/../Frameworks` on the main binary, and `@executable_path/../../Frameworks` on `Helpers/codeisland-bridge`.
7. **Icon compile** — `xcrun actool` against `Assets.xcassets` + `AppIcon.icon`, output partial plist `.build/AppIcon.partial.plist`, `--minimum-deployment-target 14.0` (`build.sh (~L104)`).
8. **SPM resource bundle copy** — the first `.build/*/release/*.bundle` directory is `cp -R`'d into `Contents/Resources/` so code signing accepts the bundle (`build.sh (~L119)`).

## Code signing (inside-out)
Order in `build.sh (~L141)` matters — every nested component must be signed before its parent:

1. `Sparkle.framework/Versions/B/XPCServices/*.xpc`
2. `Sparkle.framework/Versions/B/Updater.app` (if present)
3. `Sparkle.framework/Versions/B/Autoupdate` (if present)
4. `Sparkle.framework` itself
5. `Contents/Helpers/codeisland-bridge`
6. `CodeIsland.app` — the only step that passes `--entitlements CodeIsland.entitlements`.

All steps use `--force --options runtime --sign "$SIGN_ID"`.

**`SIGN_ID` auto-detect** (`build.sh (~L130)`):

1. If `$SIGN_ID` is set in env, use it as-is.
2. Else `security find-identity -v -p codesigning | grep "Developer ID Application" | head -1`.
3. Else any non-revoked codesigning identity, `head -1`.
4. Else fall back to ad-hoc `-` (printed warning: "No developer certificate found, using ad-hoc signing...").

## Entitlements
`CodeIsland.entitlements`:

- `com.apple.security.automation.apple-events` = true — required to drive Terminal.app / iTerm2 / Ghostty etc. via AppleScript for the terminal-jump feature.
- `com.apple.security.device.bluetooth` = true — Buddy BLE bridge (CoreBluetooth).

**Not sandboxed.** There is no `com.apple.security.app-sandbox` key. The app needs filesystem access to `~/.claude/`, `~/.codex/`, Warp's group container SQLite, etc.; sandboxing would break all of it. `NSAppleEventsUsageDescription` and `NSBluetoothAlwaysUsageDescription` are declared in `Info.plist`.

## Sparkle / auto-update
Keys in `Info.plist`:

- `SUFeedURL` = `https://raw.githubusercontent.com/wxtsky/CodeIsland/main/appcast.xml`
- `SUPublicEDKey` = `oqLtx5s2hc8Xgsp4rEuTwnQ8UGRT4ma4tjlf+1i3YHA=` (EdDSA / ed25519 public key)
- `SUAutomaticallyUpdate` = `false` — never silently apply.
- `SUEnableAutomaticChecks` = `true` — check in the background.
- `SUScheduledCheckInterval` = `14400` seconds (4 hours).

`appcast.xml` is checked in at the repo root and consumed by Sparkle 2.6+ (pinned in `Package.swift (~L10)`).

## Notarize + DMG
`build.sh (~L160)` — guarded by `[ "$NOTARIZE" = true ] && [[ "$SIGN_ID" == *"Developer ID"* ]]`.

1. `ditto -c -k --keepParent` zips the `.app`.
2. `xcrun notarytool submit "$ZIP_PATH" --keychain-profile "CodeIsland" --wait` and grep for `status: Accepted`.
3. On success: `xcrun stapler staple "$APP_BUNDLE"`.
4. On failure: prints `xcrun notarytool log <submission-id> --keychain-profile CodeIsland` hint and exits non-zero.
5. `create-dmg` builds `.build/release/CodeIsland.dmg` (drag-to-Applications layout, 600×400 window, no internet-enable).
6. DMG is `codesign --force --sign "$SIGN_ID"`-ed, then notarized + stapled. A DMG notarization failure is non-fatal (the `.app` is already notarized).

## Android watch build
`build.sh::build_watch (~L53)` — runs `./android-watch/gradlew -p android-watch testDebugUnitTest` then `assembleDebug`. Output: `android-watch/app/build/outputs/apk/debug/app-debug.apk`.

## Local dev loop
After a successful `./build.sh`:

```sh
open .build/release/CodeIsland.app
```

The script also prints `Run: open .build/release/CodeIsland.app` (`build.sh (~L200)`).

## Troubleshooting
- **`Missing Sparkle.framework at .build/artifacts/sparkle/...`** — SPM hasn't downloaded the binary artifact yet. Run `swift package resolve` once, or just re-run `./build.sh` (the first build will fetch it).
- **`No developer certificate found, using ad-hoc signing...`** — script keeps going with `SIGN_ID="-"`. The resulting `.app` runs locally but cannot be distributed and cannot be notarized.
- **Notarization rejected** — script aborts with `ERROR: Notarization failed.` and prints the exact `xcrun notarytool log <submission-id> --keychain-profile CodeIsland` command to run for the rejection report.
- **`Missing executable Gradle wrapper`** — `--watch` / `--with-watch` was passed but `android-watch/gradlew` isn't executable; `chmod +x android-watch/gradlew`.

## See also
- [architecture.md](./architecture.md)
- [conventions.md](./conventions.md)

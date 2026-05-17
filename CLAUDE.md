# aw-android-realtime — Project Rules

## Coding Principles

1. **Think before coding** — Don't assume. Don't hide confusion. Surface tradeoffs. State assumptions explicitly and ask if uncertain.
2. **Simplicity first** — Minimum code that solves the problem. No features beyond what was requested. No abstractions for single-use code. No error handling for impossible scenarios.
3. **Surgical changes** — Touch only what you must. Don't refactor working code. Match existing style. Remove only what your changes made unused.
4. **Goal-driven execution** — Define success criteria. Loop until verified. For multi-step tasks, outline a brief plan with steps and verification checks.

## Build Commands

```bash
export JAVA_HOME="$(pwd)/jdk-17.0.18+8"
export ANDROID_HOME="$(pwd)/android-sdk"
export PATH="$JAVA_HOME/bin:$ANDROID_HOME/platform-tools:$ANDROID_HOME/cmdline-tools/latest/bin:$PATH"

# Batch version
./gradlew assembleDebug
adb install -r mobile/build/outputs/apk/debug/mobile-debug.apk

# Real-time version
./gradlew assembleRealtimeDebug
adb install -r mobile/build/outputs/apk/realtime/debug/mobile-realtime-debug.apk
```

## Architecture

- **Batch mode** (`ActivityWatch`): UsageStatsManager → WorkManager (15min) → HTTP POST → remote server
- **Real-time mode** (`ActivityWatch 实时版`): AccessibilityService → instant app switch detection → HTTP POST → remote server
- Both use `RustInterface.kt` as HTTP client with Basic Auth support
- Both send to remote ActivityWatch server via `POST /api/0/buckets/{id}/heartbeat?pulsetime={seconds}`

## Key Files

| File | Purpose |
|------|---------|
| `RustInterface.kt` | HTTP client (no JNI), Basic Auth, auto `http://` prefix |
| `ActivityWatcher.kt` | Real-time: AccessibilityService, app switch detection, 60s periodic refresh |
| `AfkWatcher.kt` | Real-time: screen on/off AFK detection (dynamic BroadcastReceiver) |
| `HeartbeatWorker.kt` | Batch: WorkManager periodic background sync |
| `UsageStatsWatcher.kt` | Batch: UsageStatsManager data collection |
| `AWPreferences.kt` | SharedPreferences: URL, username, password |
| `MainActivity.kt` | Remote server config dialog (URL/user/pass) |
| `WebUIFragment.kt` | WebView with HTTP Basic Auth challenge handling |

## Red Lines

- **Never ask user to run adb/curl/logcat** — always use Bash tool with PATH setup
- **Network requests must be on background thread** — `onAccessibilityEvent` runs on main thread
- **SCREEN_OFF/ON must be registered dynamically** — Android 8.0+ blocks static registration in Manifest
- **Skip list is user-configurable** — stored in SharedPreferences, managed via app UI (☰ → Skip List), never hardcode filter packages

## Bucket Names

| Bucket | Mode | Type |
|--------|------|------|
| `aw-watcher-android-plus` | Batch | currentwindow |
| `aw-watcher-android-plus-unlock` | Batch | os.lockscreen.unlocks |
| `aw-watcher-android-realtime` | Real-time | currentwindow |
| `aw-watcher-android-realtime-afk` | Real-time | afkstatus |

## Server

- Default: `http://127.0.0.1:5600` (local)
- User configurable via app UI (URL + username + password)
- nginx reverse proxy with Basic Auth supported

## Deep Docs

- [HANDOVER.md](HANDOVER.md) — Full handover manual with architecture, build guide, diagnostics
- [README.md](README.md) — English feature overview
- [README_zh.md](README_zh.md) — Chinese feature overview

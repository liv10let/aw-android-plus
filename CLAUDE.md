# aw-android-plus — Project Rules

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
- **Filter list in ActivityWatcher** — must always include: 传送门, 个人助理, 桌面, 微信输入法, 搜索框, 系统界面组件

## Filter List (ActivityWatcher.kt)

```kotlin
val skipPackages = setOf(
    "com.miui.contentextension",   // 传送门
    "com.miui.personalassistant",  // 个人助理（负一屏）
    "com.miui.home",               // 桌面
    "com.tencent.wetype",          // 微信输入法
    "com.android.quicksearchbox",  // 系统搜索框
    "miui.systemui.plugin"         // MIUI 系统界面组件
)
```

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

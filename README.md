# aw-android-plus

> **[中文版 README](README_zh.md)**

A fork of [ActivityWatch/aw-android](https://github.com/ActivityWatch/aw-android) with **remote HTTP forwarding**, **real-time monitoring**, **AFK detection**, and **nginx Basic Auth** support.

---

## Features

### Two Monitoring Modes

| Mode | App Name | Latency | Method | Battery |
|------|----------|---------|--------|---------|
| **Batch** | ActivityWatch | 1~2 hours | UsageStatsManager + WorkManager (every 15min) | Low |
| **Real-time** | ActivityWatch 实时版 | ~100ms | AccessibilityService | Medium |
| **Power-save** | ActivityWatch 省电版 | 15 minutes | WorkManager + UsageStatsManager | Lowest |

Both apps can be installed side-by-side on the same device (different package names).

### Core Features

- **Pure Remote Forwarding** — Data sent via HTTP to a remote ActivityWatch server, no local storage
- **Real-time Monitoring** — AccessibilityService detects app switches with ~100ms latency
- **60s Periodic Refresh** — Heartbeats sent every 60s even without app switch, for long sessions
- **AFK Detection** — Monitors screen on/off, sends afk/not-afk events (bucket: `aw-watcher-android-realtime-afk`)
- **HTTP Basic Auth** — Supports nginx reverse proxy with username/password
- **Auto URL Prefix** — Automatically prepends `http://` if missing
- **Configurable Skip List** — Users can add/remove filtered package names via app UI (☰ → Skip List)
- **Dynamic WebUI** — Embedded WebUI shows remote dashboard
- **Native Toolbar Menu** — Tap ☰ to open navigation drawer

---

## Bucket Names

| Bucket | Data Source |
|--------|-----------|
| `aw-watcher-android-plus` | Batch: app usage (UsageStatsManager) |
| `aw-watcher-android-plus-unlock` | Batch: screen unlock events |
| `aw-watcher-android-realtime` | Real-time: app switches (AccessibilityService) |
| `aw-watcher-android-realtime-afk` | Real-time: AFK status (screen on/off) |

---

## Modified Files

| File | Changes |
|------|---------|
| `RustInterface.kt` | HTTP client (removed JNI); Basic Auth support; auto `http://` prefix |
| `ActivityWatcher.kt` | **New**: AccessibilityService for real-time app monitoring |
| `AfkWatcher.kt` | **New**: Screen on/off AFK detection via BroadcastReceiver |
| `HeartbeatWorker.kt` | **New**: WorkManager-based background data sync |
| `UsageStatsWatcher.kt` | Bucket renamed; lastUpdated fix; WorkManager periodic task |
| `MainActivity.kt` | Remote server dialog with URL/username/password fields |
| `AWPreferences.kt` | Store URL, username, password in SharedPreferences |
| `WebUIFragment.kt` | Handle HTTP Basic Auth challenges in WebView |
| `build.gradle` | Added WorkManager dependency; `realtime` flavor |
| `AndroidManifest.xml` | Registered ActivityWatcher service |
| `accessibility_service_config_realtime.xml` | **New**: AccessibilityService config |
| `network_security_config.xml` | Cleartext HTTP allowed |

---

## Build

### Environment (Windows 11, no Android Studio)

| Component | Path |
|-----------|------|
| JDK 17 | `./jdk-17.0.18+8` |
| Android SDK | `./android-sdk` |

### Build Commands

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

---

## Configuration

### 1. Prepare Remote Server

You need a running ActivityWatch server (`aw-server-rust` or `aw-server`).

For nginx reverse proxy with Basic Auth:
```nginx
server {
    listen 5601;
    auth_basic "ActivityWatch";
    auth_basic_user_file /etc/nginx/.htpasswd;
    proxy_pass http://127.0.0.1:5600;
}
```

### 2. Configure in App

1. Open the app, tap **☰** → **Remote Server**
2. Enter URL (e.g. `http://your-server:5601`), username, password
3. Tap **Save**

### 3. Enable Accessibility (Real-time version only)

Settings → Accessibility → Find **ActivityWatch Realtime** → Enable

---

## Debugging

```bash
# Batch version logs
adb logcat -s RustInterface:D UsageStatsWatcher:D

# Real-time version logs
adb logcat -s ActivityWatcher:D RustInterface:D

# Verify remote forwarding
curl -u username:password http://your-server:5601/api/0/buckets
```

---

## Known Limitations

- Batch mode: app usage data delayed 1~2 hours (Android system limitation)
- Real-time mode: requires AccessibilityService permission (manual enable)
- Real-time mode: higher battery usage than batch mode
- Network disconnection causes data loss (fire-and-forget, no local cache)
- MIUI may kill AccessibilityService; set battery policy to "Unrestricted"

---

## Original Project

https://github.com/ActivityWatch/aw-android

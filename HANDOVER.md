# aw-android-plus / 交接手册 (Handover Manual)

> 本文档供后续模型快速理解项目上下文。
> **最后更新**: 2026-05-14
> **当前状态**: 纯远程转发 + 批量/实时双模式 + nginx Basic Auth

---

## 1. 项目概况

- **原项目**: https://github.com/ActivityWatch/aw-android
- **用户仓库**: https://github.com/liv10let/aw-android-plus
- **核心目标**: 将 aw-android 改为纯远程 HTTP 转发，支持批量和实时两种监控模式
- **工作目录**: `e:\Project_workspace\aw-android`

### 1.1 两种监控模式

| 模式 | App 名称 | 包名 | Bucket | 延迟 | 触发方式 |
|------|----------|------|--------|------|---------|
| 批量 | ActivityWatch | `.debug` / `.realtime.debug` | `aw-watcher-android-plus` | 1~2h | WorkManager 每15分钟 |
| 实时 | ActivityWatch 实时版 | `.realtime.debug` | `aw-watcher-android-realtime` | ~100ms | AccessibilityService |

实时版基于 AccessibilityService 监听 `TYPE_WINDOW_STATE_CHANGED` 事件，每次 app 切换立即记录，并每 60 秒定时刷新当前 app 的使用时长。

---

## 2. 核心改动

### 2.1 RustInterface.kt — HTTP 客户端 + Basic Auth

移除所有 JNI 代码，改为纯 HTTP 客户端：
- `httpGet()` / `httpPost()` 辅助方法，自动携带 Basic Auth 头
- `setAuthHeader()` 从 SharedPreferences 读取 username/password，构造 `Authorization: Basic base64(user:pass)`
- `getServerUrl()` 自动补全 `http://` 前缀
- 远程地址从 `AWPreferences.getRemoteServerUrl()` 读取

**关键代码位置**: [RustInterface.kt](mobile/src/main/java/net/activitywatch/android/RustInterface.kt)

### 2.2 ActivityWatcher.kt — 实时监控（新增）

AccessibilityService，监听所有 app 切换：
- `onAccessibilityEvent()` 检测 `TYPE_WINDOW_STATE_CHANGED`
- 过滤列表（6 个包名）：传送门、个人助理、桌面、微信输入法、搜索框、系统界面组件
- AFK 状态下暂停记录（`if (AfkWatcher.isAfk) return`）
- `logAppUsage()` 在后台线程执行网络请求（`executor.execute`）
- 每 60 秒定时刷新当前 app 使用时长（`scheduler.scheduleAtFixedRate`）
- 使用 `pulsetime=60` 让服务器自动合并相同事件

**关键代码位置**: [ActivityWatcher.kt](mobile/src/main/java/net/activitywatch/android/watcher/ActivityWatcher.kt)

### 2.2b AfkWatcher.kt — AFK 检测（新增）

监听屏幕开关，发送 afk/not-afk 事件：
- **必须动态注册**（Android 8.0+ 不允许在 Manifest 中静态声明 SCREEN_OFF/ON）
- 由 ActivityWatcher 在 `onServiceConnected` 时调用 `afkWatcher.register()`
- 屏幕关闭 → `{"status": "afk"}`，屏幕打开 → `{"status": "not-afk"}`
- Bucket: `aw-watcher-android-realtime-afk`（type: `afkstatus`）
- 使用 `PowerManager.isInteractive` 检测初始屏幕状态

**关键代码位置**: [AfkWatcher.kt](mobile/src/main/java/net/activitywatch/android/watcher/AfkWatcher.kt)

### 2.3 HeartbeatWorker.kt — WorkManager 后台采集（新增）

替代原来的 AsyncTask：
- `PeriodicWorkRequest` 每 15 分钟执行一次
- `OneTimeWorkRequest` 用于打开 app 时手动触发
- Worker 在独立后台进程中执行，app 切走后仍会继续

**关键代码位置**: [HeartbeatWorker.kt](mobile/src/main/java/net/activitywatch/android/watcher/HeartbeatWorker.kt)

### 2.4 UsageStatsWatcher.kt — 批量采集

- Bucket 改名：`aw-watcher-android-plus` / `aw-watcher-android-plus-unlock`
- `lastUpdated` 修复：服务器数据超过 7 天时，回退到 `now - 1hour`
- `setupAlarm()` 改为 WorkManager `PeriodicWorkRequest`
- `sendHeartbeats()` 改为 WorkManager `OneTimeWorkRequest`

### 2.5 MainActivity.kt — 配置 UI

- `showRemoteServerDialog()` 增加 username/password 输入框
- 密码输入框使用 `TYPE_TEXT_VARIATION_PASSWORD` 隐藏显示
- `baseURL` 动态读取远程地址

### 2.6 WebUIFragment.kt — WebView Basic Auth

- `onReceivedHttpAuthRequest()` 回调处理 nginx 401 挑战
- 自动用 SharedPreferences 中的 username/password 认证

### 2.7 AWPreferences.kt — 持久化配置

存储三个字段：
- `remoteServerUrl` — 服务器地址
- `remoteServerUsername` — Basic Auth 用户名
- `remoteServerPassword` — Basic Auth 密码

### 2.8 build.gradle — realtime flavor

```groovy
flavorDimensions "mode"
productFlavors {
    realtime {
        dimension "mode"
        applicationIdSuffix ".realtime"
        resValue "string", "app_name", "ActivityWatch 实时版"
    }
}
```

构建命令：
- 批量版：`./gradlew assembleDebug`
- 实时版：`./gradlew assembleRealtimeDebug`

---

## 3. 已解决的问题

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 数据丢失 95.2% | `lastUpdated` 被服务器旧数据拉回 9 天前 | 超过 7 天使用 `now - 1hour` |
| 找不到 Remote Server 菜单 | Toolbar 被注释 | 启用 Toolbar + ActionBarDrawerToggle |
| Cleartext HTTP 被拦截 | Android 9+ 默认禁止 | `network_security_config.xml` 允许 |
| `no protocol` 错误 | URL 缺少 `http://` | `getServerUrl()` 自动补全 |
| WebView 显示 401 | WebView 不携带 Basic Auth | `onReceivedHttpAuthRequest()` 处理 |
| AsyncTask 被 MIUI 杀掉 | 后台任务不可靠 | 改用 WorkManager |
| AccessibilityService 崩溃 | `onCreate` 中执行网络请求 | 改为 `onServiceConnected` + 后台线程 |
| 传送门误报 | MIUI 浮窗触发 `TYPE_WINDOW_STATE_CHANGED` | 过滤 6 个包名（传送门、个人助理、桌面、微信输入法、搜索框、系统界面组件） |
| 长时间同一 app 不上报 | 只在切换时记录 | 每 60 秒定时刷新 |
| SCREEN_OFF/ON 静态注册不触发 | Android 8.0+ 限制 | 改为动态注册（ActivityWatcher.onServiceConnected） |
| 锁屏后仍记录 app 切换 | 无 AFK 检测 | 新增 AfkWatcher，屏幕关闭时暂停记录 |

---

## 4. 已知问题

1. **实时模式电量消耗**：AccessibilityService 持续运行，比批量模式费电
2. **MIUI 后台限制**：可能杀掉 AccessibilityService，需设置省电策略为"无限制"
3. **网络断开丢数据**：fire-and-forget，无本地缓存
4. **数据碎片化**：批量模式的 UsageStatsManager 事件本身很碎

---

## 5. 构建指南

### 5.1 环境（Windows 11，无 Android Studio）

| 组件 | 路径 |
|------|------|
| JDK 17 | `./jdk-17.0.18+8` |
| Android SDK | `./android-sdk` |

### 5.2 构建命令

```bash
export JAVA_HOME="$(pwd)/jdk-17.0.18+8"
export ANDROID_HOME="$(pwd)/android-sdk"
export PATH="$JAVA_HOME/bin:$ANDROID_HOME/platform-tools:$ANDROID_HOME/cmdline-tools/latest/bin:$PATH"

# 批量版
./gradlew assembleDebug
adb install -r mobile/build/outputs/apk/debug/mobile-debug.apk

# 实时版
./gradlew assembleRealtimeDebug
adb install -r mobile/build/outputs/apk/realtime/debug/mobile-realtime-debug.apk
```

---

## 6. 快速诊断

### 6.1 批量版日志

```bash
adb logcat -s RustInterface:D UsageStatsWatcher:D HeartbeatWorker:D
```

### 6.2 实时版日志

```bash
adb logcat -s ActivityWatcher:D RustInterface:D
```

### 6.3 验证服务器数据

```bash
curl -u username:password http://your-server:5601/api/0/buckets
curl -u username:password http://your-server:5601/api/0/buckets/aw-watcher-android-realtime/events?limit=5
```

### 6.4 验证无障碍服务状态

```bash
adb shell "settings get secure enabled_accessibility_services"
```

应包含 `net.activitywatch.android.realtime.debug/net.activitywatch.android.watcher.ActivityWatcher`

---

## 7. 文件清单

```
M  .gitignore
M  README.md
M  README_zh.md
M  HANDOVER.md
M  mobile/build.gradle
M  mobile/src/main/AndroidManifest.xml
M  mobile/src/main/java/net/activitywatch/android/AWPreferences.kt
M  mobile/src/main/java/net/activitywatch/android/MainActivity.kt
M  mobile/src/main/java/net/activitywatch/android/RustInterface.kt
M  mobile/src/main/java/net/activitywatch/android/fragments/WebUIFragment.kt
M  mobile/src/main/java/net/activitywatch/android/watcher/AlarmReceiver.kt
A  mobile/src/main/java/net/activitywatch/android/watcher/ActivityWatcher.kt
A  mobile/src/main/java/net/activitywatch/android/watcher/AfkWatcher.kt
A  mobile/src/main/java/net/activitywatch/android/watcher/HeartbeatWorker.kt
M  mobile/src/main/java/net/activitywatch/android/watcher/UsageStatsWatcher.kt
M  mobile/src/main/res/layout/activity_main.xml
M  mobile/src/main/res/layout/app_bar_main.xml
M  mobile/src/main/res/menu/activity_main_drawer.xml
M  mobile/src/main/res/values/strings.xml
M  mobile/src/main/res/xml/network_security_config.xml
A  mobile/src/main/res/xml/accessibility_service_config_realtime.xml
```

---

## 8. 演进记录

1. **阶段一：双写模式** — 本地 JNI + 异步 HTTP 远程。远程数据不完整。
2. **阶段二：纯远程单写** — 移除 JNI，纯 HTTP。解决数据空洞。
3. **阶段三：回退本地** — Tailscale 网络不稳定，暂时回退。
4. **阶段四：公网 IP 恢复双写** — 改用公网 IP，数据正常。
5. **阶段五：纯远程模式** — 移除本地存储，纯远程转发。
6. **阶段六：WorkManager** — AsyncTask 改为 WorkManager，后台更可靠。
7. **阶段七：Basic Auth** — 支持 nginx 反代 + 用户名密码认证。
8. **阶段八：实时版** — 新增 AccessibilityService 实时监控，100ms 延迟。
9. **阶段九：AFK 检测** — 新增 AfkWatcher，监听屏幕开关，锁屏时暂停记录。

---

*文档最后更新: 2026-05-14*
*当前状态: 纯远程转发 + 批量/实时双模式 + nginx Basic Auth*

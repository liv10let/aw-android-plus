# aw-android-plus

> **[English README](README.md)**

基于 [ActivityWatch/aw-android](https://github.com/ActivityWatch/aw-android) 的 fork，增加了**远程 HTTP 转发**、**实时监控**、**AFK 检测**和 **nginx Basic Auth** 支持。

---

## 功能说明

### 两种监控模式

| 模式 | App 名称 | 延迟 | 采集方式 | 电量消耗 |
|------|----------|------|---------|---------|
| **批量模式** | ActivityWatch | 1~2 小时 | UsageStatsManager + WorkManager（每15分钟） | 低 |
| **实时模式** | ActivityWatch 实时版 | ~100ms | AccessibilityService | 中等 |
| **省电模式** | ActivityWatch 省电版 | 15 分钟 | WorkManager + UsageStatsManager | 最低 |

两个 App 可以同时安装在同一台手机上（包名不同）。

### 核心特性

- **纯远程转发** — 数据通过 HTTP 直接发送到远程 ActivityWatch 服务器，不在本地保存
- **实时监控** — AccessibilityService 检测 app 切换，延迟约 100ms
- **60 秒定时刷新** — 即使不切换 app，每分钟也会发送一次 heartbeat，确保长时使用也有数据
- **AFK 检测** — 监听屏幕开关，发送 afk/not-afk 事件（bucket: `aw-watcher-android-realtime-afk`）
- **HTTP Basic Auth** — 支持 nginx 反代 + 用户名密码认证
- **自动补全 URL** — 输入地址时自动补全 `http://` 前缀
- **可配置屏蔽列表** — 用户可在 app 里添加/删除屏蔽的包名（☰ → Skip List）
- **动态 WebUI** — 内嵌 WebUI 自动展示远程仪表盘
- **原生 Toolbar 菜单** — 点击 ☰ 打开导航抽屉

---

## Bucket 名称

| Bucket | 数据来源 |
|--------|---------|
| `aw-watcher-android-plus` | 批量模式：应用使用数据（UsageStatsManager） |
| `aw-watcher-android-plus-unlock` | 批量模式：屏幕解锁事件 |
| `aw-watcher-android-realtime` | 实时模式：应用切换事件（AccessibilityService） |
| `aw-watcher-android-realtime-afk` | 实时模式：AFK 状态（屏幕开关） |

---

## 改动文件

| 文件 | 改动说明 |
|------|---------|
| `RustInterface.kt` | HTTP 客户端（移除 JNI）；Basic Auth 支持；自动补全 `http://` |
| `ActivityWatcher.kt` | **新增**：AccessibilityService 实时监控 app 切换 |
| `AfkWatcher.kt` | **新增**：屏幕开关 AFK 检测（BroadcastReceiver） |
| `HeartbeatWorker.kt` | **新增**：WorkManager 后台数据同步 |
| `UsageStatsWatcher.kt` | Bucket 改名；lastUpdated 修复；WorkManager 定时任务 |
| `MainActivity.kt` | 远程服务器配置对话框（URL/用户名/密码） |
| `AWPreferences.kt` | 存储 URL、用户名、密码到 SharedPreferences |
| `WebUIFragment.kt` | 处理 WebView 中的 HTTP Basic Auth 挑战 |
| `build.gradle` | 添加 WorkManager 依赖；新增 `realtime` flavor |
| `AndroidManifest.xml` | 注册 ActivityWatcher 服务 |
| `accessibility_service_config_realtime.xml` | **新增**：AccessibilityService 配置 |
| `network_security_config.xml` | 允许明文 HTTP |

---

## 构建

### 环境（Windows 11，无 Android Studio）

| 组件 | 路径 |
|------|------|
| JDK 17 | `./jdk-17.0.18+8` |
| Android SDK | `./android-sdk` |

### 构建命令

```bash
export JAVA_HOME="$(pwd)/jdk-17.0.18+8"
export ANDROID_HOME="$(pwd)/android-sdk"
export PATH="$JAVA_HOME/bin:$ANDROID_HOME/platform-tools:$ANDROID_HOME/cmdline-tools/latest/bin:$PATH"

# 批量版本
./gradlew assembleDebug
adb install -r mobile/build/outputs/apk/debug/mobile-debug.apk

# 实时版本
./gradlew assembleRealtimeDebug
adb install -r mobile/build/outputs/apk/realtime/debug/mobile-realtime-debug.apk
```

---

## 配置

### 1. 准备远程服务器

需要一个运行中的 ActivityWatch 服务器（`aw-server-rust` 或 `aw-server`）。

如果需要 nginx 反代 + Basic Auth：
```nginx
server {
    listen 5601;
    auth_basic "ActivityWatch";
    auth_basic_user_file /etc/nginx/.htpasswd;
    proxy_pass http://127.0.0.1:5600;
}
```

### 2. 在 App 中配置

1. 打开 App，点击 **☰** → **Remote Server**
2. 填写 URL（如 `http://your-server:5601`）、用户名、密码
3. 点击 **Save**

### 3. 开启无障碍权限（仅实时版本）

设置 → 无障碍 → 找到 **ActivityWatch Realtime** → 开启

---

## 调试

```bash
# 批量版本日志
adb logcat -s RustInterface:D UsageStatsWatcher:D

# 实时版本日志
adb logcat -s ActivityWatcher:D RustInterface:D

# 验证远程服务器
curl -u username:password http://your-server:5601/api/0/buckets
```

---

## 已知限制

- 批量模式：应用使用数据延迟 1~2 小时（Android 系统限制）
- 实时模式：需要手动开启无障碍权限
- 实时模式：电量消耗比批量模式略高
- 网络断开时数据会丢失（fire-and-forget，无本地缓存）
- MIUI 可能会杀掉 AccessibilityService，需设置省电策略为"无限制"

---

## 原项目

https://github.com/ActivityWatch/aw-android

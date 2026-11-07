# read_this_if_you_are_agent

写给第一次进入这个 fork 的 agent 和人类维护者。仓库自带的 `AGENTS.md` 与 `.agents/*.md` 描述的是**上游**工程约定，仍然有效；这个文件只补充**本 fork 独有的、看起来像 bug 但其实是有意为之**的东西。

先说结论：这里只有 3 处 fork 改动，都是为了「在 Android 上用自己的包名、自己出 APK」。除此之外的代码请当作上游代码对待，不要在 fork 里顺手「修复」它们。

---

## 1. 仓库关系

| | |
|---|---|
| fork | `Temsir/FlClash`（public，default branch `main`） |
| upstream | `chen08209/FlClash` |
| 改动范围 | 仅 Android 打包；桌面端未改 |
| 出包方式 | GitHub Actions 手动触发，不依赖本地环境 |

本地可能还没有 `upstream` remote，需要时加一次：

```powershell
git remote add upstream https://github.com/chen08209/FlClash.git
```

> 推送用的是 SSH URL（`git@github.com:Temsir/FlClash.git`），而 `origin` 是 HTTPS，所以推完 `git status` 可能显示 `ahead N`，`git fetch origin` 一下就正常。这不是提交丢失。
> 当前 token 没有 `workflow` scope，改 `.github/` 下的文件必须走 SSH 推送。

---

## 2. 三处 fork 改动（不要"修正"）

### 2.1 只改了 `applicationId`

`android/app/build.gradle.kts`：

```kotlin
namespace = "com.follow.clash"   // 未改
applicationId = "com.follow.net" // 唯一改动
```

### 2.2 `namespace`、Dart `packageName`、Kotlin `Components.PACKAGE_NAME` 全部保持 `com.follow.clash`

这些**故意**没跟着改，因为它们不是「应用标识」，而是：

- `lib/common/constant.dart` 的 `const packageName = 'com.follow.clash'` → 用作 MethodChannel 前缀，如 `$packageName/app`（`lib/plugins/*.dart`）；
- `android/common/.../Components.kt` 的 `PACKAGE_NAME` → 拼 `ComponentName`（`MainActivity`、`QuickActionActivity`、`ServiceBroadcastReceiver`）和各插件 channel 名。

动了它们会同时打断 Flutter↔Android 通道、组件启动和通知/快捷方式跳转。真正的运行时包名由 `GlobalState.packageName`（`context.packageName`）提供，跟这些常量无关。

**所以 `rg "com.follow.clash"` 搜出一堆结果是对的，不是漏改。**

### 2.3 release 的签名是条件式的

同一个 gradle 文件里：

```kotlin
if (hasReleaseSigning) {
    signingConfig = signingConfigs.getByName("release")
} else {
    signingConfig = signingConfigs.getByName("debug")
    applicationIdSuffix = ".dev"
}
```

`hasReleaseSigning` = `android/app/keystore.jks` 存在 **且** `android/local.properties` 里三个属性齐全。

| 情况 | 包名 | 签名 |
|---|---|---|
| 有 keystore + 密码 | `com.follow.net` | 固定 release key |
| 缺任意一项 | `com.follow.net.dev` | 本机/runner 自动生成的 debug key |

---

## 3. 构建：只走 CI

### 3.1 为什么不能靠上游 workflow

`.github/workflows/build.yaml` 是上游的，它的 Android release job 带条件：

```yaml
if: github.repository == 'chen08209/FlClash' && github.event_name == 'push' && startsWith(github.ref, 'refs/tags/v')
```

在 fork 里**永远不会执行**。不要为了出包去改它，也不要指望打 tag 触发。

### 3.2 fork 自己的 workflow

`.github/workflows/android-apk.yml`，仅 `workflow_dispatch`：

- 输入：`app_env`（`stable`/`pre`/`dev`，默认 `stable`）、`abi`（`arm64`/`arm`/`amd64`，默认 `arm64`）；
- 产物：artifact `flclash-<abi>-<env>`，内含 `app-release.apk`，保留 14 天；
- 单次约 14 分钟。

构建命令等价于：

```
flutter build apk --release --target-platform android-arm64 --dart-define=APP_ENV=stable
```

> 注意：直接从命令行跑 `dart setup.dart` 时 `--env` 默认是 `pre`（会带 PRE 横幅）。CI 走 `--dart-define=APP_ENV=stable`，所以 CI 产物没有横幅。

### 3.3 需要 4 个仓库 secret

`Settings → Secrets and variables → Actions`：

| Secret | 内容 |
|---|---|
| `KEYSTORE_BASE64` | `android/app/keystore.jks` 的 base64（单行） |
| `KEYSTORE_PASSWORD` | keystore 密码 |
| `KEY_ALIAS` | `flclash` |
| `KEY_PASSWORD` | 同 keystore 密码 |

workflow 在 `Build APK` 前把它们还原成 `android/app/keystore.jks` 和 `android/local.properties`。**secret 没配不会失败**，只会打一条 `::warning::` 并产出 `.dev` 的 debug 包——看到带 `.dev` 的包名，先去查 secret。

密码本身不写进仓库。下面两处可以找到（都已被 git 忽略）：

- GitHub 仓库 secrets；
- 本机 `C:\Users\temsir\.flclash-signing\`（`keystore.jks` + `credentials.txt`）以及 `android/local.properties`。

> ⚠️ keystore 丢失 = 已安装用户永远无法升级，只能卸载重装。改动/迁移这台机器前先确认备份还在。

### 3.4 workflow 里钉死的版本要跟着上游走

`android-apk.yml` 里的 `flutter-version: 3.47.6`、Go `1.27.1`、`ndk-version: r28c`、`CA_BUNDLE` 及其 SHA256，都是照抄 `.github/workflows/build.yaml` 顶部 `FLUTTER_VERSION` / `GO_VERSION` / `NDK_VERSION` 和 `CA_BUNDLE` 步骤的。

每次 sync 上游后顺手对一眼：上游升了版本、而这里没升，会出现「代码要求新版 Flutter 但 CI 装的是旧版」这类失败。尤其别漏 `CA_BUNDLE`——它会在构建时覆盖上游的证书包，用过期 bundle 可能导致部分站点 TLS 校验失败。

---

## 4. 签名与升级的历史包袱

- 2026-11-07 之前的 CI 产物没有固定 keystore，用的是 GitHub runner 每次随机生成的 debug key，**不同 run 之间签名不同，互相不能覆盖安装**。
- 同一天加入固定 keystore 后，包名从 `com.follow.net.dev` 变成 `com.follow.net`，所以**跨越那次改动需要卸载一次旧包**（丢应用内数据）。此后同一签名的包可以一直覆盖升级。
- `versionCode` 来自 `pubspec.yaml` 的 `version: 0.9.3+2026110401` 的 build number（`2026110401`），上游发版会递增，升级条件正常。
- 本地 `flutter build apk` 现在也会读取 `android/local.properties` 里的签名配置，产出与 CI 同签名的 `com.follow.net`。

---

## 5. 同步上游

```powershell
git fetch origin
git fetch upstream
git merge upstream/main
```

冲突面积很小，通常只出现在 `android/app/build.gradle.kts` 的 `applicationId` / `buildTypes` 一带。解冲突原则：**保留 `applicationId = "com.follow.net"` 和无签名时的 `.dev` 回退分支，其余全取上游。**

合并后检查两件事：

1. `android/app/build.gradle.kts` 里签名逻辑是否被上游改写/挪动；
2. `build.yaml` 顶部的 `FLUTTER_VERSION` / `GO_VERSION` / `NDK_VERSION`、以及 `CA_BUNDLE` 是否变了 → 同步到 `android-apk.yml`。

提交信息里带 `[skip ci]` 是刻意的：上游 `build.yaml` 会对任意分支的 push 触发一堆 dart/desktop 检查，在 fork 里纯属浪费 Actions 额度。

---

## 6. 万不得已要在本机 Windows 构建

通常不需要。真要做，注意这些环境细节（缓存被清理过，首次构建会重新下载几个 GB）：

| 项目 | 位置/值 |
|---|---|
| Flutter | `D:\Software\flutter` |
| Android SDK / NDK | `D:\Software\Android\Sdk`，NDK `28.2.13676358` |
| JDK | `D:\Software\jdk\jdk-17.0.12`（`JAVA_HOME` 未写入环境变量，`gradlew.bat` 会报错） |
| Go | `D:\Software\go\bin\go.exe`（不在 PATH） |
| Rust | 默认 `stable-x86_64-pc-windows-msvc`，另有 `1.99.0-x86_64-pc-windows-gnu` |

已知的 Windows 专有坑（`plugins/rust_api/hook/build.dart` 在 Windows 上只查 `lib`/`lib64` 的 `libclang`）：

1. `python -m pip install libclang==18.1.1`；
2. 把包里的 `libclang.dll` 复制到 NDK 的 `toolchains/llvm/prebuilt/windows-x86_64/lib/`；
3. 建 junction：`~/.cargo/registry/src/index.crates.io-*/rquickjs-sys-*/lib/clang/18` → NDK 的 `lib/clang/19`；
4. `rustup target add aarch64-linux-android`。

上游升级 `rquickjs-sys` 或 NDK 版本后，第 2、3 步需要重做。CI（Ubuntu）不需要这些。

构建前记得 `$env:JAVA_HOME = "D:\Software\jdk\jdk-17.0.12"`；`android/gradlew.bat --stop` 可以停掉常驻 daemon。

---

## 7. 快速排错

| 现象 | 原因 |
|---|---|
| 包名是 `com.follow.net.dev` | 签名 secret 没配 / `local.properties` 缺项，走了 debug 回退 |
| 新 APK 装不上，提示签名冲突 | 装的是 2026-11-07 之前的随机 debug 签名包，卸载一次即可 |
| `rg "com.follow.clash"` 到处都有 | 正常，`namespace`/通道名/组件名故意保留 |
| `git status` 显示 ahead | 用 SSH URL 推送未更新 HTTPS tracking ref，`git fetch origin` |
| 改了 `.github/` 推不上去 | token 缺 `workflow` scope，改用 SSH |
| CI 报 Flutter/Go/NDK 版本问题 | `android-apk.yml` 里钉的版本落后于上游 `build.yaml` |
| Actions 列表里没有 `Android APK` | workflow 文件不在默认分支上，或 YAML 写坏了 |

---

## 8. 关键文件

- `android/app/build.gradle.kts` —— 唯一的包名改动 + 签名回退逻辑
- `android/app/keystore.jks` —— release keystore（git 忽略，`android/.gitignore` 的 `**/*.jks`）
- `android/local.properties` —— 本机签名参数（git 忽略）
- `.github/workflows/android-apk.yml` —— fork 的手动构建流程
- `.github/workflows/build.yaml` —— 上游流程，维持原样
- `lib/common/constant.dart` —— Dart 侧 `packageName` 常量（勿改）
- `android/common/src/main/java/com/follow/clash/common/Components.kt` —— `PACKAGE_NAME` 常量（勿改）

# 创维电视专属适配记录与维护指南

本仓库为 [pengchujin/aerial-tv](https://github.com/pengchujin/aerial-tv) 针对创维电视（Skyworth 8A002_A3 / T963 芯片 / Android 9 SDK 28）的深度机型适配分支。

---

## 一、本次定制修改记录

### 1. 遥控器音量失控（自动加减到极限值）修复
- **故障根因**：
  在创维固件上，无障碍服务启用 `FLAG_REQUEST_FILTER_KEY_EVENTS`（挂载系统 `AccessibilityInputFilter`）后，固件底层丢失了遥控器按键抬起（`ACTION_UP`）事件，应用层持续收到源源不断的 `ACTION_DOWN` 重复事件，导致按一次音量加/减即狂飙至 0 或 100（官方 Issue #448、#451）。
- **修改文件**：
  `app/src/main/java/app/orchard/tv/guard/HomeGuardService.kt`
- **实现方案**：
  在 `isSkyworthDevice` 判据中识别创维机型，并加入 `keysPassThrough` 白名单：
  ```kotlin
  private val isSkyworthDevice by lazy {
      Build.MANUFACTURER.equals("Skyworth", ignoreCase = true) ||
          Build.BRAND.equals("Skyworth", ignoreCase = true) ||
          Build.BRAND.equals("Coocaa", ignoreCase = true) ||
          Build.MODEL.contains("8A002", ignoreCase = true) ||
          Build.DEVICE.contains("T963", ignoreCase = true)
  }

  private val keysPassThrough by lazy {
      isSkyworthDevice || VIDAA_PACKAGES.any { runCatching { packageManager.getPackageInfo(it, 0) }.isSuccess }
  }
  ```
  彻底关闭按键拦截 Flag，物理按键 100% 直通原生驱动；桌面接管交由稳定的窗口状态监听（`onAccessibilityEvent`）在后台守护拉回。

### 2. 物理 Home 键闪回酷开桌面根治
- **故障根因**：
  创维系统在处理物理遥控器 Home 键时，并非走 Android 原生选择器，而是硬编码读取私有系统属性 `persist.service.homepage.pkg` 和 `cls` 启动酷开大厅（`com.tianci.movieplatform`）。
- **生效命令（具备开机持久化）**：
  ```bash
  adb shell setprop persist.service.homepage.pkg app.orchard.tv
  adb shell setprop persist.service.homepage.cls app.orchard.tv.MainActivity
  ```
  设置后系统底层直接定向到 Aerial，源头上彻底消除原厂桌面闪屏。

### 3. 云端自动编译流水线
- **配置文件**：
  `.github/workflows/build.yml`
- **机制**：
  任何分支推送到本 Fork 仓库，GitHub Actions 将自动执行 Java 17 + Gradle 打包，自动产出 `aerial-guard-skyworth` 与 `aerial-standard-skyworth` 专属 Release APK。

### 4. 官方上游 PR 跟踪
- **PR 链接**：[pengchujin/aerial-tv#1124](https://github.com/pengchujin/aerial-tv/pull/1124)
- **PR 状态**：已提交等待上游合并。

---

## 二、后续跟进上游（Upstream）功能更新指南

当官方仓库有新功能或版本发布时，按照以下步骤同步：

### 场景 A：上游官方已合并 PR #1124（推荐）
此时官方版本已原生包含您的机型适配：
```bash
git checkout main
git fetch upstream
git merge upstream/main
git push origin main
```
或者直接在电视上使用官方 Release 安装包在线升级。

---

### 场景 B：上游尚未合并，但需要引入官方最新功能
使用 Git 变基（Rebase），将本地专属适配提交与 GitHub Actions 配置叠加在官方最新代码之上：

```bash
# 1. 确保处于本地 main 分支
git checkout main

# 2. 拉取官方最新代码
git fetch upstream

# 3. 将您的定制提交变基到官方 main 之上
git rebase upstream/main

# 4. 若出现代码冲突：
#    手动编辑冲突文件（通常只需保留上面的 isSkyworthDevice 逻辑）
#    git add <冲突文件>
#    git rebase --continue

# 5. 强制推送回您的 Fork 仓库
git push -f origin main
```

**推送完成后**：
1. 访问您的仓库 Actions 页面：`https://github.com/fxxisme/aerial-tv_copy/actions`
2. 等待云端自动构建完成（约 2 分钟）；
3. 下载最新的 `aerial-guard-skyworth` 产物 zip；
4. 解压并通过 ADB 安装到电视：
   ```powershell
   adb connect 192.168.1.9:5555
   adb install -r app-guard-release.apk
   adb disconnect 192.168.1.9:5555
   ```

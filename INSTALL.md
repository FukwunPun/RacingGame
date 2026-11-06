# 怎么把它装到安卓手机上

## 先明确一件事

`gyro_steer_app.zip` 里是**源代码工程**，不是安装包。安卓不能安装 zip，也不能安装 `.kt` / `.gradle` 这些文件。
必须先把它**编译成 `app-debug.apk`**，再把这个 apk 装进手机。

下面三条路线选一条，A 最省事。

---

## 路线 A：用 Android Studio 编译（推荐，图形界面）

**电脑要求**：Windows / macOS，8 GB 内存以上，能上网（首次要下 Gradle 和 SDK，约 1–2 GB）。

1. 解压 `gyro_steer_app.zip`，得到 `gyro_steer_app` 文件夹。
2. 装 Android Studio：<https://developer.android.com/studio>（一路默认下一步，首次启动会自己装 SDK）。
3. Android Studio 打开后选 **Open**，选中 `gyro_steer_app` 文件夹（注意是含 `settings.gradle.kts` 的那一层，不要选到 `app` 子目录）。
4. 等右下角 Gradle Sync 跑完（第一次 3–10 分钟）。若弹窗问 “Gradle wrapper 缺失 / 是否使用 Gradle 8.7”，选**使用/下载**。
5. 菜单 **Build → Build App Bundle(s) / APK(s) → Build APK(s)**。
6. 完成后右下角弹提示，点 **locate**，得到：
   `app\build\outputs\apk\debug\app-debug.apk`

这就是安装包（debug 签名，可直接装，不用额外签名）。

---

## 路线 B：命令行编译（不装 Android Studio）

需要 **JDK 17** + **Android SDK 命令行工具**。

```powershell
# 1) JDK 17
winget install --id EclipseAdoptium.Temurin.17.JDK
# 装完重开一个终端，确认：java -version 显示 17.x

# 2) Android SDK 命令行工具
# 下载 https://developer.android.com/studio#command-tools 的 Windows zip
# 解压到 %LOCALAPPDATA%\Android\Sdk\cmdline-tools\latest\
$env:ANDROID_HOME = "$env:LOCALAPPDATA\Android\Sdk"
$env:PATH = "$env:ANDROID_HOME\cmdline-tools\latest\bin;$env:PATH"

sdkmanager --licenses            # 一路 y
sdkmanager "platform-tools" "platforms;android-34" "build-tools;34.0.0"

# 3) Gradle 8.7
winget install --id Gradle.Gradle

# 4) 编译（在 gyro_steer_app 目录里执行）
cd gyro_steer_app
gradle wrapper --gradle-version 8.7
.\gradlew.bat assembleDebug
```

产物同样是 `app\build\outputs\apk\debug\app-debug.apk`。

---

## 路线 C：让 GitHub 替你编译（本机什么环境都不用装）

工程里已经放了 `.github/workflows/build-apk.yml`。

1. 注册/登录 GitHub，新建一个**私有或公开**仓库。
2. 把 `gyro_steer_app` 里面的内容（含 `.github`、`app`、`settings.gradle.kts`）上传到仓库根目录。
   - 网页上传：仓库页 → **Add file → Upload files** → 把解压后的文件拖进去 → Commit。
3. 进仓库 **Actions** 标签 → 左侧 `build-apk` → 右侧 **Run workflow** → 运行。
4. 等 3–6 分钟变绿，点开这次运行，页面底部 **Artifacts** 下载 `GyroSteer-debug-apk.zip`，解压出 `app-debug.apk`。

> 如果第一次构建报错，把 Actions 日志里 `Assemble debug APK` 那段的红色报错发我，我改代码。

---

## 装进手机

**方法 1：文件传输（最简单）**

1. 把 `app-debug.apk` 传到手机（USB 复制 / 微信文件传输助手 / 网盘）。
2. 手机上用文件管理器点开 apk → 提示“禁止安装未知应用”时，点**设置 → 允许来自此来源**，返回继续安装。
3. 装完桌面出现 **GyroSteer 陀螺转向**。
4. 小米/红米、OPPO、vivo 等还可能要求关闭“纯净模式/安全检测”，按提示允许即可。

**方法 2：adb（电脑连着手机）**

```powershell
# 手机开发者选项里打开 USB 调试
adb devices                 # 能看到设备序列号
adb install -r app-debug.apk
# 覆盖安装时加 -r；卸载旧版：adb uninstall com.gyrosteer
```

---

## 装好后必做的四步

1. 打开 GyroSteer → **1. 开启无障碍服务** → 系统设置里找到「GyroSteer 转向注入」并打开（各家 ROM 路径：设置 → 无障碍/辅助功能 → 已下载的服务）。
2. **2. 授予悬浮窗权限** → 允许。
3. 打开 AetherSX2 / NetherSX2，进 GT4 赛道，画面里能看到屏幕左摇杆。
4. 切回 GyroSteer → **3. 校准摇杆位置** → 拖准星到左摇杆中心、调半径 → 保存 → 打开「陀螺转向总开关」→ 点「以当前姿态作为正前方」。

之后跑赛道用体感，进菜单点屏幕上的「陀螺 ON」气泡变 OFF 即可。

---

## 常见问题

| 现象 | 原因 / 处理 |
|---|---|
| 手机提示“无法解析软件包” | 文件没传完整，或下载的是 zip。确认文件名是 `app-debug.apk` |
| 电脑上 `gradlew.bat` 报 “JAVA_HOME not set” | 没装 JDK 17，或装完没重开终端 |
| Gradle Sync 卡在下载 | 国内网络问题，可给 Gradle 配镜像，或直接走路线 C |
| 编译报 `Unresolved reference: R` | 资源目录没被识别，`Build → Clean Project` 后 Rebuild |
| 装好但气泡不出现 | 悬浮窗权限没给，或无障碍服务被系统杀了，重开一次 |
| 转向不生效但状态里 X 在动 | 校准没对准左摇杆，或 AetherSX2 里隐藏了屏幕控件 |

---

## 我这边为什么不能直接给你 apk

生成 apk 需要 Android SDK + JDK + 首次联网拉依赖（构建工具、平台包），当前工作环境三者都没有（无 JDK、无 SDK、命令行无外网）。工程本身已经写成“零第三方依赖”，只要上面任意一条路线跑通就能出包。

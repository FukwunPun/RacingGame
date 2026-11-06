# GyroSteer —— 给 AetherSX2 / NetherSX2 用的“重力感应转向 + 一键切换”外挂方案

> 目标场景：安卓手机 + AetherSX2 / NetherSX2 运行《Gran Turismo 4》。
> 跑赛道时用手机转动（重力感应）控制转向；进菜单/买配件时按一下悬浮小按钮，立刻关掉感应、回到按键操作。

---

## 一、先给结论

**“插件”这条路走不通，但功能本身可以做出来——只能做成“外部辅助应用”，做不成模拟器插件。**

原因（都可查证）：

| 事实 | 后果 |
|---|---|
| AetherSX2 的 UI/输入层是闭源的，作者只按 LGPL 要求放出过 libemucore 等部分源码，没有开放插件 API、没有 SDK、没有脚本扩展点 | 第三方无法向其中挂载任何代码 |
| NetherSX2 不是 AetherSX2 的重写版，而是对官方 APK 做**二进制补丁**（去广告 / 去联网 / 换图标等）得到的同一套可执行文件，官方仓库里只有 xdelta 补丁和打包脚本（HTML 工程） | 它继承 AetherSX2 的全部约束：同样没有插件接口 |
| AetherSX2 的输入子系统只认「物理手柄的按键/摇杆」和「屏幕触摸控件」，没有陀螺仪/加速度计输入源 | 光靠模拟器内置设置，调不出体感转向 |

所以：**不存在能“装进”AetherSX2/NetherSX2 的插件**。任何声称是“AetherSX2 陀螺仪插件”的东西，本质都是本仓库这种外部辅助应用（或者是对 APK 的二次破解，风险高、随版本失效）。

**能实现功能的三条现实路线**（详见第六节对比）：

1. 外部辅助 App + 无障碍手势注入（**本仓库实现**，免 root，今日可用）；
2. 外部辅助 App + root（uinput 虚拟手柄），延迟最低、手感最像真手柄，但要 root + NDK 编译；
3. 等模拟器作者/分支加入原生体感（社区呼吁多年，至今没有）。

---

## 二、本方案怎么工作

```
手机姿态 ──加速度计/陀螺仪──► SteeringEngine ──归一化转向量(-1..1)──► StickInjector
                                                                      │
                                       无障碍 dispatchGesture 持续拖动 ─┘
                                                                      ▼
                                    AetherSX2 的屏幕左摇杆控件（X 轴）
                                                                      ▼
                                                       GT4：左摇杆 X = 转向
```

- **倾斜模式（默认，推荐 GT4）**：取重力在「屏幕平面」内的分量，算出手机像方向盘一样被转了多少度。手机回正 → 方向自动回正，不用手动归中，也几乎不漂移。
- **方向盘模式（可选）**：对陀螺仪绕屏幕法线（设备 Z 轴）的角速度积分，得到“虚拟方向盘角度”，松手后按“回中”强度缓慢回中。手感更像真轮，但会漂移、需要偶尔归中。
- **悬浮开关气泡**：游戏内单击 = 开/关陀螺转向（关掉后完全不注入，你手里的实体手柄/屏幕按键照常工作）；长按 = 回设置界面；拖动 = 挪位置。这就是需求里说的“一个虚拟功能键快速关闭重力感应”。
- **校准**：把十字准星拖到 AetherSX2 屏幕左摇杆的中心、调好半径，存成屏幕比例，换分辨率不用重来。

> 为什么注入“触摸”而不是“手柄事件”？非 root 应用没有 `INJECT_EVENTS` 权限，唯一被系统允许的注入通道就是无障碍手势（`canPerformGestures`）。root 方案可以做得更好，见第六节。

---

## 三、工程结构

```
gyro_steer_app/
├─ settings.gradle.kts / build.gradle.kts / gradle.properties
└─ app/
   ├─ build.gradle.kts
   └─ src/main/
      ├─ AndroidManifest.xml
      ├─ java/com/gyrosteer/
      │  ├─ AppPrefs.kt                 参数持久化
      │  ├─ SteeringEngine.kt           转向算法（倾斜 / 方向盘两种模式、死区、平滑、回中）
      │  ├─ StickInjector.kt            无障碍手势注入（连续手势 / 兼容模式，自动重连）
      │  ├─ SteerAccessibilityService.kt 传感器 → 算法 → 注入，悬浮开关与校准的总控
      │  ├─ OverlayManager.kt           悬浮气泡（开关键 + 长按回设置 + 拖动）
      │  ├─ CalibrationOverlay.kt       摇杆位置/半径校准浮层
      │  └─ MainActivity.kt             设置界面 + 实时状态（当前 X、姿态是否可用）
      └─ res/  layout / values / drawable / xml
```

依赖：**零第三方依赖**（不用 AndroidX / Compose），只用系统 API，`minSdk 24`、`compileSdk 34`。

---

## 四、怎么编译成 APK

**⚠️ zip 是源码，不是安装包，安卓无法直接安装。** 必须先编译出 `app-debug.apk`。

三种编译方式（Android Studio / 命令行 / GitHub Actions 免环境）和安装步骤、常见报错，全部写在 **[INSTALL.md](INSTALL.md)** 里。下面只放命令行版的速查：


```powershell
# 1) JDK 17（Android Gradle Plugin 8.x 要求）
winget install --id EclipseAdoptium.Temurin.17.JDK

# 2) Android 命令行工具（含 sdkmanager）
#    下载 commandlinetools-win-*.zip 解压到 %LOCALAPPDATA%\Android\Sdk\cmdline-tools\latest
$env:ANDROID_HOME = "$env:LOCALAPPDATA\Android\Sdk"
& "$env:ANDROID_HOME\cmdline-tools\latest\bin\sdkmanager.bat" `
    "platform-tools" "platforms;android-34" "build-tools;34.0.0"

# 3) Gradle 8.7（或直接用 Android Studio 打开本目录，让它下载 Gradle）
winget install --id Gradle.Gradle

# 4) 生成 wrapper 并打包（在 gyro_steer_app 目录下）
gradle wrapper --gradle-version 8.7
.\gradlew.bat assembleDebug
# 产物：app\build\outputs\apk\debug\app-debug.apk（用 debug 签名，可直接安装）
```

Android Studio 路线：`File → Open` 选 `gyro_steer_app`，等 Gradle Sync 完，`Build → Build APK(s)`。

正式分发需要自签：`keytool -genkeypair ...` 生成 keystore，再用 `apksigner sign --ks ...` 签 `assembleRelease` 的产物。

---

## 五、手机端使用步骤

**准备（只需一次）**

1. 安装 APK，打开 GyroSteer。
2. 点「1. 开启无障碍服务」→ 系统设置里找到 **GyroSteer 转向注入** → 打开。（系统会提示“可查看/控制屏幕”，本应用只用手势注入能力，不读取界面内容。）
3. 点「2. 授予悬浮窗权限」→ 允许。
4. 打开 AetherSX2 / NetherSX2，进 GT4 的赛道（能看到屏幕左摇杆为止）。
5. 切回 GyroSteer，点「3. 校准摇杆位置」：会把应用切后台，并在游戏画面上叠一层准星。
   - 拖动十字到 **屏幕左摇杆圆心的正中**；
   - 「半径 +/−」让青色圆圈大致等于摇杆可推到的最大半径；
   - 「保存」。
6. 回到 GyroSteer，打开「陀螺转向总开关」。此时屏幕上出现「陀螺 ON」小气泡。
7. 把手机横过来，摆成你打游戏时的姿势，点「以当前姿态作为正前方（归中）」。

**日常使用**

- 跑赛道：转动手机 = 打方向；屏幕左摇杆会被自动“推”到对应位置。
- 进菜单：点一下悬浮气泡 → 变「陀螺 OFF」，注入停止，方向立刻回中，之后完全用按键/实体手柄操作（GT4 菜单用十字键或左摇杆都行）。
- 出菜单：再点一下气泡 → 回到体感转向。

**AetherSX2 / NetherSX2 里建议的设置**

- 控制器设置里保留屏幕控件（不要设成“隐藏触摸控制”），左摇杆放在左下角，透明度调低但别设为 0。
- Pad 1 用实体手柄时，屏幕触摸一般仍然有效；若你的版本里接上手柄后触摸被忽略，请临时断开手柄或用第六节的 root 方案。
- GT4 游戏内设置：控制器选 **A（默认）**，转向就是左摇杆 X 轴；不要选成 900° 方向盘模式（那是给带角度的手柄用的）。

**调参建议（GT4 起手）**

| 参数 | 起手值 | 说明 |
|---|---|---|
| 灵敏度 gain | 1.0 | 转向太灵就降到 0.7–0.8；高速发飘可再降 |
| 满舵倾角 | 25° | 手机要转 25° 才打满；想更省力改成 15–20°，想更细腻改成 30–35° |
| 死区 | 0.06 | 直道上方向轻微画龙时加大到 0.10–0.12 |
| 平滑 | 0.35 | 抖动/抽筋感加大到 0.5；嫌反应慢降到 0.2 |
| 方向盘回中 | 1.0 | 只对方向盘模式有效，越大回中越快 |
| 注入帧间隔 | 32 ms | 卡顿或方向跳变时先试 40–48 ms；仍不行就打开「注入兼容模式」 |

---

## 六、和其他路线对比

| 方案 | 免 root | 延迟 | 手感 | 风险 | 现状 |
|---|---|---|---|---|---|
| **本仓库：无障碍手势注入** | ✅ | 中（每帧一次手势 IPC，约 30–60 ms 端到端） | 够用，转向是模拟量 | 个别 ROM 会取消长手势，需开“兼容模式”；AetherSX2 隐藏触摸时无效 | 可用 |
| root + `uinput` 虚拟手柄 | ❌ 需 root | 低（内核输入设备，几 ms） | 最好，模拟器当成真手柄 | 需要 NDK 编译一个 native 注入器；root 环境风险自担 | 可做（未实现） |
| 改 APK / 逆向 AetherSX2 输入层 | ❌ | 低 | 最好 | 闭源混淆、每个版本重做、法律与安全风险 | 不推荐 |
| 等官方加体感 | ✅ | — | — | 无 | 多年无进展，不要等 |

如果日后要做 root 版：核心是写一个 native 小工具，`open("/dev/uinput")` → `UI_SET_EVBIT`/`UI_SET_ABSBIT` → 创建 `ABS_X` 的虚拟手柄，算法层（`SteeringEngine.kt`）可以原样复用，把 `StickInjector` 换成 uinput 写入即可。

---

## 七、已知限制与风险（务必先看）

1. **本仓库的 APK 尚未经过真机编译与测试**：编写环境里没有 JDK / Android SDK，也没有真机。代码按 Android 官方 API 编写（`dispatchGesture` / `StrokeDescription.continueStroke` 均为 API 24+），但首次 `assembleDebug` 仍可能需要修一两个编译错误，注入手感也需要你在真机上按第五节的参数微调。
2. **手势注入是“模拟手指”**，不是内核级输入：端到端延迟比实体手柄高，快速反打方向时可能感觉“慢半拍”。这也是建议 root 版的原因。
3. **AetherSX2 必须保留屏幕摇杆控件**。若你把屏幕控件隐藏，注入没有目标，转向无效。
4. **手机接近平放时重力平面分量不足**，倾斜模式会保持上一帧方向（界面上“姿态可用”会显示“否”）。打游戏时手机基本是竖着拿的，正常不会触发。
5. 无障碍服务被系统或省电策略杀掉后需要重新开启；悬浮气泡消失即代表服务已断。
6. 本应用不采集、不上传任何数据；所有参数只存在本机 SharedPreferences。

---

## 八、后续可以做的

- root/uinput 版本（延迟降到个位数毫秒，模拟器识别为原生手柄）；
- 转向曲线（线性/指数）与速度相关的动态灵敏度；
- 按游戏自动切换配置（GT4 / 其他赛车）；
- 蓝牙手柄的“体感叠加”（手柄摇杆 + 体感微调，做精细修正）。

---

## 九、参考来源

- NetherSX2 官方补丁工程（对 AetherSX2 APK 打二进制补丁，非源码分支）：<https://github.com/Trixarian/NetherSX2-patch>
- NetherSX2-classic（基于 AetherSX2 3668 的续作，同为补丁/打包工程）：<https://github.com/Trixarian/NetherSX2-classic>
- AetherSX2 的 LGPL 源码发布（只有合规要求的部分，UI/输入层未开放）：<https://github.com/Trixarian/AetherSX2>
- NetherSX2 手柄/触摸设置说明（输入源只有物理手柄与屏幕控件）：<https://sx2nether.com/blog/nethersx2-controller-setup-guide/>

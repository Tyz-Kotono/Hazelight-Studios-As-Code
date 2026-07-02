# Core / MiniGames

> 职责：玩法 QTE 小游戏与手柄震动系统——三个近乎同构的 QTE（连打 / 转摇杆 / 抖摇杆）+ 一套四通道力反馈（ForceFeedback）。共 4 子模块、20 个文件。

## 模块定位

本功能域服务双人（Mio / Zoe）协作玩法的“即时输入挑战”与“触觉反馈”：

- **三件套 QTE**（ButtonMash / StickSpin / StickWiggle，各 5 文件）共享同一套 5 文件架构与网络模型。
- **ForceFeedback**（5 文件，含 Debug）为独立的力反馈基础设施，可被任意系统（含 QTE Widget 之外）消费。

## QTE 三件套：共有的 5 文件模式

三个子模块文件名只换前缀，结构与职责完全对应：

| 角色 | 基类 | 职责 | ButtonMash | StickSpin | StickWiggle |
|------|------|------|-----------|-----------|-------------|
| **Capability** | `UHazePlayerCapability`（NetworkMode = **Crumb**） | 驱动端：读输入、算进度、写同步、阻塞他玩法、显隐 Widget | `ButtonMashCapability` | `StickSpinCapability` | `StickWiggleCapability` |
| **Component** | `UActorComponent`（**控制端权威**，`HasControl()` 守卫） | 数据注册表：以 `FInstigator` 为键管理 Active 列表与 State，Start/Stop/Snap/Get | `ButtonMashComponent` | `StickSpinComponent` | `StickWiggleComponent` |
| **Widget** | `UHazeUserWidget` | 表现：进度环按 Mio/Zoe 着色（`MioProgressTexture`/`ZoeProgressTexture`） | `ButtonMashWidget` | `StickSpinWidget` | `StickWiggleWidget` |
| **Statics** | 全局 / mixin 函数 | `BlueprintCallable` 入口：`Start* / Stop* / Get* / Snap*`，内部 `GetOrCreate` 组件转发 | `ButtonMashStatics` | `StickSpinStatics` | `StickWiggleStatics` |
| **Data** | 委托/枚举/Settings/State | 配置与状态数据 + 无障碍 CVar（`Haze.Remove*_Mio/Zoe`） | `ButtonMashData` | `StickSpinData` | `StickWiggleData` |

### 共有的运行模型
1. 外部调 `Start*`（Statics, BlueprintCallable）→ Component（仅 `HasControl()`）入列一条 Active 记录（带 `FInstigator`）。
2. Capability 的 `ShouldActivate` 检测到 Active 列表非空 → 激活，添加 Widget、按 Settings 阻塞 `GameplayAction`/`MovementInput`（保留 `n"UsableWhileButtonMashing"`）、可选取消提示。
3. `TickActive`：**控制端**读输入算进度并写 `UHazeCrumbSynced*Component`；**远端**从同步组件读回，供 Widget 与查询使用。
4. 完成/取消 → `OnDeactivated` 触发 `OnCompleted`/`OnCanceled` 委托，清状态并停止。

### 三者的差异（核心算法）
- **ButtonMash**：测连打**速率**（按时间窗内按键间隔），按难度（Easy…ActuallyImpossible）映射目标速率，进度条充满即完成；支持**双人连打**（`StartDoubleButtonMash`，双方都达标且由 world host 裁决，用 `UNetworkLockComponent`）。
- **StickSpin**：用 `Math::Atan2` 算左摇杆相邻帧**角度差**累计为 `SpinPosition`，方向（顺/逆时针）+ 速度桶平滑出 `SpinVelocity`。
- **StickWiggle**：测水平输入**左右反转**次数（`MoveRight` 越过 `HorizontalWiggleThreshold`），累计 `WiggledAlpha`，支持弹簧/线性两种进度模式，停手后衰减。

### 共有的无障碍（CVar）
每个 QTE 都有 `Haze.Remove*_Mio` / `_Zoe` CVar：值 1 = 转为按住/方向保持（简化），值 2 = 自动完成；StickWiggle/Spin 的 Widget 据此切 `bIsSimplified`。ButtonMash 另有 `PlayerInputDevToggles::ButtonMash::AutoButtonMash`。

## 子模块导航

| 子模块 | 文件数 | 一句话职责 |
|--------|-------|-----------|
| [ButtonMash](ButtonMash/README.md) | 5 | 连打 QTE，测速率/难度，支持双人连打 |
| [StickSpin](StickSpin/README.md) | 5 | 转摇杆 QTE，Atan2 角度差累计旋转 |
| [StickWiggle](StickWiggle/README.md) | 5 | 抖摇杆 QTE，测水平反转次数 |
| [ForceFeedback](ForceFeedback/README.md) | 5 | 四通道力反馈：管理组件 + 世界/方向静态入口 + 预设 + Debug |

## 跨域协同（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
|------|------|------|------|
| 入口 | `Start*` / `StartDoubleButtonMash`（各 Statics） | 直接调用（BlueprintCallable/mixin） | 玩法蓝图与脚本启动 QTE（grep 命中仅各自 Component/Statics 内部，外部消费者多在蓝图层） |
| 同步 | `UHazeCrumbSynced{Vector,Float}Component` | Crumb | 各 Capability 用 PlayerSynced crumb 同步进度/状态 |
| 委托 | `OnCompleted` / `OnCanceled` / `OnStopped` | 委托 | QTE 结果回调外部玩法 |
| 被调用 | `CameraShakeForceFeedback/CameraShakeForceFeedbackComponent.as` | 直接调用 `ForceFeedback::` | 相机震动经 `ForceFeedback::PlayWorldForceFeedback` 触发手柄震动（grep 实证） |
| 关联 | `SequencerAndTime/TimeDilation` | 参数解耦 | ForceFeedback `bIgnoreTimeDilation` 决定震动是否随时间缩放 |

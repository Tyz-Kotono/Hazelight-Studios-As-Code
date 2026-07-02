# Core / MiniGames / StickWiggle

> 职责：抖摇杆 QTE——测量左摇杆水平方向的左右反转次数，累计进度直至完成。共 5 文件。
>
> 本子模块遵循 QTE 三件套的共有 5 文件模式（Capability + Component + Widget + Statics + Data），共有点详见 [MiniGames 总览](../README.md)；本页只列 StickWiggle 的特有逻辑。

## 内部架构（关键类/数据流）

- **`UStickWiggleComponent`**（`UActorComponent`，NotBlueprintable，控制端权威）：以 `FInstigator` 为键管理 `ActiveWiggles` 与 `WiggleState`。提供 `StartStickWiggle` / `StopStickWiggle` / `GetStickWiggleState` / `SnapStickWiggleState`，写操作 `HasControl()` 守卫。
- **`UStickWiggleCapability`**（`UHazePlayerCapability`，NetworkMode = Crumb）：核心算法在 `TickActive`。
  - 用 `UHazeCrumbSyncedFloatComponent`（PlayerSynced）同步单值 `WiggledAlpha`。
  - 控制端读 `AttributeNames::MoveRight`，当水平输入越过 `HorizontalWiggleThreshold` 且**方向相对上帧发生反转**时记一次有效抖动，刷新 `LastWiggleTime`。
  - 进度推进两种模式：`bChunkProgress`（用 `FHazeAcceleratedFloat` 弹簧 `SpringTo`，每次反转加 `1/WigglesRequired`）或平滑（`FInterpConstantTo` 按 `WiggleIntensityIncreaseTime`）。停手超过 `WiggleStartDecreasingDelay` 后按 `WiggleIntensityDecreaseTime` 衰减。
  - 完成判定：`FStickWiggleState::IsFinished()`（`WiggledAlpha` ≈ 1）。
  - `OnLogActive` 输出每个活动 Wiggle 的状态到时序日志。
- **`FStickWiggleSettings` / `FStickWiggleState`**（Data）：Settings 含阈值、chunk/平滑开关、`WigglesRequired`、增减时间、Widget 选项；State 含 `WiggleInput`(-1/0/1) 与 `WiggledAlpha`。
- **`UStickWiggleWidget`**（Abstract）：进度环按 Mio/Zoe 着色（`MioProgressTexture`/`ZoeProgressTexture`），`bShowProgressBar` 控制显隐，`bIsSimplified` 切换时 `BP_UpdateSettings()`。

### 特有点
- 唯一以**方向反转计数**为核心的 QTE，且**有明确完成态**（`bWasCompleted`），与 ButtonMash 类似走 `OnCompleted`/`OnCanceled` 双委托。
- 弹簧式 chunk 进度（`SpringStiffness=1500`, `SpringDamping=0.3`）让分段进度有回弹手感。

## 协同关系（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
|------|------|------|------|
| 入口 | `StartStickWiggle`（mixin）/ `GetStickWiggleState` / `SnapStickWiggleState` / `StopStickWiggle` | 直接调用（mixin/BlueprintCallable） | 内部 `UStickWiggleComponent::GetOrCreate` 转发（grep: StickWiggleStatics.as），Start 时 devEnsure IncreaseTime>0 |
| 同步 | `UHazeCrumbSyncedFloatComponent` | Crumb（PlayerSynced） | 同步单值 WiggledAlpha |
| 输入 | `AttributeNames::MoveRight` | 直接读属性 | 水平输入做反转检测 |
| 标签阻塞 | `n"GameplayAction"` / `n"MovementInput"`（排除 `n"UsableWhileButtonMashing"`） | 标签阻塞 | `bBlockOtherGameplay` 时阻塞其它玩法 |
| 委托 | `FOnStickWiggleCompleted` / `FOnStickWiggleCanceled` | 委托 | 完成/取消回调外部 |
| CVar | `Haze.RemoveStickWiggle_Mio/Zoe` | 可组合设置（无障碍） | 值1=方向保持简化，值2=自动 |

## 关键文件

| 文件 | 一句话 |
|------|--------|
| `StickWiggleCapability.as` | 驱动端：水平反转计数 + 弹簧/平滑进度 + 衰减 + 完成态（Crumb） |
| `StickWiggleComponent.as` | 控制端权威数据注册表（ActiveWiggles + WiggleState，按 Instigator 键） |
| `StickWiggleData.as` | Settings/State（含 IsFinished）+ 无障碍 CVar |
| `StickWiggleStatics.as` | `StartStickWiggle` 等 mixin/BlueprintCallable 入口 |
| `StickWiggleWidget.as` | 按 Mio/Zoe 着色的进度环 Widget，支持简化切换 |

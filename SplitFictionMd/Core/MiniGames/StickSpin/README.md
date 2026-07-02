# Core / MiniGames / StickSpin

> 职责：转摇杆 QTE——用 Atan2 计算左摇杆相邻帧角度差，累计为旋转位置与速度。共 5 文件。
>
> 本子模块遵循 QTE 三件套的共有 5 文件模式（Capability + Component + Widget + Statics + Data），共有点详见 [MiniGames 总览](../README.md)；本页只列 StickSpin 的特有逻辑。

## 内部架构（关键类/数据流）

- **`UStickSpinComponent`**（`UActorComponent`，控制端权威）：以 `FInstigator` 为键管理 `ActiveSpins` 与 `SpinState`。提供 `StartStickSpin` / `StopStickSpin` / `GetStickSpinState` / `SnapStickSpinState`，写操作 `HasControl()` 守卫。
- **`UStickSpinCapability`**（`UHazePlayerCapability`，NetworkMode = Crumb）：核心算法在 `TickActive`。
  - 用 `UHazeCrumbSyncedVectorComponent`（PlayerSynced）同步 `(SpinPosition, SpinVelocity)`。
  - 控制端读 `AttributeVectorNames::LeftStickRaw`，当前后两帧摇杆量级都 ≥0.7 时，用 `Math::Atan2` 求两帧角度，`Math::FindDeltaAngleRadians` 得角度差（绝对值 <½π 才算有效），按方向累加 `SpinPosition += AngleDifference / TWO_PI`，受 `bAllowSpinClockwise/CounterClockwise` 与可选 min/max 限制。
  - 速度用 0.2s 时间桶（当前桶 + 上一桶）平滑算出 `SpinVelocity`。
  - 简化模式（CVar）：键鼠或被禁用时，按水平方向以 `RemovedSpinSpeed` 恒速旋转。
  - `OnLogActive` 输出 SpinPosition/Velocity 到时序日志。
- **`FStickSpinSettings` / `FStickSpinState` / `EStickSpinDirection`**（Data）：方向枚举（NotSpinning/Clockwise/CounterClockwise）；Settings 控制允许方向、min/max 位置、取消、阻塞、Widget；`FStickSpinState` 含 Direction/SpinPosition/SpinVelocity（带 `opEquals`）。
- **`UStickSpinWidget`**：显示自旋状态，`bIsSimplified` 切换时调 `BP_UpdateSettings()`；附着支持 HUD `Tutorial` 槽或世界位置。

### 特有点
- 唯一以**角度（弧度）几何**为核心的 QTE：进度是“相对起点旋转的圈数”，可正可负，可设上下限。
- 无“完成”概念（Statics 只有 `OnStopped` 委托）——通常由外部读 `GetStickSpinState` 自行判定，故只在取消/外部停止时反激活。

## 协同关系（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
|------|------|------|------|
| 入口 | `StartStickSpin`（mixin）/ `GetStickSpinState` / `SnapStickSpinState` / `StopStickSpin` | 直接调用（mixin/BlueprintCallable） | 内部 `UStickSpinComponent::GetOrCreate` 转发（grep: StickSpinStatics.as） |
| 同步 | `UHazeCrumbSyncedVectorComponent` | Crumb（PlayerSynced） | 同步旋转位置/速度 |
| 输入 | `AttributeVectorNames::LeftStickRaw` | 直接读属性 | 原始左摇杆输入做 Atan2 |
| 标签阻塞 | `n"GameplayAction"` / `n"MovementInput"`（排除 `n"UsableWhileButtonMashing"`） | 标签阻塞 | `bBlockOtherGameplay` 时阻塞其它玩法 |
| 委托 | `FOnStickSpinStopped`（OnStopped） | 委托 | 停止回调外部 |
| CVar | `Haze.RemoveStickSpin_Mio/Zoe` | 可组合设置（无障碍） | 转为方向恒速旋转（简化） |

## 关键文件

| 文件 | 一句话 |
|------|--------|
| `StickSpinCapability.as` | 驱动端：Atan2 角度差累计 + 时间桶测速 + 简化模式（Crumb） |
| `StickSpinComponent.as` | 控制端权威数据注册表（ActiveSpins + SpinState，按 Instigator 键） |
| `StickSpinData.as` | Settings/State/方向枚举 + 无障碍 CVar 与 RemovedSpinSpeed |
| `StickSpinStatics.as` | `StartStickSpin` 等 mixin/BlueprintCallable 入口 |
| `StickSpinWidget.as` | 显示自旋状态、支持简化切换的 Widget |

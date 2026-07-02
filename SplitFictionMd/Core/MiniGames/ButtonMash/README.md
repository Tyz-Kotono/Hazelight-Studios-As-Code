# Core / MiniGames / ButtonMash

> 职责：连打 QTE——测量玩家连打速率，按难度判定进度，支持单人/双人连打。共 5 文件。
>
> 本子模块遵循 QTE 三件套的共有 5 文件模式（Capability + Component + Widget + Statics + Data），共有点详见 [MiniGames 总览](../README.md)；本页只列 ButtonMash 的特有逻辑。

## 内部架构（关键类/数据流）

- **`UButtonMashComponent`**（`UActorComponent`，控制端权威）：以 `FInstigator` 为键管理 `ActiveMashes` 与 `MashState`。提供 `StartButtonMash` / `StopButtonMash` / `GetButtonMashProgress` / `GetButtonMashCurrentRate` / `SnapButtonMashProgress` / `SetButtonMashGainMultiplier` / `SetAllowButtonMashCompletion`。`OnVisualPulse`（event）广播视觉脉冲。所有写操作 `HasControl()` 守卫。`ApplyRemoveButtonMashesSetting` 据 CVar 把连打转为按住。
- **`UButtonMashCapability`**（`UHazePlayerCapability`，NetworkMode = Crumb）：核心算法在 `TickActive`。
  - 用 `UHazeCrumbSyncedVectorComponent`（PlayerSynced）同步 `(MashRate, Progress, TotalPresses)`。
  - 控制端按难度经 `GetConfigForButtonMashDifficulty` 取 `MinRate`/`TargetRate`，用 impulse + decay 模型推进进度；记录 `ButtonPressRealTimes` 算近 1 秒平均按键间隔得 `MashRate`。
  - 远端按同步的剩余按键数模拟 `Widget.Pulse()`。
  - **双人连打**：`IsDoubleMash()` 时由 `Network::HasWorldControl()` 端检查双方进度均达标才置完成；`OnDeactivated` 同时停止另一玩家的连打。
- **`FButtonMashSettings` / `FButtonMashState` / 枚举**（Data）：
  - `EButtonMashMode`（Mash/Hold）、`EButtonMashProgressionMode`（MashToProgress / StartFullDecayDown / MashToProceedOnly / MashRateOnly / …）、`EButtonMashDifficulty`（Easy…ActuallyImpossible）。
  - Settings 含时长、按钮 Action、是否阻塞他玩法、Widget 附着等；`IsButtonHold/IsAutomatic/ShouldShowProgress` 据 CVar 与模式裁决。
- **`UButtonMashWidget`**：进度环用 `MioProgressTexture`/`ZoeProgressTexture` 按玩家着色，`Pulse()` 播脉冲动画，按是否面键调整缩放，按 hold/mash 切换指示器。

### 特有点
- 唯一支持**双人协作连打**的 QTE：`ButtonMash::StartDoubleButtonMash`（Statics）同时给 Mio/Zoe 起一条同 Instigator 的连打，host 裁决联合完成；双人连打禁用取消（网络安全）。
- 完成判定 `DoesProgressCountAsCompleted`：随 `ProgressionMode` 不同为 `>=1.0` 或 `<=0.0`。

## 协同关系（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
|------|------|------|------|
| 入口 | `StartButtonMash`（mixin）/ `ButtonMash::StartDoubleButtonMash` | 直接调用（BlueprintCallable） | 外部玩法启动，内部 `UButtonMashComponent::GetOrCreate` 转发（grep: ButtonMashStatics.as） |
| 同步 | `UHazeCrumbSyncedVectorComponent` | Crumb（PlayerSynced） | 同步速率/进度/按键数 |
| 标签阻塞 | `n"GameplayAction"` / `n"MovementInput"`（排除 `n"UsableWhileButtonMashing"`） | 标签阻塞 | `bBlockOtherGameplay` 时阻塞其它玩法 |
| 委托 | `FOnButtonMashCompleted`（OnCompleted/OnCanceled） | 委托 | 完成/取消回调外部 |
| 直接调用 | `Player.OtherPlayer` 的 `UButtonMashComponent` | 直接调用 `Get` | 双人连打停止另一侧 |
| 网络锁 | `UNetworkLockComponent`（DoubleMashLock） | Crumb/锁 | 双人连打完成裁决 |
| CVar | `Haze.RemoveButtonMashes_Mio/Zoe` + `AutoButtonMash` Dev Toggle | 可组合设置（无障碍） | 连打→按住/自动 |

## 关键文件

| 文件 | 一句话 |
|------|--------|
| `ButtonMashCapability.as` | 驱动端：impulse/decay 进度模型 + 速率测量 + 双人裁决（Crumb） |
| `ButtonMashComponent.as` | 控制端权威数据注册表（Active 列表 + State，按 Instigator 键） |
| `ButtonMashData.as` | Settings/State/枚举 + 难度配置表 + 无障碍 CVar |
| `ButtonMashStatics.as` | `StartButtonMash` mixin + `StartDoubleButtonMash` 等 BlueprintCallable 入口 |
| `ButtonMashWidget.as` | 按 Mio/Zoe 着色的进度环 Widget，含脉冲与 hold/mash 指示 |

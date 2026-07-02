# Core / MiniGames / ForceFeedback

> 职责：手柄力反馈系统——按时长在四通道（左右马达 + 左右扳机）上 lerp 震动，提供世界定位/方向性触发入口与一组预设曲线。共 5 文件（含 Debug 2 文件）。

## 内部架构（关键类/数据流）

- **`UPlayerForceFeedbackEffectsManagerComponent`**（`UActorComponent`）：每玩家一个，tick 期间驱动定时效果。
  - 维护 `TArray<FPlayerForceFeedbackDuration>`，每条记 Instigator / Duration / RemainingTime / 起止四通道值（`FHazeFrameForceFeedback`：LeftMotor/RightMotor/LeftTrigger/RightTrigger）。
  - `AddDuration`（恒定）/ `AddBlendedDuration`（起止插值）入列并启用 Tick。
  - `Tick`：按 `Alpha = 1 - RemainingTime/Duration` 对四通道 `Math::Lerp`，调 `Player.SetFrameForceFeedback`；到期移除，列表空则停 Tick。
- **`ForceFeedbackStatics.as`**：两类入口。
  - `ForceFeedback::` 命名空间（世界级，BlueprintCallable）：`PlayWorldForceFeedback`、`PlayWorldForceFeedbackForFrame`、`PlayDirectionalWorldForceFeedbackForFrame`、`StopWorldForceFeedback`，按 `Epicenter` + 内/外半径 + 衰减指数对 `Game::Players` 计算每人强度（`GetWorldForceFeedbackIntensityForPlayer`），可选 `AffectedPlayers`（Mio/Zoe/Both）与 `bIgnoreTimeDilation`。`ConvertWorldDirectionToForceFeedback` 把世界方向按玩家视角右向量投到左右马达。
  - 玩家级 mixin（BlueprintCallable）：`PlayForceFeedbackDuration` / `PlayForceFeedbackBlendedDuration` / `PlayForceFeedbackWorldDirection` / `PlayForceFeedbackBlendedWorldDirection`，内部 `UPlayerForceFeedbackEffectsManagerComponent::GetOrCreate` 转发。
- **`DefaultForceFeedbacks.as`**：`ForceFeedback::` 命名空间下定义一组 `UForceFeedbackEffect` 预设资产（Light / Very_Light / Light_Tap / Medium / Medium_Short / Heavy / Heavy_Short），各绑定 `Curves::` 子命名空间里的 `UCurveFloat` 衰减曲线（文件含 ASCII 曲线示意）。
- **Debug/**：
  - `ForceFeedbackDevMenu.as`（`UHazeDevMenuEntryimmediateWidget`）：调试面板，显示当前玩家 FF 启用状态与已注册 FF 资产列表。
  - `ForceFeedbackDevToggleCapability.as`（`UHazePlayerCapability`）：Dev Toggle 开关，按 `ForceFeedbackDevToggles::DisableFor`/`DisableTriggersFor` 设置 `Haze.EnableForceFeedback`/`EnableTriggerForceFeedback` CVar。

### 数据流
调用方 → Statics（世界级或玩家级）→ 世界级先按距离/衰减算每人强度后下发，玩家级入列管理组件 → 管理组件每帧 lerp 四通道 → `Player.SetFrameForceFeedback`。

## 协同关系（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
|------|------|------|------|
| 被调用 | `CameraShakeForceFeedback/CameraShakeForceFeedbackComponent.as` | 直接调用 `ForceFeedback::` | 相机震动经 `ForceFeedback::PlayWorldForceFeedback(...)` 触发手柄震动（grep 实证，第 66 行） |
| 入口 | `PlayForceFeedbackDuration` 等 mixin | 直接调用 `GetOrCreate` | 玩家级入口转发到管理组件 |
| 调用 | 引擎 `Player.SetFrameForceFeedback` / `PlayForceFeedback` / `SetFrameDirectionalForceFeedback` | 直接调用 | 底层下发震动 |
| 选择 | `EHazeSelectPlayer`（Mio/Zoe/Both）+ `Game::Players` | 可组合设置 | 世界级效果按玩家筛选与距离衰减 |
| 参数解耦 | `bIgnoreTimeDilation` ↔ `SequencerAndTime/TimeDilation` | 参数解耦 | 决定震动强度是否随世界时间缩放 |
| CVar | `Haze.EnableForceFeedback` / `EnableTriggerForceFeedback` | 可组合设置（Dev Toggle） | Debug 能力按 Dev Toggle 开关 |

## 关键文件

| 文件 | 一句话 |
|------|--------|
| `PlayerForceFeedbackEffectsManagerComponent.as` | 每玩家定时四通道 lerp 震动管理组件 |
| `ForceFeedbackStatics.as` | 世界定位/方向性 + 玩家级 mixin 的力反馈入口 |
| `DefaultForceFeedbacks.as` | Light…Heavy 预设 FF 资产 + 衰减曲线 |
| `Debug/ForceFeedbackDevMenu.as` | 调试面板：显示 FF 启用状态与已注册资产 |
| `Debug/ForceFeedbackDevToggleCapability.as` | Dev Toggle 开关，控制 FF/扳机震动 CVar |

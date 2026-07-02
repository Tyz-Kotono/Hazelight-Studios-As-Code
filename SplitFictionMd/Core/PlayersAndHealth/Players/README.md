# Core / PlayersAndHealth / Players

> 职责：定义玩家 Actor 本体 `APlayerCharacter` 及其配色、变体换装、高光、视角模式、聚焦等附属系统与一整套开发调试工具，共 22 文件（含 Debug 子目录 14 文件）。

## 内部架构

### 玩家本体（`PlayerCharacter.as`）
`class APlayerCharacter : AHazePlayerCharacter`，是整个功能域的核心承载者：
- 用 `default` 在类默认值里挂载所有玩家组件：`UHazeInputComponent`、`UHazeCapabilityComponent`、相机相关组件、`UPlayerRespawnComponent`、`UPlayerHealthComponent`、`UPlayerVariantComponent`（实为父类 `UHazeBasePlayerVariantComponent` 槽位）等。
- 通过 `CapabilityComponent.DefaultCapabilities.Add(...)` 注册 PlayerHealth 模块的全部健康能力（`PlayerRespawnCapability`、`PlayerDeathCapability`、`PlayerHealthRegenerationCapability`、`PlayerHealthDisplayCapability`、`PlayerGameOverCapability`、`PlayerRespawnMashCapability`）以及相机/音频/动画/AI 等海量能力。
- `GetFocusLocation()` 重写转发到 `PlayerFocus::GetPlayerFocusLocation`。

### 数据/工具类
- **`PlayerColor.as`**：`namespace PlayerColor` 定义 Mio（品红 `0xee009a`）、Zoe（青柠 `0xcaf431`）、BothPlayers 三色常量；`GetColorForPlayer`、mixin `GetPlayerUIColor`/`GetPlayerDebugColor` 按 `EHazePlayer` 取色。
- **`PlayerVariantComponent.as`** + **`PlayerVariantStatics.as`**：`UPlayerVariantComponent` 管理玩家当前变体（RealWorld/Scifi/Fantasy），按 Mio/Zoe 切换骨骼网格、音效 SoundDef、特效处理器（`UHazeEffectEventHandlerComponent`）与预设；statics 提供 `ApplyPlayerVariantOverride` 等 mixin。
- **`PlayerHighlight.as`**：`APlayerHighlight`（聚光/点光/球体高光 Actor）+ `UPlayerHighlightSettings`（可组合设置）+ `UPlayerHighlightCapability`（按设置生成/淡出高光、设置灯光通道）。
- **`PlayerMovementPerspectiveModeComponent.as`**：`UPlayerMovementPerspectiveModeComponent` 持有 `TInstigated<EPlayerMovementPerspectiveMode>`（ThirdPerson/SideScroller/TopDown/MovingTowardsCamera），供相机行为判断 2D/3D 视角。
- **`PlayerFocusSettings.as`**：`UPlayerFocusTargetSettings`（可组合）+ `namespace PlayerFocus`，决定玩家是否可被聚焦及聚焦点位置；死亡时可改聚焦存活的另一名玩家（消费 `UPlayerHealthComponent`）。

### Debug 子目录（14 文件，多为 `#if TEST/EDITOR` 注册）
以 `UHazeDevInputHandler` 派生的开发输入（传送、反转、锁输入、换手柄、复制相机变换、Kill）、`UPlayerDebugWhoIsWhoCapability`/`UPlayerGlobalTemporalLogCapability` 调试能力、`UDebugViewModeManager` 视图模式、`namespace PlayerInputDevToggles`、`UDebugActorBlockersComponent` 等。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|------|------|------|------|
| 挂载 | PlayerHealth 全部组件/能力 | 默认值组件 + DefaultCapabilities | `PlayerCharacter.as` 挂 `UPlayerHealthComponent`/`UPlayerRespawnComponent` 并注册 6 个健康能力 |
| 调用→ | `PlayerFocus::GetPlayerFocusLocation` | 直接调用 | `PlayerCharacter.as:172` 重写 `GetFocusLocation` |
| 调用→ | `UPlayerHealthComponent::Get` | `Type::Get` | `PlayerFocusSettings.as` 5 处查死亡状态以切换聚焦目标 |
| 调用← | `UPlayerVariantComponent::Get` | `Type::Get` + mixin | `PlayerVariantStatics.as` 暴露给全局换装 |
| 设置 | `UPlayerHighlightSettings`/`UPlayerFocusTargetSettings` | ApplySettings/GetSettings | 继承 `UHazeComposableSettings`，运行时叠加 |
| 配色 | `PlayerColor::Mio/Zoe` | 命名空间常量 | 被 PlayerHealth（重生闪烁色）、Highlight、Debug 等广泛复用 |
| 阻塞 | `n"MovementCameraBehavior"` 等 | 标签阻塞 | `PlayerMovementPerspectiveModeComponent.IsCameraBehaviorEnabled` |

## 关键文件

- `PlayerCharacter.as`：玩家 Actor 本体，挂载所有组件并注册全部默认能力。
- `PlayerColor.as`：Mio/Zoe/双人配色常量与取色 mixin。
- `PlayerVariantComponent.as`：按 Mio/Zoe + 变体切换网格/音效/特效处理器/预设。
- `PlayerVariantStatics.as`：变体覆盖的全局 mixin（Apply/Clear/GetType）。
- `PlayerHighlight.as`：高光 Actor + 可组合设置 + 高光能力。
- `PlayerMovementPerspectiveModeComponent.as`：玩家视角模式状态（含 `EPlayerMovementPerspectiveMode` 枚举）。
- `PlayerFocusSettings.as`：聚焦目标设置与聚焦点/可聚焦判定（死亡时切换）。
- `Debug/`（14 文件）：开发期传送/锁输入/换手柄/反转/Kill 等 DevInput、WhoIsWho 与 TemporalLog 调试能力、视图模式、调试图表与开关。

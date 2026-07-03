# LevelSpecific / Tundra / Shapeshifting — 多形态变形（核心机制）

> 路径：`LevelSpecific/Tundra/Shapeshifting/`（204 文件）
>
> Tundra 的招牌核心：玩家能在多种动物/生灵形态间**变形**。按体型分三档——**小型（Small）**：水獭、仙灵；**玩家本体（Player）**；**大型（Big）**：雪猴、树守。不同形态有各自的移动与交互能力，用于破解对应地形谜题。变形是 River / Crack 两大机制的通用能力底座。

承接 [Tundra 总览](../README.md)。变形是本关最重要机制，River 与 Crack 都建立在它之上。

---

## 职责

实现"玩家 ↔ 形态角色"的切换系统：变形请求、碰撞检测、变身动画/失败反馈、形态间的能力表切换，以及各形态自身的移动/交互能力。

---

## 内部架构

### 变形系统内核（Shapeshifting/ 根目录）

系统核心是**玩家侧组件 + 一组变形能力**，是能力四件套 + 组件容器范式的集中体现：

| 类 | 基类 | 说明 |
|---|---|---|
| `UTundraPlayerShapeshiftingComponent` | `UActorComponent` | 变形核心状态容器：当前/上一形态（`FShapeShiftTriggerData`）、`SmallFormSheet`/`BigFormSheet`（两档形态的能力表）、小/大形态子组件引用、形态阻挡列表（Blockers）、`OnChangeShape` 事件。用 `access:ShapeshiftingSystem` 私有暴露给变形能力 |
| `UTundraPlayerShapeBaseComponent` | `UActorComponent` | 各形态子组件基类：定义 `ShapeType`、`GetShapeActor()`/`GetShapeMesh()`/碰撞尺寸/材质染色接口，由各形态覆写 |
| `UTundraPlayerShapeshiftingCapability` | `UHazePlayerCapability` | 变形入口能力：读 `ForceShapeOverride`（强制）或 `InputShapeRequest`（输入），检查 `ShapeTypeIsBlocked` 后触发变形 |
| `UTundraPlayerShapeshiftingInputCapability` | `UHazePlayerCapability` | 采集玩家变形输入 |
| `UTundraPlayerShapeshiftingMorphCapability` / `...MorphFailCapability` | `UHazePlayerCapability` | 变身过渡动画 / 变身失败反馈 |
| `UTundraPlayerShapeshiftingComponent`（枚举） | — | `ETundraShapeshiftShape { None, Small, Player, Big }`、`ETundraShapeshiftActiveShape { Small, Player, Big }` |

其余内核件：`TundraPlayerShapeshiftingSettings`（设置）、`TundraShapeshiftingStatics`（静态）、`TundraPlayerShapeshiftingDeathResponseCapability`（死亡）、`...IgnoreCollisionContainerComponent`（变形穿模处理）、`...NetworkBlockCapability`（联机阻塞）、`TundraShapeshiftingEffectHandler`（表现）。

标签：`TundraShapeshiftingTags::Shapeshifting` / `ShapeshiftingActivation`，并含 `BlockedWhileIn::LedgeGrab` 等互斥标签。

### 四种形态（体型三档）

每种形态是一个独立 `AHazeCharacter` + 配套 Capability/Component/Settings/EffectHandler，用 `default ShapeType` 声明所属档位：

| 形态 | 角色类（`: AHazeCharacter`） | 档位 `ShapeType` | 特色能力 |
|---|---|---|---|
| **水獭 Otter** | `ATundraPlayerOtterActor` | `Small` | 游泳（`Swimming/`）、声呐冲击（`SonarBlast/`）、水面破出（`BreakWaterSurface`）、水中弹射球（`WaterLaunchSphere`）、瞬变雪猴 |
| **仙灵 Fairy** | `ATundraPlayerFairyActor` | `Small` | 地面冲刺（`GroundDash`）、跳跃、飞跃（`Leap`/`LeapActive`）、样条飞行/栖停（根目录 `FairyMoveSpline`、`Fairy/PerchSpline*`） |
| **雪猴 SnowMonkey** | `ATundraPlayerSnowMonkeyActor` | `Big` | 天花板攀爬（`CeilingMovement`）、可攀藤（`ClimbableVine`）、边缘抓取（`LedgeGrab`）、地面运动、拳击交互（`PunchInteract`）、冰王 Boss 拳（`IceKingBossPunch`） |
| **树守 TreeGuardian** | `ATundraPlayerTreeGuardianActor` | `Big` | 赋生（`LifeGiving/`）、远程交互（`RangedInteractions/`：抓钩 Grapple / 射击 Shoot / 赋生 Life / 按压冰王 HoldDown）、踩扁水獭（`SquashOtter`） |

> 小型两形态（Otter/Fairy）与大型两形态（SnowMonkey/TreeGuardian）分别对应双人各一侧，通过 `SmallFormSheet` / `BigFormSheet` 能力表在变形时整表切换。

### 树守远程交互（TreeGuardian/RangedInteractions/）— 展开

树守是交互最丰富的形态，其远程交互自成子系统：通用瞄准（`InteractionAim`/`AimCamera`/`Aim2D` + `Crosshair`/`Targetable` widget）+ 四类动作——抓钩（`Grapple`：Enter/Kickback）、射击（`Shoot`：投射物 + Spawner + Targetable）、远程赋生（`Life`：`TundraRangedLifeGivingActor`）、按住冰王（`HoldDown`：`TreeGuardianHoldDownIceKingCapability`，用于 Boss）。赋生对象实现 `UTundraGroundedLifeReceivingTargetableComponent` / `UTundraLifeReceivingComponent`。

### 触发与场景（ShapeshiftingTriggers/ 与 ShapeshiftingInteractions/）

- `ShapeshiftingTriggers/`：`ATundraShapeshiftingTriggerBox`（`APlayerTrigger`，强制变形区）、`ChangeSettingTriggerBox`（切设置）、`CameraVolume`（`AHazeCameraVolume`）、`RespawnPoint`（`ARespawnPoint`）、`TeleportTrigger/`。
- `ShapeshiftingInteractions/`：`UTundraShapeshiftingInteractionComponent`（`UInteractionComponent`）、`OneShotInteraction`（`UOneShotInteractionComponent`）——把关卡交互点绑定到特定形态。
- 其它：`BreakingIce/`（破冰）、`IceWaterVolume/`（冰水体积）、`Cutscene/`（`UTundraCutsceneShapeshiftComponent` 过场变形）、`WindCurrentSpline`（风流）、`PoleClimb`（爬杆，多个 `TundraPlayerShapeShiftingPoleClimb*`）。

### 数据流（一次变形）

```
输入 InputCapability → ShapeshiftingComponent.InputShapeRequest
   或 TriggerBox → ForceShapeOverride
   → ShapeshiftingCapability.ShouldActivate：检查 ShapeTypeIsBlocked / 碰撞
   → MorphCapability 过渡动画 → 切换 Small/BigFormSheet 能力表
   → 新形态 Character 激活其 Capability 组
   （失败 → MorphFailCapability 反馈）
```

---

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 被复用 | `River/`（38 处）、`Crack/`（21 处引用 Shapeshift；30 处引用具体形态） | 河流 / 冰裂 | 两大机制的谜题都要求特定形态，变形是通用底座 |
| 依赖 | `UPlayerMovementComponent`、`UPlayerPoleClimbComponent`、`UPlayerGrappleComponent`、`UPlayerWallRunComponent`、`UPlayerSwimmingComponent`、`UPlayerSwingComponent` | Gameplay/Move | 变形组件缓存这些玩家移动组件，形态切换时改写重力/终端速度等 |
| 依赖 | `UInteractionComponent` / `UOneShotInteractionComponent` | Gameplay | 形态绑定交互点走通用交互系统 |
| 输出 | 冰王 Boss | River | 树守 `HoldDownIceKing`、雪猴 `IceKingBossPunch` 是 Boss 战招式 |
| 输出 | 生长植物 | River / Crack | 树守赋生 → `TundraRiver_GrowingFlower`、`TundraGrowingPlant` |
| 标签互斥 | `TundraShapeshiftingTags::*` + `SmallFormSheet`/`BigFormSheet` | Core/Tags | 形态能力用能力表整表切换 + 标签互斥 |

---

## 关键文件

- `Shapeshifting/TundraPlayerShapeshiftingComponent.as` — 变形核心状态容器 + 形态枚举。
- `Shapeshifting/TundraPlayerShapeshiftingCapability.as` — 变形入口能力。
- `Shapeshifting/PlayerShapeBaseComponent.as` — 形态子组件基类。
- `Shapeshifting/Otter/PlayerOtterActor.as`、`Fairy/PlayerFairyActor.as`、`SnowMonkey/PlayerSnowMonkeyActor.as`、`TreeGuardian/PlayerTreeGuardianActor.as` — 四形态角色。
- `Shapeshifting/TreeGuardian/RangedInteractions/` — 树守远程交互子系统。
- `Shapeshifting/ShapeshiftingTriggers/TundraShapeshiftingTriggerBox.as` — 强制变形区。
- `Shapeshifting/TundraPlayerShapeshiftingSettings.as` / `TundraShapeshiftingStatics.as` — 设置/静态四件套。

# LevelSpecific / Island / Jetpack（喷气背包 + 悬浮踏板）

> 路径：`LevelSpecific/Island/Jetpack/`（31 文件）+ `LevelSpecific/Island/HoverPerch/`（32 文件）
>
> 本篇合并两套垂直/穿梭移动机制：**喷气背包 Jetpack**（燃料悬浮 + 冲刺 + 相位穿墙）与 **悬浮踏板 HoverPerch**（磨轨滑行）。

## 职责

- **Jetpack**：玩家在高塔间垂直穿梭的燃料飞行器。悬浮消耗燃料、冲刺加速、可穿过特定相位墙（Phasable Wall）；着地回充燃料。带横版段（Sidescroller）特化燃料 UI 与教学。
- **HoverPerch**：玩家踩上的悬浮踏板，沿磨轨样条 `GrindSpline` 高速滑行，含冲刺、撞击障碍加速、跳跃、相机转向等磨轨玩法。

## 内部架构

### Jetpack（31）

- **实体** `AIslandJetpack : AHazeActor`：网格 + 燃料表材质 `FuelMeterMaterial`（按状态染色：Boosting / Charged）+ 世界空间燃料 UI。全局工具 `IslandToggleJetpack(bToggleOn, PlayersToToggle)` 按 `EHazeSelectPlayer` 给 Mio/Zoe 开关背包。
- **玩家组件** `UIslandJetpackComponent : UActorComponent`：燃料中枢，管理充/耗、燃料表 UI 刷新、`Trigger_FuelEmpty` 事件。相位穿墙状态在 `IslandJetpackPhasableComponent`。
- **能力** `Capabilities/`：`Activation`（主控，`CapabilityTags = Movement/GameplayAction/Jump/AirJump`，**大量 `BlockedWhileIn`** 屏蔽 WallRun/WallScramble/Swimming/Grapple/Ladder/PoleClimb/Perch/LedgeGrab/Swing/Vault/LedgeMantle，`NetworkMode = Crumb`）、`Hover`（悬浮）、`Boost` / `Dash`（+`DashFOVIncrease`）、`LandRecharge`（着地回燃料）、`Attachment`、`Pickup`；相位穿墙 `Phasable Movement / Slowdown / PhaseWallBlockShooting / PhaseWallFOVIncrease`。
- **拾取与场景** `Pickup` / `Holder`（+交互能力）/ `RespawnPoint` / `ChargeDepletionTrigger`（耗尽触发）/ `HatchPlatform` / `FloatingBlimp` / `PanelIndicator`。
- **横版 UI** `FuelWidget/`：`SidescrollerFuelWidget`(+Capability) 与 `WorldSpaceFuelWidget`；`Tutorial/` 三个横版教学（激活 / 助推 / 取消）。

### HoverPerch（32）

- **实体与组件** `Actors/`：`HoverPerchActor : AHazeActor`（踏板本体，带磨轨/撞墙/冲刺相机抖动）、`HoverPerchComponent`、`HoverPerchMovementComponent`、`HoverPerchPlayerComponent`（玩家侧）。磨轨样条 `HoverPerchGrindSpline`(+`ConnectionGrindSpline` 连接)，重生 `GrindSplineRespawnManager / Point / Volume`，`ResetLocation`。`HoverPerchActorSweepingResolver` 扫掠移动解算。
- **能力** `Capabilities/`：`GrindSplineCapability`（磨轨主控）、`Movement`(+Input)、`Dash`(+Input)、`Bump`（撞击弹开）、`HitObstacle`(+Boost)、`WorldCollision`、`FollowPlayer`、`ResetHeightMovement`、跳跃时屏蔽移动输入、磨轨时屏蔽他人重生。`Capabilities/Player/`：磨轨相机（`Camera` / `FocusActor` / `Turn180`）+ 空中跳跃/教学。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| 依赖 | Core 移动/标签互斥 | `BlockedWhileIn::*` | `IslandJetpackActivationCapability` 屏蔽全部攀爬/游泳/摆荡移动，独占移动态 |
| 依赖 | Core 联机 | Crumb 同步 | 主控能力 `NetworkMode = EHazeCapabilityNetworkMode::Crumb` |
| → | 相位墙机关 | 相位穿墙 | `IslandJetpackPhasableComponent` + `Phasable*` 能力，穿墙时屏蔽射击、拉 FOV |
| → | 射击系统 | 状态门控 | `PhaseWallBlockShootingCapability` 穿墙期间屏蔽武器 |
| 依赖 | 磨轨样条 | GrindSpline | HoverPerch 沿 `HoverPerchGrindSpline` 滑行，`GrindSplineRespawnManager` 处理坠落重生 |
| → | 玩家身份 | 按玩家开关 | `AIslandJetpack::IslandToggleJetpack` 按 `EHazeSelectPlayer`(Mio/Zoe/Both) 挂/卸背包 |

## 关键文件

- `LevelSpecific/Island/Jetpack/IslandJetpack.as` — 背包实体 + 全局开关 `IslandToggleJetpack`
- `LevelSpecific/Island/Jetpack/IslandJetpackComponent.as` — 燃料中枢组件
- `LevelSpecific/Island/Jetpack/Capabilities/IslandJetpackActivationCapability.as` — 主控 + 标签互斥
- `LevelSpecific/Island/Jetpack/IslandJetpackPhasableComponent.as` — 相位穿墙状态
- `LevelSpecific/Island/HoverPerch/Actors/HoverPerchActor.as` — 悬浮踏板实体
- `LevelSpecific/Island/HoverPerch/Capabilities/HoverPerchGrindSplineCapability.as` — 磨轨主控
- `LevelSpecific/Island/HoverPerch/Actors/HoverPerchGrindSpline.as` — 磨轨样条

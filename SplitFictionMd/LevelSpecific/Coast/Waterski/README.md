# LevelSpecific / Coast / Waterski（滑水）

> 路径：`LevelSpecific/Coast/Waterski/`（22 文件）

## 职责

追逐序列第三段。翼装落水后无缝衔接为**拖曳滑水**：玩家被绳索牵引贴水面高速前进，可左右摆荡、跳跃越浪、撞浪减速/死亡。上承翼装（WingSuit）、下接空中 Boss 战。

## 内部架构

- **载具实体** `ACoastWaterskiActor : AHazeActor`：滑水板/绳索的表现载体；`CoastWaterskiAttachPointActor` 是绳索牵引锚点，`CoastFloatingMesh` 水面浮物。
- **玩家侧** `UCoastWaterskiPlayerComponent : UActorComponent`：持 `WaterskiActorClass` 并生成，管理相机抖动等。能力集 `Capabilities/`：
  - `CoastWaterskiCapability`（主控，`CapabilityTags.Add(n"Waterski")`，`NetworkMode = Crumb`）
  - `Movement`（`CoastWaterskiMovementResolver : USweepingMovementResolver` 解算）、`Camera`、`Jump`（越浪）、`Death`、`BlockRespawn`。
- **段落衔接** `CoastWingsuitToWaterskiTransitionCapability`：持有 `UCoastWaterskiPlayerComponent` 与 `UWingSuitPlayerComponent` 两个组件，翼装↔滑水互转（与 WingSuit 的成对 Transition 呼应）。
- **场景与规则** `CoastWaterskiManager` 编排；`BoostZone`（加速区）、`BlockRespawnZone`、`WaveCollisionComponent`（撞浪碰撞）、`ForceDeathComponent` / `IgnoreDeathComponent`（强制/免疫死亡）、`CoastBossPillarTarget`（衔接 Boss 段的柱状目标）。
- **表现** `CoastWaterskiEffectHandler` + `ActivateRopeAnimNotify`（动画通知激活绳索）。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| 依赖 | Core 移动 | SweepingResolver | `CoastWaterskiMovementResolver : USweepingMovementResolver` |
| 依赖 | Core 联机 | Crumb 同步 | `CoastWaterskiCapability` `NetworkMode = EHazeCapabilityNetworkMode::Crumb` |
| 衔接 | Coast/WingSuit | 形态互转 | `CoastWingsuitToWaterskiTransitionCapability` 持 `UWingSuitPlayerComponent` + `UCoastWaterskiPlayerComponent` |
| 衔接 | Coast/Boss | 进入 Boss 段 | `CoastBossPillarTarget` 作为滑水段末尾/Boss 段目标 |
| 生成 | 载具实体 | Player 组件 | `UCoastWaterskiPlayerComponent` 持 `WaterskiActorClass` 生成滑水板 |
| → | 浪花碰撞 | 减速/死亡 | `WaveCollisionComponent` + `ForceDeathComponent` 撞浪判定 |

## 关键文件

- `LevelSpecific/Coast/Waterski/CoastWaterskiActor.as` — 滑水载具实体
- `LevelSpecific/Coast/Waterski/CoastWaterskiPlayerComponent.as` — 玩家挂接 + 生成
- `LevelSpecific/Coast/Waterski/Capabilities/CoastWaterskiCapability.as` — 主控能力
- `LevelSpecific/Coast/Waterski/CoastWaterskiMovementResolver.as` — 移动解算（SweepingResolver）
- `LevelSpecific/Coast/Waterski/Capabilities/CoastWingsuitToWaterskiTransitionCapability.as` — 翼装↔滑水衔接
- `LevelSpecific/Coast/Waterski/CoastWaterskiManager.as` — 段落编排

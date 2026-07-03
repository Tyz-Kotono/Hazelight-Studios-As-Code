# LevelSpecific / Sanctuary / Aviation（骑乘飞行）

> 路径：`LevelSpecific/Sanctuary/Aviation/`（72 文件）

## 职责

圣域的招牌骑乘玩法。玩家骑上巨型伙伴 MegaCompanion（可在鸟形态与其它形态间变化）在空中飞行，沿样条推进、俯冲发起攻击、双人协同（Co-op Attack）配合发力。玩法分**飞行移动**与**俯冲攻击**两大循环，并按 Boss 战阶段（Phase1 / Phase2）切换目标与镜头。全段大量用于九头蛇 Boss 战的空中阶段。

## 内部架构

- **玩家侧** `SanctuaryCompanionAviationPlayerComponent : UActorComponent`（骑乘中枢）+ `PlayerEventHandler` + `Data` / `Settings` / `Statics` + `CapabilitiesData`（能力表数据）。双人按键连打 UI `TwoPlayerButtonMashWidget`。
- **巨型伙伴** `Actors/MegaCompanion/`：`ASanctuaryMegaCompanion : AHazeActor`（`BaseMovementTag = LocomotionFeatureAITags::Flying`，物理动画）+ 攻击盘 `MegaCompanionAttackDisc` + 玩家组件 `MegaCompanionPlayerComponent`。骑乘能力链：`EnableMegaCompanion` / `RideMegaCompanion` / `MegaCompanion`(主控) / `MegaLerpRidingOffset`（骑乘位插值）/ `MegaGoingSoloSpline`（单飞样条）。
- **能力分层** `Capas/`：
  - `Movement/`：`Movement`(主)、`Entry`(进入)、`Attack`(攻击移动)、`Lerp`、`Spline`、`SidewaySwing`（横摆）、`ToAttack`。
  - `Attack/`：`InitiateAttack` + `InitiateAttackWindow`（攻击窗口）、`ShowAttackDisc`（显示攻击盘）、`Attack`、`ToAttackDestination`；`Utility/` 协同攻击 `CoopAttackWidget` / `InputCoopAttack` + 教学。
  - `Camera/`：攻击相机进/出（`InitAttackCamera` / `AttackCamera` / `ExitAttackCamera` / `ToAttackCamera`）、`SwoopBack`（俯冲返回）、横版镜头 `BossArenaSidescrollCameraFocus` / `DisableSidescrolling`。
  - `Dest/`：目标点分配——`AssignPhase1AttackHydraDestination` / `AssignPhase1SwoopIn` / `AssignPhase1SwoopBack` / `AssignPhase2Destination` / `AssignToAttack`（含 `Phase1DestinationComponent` + `Phase1LerpDestinationToQuadCenter`）。
  - `StartStop/`：`Start` / `EvaluateStop` / `RideReady` / `Skydive`（跳伞进入）+ 教学。
  - `Utility/`：动画、禁用伙伴激活、Phase1/Phase2 禁死、横版锁定。
- **场景与相机** `Actors/`：着陆点 `LandingPoint`、俯冲序列 `SwoopSequence`、攻击相机 `AttackCamera` / `PostAttackCamera` / `ToAttackCameraFocusActor`。
- **教学入口** `Tutorial/`：`AviationTutorialSpline` / `Hinder` / `ReferencesActor`，圣域神殿入口 `HydraTempleEntrancePodium` / `HydraTempleGate`。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| 依赖 | Core/AI 移动 | Flying 运动特征 | `ASanctuaryMegaCompanion` `BaseMovementTag = LocomotionFeatureAITags::Flying` |
| 依赖 | Core 能力四件套 | Capability/Component/Settings/Data | `Capas/` 按 Movement/Attack/Camera/Dest/StartStop 分层组织 |
| → | Sanctuary/Boss | 空中阶段进攻 | `Dest/` 分配 Phase1/Phase2 目标点，俯冲攻击九头蛇 Hydra |
| 协同 | 双人玩家 | Co-op Attack | `Utility/InputCoopAttackCapability` + `CoopAttackWidget` + `TwoPlayerButtonMashWidget` |
| → | Core 相机 | 攻击镜头 + 横版 | `Camera/` 攻击相机切换 + `BossArenaSidescroll*` 横版聚焦 |
| 生成 | 骑乘实体 | Player 组件 | `SanctuaryCompanionAviationPlayerComponent` 管理骑乘接入 |

## 关键文件

- `LevelSpecific/Sanctuary/Aviation/SanctuaryCompanionAviationPlayerComponent.as` — 骑乘玩家中枢
- `LevelSpecific/Sanctuary/Aviation/Actors/MegaCompanion/SanctuaryMegaCompanion.as` — 巨型伙伴实体
- `LevelSpecific/Sanctuary/Aviation/Actors/MegaCompanion/SanctuaryCompanionAviationRideMegaCompanionPlayerCapability.as` — 骑乘接入
- `LevelSpecific/Sanctuary/Aviation/Capas/Movement/SanctuaryCompanionAviationMovementCapability.as` — 飞行移动主控
- `LevelSpecific/Sanctuary/Aviation/Capas/Attack/SanctuaryCompanionAviationAttackCapability.as` — 俯冲攻击
- `LevelSpecific/Sanctuary/Aviation/Capas/Dest/` — Phase1/Phase2 目标点分配
- `LevelSpecific/Sanctuary/Aviation/Capas/Attack/Utility/SanctuaryCompanionAviationInputCoopAttackCapability.as` — 双人协同攻击

# LevelSpecific / Prison / Stealth（含 MaxSecurity / CellBlock）— 潜行、安保与牢区

> 路径：`LevelSpecific/Prison/Stealth/`（42）、`MaxSecurity/`（52）、`CellBlock/`（23）
>
> Prison 越狱主线的三段"安保穿越"体验，合篇讲述：
> - **Stealth**：躲避守卫巡逻与监控视锥的潜行段（纸箱伪装、击晕）。
> - **MaxSecurity**：最高安保区的激光地狱（激光墙、激光切割机、可黑客机械）。
> - **CellBlock**：越狱开场的牢区（牢门、囚犯运输、垃圾滑道/压缩机等可黑客物件）。

承接 [Prison 总览](../README.md)。

---

## Stealth — 潜行（42）

### 职责
实现敌人视野检测 + 玩家潜行躲避的一整套：视锥、检测状态机、纸箱伪装、击晕。

### 内部架构

| 子域 | 关键类 | 基类 | 说明 |
|---|---|---|---|
| 敌人基类 | `APrisonStealthEnemy` | `AHazeActor` | 潜行敌人根类，挂检测/事件/开火能力 |
| 守卫 | `APrisonStealthGuard` | `APrisonStealthEnemy` | 巡逻守卫，`Guard/Capabilities/`：Move/PatrolSpline/PatrolWait/Search/Stunned；`PrisonStealthGuardPatrolComponent` 存巡逻路径 |
| 监控摄像头 | `APrisonStealthCamera` | `APrisonStealthEnemy` | 固定视锥敌人，`Camera/Capabilities/`：Idle/Search/Stunned |
| 视野 | `UPrisonStealthVisionComponent`（`USceneComponent`）、`APrisonStealthVisionCone`、`PrisonStealthVisionCapability` | — | 视锥检测核心 |
| 玩家检测状态 | `UPrisonStealthDetectionComponent`、`UPrisonStealthStunnedComponent` | `UActorComponent` | 被发现进度 / 被击晕状态 |
| 纸箱伪装 | `APrisonStealthCardboardBox`（`AHazeActor`）+ `CardboardBox/Capabilities/`（Attached/Respawn/Simulated）+ `...PlayerComponent` | — | 躲进纸箱规避视野 |
| 可击晕机器人 | `APrisonStealth_ShootableRobot`、`Enemy/Capabilities/ShootPlayer` | — | 可被击晕/射击的敌方单位 |
| 管理 | `PrisonStealthManager`、`PrisonStealthLevelEvents`、`PrisonStealthStatics` | — | 关卡编排与静态 |

数据流：`VisionComponent` 视锥命中玩家 → `DetectionComponent` 累积发现度 → 触发敌人 `Search`/`ShootPlayer`；玩家进纸箱或击晕敌人（`StunnedComponent`）打断。

---

## MaxSecurity — 最高安保激光区（52）

### 职责
高压激光穿越段：多种激光装置 + 激光切割机 Boss 式追逐 + 可黑客机械。

### 内部架构

| 子域 | 关键类 | 说明 |
|---|---|---|
| 激光装置 | `Laser/`：`AMaxSecurityLaser` + `UMaxSecurityLaserComponent`、`MaxSecurityLaserDeathVolume`、`LaserGroupDisabler`、`StaticMaxSecurityLaser`、`BackdropLaser` | 激光墙/束，命中即死 |
| 激光地狱 | `LaserHell/`：`MaxSecurityLevelScriptActor`、`LaserHellDoor`/`Elevator`/`LaserAlarm`、`PressurePlate` | 激光房间关卡脚本 |
| 激光切割机 | `LaserCutting/`：`AMaxSecurityLaserCutter` + `...Clamp`/`Crosshair`/`WeakPoint`/`Laser`/`WeldingBot`/`MagneticCover`/`PathPiece`；`Capabilities/`（Cutter/CutterLaser/Stunned） | 追逐式激光切割机，含弱点与磁力盖 |
| 场景机关 | `MaxSecurityVaultDoor`、`PillarElevator`(+Button)、`FloorGate`、`ExplosiveSwingPoint`、`LaserShaftSpinner`、`RemoteHackableMachinery/`（Cell/ControlPanel） | 金库门、柱式电梯、可黑客机械 |

---

## CellBlock — 牢区（23）

### 职责
越狱开场的牢区场景物件，多为可远程黑客/机械交互的独立 actor。

### 关键 actor

| 类 | 说明 |
|---|---|
| `RemoteHackableCellDoor` / `RemoteHackableToilet` | 可黑客牢门 / 马桶 |
| `PrisonerTransportPlatform/`（`APrisonerTransportPlatform` + MoveCapability + DisableComponent） | 囚犯运输平台 |
| `PrisonerChute/` | 囚犯滑道 |
| `TrashCompactor/`（`ATrashCompactor` + Door + HackableFuseBox + GarbageTruckTrash） | 垃圾压缩机（含可黑客保险丝盒） |
| `CellOnTrack`、`CellBlockFlyingPlatform`、`CellBlockGuardSpawner`、`CellBlockJumpScarePrisoner`、`PrisonPerchSpiral`、`TelescopeRobotGate`、`TrashChuteFlap` | 轨道牢房、飞行平台、守卫刷怪、惊吓囚犯、栖停螺旋等 |

---

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 复用 | 远程黑客系统 | LevelActors/RemoteHackables | CellBlock/MaxSecurity 大量 `RemoteHackable*` actor 走统一黑客交互 |
| 交叠 | 蜂群无人机 | Drones | Stealth 段出现 `SwarmBoat`/蜂群单位（`LevelActors/Stealth/SwarmBoat`、`SwarmDroneDancing`） |
| 依赖 | `AHazeActor` + Capability | Core | 敌人/激光切割机全用能力四件套 + 标签互斥切状态 |
| 依赖 | 巡逻样条 / 相机 | Gameplay | 守卫巡逻走样条系统，检测走相机/感知系统 |

---

## 关键文件

- `Stealth/Enemy/PrisonStealthEnemy.as` — 潜行敌人基类。
- `Stealth/Vision/PrisonStealthVisionComponent.as` — 视锥检测核心。
- `Stealth/CardboardBox/PrisonStealthCardboardBox.as` — 纸箱伪装。
- `MaxSecurity/LaserCutting/MaxSecurityLaserCutter.as` — 激光切割机。
- `MaxSecurity/Laser/MaxSecurityLaser.as` — 激光装置。
- `CellBlock/TrashCompactor/TrashCompactor.as` — 垃圾压缩机。
- `CellBlock/PrisonerTransportPlatform/PrisonerTransportPlatform.as` — 囚犯运输平台。

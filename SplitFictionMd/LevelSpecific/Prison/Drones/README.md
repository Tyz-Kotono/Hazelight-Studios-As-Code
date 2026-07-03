# LevelSpecific / Prison / Drones — 双人无人机

> 路径：`LevelSpecific/Prison/Drones/`（193 文件）
>
> Prison 的核心载具：两名玩家分别操控**磁力无人机（MagnetDrone）**与**蜂群无人机（SwarmDrone）**。磁力无人机能吸附金属表面攀爬跳跃；蜂群无人机能滑翔、劫持关卡单位、驾船。是"双人异步协作载具"玩法。

承接 [Prison 总览](../README.md)。本机制被 [Pinball 弹球](../Pinball/) 大量复用。

---

## 职责

实现两种可驾驶无人机的移动、能力与关卡交互，并提供两者共享的移动/组件基底。

---

## 内部架构

### 共享基底（Shared/）

两种无人机共用的玩家侧数据与移动能力：

| 类 | 基类 | 说明 |
|---|---|---|
| `UDroneComponent` | `UActorComponent` | 挂在玩家上的无人机数据容器（相机弹簧臂、碰撞体、冲刺曲线等）；`access Resolver` 私有暴露给 `UDroneMovementResolver` |
| `UDroneMoveCapability` | `UHazePlayerCapability` | 地面移动，`NetworkMode=Crumb`，用 `SetupMovementData(UDroneMovementData)` |
| `UDroneMoveAirCapability` / `UDroneDashCapability` / `UDroneSwarmHoverDashCapability` | `UHazePlayerCapability` | 空中移动、冲刺、蜂群悬停冲刺 |
| `UDronePossessionCapability` / `UDroneBoxCapability` | `UHazePlayerCapability` | 附身/进入无人机、载箱 |
| `UDroneMovementResolver` / `UDroneMovementData` | — | 移动解算与数据 |

共享标签：`PrisonTags::Prison`、`PrisonTags::Drones`、`DroneCommonTags::BaseDroneMovement`/`BaseDroneGroundMovement`。

### 磁力无人机（Magnet/）— 最大子模块

磁力吸附是核心玩法，`Magnet/` 展开为完整子系统：

| 子域 | 内容 |
|---|---|
| `Attraction/` | 吸引核心：`AttractionModes/`（多种吸附模式 + Capabilities）、`StartAttract/` |
| `Attached/` | 吸附后：`Camera/`（含 Surface 表面相机）、`Socket/`、`Surface/`、`Components/` |
| `Aim/` | 磁力瞄准 |
| `Jumping/` | `AttractJump/`（吸附跳）、`ChainJump/`（连跳） |
| `Bounce/`、`Movement/`、`ProcAnim/` | 弹跳、移动、程序动画 |
| `Actors/` | `MagnetDroneCatapult`（弹射器）、`MagnetDroneSwitch`（开关）、`MagnetDroneBreakable`、`MagnetDroneRotatingArm`、`MagnetDroneSocket` 等台面可交互物 |
| `AttachToBoat/` | 吸附到船 |

### 蜂群无人机（Swarm/）

| 子域 | 内容 |
|---|---|
| `Glider/` | 滑翔：`SwarmDroneGliderCannon`（滑翔炮）+ Capabilities/Components |
| `Capabilities/Hijack/`、`Actors/Level/Hijack/` | **劫持**关卡单位：`ASwarmDroneHijackable`（基类 `AHazeActor`）、`ASwarmDroneGroundMovementHijackable`、`ASwarmDroneSimpleMovementHijackable` |
| `Boat/` | 驾船：Capabilities/Components/EventHandlers |
| `Components/Airduct/` | 通风管道穿越 |
| `Bounce/`、`Parachute/`、`Traversal/` | 弹跳、降落伞、穿越 |

### 数据流（磁力吸附）

```
玩家输入 → UDroneMoveCapability(地面) / MoveAirCapability(空中)
   → 靠近金属表面 → Magnet/Attraction StartAttract 激活
   → Attached：Socket 吸附 + Surface 相机切换
   → Jumping：AttractJump / ChainJump 在表面间弹跳
```

---

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 被复用 | `Pinball/MagnetDrone/` | 弹球 | 弹球段玩家单元整体照搬 `Magnet/`（Aim/Attached/Attraction/StartAttract 同构） |
| 交互 | `LevelActors/DroneHackables/` | LevelActors | 磁力无人机吸附/触发起重臂、狙击炮塔、电板、水齿轮等可黑客物 |
| 依赖 | `UHazeMovementComponent` + `UDroneMovementData` | Core/Move | 所有无人机移动经 Haze 移动组件解算 |
| 交叠 | `SwarmBoat` / `Stealth` | 潜行 | 潜行段出现蜂群船（`LevelActors/Stealth/SwarmBoat`） |
| 标签互斥 | `PrisonTags::Drones` + `DroneCommonTags::*` | Core/Tags | 磁力/蜂群/地面/空中状态用标签互斥 |

---

## 关键文件

- `Drones/Shared/Components/DroneComponent.as` — 玩家侧无人机数据容器。
- `Drones/Shared/Capabilities/DroneMoveCapability.as` — 共享地面移动能力。
- `Drones/Shared/Movement/DroneMovementResolver.as` — 移动解算。
- `Drones/Magnet/Attraction/` — 磁力吸附核心。
- `Drones/Magnet/MagnetDroneStatics.as` — 磁力无人机静态。
- `Drones/Swarm/Actors/Level/Hijack/SwarmDroneHijackable.as` — 蜂群劫持基类。
- `Drones/Swarm/Glider/` — 蜂群滑翔。

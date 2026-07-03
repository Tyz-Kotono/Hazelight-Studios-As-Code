# LevelSpecific / Skyline / 飞行车（FlyingCar）

> 路径：`LevelSpecific/Skyline/FlyingCar/`（74 文件，含横版变体 `FlyingCar2D/` 5 + 追逐牵引 `CarChaseTether/` 5）
>
> 双人飞车高速公路追逐战：一名玩家开车（驾驶员/Pilot），一名玩家操炮（炮手/Gunner），躲避障碍、击毁追兵、贯穿霓虹高速公路。

承接 [Skyline 关卡总览](../README.md)。能力四件套/标签范式见 [Gameplay 设计哲学](../../../Gameplay/DesignPhilosophy.md)，不重复。

---

## 职责

飞行车是一辆 `AHazeActor` 载具（不是角色），有驾驶员座与炮手座。驾驶员用 `UHazePlayerCapability` 附着并转发输入驱动车辆移动，炮手用另一套能力操作车载武器。车辆移动复用 Gameplay 的扫掠移动求解框架（与重力摩托同源）。

---

## 内部架构

### 载具本体（`FlyingCar/` 根）

- **`ASkylineFlyingCar : AHazeActor`**（Abstract）— `TG_PostPhysics` tick；`access Resolver = private, USkylineFlyingCarMovementResolver`。结构：`SphereCollision` + `MeshOffset`（`UHazeOffsetComponent`，做 roll 偏移）+ `Mesh` + `PilotSeat` / `PassengerSeat` 两个座位 + VFX 根。`ASkylineFlyingCarStarter` 触发上车。
- **移动（`Movement/`）**：`USkylineFlyingCarMovementResolver : USweepingMovementResolver` + `USkylineFlyingCarMovementData : USweepingMovementData`——与重力摩托共享 Gameplay 扫掠移动系统。
- **驾驶员能力（`Capabilities/Pilot/`）**：`FlyingCarPilotCapability`（`UHazePlayerCapability`，主驾驶）、`PilotCameraPreMovementCapability` / `PilotCameraPostMovementCapability`（前后置相机）。`Capabilities/Car/` 存车辆自身能力（含 `GotyStyle/` / `OldPlayStyle/` 两套操控风格、`Debug/`）。
- **驾驶员/座舱**：`SkylineFlyingCarPilot`、`SkylineCarPlatform`、`SkylineFlyingHighway` / `SkylineFlyingCarHighwayRamp`（高速公路与匝道）、`SkylineFlyingCarGroundMovementTrigger`。

### 炮手系统（`Gunner/`）

炮手在副驾操作车载武器击毁目标：

- **武器（`Gunner/Capabilities/`）**：`Rifle/`（步枪，速射）、`Bazooka/`（火箭筒，锁定爆发）。
- **弹体/目标**：`SkylineFlyingCarGun`、`SkylineFlyingCarGunnerProjectile`，`Gunner/Actors/`、`Gunner/Components/`、`Gunner/Widgets/`（准星/锁定 UI）；武器设定 `FlyingCarGunnerWeaponSettings`（Rifle/Bazooka 各一份 `UHazeComposableSettings`）。

### 车辆组件与响应（`Components/`）

- 血量 `SkylineFlyingCarHealthComponent`（+ `UFlyingCarHealthSettings`）、撞击 `ImpactResponseComponent`、被炮击响应 `BazookaResponseComponent` / `GunResponseComponent`、可瞄准目标 `BazookaTargetableComponent : UTargetableComponent` / `RifleTargetableComponent : UAutoAimTargetComponent`。

### 障碍/破坏/演出

- `HighwayObstacles/`（公路障碍）、`Destructibles/`（可破坏物）、`SkylineFlyingCarShootableDoor`（可射门）、`OfficeAssistedJump/`（办公室助跳段，含 `Capabilities/`）、`OfficeCrash/`（撞入办公室演出）、`EventHandlers/`。

### 关联子段

- **`FlyingCar2D/`（5）** — 飞车 2D 横版变体：`ASkylineFlyingCar2D` + `SkylineFlyingCar2DPilotComponent` + `SkylineFlyingCar2DTractorBeam`（牵引光束）。
- **`CarChaseTether/`（5）** — 追逐牵引段：`UCarChaseTetherPlayerSettings`（`UHazeComposableSettings`）把两车/玩家用绳索连接。配套 `CarChaseDive/`（俯冲）见 [Skyline 总览](../README.md)。

---

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 依赖 | `AHazeActor` | Core | `ASkylineFlyingCar` 为载具 actor，非角色 |
| 依赖 | `USweepingMovementResolver` / `USweepingMovementData` | Gameplay Move | 车辆移动复用扫掠移动求解，与重力摩托同源 |
| 依赖 | `UHazePlayerCapability` / `UHazeCapabilityComponent` | Gameplay | 驾驶员/炮手能力挂玩家容器（能力四件套） |
| 依赖 | `UHazeOffsetComponent` | Core | 车身 `MeshOffset` 做 roll 偏移视觉 |
| 依赖 | `UTargetableComponent` / `UAutoAimTargetComponent` | Core 目标系统 | 炮手 Bazooka/Rifle 目标锁定与自动瞄准 |
| 依赖 | `UHazeComposableSettings` | Gameplay | `FlyingCarGunnerWeaponSettings`、`CarChaseTetherPlayerSettings` |
| 双人 | Pilot + Gunner 座位 | Skyline | `PilotSeat` / `PassengerSeat` 分工，延续双人协作 |
| 关联 | `FlyingCar2D` / `CarChaseTether` / `CarChaseDive` | Skyline | 追逐段的横版/牵引/俯冲变体 |
| 特效 | `UHazeEffectEventHandler` | Core | `SkylineFlyingCarEventHandler` 等处理撞击/射击表现 |

---

## 关键文件

- 载具：`FlyingCar/SkylineFlyingCar.as`、`FlyingCar/Movement/SkylineFlyingCarMovementResolver.as`、`FlyingCar/Movement/SkylineFlyingCarMovementData.as`
- 驾驶员：`FlyingCar/Capabilities/Pilot/SkylineFlyingCarPilotCapability.as`、`FlyingCar/Capabilities/Car/`
- 炮手：`FlyingCar/Gunner/Capabilities/Bazooka/`、`FlyingCar/Gunner/Capabilities/Rifle/`、`FlyingCar/Gunner/FlyingCarGunnerWeaponSettings.as`
- 车辆组件：`FlyingCar/Components/SkylineFlyingCarHealthComponent.as`、`FlyingCar/Components/SkylineFlyingCarBazookaTargetableComponent.as`
- 关联段：`FlyingCar2D/SkylineFlyingCar2D.as`、`CarChaseTether/CarChaseTetherPlayerSettings.as`

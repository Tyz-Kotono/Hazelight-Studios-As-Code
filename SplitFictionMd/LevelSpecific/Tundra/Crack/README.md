# LevelSpecific / Tundra / Crack — 冰裂谷攀爬闯关

> 路径：`LevelSpecific/Tundra/Crack/`（150 文件）
>
> Tundra 的攀爬/移动闯关段：一道冰裂谷（Crack）与其中的常青林（Evergreen）。核心是用弹跳、弹射与各种移动装置在垂直/水平地形间穿梭——弹跳蘑菇、裂隙鸟（CrackBird）弹射、木桶滚射、拐杖（WalkingStick）移动等，配合变形形态解谜。

承接 [Tundra 总览](../README.md)。冰裂段大量引用[变形](../Shapeshifting/)形态（30 处引用具体形态）。

---

## 职责

实现冰裂谷的移动/弹射装置、裂隙鸟载具、常青林谜题群与多种特殊移动体（木桶、拐杖、雪橇）。

---

## 内部架构

Crack 由一批各自独立的移动/交互 actor 构成，主要分为几族：

### 裂隙鸟（CrackBird/）

可被玩家举起/放下、装上弹射器发射的大鸟：

| 类 | 基类 | 说明 |
|---|---|---|
| `ABigCrackBird` | `AHazeActor` | 大鸟本体，`Capabilities/Bird/`：InNest/OnCatapult/JumpOffCatapult/Launched/PickedUp/RunAway/WallImpact/Rotation 等状态能力 |
| `ABigCrackBirdCatapult` | `ASnowMonkeyCatapult` | 鸟弹射器（继承猴族弹射器） |
| `ABigCrackBirdNest`、`UBigCrackBirdCarryComponent`、`UBigCrackBirdInteractionComponent`(`UInteractionComponent`) | — | 鸟巢、被携带、交互 |
| `Capabilities/Player/` | `UHazePlayerCapability` | 玩家侧：Lift/Carry/PutDown/WalkToTarget CrackBird |

### 常青林（Evergreen/）— 最大子族

冰裂谷中段的森林谜题群，含滚球、活塞、发射植物、旋转木、生命管理等：`AEvergreenBall`、`EvergreenBigPiston`、`EvergreenBouncePad`、`EvergreenPlantProjectile`/`ShootingPlant`、`EvergreenSpinLog`/`SpinlogPlatform`、`EvergreenPoleCrawler`(+Group/SpawnManager)、`EvergreenLifeManager`、`EvergreenWaterTrap`/`SpikeTrap`/`SlopeObstacle`、`SchmellTower/`（塔楼拼装）、`Barrel/`（见下）。

### 木桶（Evergreen/Barrel/）

`AEvergreenBarrel`（`AHazeActor`）+ `UEvergreenBarrelPlayerComponent` + `Capabilities/`（Enter/Launch/LaunchBlockBarrel/Kill）——玩家钻进木桶被弹射的移动体。

### 拐杖（WalkingStick/）

`ATundraWalkingStick` + `UTundraWalkingStickContainerComponent`（组件容器）+ `UTundraWalkingStickMovementComponent` + 一整组状态能力（`Capabilities/`：Idle/Movement/Rise/Fall/ChargeScream/CrashInWall/CrashWithLegs/Death/SetLocation）——一种高脚移动载具，配 `TundraBeaverSpear`、`WalkingStickStatics`。

### 弹跳/弹射与其它移动体

| 类别 | 代表类 |
|---|---|
| 弹跳蘑菇 | `TundraBouncyMushroomActor` + `BounceCapability` + `PlayerComponent`（`UTundraMushroomPlayerBounceComponent`） |
| 发射台 | `LaunchPad/TundraCrackLaunchpad`(+AnimNotify)、`TundraCrackSpringLog`(+Nyparn)、`SnowMonkeyCatapult`、`MonkeySmasherForFairyLauncher` |
| 藤蔓/生长 | `FairyBranch/TundraFairyBranch`、`VineGrowerr`、`VineMoverCube`/`Sphere`、`MovingVine`、`TundraGrowingPlant`、`GrowingFlowers.as/TundraGrowingFlower`、`TundraBoulderLiftingPlant` |
| 树守相关 | `TreeGuardianClimbPoleMonkeySmash`、`TundraCrackTreeGuardianSinkingSplinePlatform`、`TundraCrackRootElevator`/`RootMonkeyHanger` |
| 摆荡 | `SwingVista/`（`TundraPlayerSwingComponent` + Fall/Grounded/Interaction/Launch/SnapPosition 能力 + `TundraSwing`）、`SwingPerch` |
| 雪橇/滚石 | `SnowSailer`、`TundraRollingBoulder`、`RotateSlammer`、`TiltBridgeInTheJungle` |
| 生物/诱饵 | `ScaredBird`、`TreeMoleFlee`、`TundraPondBeastBait`(+Gun/Trigger)、`OtterKingSpearFolk` |
| 支线 | `SideInteracts/`（`TundraSwing/`、`FairyGroundWarp`、`MonkeyKarmaLog`、`DancingEnt`、`SidePunchableTree`） |

### 数据流（裂隙鸟弹射示例）

```
玩家 LiftCrackBird → CarryCrackBird → 走到弹射器 WalkToTarget
   → 放上 Catapult（OnCatapult）→ 发射 Launched
   → 鸟带玩家飞越裂谷 → WallImpact / 落入 Nest
```

---

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 依赖 | 变形形态 | Shapeshifting | Crack 有 21 处引用 `Shapeshift`、30 处引用具体形态（Otter/SnowMonkey/Fairy/TreeGuardian）；仙灵发射器、树守爬杆等按形态解谜 |
| 复用 | `ASnowMonkeyCatapult` | 本机制 | `ABigCrackBirdCatapult` 继承猴族弹射器基类 |
| 依赖 | `UInteractionComponent` / 组件容器 | Gameplay/Core | 裂隙鸟交互、拐杖 `ContainerComponent` 走通用交互与容器范式 |
| 依赖 | `UPlayerMovementComponent` + AirMotion | Gameplay/Move | 蘑菇/发射台/木桶弹射走玩家移动与空中运动系统 |
| 输出 | 生长植物 | 本机制 | 树守赋生 → `TundraGrowingPlant`/`GrowingFlowers` |

---

## 关键文件

- `Crack/CrackBird/BigCrackBird.as` — 裂隙鸟本体与状态能力。
- `Crack/CrackBird/BigCrackBirdCatapult.as` — 鸟弹射器（继承 `ASnowMonkeyCatapult`）。
- `Crack/Evergreen/` — 常青林谜题群。
- `Crack/Evergreen/Barrel/EvergreenBarrel.as` — 木桶弹射移动体。
- `Crack/WalkingStick/TundraWalkingStick.as` — 拐杖移动载具（组件容器 + 状态能力）。
- `Crack/BouncyMushroom/TundraBouncyMushroomActor.as` — 弹跳蘑菇。
- `Crack/SwingVista/` — 摆荡观景段。

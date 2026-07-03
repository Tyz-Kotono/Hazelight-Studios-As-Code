# LevelSpecific / Skyline / 机械 Boss（Boss）

> 路径：`LevelSpecific/Skyline/Boss/`（336 文件，可含无人机 Boss `DroneBoss/` 26 文件）
>
> Skyline 的终局：一个多形态巨型机械 Boss。核心是一具会站立/倒下/崩解的巨型机器人（`ASkylineBoss`），配套球 Boss、坦克、飞行相等子战斗，玩家骑重力摩托/用重力武器逐段击破。

承接 [Skyline 关卡总览](../README.md)。能力四件套/标签范式见 [Gameplay 设计哲学](../../../Gameplay/DesignPhilosophy.md)，不重复。

---

## 职责

Boss 目录是 Skyline 最复杂的战斗集合，实为**多个独立 Boss 形态 + 一套状态机核心**。主 Boss `ASkylineBoss : AHazeActor` 用状态机在站立/攻击/倒下/崩解间切换；玩家借重力摩托把 Boss 表面当跑道贴面攻击。

---

## 内部架构

### 主 Boss 状态机（`Boss/` 根 + `Capabilities/` + `Core/`）

- **`ASkylineBoss : AHazeActor`** — 巨型机器人。状态枚举 `ESkylineBossState`、腿部 `ESkylineBossLeg`、倒地方向 `ESkylineBossFallDirection`。用 `access Compounds = private, USkylineBossCompoundCapability` 复合能力驱动。
- **表面即摩托跑道**：Boss 身上直接挂 `UGravityBikeFreeHalfPipeTriggerComponent` + `UGravityBikeFreeAutoSteerTargetComponent`（左右 Ramp Pivot）——玩家骑重力摩托沿 Boss 肢体飞驰。
- **状态子能力（`Capabilities/States/`）**：`Assemble`（组装）、`Rise`（起身）、`Combat`（战斗）、`PendingDown` → `Down`（倒下）、`Fall`（崩塌）、`Dead`（死亡），多个含 `Children/` 子状态。另有 `Capabilities/Attacks/`、`Capabilities/Movement/`、`Capabilities/BaseClasses/`。
- **核心/组件**：`Core/SkylineBossCoreComponent`、`SkylineBossManager`（`AHazeActor` 总控）、脚部移动 `SkylineBossFootMovementComponent` / `FootTarget` / `HalfPipeJumpComponent`、相机 `SkylineBossEventCameraComponent` / `CenterViewTargetComponent` / `FocusGetter`、护盾 `ForceFieldComponent`、样条 `SkylineBossSpline` / `SplineHub`、`SkylineBossSettings`（`UHazeComposableSettings`）。
- **攻击 actor/子模块**：`RocketBarrage`（火箭齐射）、`SeekerMissile`（追踪导弹）、`FocusBeam`（聚焦光束）、`FootStomp`（踩踏）、`ShockWave` / `PulseAttack`、`CarpetBomb`（地毯轰炸）、`ObeliskDrop`、`ProximityMine`、`StrafeRun`、`DeathSphere`、`Leg` / `Hatch` / `Vulcano`（部位）、`SkylineBossProjectile` 及 `GroundSweepingProjectile`。

### 子 Boss 形态

- **`BallBoss/`** — 球形 Boss（`ASkylineBallBoss : AHazeActor`），最大子系统：分位置/旋转/偏移/护盾/攻击/动作/音频等 `Capabilities/` 分组；大量攻击 actor（`CarSmash`/`LobbingCar`/`Motorcycles`/`RollingBuss`/`SlidingCar`/`Shockwave`/`ThrowableDetonators`）；含 `Chase/`（追逐 + 相机）、`Laser/`、`SmallBoss/`（跳跃/弹射循环）、`InsideAttackers` / `SurfaceAttackers`、`HardMode/`、`SkylineBallBossSoul`。`BallMine/` 为配套地雷。
- **`Tank/`** — 坦克 Boss：`Arena/` 竞技场，攻击组 `Attacks/`（`AutoCannon`/`SideCannon`/`Crusher`/`ExhaustBeam`/`MortarBall`/`TurretFocus`/`Trail`，各带 `Capabilities/`）。
- **`Chaser/`** — 追击者（`Capabilities/` + `EventHandlers/`）。
- **`SkylineBossFlyingPhase/`** — 飞行相（`Capabilities/`）。
- **`ArenaActors/`** — 战斗竞技场物件。

### 无人机 Boss（`DroneBoss/`，26）

独立的小型无人机 Boss，可作为 Boss 战一环：

- **`ASkylineDroneBoss : AHazeActor`** — 用 `UHazeCapabilityComponent` + `SkylineDroneBossCompoundCapability` 驱动；挂 `UGravityBladeGravityShiftComponent`（球形重力场，`bForceSticky`）与 `UGravityBladeGrappleComponent`（可被重力刀抓钩）。
- 配套：`Attachment`、`Hatch`、`HealthComponent`、`Phase`（`UDataAsset` 阶段）、`Settings`（`UHazeComposableSettings`）、`Tags`。

---

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 依赖 | `AHazeActor` | Core | `ASkylineBoss`、`ASkylineBallBoss`、`ASkylineDroneBoss` 等均继承 `AHazeActor` |
| 依赖 | `UHazeCapabilityComponent` + Compound | Gameplay | 主 Boss / DroneBoss 用 `*CompoundCapability` 复合能力驱动状态机 |
| 复用 | `UGravityBikeFreeHalfPipeTriggerComponent`、`AutoSteerTargetComponent` | Skyline GravityBike | Boss 表面即摩托半管跑道，玩家骑摩托贴面攻击 |
| 复用 | `UGravityBladeGrappleComponent` / `GravityShiftComponent` | Skyline GravityWeapons | DroneBoss 可被重力刀抓钩，自带球形重力场 |
| 依赖 | `UHazeComposableSettings` | Gameplay | `SkylineBossSettings`、`SkylineDroneBossSettings` |
| 依赖 | `UHazeSplineComponent` | Gameplay | `SkylineBossSpline` / `SplineHub` 编排 Boss 移动与相机 |
| 目标 | `UCenterViewTargetComponent` / `UAutoAimTargetComponent` | Core | Boss 相机聚焦与玩家自动瞄准 |
| 特效 | `UHazeEffectEventHandler` | Core | 各攻击 / Chaser / Tank EventHandler 处理命中表现 |

---

## 关键文件

- 主 Boss：`Boss/SkylineBoss.as`、`Boss/SkylineBossManager.as`、`Boss/Core/SkylineBossCoreComponent.as`、`Boss/Capabilities/States/`、`Boss/SkylineBossSettings.as`
- 攻击：`Boss/RocketBarrage/`、`Boss/SeekerMissile/`、`Boss/FocusBeam/`、`Boss/SkylineBossProjectile.as`
- 球 Boss：`Boss/BallBoss/SkylineBallBoss.as`、`Boss/BallBoss/AttackActors/`、`Boss/BallBoss/Chase/`
- 坦克：`Boss/Tank/`、`Boss/Tank/Attacks/`
- 无人机 Boss：`DroneBoss/SkylineDroneBoss.as`、`DroneBoss/SkylineDroneBossPhase.as`

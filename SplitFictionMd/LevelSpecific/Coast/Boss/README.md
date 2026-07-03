# LevelSpecific / Coast / Boss（空中弹幕 Boss + 肩炮）

> 路径：`LevelSpecific/Coast/Boss/`（96 文件）+ `LevelSpecific/Coast/ShoulderTurret/`（21 文件）
>
> 本篇合并追逐关的最终战：主 Boss（`CoastBoss`，可变形无人机群）、翼装 Boss（Helios）、玩家化身的射击无人机（Aeronautic），以及肩炮武器 `ShoulderTurret`。

## 职责

Coast 高潮弹幕空战。玩家化身射击无人机，被约束在一个 2D 平面（`CoastBoss2DPlane`）内躲弹幕并反击。主 Boss 在**两种形态间变形**——`BigGunDrone`（单体大炮无人机）与 `ManyDrones`（多无人机编队），各有一套攻击能力（弹球 / 弹幕磨盘 / 布雷）。中途穿插翼装 Boss（Helios）追击段与肩炮火力段。

## 内部架构

### 主 Boss `Boss/Boss/`

- **实体** `ACoastBoss : AHazeActor`：持各类攻击体子类（`BulletBall` / `BulletMill` / `BulletMine`）。`CoastBossDroneActor` + `DroneComponent` 是编队无人机；`FormationTemplate` 定义队形；`HealthBar` 组件 + Boss 血条 UI。数据/常量在关卡根 `CoastBossData` / `CoastBossConstants` / `CoastBossTypes` / `CoastBossActorReferences` / `CoastBossDevToggles`。
- **形态切换** `Capabilities/Attacks/CoastBossChangeShapeCapability : UHazeActionQueueCapability`（`SetNewFormation(ECoastBossFormation)`）+ `CoastBossChangePhaseCapability`（阶段推进）。攻击走**动作队列**（Action Queue）范式。
- **两形态攻击**：
  - `Attacks/BigGunDrone/`：`AttackWithGunsQueue` 编排，`GunShootBall` / `GunShootAutoBall` / `GunShootMill` / `GunShootMine` + `ToggleMineLauncherExtended` + `GunChangeMovement` / `MoveGunBoss` / `HideBoss` / `Death`。
  - `Attacks/ManyDrones/`：`AttackWithDronesQueue` 编排，`ShootBall` / `ShootMill` / `ShootMine` + `MoveManyDronesBoss`。
- **弹幕移动** `MoveBulletsCapability` / `MoveDronesCapability`（Boss 侧统一推进场上弹幕/无人机）。
- **强化道具** `PowerUps/`：`QueuePowerUps` / `SpawnPowerUp`；`Pickups/`：`MovePickups` + `PlayerNormalPowerUp` + 特效。
- **攻击体** `AttackActors/`：`BulletBall`（弹球）、`BulletMill`（旋转弹幕磨盘）、`BulletMine`（地雷）、`DrillbazzTelegraph`（攻击预告）。

### 玩家射击无人机 `Boss/Player/`（Aeronautic）

- **组件** `UCoastBossAeronauticComponent`（持 Mio/Zoe 各自子弹类 `MioBulletClass` / `ZoeBulletClass`）+ `PlayerSheet` + `EventHandler`。实体 `CoastBossPlayerDrone` + 子弹 `CoastBossPlayerBullet` + 激光 `CoastBossPlayerLaser`。
- **能力** `Player/Capabilities/`：移动（`MovementPlayer` / `MoveTo2DPlane` / `MoveToPortal` / `AttachedToDrone`）、射击（`Shoot` / `ShootHoming` + `LaserPowerUp` / `HomingPowerUp`）、`Dash`、护盾（`Shield` / `RegenerateShield` / `Invulnerable`）、伤害（`ResolveDamage` / `ScaleDamage` / `DamageFeedback`）、相机、移动弹幕 `MoveBullets`。

### 2D 约束平面 `Boss/Plane/`

`ACoastBoss2DPlane` + `MovementCapability` + `SizeCapability`：把玩家无人机限制在一张可移动/缩放的 2D 平面内飞行（弹幕游戏的经典约束）。

### 翼装 Boss `Boss/WingsuitBoss/`（Helios）

`AWingsuitBoss : AHazeActor`：翼装追击段的空中 Boss。攻击能力 `MachineGunAttack` / `MineLauncher` / `ShootAtTarget` / `TrainMissileAttack` / `MineBlockadeLaunch`；移动 `SplineMovement` / `StationKeepingMovement`。投射物族 `HeliosProjectile` / `WingsuitBossProjectile` / `Mine`(+Launcher/+Blockade) / `ShootAtTargetProjectile`(+ResponseComponent)。

### 肩炮 `ShoulderTurret/`（21）

`ACoastShoulderTurret : AHazeActor` + `UCoastShoulderTurretComponent`（挂玩家肩上，持 `TurretClass` + 准星 UI 类）。瞄准 `AimCapability` + 附着 `AttachCapability`。两种炮 `Gun/`：`Cannon/`（实弹，`FireCapability` + `ReloadCapability` + `AmmoComponent` + 准星/框/射击 UStack）与 `Laser/`（激光，`ShootingCapability` + `CoolDownCapability` + `OverheatComponent`）。共享 `GunComponent` + `GunResponseComponent`。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| 依赖 | Core 动作队列 | Action Queue | `CoastBossChangeShapeCapability : UHazeActionQueueCapability`，攻击/变形排队执行 |
| 依赖 | 2D 平面 | 移动约束 | 玩家 `MoveTo2DPlanePlayerCapability` 把无人机锁在 `ACoastBoss2DPlane` |
| → | 玩家身份 | 双色子弹 | `AeronauticComponent` 持 `MioBulletClass` / `ZoeBulletClass` 分色 |
| → | 护盾/无敌 | 伤害门控 | `AeronauticShieldCapability` 切 `InvulnerableShield` 可见性 + `ECoastBossPlayerDroneShield` |
| 衔接 | Coast/Waterski | 进入 Boss | 由滑水段 `CoastBossPillarTarget` 过渡进入 |
| 衔接 | Coast/WingSuit | 翼装 Boss | `WingsuitBoss`(Helios) 在翼装段作为追击 Boss |
| → | 玩家 | 肩炮火力 | `ShoulderTurret` 挂玩家肩上，实弹/激光双炮参与火力段 |

## 关键文件

- `LevelSpecific/Coast/Boss/Boss/CoastBoss.as` — 主 Boss 实体
- `LevelSpecific/Coast/Boss/Boss/Capabilities/Attacks/CoastBossChangeShapeCapability.as` — 形态变形（动作队列）
- `LevelSpecific/Coast/Boss/Boss/Capabilities/Attacks/BigGunDrone/` 与 `ManyDrones/` — 两形态攻击集
- `LevelSpecific/Coast/Boss/Player/CoastBossAeronauticComponent.as` — 玩家射击无人机组件
- `LevelSpecific/Coast/Boss/Plane/CoastBoss2DPlane.as` — 2D 弹幕约束平面
- `LevelSpecific/Coast/Boss/WingsuitBoss/WingsuitBoss.as` — 翼装 Boss（Helios）
- `LevelSpecific/Coast/ShoulderTurret/CoastShoulderTurret.as` — 肩炮实体（实弹 + 激光双炮）

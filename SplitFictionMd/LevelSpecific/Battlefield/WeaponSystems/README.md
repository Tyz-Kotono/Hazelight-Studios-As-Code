# LevelSpecific / Battlefield / WeaponSystems（武器系统合集）

> 路径：`LevelSpecific/Battlefield/{ProjectileSystems, MissileSystem, Turrets, LaserSystems, BaseSystems}/`（合并 29 文件）
>
> 本篇合并 Battlefield 的五个武器相关目录：抛射物、导弹、炮塔、激光，以及它们共享的基础系统 BaseSystems。

## 职责

在玩家高速骑悬浮板穿越战区时，制造威胁与视觉压迫的四类火力：**抛射物**（机甲/炮塔的子弹）、**导弹**（追踪/齐射）、**炮塔**（坦克炮塔瞄准开火）、**激光**（定时锁定射线）。四类共享 `BaseSystems/` 的攻击基础组件，普遍采用**对象池化**与**沿样条飞行**，以在高帧率密集火力下保持性能与可编排性。

## 内部架构

### 共享基础 `BaseSystems/`（5）

- `UBattlefieldAttackComponent : USceneComponent`——所有攻击的空组件基（挂点/标记语义）。
- `ABattlefieldAttackFollowSpline`——沿样条运动的攻击基类（`bMakingRun` / `Direction`）。
- `UBattlefieldProjectileFollowSplineComponent` / `UBattlefieldLaserFollowSplineComponent`——让抛射物/激光沿样条按 `FireRate` 节奏生成/推进。
- `UBattlefieldBobbingComponent`——物件上下浮动表现。

### 抛射物 `ProjectileSystems/`（7）

- `ABattlefieldProjectile : AHazeActor`——子弹实体，持有 `FBattlefieldProjectileParams`（LifeTime / Speed / bShouldTrace / Type = `EBattlefieldProjectileType`）与 `OwningManager`。
- `ABattlefieldProjectilePoolManager`——**对象池**：`BeginPlay` 预生成 `ObjectCount`（默认 50）个并 `AddActorDisable` 入池，`ActivateProjectile` 复用。
- `UBattlefieldProjectileComponent`——挂在发射者上的发射逻辑（`FireRate` / `ManualSpawnProjectile`）。
- `ABattlefieldProjectileActor` / `ResponseComponent` / `EffectHandler` / `VFXBattlefieldMechShooter`（机甲射手表现）。

### 导弹 `MissileSystem/`（7）

结构与抛射物平行但更重：`BattlefieldMissile` / `MissileActor` / `MissileComponent` / `MissileResponseComponent` / `MissileEffectHandler` + `BattlefieldMissileBatchActor`（齐射批量）+ `BattlefieldMissilePoolManager`（池化）。

### 炮塔 `Turrets/`（5）

- `ABattlefieldTurret : AHazeActor`——最小炮塔：`MeshRoot` + `UBattlefieldProjectileComponent`（复用抛射物发射）。
- `ABattlefieldTankTurret` + `TargetingCapability`（瞄准）+ `FireCapability`（开火）+ `EffectHandler`——完整坦克炮塔：能力驱动瞄准与开火。

### 激光 `LaserSystems/`（5）

- `ABattlefieldLaserActor : AHazeActor`——激光发射器，`ActivateSetTargetFire(KillTarget, FireTime)` → `LaserComp.SetLaserTargetWithTimer`（先锁定、计时后开火，给玩家躲避窗口）。
- `UBattlefieldLaserComponent`（射线逻辑）+ `BattlefieldLaserImpactActor`（命中点）+ `LaserResponseComponent`（受击响应）+ `EffectHandler`。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| → | BaseSystems | 共享基类 | 炮塔/抛射物/激光沿样条飞行走 `UBattlefieldProjectileFollowSplineComponent` → `ProjectileComp.ManualSpawnProjectile(FollowSplineAttack.Direction)` |
| → | 对象池 | 池化生成 | `ABattlefieldProjectilePoolManager.BeginPlay` 预建 50 个 + `ActivateProjectile` 复用；导弹同构 `MissilePoolManager` |
| → | 炮塔复用抛射物 | 组合 | `ABattlefieldTurret` 直接内含 `UBattlefieldProjectileComponent` |
| ← | 悬浮板玩家 | 施加威胁 | 命中玩家触发受击/死亡（悬浮板侧 Trick UI 的 `Death`）；`ResponseComponent` 处理命中判定 |
| ← | 场景剧本 | 被编排 | `Scenarios/`（如 `AlienCruiser` / 终局主炮）生成并驱动这些武器的开火时序 |
| → | 表现 | EffectHandler | 各系统均带 `*EffectHandler` 解耦 VFX/SFX |

## 关键文件

- `BaseSystems/BattlefieldAttackFollowSpline.as` — 沿样条攻击基类（四类武器共享）
- `BaseSystems/BattlefieldProjectileFollowSplineComponent.as` — 按 FireRate 沿样条生成抛射物
- `ProjectileSystems/BattlefieldProjectile.as` — 抛射物实体 + 参数结构
- `ProjectileSystems/BattlefieldProjectilePoolManager.as` — 抛射物对象池
- `MissileSystem/BattlefieldMissileBatchActor.as` — 导弹齐射批量
- `Turrets/BattlefieldTankTurret.as` + `BattlefieldTankTurretTargetingCapability.as` — 坦克炮塔瞄准/开火
- `LaserSystems/BattlefieldLaserActor.as` — 定时锁定激光（给玩家躲避窗口）

# LevelSpecific / Island / Weapons（其它武器：夺枪 / 破盾 / 双节棍）

> 路径：合并 `LevelSpecific/Island/ScifiCopsGun/`（32）+ `ScifiShieldBusterWeapon/`（18）+ `IslandNunchuck/`（23），共 73 文件。
>
> 红蓝双色主武器见 [RedBlueWeapons/](../RedBlueWeapons/)。本篇是三套辅助武器：夺取式警枪、破盾武器、双节棍近战。三者共享同一套"玩家管理组件 + 可锁定/可伤害/命中响应世界组件"的武器范式。

## 职责

- **ScifiCopsGun 夺枪**：从科幻警卫手中夺取的浮空枪。可手持射击、抛出后自主飞行索敌（贴墙 / 追目标 / 回到玩家）、召回；含炮塔形态与俯视（TopDown）段。
- **ScifiShieldBusterWeapon 破盾**：专破敌方能量护盾/力场的武器。射弹可切割能量墙 `EnergyWall`、部署力场 `Field` / 重力场 `GravityField`。
- **IslandNunchuck 双节棍**：科幻近战系统，基于数据资产驱动的招式（Move）连招——默认连招、无有效目标变体、带收尾后空翻变体。

## 内部架构

三套均为同一范式：`Player/` 侧一个 `Manager*Component`（`UActorComponent`）持有武器类并生成/管理，一组 `*Capability`（`UHazePlayerCapability`）驱动行为；`World/` 侧三件套 `*TargetableComponent`（可锁定）+ `*DamageableComponent`（可受伤）+ `*ImpactResponseComponent`（命中响应）。武器实体均 `AHazeActor`。

### ScifiCopsGun（32）

- **实体** `Core/`：`AScifiCopsGun`（枪）+ `AScifiCopsGunBullet`（同文件）、`CopsGunTurret`（炮塔）、`Crosshair` / `HeatWidget` UI、`Damage` / `Data` / `Settings` / `EventHandler`。
- **枪自主行为** `Guns/`：抛出后飞行——`FlyingShoot` / `HandShoot` / `MoveToTarget` / `MoveToWall` / `MoveToPlayer` / `MoveToNothingAndBack`。
- **玩家管理** `Manager/`：`UScifiPlayerCopsGunManagerComponent`（持 `GunClass` / `TurretClass` / `BulletClass`）+ Capability。玩家能力 `Player/`：Equip / HandShoot / FreeFlyShoot / TurretShoot / PickTarget / Recall（召回）/ Throw（抛出，`AimDownSightInstigators` + `ClearCameraSettingsByInstigator`）/ Heat（过热）/ PassiveCrosshair / Tutorial；俯视段 `TopDown/IslandTopDownComponent`（`bIsInTopDownMode`）。
- **世界** `World/`：Targetable / Damageable / ImpactResponse + `InteractionMechanism`（夺枪交互）。

### ScifiShieldBusterWeapon（18）

- **实体** `Player/`：`AScifiPlayerShieldBusterWeapon` + `AScifiPlayerShieldBusterWeaponProjectile`（同文件）+ `Widget` + `EventHandler`。管理 `ManagerComponent`（持 weapon/projectile 类）+ `ManagerCapability`；玩家能力 Equip / Shoot。
- **目标形态**：`Core/GenericTargetCapability` + `Data`；`EnergyWall/`（能量墙 + 切割能力 `EnergyWallCutter` + Visualizer，`EnergyWallTargetableComponent : ScifiShieldBusterTargetableComponent`）；`Field/`（力场 + Capability）；`GravityField/`（重力场 Actor + Component）。
- **世界** `World/`：`ShieldBusterTargetableComponent` + `ImpactResponseComponent`。

### IslandNunchuck（23）

- **实体** `Player/`：`AIslandNunchuck` + `UIslandNunchuckMeshComponent` + `Visualizer` + `EffectHandler`。玩家组件 `UPlayerIslandNunchuckUserComponent`（"科幻近战系统基类"，持 `WeaponClass`）+ `ManagerCapability`。
- **招式** `Player/Moves/`：数据资产 `UIslandNunchuckMoveAssetBase / UIslandNunchuckMoveAsset` 驱动 `IslandNunchuckMove` + 变体 `_DefaultCombo` / `_NoValidTarget` / `_WithEndingBackflip`。`Notifies/`：动画通知触发允许移动 / 特效 / 命中标记。`Player/OLD/` 是旧版连招实现（保留）。
- **核心/世界** `Core/`：Damage / Data；`World/`：Targetable / Damageable / ImpactResponse。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| 依赖 | Core 武器范式 | 组件三件套 | 三套均用 `*TargetableComponent` + `*DamageableComponent` + `*ImpactResponseComponent`（`World/` 目录一致） |
| 依赖 | Core 相机/瞄准 | Instigator 设置 | `CopsGunThrowCapability` 用 `AimDownSightInstigators` + `ClearCameraSettingsByInstigator` |
| 生成 | 武器实体 | Manager 组件 | 各 `Manager*Component` 持有 `TSubclassOf<AXxxWeapon>` 并生成 |
| → | 敌方护盾/力场 | 破盾 | `ScifiShieldBuster` 切割 `EnergyWall`、部署力场，专克 Island 力场机关 |
| → | 科幻警卫 AI | 夺枪交互 | `ScifiCopsGunInteractionMechanism` 从敌人处夺枪 |
| 依赖 | Core 动画通知 | AnimNotify | Nunchuck 招式由 `Notifies/` 动画通知驱动命中/移动时序 |

## 关键文件

- `LevelSpecific/Island/ScifiCopsGun/Manager/ScifiPlayerCopsGunManagerComponent.as` — 夺枪玩家管理组件
- `LevelSpecific/Island/ScifiCopsGun/Guns/` — 抛出后自主飞行索敌能力
- `LevelSpecific/Island/ScifiCopsGun/Player/ScifiPlayerCopsGunThrowCapability.as` / `RecallCapability.as` — 抛出/召回
- `LevelSpecific/Island/ScifiShieldBusterWeapon/Player/ScifiPlayerShieldBusterManagerComponent.as` — 破盾管理组件
- `LevelSpecific/Island/ScifiShieldBusterWeapon/EnergyWall/ScifiShieldBusterEnergyWall.as` — 能量墙 + 切割
- `LevelSpecific/Island/IslandNunchuck/Player/PlayerIslandNunchuckUserComponent.as` — 近战系统基类组件
- `LevelSpecific/Island/IslandNunchuck/Player/Moves/IslandNunchuckMove.as` — 招式数据资产驱动

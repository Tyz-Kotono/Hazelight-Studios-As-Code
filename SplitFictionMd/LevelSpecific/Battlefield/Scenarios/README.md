# LevelSpecific / Battlefield / Scenarios（场景剧本）

> 路径：`LevelSpecific/Battlefield/Scenarios/`（50 文件）

## 职责

Battlefield 沿途一连串**一次性脚本化大场面**的编排。玩家骑悬浮板前进时，这些剧本在特定路段被触发，制造电影化奇观与阶段性挑战：外星巡洋舰俯冲轰炸、慢动作抓钩躲残骸、飞船残骸段、坦克巨脚踩踏、冰面激光、环形轨道，直到**终局**——玩家抓上战列巡洋舰主炮炮弹、给主炮充能、反轰敌舰。每个剧本是"触发体积/管理器 + 若干专属 actor + 能力/EffectHandler"的组合，负责在正确时机把武器系统、悬浮板子系统、相机编排到一起。

## 内部架构

### 终局 `Finale/`（最大，约 20 文件）

整关高潮，围绕战列巡洋舰主炮：

- `BattlefieldBattleCruiser` / `BattleCruiserCannon` / `BattleCruiserShell`——敌舰、主炮、炮弹实体。
- **充能**：`BattleCruiserCannonPowerupManager` + `BattleCruiserCannonPoweringRing` / `ChargeUpPiece` + `ChargeUpRings/BattleCruiserChargeUpRing`——玩家穿过充能环给主炮蓄力。
- **抓炮弹**：`BattleCruiserGrabOnShell` + `GrabOnShellCapability` + `EffectHandler`——玩家跳上并抓住飞行中的炮弹。
- **敌方火力**：`BattlefieldAlienGunner` + `AlienGunnerFireCapability`、`BattlefieldAlienIncinerator`。
- `Capabilities/`：`BattleCruiserEnterSideViewCapability`（切横版侧视）、`BattleCruiserFireShellCapability`（开炮）。

### 环形轨道段 `Loop/`（约 16 文件）

- **外星巡洋舰** `Loop/AlienCruiser/`：`AlienCruiser` 敌舰 + 一整套 `Capabilities/`（Idle / MoveDown / MoveBack / SpinUp / Shooting / SpinDown / Stop / Compound 复合）驱动其攻击编排；`AlienCruiserMissile` + `ResponseComponent` + `MissileDestructionPlatform`（玩家可摧毁的平台）。
- **慢动作抓钩** `Loop/GrindSlowMo/`：`SlowMoGrappleManager` + `Check`/`Active` 能力——进入子弹时间用抓钩躲避（复用悬浮板 Grapple）。
- `Loop/SlowMoDebris/`：慢动作飞散残骸。
- `BattlefieldLoopSequenceManager`——读磨轨样条 `OnPlayerStoppedGrinding` 触发整段；`BattlefieldLoopSplineCameraVolume` 相机。

### 其它剧本段

- `Artillery/BattlefieldSearchLights`——探照灯搜索。
- `Cannons/BattlefieldAutoCannon`（+ EventHandler）——自动炮。
- `IceCorkscrewLoop/CorkScrewCameraManager`——冰面螺旋环相机。
- `LaserIce/BattlefieldLaserIceManager`（+ EventHandler）——冰面激光段（驱动 LaserSystems）。
- `ShipRun/BattlefieldShipPartExplosion`——飞船部件爆炸。
- `ShipWreckage/`：`ShipWreckageCannons` + `ShipWreckageShell`——残骸炮击段。
- `TankfootSection/BattlefieldFootCrushedTank`（+ EffectHandler）——坦克巨脚踩踏段。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| ← | 悬浮板磨轨 | 触发 | `BattlefieldLoopSequenceManager.BeginPlay` 取 `UBattlefieldHoverboardGrindSplineComponent` 并绑 `OnPlayerStoppedGrinding` 触发剧本 |
| → | 悬浮板抓钩 | 复用 | `GrindSlowMo` 段复用悬浮板 Grapple 做慢动作躲避 |
| → | 武器系统 | 生成/编排 | `AlienCruiser`/`ShipWreckage`/`Cannons`/`LaserIce` 驱动 `MissileSystem`/`ProjectileSystems`/`LaserSystems` 开火 |
| → | 相机系统 | 编排 | `LoopSplineCameraVolume` / `CorkScrewCameraManager` / `EnterSideViewCapability` 切镜头与侧视 |
| → | 玩家能力 | 挂载/横版 | `GrabOnShellCapability` / `FireShellCapability` 挂到玩家；终局切横版侧视 |
| → | 时间膨胀 | 慢动作 | `SlowMoGrapple` / `SlowMoDebris` 走子弹时间 |
| → | 表现 | EventHandler/EffectHandler | 各段带 `*EventHandler` / `*EffectHandler` 解耦 VFX/SFX |

## 关键文件

- `Scenarios/Loop/BattlefieldLoopSequenceManager.as` — 环形段总控：绑磨轨样条事件触发剧本
- `Scenarios/Loop/AlienCruiser/AlienCruiser.as` + `Capabilities/AlienCruiserCompoundCapability.as` — 外星巡洋舰及其攻击编排
- `Scenarios/Loop/GrindSlowMo/BattlefieldSlowMoGrappleManager.as` — 慢动作抓钩躲避
- `Scenarios/Finale/BattleCruiserCannonPowerupManager.as` — 终局主炮充能
- `Scenarios/Finale/BattleCruiserGrabOnShellCapability.as` — 抓上飞行炮弹
- `Scenarios/Finale/Capabilities/BattleCruiserEnterSideViewCapability.as` — 终局切横版侧视
- `Scenarios/TankfootSection/BattlefieldFootCrushedTank.as` — 坦克巨脚踩踏段
- `Scenarios/LaserIce/BattlefieldLaserIceManager.as` — 冰面激光段（驱动 LaserSystems）

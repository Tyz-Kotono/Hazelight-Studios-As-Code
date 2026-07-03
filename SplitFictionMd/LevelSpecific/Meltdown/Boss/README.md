# LevelSpecific / Meltdown / Boss（三阶段终极 Boss 战）

> 路径：`BossBattle/`(64) + `BossPhaseOne/`(10) + `BossPhaseTwo/`(27) + `BossPhaseThree/`(106)。合计 207 文件。

## 职责

Meltdown 是终章，Boss 战是全游戏的最终对决，分三个阶段递进升级。三阶段共享同一个 Boss 基类 `AMeltdownBoss`，但各自换一套战斗模态：一阶段是网格竞技场近战、二阶段是穿梭多个"世界"的弹幕演出、三阶段是传送门召唤 + 飞行/跳伞的多模态混战。玩家用[故障武器](../GlitchShooting/)对 Boss 的弱点输出。`BossBattle/` 是共用的攻击/机关道具库（供各阶段引用），三个 `BossPhaseN/` 是各阶段专属逻辑。

## 内部架构

### 共享 Boss 基类

`AMeltdownBoss : AHazeCharacter`（定义于 `GlitchShooting/MeltdownBoss.as`），三阶段 Boss 全部继承它。它装配：
- `UMeltdownBossHealthComponent`（血量/阶段阈值 `HealthThreshold` + `ThresholdReached` 事件）
- `UHazeActionQueueComponent`（攻击序列排队）
- `UHazeCapabilityComponent`（能力容器——每种攻击是一个 Capability）
- `UMeltdownGlitchShootingResponseComponent`（**关键**：让 Boss 能响应玩家的故障武器命中，`OnGlitchHit`）
- `AMeltdownBossWeakPoint : AHazeActor`——可被射击的弱点。

Boss 的攻击组织为"能力清单"：每种攻击 = 一个 `UHazeCapability`（AI 能力，非玩家能力），由 `CapabilityComp.DefaultCapabilityClasses` 装配，`UHazeActionQueueComponent` 排队触发。这与 Gameplay 的 AI 同构框架一致（组件容器 + 能力清单）。

### 一阶段 BossPhaseOne（10）——网格竞技场近战

`AMeltdownBossPhaseOne : AMeltdownBoss`。最紧凑的一阶段：一小组攻击能力驱动一个在方块网格竞技场里活动的 Boss——`MeltdownPhaseOneIdleCapability`、`...SlamCapability`（砸地）、`...SeekSlamCapability`（追踪砸地）、`...LineAttackCapability`（直线攻击）、`...CylinderAttackCapability`、`...ShockwaveArrowAttackCapability`（冲击波箭）、`...DropAttackCapability`。特效走 `UHazeEffectEventHandler`。`MeltdownBossPhaseOneVelocityAfterKnockbackActor`、`MeltdownRaderSplineMove` 处理击退与轨道位移。`BossBattle/` 里的 `MeltdownBossCubeGrid` / `MeltdownBossBlockManager` / `MeltdownBossBlockMover` 等提供竞技场网格。特点：**能力驱动的近战 + 网格竞技场**。

### 二阶段 BossPhaseTwo（27）——多世界弹幕

`AMeltdownBossPhaseTwo : AMeltdownBoss`，由 `AMeltdownBossPhaseTwoManager : AHazeActor` 编排。核心是穿梭三个"世界"打弹幕：`enum EMeltdownBossPhaseTwoWorld { LavaRiver, Vortex, Space }`，Manager 持 `TArray<UHazePlayerVariantAsset> PlayerVariants`（每世界一套玩家外观）+ `AMeltdownBossPhaseTwoLevelAnchor` 做各世界锚点定位，`Position_Convert(Src,Tgt)` 在世界间换算坐标。攻击是成对的 Capability + EffectHandler 模块（火剑、火冲击波、火蠕虫、大锤、龙卷风、陨石池、飞船 + 导弹、滚方块…，见 `BossBattle/Phase 2/`）+ 投射物 Actor。特点：**空/多世界切换的弹幕演出**。

### 三阶段 BossPhaseThree（106）——传送门召唤 + 多模态混战

`AMeltdownPhaseThreeBoss : AMeltdownBoss`（+ `UMeltdownPhaseThreeRaderAnimationCapability`、`UMeltdownShakeComponent`）。终章高潮，把前两关所有花样叠加，按子系统分文件夹，各自成套：
- **Flying/**——完整飞行小游戏：`AMeltdownBossFlyingPhaseManager` + `UMeltdownBossFlyingSettings : UHazeComposableSettings` + `MeltdownBossFlyingCapability` / `...Component` + 机翼/狙击/集束/AOE 投射物。
- **Skydive/**——跳伞段：`UMeltdownSkydiveCapability : UHazePlayerCapability` + `UMeltdownSkydiveComponent` + `UMeltdownSkydiveSettings` + 桶滚/受击反应能力 + 样条/追踪投射物。
- **Falling/**——坠落段：`MeltdownFallingManager` + 坠落障碍/激光/漩涡。
- **传送门召唤的小 Boss/杂兵**：`DecimatorPortal`、`IceKingPortal`、`SpinnerPortal`、`OgreShaker`、`Punchotrons`、`ShootingStoneBeast`（自带 idle/track/fire/spawn/select 能力状态机）、`BallSmash`、`HandGrabAttack`、`RainAttack`、`TrackingAimAttack`、`DropProjectileAttack`。
- `MeltdownBossPhaseThreeArena` + `...GenericPortal` 框定整场遭遇。

特点：**Boss 在飞行/跳伞/坠落与传送门召唤敌之间循环切换的多模态战**。

### 共用道具库 BossBattle（64）

不含独立 Boss Actor，是三阶段共享的攻击/机关/演出库：网格与方块（`MeltdownBossCubeGrid`、`MeltdownMovingBossCubeGrid`、`MeltdownBossBlockManager/Mover`、`MeltdownBossPhaseOneClosingCube`）、重力机关（`MeltdownGravityShift`、`MeltdownBossGrapple(Perch)GravityShifter`）、激光/发射器（`MeltdownBossLaserSpawner`、`MeltdownBossLauncer(CustomLaunch)`、`MeltdownBossTurret(Wall)`）、波浪/蠕虫/冲击波等一阶段攻击原型，以及基类攻击变换 Actor（`MeltdownBossBaseAttack{Location,Rotation,Scale,InverseScale}Actor`）。`Phase 2/` 子文件夹放二阶段专属道具。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| ← | 故障武器 | 受击响应 | `AMeltdownBoss` 装 `UMeltdownGlitchShootingResponseComponent`，`OnGlitchHit` 绑定 → 玩家用故障枪打弱点造成伤害 |
| → | Gameplay AI 框架 | 组件+能力清单 | `AMeltdownBoss : AHazeCharacter` + `UHazeCapabilityComponent` + `UHazeActionQueueComponent`，攻击=`UHazeCapability` |
| → | Core Settings 系统 | 参数分层 | `UMeltdownBossFlyingSettings` / `UMeltdownSkydiveSettings : UHazeComposableSettings` |
| → | 打破分屏机制群 | 世界换算复用 | 二阶段 `Position_Convert` + `PlayerVariants` 与 SplitMechanics 同源；三阶段跳伞/飞行段复用镜头编排 |
| → | Gameplay 玩家能力 | 段内注入 | 三阶段 `UMeltdownSkydiveCapability` 等 `UHazePlayerCapability` 在段落激活时加到玩家身上 |
| → | Core 血量/阈值 | 阶段推进 | `UMeltdownBossHealthComponent.HealthThreshold` + `ThresholdReached` 事件切换阶段 |

## 关键文件

- `GlitchShooting/MeltdownBoss.as` — `AMeltdownBoss : AHazeCharacter` 共享基类 + `AMeltdownBossWeakPoint`
- `BossPhaseOne/MeltdownBossPhaseOne.as` — 一阶段 Boss + 其攻击能力清单
- `BossPhaseTwo/MeltdownBossPhaseTwoManager.as` — 二阶段多世界编排（`EMeltdownBossPhaseTwoWorld` 三世界）
- `BossPhaseThree/MeltdownPhaseThreeBoss.as` — 三阶段 Boss
- `BossPhaseThree/Flying/` + `BossPhaseThree/Skydive/` — 三阶段飞行/跳伞两大子系统（各含 Capability+Component+Settings）
- `GlitchShooting/MeltdownBossHealthComponent.as` — Boss 血量与阶段阈值
- `BossBattle/` — 三阶段共享的攻击/机关/网格道具库

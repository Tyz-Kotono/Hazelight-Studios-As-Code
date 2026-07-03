# LevelSpecific / Summit / 风暴攻城战（StormSiege）

> 路径：`LevelSpecific/Summit/StormSiege/`（212 文件）
>
> Summit 后半程的大型场景剧本：一座被风暴（Storm）笼罩的城市攻防战，从守城 → 巨蛇（Serpent）遭遇 → 龙背追逐终局。这是把前面所有龙能力"编排进一场连续演出"的关卡段，本身以 actor/管理器/剧本触发为主，能力多为一次性桥段能力。

承接 [Summit 总览](../README.md) 与 [龙骑乘体系](../Dragons/README.md)。能力四件套/标签范式见 [Gameplay 设计哲学](../../../Gameplay/DesignPhilosophy.md)，不重复。

---

## 职责

风暴攻城战是一段**线性演出关卡**，把攻城单位、风暴环境、巨蛇 Boss、龙背追逐串成连续流程。它不发明新的移动范式，而是复用成年龙飞行（`Dragons/AdultDragons/`）与俯视/滑翔段，外加大量场景 actor（攻城塔、落石、风暴雷电、屏障）和管理器（Manager）驱动节奏。

---

## 内部架构

按剧情顺序分为四大阶段（对应四个子目录族）：

### 一、Intro（守城/攻城单位）

城市攻防的开场。`Intro/SiegeUnits/` 是敌方攻城军团：

- **GemLightningMaster / GemSpearMaster** — 宝石系操作员（承接 Summit `CombatPrototypes` 的宝石变形语义）。
- **MetalFortification / MetalGemGuardians / MetalGemSiegeTowers** — 金属工事、守卫、攻城塔；攻城塔含 `SiegeMagicBeam/` 魔法光束攻击与专属 `Capabilities/`。
- **SplineMovers/** — 沿样条推进的攻城单位移动。
- 环境交互：`Intro/Interactables/`（`BirdFlocks/` 鸟群、`Fruit/` 果实）、`CaveIn/` 塌方、`MagicBarrier/` 魔法屏障、`Obstacles/`、`BlockingStructures/`。

### 二、StormLoop（风暴环境 + 龙登场）

- `StormLoop/StormDragonIntro/`（含 `Capabilities/`）— 风暴中龙的登场桥段。
- `StormLoop/Debris/` — 风暴碎片/飞行障碍。
- 环境 actor：`StormSiegeLightning`（`AStormSiegeLightning : AHazeActor` + `UStormSiegeLightningEffectsHandler` 特效处理器）、`StormSiegeVineGateway`（藤蔓门）、`SeaCloud/` 云海、`StormActors/`。

### 三、Serpent（巨蛇遭遇战，本关最大子系统）

风暴之眼（EyeOfTheStorm）中的巨蛇 Boss，是一段多形态追逐/攀爬战：

- **`SerpentHead.as`** — 核心数据：状态机枚举 `ESerpentMovementState` / `ESerpentAttackMovementState`，弱点数据 `FSerpentHeadWeakpointData`、重生点 `FSerpentHeadRespawnPointData`、受击旋转反馈 `FSerpentHurtRotationResponse`。
- **`ASerpentArmor : AHazeActor`** — 可破的蛇甲（对应弱点破甲）。
- **攻击 actor**：`SerpentHomingMissile`（追踪导弹 actor + `Capability` + `EventHandler` + `Spawner` 全套）、`SerpentLightningStorm` / `SerpentLightningStrike`（雷暴/落雷）、`CrystalBreath/`（晶体吐息）、`ObstacleAttacks/`、`SerpentRunAttacks/`。
- **玩家应对段**：`Climb/`（攀爬蛇身）、`RollingSerpent/` `SpikeRoll/`（滚动躲避）、`VerticalSerpent/`、`FallingFloor/`、`Platforms/`、`SplineMovement/` `SerpentRunCamera/`（在轨跑酷 + 相机）。
- **调度**：`SerpentMovementSettings`（`UHazeComposableSettings`）、`SerpentSettingsVolume`（`AActorTrigger`，进区切设定）、`SerpentEventActivator`（剧本事件触发）、`CirclingEncounter/`（绕圈遭遇）、`DragonHelp/`（龙协助）。

### 四、EyeOfTheStorm + Finale（龙背追逐终局）

- `EyeOfTheStorm/StormDragonSummitPeak/`（含 `Capabilities/`）、`StormKnight/`、`ArmorPieces/`、`Barriers/`、`MagicBarrier`（`AMagicBarrier` + `USummitMagicBarrierEventHandler`）。
- **`Finale/DragonRunBoss/`** — 龙背 Boss `AStormRideDragon : AHazeCharacter`，配 `Fin`/`Spike`/`Colliders`（`UStormRideDragonAttackBoxComponent : UBoxComponent`）、样条移动能力 `UStormRideDragonSplineMovementCapability`、网格旋转 `UStormRideDragonMeshRotationCapability`；弱点 `DragonRunWeakpoints/`。
- **`Finale/DragonRunPlayerDragons/`** — 玩家龙 `ADragonRunPlayerDragon : AHazeCharacter`，派生 `ADragonRunAcidDragon`（Mio 酸龙）/ `ADragonRunTailDragon`（Zoe 尾龙），复用成年龙的酸弹/尾击语义。
- **`Finale/DragonRun/`** — 途中障碍 `DragonRunGemObstacle` / `DragonRunMetalObstacle`。
- **`Finale/StormFall/` + `StormFallDragon/`** — 坠落桥段：`AStormFallObject` 及派生（`GrappleObject` 抓钩物、`RockSpawner` 落石、`BobbingObject` 漂浮物、`CustomMoveObject`），由 `StormFallObjectSpawnerManager` / `StormFallSetPlayerVelocityManager` 统一调度坠落速度与生成。
- **`Finale/Managers/`** — `StoneBeastCameraManager`(`PartOne`) 相机总控，衔接 StoneBoss 终局。
- `Chase/`、`PlayerDragonSignal/` — 追逐段与玩家龙信号。

---

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 依赖 | `AHazeCharacter` | Core | `AStormRideDragon`、`ADragonRunPlayerDragon` 均继承 `AHazeCharacter` |
| 依赖 | `AHazeActor` / `AActorTrigger` | Core | 攻城单位、风暴 actor、`SerpentSettingsVolume` 触发体等 |
| 依赖 | `UHazeCapability` / `UHazeComposableSettings` | Gameplay | 桥段能力与 `SerpentMovementSettings` 走能力四件套 |
| 复用 | `AdultDragon` 酸弹/尾击语义 | Dragons | `DragonRunAcidDragon`/`DragonRunTailDragon` 承接成年龙双人双龙约定 |
| 复用 | `UHazeSplineComponent` | Gameplay | `StormRideDragonSplineMovementCapability`、Serpent `SplineMovement/` 在轨演出 |
| 承接 | 宝石/金属变形 | Summit CombatPrototypes | `SiegeUnits` 的 Gem*/Metal* 单位延续夜后原型的材质变形语义 |
| 衔接 | `StoneBeastCameraManager` | Summit StoneBoss | Finale 相机管理器接续石兽终局 |
| 特效 | `UHazeEffectEventHandler` | Core | Lightning/StormFall/Serpent 系列 EffectHandler 处理命中/生成表现 |

---

## 关键文件

- 攻城：`StormSiege/Intro/SiegeUnits/MetalGemSiegeTowers/`、`StormSiege/Intro/SiegeUnits/GemLightningMaster/`
- 风暴环境：`StormSiege/StormSiegeLightning.as`、`StormSiege/StormLoop/StormDragonIntro/`
- 巨蛇：`StormSiege/Serpent/SerpentHead.as`、`StormSiege/Serpent/SerpentArmor.as`、`StormSiege/Serpent/SerpentHomingMissile.as`、`StormSiege/Serpent/Climb/`
- 龙背终局：`StormSiege/Finale/DragonRunBoss/StormRideDragon.as`、`StormSiege/Finale/DragonRunPlayerDragons/DragonRunPlayerDragon.as`、`StormSiege/Finale/StormFall/StormFallObject.as`、`StormSiege/Finale/Managers/StoneBeastCameraManager.as`

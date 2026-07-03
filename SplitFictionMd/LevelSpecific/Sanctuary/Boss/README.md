# LevelSpecific / Sanctuary / Boss（九头蛇 Boss + 蜈蚣穿越）

> 路径：`LevelSpecific/Sanctuary/Boss/`（401 文件）+ `LevelSpecific/Sanctuary/Centipede/`（115 文件）
>
> 本篇合并 Sanctuary 全关高潮：多阶段九头蛇（Hydra）Boss 战，以及作为其中一段的蜈蚣（Centipede）巨怪穿越。这是全关最大目录，由多套玩法段落拼成完整 Boss 流程。

## 职责

与巨型九头蛇 Hydra 的多阶段决战。Hydra 有 5 颗头，每颗独立行动（潜入/浮出、追击、撕咬、投射、下雨攻击）。玩家在不同阶段切换玩法：竞技场对战（Arena）、乘弩炮（Ballista）、勋章 2D 平面段（Medallion）、滑行（Slide）、样条奔跑（SplineRun）、跳伞（Skydive），最终进入 Boss 体内（Inside）打心脏收尾。蜈蚣段是穿越巨怪身体的独立玩法（攀爬/撕咬/摆荡/水管/熔岩）。

## 内部架构

### 九头蛇核心 `Boss/Hydra/`

- **实体** `ASanctuaryBossHydraBase : AHazeActor`：持 5 颗头 `TArray<ASanctuaryBossHydraHead> Heads`（`SetNumZeroed(5)`，按 `Head.Identifier` 索引注册）。头 `ASanctuaryBossHydraHead : AHazeActor`。
- **管理与数据** `SanctuaryBossHydraManagerComponent : UActorComponent`（Boss 编排中枢）+ `HydraSettings` / `HydraAnimationData` / `HydraAttackData`（`PointAttackData` / `SweepAttackData` 攻击子类）+ `HydraTelegraphComponent`（攻击预告）+ `HydraPlatformComponent`（头可作平台）+ `HydraProjectile` + `HydraResponseComponent`。

### 阶段玩法段落（拼装 Boss 流程）

| 段落目录 | 核心类 | 玩法 |
|---|---|---|
| `Arena/` | `ASanctuaryBossArenaManager` | 竞技场对战：Hydra 头动作能力（Idle/Projectile/Rain/Wave/Dance 选择）、GhostRain 幽灵雨、平台 `Platforms/`、波浪 `Wave/`、`InfuseEssence/`（注入精华）、`CameraClamp/` |
| `Ballista/` | `ABallistaHydraActorReferences` | 乘弩炮射击 Hydra，`Player/` + `Spline/` |
| `Medallion/` | `UMedallionPlayerComponent` + `ASanctuaryBossMedallion2DPlane` | 勋章段：玩家约束在 2D 平面（`Plane/`），沿样条移动（`SanctuaryBossSplineMovementComponent`）+ `CoopFlyingActions/` + `MedallionRespawn/` |
| `Slide/` | `ASanctuaryBossSlideActor` + `SlideBall` | 滑行段，`PlayerSlideCapability` + 隐藏伙伴 |
| `SplineRun/` | `ASanctuaryBossSplineRun : ASplineActor` | 样条奔跑逃脱：平台 `SplineRunPlatform`、波浪 `LaunchWave`、弩 `CrossBow`、精华交互 `PushEssence`、Hydra 追击 `SpamAttack` |
| `Skydive/` | `ASanctuaryBossSkydiveActor` | 跳伞段 + 攻击体 + 光束 |
| `Inside/` | `ASanctuaryBossHeartClaf` / `HeartBeatManager` | Boss 体内：心脏 + 心跳、酸液 `InsideAcid`、血泡 `InsideBlob`、船 `Boat/`、光鸟护盾 `LightBirdShield/`、巨型伙伴弓 `MegaCompanionBow/`、Zoe 雕像 `ZoeStatue/`、`FinalPhase/` 收尾 |
| `AbilityUpgrade/` | 能力升级 | 阶段间玩家能力升级 |

### 蜈蚣穿越 `Centipede/`（115）

- **实体** `ACentipede : AHazeActor` + `CentipedeSheets`（能力表）+ `CentipedeSplineCollision : ASplineActor`（身体样条碰撞）+ `CentipedeVoActor`（语音）。`CentipedeAllowStretchVolume : AVolume`（允许拉伸区）。
- **玩法能力**（各成子系统）：
  - `Crawl/`：在蜈蚣身上攀爬（`CentipedeCrawlableComponent` + `CrawlConstraint`）。
  - `Bite/`：撕咬交互（`BiteActivation` / `Bite` / `BiteTargeting` / `BiteVisuals` + `BiteComponent` + `BiteResponseComponent`）。
  - `Swing/`：摆荡（Actors + Capabilities + Components）。
  - `WaterHose/`：水管解谜——水口 `WaterOutlet`（喷水/撕咬）+ 水塞 `WaterPlug`（携带/拖拽/塞入）。
  - `Lava/`：熔岩灼烧死亡；`MoleCombat/`：熔岩鼹战斗；`DraggableGate/`：可拖拽闸门；`ConsumableProjectile/`：可吞投射物；`ProjectileTargeting/` / `Movement/`。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| 依赖 | Core 能力四件套 | Capability/Component/Settings/Data | Hydra 各段落均按 `*Capability` + `*Component` + `*Settings` + `*Data` 组织 |
| 组合 | 5 颗头 | 容器数组 | `ASanctuaryBossHydraBase.Heads : TArray<ASanctuaryBossHydraHead>`（`SetNumZeroed(5)`，按 Identifier 注册） |
| 依赖 | 2D 平面约束 | 移动约束 | Medallion 段 `ASanctuaryBossMedallion2DPlane` + `SanctuaryBossSplineMovementComponent` |
| → | Sanctuary/Aviation | 骑乘进攻 | 部分阶段用巨型伙伴 MegaCompanion 飞行进攻（`Inside/MegaCompanionBow/`） |
| → | Sanctuary/LightBird | 光鸟护盾 | `Inside/LightBirdShield/` 复用光鸟道具 |
| 依赖 | Core 样条 | ASplineActor | `SplineRun` 与蜈蚣身体均基于 `ASplineActor` |
| → | 交互系统 | InteractionComponent | `SanctuaryBossPushEssenceInteractionComponent : UInteractionComponent` 精华推动交互 |

## 关键文件

- `LevelSpecific/Sanctuary/Boss/Hydra/SanctuaryBossHydraBase.as` — 九头蛇实体（5 头容器）
- `LevelSpecific/Sanctuary/Boss/Hydra/SanctuaryBossHydraManagerComponent.as` — Boss 编排中枢
- `LevelSpecific/Sanctuary/Boss/Hydra/SanctuaryBossHydraHead.as` — 单头实体
- `LevelSpecific/Sanctuary/Boss/Arena/SanctuaryBossArenaManager.as` — 竞技场阶段管理
- `LevelSpecific/Sanctuary/Boss/Medallion/Plane/SanctuaryBossMedallion2DPlane.as` — 勋章段 2D 约束平面
- `LevelSpecific/Sanctuary/Boss/SplineRun/SanctuaryBossSplineRun.as` — 样条奔跑逃脱段
- `LevelSpecific/Sanctuary/Boss/Inside/SanctuaryBossHeartClaf.as` — Boss 体内心脏收尾段
- `LevelSpecific/Sanctuary/Centipede/Actors/Centipede.as` — 蜈蚣巨怪实体
- `LevelSpecific/Sanctuary/Centipede/WaterHose/CentipedeWaterPlug.as` — 蜈蚣水管解谜

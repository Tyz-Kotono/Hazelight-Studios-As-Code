# LevelSpecific / Summit / 龙骑乘体系（Dragons）

> 合并四个目录：`BabyDragons/`（49）、`ControlledBabyDragons/`（10）、`TeenDragons/`（192）、`AdultDragons/`（156）——共约 407 文件。
>
> Summit 的核心成长线：一条龙随剧情"长大"，玩家的操作从**龙挂在身上辅助移动**（幼龙），到**玩家变身为龙**（少年龙），再到**空战骑乘**（成年龙）。四个年龄各是一套独立的能力组，但共享同一套"双人双龙"约定。

承接 [Summit 总览](../README.md)。能力四件套/标签范式见 [Gameplay 设计哲学](../../../Gameplay/DesignPhilosophy.md)。

---

## 贯穿约定：双人双龙

两名玩家各持一条龙，全程语义固定：

- **Mio → 酸液龙（Acid Dragon）**：远程酸液攻击（喷射/光束/酸弹）。
- **Zoe → 尾巴龙（Tail Dragon）**：近战尾巴攻击。

每阶段的龙 actor 都用 `IsAcidDragon()` / `IsTailDragon()` 按 `Player == Game::Mio / Game::Zoe` 区分（见 `AAdultDragon`、`ATeenDragon`）。各阶段都有成对的 `PlayerAcid<X>DragonComponent` 与 `PlayerTail<X>DragonComponent`。

三个年龄的骑乘挂载模式**根本不同**，这是理解本体系的关键：

| 阶段 | 挂载模式 | 谁在移动 | 代表基类 |
|---|---|---|---|
| 幼龙 Baby | 龙 attach 到玩家身上（配件） | 玩家自己 | `ABabyDragon : AHazeCharacter` |
| 可操控幼龙 | 换玩家 Mesh 为龙 Mesh | 玩家自己 | 无独立 actor，仅换皮 |
| 少年龙 Teen | 龙作为"死"actor 贴在玩家 Spine4 骨 | **玩家 = 龙**，玩家持有全部能力 | `ATeenDragon`（玩家挂能力） |
| 成年龙 Adult | 玩家 MeshOffset attach 到龙 Mesh | 龙 actor 移动，玩家被带飞 | `AAdultDragon : AHazeCharacter` |

---

## 一、幼龙 BabyDragons（49）

龙是**挂在玩家身上的配件**，玩家仍是移动主体，龙提供额外移动动词。

- **生成/挂载**：`UPlayerBabyDragonCapability`（TickGroup=Input）在 `OnActivated` 调 `DragonComp.SpawnBabyDragon()` + `AttachBabyDragon()`；`OnDeactivated` 销毁龙。组件 `UPlayerBabyDragonComponent`（酸/尾各一份 `PlayerAcid/TailBabyDragonComponent`）。
- **`ABabyDragon : AHazeCharacter`**：极简，只持 `Player` 引用、姿态调试、移动音频组件。命名空间 `BabyDragon` 提供 `GetBabyTailDragon()/GetBabyAcidDragon()` 便捷取用。
- **移动动词子模块**：
  - `AirBoost/` — 借上升气流升空（`BabyDragonAirCurrentCapability` 读 `AirCurrent/` 体积）。
  - `TailClimbUsingPoints/` — 沿预设攀爬点（`ABabyDragonTailClimbPoint`）攀爬：进入/悬挂/伸手/攀边/跳离，一套能力状态机。
  - `TailClimbFreeForm/` — 自由瞄准式尾巴攀爬（带教程、相机切换、取消体积）。
  - `Zipline/` — 尾巴滑索：进入/跟随/取消（`ABabyDragonZiplinePoint`）。
- **支线场景**：`NightQueenMonster/`、`FallingRocks/`、`SlidingRock` 等专用互动物。

## 二、可操控幼龙 ControlledBabyDragons（10）

短桥段里**直接操控一条小龙**——不生成新 actor，而是把玩家 Mesh 换成龙 Mesh。

- `UPlayerControlledBabyDragonCapability`：`OnActivated` 用 `Player.Mesh.SetSkeletalMeshAsset(DragonComp.BabyDragonMesh)` 换皮 + 换相机；`OnDeactivated` 还原。
- 一套精简移动能力：`Floor/Air Movement`、`Jump`、`Dash`、`Sprint`（含 Activate/Deactivate 三段）。设定集中在 `ControlledBabyDragonSettings`。

## 三、少年龙 TeenDragons（192）

**玩家变身为龙本体**——龙是"死的"actor 贴在玩家 Spine4，`// The player IS the dragon`（组件注释原文）。玩家 CapabilityComponent 承载全部龙能力，最大最复杂的一阶段。

- **组件 `UPlayerTeenDragonComponent`**：管龙生成（`SpawnDragon`，`MakeNetworked` 联机）、体力（Stamina）、俯视模式（`ActivateTopDownMode` → `ApplyGameplayPerspectiveMode(TopDown)`）、动画状态机枚举 `ETeenDragonAnimationState`、双端联动动画 `RequestLocomotionDragonAndPlayer`（龙和玩家 Mesh 同步请求 locomotion）。
- **骑乘外壳 `UPlayerTeenDragonRidingCapability`**（AfterGameplay）：应用重力/相机/阴影设定，龙 Mesh 加入次表面渲染。
- **动作组（每个都是完整能力子系统）**：
  - `Movement/` — 地面/空中移动、跳、冲刺、Sprint、坡面滑行、Ledge 抓边。
  - `Roll/` — 招牌"滚球"：滚动、跳、撞墙反弹、轨道（Rail）滑行、追踪攻击（HomingAttack）、发射器、自动瞄准，自带 `TeenDragonRollMovementResolver`。
  - `AirGlide/` — 滑翔：气流、加速环（BoostRing）、重力、相机跟随。
  - `AcidSpray/`（Mio）/ `FireBreath/`（火焰）/ `TailAttack/` `TailBomb/`（Zoe）— 攻击能力。
  - `Chase/` — 追逐段（滚动/滑翔 strafe + 重生）。`ConstrainToScreen/` 俯视限屏。
- **移动求解**：`Resolver/TeenDragonMovementResolver` + `TeenDragonMovementData`（自定义 movement resolver，见 Gameplay Move 系统）。

## 四、成年龙 AdultDragons（156）

**真正的空战骑乘**：玩家 MeshOffset attach 到龙 Mesh，龙 actor 是移动主体带着玩家飞。

- **`AAdultDragon : AHazeCharacter`**：自带 `UHazeCapabilityComponent`，`BeginPlay` 建 `AttachComp`（挂到 `PlayerAttachSocket`）；提供进出过场的 attach/detach（`AttachPlayerToDragonAfterIntroSequence`）。
- **骑乘外壳 `UPlayerAdultDragonRidingCapability`**（AfterGameplay）：`Setup` 里 `DragonComp.SpawnDragon()`；`OnActivated` 把玩家 MeshOffset 贴到龙、覆盖 capsule 尺寸、应用相机、加轮廓（Outline）、设飞行模式 `EAdultDragonFlightMode`、阻断"找队友"等地面能力。组件 `UPlayerAdultDragonComponent`。
- **飞行/移动子系统（`Movement/`，本阶段重头）**：
  - `Flying/` `FreeFlying/`（含 RubberBanding 橡皮筋回拉）— 自由飞行。
  - `Strafe/`（含 `SplineFollow/` 样条跟随 + RubberBand、`SelectorCapability` 选样条）— 引导式扫射飞行。
  - `CircleStrafe/` — 绕圈扫射（`CircleStrafeManager` 统一调度攻击跑位）。
  - `ChaseStrafe/`、`AirDash/`、`AirBreak/`（刹车）、`AirDrifting/`、`TurnBack/`、`Boundaries/`（关卡边界转向）、`RespawnSpline/`。
- **攻击子系统**：`AcidBeam/`（酸液光束，Mio）、`AcidProjectile/` `AcidChargeProjectile/`（酸弹/蓄力酸弹）、`SpikeProjectile/`（尖刺弹）、`TailSmash/` `HomingTailSmash/` `SmashMode/`（尾巴猛击/追踪/猛击模式，Zoe）。
- `Tutorial/` 教程 + `Widgets/` 准星血条。相机能力：`PlayerAdultDragonCameraLag/Tilt`。

---

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 依赖 | `AHazeCharacter` | Core | Baby/Adult 龙及石兽龙均继承 `AHazeCharacter`；Teen "变身"仍是玩家角色本体 |
| 依赖 | `UHazeCapabilityComponent` / `UHazePlayerCapability` | Gameplay | 所有龙能力挂玩家或龙的 CapabilityComponent（能力四件套） |
| 依赖 | `UPlayerMovementComponent` / 自定义 Resolver | Gameplay Move | Teen 用 `TeenDragonMovementResolver`，Adult 骑乘里操作 `MoveComp.ActiveConstrainRotationToHorizontalPlane` |
| 借力 | `AirCurrent/` | Summit | Baby `BabyDragonAirCurrentCapability`、Teen `TeenDragonAirCurrentCapability` 读上升气流升空 |
| 触发 | `Acid/`（`UAcidDissolvable`） | Summit | 酸液龙的喷射/光束/酸弹命中通用酸液系统溶解物体 |
| 被复用 | `AAdultDragonAcidProjectile`、Acid 设定 | StormSiege | `DragonRunAcidDragon` 复用成年酸龙的弹药设定驱动龙背 Boss 追逐 |
| 特化 | `AStoneBeastAdulDragonBase : AHazeCharacter` | StoneBoss | 石兽头部是成年龙形态的放大特化 |
| 联机 | `MakeNetworked` / Crumb | Core Network | 龙 actor 生成时 `MakeNetworked(Player)` + `SetActorControlSide` |

---

## 关键文件

- 幼龙：`BabyDragons/BabyDragon.as`、`BabyDragons/PlayerBabyDragonCapability.as`、`BabyDragons/PlayerBabyDragonComponent.as`、`BabyDragons/TailClimbUsingPoints/`、`BabyDragons/Zipline/`
- 可操控幼龙：`ControlledBabyDragons/PlayerControlledBabyDragonCapability.as`、`ControlledBabyDragons/ControlledBabyDragonSettings.as`
- 少年龙：`TeenDragons/PlayerTeenDragonComponent.as`、`TeenDragons/PlayerTeenDragonRidingCapability.as`、`TeenDragons/Roll/`、`TeenDragons/AirGlide/`、`TeenDragons/Resolver/TeenDragonMovementResolver.as`
- 成年龙：`AdultDragons/AdultDragon.as`、`AdultDragons/PlayerAdultDragonRidingCapability.as`、`AdultDragons/PlayerAdultDragonComponent.as`、`AdultDragons/Movement/FreeFlying/`、`AdultDragons/Movement/Strafe/SplineFollow/`、`AdultDragons/AcidBeam/`、`AdultDragons/TailSmash/`
</content>

# LevelSpecific / Summit / 巨人协作（Giants）

> 路径：`LevelSpecific/Summit/Giants/`（77 文件）
>
> 与巨人 NPC 协作的桥段：巨人提供舞台与辅助（拉杆、大钟、跳伞助推），玩家使用两件远程武器——冰弓（IceBow）与风矛（WindJavelin）——远程解谜与战斗。

承接 [Summit 总览](../README.md)。能力四件套/标签范式见 [Gameplay 设计哲学](../../../Gameplay/DesignPhilosophy.md)，不重复。

---

## 职责

Giants 段把"巨人 NPC + 玩家远程武器"组合成协作解谜/战斗。巨人本身是场景 actor（`AGiant` / `ATheGiant`）与互动物（钟、拉杆），玩家武器则是两套标准的"武器 actor + 玩家能力 + 弹药 actor"结构。

---

## 内部架构

### 巨人 NPC 与场景互动

- **`ATheGiant : AHazeSkeletalMeshActor`** — 主巨人（带骨骼动画的大型 NPC）。`AGiant : AHazeActor` 为另一巨人 actor。
- **互动物**：`GiantBell`（大钟）+ `GiantBellHitter`（撞钟器）、`GiantLever : AHazeAnimActor`（拉杆）、`GiantsSkydiveVelocityGiver`（跳伞助推给玩家赋速）、`GPUCollision`（大体量碰撞）。
- `GiantsDevToggles`（调试开关）、`Audio/`（巨人音频）。

### 一、冰弓 IceBow（多箭种远程武器）

一把弓，切换四种箭矢，每种箭是独立的"actor + 移动能力 + 玩家组件 + 设定 + 响应"子模块：

- **弓能力（`IceBow/Capabilities/`）**：`EquipCapability`（装备）、`AimCapability`（瞄准）、`ChargeCapability`（蓄力）、`ShootCapability`（射击总控）+ 分箭种射击 `ShootIce/ShootWind/ShootRope/ShootBlizzard`；教程 `TutorialCapability` / `BlizzardArrowTutorialCapability`。
- **箭种子目录**（各一套 `Arrow` actor + `MovementCapability` + `PlayerComponent` + `Settings`）：
  - `IceArrow/` — 冰箭（含 `ResponseComponent` 命中响应）。
  - `WindArrow/` — 风箭（`AttachedCapability` 附着、`FollowBowCapability` 跟弓、`ResponseComponent`）。
  - `RopeArrow/` — 绳箭（`AttachedCapability` 拉索）。
  - `BlizzardArrow/` — 暴风雪箭（`AttachedCapability` + 专属 `PlayerComponent`）。
- **数据/静态**：`IceBowSettings`（`UHazeComposableSettings`）、`IceBowStatics`、`IceBowDataTypes`、`IceBowEventHandler`。

### 二、风矛 WindJavelin（蓄力投掷武器）

一杆可蓄力投掷的长矛，投出后成为独立弹体 actor：

- **投掷能力（`WindJavelin/Capabilities/`）**：`SpawnCapability`（生成）、`AimCapability`（瞄准）、`ChargeCapability`（蓄力）、`PrepareThrowCapability`（预备）、`ThrowCapability`（投出）、`TutorialCapability`。
- **弹体（`WindJavelin/Projectile/`）**：投出的矛作为独立 actor，自带 `Capabilities/`（飞行）与 `Components/`，命中走 `WindJavelinProjectileEventHandler`。
- **数据**：`WindJavelinSettings`（`UHazeComposableSettings`，命名空间 `WindJavelin` 存参数）、`WindJavelinStatics`、`WindJavelinEventHandler`、`Components/`。

### 三、跳伞辅助（`SkydiveAssistance/`）

巨人段的空中位移辅助，帮玩家在跳伞时吸附到目标点：

- `GiantsSkydiveMagnetCapability`（`UHazePlayerCapability`，磁吸）+ `GiantsSkydiveMagnetPlayerComponent` + `GiantsSkydiveMagnetPoint`（吸附点 actor）+ `GiantsSkydiveMagnetStatics`（命名空间 `GiantsStatics`）。
- `GiantsSkydiveAssistancePoint` / `GiantsSkydiveAssistancePlayerComponent`（辅助点）、`GiantsSkydiveBreakingSwingPoint`（摆荡减速点）。

---

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 依赖 | `AHazeSkeletalMeshActor` / `AHazeActor` / `AHazeAnimActor` | Core | 巨人 NPC、大钟、拉杆等场景 actor |
| 依赖 | `UHazePlayerCapability` / `UHazeCapabilityComponent` | Gameplay | 冰弓/风矛/跳伞磁吸的全部能力挂玩家容器（能力四件套） |
| 依赖 | `UHazeComposableSettings` | Gameplay | `IceBowSettings`、`WindJavelinSettings` 设定集 |
| 弹药范式 | Arrow / Javelin Projectile actor | Core | 箭矢/矛均为"武器发射独立弹体 actor + 移动能力"模式 |
| 赋速 | `GiantsSkydiveVelocityGiver` | Summit | 巨人给玩家跳伞助推，衔接 `SkydiveAssistance/` 磁吸段 |
| 特效 | `UHazeEffectEventHandler` | Core | IceBow / WindJavelin(Projectile) EventHandler 处理射击/命中表现 |
| 音频 | `Audio/` / KuroAudio | Audio | 巨人与武器音频挂接，见 [Audio](../../../Audio.md) |

---

## 关键文件

- 巨人：`Giants/TheGiant.as`、`Giants/Giant.as`、`Giants/GiantBell.as`、`Giants/GiantLever.as`
- 冰弓：`Giants/IceBow/Capabilities/IceBowShootCapability.as`、`Giants/IceBow/IceBowSettings.as`、`Giants/IceBow/IceArrow/IceArrow.as`、`Giants/IceBow/WindArrow/`、`Giants/IceBow/RopeArrow/`、`Giants/IceBow/BlizzardArrow/`
- 风矛：`Giants/WindJavelin/Capabilities/WindJavelinThrowCapability.as`、`Giants/WindJavelin/WindJavelinSettings.as`、`Giants/WindJavelin/Projectile/`
- 跳伞：`Giants/SkydiveAssistance/GiantsSkydiveMagnetCapability.as`、`Giants/SkydiveAssistance/GiantsSkydiveMagnetPoint.as`

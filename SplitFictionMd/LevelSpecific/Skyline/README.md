# LevelSpecific / Skyline — 赛博城市（科幻线）

> 路径：`LevelSpecific/Skyline/`（2119 文件）
>
> Mio 科幻线的高潮关卡：一座反重力赛博城市。核心体验是**高速载具 + 重力武器**——玩家骑重力摩托贴墙飞驰、开飞行车追逐、用重力刀/鞭近战，最终挑战多形态机械 Boss。

本文承接 [LevelSpecific 总览](../README.md) 与 [Gameplay 设计哲学](../../Gameplay/DesignPhilosophy.md)（能力四件套、标签互斥、组件容器）。Skyline 不发明新范式，是这套范式在"高速驾驶 + 重力操控"玩法族上的集中应用。哲学不再重复。

---

## 关卡主题与流程骨架

科幻线以"反重力"作为贯穿母题，机制围绕高速移动与重力操控展开：

1. **重力摩托（GravityBike）** — 贴墙/管道/斜坡飞驰的招牌载具，自由驾驶与在轨追逐。
2. **飞行车（FlyingCar）** — 双人飞车（驾驶员 + 炮手）高速公路追逐战。
3. **重力刀 / 重力鞭（GravityWeapons）** — 近战武器 + 抓取/处决（GloryKill），科幻线的人形战斗核心。
4. **喷气战斗（JetpackCombat）** — 喷气背包空中小段。
5. **机械 Boss** — 多形态巨型机器 Boss（含无人机 Boss DroneBoss）终局。

---

## 招牌机制索引

| 机制 | 文件数 | 一句话职责 | 文档 |
|---|---:|---|---|
| **重力摩托 GravityBike** | 414 | 贴墙/管道高速载具，自由驾驶 + 在轨追逐 | [GravityBike/](./GravityBike/) |
| **机械 Boss** | 336 (+26) | 多形态机械 Boss 终局（球 Boss/坦克/腿/飞行相，含无人机 Boss） | [Boss/](./Boss/) |
| **重力武器 GravityWeapons** | 87+38 | 重力刀（近战连招 + 抓钩）+ 重力鞭（抓取/投掷/处决） | [GravityWeapons/](./GravityWeapons/) |
| **飞行车 FlyingCar** | 74 | 双人飞车（驾驶员 + 炮手）公路追逐战 | [FlyingCar/](./FlyingCar/) |
| **热接线 Hotwire** | 9 | 破解/接线小交互 | [Hotwire/](./Hotwire/) |
| JetpackCombat | 7 | 喷气背包空中战斗小段 | 见下表说明 |

### 小机制说明

| 机制 | 关键类/目录 | 说明 |
|---|---|---|
| JetpackCombat | `Skyline/JetpackCombat/`（7） | 喷气背包空中战斗的一次性桥段能力，复用玩家能力容器 |
| CarChaseDive / CarChaseTether | `Skyline/CarChaseDive/`（5）、`Skyline/CarChaseTether/`（5，`UCarChaseTetherPlayerSettings`） | 飞车追逐的俯冲/牵引连接段，见 [FlyingCar](./FlyingCar/) |
| FlyingCar2D | `Skyline/FlyingCar2D/`（5，`ASkylineFlyingCar2D` + `TractorBeam`） | 飞车的 2D 横版变体段，见 [FlyingCar](./FlyingCar/) |
| Combat / EventHandlers / PlayerVisibility / BargeHatch | 各 1~3 | 关卡零散共享逻辑（近战判定、PVP 特效、玩家可见性、驳船舱门） |

---

## 通用目录（一笔带过，不建分篇）

以下目录是每关都有的样板基础设施，本文不展开：

- **`LevelActors/`（581）** — 通用关卡物件库：门、平台、可破坏结构、高速公路道具、霓虹/全息装饰、谜题道具等独立可放置 actor，按需查子目录名即可。
- **`AI/`（510）** — 通用敌人库。敌人 actor 多继承自 Gameplay 的 AI 基类，用 `UHazeCapabilityComponent` 挂行为能力。含机械兵、无人机、炮台等赛博单位。AI 框架本身见 [Gameplay/AI](../../Gameplay/README.md)。
- **`Audio/`（5）** — 关卡音频挂接，基于 KuroAudio/Wwise，见 [Audio](../../Audio.md)。
- 关卡根脚本：`SkylineLevelScriptActor.as`、`SkylineRootActor.as`、`SkylineTags.as`、`SkylineDevToggles.as`、`SkylinePlayerConditions.as`、`SkylinePVPEffectHandler.as`、`SkylineHighwayCombatStatics.as` 等。

---

## 机制间关系

- **Boss ↔ 重力摩托**：`ASkylineBoss` 身上挂 `UGravityBikeFreeHalfPipeTriggerComponent` / `UGravityBikeFreeAutoSteerTargetComponent`——Boss 表面就是摩托的半管跑道，玩家骑摩托在 Boss 身上贴面攻击。
- **飞行车 ↔ Move 系统**：`ASkylineFlyingCar` 用 `USkylineFlyingCarMovementResolver : USweepingMovementResolver`，与重力摩托共享 Gameplay 的扫掠移动求解框架。
- **重力刀/鞭 ↔ 自动瞄准**：两者的抓取/抓钩都基于 `UAutoAimTargetComponent` / `UTargetableComponent`（Core 目标系统），处决共用 GloryKill 语义。
- **飞车追逐链**：`FlyingCar` ↔ `CarChaseDive` / `CarChaseTether` / `FlyingCar2D` 组成追逐段的不同视角/形态。

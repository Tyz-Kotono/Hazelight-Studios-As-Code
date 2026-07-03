# LevelSpecific / Summit — 巨龙之山（奇幻线）

> 路径：`LevelSpecific/Summit/`（2327 文件，全库最大关卡）
>
> Zoe 奇幻线的高潮关卡：一座被风暴与石兽（StoneBeast）笼罩的巨龙之山。核心体验是**驯龙成长**——玩家从幼龙一路骑到成年龙，配合龙剑近战、巨人协助、风暴攻城战，最终登顶挑战石兽 Boss。

本文承接 [LevelSpecific 总览](../README.md) 与 [Gameplay 设计哲学](../../Gameplay/DesignPhilosophy.md)（能力四件套 Capability+Component+Settings+Statics、标签互斥、组件容器范式）。Summit 不发明新范式，是这套范式在"骑乘载具"这一超大玩法族上的集中应用。哲学不再重复。

---

## 关卡主题与流程骨架

奇幻线叙事把"龙"作为贯穿始终的成长载体，机制随龙的年龄递进：

1. **幼龙（BabyDragons）** — 龙挂在玩家身上（尾巴龙 Zoe / 酸液龙 Mio），提供攀爬、滑索、气流升空等辅助移动。
2. **可操控幼龙（ControlledBabyDragons）** — 短暂直接操控一条小龙的独立小段。
3. **少年龙（TeenDragons）** — 玩家"变身"为龙本体，获得地面奔跑、滚球、滑翔、酸液喷射、火焰吐息、尾巴攻击的完整动作组。
4. **成年龙（AdultDragons）** — 真正的空战骑乘：自由飞行、绕圈扫射、样条跟随、尾巴猛击、酸液光束/尖刺弹幕。
5. **龙剑（DragonSword）** — 落地后的近战连招武器，含蓄力回旋镖，用于石兽终局的破甲与弱点攻击。
6. **巨人（Giants）** — 与巨人 NPC 协作，使用冰弓（IceBow）与风矛（WindJavelin）远程武器。
7. **风暴攻城（StormSiege）** — 攻守城市与龙背 Boss 追逐的大型场景剧本。
8. **石头 Boss（StoneBoss）** — 石兽最终战，分追逐 / 峰顶 / 终局多阶段，回收前述所有能力。

详见下方招牌机制索引。

---

## 招牌机制索引

| 机制 | 文件数 | 一句话职责 | 文档 |
|---|---:|---|---|
| **龙骑乘体系** | 49+192+156+10 | Baby/Teen/Adult/ControlledBaby 四阶段龙的骑乘、飞行、能力 | [Dragons/](./Dragons/) |
| **石头 Boss（StoneBeast）** | 195 | 石兽最终战：追逐→峰顶→终局多阶段，回收全部能力 | [StoneBoss/](./StoneBoss/) |
| **风暴攻城战** | 212 | 攻守城市、攻城单位、龙背 Boss 追逐、坠落桥段 | [StormSiege/](./StormSiege/) |
| **龙剑 + 回旋镖** | 53+3 | 近战连招武器、蓄力回旋镖，破甲/弱点攻击 | [DragonSword/](./DragonSword/) |
| **巨人协作** | 77 | 巨人 NPC、冰弓、风矛、跳伞辅助 | [Giants/](./Giants/) |
| TreasureTemple | 34 | 宝藏神殿：俯视竞技场、滚木、地板破碎、宝藏门谜题 | 见下表说明 |
| CombatPrototypes | 26 | 夜后（NightQueen）战斗原型：晶体/金属变形、方阵、旋转盾墙 | 见下表说明 |
| Acid | 7 | 通用酸液系统：可溶解物、酸池、酸液响应组件 | 见下表说明 |
| AirCurrent | 7 | 上升气流：风口、风兜、激活交互，供飞行/滑翔借力 | 见下表说明 |

### 小机制说明

| 机制 | 关键类 | 说明 |
|---|---|---|
| TreasureTemple | `ATreasureGate`、`ASummitRollingLog*`、`ATreasureBreakableFloor`、`ATreasureTempleGemArenaTotem` | 俯视视角（Topdown）的宝藏神殿，含滚木躲避、地板逐块破碎、图腾竞技场，复用少年龙的俯视模式 |
| CombatPrototypes | `ANightQueenGemConjurer`、`ANightQueenMetalPhalanx`、`ANightQueenShieldRotator`、`AMetalMorpherAttack` | 反派夜后的战斗原型集：宝石/金属材质变形攻击、长枪方阵、旋转盾墙、悬浮岛攻城小兵 |
| Acid | `UAcidDissolvable`、`AAcidPuddle`、`UAcidResponseComponent`、`UAcidManagerComponent` | 通用酸液语义层：物体标记为可溶解，被酸命中后削弱/破坏；被龙的酸液能力与酸弹共用 |
| AirCurrent | `ASummitAirCurrent`、`ASummitRingAirCurrent`、`ASummitAirCurrentWindCatcher`、`ASummtiAirCurrentActivator` | 上升气流体积，玩家/龙进入后获得垂直推力；`AirCurrentCapability` 在 Baby/Teen 龙里都有对应版本借力升空 |

---

## 通用目录（一笔带过，不建分篇）

以下目录是每关都有的样板基础设施，本文不展开：

- **`LevelElements/`（724）** — 通用关卡物件库：门、吊桥、可破坏结构、链式物体、平台、锻造场（Forge）系列谜题道具、彩蛋路径等。数量最大但都是独立可放置 actor，按需查子目录名即可（如 `Doors/`、`Ballista/`、`ForgeLavaPuzzle/`）。
- **`AI/`（520）** — 通用敌人库。敌人 actor 多继承自 Gameplay 的 `ABasicAIGroundMovementCharacter`，用 `UHazeCapabilityComponent` 挂行为能力（Behaviours 子目录）。含骑士（Knight）、法师（Mage）、石兽小怪群（StoneBeast* 系列）、攻城操作员等。AI 框架本身见 [Gameplay/AI](../../Gameplay/README.md)。
- **`Audio/`（17）** — 龙的音频挂接：脚步/振翅/发声的 AnimNotify、移动音频能力与组件、俯视监听器。基于 KuroAudio/Wwise，见 [Audio](../../Audio.md)。
- 其余零散目录：`LevelScriptActors/`、`Tags/`、`Sidecontent/`（幼龙/少年龙的支线互动：果实、宝箱、可吃物）、`RollableLog/`、`AugustTestStuff/`（测试）。

---

## 机制间关系

- **龙 ↔ 酸液**：Baby/Teen/Adult 三阶段龙都有"酸液"变体（Mio 的酸液龙）与"尾巴"变体（Zoe 的尾巴龙），酸液能力命中 `UAcidDissolvable` 触发通用 `Acid/` 系统。
- **龙 ↔ 气流**：Baby 的 `AirBoost`、Teen 的 `AirGlide` 都通过 `AirCurrentCapability` 读取 `AirCurrent/` 体积获得升空推力。
- **龙剑 ↔ 石头 Boss**：`DragonSword` 与回旋镖用于石兽终局——攻击 `AStoneBeastCrystalJungle` 晶体、破 `StoneBreakable*`、执行弱点 QTE（`StoneBoss/Finale/Weakpoint/`）。
- **成年龙 ↔ 风暴/石兽追逐**：`StormSiege/Finale/DragonRunPlayerDragons/` 与 `StoneBoss/Chase/` 复用 `AdultDragon` 的酸弹/尾击设定驱动龙背 Boss 追逐。
- **石兽 = 一条巨型成年龙**：石兽头部 `AStoneBeastAdulDragonBase : AHazeCharacter` 是成年龙形态的放大特化版。
</content>

# LevelSpecific — 关卡专属实现

> 路径：`LevelSpecific/`（11,992 文件，占全库 76%，24 个关卡）
>
> 每个关卡一个文件夹，实现该关卡的**招牌机制**（signature mechanics）——只在这一关出现的独特玩法。Split Fiction 双线叙事：**Sci-Fi（Mio 科幻线）** 与 **Fantasy（Zoe 奇幻线）**。

---

## 每个关卡的通用结构

一个 `LevelSpecific/<关卡>/` 文件夹典型由三类子目录组成：

| 类别 | 常见目录 | 说明 |
|---|---|---|
| **通用基础设施** | `AI/`、`LevelActors/`、`Audio/`、`Editor/`、`LevelScriptActors/`、`Tags/` | 每关都有的样板：敌人、可放置道具、音频挂接、关卡脚本 actor。**本文档一笔带过** |
| **招牌机制** ⭐ | 如 `GravityBike/`、`Train/`、`Pinball/`、`Shapeshifting/` | 该关独有的核心玩法，是文档**重点** |
| **Boss / 分阶段** | `Boss/`、`StoneBoss/`、`BossPhaseOne/Two/Three` | Boss 战与分阶段 |

> 招牌机制模块内部仍遵循 Core/Gameplay 的**能力四件套**（Capability+Component+Settings+Statics）与**标签互斥**范式——见 [Core 设计哲学](../Core/DesignPhilosophy.md)、[Gameplay 设计哲学](../Gameplay/DesignPhilosophy.md)。LevelSpecific 不发明新范式，是这套范式在具体玩法上的大规模应用。

---

## 大关卡文档（文件数 ≥ 200，共 11 个）

每个关卡一个文件夹，含 `README.md`（关卡总览 + 招牌机制索引）与主要机制的分篇。

| 关卡 | 文件数 | 类型 | 招牌机制 | 文档 |
|---|---:|---|---|---|
| [Summit](./Summit/) | 2327 | 奇幻 | 各年龄龙骑乘、龙剑/回旋镖、风暴攻城、巨人、石头 Boss | [Summit/](./Summit/) |
| [Skyline](./Skyline/) | 2119 | 科幻 | 重力摩托/刀/鞭、飞行汽车、热接线、无人机 Boss | [Skyline/](./Skyline/) |
| [Island](./Island/) | 1361 | 科幻 | 喷气背包/水上摩托、红蓝武器、传送门、破盾武器 | [Island/](./Island/) |
| [Sanctuary](./Sanctuary/) | 1346 | 奇幻 | 飞行、蜈蚣 Boss、暗黑传送门、光鸟、Zoe 全屏 | [Sanctuary/](./Sanctuary/) |
| [Prison](./Prison/) | 1032 | 科幻 | 弹球、无人机、潜行、磁场、最高安保、Boss | [Prison/](./Prison/) |
| [Tundra](./Tundra/) | 823 | 奇幻 | 变形、河流、冰裂、抓取藤蔓 | [Tundra/](./Tundra/) |
| [Meltdown](./Meltdown/) | 545 | 科幻 | 打破分屏机制群、故障射击、三阶段 Boss | [Meltdown/](./Meltdown/) |
| [MoonMarket](./MoonMarket/) | 348 | 奇幻 | 药水、魔杖、猫、坩埚、南瓜发射器 | [MoonMarket/](./MoonMarket/) |
| [Coast](./Coast/) | 305 | 科幻 | 火车、滑水、翼装、肩炮、Boss | [Coast/](./Coast/) |
| [Dentist](./Dentist/) | 286 | 奇幻 | 单场 Boss 战竞技场 | [Dentist/](./Dentist/) |
| [Battlefield](./Battlefield/) | 254 | 科幻 | 悬浮板、火炮/导弹/激光/炮塔系统、场景剧本 | [Battlefield/](./Battlefield/) |

---

## 小关卡（文件数 < 200，共 13 个）→ [SmallLevels.md](./SmallLevels.md)

| 关卡 | 文件数 | 类型 | 主题 |
|---|---:|---|---|
| SolarFlare | 243 | 科幻 | 黑洞、太阳能激活物、终极桥段 |
| Sketchbook | 235 | 奇幻 | 手绘铅笔世界、弓/近战、骑马、独眼巨人 |
| Desert | 202 | 奇幻 | 沙鲨/沙手、抓钩鱼、滑坡漩涡 |
| GameShowArena | 126 | 科幻 | 拆弹、移动平台臂、主持人障碍赛 |
| PigWorld | 111 | 奇幻 | 可玩猪、猪迷宫、筒仓 |
| SpaceWalk | 71 | 科幻 | 零重力系绳、氧气、黑客小游戏 |
| OilRig | 68 | 科幻 | 可操控运兵船 |
| Village | 65 | 奇幻 | 潜行避食人魔、村民 |
| KiteTown | 52 | 奇幻 | 风筝飞行、母风筝、浮岛 |
| GoatWorld | 43 | 奇幻 | 激光眼/传送/泡泡/彩虹路羊技能 |
| RedSpace | 28 | 科幻 | 重力切换、方块操控 |
| SciFiTutorial | 4 | 科幻 | 白空间混合入门 |
| MovementTutorial | 1 | 科幻 | 移动入门 |

> SolarFlare(243)/Sketchbook(235)/Desert(202) 虽超 200，因整体较简单归入小关卡汇总；如需可单独展开。

---

## 阅读建议

- 想理解某关卡玩法**怎么实现的** → 进对应关卡文件夹看招牌机制分篇。
- 想理解**为什么这样写**（能力/标签/联机） → 先读 [Gameplay 设计哲学](../Gameplay/DesignPhilosophy.md)。
- 关卡机制大量复用 Gameplay 的 Move 系统、AI 框架、Combat、Pickups——见 [Gameplay](../Gameplay/README.md)。

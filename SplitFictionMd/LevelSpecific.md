# LevelSpecific — 关卡专属

> 路径：`SplitFiction/LevelSpecific`（11,992 文件，占整个代码库 76%）
>
> 每个游戏世界/关卡一个文件夹。Split Fiction 双线叙事：**Sci-Fi（Mio 的科幻故事）** 与 **Fantasy（Zoe 的奇幻故事）**。

---

## 通用文件夹结构

一个典型的 `LevelSpecific/<关卡>` 文件夹组织方式：

- **扁平的 depth-1 子文件夹**，每个按"自包含玩法系统/机制/actor 类型"命名（而非按代码层）。
- **共享的"基础设施"文件夹**（几乎每个关卡都有）：
  - `Audio` —— Wwise/关卡音频挂接
  - `AI` —— 敌人行为
  - `LevelActors` —— 关卡专属可放置 actor/道具
  - 常见：`EventHandlers`、`Editor`、`Player`、`Capabilities`
- **主体** 则是每个招牌机制/布景一个子文件夹（如 Skyline 的 `GravityBike`/`FlyingCar`/`Hotwire`；Island 的 `Jetpack`/`Jetski`/`Portal`）。
- **Boss 关卡** 额外加 `Boss`/`StoneBoss`/`DroneBoss` 及分阶段文件夹（Meltdown 有 `BossPhaseOne/Two/Three`）。
- **命名约定**：`<关卡><机制><角色>.as`，角色如 `Actor`、`Capability`、`Component`、`EffectHandler`、`Manager`、`Widget`。
- **小型关卡**（MovementTutorial、SciFiTutorial、SpaceWalk）跳过子文件夹层级，只放少量散落的 `.as` 文件。

---

## 关卡列表（按文件数排序）

| 关卡 (文件数) | 类型 | 主题与招牌机制 |
|---|---|---|
| **Summit** (2327) | 奇幻 | 巨龙之山史诗：各年龄可骑乘龙、龙剑/回旋镖、巨人、风暴攻城、石头 Boss。（注：`Giants` 巨人机制是 Summit 的子文件夹，非独立关卡） |
| **Skyline** (2119) | 科幻 | 赛博城市：重力摩托/刀/鞭、飞行汽车、热接线(Hotwire)、喷气背包战斗、无人机 Boss |
| **Island** (1361) | 科幻 | 科幻堡垒岛：喷气背包/水上摩托、传送门、力场、重力手雷、破盾武器 |
| **Sanctuary** (1346) | 奇幻 | 光暗圣域：飞行、暗黑寄生虫与传送门，以光鸟/光束之力对抗 |
| **Prison** (1032) | 科幻 | 科幻越狱：潜行、无人机、磁场、远程黑客、弹球、俯冲逃脱 |
| **Tundra** (822) | 奇幻 | 冰冻荒野：变形、抓取藤蔓、冰裂、河流穿越 |
| **Meltdown** (545) | 科幻 | 现实故障终章：打破分屏机制、故障剑/火箭/光束、三阶段 Boss |
| **MoonMarket** (348) | 奇幻 | 诡异女巫/万圣市集：药水、魔杖、坩埚、幽灵、南瓜发射器、猫谜题 |
| **Coast** (305) | 科幻 | 高速海岸追逐：水上滑水、翼装飞行、火车序列、肩炮战斗 |
| **Dentist** (286) | 奇幻 | 紧凑 Boss 竞技场关，围绕单场 Boss 战 |
| **Battlefield** (254) | 科幻 | 机械化战区：火炮、导弹、激光/炮塔、悬浮板穿越、钻地外星人 |
| **SolarFlare** (242) | 科幻 | 宇宙灾难：黑洞 actor、太阳能激活物、可移动物系统、终极桥段 |
| **Sketchbook** (235) | 奇幻 | 手绘铅笔世界：可绘制物、弓/近战、骑马、独眼巨人、Boss |
| **Desert** (202) | 奇幻 | 沙漠求生：沙鲨/沙手危险、抓钩鱼、滑坡、漩涡穿越 |
| **GameShowArena** (126) | 科幻 | 致命游戏秀竞技场：拆弹、移动平台臂、舱口、主持人障碍赛 |
| **PigWorld** (111) | 奇幻 | 异想猪世界：可玩猪角色、猪迷宫、筒仓、支线交互 |
| **SpaceWalk** (71) | 科幻 | 零重力太空行走：系绳/钩抓、氧气管理、黑客小游戏 |
| **OilRig** (68) | 科幻 | 工业钻井平台突袭：可操控运兵船、能力驱动关卡 actor |
| **Village** (65) | 奇幻 | 中世纪村庄：潜行避开食人魔、平民村民 |
| **KiteTown** (52) | 奇幻 | 天空风筝聚落：风筝飞行、母风筝、坠落危险、浮岛平台跳跃 |
| **GoatWorld** (42) | 奇幻 | 荒诞山羊乐园：激光眼、传送、泡泡、吞噬、彩虹路羊技能 |
| **RedSpace** (28) | 科幻 | 抽象重力切换维度：蔓延异常、粉碎机、方块操控、重力平台 |
| **SciFiTutorial** (4) | 科幻 | 简短科幻入门：白空间混合、淡入/移动物 |
| **MovementTutorial** (1) | 科幻 | 最小化移动入门关，单个教程组件 |

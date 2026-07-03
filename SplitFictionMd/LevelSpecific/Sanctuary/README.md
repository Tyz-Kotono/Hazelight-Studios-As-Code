# LevelSpecific / Sanctuary（奇幻光暗圣域）

> 路径：`LevelSpecific/Sanctuary/`（1346 文件，奇幻 / Zoe 线）
>
> Sanctuary 是 Zoe 奇幻线的高潮关卡：一座光与暗交织的圣域神殿。玩法主线是**光/暗能力对抗 + 骑乘飞行 + 全屏演出机关 + 多阶段九头蛇（Hydra）Boss 战**。玩家操纵光鸟（LightBird）、暗黑传送门（DarkPortal）等光暗道具，骑乘巨型伙伴（MegaCompanion）飞行进攻，穿越蜈蚣（Centipede）巨怪体内，在崩塌的圣域中与九头蛇决战。本层不发明新范式，是 Core/Gameplay 的**能力四件套**（Capability + Component + Settings + Statics）、**标签互斥**、**组件容器**在大规模奇幻 Boss 战与演出机关上的应用。上层设计见 [LevelSpecific 总览](../README.md)、[Gameplay 设计哲学](../../Gameplay/DesignPhilosophy.md)。

---

## 目录构成

| 类别 | 目录 | 说明 |
|---|---|---|
| **通用基础设施** | `AI/`(237)、`LevelActors/`(150)、`Audio/`、`Camera/`、`Essentials/`、`EventHandlers/` | 敌人 AI 库（Doppelganger 分身、Ghosts 幽灵、Lavamole 熔岩鼹、Weeper、FlightBoss 等原型）、可放置演员、音频、相机、关卡入口设施。沿用 Gameplay 的 AI 框架，**本文档一笔带过** |
| **招牌 Boss** ⭐ | `Boss/`(401)、`Centipede/`(115) | 多阶段九头蛇 Boss、蜈蚣巨怪穿越段 |
| **招牌载具/机关** ⭐ | `Aviation/`(72)、`ZoeFullScreen/`(86)、`DarkPortal/`(35)、`WeeperTopDown/`(32)、`SlidingDisc/`(30) | 骑乘飞行、全屏演出段、暗黑传送门、俯视光鸟解谜、滑行盘 |
| **光暗道具族** | `LightBird/`(21)、`LightProjectile/`(11)、`LightBeam/`(9)、`DarkMass/`(15)、`DarkParasite/`(14)、`DarkProjectile/`(12)、`Ghosts/`(20) | 光鸟、光束、光弹与暗黑质、暗黑寄生体、暗弹、幽灵——光/暗两系对抗道具 |
| **场景机关** | `Flight/`、`Boat/`、`Snake/`、`Essentials/` 等 | 飞行段、船、蛇形障碍等一次性关卡机关 |

---

## 招牌机制分篇

| 机制 | 文件数 | 一句话 | 文档 |
|---|---:|---|---|
| [Boss（九头蛇）](./Boss/) | 401(+115) | 多阶段 Hydra Boss：弩炮/勋章/滑行/样条奔跑/体内多段，含蜈蚣穿越 | [Boss/](./Boss/) |
| [骑乘飞行 Aviation](./Aviation/) | 72 | 骑乘巨型伙伴 MegaCompanion 飞行、俯冲攻击、双人协同 | [Aviation/](./Aviation/) |
| [全屏演出 ZoeFullScreen](./ZoeFullScreen/) | 86 | 分屏合并为全屏的世界操纵段：倾斜/旋转/转段/冰湖/光群 | [ZoeFullScreen/](./ZoeFullScreen/) |
| [暗黑传送门 DarkPortal](./DarkPortal/) | 35 | 瞄准发射传送门，抓取/拉扯/召回场景物体 | [DarkPortal/](./DarkPortal/) |

> 未单独分篇但值得一提的招牌机关：`WeeperTopDown/`(32) 俯视视角光鸟解谜（放大镜/符文/滑弹弓/光锥照明）、`SlidingDisc/`(30) 滑行盘（在九头蛇 Hydra 表面磨轨、变身为船的假切换）。二者结构与本目录其它玩家道具一致，可按需展开。

---

## Sanctuary 通用范式（读一处懂全部）

1. **光 / 暗二系对抗**
   全层围绕光（Light*）与暗（Dark*）两系道具展开：光系 `LightBird` / `LightBeam` / `LightProjectile` 用于照明、索敌、净化；暗系 `DarkPortal` / `DarkMass` / `DarkParasite` / `DarkProjectile` 用于抓取、腐蚀、传送。道具均遵循"可锁定 Target + 命中响应 Response + 玩家 UserComponent"三件套。

2. **Player 组件挂载 + 道具/载具按需生成**
   道具不常驻场景，状态存在玩家身上的 `UActorComponent`（`ULightBirdUserComponent`、`UDarkPortalUserComponent`、`USanctuaryCompanionAviationPlayerComponent` 等），能力激活时 `SpawnActor` 出实体、`ApplySettings` 施加设置，退出 `ClearSettingsByInstigator`（instigator 引用计数，与 Core/Flow 一致）。

3. **能力四件套 + 标签互斥 + Crumb 联机**
   每套玩法一组 `Capability`（Input / Movement / Camera / Attack / Tutorial 子类）配 `Settings`(DataAsset) + `Statics`(namespace) + `Data`。用 `CapabilityTags` + `BlockedWhileIn::*` 与移动系统互斥，联机统一走 `NetworkMode = Crumb`。

4. **全屏视点接管（Fullscreen ViewPoint）**
   `ZoeFullScreen/` 段落把常态分屏合并为单人全屏演出：`Player.ApplyViewSizeOverride(this, EHazeViewPointSize::Fullscreen)`，退出时 `ClearCameraSettingsByInstigator`。这是圣域"一人操纵世界、另一人观察"的招牌演出手法。

5. **Boss = 九头蛇 Hydra，多段拼接**
   `Boss/` 是全关最大目录（401 文件），以 `ASanctuaryBossHydraBase` + `SanctuaryBossHydraManagerComponent` 为核心，把弩炮（Ballista）、勋章（Medallion）、滑行（Slide）、样条奔跑（SplineRun）、跳伞（Skydive）、体内战（Inside）等一系列玩法段落拼成完整 Boss 战流程。

# LevelSpecific — 小关卡汇总（SmallLevels）

> 覆盖 13 个文件数较少的关卡。每关一节：主题 + 招牌机制子目录表 + 关键机制点。
>
> 大关卡（≥200 文件的 11 个）见 [各自文件夹](./README.md)。所有关卡机制都复用 [Gameplay](../Gameplay/README.md) 的能力四件套 / AI / Combat / Move 系统，本文只列各关**独有的招牌机制**。

---

## 索引

| 关卡 | 文件数 | 类型 | 招牌机制 |
|---|---:|---|---|
| [SolarFlare](#solarflare) | 243 | 科幻 | 太阳能系统、可移动物、终极桥段剧本 |
| [Sketchbook](#sketchbook) | 235 | 奇幻 | 弓/近战、铅笔绘制、飞船、山羊、Boss |
| [Desert](#desert) | 202 | 奇幻 | 沙鲨、漩涡、抓钩鱼、沙手 |
| [GameShowArena](#gameshowarena) | 126 | 科幻 | 拆弹、障碍、主持人、平台臂 |
| [PigWorld](#pigworld) | 111 | 奇幻 | 可玩猪、猪迷宫、筒仓 |
| [SpaceWalk](#spacewalk) | 71 | 科幻 | 氧气、钩爪、黑客小游戏 |
| [OilRig](#oilrig) | 68 | 科幻 | 可操控运兵船 |
| [Village](#village) | 65 | 奇幻 | 食人魔、潜行、村民 |
| [KiteTown](#kitetown) | 52 | 奇幻 | 风筝、风筝飞行、母风筝 |
| [GoatWorld](#goatworld) | 43 | 奇幻 | 山羊技能（激光眼/传送/泡泡/吞噬/彩虹路） |
| [RedSpace](#redspace) | 28 | 科幻 | 异常扩散、系绳、方块 |
| [SciFiTutorial](#scifitutorial) | 4 | 科幻 | 白空间混合入门 |
| [MovementTutorial](#movementtutorial) | 1 | 科幻 | 移动入门 |

---

## SolarFlare
> 科幻·宇宙灾难。243 文件。

| 子目录 | 文件 | 说明 |
|---|---:|---|
| `Scenarios` | 155 | 关卡剧本序列（终极桥段等） |
| `SolarFlareSystems` | 53 | 太阳能核心系统 |
| `SolarMovableObjectSystems` | 4 | 可移动物 |
| `InteractableSystems` / `SolarFlareActivatables` | 4/3 | 可交互/可激活物（太阳能驱动） |
| `VOHandlers` | 13 | 语音处理 |

主体是 `Scenarios` 驱动的线性剧本 + 太阳能激活谜题。

## Sketchbook
> 奇幻·手绘铅笔世界。235 文件。

| 子目录 | 文件 | 说明 |
|---|---:|---|
| `Boss` | 44 | 关卡 Boss |
| `Ship` | 37 | 飞船段 |
| `Goat` | 22 | 山羊 |
| `Pencil` | 17 | 铅笔绘制机制 |
| `Bow` / `MeleeWeapon` | 17/15 | 弓 / 近战武器（复用 Gameplay/Combat） |

战斗为主（弓+近战），配铅笔绘制与飞船段。

## Desert
> 奇幻·沙漠求生。202 文件。

| 子目录 | 文件 | 说明 |
|---|---:|---|
| `SandShark` (+`SandSharkHazard`) | 58+6 | 沙鲨捕食威胁 |
| `Vortex` | 46 | 漩涡传送/移动 |
| `GrappleFish` | 32 | 抓钩鱼（复用 Core 抓钩/样条） |
| `SandHand` | 19 | 沙手 |

## GameShowArena
> 科幻·致命游戏秀。126 文件。

| 子目录 | 文件 | 说明 |
|---|---:|---|
| `Bomb` (+`BombDisposal`/`BombAttachmentPlate`) | 23+ | 拆弹机制 |
| `LevelObstacles` | 15 | 障碍 |
| `Announcer` (+`AnnouncerHatch`) | 14+6 | 主持人/舱口 |
| `PlatformManager`/`PlatformArm`/`PlatformPlayerReaction` | 3+2+2 | 移动平台臂 |

> 注：`OLD/`(23) 为废弃代码。

## PigWorld
> 奇幻·异想猪世界。111 文件。

| 子目录 | 文件 | 说明 |
|---|---:|---|
| `PlayerPig` | 31 | 可玩猪角色（独立于标准玩家的操控） |
| `Silo` | 10 | 筒仓机制 |
| `PigMaze` | 9 | 猪迷宫 |

## SpaceWalk
> 科幻·零重力 EVA。71 文件。

| 子目录 | 文件 | 说明 |
|---|---:|---|
| `Oxygen` | 12 | 氧气管理 |
| `Hook` | 8 | 钩爪/系绳移动 |
| `HackingMinigame` | 4 | 黑客小游戏 |

## OilRig
> 科幻·钻井平台突袭。68 文件。

| 子目录 | 文件 | 说明 |
|---|---:|---|
| `ControllableDropShip` | 26 | 可操控运兵船（载具驾驶） |
| `LevelActors` | 40 | 关卡物件 |

## Village
> 奇幻·中世纪村庄。65 文件。

| 子目录 | 文件 | 说明 |
|---|---:|---|
| `Ogres` | 15 | 食人魔敌人 |
| `Stealth` | 10 | 潜行机制 |
| `Villagers` | 1 | 平民 NPC |

## KiteTown
> 奇幻·天空风筝聚落。52 文件。

| 子目录 | 文件 | 说明 |
|---|---:|---|
| `Kites` | 27 | 风筝物件 |
| `KiteFlight` | 11 | 风筝飞行操控 |
| `MotherKite` | 5 | 母风筝 |
| `FallingDeath` | 1 | 坠落死亡 |

## GoatWorld
> 奇幻·荒诞山羊乐园。43 文件。

| 子目录/文件 | 文件 | 说明 |
|---|---:|---|
| `GoatDevour` | 13 | 吞噬 |
| `GenericGoat` | 11 | 通用山羊 |
| `GoatLaserEyesCapability.as` | 1 | 激光眼 |
| `GoatTeleport`/`GoatBubble`/`GoatRainbowRoad` | 4/3/2 | 传送/泡泡/彩虹路 |

各山羊技能都是标准能力四件套的小实现，是"一技能一能力"的清晰范例。

## RedSpace
> 科幻·抽象重力切换维度。28 文件。

| 子目录 | 文件 | 说明 |
|---|---:|---|
| `Anomaly` | 5 | 异常扩散 |
| `Tether` | 4 | 系绳 |
| `LevelActors` | 19 | 方块等可操控物 |

## SciFiTutorial
> 科幻·入门教学。4 文件。白空间混合、淡入/移动物，把玩家引入科幻世界。

## MovementTutorial
> 科幻·移动入门。1 文件。最小化的移动教学组件。

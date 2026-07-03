# LevelSpecific / Tundra — 冰冻荒野（奇幻线）

> 路径：`LevelSpecific/Tundra/`（823 文件）
>
> Zoe 奇幻线的冰雪关卡：一片被冰王（IceKing）冻结的苔原荒野与猴族王国。核心体验是**变形（Shapeshifting）**——玩家在水獭 / 雪猴 / 仙灵 / 树守多种形态间切换解谜，配合顺流而下的河流段与冰裂谷的攀爬闯关，最终在冰宫挑战冰王。

本文承接 [LevelSpecific 总览](../README.md) 与 [Gameplay 设计哲学](../../Gameplay/DesignPhilosophy.md)（能力四件套 Capability+Component+Settings+Statics、标签互斥、组件容器范式）。Tundra 不发明新范式，是这套范式在"多形态角色切换"玩法族上的集中应用，哲学不再重复。

---

## 关卡主题与流程骨架

奇幻线以"解冻苔原、击败冰王"为主线，机制随地形推进：

1. **变形（Shapeshifting）** — 贯穿全关的核心：玩家在小型（水獭/仙灵）、玩家本体、大型（雪猴/树守）三档形态间切换，不同形态解不同谜题。
2. **河流（River）** — 大量顺流场景剧本：滑水（Waterslide）、巨石追逐、猴族的康茄舞/雪球战/舞蹈对决/西蒙说小游戏，以及冰宫 + 冰王 Boss。
3. **冰裂（Crack）** — 冰裂谷攀爬闯关：弹跳蘑菇、裂隙鸟（CrackBird）弹射、常青林（Evergreen）谜题群、木桶/拐杖等移动装置。
4. **抓取藤蔓（GrabberVines）** — 小型辅助机制，藤蔓抓取。

详见下方招牌机制索引。

---

## 招牌机制索引

| 机制 | 文件数 | 一句话职责 | 文档 |
|---|---:|---|---|
| **河流（River）** | 348 | 顺流场景群 + 猴族小游戏 + 冰宫冰王 Boss | [River/](./River/) |
| **变形（Shapeshifting）** ⭐ | 204 | 水獭/雪猴/仙灵/树守多形态切换核心系统 | [Shapeshifting/](./Shapeshifting/) |
| **冰裂（Crack）** | 150 | 冰裂谷攀爬闯关：裂隙鸟、常青林、木桶、拐杖 | [Crack/](./Crack/) |
| GrabberVines | 2 | 抓取藤蔓：玩家远程交互抓取点 | 见下表说明 |

### 小机制说明

| 机制 | 关键类 | 说明 |
|---|---|---|
| GrabberVines | `ATundraGrabberVines`（`AHazeActor`）、`UTundraGrabberVinesResponseComponent`（`UActorComponent`） | 关卡里的可抓取藤蔓：玩家远程交互触发，藤蔓响应组件处理被抓状态。仅 2 文件，通常与树守/仙灵的远程交互配合 |

---

## 通用目录（一笔带过，不建分篇）

- **`AI/`（90）** — 通用敌人/生物库：鱼（Fishie）、蚊虫（Gnat，含 Gnatapult/MonkeyTower）、Goom、猛禽（Raptor，含 Circle/Return/Targeting/Trapped 行为）、Shelly。多继承 Gameplay AI 框架并挂 `UHazeCapabilityComponent`（Behaviours）。见 [Gameplay/AI](../../Gameplay/README.md)。
- **`Audio/`（18）** — 音频挂接，含 `MonkeyConga`（猴族康茄舞音频）。基于 KuroAudio/Wwise，见 [Audio](../../Audio.md)。
- **`SideInteractions/`（6）** — 支线互动，如 `CrackBirdPlayerStuck`。

---

## 机制间关系

- **变形贯穿河流与冰裂**：River 与 Crack 都大量引用变形形态——`River/` 有 38 处、`Crack/` 有 21 处引用 `Shapeshift`；Crack 有 30 处引用具体形态（Otter/SnowMonkey/Fairy/TreeGuardian）。变形是这两大机制的通用能力底座。
- **形态 ↔ 谜题**：小型形态（水獭潜水、仙灵飞跃）过窄缝/水路，大型形态（雪猴攀爬、树守赋生/远程）破障碍——形态与地形谜题一一对应。
- **河流 ⊃ 冰王 Boss**：`River/Boss/`（SetupStage → ArenaStage）与 `River/IcePalace/`、`Tundra_River_Iceking` 构成冰宫终局。
- **树守赋生（LifeGiving） ↔ 生长物**：树守形态给枯萎植物赋予生命，河流段的 `TundraRiver_GrowingFlower`、冰裂段的 `TundraGrowingPlant`/`GrowingFlowers` 是其交互对象。

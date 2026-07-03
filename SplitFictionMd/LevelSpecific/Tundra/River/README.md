# LevelSpecific / Tundra / River — 河流场景群与冰王 Boss

> 路径：`LevelSpecific/Tundra/River/`（348 文件，Tundra 最大机制）
>
> Tundra 篇幅最大的机制，实为一连串**沿河推进的场景剧本集合**：滑水（Waterslide）顺流冲刺、巨石追逐、猴族王国的多种小游戏（康茄舞、雪球战、舞蹈对决、西蒙说），以及终点的冰宫（IcePalace）与冰王（IceKing）Boss 战。

承接 [Tundra 总览](../README.md)。河流段大量要求玩家[变形](../Shapeshifting/)成对应形态。

---

## 职责

编排河流沿线的顺流玩法、猴族互动小游戏与冰王 Boss 战。它不是单一系统，而是一组独立场景剧本 + 一批可放置河道障碍物。

---

## 内部架构

### 河道玩法与障碍物（River/ 根目录，64 文件）

大量以 `Tundra_River_*` 命名的独立 actor，构成顺流关卡：

| 类别 | 代表类 | 说明 |
|---|---|---|
| 滑水 | `Tundra_River_WaterslideLaunch_PlayerCapability`(+Component)、`WaterslideRamp`/`MiniRamp`/`WaterslideRocks`/`Waterslide_Geyser`、`Tunda_River_WaterslideFlipper` | 顺流滑水冲刺：斜坡加速、间歇泉弹射、翻板 |
| 浮冰/漂流 | `Tundra_River_FloatingIce`(+`AttachRope`)、`GeyserPlatform`、`BalanceGeyser` | 站浮冰漂流、间歇泉平台 |
| 障碍陷阱 | `BearTrap`(+DeathVolume+TriggerPlayerComponent)、`LogTrap`、`Stalactite`/`BreakStalactites`、`RollingRock`/`SlidingRock`/`AvalancheRock`(+RespawnSystem)、`BridgeObstacles` | 熊夹、滚木、钟乳石、雪崩巨石 |
| 破障 | `BreakableIceWall`、`SmashStoneWall`、`RockBlockage`、`VinesThatHoleTheStoneWall` | 需变形/交互破坏的墙 |
| 图腾谜题 | `TotemPuzzle`(+Button/DisplayHeads/MovingRoot/TreeControl)、`DuoTotemManager` | 双人图腾解谜 |
| 树/生长交互 | `TundraRiver_GrowingFlower`(+Spline 版)、`BridgeTreeInteraction`、`TreeGrappleSinkingTotemPole`、`RangedInteract_MovingMonkeyClimbActor` | 树守赋生/远程交互对象 |
| 摆荡 | `RotatingSwing`、`SwingWithWaterBucket` | 摆荡机关 |
| 冰王引子 | `Tundra_River_Iceking`（`AHazeActor`） | 河道中的冰王元素 |

### 冰王 Boss（Boss/，89 文件）

分两阶段编排：`SetupStage/`（Actors + Capabilities，布置阶段）→ `ArenaStage/`（Actors + Capabilities，竞技场决战），配 `PlayerCapabilities/`（Boss 战玩家专属能力）。相关：`Courtyard/IceKingStatue`、`IcePalace/`（37 文件，含 `LevelScripts/Tundra_IcePalace_IceKingLevelScriptActor`、`SideInteractions/`）。

### 猴族小游戏（River/ 子目录）

猴族王国的一系列双人/对抗小游戏，各自独立编排：

| 小游戏 | 目录（文件数） | 说明 |
|---|---|---|
| 西蒙说 | `SimonSays/`（40：Monkey/MonkeyKing/Player/States） | 状态机驱动的模仿小游戏 |
| 康茄舞 | `CongaLine/`（36：Mio/Monkey/MonkeyKing + `Mio/Capabilities/StrikePose`；含 `OldSystem/`） | 排队跳舞 |
| 舞蹈对决 | `DanceShowdown/`（35：Actors/ThrowableMonkey、EventHandlers、Managers、Player） | 舞蹈对抗 |
| 雪球战 | `MonkeySnowballFight/`（16：Monkey/Player + 各自 Projectiles） | 投掷雪球对战 |
| 猴族领地/巨石 | `MonkeyRealm/`（6）、`BoulderChase/`（9） | 领地互动、巨石追逐 |
| 浮杆攀爬 | `FloatingPoleClimb/`（8：Capabilities/Player） | 浮杆攀爬段 |
| 其它 | `InteractableBell/`（3）、`SphereProjectile/`（3）、`Courtyard/`（2） | 可敲铃、球形投射物 |

### 数据流（滑水示例）

```
玩家进入滑水段 → Tundra_River_WaterslideLaunch_PlayerCapability
   （AirSettings 关闭空气阻力/控制）
   → 沿 WaterslideRamp/MiniRamp 加速，Geyser 弹射
   → 躲避 BearTrap/RollingRock 等障碍 → 抵达下一场景
```

---

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 依赖 | 变形形态 | Shapeshifting | River 有 38 处引用 `Shapeshift`；破墙/水路/树守交互都要求特定形态 |
| 依赖 | `UPlayerMovementComponent` + AirMotion | Gameplay/Move | 滑水用 `UPlayerAirMotionSettings` 关闭空气阻力做顺流冲刺 |
| 输出 | 冰王 Boss | 本机制 | 树守 `HoldDownIceKing`、雪猴 `IceKingBossPunch`（定义在 Shapeshifting）在 `Boss/ArenaStage` 触发 |
| 引用 | 猴族 AI | AI | 小游戏中的猴子复用 `AI/`（Gnat/MonkeyTower 等）与康茄舞音频（`Audio/MonkeyConga`） |
| 依赖 | 交互/样条 | Gameplay | 图腾谜题、摆荡、浮杆走通用交互与样条系统 |

---

## 关键文件

- `River/Tundra_River_WaterslideLaunch_PlayerCapability.as` — 滑水顺流冲刺能力。
- `River/Tundra_River_FloatingIce.as` — 浮冰漂流。
- `River/Tundra_River_TotemPuzzle.as` — 双人图腾谜题。
- `River/Boss/SetupStage/` 与 `River/Boss/ArenaStage/` — 冰王 Boss 两阶段。
- `River/IcePalace/LevelScripts/Tundra_IcePalace_IceKingLevelScriptActor.as` — 冰宫关卡脚本。
- `River/SimonSays/`、`River/CongaLine/`、`River/DanceShowdown/`、`River/MonkeySnowballFight/` — 猴族小游戏。

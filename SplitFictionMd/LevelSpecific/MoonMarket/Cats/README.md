# LevelSpecific / MoonMarket / Cats（猫魂进程与子谜题）

> 路径：`MoonMarket/Cats/`(80) + `CatProgression/`(2) + `FlowerCatPuzzle/`(16)。合计约 98 文件。

## 职责

MoonMarket 体量最大的机制，也是关卡主线骨架。玩家在市集各处**收集猫魂**并送到猫头门，逐步解锁进程；沿途穿插几个以猫为主题的大谜题/Boss：巨守护猫的竖琴潜行、行走的芭芭雅嘎鸡腿屋、双人塔谜题与符号轮盘。猫进程用 `UMoonMarketCatProgressComponent` 全关持久化，是把散落玩法串成一条主线的中枢。花猫谜题（FlowerCatPuzzle）作为花园进程的一环并入本篇。

## 内部架构

### 核心：猫魂收集与进程

- **猫实体** `AMoonMarketCat : AHazeActor`（`Cats/MoonMarketCat.as`）：每只有 `CatType`（`EMoonMarketCatProgressionAreas`：GardenCat / BabaYagaCat / GraveyardCat / EntranceCat）、猫魂碰撞球，默认能力 `MoonMarketCatSoulFollowCapability` + `MoonMarketCatSoulToGateCapability`。
- **收集链路**：玩家交互 → `MoonMarketPlayerSoulCollectCapability`（`Cats/SoulCapabilities/`）播捕捉动画 → 猫魂跟随玩家（`MoonMarketCatSoulFollowCapability : UHazeCapability`）→ 飞向猫头门交付（`MoonMarketCatSoulToGateCapability`）解锁进程。
- **门与容器**：`MoonMarketCatHolder`、`MoonGateCatHead` / `MoonTowerCatHead`、`MoonTowerGate`、`TownSquareCatIntroGate`、`CatCollectionGate/`（`AMoonMarketSoulCollectionGate`，`ADoubleInteractionActor` 门控）、`CatPawPrints/`（样条足迹）、`GardenCat/`、`FlowerGate/`。

### 进程持久化 CatProgression（2）

- `CatProgression/MoonMarketCatProgressComponent.as`（`UActorComponent`，单一 `OnProgressionActivated` 事件）——被 Cats、宝箱怪 Mimic、毛线屋状态管理器等**全关复用**的进程钩子。
- `CatProgression/MoonMarketCatProgressManager.as`（`AMoonMarketCatProgressManager : AHazeActor`）——持 `FMoonMarketProgressData`（猫 + 联动 Actor），监听 `AMoonMarketCat` 的 `OnMoonCatSoulCaught` / `OnMoonCatFinishDelivering`，用 `Save::ModifyPersistentProfileFlag` 持久化；读档时对每个联动 Actor 的组件调 `SetProgressionActivated()` 回放状态。

### 猫主题子机制（都在 Cats/ 内）

| 子机制 | 文件 | 关键类 | 玩法 |
|---|---|---|---|
| 巨守护猫 BigCat | 18 | `AMoonGuardianCat : AHazeActor` | 沉睡巨猫潜行谜题：玩家弹竖琴（`Harp/`，7 文件，`AMoonGuardianHarp` + 演奏能力/控件/平台管理）维持其睡眠；脚步/错音吵醒（`EMoonGuardianCatWakeUpReason`）触发 `Roar()` 把玩家弹飞。含 Sleep/Awake/Roar/Animation 能力、`MoonMarketCatTower(Rotator)`、`MoonGuardianDreamPlatform`、`Zees`（睡眠 Z）|
| 毛线猫/芭芭雅嘎 YarnCat | 18 | `AYarnCatManager` + `BabaYaga/` | 行走的芭芭雅嘎鸡腿屋 Boss：腿 Actor、根移动器、动画实例、眼睛 Actor；`YarnCatManager` 事件后启用猫；钟楼拉杆/梯子管理器；`BabaYagaSwingCamera` 自定义相机；死亡体积/几何管理器 |
| 塔路径激活 TowerPathActivation | 10 | `ATowerCube : AHazeActor` | 双人协作：两人站上连通方块（`OnAnyImpactByPlayer`，各自符号）点亮路径；`TowerCubeManager`、`TowerControllable`、`TowerPathPiece`、旋转/移动能力 |
| 符号轮盘 TowerRoulette | 9 | `AMoonMarketSymbolRouletteManager : AHazeActor` | 符号配对状态机（`EMoonMarketSymbolRouletteState`：Initiate/SetLine/RouletteSpin/Countdown/CheckAnswer），5 个能力驱动；`MoonMarketSymbolCube`、`SymbolCompletedPlatform` |

### 花猫谜题 FlowerCatPuzzle（16）

`AFlowerCatPuzzle : AHazeActor`。花园谜题，解开后经共享的 `MoonMarketGardenGate` / 猫进程系统门控花园/猫进程。玩家戴花帽（`MoonMarketFlowerHat`，交互能力），跳舞用 `MoonMarketFlowerTrailCapability : UHazePlayerCapability` 留下彩色花径（`ApplySettings` + `Player.Mesh.RequestLocomotion` + `UMoonMarketPlayerFlowerSpawningComponent.CrumbSpawnFlowers`）；花（`AMoonMarketFlower : AStaticMeshActor`）须以正确颜色覆盖 `AFlowerCatPuzzlePiece` 圆圈；全部完成广播 `OnFlowerCatPuzzleCompleted`、开 `FlowerCatPuzzleVineDoor`。含擦除/教学能力、`FlowerPuzzleStaticSettings`、`MoonMarketFlowerInstanceData`。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| → | Core 存档 | 进程持久化 | `AMoonMarketCatProgressManager` 用 `Save::ModifyPersistentProfileFlag`，读档 `SetProgressionActivated()` 回放 |
| ← | 猫实体事件 | 进程推进 | Manager 监听 `AMoonMarketCat.OnMoonCatSoulCaught` / `OnMoonCatFinishDelivering` |
| → | 全关联动对象 | 进程钩子复用 | `UMoonMarketCatProgressComponent` 被 ChestLauncher(Mimic)、`MoonMarketYarnHouseStateManager`、门/拟态等内嵌 |
| → | Core 双人交互 | 收集门 | `AMoonMarketSoulCollectionGate` 由 `ADoubleInteractionActor` 门控 |
| → | Core Settings/动画 | 花径 | FlowerTrail `ApplySettings` + `Player.Mesh.RequestLocomotion` |
| ↔ | Core Crumb 联机 | 花/符号同步 | `CrumbSpawnFlowers`；塔/轮盘能力 `NetworkMode=Crumb` |
| → | Core 相机 | Boss 演出 | YarnCat `BabaYagaSwingCamera` 自定义相机 |

## 关键文件

- `Cats/MoonMarketCat.as` — 猫实体 `AMoonMarketCat` + `CatType` + 猫魂能力
- `Cats/SoulCapabilities/MoonMarketPlayerSoulCollectCapability.as` — 玩家收集猫魂
- `CatProgression/MoonMarketCatProgressManager.as` — 进程持久化管理器（全关中枢）
- `CatProgression/MoonMarketCatProgressComponent.as` — 全关复用的进程钩子组件
- `Cats/BigCat/` + `Cats/BigCat/Harp/` — 巨守护猫竖琴潜行（18 文件）
- `Cats/YarnCat/` + `Cats/YarnCat/BabaYaga/` — 芭芭雅嘎鸡腿屋 Boss（18 文件）
- `FlowerCatPuzzle/FlowerCatPuzzle.as` — 花猫谜题 `AFlowerCatPuzzle` + 花径玩法

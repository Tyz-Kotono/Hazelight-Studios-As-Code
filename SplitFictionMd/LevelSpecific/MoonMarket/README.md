# LevelSpecific / MoonMarket（月光女巫市集）

> 路径：`LevelSpecific/MoonMarket/`（348 文件，奇幻 / Zoe 线）
>
> MoonMarket 是一座奇幻女巫夜市。玩法主线是**女巫魔法 + 变身**：坩埚熬药、喝下药水变身（雷云 / 喇叭头 / 弹力糖球…），挥动魔杖把物件与自己变形（polymorph），沿途收集猫魂推进主线，还要在巨猫脚下潜行、躲避行走的芭芭雅嘎鸡腿屋。本关不发明新范式，是 Core/Gameplay 的**能力四件套**、**统一交互基类**、**共享响应组件事件总线**在一整套魔法玩法上的应用。上层设计见 [Core 设计哲学](../../Core/DesignPhilosophy.md)、[Gameplay 设计哲学](../../Gameplay/DesignPhilosophy.md)。

---

## 目录构成

| 类别 | 目录 | 说明 |
|---|---|---|
| **通用基础设施** | `Interactions/`(9)、`Editor/`、`AmbientCritter/`、`FloatingLights/`、`Fireworks/`、`Bonfire/`、`WaterBubbles/`、`Moth/`、`Ghosts/`、`FollowBalloon/` 及根目录散件 | 交互基类、氛围演出、根目录通用组件（Bobbing/FollowSpline/NPCStun/NPCWalk/Shapeshift）。**本文档一笔带过** |
| **招牌机制** ⭐ | `Cats/`(80)、`Potions/`(50)、`Wands/`(32)、`Cauldron/`(12) | 猫魂进程、药水变身、魔杖施法、坩埚熬药 |
| **小机制** | `FlowerCatPuzzle/`(16)、`Snail/`(13)、`Mole/`(13)、`BouncyMushroom/`(13)、`ChestLauncher/`(12)、`PumpkinLauncher/`(3)、`Yarn/`(9) | 花猫谜题、蜗牛坐骑、鼹鼠、弹跳蘑菇、宝箱怪、南瓜发射器 |

---

## 招牌机制分篇

| 机制 | 文件数 | 一句话 | 文档 |
|---|---:|---|---|
| [药水 Potions](./Potions/) | 50 | 喝药水变身：雷云 / 喇叭头 / 弹力糖球 / 毛线球 / 蘑菇 / 变形，共享 `UMoonMarketPlayerShapeshiftCapability` 变身基类 | [Potions/](./Potions/) |
| [魔杖 Wands](./Wands/) | 32 | 瞄准施法：变形术为主，把命中物件/自己 polymorph 成奶酪/香肠/椅子 | [Wands/](./Wands/) |
| [猫 Cats](./Cats/) | 80 | 收集猫魂推进主线，含巨守护猫竖琴潜行、芭芭雅嘎鸡腿屋、塔谜题、猫进程持久化（含 FlowerCatPuzzle） | [Cats/](./Cats/) |

---

## 小机制速览（表格带过）

| 机制 | 目录(文件) | 基类 | 做什么 |
|---|---|---|---|
| 坩埚 Cauldron | `Cauldron/`(12) | `AMoonMarketWitchCauldron : AHazeActor` | 玩家搬运/投入两份食材（`AMoonMarketCauldronIngredient`，`HoldIngredient`/`ThrowIngredientInCauldron` 能力），匹配 `UMoonMarketCauldronRecipeDataAsset` 熬出赋予变身形态的药水——药水系统的上游 |
| 花猫谜题 FlowerCatPuzzle | `FlowerCatPuzzle/`(16) | `AFlowerCatPuzzle : AHazeActor` | 戴花帽跳舞留下彩色花径（`MoonMarketFlowerTrailCapability`），花色对准 `AFlowerCatPuzzlePiece` 圆圈解谜，完成开藤门、推进猫/花园进程（并入 [Cats](./Cats/)） |
| 蜗牛 Snail | `Snail/`(13) | `AMoonMarketSnail : AMoonMarketInteractableActor`(Vehicle) | 可骑蜗牛坐骑，留下打滑黏液轨迹（`SnailTrailComponent` + 滑倒能力），带 idle/回家/移动能力与多种工具响应 |
| 鼹鼠 Mole | `Mole/`(13) | `AMoonMarketMole : AHazeActor` | 沿样条游走的 NPC，用响应组件对雷/糖/烟花/喇叭/变形等所有玩家工具反应，走优先级 `AMoonMarketMoleManager` 反应系统 |
| 弹跳蘑菇 BouncyMushroom | `BouncyMushroom/`(13) | `AWitchBouncyMushroomActor : AHazeActor` | 玩家踩踏即向上弹射（`OnGroundImpactedByPlayer` 大上冲量，糖球形态减半）；另含蘑菇人 NPC 群 |
| 宝箱怪 ChestLauncher | `ChestLauncher/`(12) | `AMoonMarketMimic : AHazeActor` | 拟态宝箱"吞"下玩家再发射（`PlayerEnterChest`→`PlayerChestLauncher` 能力）；可被踢来吞/放猫，内嵌 `UMoonMarketCatProgressComponent` |
| 南瓜发射器 PumpkinLauncher | `PumpkinLauncher/`(3) | `AMoonMarketPumpkinLauncher : AHazeActor` | 交互钻进南瓜，挤压瞄准 `LandingLocation` 后 `LaunchPlayerTo` 把玩家弹过缺口（带镜头/震动） |

---

## MoonMarket 通用范式（读一处懂全关）

1. **统一交互基类**
   `AMoonMarketInteractableActor : AHazeActor`（`Interactions/`）持 `UInteractionComponent` + `InteractableTag`（Balloon/Potion/Shapeshift/FlowerHat/Lantern/Wand/Ingredient/Vehicle）+ `bCancelByThunder`。可拾取物再派生 `AMoonMarketHoldableActor`（加拾取/丢弃/回原点）。药水、魔杖、坩埚食材、蜗牛、猫、发射器全部走这条链。

2. **变身系统单一中枢**
   根目录 `UMoonMarketShapeshiftComponent : UActorComponent` 是药水与魔杖变身的共同心脏：生成/持有联网的形态 holder（`AMoonMarketShapeshiftShapeHolder`），存 `FMoonMarketShapeshiftShapeData`（半径、移动标志、雷电可取消），广播变形/复原事件。药水的 `UMoonMarketPlayerShapeshiftCapability` 与魔杖变形术**共用它**——魔杖 `MoonMarketPolymorphPlayerCapability` 直接继承药水的变身基类。

3. **"工具打 NPC"事件总线**
   一组响应组件构成统一事件总线，被蜗牛/鼹鼠/蘑菇人/坩埚等复用：`UMoonMarketThunderStruckComponent`（雷击）、`UFireworksResponseComponent`（烟花）、`UMoonMarketBouncyBallResponseComponent`（弹球）、`UMoonMarketTrumpetHonkResponseComponent`（喇叭吹飞）、`UPolymorphResponseComponent`（变形术）。通用眩晕 `UMoonMarketNPCStunComponent` + `Capability` 订阅这些事件并封锁 NPC 移动/动作。

4. **能力四件套 + Crumb 联机**
   全关 85 个类派生自 `UHazePlayerCapability` / `UHazeCapability`，配 Component/Settings/EventHandler，`NetworkMode=Crumb`，与 Gameplay 一致。`ApplySettings` / `ApplyCameraSettings` / `ApplySplineCollision` 在 11 处用于气球重力、雷云相机+样条、蜗牛样条、花径地面运动等，走标准 Haze 设置栈而非自写状态。

5. **猫进程持久化**
   `UMoonMarketCatProgressComponent`（`CatProgression/`）是全关进程钩子，被 Cats、宝箱怪 Mimic、毛线屋状态管理器等复用；`AMoonMarketCatProgressManager` 监听猫魂捕获/交付事件，用 `Save::ModifyPersistentProfileFlag` 持久化，读档时回放状态。

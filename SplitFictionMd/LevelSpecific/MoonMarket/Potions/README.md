# LevelSpecific / MoonMarket / Potions（药水·变身）

> 路径：`MoonMarket/Potions/`（50 文件）

## 职责

MoonMarket 的核心变身玩法。女巫坩埚熬出药水，玩家喝下即变身成不同形态，每种形态有专属移动与技能：雷云降雷、喇叭头吹飞、弹力糖球滚射、毛线球滚动、蘑菇、变形。所有变身共享同一个变身基类能力与根目录的 `UMoonMarketShapeshiftComponent` 中枢，是本关"魔法即玩法"的主干。药水的上游是[坩埚 Cauldron](../README.md#小机制速览表格带过)，下游变身系统还被[魔杖 Wands](../Wands/)复用。

## 内部架构

### 药水实体与"喝→变身"链路

- **药水实体** `AMoonMarketPotion : AMoonMarketInteractableActor`（`Potions/MoonMarketPotion.as`），带 `EMoonMarketPotionID`（ThunderCloud / TrumpetHead / Polymorph / YarnBall / Mushroom / BallBlaster）与一个 `SheetToStartWhenConsumed`（`UHazeCapabilitySheet`）。
- **交互喝药** `UMoonMarketPotionInteractionCapability : UHazePlayerCapability`（`Potions/Interaction/`）：播放喝药动画，经 `UMoonMarketPotionInteractionComponent`（`UActorComponent`）启动药水携带的能力表 → 激活对应形态能力。变身达成的成就（`AllCauldronForms`）用 `Profile::SetProfileValue` 记录。
- **变身基类** `UMoonMarketPlayerShapeshiftCapability : UHazePlayerCapability`（`Potions/MoonMarketPlayerShapeshiftCapability.as`）：生成形态 Actor、联网、驱动根目录 `UMoonMarketShapeshiftComponent`。**每种药水形态都继承它**（魔杖变形术也继承它）。

### 药水形态

| 形态 | 目录(文件) | 关键类 | 玩法 |
|---|---|---|---|
| 雷云 Thundercloud | `Thundercloud/`(8) | `AMoonMarketLightningStrike` + `UMoonMarketThunderStruckComponent` | 变悬浮雨云，主技能向地面/NPC 落雷（眩晕）；自定义扫掠移动 + 样条碰撞 + 阴影贴花 |
| 喇叭头 TrumpetHead | `TrumpetHead/`(5) | `UMoonMarketTrumpetHonkResponseComponent` | 头变喇叭，"吹奏"做胶囊检测的空气推力冲飞 NPC |
| 弹力糖球 BouncyBallBlaster | `BouncyBallBlaster/`(19，最大) | `AMoonMarketBouncyBallBlaster : AHazeActor` | 可骑的滚动射击炮，发射 `MoonMarketBouncyBall` 弹；完整移动栈（Movement/Rolling/Rotation/Shoot/Bounce 能力）+ resolver + 响应组件 |
| 气球 Balloon | `Balloon/`(5) | `UMoonMarketBalloonPotionCapability` + `AMoonMarketCandyBalloonForm` | 变弹跳糖果气球，自定义重力/弹跳物理；含吃糖交互 |
| 毛线球 YarnBall | `YarnBall/`(3) | `MoonMarketPlayerExitYarnBallCapability` | 滚成会散开的毛线球，跳跃/取消退出；共用 `Yarn/` 移动解算 |
| 蘑菇 Mushroom | `Mushroom/`(2) | — | 最简形态，变身成蘑菇 Actor |
| 变形 Polymorph | `Polymorph/`(2) | `UMoonMarketPolymorphPotionCapability` + `UMoonMarketPolymorphPotionComponent` | 切换持久玩家形变（`bIsTransformed`） |

根目录还有 `MoonMarketPotionEventHandler`（`UHazeEffectEventHandler`）与 `AMoonMarketShapeshiftShapeHolder`（联网形态 holder）。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| ← | 坩埚 Cauldron | 药水来源 | 坩埚匹配 `UMoonMarketCauldronRecipeDataAsset` 熬出 `AMoonMarketPotion` |
| → | 根 `UMoonMarketShapeshiftComponent` | 变身中枢 | `UMoonMarketPlayerShapeshiftCapability` 生成/驱动形态 holder + `FMoonMarketShapeshiftShapeData` |
| ↔ | 魔杖变形术 | 共用变身基类 | `Wands/.../MoonMarketPolymorphPlayerCapability : UMoonMarketPlayerShapeshiftCapability` |
| → | Core 交互 | 喝药 | `AMoonMarketPotion : AMoonMarketInteractableActor` + `UInteractionComponent`；能力表 `UHazeCapabilitySheet` |
| → | Core Settings/相机/样条 | 形态特化 | `ApplySettings` / `ApplyCameraSettings` / `ApplySplineCollision`（气球重力、雷云相机+样条） |
| → | "工具打 NPC"总线 | 技能命中 | 雷云→`UMoonMarketThunderStruckComponent`、喇叭→`UMoonMarketTrumpetHonkResponseComponent`（被蜗牛/鼹鼠订阅） |
| → | Core 存档 | 成就 | `Profile::SetProfileValue`（`AllCauldronForms`） |

## 关键文件

- `Potions/MoonMarketPotion.as` — 药水实体 `AMoonMarketPotion` + `EMoonMarketPotionID`
- `Potions/MoonMarketPlayerShapeshiftCapability.as` — 变身基类能力（全形态与魔杖共用）
- `Potions/Interaction/MoonMarketPotionInteractionCapability.as` — 喝药→启动形态能力表
- `Potions/BouncyBallBlaster/` — 最复杂形态（可骑滚射炮，19 文件）
- `Potions/Thundercloud/` — 雷云降雷（8 文件，落雷+样条+相机）
- `../MoonMarketShapeshiftComponent.as` — 根目录变身中枢组件

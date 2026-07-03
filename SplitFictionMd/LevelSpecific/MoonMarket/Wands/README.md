# LevelSpecific / MoonMarket / Wands（魔杖·施法）

> 路径：`MoonMarket/Wands/`（32 文件）

## 职责

MoonMarket 的瞄准施法玩法。玩家拾取魔杖，瞄准并施放法术——本关主打**变形术（Polymorph）**：把命中的物件或自己变成奶酪、香肠、椅子等形态（也预留冰弹、悬浮、缩放等杖型枚举）。魔杖是拾取式 `AMoonMarketHoldableActor`，走标准"拾取→瞄准→施法"三段能力链，变形结果复用[药水](../Potions/)的变身系统。

## 内部架构

### 魔杖实体与玩家组件

- **魔杖基类** `AWandBase : AMoonMarketHoldableActor`（`Wands/WandBase.as`，默认 `InteractableTag = Wand`，带 bobbing 悬浮）。
- **玩家组件** `UWandPlayerComponent : UActorComponent`（`Wands/WandPlayerComponent.as`）：拾取后登记 `FWandPlayerData`，存杖型 `EMoonMarketWandType`（None / IceBolt / Levitate / Resizer / Polymorph）。

### 三段施法能力链（均 `UHazePlayerCapability`）

1. `Wands/Capabilities/WandAimCapability.as`——用 `UPlayerAimingComponent` 起瞄，自动瞄准 + 自定义准星 `MoonMarketWandCrosshair`。
2. `Wands/Capabilities/WandCastCapability.as`——`PrimaryLevelAbility` 触发：播施法动画 → `ECC_WorldDynamic` 线检找 `UPolymorphResponseComponent` 目标 → 调 `Wand.FinishCasting(FSpellHitData)`。
3. `Wands/WitchWandCapability.as`——门控能力，要求手持魔杖才放行。

### 变形杖 Polymorph（25，主体）

- **杖实体** `AWandPolymorph : AWandBase`：轮换 `PossibleMorphs`，对命中 Actor 调 `UPolymorphResponseComponent::NetRequestPolymorph`。
- **玩家侧变形** `MoonMarketPolymorphPlayerCapability : UMoonMarketPlayerShapeshiftCapability`（**复用药水变身基类**）+ 移动/冲刺/空中冲刺能力 + `MoonMarketPolymorphShapeComponent`（`UActorComponent`，持 `FMoonMarketShapeshiftShapeData` + `UHazeCapabilitySheet`）+ 自动瞄准组件 + 阻断组件。
- **具体形态** `Polymorph/Polymorphs/`：**Cheese**（4，自定义移动/resolver）、**Sausage**（6，空中/跳跃/移动能力 + settings）、**Chair**（`MoonMarketPolymorphChair`）。
- `WandEffectHandler`、`MoonMarketWandCrosshair` 收尾。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| → | Core 交互/拾取 | 魔杖基类 | `AWandBase : AMoonMarketHoldableActor`（→`AMoonMarketInteractableActor` + `UInteractionComponent`） |
| → | Core 瞄准 | 起瞄 | `WandAimCapability` 走 `UPlayerAimingComponent` + 自定义准星 |
| ↔ | 药水变身系统 | 共用基类 | `MoonMarketPolymorphPlayerCapability : UMoonMarketPlayerShapeshiftCapability`（药水基类） |
| → | `UPolymorphResponseComponent` | 变形契约 | 施法线检找该组件、`NetRequestPolymorph`；蜗牛/鼹鼠等 NPC 都挂它 |
| → | Core Settings | 形态参数 | Sausage 等携带 `UHazeComposableSettings` 风格设置 |
| ↔ | Core Crumb 联机 | 联网施法 | `NetRequestPolymorph` 等走网络请求，能力 `NetworkMode=Crumb` |

## 关键文件

- `Wands/WandBase.as` — 魔杖基类 `AWandBase : AMoonMarketHoldableActor`
- `Wands/WandPlayerComponent.as` — 玩家魔杖组件 + `EMoonMarketWandType`
- `Wands/Capabilities/WandCastCapability.as` — 施法：线检 + `FinishCasting(FSpellHitData)`
- `Wands/Polymorph/` — 变形杖主体（25 文件，含玩家侧变形能力与形态组件）
- `Wands/Polymorph/Polymorphs/` — 奶酪/香肠/椅子具体形态

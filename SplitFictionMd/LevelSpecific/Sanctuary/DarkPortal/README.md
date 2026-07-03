# LevelSpecific / Sanctuary / DarkPortal（暗黑传送门）

> 路径：`LevelSpecific/Sanctuary/DarkPortal/`（35 文件）

## 职责

圣域暗系招牌道具。玩家瞄准发射暗黑传送门，把场景物体（或敌人）**抓取、拉扯、召回**——传送门吸附到目标上，再拉近/拖动/回收。玩法状态机 `EDarkPortalState`：`Absorb`（吸附）→ `Launch`（发射）→ `Settle`（附着稳定）→ `Recall`（召回）。配合暗黑伙伴（DarkPortalCompanion，触手怪）辅助瞄准与操作。

## 内部架构

- **实体** `ADarkPortalActor : AHazeActor`（传送门本体）+ 数据结构 `FDarkPortalTargetData`（目标场景组件 / 插槽 / 相对位置法线 / 是否被遮挡 / 可旋转特例）。核心枚举 `EDarkPortalState`。
- **玩家侧** `UDarkPortalUserComponent : UActorComponent`（中枢，持 `DarkPortalClass` / `CrosshairWidgetClass` / `Companion`）+ `DarkPortalData` / `DarkPortalSettings` / `DarkPortalStatics` / `DarkPortalCrosshairWidget`。
- **传送门组件族**：`DarkPortalArmComponent`（暗黑手臂表现）、`DarkPortalAutoAimComponent`（自动瞄准）、`DarkPortalAutoPlacementComponent`（自动落点）、`DarkPortalForceAnchorComponent`（强制锚点）、`DarkPortalTargetComponent`（目标）、`DarkPortalResponseComponent`（场景侧命中响应）。
- **能力**（`Capabilities/`）：
  - 传送门行为：`ArmEffect`（手臂特效）、`Grab`（抓取）、`Pull`（拉扯）、`Recall`（召回）、`Explosion`（爆炸）、`PlacementValidation`（落点校验）。
  - 玩家控制 `Capabilities/Player/`：`Aim`（瞄准）、`Fire`（发射，`CapabilityTags = DarkPortal + DarkPortalFire`）、`Grab`、`Recall`、`Animation`、`Companion`（生成暗黑伙伴 `SpawnActor(CompanionClass, ..., bDeferredSpawn) + MakeNetworked`）。
- **响应与表现** `Responses/`：`FauxPhysicsReactionComponent`（伪物理反应，被拉扯物的物理表现）、`FocusEffectComponent`（聚焦特效）、`DebugComponent`。事件 `EventHandlers/`（Portal + Player）。
- **教学** `Tutorial/`：瞄准 / 发射 / 抓取 / 推动四段 + `TutorialComponent`。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| 依赖 | Core 能力四件套 | Capability/Component/Settings/Statics | 标准四件套 + `Data` 组织 |
| 依赖 | Core 瞄准 | 自动瞄准 | `DarkPortalAutoAimComponent` + `Player/DarkPortalPlayerAimCapability` |
| 生成 | 传送门实体 | UserComponent | `UDarkPortalUserComponent` 持 `DarkPortalClass` 生成 |
| 生成 | 暗黑伙伴 | 联机生成 | `DarkPortalPlayerCompanionCapabilty` → `SpawnActor(CompanionClass, bDeferredSpawn) + MakeNetworked(Player, n"DarkPortalCompanion")` |
| → | Sanctuary/AI | 暗黑伙伴触手 | 伙伴实现见 `AI/DarkPortalCompanion/`（Behaviour + Tentacles） |
| → | 场景物体 | 抓取/拉扯 | `DarkPortalResponseComponent` + `FauxPhysicsReactionComponent` 处理被拉物的响应与伪物理 |
| 状态机 | 传送门 | Absorb/Launch/Settle/Recall | `EDarkPortalState` 驱动抓取→发射→附着→召回全流程 |

## 关键文件

- `LevelSpecific/Sanctuary/DarkPortal/DarkPortalActor.as` — 传送门实体 + `EDarkPortalState` 状态机 + `FDarkPortalTargetData`
- `LevelSpecific/Sanctuary/DarkPortal/DarkPortalUserComponent.as` — 玩家中枢组件
- `LevelSpecific/Sanctuary/DarkPortal/Capabilities/Player/DarkPortalPlayerFireCapability.as` — 发射
- `LevelSpecific/Sanctuary/DarkPortal/Capabilities/DarkPortalPullCapability.as` — 拉扯
- `LevelSpecific/Sanctuary/DarkPortal/Capabilities/Player/DarkPortalPlayerCompanionCapability.as` — 暗黑伙伴生成
- `LevelSpecific/Sanctuary/DarkPortal/Responses/DarkPortalFauxPhysicsReactionComponent.as` — 被拉物伪物理响应
- `LevelSpecific/Sanctuary/DarkPortal/DarkPortalAutoAimComponent.as` — 自动瞄准

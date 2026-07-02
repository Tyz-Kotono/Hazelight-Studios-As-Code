# Core / InteractionAndAiming / Targetable

> 职责：整个功能域的地基——把场景中“可被瞄准/交互的点”按分类注册到玩家身上，每帧执行查询、评分、选出每分类的 primary 目标，并驱动浮窗(Widget)与描边(Outline)。共 8 个文件。

## 内部架构

### 关键类与基类
- **`UTargetableComponent : USceneComponent`**（TargetableComponent.as）：所有可瞄准目标的基类。
  - `TargetableCategory : FName`——目标分类，**每个分类同一时刻只能有一个 primary**（如 `n"Interaction"`、`n"AutoAim"`）。
  - `UsableByPlayers : EHazeSelectPlayer`——区分 Mio/Zoe/Both。
  - `CheckTargetable(FTargetableQuery&) const`——**核心评分钩子**，子类必须重写，返回是否参与本次查询；通过 `Targetable::` 辅助函数填充 `Query.Result`。
  - 注册流：`BeginPlay`/`Enable`/`SetTargetableConsidered` → `UpdateRegistration` → `UPlayerTargetablesComponent::GetOrCreate(Player).RegisterTargetable(Category, this)`。
  - 禁用按 instigator 计数（`DisableInstigators`），按玩家分别管理（`TPerPlayer<FTargetableTriggerPlayerData>`）。
- **`UPlayerTargetablesComponent : UHazeBaseTargetSystemComponent`**（PlayerTargetablesComponent.as）：挂在玩家上的查询中枢。
  - 按分类持有 `FTargetableCategoryData`（候选列表 + 上帧查询缓存 `PrevQuery` + 该分类 AimRay）。
  - `GetQueryThisFrame(Category)`：每帧每分类只跑一次——按距离排序候选 → 逐个 `CheckTargetable` → 用 `FilterScore`/`Score` 选出 primary（含 1e-12 偏向上帧 primary 防抖）。
  - `CurrentAimingRay` / `OverrideTargetableAimRay(Category, Ray)`：外部（Aiming）写入瞄准射线供 `CheckTargetable` 使用。
  - `ShowWidgetsForTargetables(FTargetableWidgetSettings)`、`ShowOutlinesForTargetables(FTargetableOutlineSettings)`：把查询结果渲染成浮窗/描边。
  - 查询 API：`GetPrimaryTargetForCategory`、`Internal_GetPrimaryTarget(Class)`、`GetVisible/Possible/RegisteredTargetables`、`GetMostVisibleTargetAndResult`。

### 关键数据结构（TargetableData.as）
- `FTargetableResult`：`Score`/`bPossibleTarget`/`bVisible`/`VisualProgress`/`FilterScore`/`FilterScoreThreshold`——评分与可见性结果。
- `EPlayerTargetingMode`：ThirdPerson / SideScroller / TopDown / MovingTowardsCamera，影响 2D/3D 评分分支。
- `FTargetableQuery`（在 TargetableComponent.as）：单次评分的全部上下文（玩家视角、输入、AimRay、当前 primary 等）。

### 评分/筛选工具库（TargetableHelpers.as，`namespace Targetable`）
距离评分 `ApplyDistanceToScore`/`ApplyTargetableRange(WithBuffer)`/`ApplyVisibleRange`；屏幕评分 `ScoreCameraTargetingInteraction`/`ScoreLookAtAim`/`ScoreWantedMovementInput`/`Score2DTargeting`/`Score2DInteractionTargeting`；遮挡/可达 trace `RequireNotOccludedFromCamera`/`RequireAimNotOccluded`/`RequireAimToPointNotOccluded`/`RequireSweepUnblocked`/`RequirePlayerCanReachUnblocked`/`RequireUnobstructedLineFromPlayer`；标签 `RequireCapabilityTagNotBlocked`；可见进度 `ApplyVisualProgressFromRange`。

### 数据流
注册 → `UpdateAiming/UpdateTargeting` 触发 `GetQueryThisFrame` → 排序候选 → `CheckTargetable` 评分 → 选 primary → 消费方读 primary，渲染方 `ShowWidgets/ShowOutlines`。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 被继承 | `UInteractionComponent`、`UAutoAimTargetComponent` | 子类化基类 | 二者 `: UTargetableComponent`，重写 `CheckTargetable`（InteractionComponent.as:310、AutoAimTargetComponent.as:128） |
| 被调用 | `UInteractionEnterCapability` | 直接调用 | `PlayerTargetablesComp.ShowWidgetsForTargetables(UInteractionComponent,...)` / `GetPrimaryTarget(UInteractionComponent)`（InteractionEnterCapability.as:54、95） |
| 被调用 | `UPlayerAimingComponent` | 直接调用 | 写 `TargetablesComp.CurrentAimingRay`、`OverrideTargetableAimRay(...)`、读 `GetPrimaryTargetForCategory(n"AutoAim")`（PlayerAimingComponent.as:345、371、393） |
| 被调用 | Camera 域（如 `CameraWeightedTargetComponent`、`FocusCameraActor`） | 直接调用 | Camera 子系统亦消费 `UPlayerTargetablesComponent` 做取景（grep Camera/* 命中） |
| 调用出 | `UTargetableWidget` | 直接调用 | `ShowWidgetsForTargetables` 从 `UWidgetPoolComponent` 取池化 widget 并 `UpdateWidget`（PlayerTargetablesComponent.as:774） |
| 调用出 | `UTargetableOutlineComponent` | 直接调用 | `ShowOutlinesForTargetables` 调 `OutlineComp.ShowOutlines(Player, Type)`（PlayerTargetablesComponent.as:969） |
| 调用出 | 玩家能力标签 | 标签阻塞 | `Targetable::RequireCapabilityTagNotBlocked` → `Player.IsCapabilityTagBlocked`（TargetableHelpers.as:822） |
| 双人 | Mio / Zoe | 可组合设置 | 每个 Targetable 按 `UsableByPlayers` 注册；`TPerPlayer` 数据 + `Game::Mio/Zoe` 分屏处理 |

## 关键文件

| 文件 | 说明 |
|---|---|
| TargetableComponent.as | 基类 `UTargetableComponent` + `FTargetableQuery` + 注册/禁用逻辑 |
| PlayerTargetablesComponent.as | 玩家级查询中枢；每帧评分选 primary，驱动 Widget/Outline |
| TargetableData.as | `FTargetableResult`、`EPlayerTargetingMode` 等结果数据结构 |
| TargetableHelpers.as | `namespace Targetable` 评分/遮挡/范围/标签辅助函数库 |
| TargetableStatics.as | `GetCurrentTargetingMode` 等便捷 mixin |
| TargetableWidget.as | 目标浮窗基类 `UTargetableWidget : UPooledWidget` + 另一玩家状态枚举 |
| TargetableOutlineComponent.as | 描边组件 `UTargetableOutlineComponent` + `UTargetableOutlineDataAsset` 数据资产 |
| TargetableTemporalLog.as | 编辑器 Temporal Log 调试扩展（仅 EDITOR） |

# Core / SplineAndSpace / Spline

> 职责：本功能域的核心。提供完整的样条系统——样条数据建模、曲线计算、丰富的位置/变换查询 API、跨样条连接、可沿样条移动的逻辑位置、网络同步以及"沿样条放置物件"的索引。共 15 个文件（含 AlongSpline 子目录 3 个）。

## 内部架构（关键类/基类/数据结构/数据流）

样条系统分四层：**数据建模 → 曲线计算 → 运行时查询 → 上层应用**。

```
[编辑器/上层]
   │ 编辑 SplinePoints (FHazeSplinePoint 数组)
   ▼
UHazeSplineComponent (: UHazeSplineBaseComponent)
   │ UpdateSpline() → SplineComputation::ComputeSpline()
   ▼
FHazeComputedSpline  ← 烘焙结果：Points/Segments/采样表(Samples_*)/Bounds/SplineLength
   │ 查询时按 距离/比例/最近点 检索
   ▼
查询 API: GetWorldTransformAtSplineDistance / GetClosestSplineDistanceToWorldLocation ...
   │
   ├─ FSplinePosition         ← 样条上"可移动的逻辑点"，自动穿越连接与闭环
   ├─ FHazeRuntimeSpline      ← 由采样烘出的轻量折线，供调试/简易场景使用
   └─ UAlongSpline*           ← 沿样条放置的静态物件索引（二分查询）
```

- **`UHazeSplineComponent`**（`HazeSplineComponent.as`）：核心组件。持有可编辑的 `TArray<FHazeSplinePoint> SplinePoints` 与烘焙后的 `FHazeComputedSpline ComputedSpline`。`UpdateSpline()` 调用 `SplineComputation::ComputeSpline` 重算曲线；提供约 30 个查询函数（按 `SplineDistance`、按 `Fraction[0..1]`、按"最近点"三类，覆盖 World/Relative 空间下的 Location/Rotation/Scale/Transform/Tangent）。`bClosedLoop` 闭环；`SplineConnections`（`TArray<FSplineConnection>`）记录与其它样条的运行时连接，`bSpecifyConnections` + `FSpecifiedSplineConnection` 在编辑器中配置，`BeginPlay` 时应用并可生成互反连接。`GetAllLinkedSplines()` 广搜全部连通样条。
- **`SplineComputation.as`**：`namespace SplineComputation_AS`，纯函数曲线库（与 C++ 版 `HazeSplineComputationStatics.h` 双份维护，C++ 为默认，AS 版供快速迭代）。包含三次 Hermite 插值、Legendre-Gauss 弧长积分、二分查"距离→段Alpha"、牛顿法求"最近点/最近线段/平面或轴约束下的最近点"。
- **`SplinePosition.as`**：`FSplinePosition` 结构——样条上的一个可移动点（持 `SplineComponent`+`SplineDistance`+`bForwardOnSpline`）。`Move(Distance)` 沿样条前进并自动穿越连接/环；`Distance`/`DeltaToReachClosest`/`CanReach`/`Lerp`/`IsBetweenPositions` 在连接图上做广度搜索（`FSplineRange`），带极性 `ESplineMovementPolarity`。
- **`SplineActor.as`**：`ASplineActor` 把 `UHazeSplineComponent` 封装成可摆放的最简 Actor。
- **网络同步**：`HazeCrumbSyncedSplinePositionComponent.as`（`UHazeCrumbSyncedSplinePositionComponent`）按 crumb 插值同步 `FSplinePosition`；`CrumbSplineHelper.as`（`UCrumbSplineHelper`）为 `UHazeCrumbSyncedActorPositionComponent` 提供样条相对位置的 lerp 与回退逻辑。
- **轻量采样**：`RuntimeSplineStatics.as` 把 `UHazeSplineComponent`/`FHazeComputedSpline` 烘成 `FHazeRuntimeSpline`（`BuildRuntimeSplineFrom*`）并提供大量调试/可视化 mixin。
- **复用与调度**：`SplinesContainer.as`（`FSplinesContainer` + `SplineContainerStatics`）管理一组样条的"轮用/冷却/随机/在视野内优先"挑选；`SplineUsageComponent.as`（`USplineUsageComponent`）记录每条样条最后被谁、何时使用。
- **安全获取**：`GameplaySplineCheck.as`（`namespace Spline`）的 `GetGameplaySpline(Actor)` 统一获取用于 gameplay 的样条组件，并对"editor-only 样条被用于 gameplay"报错。
- **位置查询 mixin**：`SplinePositionMixins.as` 在 `UHazeSplineComponent` 上挂一组 `GetClosest*SplinePosition*` mixin（点/线段、轴约束、平面约束，可跨连接搜索），返回 `FSplinePosition`。
- **编辑器对齐**：`ActorAlignedToSpline.as`（`AActorAlignedToSpline`）仅编辑器内把 Actor 吸附到最近样条点。
- **AlongSpline 子目录**：见下"沿样条物件索引"。

### 沿样条物件索引（AlongSpline）

- **`UAlongSplineComponent`**（抽象基类）：放在样条 Actor 上、代表"样条上某一点的物件"，`SnapToSpline` 把自己吸附到样条最近变换。
- **`UAlongSplineComponentManager`**：按需自动创建，维护 `TMap<UClass, 按距离排序的数组>`。`FindClosest/Previous/Next/Adjacent/InRange` 用**二分查找**在样条距离上检索；闭环样条通过在首尾插入"假点"来处理环绕。`FAlongSplineComponentData` 携带 `DistanceAlongSpline` 与用于排序的 `SortDistanceAlongSpline`。
- **`AlongSplineMixins.as`**：在 `UHazeSplineComponent` 上暴露 `FindClosestComponentAlongSpline` 等 mixin，内部转发到 Manager（机制 5/1）。

## 协同关系（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 被调用 | `Camera/CameraTypes/CameraTypeSplineFollow`、`SplineFollowCameraActor` 等相机 | 直接调用 `Type::Get` | 相机域大量消费 `UHazeSplineComponent` 做样条跟随（grep `UHazeSplineComponent` 命中 39 文件，相机域占多数） |
| 被调用 | `Movement/SplineLock/*` | 直接调用 + `FSplinePosition` | 运动域把角色约束到样条平面，使用 `FSplinePosition`（grep `FSplinePosition` 命中 SplineLockComponent/Statics/ResolverExtension） |
| 被调用 | `Aiming/PlayerAimingComponent`、`Props/PropLineProgression`、`Audio/AudioStatics` | `FSplinePosition` | 瞄准、道具线进度、音频沿样条定位均消费 `FSplinePosition` |
| 被调用 | `Props/PropLine`、`PropLineMeshGeneration` | 直接调用 | 道具线沿样条生成网格 |
| 被调用 | `RespawnPoint/RespawnOnSplineNearOtherPlayerVolume` | 直接调用 + 最近点 | 在样条上靠近另一玩家处重生 |
| 调用 | `Spline::GetGameplaySpline` 消费方 | 安全获取 | `AlongSplineComponent*`、`PropLineProgression`、`RespawnOn...` 等通过它取样条（grep `GetGameplaySpline` 命中 10 文件） |
| 依赖 | `Volumes/AutoScaleSplineBoxComponent` | 直接调用 | 盒体读取样条点贴合包围（见 Volumes 模块） |
| 依赖 | `SceneView::IsInView`（Mio/Zoe） | 直接调用 | `SplinesContainer` 选择"在任一玩家视野内"的样条（双人 Mio/Zoe 实证） |

## 关键文件（逐文件一句话）

- `HazeSplineComponent.as`：核心样条组件，持点与烘焙曲线，提供按距离/比例/最近点的全套查询及跨样条连接。
- `SplineComputation.as`：纯函数曲线计算库（三次插值/弧长/最近点牛顿法），与 C++ 版双份维护。
- `SplinePosition.as`：`FSplinePosition`——可沿样条移动并自动穿连接/环的逻辑位置，含连接图搜索。
- `SplinePositionMixins.as`：在样条组件上获取"最近样条位置"的 mixin（点/线段/轴/平面约束、可跨连接）。
- `SplineActor.as`：把样条组件封装为可摆放 Actor 的最简类。
- `HazeCrumbSyncedSplinePositionComponent.as`：以 crumb 插值同步 `FSplinePosition` 的网络组件。
- `CrumbSplineHelper.as`：为同步的 Actor 相对位置提供样条空间 lerp 与回退。
- `RuntimeSplineStatics.as`：把样条烘成轻量 `FHazeRuntimeSpline` 并提供调试/可视化 mixin。
- `SplinesContainer.as`：一组样条的轮用/冷却/随机/视野内优先挑选。
- `SplineUsageComponent.as`：记录每条样条最后使用者与时间。
- `GameplaySplineCheck.as`：`Spline::GetGameplaySpline` 安全获取 gameplay 样条并报错 editor-only 误用。
- `ActorAlignedToSpline.as`：编辑器内把 Actor 吸附到最近样条点（运行时无效）。
- `AlongSpline/AlongSplineComponent.as`：沿样条放置物件的抽象组件，自动吸附样条。
- `AlongSpline/AlongSplineComponentManager.as`：按类型分组、按距离排序、二分查询沿样条物件的管理器。
- `AlongSpline/AlongSplineMixins.as`：在样条组件上转发到 Manager 的查询 mixin。

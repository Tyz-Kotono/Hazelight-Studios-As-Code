# Core / SplineAndSpace 功能域总览

> 职责：Split Fiction 中所有"样条与空间查询"相关能力的集合——样条建模与查询、碰撞开关与空间索引、导航寻路、样条贴合体积、动态附着、几何形状检测与按距离排序。共 7 个子模块、27 个文件。

> 注：本功能域的 7 个子模块在源码树中是 `Core/` 下平级的目录（`Core/Spline`、`Core/Collision`、`Core/Navigation`、`Core/Volumes`、`Core/Attachment`、`Core/Shapes`、`Core/Sorting`），在功能上同属"样条与空间"域，故归并到本 `SplineAndSpace` 文档之下。

## 子模块一览

| 子模块 | 路径 | 文件数 | 定位 |
| --- | --- | --- | --- |
| [Spline](./Spline/README.md) | `Core/Spline` | 15 | 本域核心：样条数据/计算/查询/连接、可移动逻辑位置、网络同步、沿样条物件索引（含 AlongSpline 子目录 3 个） |
| [Collision](./Collision/README.md) | `Core/Collision` | 5 | 玩家碰撞开关、形状/射线查询封装、烘焙稀疏球网格、海洋波数据接入 |
| [Navigation](./Navigation/README.md) | `Core/Navigation` | 2 | 基于 navmesh 的可达性/最远直线点查询，及"穿越体积"占位骨架 |
| [Volumes](./Volumes/README.md) | `Core/Volumes` | 1 | 盒体自动贴合所在样条的包围范围 |
| [Attachment](./Attachment/README.md) | `Core/Attachment` | 2 | 编辑器友好的"组件/Actor 动态附着到目标组件"辅助 |
| [Shapes](./Shapes/README.md) | `Core/Shapes` | 1 | 水滴/胶囊扫掠形的含点判定与调试绘制 |
| [Sorting](./Sorting/README.md) | `Core/Sorting` | 1 | 按到某点距离对 Actor 数组排序 |

## 域内主线：样条框架

`Spline` 是本域唯一成体系的大子系统，其余 6 个子模块要么消费样条（`Volumes`），要么是与"空间"相关的独立工具集（`Collision`/`Navigation`/`Shapes`/`Sorting`/`Attachment`）。样条分四层：

```
编辑 FHazeSplinePoint[]  →  UHazeSplineComponent.UpdateSpline()
                                     │ SplineComputation::ComputeSpline
                                     ▼
                           FHazeComputedSpline（烘焙曲线 + 采样表）
                                     │ 按 距离/比例/最近点 查询
        ┌────────────────────────────┼────────────────────────────┐
        ▼                            ▼                            ▼
  FSplinePosition            FHazeRuntimeSpline           UAlongSpline*（索引）
（可移动、穿连接/环）        （轻量折线，调试/简易）        （沿样条物件二分查询）
```

- 核心组件 `UHazeSplineComponent`（: `UHazeSplineBaseComponent`）持原始点 `SplinePoints` 与烘焙结果 `ComputedSpline`，暴露约 30 个 World/Relative 空间下的查询 API；支持闭环与跨样条连接 `FSplineConnection`。
- `FSplinePosition` 是关键运行时抽象——样条上一个可 `Move()` 的逻辑点，能自动穿越连接与闭环，并在连接图上做可达性/距离/lerp 搜索。
- 网络化由 `UHazeCrumbSyncedSplinePositionComponent`/`UCrumbSplineHelper` 经 crumb 同步。
- AlongSpline 子目录是"沿样条放置的**静态物件索引**"（不是沿样条移动），用二分查找按距离检索。

## 本域约定的 5 种协作机制

1. **直接调用** `Type::Get`：取得组件/Actor 后调用其方法（如 `UHazeSplineComponent::Get`、`UAlongSplineComponentManager::Get`、`UHazeMovementComponent::Get`）。
2. **Crumb**：网络化同步样条相对位置（`UHazeCrumbSyncedSplinePositionComponent`）。
3. **标签阻塞** `CapabilityTags`：如 `UPlayerCollisionCapability` 用 `CapabilityTags::Collision` 并在被阻塞时切换碰撞 profile。
4. **委托 / 异步请求**：以 `FInstigator` 为键的请求-回读（`OceanWaves` 波数据）。
5. **可组合设置 / mixin**：在 `UHazeSplineComponent` 上挂查询 mixin（`SplinePositionMixins`、`AlongSplineMixins`）、编辑器细节面板下拉配置（`Attachment`）。

双人协作（Mio / Zoe）：`SplinesContainer` 在挑选样条时优先选"在任一玩家视野内"的（`SceneView::IsInView(Zoe/Mio)`）；`UPlayerCollisionCapability` 在屏蔽碰撞时主动跳过另一玩家；`Trace::IgnorePlayers` 可一键忽略 `Game::Mio`/`Game::Zoe`。

## 跨域消费（grep 实证摘要）

- `UHazeSplineComponent` 被 39 个文件引用，集中在 **Camera**（样条跟随相机）、**Movement/SplineLock**（样条约束）、**Props/PropLine**（沿样条生成）、**Audio**、**RespawnPoint** 等域。
- `FSplinePosition` 被 11 个文件引用（瞄准、样条锁、道具线进度、音频等）。
- `FStaticSparseSphereGrid`（Collision）被 10 个文件消费做快速球体重叠。
- `Overlap::`/`Trace::`（Collision）被 PlayerHealth、Targetable 等域复用。

## 文件计数说明

子模块文件数合计 27（Spline 15 + Collision 5 + Navigation 2 + Volumes 1 + Attachment 2 + Shapes 1 + Sorting 1），与背景给定一致；其中 Spline 的 15 个含 AlongSpline 子目录的 3 个文件。

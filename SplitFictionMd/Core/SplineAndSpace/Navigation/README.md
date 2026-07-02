# Core / SplineAndSpace / Navigation

> 职责：基于 UE 导航网格（Navmesh）的寻路查询工具，以及一个尚属占位的"穿越体积"框架。共 2 个文件。

## 内部架构（关键类/基类/数据结构/数据流）

- **`PathfindingStatics.as`**（`namespace Pathfinding` + 调试 mixin）：一组围绕 `UNavigationSystemV1` 的纯函数查询库。
  - 基础判定：`IsPathNear`（带高度容差的近邻判断）、`IsNearNavmesh`/`FindNavmeshLocation`（`ProjectPointToNavigation` 先垂直后扩大水平搜索）。
  - 连通性：`StraightPathExists`（用路径漏斗化后是否仅 2 个节点判断"直线可达"）、`HasPath`（带容差的可达性）。
  - 漏斗算法：`FindLongestStraightPath`/`FunnelLongestStraightPath`/`GetFunnelMovedToEdge` 在 `FHazeNavmeshPoly`/`FHazeNavmeshEdge` 上做深度优先 + 漏斗收缩，求"从某点出发、不绕角能走到的最远直线点"，配合 `Math::GetLineSegmentSphereIntersectionPoints` 等几何工具裁剪到最大距离。
  - 调试：`DrawDebugNavmeshPoly` mixin 画出多边形边。
  - 数据流：调用方传入世界坐标 → 投影到 navmesh → 同步寻路/漏斗搜索 → 返回布尔或目标点。
- **`TraversalVolume.as`**：`ATraversalVolume : AVolume` 持 `UTraversalBuilderComponent`，目前 builder 组件体几乎为空（仅持 `TraversalVolume` 引用），是一个待完善的"可穿越区域标注"骨架。

## 协同关系（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 调用 | 引擎 `UNavigationSystemV1` | 直接调用 | `ProjectPointToNavigation`/`FindPathToLocationSynchronously` 是全部查询的底座 |
| 调用 | 引擎 `Navigation::FindNearestPoly` 与 `FHazeNavmeshPoly`/`FHazeNavmeshEdge` | 直接调用 | 漏斗算法遍历 navmesh 多边形与边 |
| 调用 | `Math::`（线段/球/平面相交、归一化） | 直接调用 | 漏斗裁剪与近邻判定的几何运算 |
| 提供 | AI / 敌人 / 程序化移动调用方 | 静态函数库（机制 1） | `Pathfinding::` 命名空间供上层做可达性与"最远直线点"查询（本功能域内仅自身定义，消费者在 Core 之外的 gameplay 层） |
| 关联 | `ATraversalVolume` ↔ `UTraversalBuilderComponent` | 默认组件直连 | `default BuilderComp.TraversalVolume = this` 自指 |

## 关键文件（逐文件一句话）

- `PathfindingStatics.as`：`Pathfinding::` 导航查询库——近邻/投影/可达性/直线漏斗最远点，外加 navmesh 调试绘制。
- `TraversalVolume.as`：`ATraversalVolume` + `UTraversalBuilderComponent` 的"可穿越区域"占位骨架。

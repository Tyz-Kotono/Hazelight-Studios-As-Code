# Core / SplineAndSpace / Sorting

> 职责：按到某世界坐标的距离对 Actor 数组排序的便捷封装。共 1 个文件。

## 内部架构（关键类/基类/数据结构/数据流）

- **`SortingStatics.as`**（`namespace Sort`）：`SortByDistanceToLocation(Location, InOutActors)`——把传入的 `TArray<AHazeActor>` 原地按到 `Location` 的距离升序排列。实现是把每个 Actor 的 `RootComponent` 包成 `FHazeComponentSortElement`，转交给引擎/上层提供的 `Sort::SortByDistanceToPoint`（组件级排序），再把排序结果写回 Actor 数组。
- 数据流：Actor 数组 → 取 RootComponent → 组件排序 → 回填 Actor 数组。纯无状态工具，是组件级排序到 Actor 级的薄适配层。

## 协同关系（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 调用 | `Sort::SortByDistanceToPoint`（组件级排序，定义在本域之外） | 直接调用（机制 1） | 本函数仅做 Actor↔组件适配并复用既有组件排序 |
| 调用 | `AHazeActor.RootComponent` / `Cast<AHazeActor>` | 直接调用 | 取根组件参与排序、再还原为 Actor |
| 提供 | 需要"按距离排 Actor"的 gameplay 调用方 | 静态函数库 | `Sort::SortByDistanceToLocation` 为通用工具（消费者在 Core 之外） |

## 关键文件（逐文件一句话）

- `SortingStatics.as`：`Sort::SortByDistanceToLocation`——把 Actor 数组原地按到指定点的距离排序（适配到组件级排序）。

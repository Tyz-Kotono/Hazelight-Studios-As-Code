# Core / SplineAndSpace / Volumes

> 职责：让盒体碰撞组件自动贴合其所在样条的包围范围。共 1 个文件。

## 内部架构（关键类/基类/数据结构/数据流）

- **`AutoScaleSplineBoxComponent.as`**（`UAutoScaleSplineBoxComponent : UBoxComponent`）：放在带样条的 Actor 上，`ConstructionScript` 中通过 `UHazeSplineComponent::Get(Owner)` 取得样条，随后：
  1. `UpdateBoxLocation()`：以 `IterationDistance`（默认 50）为步长遍历样条全长，调用 `Spline.GetRelativeLocationAtSplineDistance` 采样，求出样条点的相对空间最低/最高角（`Lowest`/`Highest`），把盒体放到二者中点。
  2. `UpdateBoxExtents()`：由 `Lowest`/`Highest` 加上 `BoxMargin`（默认 200）算出盒体 extent，`SetBoxExtent`。
- 数据流：样条点采样 → AABB（相对空间）→ 盒体居中并扩边。属于"编辑期/构造期一次性贴合"，无运行时逻辑。

## 协同关系（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 调用 | `Spline` 域 `UHazeSplineComponent` | 直接调用 `Type::Get` | `ConstructionScript` 取 Owner 上样条并采样其相对位置 |
| 调用 | `UHazeSplineComponent::GetSplineLength` / `GetRelativeLocationAtSplineDistance` | 直接调用 | 按步长遍历样条求包围 |
| 继承 | 引擎 `UBoxComponent` | 基类 | 复用盒体碰撞与 `SetBoxExtent` |

## 关键文件（逐文件一句话）

- `AutoScaleSplineBoxComponent.as`：构造期采样所在样条、把 `UBoxComponent` 自动居中并缩放到包住整条样条（含边距）。

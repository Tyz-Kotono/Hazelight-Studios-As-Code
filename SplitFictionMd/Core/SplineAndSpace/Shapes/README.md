# Core / SplineAndSpace / Shapes

> 职责：自定义几何形状的相交检测与调试绘制（目前为"水滴形/胶囊扫掠形"）。共 1 个文件。

## 内部架构（关键类/基类/数据结构/数据流）

- **`ShapeStatics.as`**：一组几何 mixin 与调试命名空间。
  - `IsInsideTeardrop(Location, Start, End, StartRadius, EndRadius)`：判断点是否落在"包住两端两个球"的水滴/胶囊扫掠形内——把点投影到 `Start→End` 线段得到比例 `Fraction`，按比例 lerp 半径，再比较点到线段的距离。
  - `IsInsideTeardrop2D`：把 Z 拍平到点的高度后做同样判断（忽略垂直差异）。
  - `namespace ShapeDebug::DrawTeardrop`：绘制水滴形轮廓（两端半圆 + 切线连接，处理一端被另一端包含的退化情形），用三角函数沿弧采样连线。
- 数据流：传入位置与两端球（中心+半径）→ 线段投影 + 半径插值 → 布尔；调试函数独立绘制。纯无状态工具。

## 协同关系（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 调用 | `Math::ProjectPositionOnLineSegment` / `Math::Lerp` / `Math::SinCos` 等 | 直接调用 | 投影、半径插值与弧线采样 |
| 调用 | `Debug::DrawDebugLine` / `DrawDebugCircle` | 直接调用 | `ShapeDebug` 轮廓绘制 |
| 被调用 | `Collision/OverlapStatics`（`FHazeShapeSettings::Make*`） | 直接调用 | 注：`OverlapStatics` 使用引擎 `FHazeShapeSettings`；本文件的 `IsInsideTeardrop` 作为独立几何判定供 gameplay 层（Core 之外）使用 |

## 关键文件（逐文件一句话）

- `ShapeStatics.as`：水滴/胶囊扫掠形的"含点"判定（含 2D 变体）与调试轮廓绘制。

# Core / MathAndUtilities / Visualization

> 职责：占位（dummy）调试可视化组件——以数据形式描述要绘制的球、柱与连线图元。共 1 文件。

## 内部架构

`DummyVisualizationComponent.as` 是一个**纯数据容器组件**：只声明要绘制什么，不含绘制逻辑（绘制由配套可视化器/构造脚本读取这些数据完成）。

### 关键单元

| 单元 | 基类 | 角色 |
| --- | --- | --- |
| `UDummyVisualizationComponent` | `UActorComponent` | 持有待绘制图元列表与连线配置 |
| `FDummyVisualizationSphere` | struct | 球：中心 `USceneComponent`、半径、粗细、分段数 |
| `FDummyVisualizationCylinder` | struct | 柱：中心组件、半径、由上下高度算出的 `HalfHeight` 与 `Offset`、粗细、分段数 |

### 关键数据

- `ConnectedActors` / `ConnectedLocalLocations`：连线目标与本地坐标点（注释提示需在构造脚本中更新）。
- `ConnectionBase` / `ConnectionBaseOffset`：连线起点基准。
- `DashSize` / `Thickness` / `Color`：连线样式。
- `Spheres` / `Cylinders`：要绘制的球与柱数组。

`FDummyVisualizationCylinder` 构造函数把传入的上/下高度换算为半高与中心偏移，便于绘制时直接使用。

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 被读取 | 配套可视化器 / 构造脚本 | 数据驱动 | 本组件仅存储图元描述，实际绘制由读取这些字段的一方完成 |
| 概念相邻 | `FloatCurves` 调试绘制 | 同属可视化 | 二者都服务于编辑器期调试，但本组件描述几何图元而非曲线 |

## 关键文件

- `DummyVisualizationComponent.as` — `UDummyVisualizationComponent` 调试图元数据容器，配套 `FDummyVisualizationSphere` / `FDummyVisualizationCylinder` 结构描述球与柱，并存连线目标与样式。

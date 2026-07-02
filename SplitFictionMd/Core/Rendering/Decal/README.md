# Core / Rendering / Decal

> 职责：随组件（或样条）移动时连续生成拖尾贴花，并对队列中的贴花做沿轨迹衰减与统一淡出。共 1 个文件。

## 内部架构

- **`UDecalTrailComponent : USceneComponent`**：
  - 配置：贴花高/宽/长间隔（`DecalWidth`/`DecalHeight`/`DecalLength`）、重叠、材质、`MaxDecals` 上限、`DecalLifetime`。
  - 队列：`SpawnedDecals` / `SpawnedDecalIndexes` / `SpawnedDecalFadeValue` 三个并行数组维护已生成贴花及其索引、淡出值。
  - 生成：`Tick` 中按当前位置与上次锚点 `SnapLocation` 的距离每超过 `DecalLength` 就 `Decal::SpawnDecalAtLocation` 新建一片，超过上限淘汰队首；每帧调整当前贴花的位置/朝向/缩放/颜色 A 通道。
  - 衰减：`DecayOverTrail` 把贴花颜色 RGB 编码成「索引 + 沿轨迹归一化进度 + 淡出值」供材质读取；`ClearNew`/`GlobalFade` 实现整条拖尾的统一淡出销毁。
  - 可选 `UHazeSplineComponent` 沿样条取位置（`GetTrailLocation`）。

数据流：`移动 → Tick 生成/淘汰贴花 → SetDecalColor 编码轨迹数据 → 贴花材质渲染`。

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 调用 | 引擎贴花 | 直接调用 | `Decal::SpawnDecalAtLocation` + `CreateDynamicMaterialInstance` |
| 可选 | 样条 | 直接调用 | `UHazeSplineComponent.GetWorldLocationAtSplineDistance` 沿样条取轨迹点 |

## 关键文件

- **DecalTrail.as**：`UDecalTrailComponent`，移动拖尾贴花生成 + 沿轨迹衰减 + 统一淡出。

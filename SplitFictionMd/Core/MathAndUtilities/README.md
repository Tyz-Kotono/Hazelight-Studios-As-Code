# Core / MathAndUtilities

> 职责：游戏通用的数学与工具层——向量/旋转钳制、缓动结构、弹道与加速度积分、标准曲线资产、运行时曲线调试绘制、占位调试图元。共 8 文件，分布于 4 个子模块。

## 概览

本功能域几乎全部是**无状态的纯计算**或**纯数据容器**，是上层系统（相机、移动、瞄准、导航等）的底层依赖，自身极少反向依赖其他系统。主要分为四类：

| 子模块 | 文件数 | 形态 | 一句话 |
| --- | --- | --- | --- |
| `Math/` | 5 | mixin / namespace / struct | 向量数学、缓动、弹道、加速度、Alpha 重映射 |
| `Curves/` | 1 | `asset of UCurveFloat` | 标准命名曲线资产（平滑/线性/缓入） |
| `FloatCurves/` | 1 | struct + mixin | `FRuntimeFloatCurve` 的运行时调试绘制 |
| `Visualization/` | 1 | `UActorComponent` + struct | 占位调试图元（球/柱/连线）数据容器 |

## 形态约定

- **mixin 函数**：以 `mixin` 修饰的全局函数，挂在第一个参数类型上调用（如 `FVector::ClampInsideCone`），多带 `UFUNCTION(BlueprintPure)` 暴露给蓝图。
- **namespace 函数库**：`Trajectory::`、`Acceleration::`、`AlphaStatics::`、`Curve::`、`FloatCurve::` 等纯命名空间，集中同类静态函数。
- **缓动结构**：`FHazeEasedFloat / Vector / Quat` 为值类型，调用方持有并每帧 `EaseTo` 推进、`GetValue` 取值。

## 跨域协同（实证）

| 方向 | 对象 | 机制 | 说明（grep 实证） |
| --- | --- | --- | --- |
| 被调用 | `Math/MathStatics.as` | 直接调用（mixin） | `Shapes/ShapeStatics.as`、`Navigation/PathfindingStatics.as`、`Camera/HideOverlappers/CameraHideOverlappersCapability.as` 调用 `ClampInsideCone` / `IsWithinDist` 等 |
| 被调用 | `Math/HazeEased.as` | 直接调用 | `Camera/Blend/CameraDefaultBlend.as`、`CameraOrbitBlend.as`、`CameraBlendStatics.as` 用 `FHazeEased*` 做视点混合缓动 |
| 被调用 | `FloatCurves/FloatCurveDebugStatics.as` | mixin + `UHazeScriptComponentVisualizer` | 由各编辑器组件可视化器在世界中绘制 `FRuntimeFloatCurve` |

> 注：`Trajectory::`、`Acceleration::`、`Curve::`、`Visualization` 在本目录内自洽，主要被项目其他模块（如投射物、移动、UI 曲线编辑）按需引用。

## 子模块

- [`Math/`](Math/README.md) — 向量数学、缓动、弹道、加速度、Alpha
- [`Curves/`](Curves/README.md) — 标准命名曲线资产
- [`FloatCurves/`](FloatCurves/README.md) — 运行时曲线调试绘制
- [`Visualization/`](Visualization/README.md) — 占位调试图元

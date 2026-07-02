# Core / MathAndUtilities / FloatCurves

> 职责：在世界空间中调试绘制 `FRuntimeFloatCurve`，含可配置的边框、采样、范围标注。共 1 文件。

## 内部架构

`FloatCurveDebugStatics.as` 提供一个空的 `FloatCurve` 命名空间占位、一个绘制参数结构、以及核心绘制 mixin。

### 关键单元

| 单元 | 形态 | 角色 |
| --- | --- | --- |
| `FRuntimeFloatCurveDrawParams` | struct | 绘制配置：曲线/边框颜色与粗细、采样步长 `SamplingSteps`、是否用曲线自身值域、标签缩放/偏移、`TimeRange` / `ValueRange` |
| `DrawRuntimeFloatCurve(...)` | mixin（挂 `UHazeScriptComponentVisualizer`） | 把一条 `FRuntimeFloatCurve` 绘制成世界空间折线图 |

### 数据流

1. 由 `BottomLeftLocation + Width/Height + UpVector/RightVector` 确定绘制平面与边框四角，按 `bDrawFrame` 画框。
2. 取值域：`bUseCurveRanges` 为真时调用 `Curve.GetTimeRange / GetValueRange`，否则用 `DrawParams` 指定的范围；`bLabelRanges` 时在四角用 `DrawWorldString` 标注 X/Y 上下限。
3. 以 `SamplingSteps` 为步长在 0→1 区间迭代：`Curve.GetFloatValue(Lerp(MinTime,MaxTime,t))` 取值，经 `Math::NormalizeToRange` 归一到 0→1，映射到平面坐标后用 `Visualizer.DrawLine` 连成折线；可选向底部画竖线。

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 依赖 | `UHazeScriptComponentVisualizer` | mixin 接收者 | 绘制 API（`DrawLine` / `DrawWorldString`）由可视化器提供，故只在编辑器组件可视化语境内调用 |
| 依赖 | 引擎 `FRuntimeFloatCurve` | 数据源 | 读取关键帧数与采样值 |
| 依赖 | `Math::NormalizeToRange` / `Lerp` | 直接调用 | 值域归一与时间插值 |

## 关键文件

- `FloatCurveDebugStatics.as` — `FRuntimeFloatCurveDrawParams` 绘制配置结构 + `DrawRuntimeFloatCurve` mixin，将运行时浮点曲线绘制为世界空间折线图（含边框与范围标签）。

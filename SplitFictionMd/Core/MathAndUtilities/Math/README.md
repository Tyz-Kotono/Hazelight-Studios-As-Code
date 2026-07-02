# Core / MathAndUtilities / Math

> 职责：游戏数学计算核心——向量/旋转钳制与 slerp、缓动结构、弹道解算、加速度积分、Alpha 重映射。共 5 文件。

## 内部架构

全部为**无状态纯函数**或**值类型结构**，无 Actor / Component。按文件组织成五组互相独立的能力：

### 关键单元

| 单元 | 形态 | 角色 |
| --- | --- | --- |
| `MathStatics.as` | mixin（挂在 `FVector` 上） | 向量工具：锥形钳制、绕轴旋转/slerp、距离/范围判定、带兜底归一化 |
| `HazeEased.as` | `enum EEasing` + struct | 缓动状态机：`FHazeEasedFloat / FHazeEasedVector / FHazeEasedQuat` |
| `TrajectoryStatics.as` | `namespace Trajectory` | 抛物线弹道：轨迹采样、命中速度反解、最高点/平面相交、移动目标拦截 |
| `AccelerationStatics.as` | `namespace Acceleration` | 加减速运动积分，输出帧率无关的补偿位移 |
| `AlphaStatics.as` | `namespace AlphaStatics` | Alpha 重映射：线性→锯齿、频率正弦 |

### 关键数据结构与数据流

- **`FHazeEasedFloat`**：私有持有 `StartValue / TargetValue / Duration / Progression / EaseType`。每帧 `EaseTo(...)` 先 `ResetIfNeccessary`（参数变化即重置进度），再累加 `DeltaTime`；`GetValue()` 按 `EEasing` 分派到 `Math::EaseInOut / SinusoidalInOut / Lerp` 等。`FHazeEasedVector / Quat` 内部复用一个 `FHazeEasedFloat` 作为 alpha 源，分别 `Lerp` / `FQuat::Slerp` 端点。
- **`Trajectory::CalculateTrajectory`**：把速度分解为水平/垂直分量（`SplitVectorIntoVerticalHorizontal`），按分辨率（上限 64 步）采样 `TrajectoryFunction`（可选终端速度版本），输出 `FTrajectoryPoints{Positions, Tangents}`。反解函数（`CalculateParamsForPathWithHorizontalSpeed / WithHeight`）用抛物线公式求初速度，返回 `FOutCalculateVelocity{Velocity, Time, MaxHeight}`。
- **`Acceleration::*InterpVelocityConstantTo...`**：在向目标速度恒加速插值的同时，输出一个 `OutVectorToAddDeltaTo` 补偿增量，供调用方额外施加以消除帧率导致的积分误差（注释强调需配合「不重复施加自身速度」的 AddDelta API）。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
| --- | --- | --- | --- |
| 被调用 | `Shapes/ShapeStatics.as` | 直接调用（mixin） | 复用 `MathStatics` 的向量工具 |
| 被调用 | `Navigation/PathfindingStatics.as` | 直接调用 | 复用向量距离/钳制 |
| 被调用 | `Camera/HideOverlappers/CameraHideOverlappersCapability.as` | 直接调用 | 使用 `ClampInsideCone` / `IsWithinDist` 类工具 |
| 被调用 | `Camera/Blend/CameraDefaultBlend.as`、`CameraOrbitBlend.as`、`CameraBlendStatics.as` | 直接调用 | 用 `FHazeEased*` 缓动结构做相机视点混合 |
| 依赖 | 引擎 `Math::` 命名空间 | 直接调用 | `EaseInOut`/`Sqrt`/`Clamp` 等底层数学 |

## 关键文件

- `MathStatics.as` — `FVector` 的 mixin 工具集：锥形钳制 `ClampInsideCone`、绕轴 `RotateVectorTowardsAroundAxis` / `SlerpVectorTowardsAroundAxis`、距离判定 `IsWithinDist(2D)` / `IsWithinRange(2D)`、带兜底归一化 `GetNormalizedWithFallback`。
- `HazeEased.as` — 缓动值结构 `FHazeEasedFloat / Vector / Quat` 与 `EEasing` 枚举，参数变更自动重置进度。
- `TrajectoryStatics.as` — `Trajectory` 命名空间：弹道采样、初速度反解、最高点/平面相交、移动目标拦截时间，及调试绘制。
- `AccelerationStatics.as` — `Acceleration` 命名空间：加减速距离/速度积分，输出帧率无关补偿位移。
- `AlphaStatics.as` — `AlphaStatics` 命名空间：`LinearToSawtooth` 锯齿映射、`FrequencyAlpha` 正弦频率映射。

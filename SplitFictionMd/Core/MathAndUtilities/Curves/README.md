# Core / MathAndUtilities / Curves

> 职责：提供一组标准命名的 `UCurveFloat` 曲线资产，供全项目复用统一的 0→1 缓动形状。共 1 文件。

## 内部架构

`StandardCurves.as` 使用 AngelScript 的 `asset ... of UCurveFloat` 语法，在 `Curve` 命名空间下**声明式地定义曲线资产**——编译时即生成可被蓝图/资产引用的 `UCurveFloat` 实例，无运行时逻辑。

### 关键资产

| 资产（`Curve::`） | 形状 | 构造方式 |
| --- | --- | --- |
| `SmoothCurveZeroToOne` | 平滑 S 形 0→1 | `AddAutoCurveKey(0,0)` + `AddAutoCurveKey(1,1)` |
| `LinearCurveZeroToOne` | 线性 0→1 | `AddLinearCurveKey(0,0)` + `AddLinearCurveKey(1,1)` |
| `EaseInCurveZeroToOne` | 缓入 0→1 | `AddAutoCurveKey(0,0)` + `AddCurveKeyTangent(1,1,1.192698)` |

源文件中以 ASCII 图直观标注了每条曲线的形状。

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 被引用 | 任意需要 `UCurveFloat` 的属性 | 资产引用 | 关卡/蓝图/数据资产可直接指向这些命名曲线，避免重复制作相同形状 |
| 概念相邻 | `MathAndUtilities/Math` 的 `FHazeEased` | 替代方案 | 缓动既可用本目录曲线资产，也可用 `FHazeEased` 程序化缓动；前者偏美术可视化配置 |

## 关键文件

- `StandardCurves.as` — 在 `Curve` 命名空间声明三条标准 0→1 曲线资产（平滑/线性/缓入），编译期生成 `UCurveFloat`。

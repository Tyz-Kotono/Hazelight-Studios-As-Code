# Gameplay / Movement / ResolverExtensions —— 运动求解器扩展

> 职责：接入 Core 运动求解管线的**约束钩子**——在 Resolver 每帧扫掠求解时改写位移，把玩家约束到某个几何形状（圆形边界、样条隧道）。共 **9 文件**，两组扩展。
>
> 前置：Core 的扩展机制 `UMovementResolverExtension` 见 [Core/Movement](../../../Core/Movement/README.md#扩展机制extension)。本文讲 Movement 域用它做了什么。

---

## 什么是 Resolver 扩展

Core 的 `UBaseMovementResolver` 在扫掠求解（sweep → depenetration → redirect）时会调用挂在它身上的 `UMovementResolverExtension` 钩子。扩展可以在**不改 Actor** 的前提下（唯一例外 `PreApplyResolvedData`），逐迭代改写 `MovementDelta`——从而把"玩家能到哪"约束在一个几何范围内。

通用签名：

```cpp
class UXxxResolverExtension : UMovementResolverExtension
{
    // 声明支持哪些求解器（通常四种全支持）
    default SupportedResolverClasses.Add(USimpleMovementResolver);
    default SupportedResolverClasses.Add(USteppingMovementResolver);
    default SupportedResolverClasses.Add(USweepingMovementResolver);
    default SupportedResolverClasses.Add(UTeleportingMovementResolver);

    void PrepareExtension(...) override        { /* 从组件取约束参数 */ }
    bool OnPrepareNextIteration(...) override  { /* 每次迭代改写 delta */ }
    void CopyFrom(...) override                { /* 支持时光重跑复制 */ }
#if !RELEASE
    void LogFinal(...) override                { /* 时光日志可视化 */ }
#endif
}
```

每组扩展的标准配套：`*ResolverExtension`（扩展本体）+ `*Component`（挂在玩家上、持约束参数）+ `*Actor`（世界里定义约束几何）+ `*Statics`（启停接口）+ 触发体。

> 注：本目录是**通用**运动约束扩展。玩家双人边界约束（间距/视锥/领跑）也用同一机制，但放在 [Player/Boundary](../Player/README.md#boundary--双人边界约束3)；`SplineLock` 动作自带的扩展在 `Player/Moves/SplineLock/`。

---

## 两组扩展

### CircleConstraint —— 圆形约束（4）

把玩家约束在一个圆盘内（绕圈战斗场、圆形平台）。

| 文件 | 作用 |
| --- | --- |
| `CircleConstraintResolverExtension.as` | 扩展本体：给定圆心/法线/半径，把越界位移钳回圆内 |
| `CircleConstraintResolverExtensionActor.as` | 世界 Actor：定义圆心、半径、是否硬约束 |
| `CircleConstraintResolverExtensionComponent.as` | 玩家侧组件：持有指向 Actor 的引用 |
| `CircleConstraintResolverExtensionStatics.as` | 启停接口 |

约束逻辑（`CalculateWantedDelta`）：算出目标位置到圆心的水平偏移，若超出半径——
- **硬约束**（`bHardConstraint`）：直接钳到半径边缘；
- **软约束**：求位移与边缘平面的交点，在交界处停下并沿边缘滑动，保留竖直分量。

### SplineCollision —— 样条碰撞（5）

沿一条样条形成"隧道/管道"碰撞（限制玩家在样条周围一定半径内，可挖洞）。

| 文件 | 作用 |
| --- | --- |
| `SplineCollisionResolverExtension.as` | 扩展本体：取最近样条，按半径约束位移；半径来自求解器的 trace 形状尺寸 |
| `SplineCollisionComponent.as` | 玩家侧组件：`GetClosestSpline`、世界上方向配置 |
| `SplineCollisionHoleComponent.as` | 在样条碰撞上"挖洞"（局部放行，如出入口） |
| `SplineCollisionStatics.as` | 启停接口 |
| `SplineCollisionPlayerTrigger.as` | 触发体：玩家进入区域时启用样条碰撞 |

---

## 与动作系统的关系

Resolver 扩展是**正交于动作**的一层：任何动作（走跑跳冲）的 `ApplyMove` 都经过同一个 Resolver，因此只要扩展挂着，**所有动作**产生的位移都会被自动约束——动作能力本身完全不感知约束的存在。这正是把约束做成 Resolver 扩展而非塞进每个动作的原因。

---

## 关键文件

| 文件 | 作用 |
| --- | --- |
| `CircleConstraint/CircleConstraintResolverExtension.as` | 圆形约束（硬/软约束逻辑范例） |
| `SplineCollision/SplineCollisionResolverExtension.as` | 样条隧道碰撞 |
| `SplineCollision/SplineCollisionHoleComponent.as` | 样条碰撞挖洞 |
</content>

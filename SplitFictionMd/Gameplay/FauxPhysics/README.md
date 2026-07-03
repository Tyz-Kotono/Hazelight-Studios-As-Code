# Gameplay / FauxPhysics 功能域总览

> 职责：**轻量伪物理工具箱**——不接引擎刚体，用少量脚本模拟力/冲量/旋转/平移/样条约束/弹簧，让道具"看起来在动"且可联机同步。共 **15 文件**。
>
> **前提**：本文承接 [Gameplay 设计哲学](../DesignPhilosophy.md)与 Core Crumb/样条框架。这里不重复哲学。

---

## 一句话定位

> **FauxPhysics 是给设计师用的、蓝图/数据驱动的假物理积木。** 两类组件：**被模拟体**（继承 `UFauxPhysicsComponentBase`，持速度/旋转状态，被积分求解）与**力发射器**（`ForceComponent`/`WeightComponent`/`SpringConstraint`/`PlayerWeightComponent`，每帧把力推给上层的被模拟体）。数据流是"发射→沿挂接链传播→累加→子步积分→应用→清零"。

---

## 目录结构

| 位置 | 文件数 | 内容 |
| --- | --- | --- |
| 根 | 4 | `FauxPhysicsComponentBase.as`（抽象基类+子步/休眠框架）、`FauxPhysicsStatics.as`（`FauxPhysics::` 传播函数）、`FauxPhysicsHelpers.as`（数学：摩擦/线性转角/四元数）、`FauxPhysicsComponentDebug.as` |
| `Components` | 10 | Force/Weight/PlayerWeight、Translate/CircleTranslate/SplineTranslate/SplineFollow、AxisRotate/ConeRotate/FreeRotate |
| `Constraints` | 1 | `FauxPhysicsSpringConstraint.as`（胡克弹簧） |

---

## 内部架构

### 两大类角色

**A. 被模拟体** `: UFauxPhysicsComponentBase : USceneComponent`（抽象）——持状态、被积分。子类重写 `PhysicsStep(dt)`（不是 Tick）：

| 组件 | 状态 | 约束 |
| --- | --- | --- |
| `UFauxPhysicsTranslateComponent` | 3D 质点速度 | 每轴 min/max 盒/球 + 反弹 + 可选弹回 |
| `UFauxPhysicsCircleTranslateComponent` | 圆周质点 | 固定半径圆 |
| `UFauxPhysicsSplineTranslateComponent` | 线性体 | 约束在样条内侧 + 高度范围 |
| `UFauxPhysicsSplineFollowComponent` | 样条 1D 珠子（`FSplinePosition`+标量速度） | 端点反弹 |
| `UFauxPhysicsAxisRotateComponent` | 1 自由度铰链（标量角+角速度） | 角度上下限 |
| `UFauxPhysicsConeRotateComponent` | 3D 四元数旋转 | 锥角约束 |
| `UFauxPhysicsFreeRotateComponent` | 3D 自由自旋 | 可选角速度上限 |

**B. 力发射器** `: USceneComponent`/`UActorComponent`——不模拟，每帧把力推给父级被模拟体：

- `UFauxPhysicsForceComponent`：恒定方向力（`bWorldSpace`）。
- `UFauxPhysicsWeightComponent`：重力（硬编码 5500）+ 惯性（数值微分速度）。
- `UFauxPhysicsPlayerWeightComponent`（`: UActorComponent`）：玩家落上/栖息/荡绳/攀爬时把体重压给物体，订阅一整套移动事件（`UMovementImpactCallbackComponent`、`UPerchPointComponent`、`USwingPointComponent` 等）。
- `UFauxPhysicsSpringConstraint`：拉向锚点的胡克弹簧（`Strength = Distance * SpringStrength/100`，可 clamp 到 `MaximumForce`）。

### 力 / 约束数据流（系统核心）

```
① 发射+传播（FauxPhysicsStatics.as）
   发射器调 FauxPhysics::ApplyFauxForceToParentsAt(Child, WorldLoc, Force)
   → 沿挂接链向上，遇到每个 UFauxPhysicsComponentBase 调其 ApplyForce
   语义：Force 随时间施加（内部 ×dt）；Impulse 瞬时一次；Movement 直接位移
② 累加：各体的 ApplyForce 累进 PendingForces / PendingImpulses（×ForceScalar），Wake()
   （旋转体先把线性力转角：ConvertToAngularDelta / Calculation::LinearToAngular；样条体投影到切向）
③ 积分（基类 UpdateFauxPhysics → 子步循环，MaxStepTime=1/120s，仅控制端）
   PhysicsStep 内：Velocity += Forces*dt；Velocity += Impulses（不乘dt，且每步清零）；
   Velocity *= Pow(Exp(-Friction), dt)；再应用约束/反弹
④ 应用：ControlUpdateSyncedPosition 写 transform 并拷进网络同步值
⑤ 清零：ResetForces() 清 PendingForces——发射器必须每帧重发
```

**休眠/唤醒**：可休眠时开局睡眠，`Wake()` 阻止休眠 100 帧；有待处理力/非零速度/活动弹簧/未收敛同步时拒绝休眠。Tick 组 `TG_PrePhysics`。

---

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 依赖 → | Core Crumb | 每个被模拟体 `UHazeCrumbSyncedFloat/Vector/SplinePositionComponent` + 五档 `NetworkMode`（Local/ActorControl/Mio/Zoe/TwoWaySynced） | 控制端算、远端读 `Synced.Value` |
| 依赖 → | Core Crumb（碰撞事件） | 碰撞 `TriggerImpact` → `NetSendImpact(CrumbTrailSendTime)`，远端按 crumb 时间对齐回放 | 冲击事件时序一致 |
| 依赖 → | Core 样条 | `Spline::GetGameplaySpline`、`UHazeSplineComponent`、`GetPlaneConstrainedClosestSplinePositionToWorldLocation` | 仅两个样条组件用 |
| 依赖 → | Movement 事件源 | `UMovementImpactCallbackComponent`、`UPerchPointComponent`、`USwingPointComponent`、`APoleClimbActor` | PlayerWeight 订阅玩家落点 |
| 被使用 ← | `Gameplay/SideInteractions/TitanicBoat/TitanicBoat.as` | `UFauxPhysicsConeRotateComponent`+`UFauxPhysicsPlayerWeightComponent` | 旗舰用例：船随人重心摇晃 |
| 被使用 ← | `Gameplay/Movement/Player/MovesContextual/Grapple/Actors/GrappleBashPoint.as` | `UFauxPhysicsTranslateComponent SpringRoot` | 抓钩点被撞后弹回 |

> **重要**：AS 层显式引用极少（真实用量在蓝图/关卡实例——所有类都是 `ClassGroup=FauxPhysics` 带编辑器可视化的 `EditAnywhere` 组件），脚本级集成点只有 TitanicBoat 与 GrappleBashPoint。

---

## 关键文件

- `Gameplay/FauxPhysics/FauxPhysicsComponentBase.as` —— 抽象基类：子步积分、休眠、禁用栈、层级更新。
- `Gameplay/FauxPhysics/FauxPhysicsStatics.as` —— `FauxPhysics::ApplyFauxForceToParents/Actor*` 传播函数（Force/Impulse/Movement 语义）。
- `Gameplay/FauxPhysics/Components/FauxPhysicsForceComponent.as` —— 最基础的恒力发射器。
- `Gameplay/FauxPhysics/Components/FauxPhysicsPlayerWeightComponent.as` —— 玩家体重发射器（订阅移动事件）。
- `Gameplay/FauxPhysics/Constraints/FauxPhysicsSpringConstraint.as` —— 胡克弹簧约束。
- `Gameplay/FauxPhysics/Components/FauxPhysicsSplineFollowComponent.as` —— 样条珠子（`FSplinePosition`）。

# Core / Movement 功能域总览

> 职责：Split Fiction 中所有角色与物体的底层运动求解系统，提供"状态机式分阶段"的逐帧运动管线、可插拔的运动模式、外部注入钩子、样条约束，以及上层程序化的"移动到目标"能力。共 2 个子模块、58 个文件。

本功能域由两个子模块组成：

| 子模块 | 路径 | 文件数 | 定位 |
| --- | --- | --- | --- |
| Movement | `Core/Movement` | 51 | 底层运动求解内核：组件 + 数据 + 求解器 + 模式 + 扩展 + 样条约束 + 静态库 + 调试 |
| MoveTo | `Core/MoveTo` | 7 | 建立在 Movement 之上的程序化"移动到目标"能力（动画/跳跃/平滑/瞬移） |

> 注：`MoveTo` 在源码树中是与 `Movement` 平级的目录（`Core/MoveTo`），但在功能上属于 Movement 域的上层应用，因此归并到本功能域文档之下。

## 两子模块的关系

- `Movement` 提供权威内核 `UHazeMovementComponent`（只持状态、不含逻辑）以及一套"准备 → 求解 → 写回"的逐帧管线。
- `MoveTo` 不直接做碰撞求解，而是作为**消费者**：它的能力（Capability）每帧调用 `UHazeMovementComponent::PrepareMove` / `ApplyMove`，借助内核把角色程序化地推向一个目标变换；平滑/瞬移类型则改走 `TeleportStatics`（同属 Movement 子模块）。
- 二者通过本域约定的 **5 种协作机制** 衔接（详见各子模块 README）：
  1. **直接调用** `Type::Get` 获取组件后调用其方法。
  2. **Crumb**（`HasControl` + `CrumbFunction`）做网络化触发。
  3. **标签阻塞** `CapabilityTags` / `BlockExclusionTags` 控制能力互斥。
  4. **委托** `event` / `Broadcast` 回调（如 `FOnMoveToEnded`、`OnPostMovement`）。
  5. **可组合设置** `ApplySettings`（`UHazeComposableSettings` 派生的各类 `*Settings`）。

双人协作（Mio / Zoe）：两名玩家各自拥有独立的 `UHazeMovementComponent`，运动求解默认在 `HasControl` 一侧进行，再通过 Crumb 同步位置（`UHazeCrumbSyncedActorPositionComponent`）到远端。

## 逐帧运动管线

内核 `UHazeMovementComponent` 本身不写运动逻辑，逻辑分散在「数据（Data）」与「求解器（Resolver）」中，按如下管线驱动：

```
[上层 Capability / MoveTo / Statics]
        │  ① 取得组件
        ▼
UHazeMovementComponent
        │
        │  ② PrepareMove(DataType)
        ▼
UBaseMovementData ──────── 记录初始状态（变换/碰撞体/dt/世界Up），
   （子类: Simple/Sweeping/      锁定组件为 Preparing 状态
    Floating/Stepping/
    Teleporting/Follow）
        │  ③ 上层在 Data 上累加速度/冲量/重力/旋转
        │     AddVelocity / AddDelta / AddImpulse / AddGravityAcceleration ...
        ▼
   ApplyMove(MoveData)
        │  ④ 通过 ResolverDataLinks 找到该 Data 绑定的 Resolver（可被 OverrideResolver 覆盖）
        ▼
UBaseMovementResolver
        │  PrepareResolver → PostPrepareResolver（注入 Extensions）
        │  ⑤ ResolveAndApplyMovementRequest:
        │       扫掠(sweep) → 去穿透(depenetration) → 沿面重定向(redirect)
        │       迭代 MaxRedirectIterations / MaxDepenetrationIterations 次
        │       期间调用 Extension 钩子（如 SplineLock 约束到样条平面）
        ▼
   ApplyResolve（唯一允许写回 MovementComponent 的阶段）
        │  ⑥ 写回 contacts / impacts / 速度 / 变换
        ▼
   PostResolve → OnPostMovement → BroadcastOnPostMovement
        │  自动跟随地面 / 跟随平台 / 留下位置 Crumb 同步给远端
        ▼
   下一帧
```

关键不变量：

- **Preparing 阶段锁定**：一旦 `PrepareMove` 被调用，对组件状态的任何修改都被禁止（`devCheck CurrentMovementStatus != Preparing`），所有改动必须在 `PrepareMove` 之前完成。
- **Resolver 只改自己**：`Resolve` 阶段求解器只允许修改自身，唯有 `ApplyResolve` 能写回组件——这是支持"时光回溯重跑（Temporal Rerun）"与"并行求解（Parallel Resolve）"的前提。
- **可重跑**：`UBaseMovementData::CopyFrom` 必须覆盖全部字段，使一帧的运动可被复制并在调试器中重放。

## 子模块文档

- [`Movement/README.md`](./Movement/README.md) — 运动求解内核。
- [`MoveTo/README.md`](./MoveTo/README.md) — 程序化"移动到目标"能力。

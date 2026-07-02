# Core / Movement / Movement

> 职责：Split Fiction 的底层运动求解内核——以"持状态不含逻辑"的 `UHazeMovementComponent` 为核心，配合分阶段的逐帧管线（PrepareMove → Data → ApplyMove → Resolver → ApplyResolve）、6 种可插拔运动模式、求解器扩展钩子、样条约束、运动静态库与时光调试系统。共 51 个文件。

## 内部架构

### 核心组件：`UHazeMovementComponent`（`MovementComponent.as`）

权威运动组件，**只持有状态、几乎不含运动逻辑**。运动逻辑外置到 Data 与 Resolver 中。它的职责：

- 持有当前/上一帧的**接触**（`FMovementContacts`：Ground/Wall/Ceiling）、**累积撞击**（`FMovementAccumulatedImpacts`）、**下落状态**（`FMovementFallingData`）、世界 Up、朝向、重力方向、冲量列表。
- 维护**跟随系统**（Follow Attachment）：让角色跟随移动平台/参考系，含延迟取消跟随（Deferred Unfollow）、自动跟随地面、速度继承。
- 管理 **Data ↔ Resolver 的链接**（`ResolverDataLinks`/`FLinkedMovementResolverDataInformation`），并支持按优先级覆盖求解器（`OverrideResolver`）。
- 通过 `UHazeCrumbSyncedActorPositionComponent` 做**网络位置同步**（双人 Mio/Zoe，求解在 `HasControl` 侧，远端读 Crumb）。
- 用 `access:` 访问修饰符把内部状态严格暴露给 Data/Resolver/Extension/SplineLock/Debug 等受信类。

状态机：`Idle → Preparing（PrepareMove 后锁定）→ Resolving（ApplyMove 中）→ MovementPerformed（PostResolve）`。Preparing 阶段对组件的任何写操作都会触发 `devCheck`。

### 运动模式（MovementTypes，可插拔三件套）

每种模式 = **Data + Resolver + Settings**：

- `UBaseMovementData`（`MovementTypes/Base/BaseMovementData.as`）：抽象基类，代表"一次移动的初始状态 + 待应用的增量"。`PrepareMove` 拍下初始变换/碰撞体/dt/世界Up/可行走坡度等；上层在其上累加 `AddVelocity` / `AddDelta` / `AddImpulse` / `AddGravityAcceleration` / `SetRotation`，以及 Crumb 同步位移。增量存于 `FMovementIterationDeltaStates`（Movement + Impulse 两类）。`CopyFrom` 支持重跑复制。
- `UBaseMovementResolver`（`MovementTypes/Base/BaseMovementResolver.as`）：抽象基类，读取 Data 后执行 `ResolveAndApplyMovementRequest`（扫掠/去穿透/沿面重定向，迭代 `MaxRedirectIterations`/`MaxDepenetrationIterations` 次）。约定：`Resolve` 只改自身（`MutableData`/`IterationState`/`AccumulatedImpacts`），仅 `ApplyResolve` 能写回组件——从而支持时光重跑与并行求解。
- `UMovementResolverMutableData`（`MovementTypes/MovementResolverMutableData.as`）：求解器每帧可变工作数据。

六种具体模式（各 3 文件，Simple/Sweeping/Floating/Stepping/Teleporting/Follow）：

| 模式 | 用途 |
| --- | --- |
| Simple | 最轻量；可选维持上坡水平速度、可选悬浮高度（不做校验） |
| Sweeping | 标准扫掠求解，处理碰撞滑动 |
| Floating | 悬浮在表面之上并做校验（避免天花板/斜墙穿插） |
| Stepping | 处理台阶上/下，避免上下楼梯抖动 |
| Teleporting | 瞬移类移动 |
| Follow | 跟随组件移动（平台/参考系），由组件的 Follow 系统专用，含 `FollowComponentMovementData/Resolver/Settings` 与 `MovementFollowDataComponent.as` |

### 扩展机制（Extension）

`UMovementResolverExtension`（`Extension/MovementResolverExtension.as`）：通用求解器扩展接口，通过 `ApplyResolverExtension`/`ClearResolverExtension` 注入到组件，并按 `SupportedResolverClasses` 过滤挂到匹配的求解器上。提供大量钩子：`PrepareExtension` / `OnPrepareNextIteration` / `PreResolveStartPenetrating` / `PreHandleMovementImpact` / `OnUnhinderedPendingLocation` / `Pre/PostApplyResolvedData`。约束：扩展内禁止改 actor，唯一例外是 `PreApplyResolvedData`。同样支持重跑复制（`CopyFrom`/`GetRerunCopy`）。

### 样条约束（SplineLock）

把角色运动约束到一条样条所定义的平面上：

- `USplineLockComponent`：管理锁定状态机（Entering/Locked/...）、进入方式（MoveInto/Snap/SmoothLerp）、平面类型；在 `BeginPlay` 把自身注册进 `UHazeMovementComponent.SplineLockComponent`。
- `USplineLockResolverExtension`：实现为一个 `UMovementResolverExtension`，在求解迭代中把位置投影到样条平面（这是 SplineLock 真正作用于管线的方式）。
- `SplineLockSettings.as` / `SplineLockStatics.as`：可组合设置与 `LockMovementToSpline*` / `UnlockMovementFromSpline*` mixin 入口。

### 静态库（Statics）

- `MovementStatics.as`：`AddMovementAlignsWith{Ground,Any}Contact` 等 mixin，配置组件的接触对齐/沿边行为。
- `MovementPhysicsStatics.as`：纯几何 `FMovementDelta` 运算（`SurfaceProject` / `PlaneProject` / `Reflect` / `Bounce` / `ProjectOntoNormal` / `LimitToNormal`），供求解器/自定义运动使用。
- `TeleportStatics.as`：`TeleportActor` / `SmoothTeleportActor` mixin 及 `UActorTeleportHelper`。

### 其他支撑

- 数据结构定义集中在 `MovementIncludes.as`（枚举、`FMovementGravityDirection`、`FHazeMovementComponentAttachment`、`FMovementFrameVelocity` 等）与 `MovementDelta` / `MovementContacts` / `MovementHitResult` / `MovementAccumulatedImpacts` / `MovementIterationState` / `MovementSettings`。
- Components：`UInheritVelocityComponent`（从平台继承速度）、`UMovementInstigatorLogComponent`（调试谁推动了运动）。
- Capabilities：`UMovementFollowFloorMarkerCapability`（跟随地面 Marker）。
- CustomMovement：`UMovementResponseBallPhysicsComponent`（球体物理式自定义响应 + 可视化）。
- Debug：`MovementDebug` 命名空间、`UMovementResolverTemporalLog`、`UMovementTemporalRerunExtender`（时光回溯重跑 UI）、Transform/ActorDetails 时光记录组件——支撑前述"可重跑"特性。
- PerformanceTest：`AParallelMovementPerformanceTest` 等，验证并行求解。

### 内部数据流

```
Capability/Statics 取组件 → PrepareMove(Data) 锁定
  → Data 累加增量(DeltaStates) → ApplyMove
  → 经 ResolverDataLinks 取 Resolver（可被 Override）
  → PrepareResolver + 注入 Extensions
  → ResolveAndApplyMovementRequest（扫掠/去穿透/重定向，逐迭代回调 Extension）
  → ApplyResolve 写回 contacts/impacts/速度/变换
  → PostResolve：自动跟随地面/平台 + 留位置 Crumb 同步远端
```

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
| --- | --- | --- | --- |
| 被调用 | 上层玩家 Capability / `MoveTo` | 直接调用 `Type::Get` + `PrepareMove`/`ApplyMove`/`Setup*MovementData` | 内核对外入口；`MoveTo/JumpToCapability.as`、`MoveTo/MoveToStatics.as` 等命中 `PrepareMove`/`ApplyMove`/`FollowComponentMovement` |
| 被调用 | 全 Core 各域 | 直接调用 `UHazeMovementComponent::Get` | 38 处命中，含 `Audio`、`Collision/PlayerTraceHelpers`、`PlayerHealth`、`Targetable` 等读取运动状态 |
| 内部 | Data ↔ Resolver | 链接表 `ResolverDataLinks` + `OverrideResolver` | `MovementComponent.as` 通过 `FLinkedMovementResolverDataInformation` 绑定与按优先级覆盖 |
| 内部 | SplineLock | 扩展注入 `UMovementResolverExtension` + 直接持有 `SplineLockComponent` | `SplineLockComponent.as` 在 `BeginPlay` 写入 `MoveComp.SplineLockComponent`；约束逻辑由 `SplineLockResolverExtension` 在迭代中执行 |
| 出 | 远端玩家（Mio/Zoe） | Crumb（`HasControl` + `UHazeCrumbSyncedActorPositionComponent`） | `PostResolve` 留下相对位置 Crumb；`CrumbSnapToGroundPostSequencer` 等 `CrumbFunction` |
| 出 | 上层逻辑 | 委托 `Broadcast` | `BroadcastOnPostMovement` / `OnPostMovement` 通知一帧运动完成 |
| 配置 | 各系统 | 可组合设置 `ApplySettings`（`UHazeComposableSettings`） | `UMovementGravitySettings` / `UMovementStandardSettings` / `UMovementResolverSettings` / 各模式 `*Settings`，`BeginPlay` 经 `::GetSettings` 读取 |
| 调试 | TemporalLog | 直接调用 + 重跑复制 | `MovementDebug`、`UMovementTemporalRerunExtender` 在 `bCanRerunMovement` 时注册 |

## 关键文件

- `MovementComponent.as` — 核心组件 `UHazeMovementComponent`，持状态、驱动管线、管理跟随与网络同步。
- `MovementIncludes.as` — 域内枚举与核心结构体（重力方向、跟随附着、帧速度、自定义状态等）。
- `MovementSettings.as` — 重力/标准/求解器等可组合设置类。
- `MovementDelta.as` / `MovementContacts.as` / `MovementHitResult.as` / `MovementAccumulatedImpacts.as` / `MovementIterationState.as` — 运动增量、接触、撞击、迭代状态的数据结构。
- `MovementFollowDataComponent.as` — 跟随移动所需的数据组件。
- `MovementTypes/Base/BaseMovementData.as` — Data 抽象基类：拍快照初始状态 + 累加增量 + 重跑复制。
- `MovementTypes/Base/BaseMovementResolver.as` — Resolver 抽象基类：扫掠/去穿透/重定向求解，只改自身、仅 ApplyResolve 写回。
- `MovementTypes/MovementResolverMutableData.as` — 求解器每帧可变工作数据。
- `MovementTypes/{Simple,Sweeping,Floating,Stepping,Teleporting,Follow}/*` — 六种可插拔模式，每组 Data+Resolver+Settings。
- `Extension/MovementResolverExtension.as` — 求解器扩展抽象基类，外部注入钩子。
- `SplineLock/SplineLockComponent.as` — 样条锁定状态机组件。
- `SplineLock/SplineLockResolverExtension.as` — 把位置投影到样条平面的求解器扩展。
- `SplineLock/SplineLockSettings.as` / `SplineLockStatics.as` — 样条锁定设置与 `LockMovementToSpline*` 入口。
- `Statics/MovementStatics.as` — 接触对齐/沿边行为配置 mixin。
- `Statics/MovementPhysicsStatics.as` — `FMovementDelta` 几何运算工具。
- `Statics/TeleportStatics.as` — `TeleportActor`/`SmoothTeleportActor` 入口与 Helper。
- `Components/InheritVelocityComponent.as` — 从平台/参考系继承速度。
- `Components/MovementInstigatorLogComponent.as` — 记录运动发起者用于调试。
- `Capabilities/MovementFollowFloorMarkerCapability.as` — 跟随地面 Marker 的能力。
- `CustomMovement/MovementResponseBallPhysics.as` — 球体物理式自定义运动响应 + 可视化。
- `Debug/*` — 时光日志、重跑扩展、Transform/ActorDetails 记录组件等调试设施。
- `PerformanceTest/*` — 并行求解性能测试 actor/capability/manager。

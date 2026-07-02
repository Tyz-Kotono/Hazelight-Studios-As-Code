# Core / Movement / MoveTo

> 职责：建立在运动内核之上的程序化"移动到目标"系统——把"让角色走到/跳到/瞬移到某个变换"封装为统一的 `MoveTo` 请求，由 `UMoveToComponent` 排队、网络化、再分派给 4 种执行能力。共 7 个文件。

## 内部架构

### 请求与目标的数据模型（`MoveToData.as`）

- `FMoveToParams`：一次移动的参数——类型 `EMoveToType`（`NoMovement` / `SmoothTeleport` / `SnapTeleport` / `AnimateTo` / `JumpTo`）、定位方式 `EMoveToPosition`（到目标点 / 距目标固定距离 / 距目标区间）、距离与是否朝向目标、跳跃额外高度等。附 `NoMovement()` / `SmoothTeleport()` / `SnapTeleport()` / `AnimateTo()` 工厂。
- `FMoveToDestination`：目标变换，可由 actor/组件（带相对变换，运行时跟随该组件世界变换）或绝对 transform/位置构造；`CalculateDestination` 据 `FMoveToParams` 计算最终落点（含距离/区间投影与朝向计算）。MoveTo 永不改变角色缩放。
- `FOnMoveToEnded`：完成回调委托，签名 `void(AHazeActor)`。

### 调度组件：`UMoveToComponent`（`MoveToComponent.as`）

- 维护 `PendingMoveTos_Control`（控制端待处理队列）与当前激活的 `ActiveMoveTo`，分配自增 `Id`。
- `MoveTo()`：仅在 `HasControl` 时入队并启用 Tick。
- `CanActivateMoveTo` / `ActivateMoveTo` / `IsMoveToActive` / `FinishMoveTo`：供各执行能力查询并领取队首请求、激活、结束（触发 `OnMoveToEnded`）。
- `Tick` + `CrumbCancelPendingMoveTo`（`CrumbFunction`）：若请求几帧未被任何能力领取则网络化地取消，避免悬挂。

### 执行能力（4 个 Capability）

请求由 `UMoveToComponent` 排队后，按 `FMoveToParams.Type` 由对应能力领取并执行：

| 能力文件 | 类型 | 执行方式 | 关键标签/调度 |
| --- | --- | --- | --- |
| `AnimateToCapability.as` | `AnimateTo` | 播放朝目标的动画位移（过短则退化为平滑瞬移，阈值 `ANIMATE_TO_INSTANT_RANGE`），每帧驱动 `UHazeMovementComponent` | `CapabilityTags: Movement, MoveTo`；`BlockExclusionTags: UsableDuringMoveTo`；`NetworkMode=Crumb`；`TickGroup=InfluenceMovement` |
| `JumpToCapability.as` | `JumpTo` | 程序化抛物跳到目标（额外高度来自 `JumpAdditionalHeight`） | 同上（`Movement` + `MoveTo` 标签、Crumb、InfluenceMovement） |
| `MoveToSmoothTeleportCapability.as` | `SmoothTeleport` | 走 `TeleportStatics::SmoothTeleportActor`（瞬移本体、网格平滑补偿） | `BlockedByCutscene`；`NetworkMode=Crumb`；`TickGroup=BeforeMovement` |
| `MoveToSnapTeleportCapability.as` | `SnapTeleport` | 走 `TeleportStatics::TeleportActor`（直接硬瞬移） | `BlockedByCutscene`；`NetworkMode=Crumb`；`TickGroup=BeforeMovement` |

### 入口静态库（`MoveToStatics.as`）

- `MovePlayerTo`（mixin，玩家专用）/ `MoveTo::MoveActorTo`：通用入口，`NoMovement` 立即回调结束，否则在 `HasControl` 侧 `UMoveToComponent::GetOrCreate(Actor).MoveTo(...)`。
- `MoveTo::ApplyMoveToInstantly`：不走网络的即时应用（调用方需自行处理网络化）。

### 内部数据流

```
MovePlayerTo / MoveActorTo(Params, Destination, OnEnded)
   → UMoveToComponent::GetOrCreate(Actor).MoveTo()  （仅 HasControl，入队 + 启用 Tick）
   → 对应 Type 的 Capability 每帧 CanActivateMoveTo 领取队首
   → ActivateMoveTo → 执行：
        AnimateTo/JumpTo  → 每帧 PrepareMove/ApplyMove 驱动 UHazeMovementComponent（运动内核）
        Smooth/SnapTeleport → TeleportStatics 瞬移
   → FinishMoveTo → 触发 FOnMoveToEnded 回调
   全程经 Crumb 网络化（NetworkMode=Crumb / CrumbFunction）同步到远端
```

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
| --- | --- | --- | --- |
| 入 | 上层逻辑（任意系统） | 直接调用 `MovePlayerTo` / `MoveTo::MoveActorTo` | `MoveToStatics.as` 暴露 mixin；Core 内 `Interaction/InteractionEnterCapability.as` 命中 `UMoveToComponent` |
| 出 | 运动内核 Movement | 直接调用 `Type::Get` + `PrepareMove`/`ApplyMove` | `AnimateToCapability` / `JumpToCapability` 在 `Setup` 取 `UHazeMovementComponent::Get`，每帧驱动管线 |
| 出 | 运动内核 Movement | 直接调用 `TeleportStatics` | `MoveToSnapTeleportCapability` 走 `TeleportActor`；Smooth 走 `SmoothTeleportActor` |
| 内部 | 队列 ↔ 能力 | 标签阻塞 `CapabilityTags` / `BlockExclusionTags` | `Movement`/`MoveTo` 标签 + `UsableDuringMoveTo` 排他，控制能力互斥与可并存 |
| 出 | 远端玩家（Mio/Zoe） | Crumb（`HasControl` + `CrumbFunction` + `NetworkMode=Crumb`） | `UMoveToComponent.MoveTo` 仅控制端入队，`CrumbCancelPendingMoveTo` 同步取消 |
| 出 | 调用方 | 委托 `FOnMoveToEnded` | `FinishMoveTo` 触发 `OnMoveToEnded.ExecuteIfBound` 通知移动结束 |

## 关键文件

- `MoveToData.as` — 请求/目标数据模型：`FMoveToParams`、`FMoveToDestination`、`EMoveToType`/`EMoveToPosition` 与完成委托 `FOnMoveToEnded`。
- `MoveToComponent.as` — `UMoveToComponent`：控制端请求队列、激活/结束、Crumb 取消调度。
- `MoveToStatics.as` — 统一入口 `MovePlayerTo` / `MoveActorTo` / `ApplyMoveToInstantly`。
- `AnimateToCapability.as` — `AnimateTo` 执行能力：动画位移驱动运动内核（过短退化为平滑瞬移）。
- `JumpToCapability.as` — `JumpTo` 执行能力：程序化抛物跳到目标。
- `MoveToSmoothTeleportCapability.as` — `SmoothTeleport` 执行能力：网格平滑补偿的瞬移。
- `MoveToSnapTeleportCapability.as` — `SnapTeleport` 执行能力：硬瞬移。

# Core / Systems / Network

> 职责：联机基础设施——每玩家互斥锁与自动所有权转移、自适应 Crumb 缓冲与时间预测、编辑器联机调试输入。共 4 文件（含 `Debug/` 子目录）。**本模块是协作 Crumb 机制的实现基础。**

## 内部架构

### 关键类

| 类 | 基类 | 角色 |
| --- | --- | --- |
| `UNetworkLockComponent` | `UActorComponent` | 同一时刻只能被一名玩家获取的网络互斥锁，带所有权（owner）转移与提示（hint）启发式 |
| `UNetworkCrumbTrail` | `UHazeCrumbTrail` | crumb 接收回放的自适应缓冲长度算法 + 对端发送时间预测 |
| `UNetworkFreezeDevInput` / `UNetworkViewMode*DevInput` | `UHazeDevInputHandler` | 编辑器联机调试：冻结网络、切换分屏网络视图（Normal/Mio/Zoe） |

### Crumb 机制要点

「Crumb」是 Haze 引擎的确定性同步原语：有控制权（`HasControl`）的一方执行 `CrumbFunction` 标记的操作并打包成 crumb 流，发往对端按相同顺序回放，从而保证两端状态一致。本模块从两个层面支撑它：

1. **`UNetworkCrumbTrail` —— crumb 流的回放节奏控制**
   - 状态机 `ENetworkCrumbTrailState{Normal, Buffering, FastForward}`：trail 过短则进入 `Buffering` 攒够缓冲、并增大 `TrailBufferIntervals`；过长则 `FastForward` 加速消化（保证 1 秒内追平）；正常区间 `TrailSpeed = 1.0`。
   - **自适应最小长度**：用环形桶 `CrumbTrailMinLength`（20 桶 × 1 秒）记录近期最小 trail 长度，当持续存在富余（headroom）时按 `TrailSyncInterval` 缩减缓冲间隔，逼近「能稳定维持的最小缓冲」。
   - **中位偏差预测（MedianPredict）**：每帧推进预测时间，每 2 秒对累积的 `MedianPredict_Divergences` 排序取中位数，作为对端发送时间的修正量；大偏差（>1s）直接瞬移预测（应对对端卡顿），小偏差忽略以防抖动。输出 `PredictedOtherSideSendTime`。旧的 `TrailPredict`（基于 ping 的 `FInterpConstantTo`）保留对照。

2. **`UNetworkLockComponent` —— 用 crumb 保证锁操作的确定性**
   - 每玩家数据 `TPerPlayer<FNetworkLockPerPlayerData>`：`LockInstigators`（谁在请求锁）、`LockDelegates`（获锁后仅在控制侧触发的回调）、`Hints`、`bIsLocked`。
   - 锁状态变更经 `NetFunction`（`NetAcquiredLock` / `NetReleasedLock` / `NetWantLock` / `NetTransferOwner`）在两端同步。
   - **关键不变量**：释放锁后递增 `DisallowLockingCounter` 禁止对端立即上锁，直到通过 `CrumbFunction CrumbAllowLocking` 在 crumb 流到达时才解禁——确保「Acquire → Crumb → Release」期间的操作在两端都已回放完毕，对端才能重新上锁。

### 所有权与提示数据流

- `BeginPlay` 按 `Network::HasWorldControl()` 把 `CurrentOwner` 初始化为本地玩家或其 `OtherPlayer`。
- `Tick → UpdateLockNetworked`：只有「`Player.HasControl()` 且 `CurrentOwner.HasControl()`」时才真正上/解锁；本侧不想要而对端想要且 `CanTransferOwner` 时 `NetTransferOwner` 转移所有权；无人争用时按 `Hints`（`ApplyOwnerHint` 写入的权重）每秒最多一次切换 owner，提升响应性。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
| --- | --- | --- | --- |
| 被调用 | `Interaction/InteractionComponent.as` | `GetOrCreate` + 锁 API | `UNetworkLockComponent::GetOrCreate(Game::Mio, n"GlobalInteractionLock")`，互斥交互共享同一把锁；并用 `ApplyOwnerHint`/`ClearOwnerHint`/`UpdateHintValues` 喂入距离权重 |
| 被调用 | `Interaction/InteractionEnterCapability.as` | 直接调用 | 进入交互前检查/获取锁 |
| 被调用 | `ButtonMash/ButtonMashCapability.as` | 直接持有 | 成员 `UNetworkLockComponent DoubleMashLock` 用于双人连打互斥 |
| 依赖 | `Network::HasWorldControl` / `Player.HasControl` | 直接调用 | 判定控制侧、初始化 owner |
| 依赖 | `Game::Mio` / `Game::Zoe` / `Game::Players` | 直接调用 | 双人玩家枚举与调试视图切换 |
| 依赖 | `TEMPORAL_LOG` | 直接调用 | 锁事件与 crumb trail 指标的时序日志 |

## 关键文件

- `NetworkLockComponent.as` — `UNetworkLockComponent`：每玩家互斥锁、`NetFunction` 同步、`CrumbFunction CrumbAllowLocking` 保证锁释放确定性、基于 hint 的所有权转移。
- `NetworkCrumbTrail.as` — `UNetworkCrumbTrail`：Normal/Buffering/FastForward 状态机、环形桶自适应缓冲长度、中位偏差预测对端发送时间。
- `Debug/NetworkFreezeDevInput.as` — `UNetworkFreezeDevInput`：编辑器内一键 `Network::DebugToggleFrozen()` 冻结网络。
- `Debug/NetworkViewModeDevInput.as` — `UNetworkViewModeNormal/Mio/ZoeDevInput`：通过 `Haze.SingleScreenNetworkViewMode` 控制台变量切换分屏联机调试视图。

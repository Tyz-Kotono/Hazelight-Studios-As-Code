# Gameplay / KineticActors 功能域总览

> 职责：**可动平台三件套**——直线平移、旋转/自旋、沿样条移动的关卡平台，天生联机同步。共 **6 文件**，是全项目关卡机关的通用底座。
>
> **前提**：本文承接 [Gameplay 设计哲学](../DesignPhilosophy.md)与 Core Crumb 框架。这里不重复哲学。

---

## 一句话定位

> **KineticActors 不逐帧积分，而是"基于时间的确定性重放"。** 每个平台存一个 `StartCrumbTime`，每帧由 `当前 crumb 时间 - StartCrumbTime` 求出 alpha，再 lerp 出位姿。因此双端无需持续同步位置，只需同步一个起始时间标量 + 状态切换的 `CrumbFunction`，天然同步。

---

## 文件清单（6 文件）

| 文件 | 关键类 | 基类 | 运动模式 |
| --- | --- | --- | --- |
| `KineticMovingActor.as` | `AKineticMovingActor` | `AHazeActor` | 起点↔目标直线平移 |
| `KineticRotatingActor.as` | `AKineticRotatingActor` | `AHazeActor` | 往返旋转 / 恒速自旋 |
| `KineticSplineFollowActor.as` | `AKineticSplineFollowActor` | `AHazeActor` | 沿 `ASplineActor` 加减速移动 |
| `KineticCameraShakeForceFeedbackComponent.as` | `UKineticCameraShakeForceFeedbackComponent` | `UCameraShakeForceFeedbackComponent` | 运动事件触发震屏/力反馈 |
| `KineticActorVisualizer.as` | `KineticActorVisualizer` 命名空间 | —（`#if EDITOR`） | 编辑器递归预览整条附着链 |
| `KineticActorTest.as` | —— | —— | DevFunction 测试驱动 |

---

## 内部架构

三个主类共享同一套骨架：Tick 默认关闭、Tick 组 `TG_HazeInput`（早于常规逻辑，保证平台位姿先于依赖者就绪）；组件 `RootComp`+`PlatformMesh`+`UDisableComponent`（距离剔除）+编辑器件；三档缓动曲线 `EvaluateMovementCurve`/`InverseMovementCurve`（EaseInOut/In/Out/Linear）。

### 运动模式

| 类 | 模式枚举 | 档位 |
| --- | --- | --- |
| Moving | `EKineticMovementMode` | `Manual`（手动 `ActivateForward`/`ReverseBackwards`，可自动回缩）、`AlwaysMoveBackAndForth`（自动往返，可 `StartOffsetTime` 错峰）；位置 = `Math::Lerp(Origin, Target, Alpha)` |
| Rotating | `EKineticRotatingMode` | `Manual`、`AlwaysMoveBackAndForth`（默认 `LerpShortestPath`，勾 `bSupportWinding` 支持 >180°）、`AlwaysSpinAround`（`Origin + RotationSpeed*Alpha` 无限自旋） |
| SplineFollow | `EKineticSplineFollowMode` | `MoveToEnd`、`MoveBackAndForth`（端点弹跳）、`LoopAround`（取模循环）；用速度/加速度运动学，`GetSplineDistanceAtTime` + `Trajectory::GetTimeToReachTarget` 处理加减速 |

各类广播丰富事件（`OnStartForward/OnReachedForward/OnFinishedForward/OnStartBackward/…`）供订阅。

### 网络同步机制

**不用 `UHazeCrumbSynced*` 组件**，而是两招组合：

1. **`UFUNCTION(CrumbFunction)` 确定性 RPC**：所有改状态的操作成对——公开 `Crumb*()` → 内部 `Internal*()`。本地移动直接 `Internal*`；联机时仅 `HasControl()` 端调 `Crumb*`，机制把调用连同 crumb-trail 时间戳同步给对端。
2. **crumb-trail 时间轴**：`GetRelevantCrumbTime()` 按 `EKineticMovementNetwork`（8 档：Default/Local/SyncedFromHost/Mio/Zoe/PredictedSyncPosition/…）选取 `Time::GetPredictedGlobalCrumbTrailTime()` 等，两端建立在同一时间轴上算 alpha。控制权在 `BeginPlay` 用 `SetActorControlSide(Game::Mio/Zoe)` 设定。

暂停走 instigator 计数：`PauseMovement/UnpauseMovement(FInstigator)`，`ControlPauseInstigators` 归零才恢复。

---

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 依赖 → | Core Crumb | `CrumbFunction` + `Time::Get*CrumbTrailTime` | 时间轴重放实现同步 |
| 依赖 → | Core 样条 | `SplineToFollow.Spline.GetWorldTransformAtSplineDistance` 等（经 `ASplineActor.Spline`，非 `UHazeSplineComponent` 类型名） | 仅 SplineFollow |
| 被继承 ← | 关卡专属平台（13 个子类） | 继承三个基类 | 如 `PinballKineticMovingActor`、`MagnetWaterArmWheel`(Rotating)、`SkylineKineticSplineFollowActor`、`TankShark`(SplineFollow) 等，全在 `LevelSpecific/` |
| 被持有 ← | 关卡管理器/机关 | 作字段/参数遍历 | 如 `OilRigPoleRoomManager` 遍历 `AKineticMovingActor` 数组、`SandHandSwingingTrigger` 持旋转门 |
| 被订阅 ← | `UKineticCameraShakeForceFeedbackComponent` 及各关卡 FeedbackComponent | 绑 `OnStart*/OnReached*/OnFinished*` | 运动切换时触发震屏/力反馈 |
| 被消费 ← | `Audio/SoundDefinitions/World_Shared_Platform_Kinetic*_SoundDef.as` | `Cast<AKineticMovingActor>` | 平台运动音效 |

> Gameplay 树内除自身外无引用——所有消费者在 `LevelSpecific/` 与 `Audio/`，印证其"关卡机关通用底座"定位。

---

## 关键文件

- `Gameplay/KineticActors/KineticMovingActor.as` —— 直线平移平台（含 8 档网络模式与 crumb 时间轴范式）。
- `Gameplay/KineticActors/KineticRotatingActor.as` —— 旋转/自旋平台（最短路径/缠绕旋转）。
- `Gameplay/KineticActors/KineticSplineFollowActor.as` —— 样条跟随平台（速度/加速度运动学）。
- `Gameplay/KineticActors/KineticCameraShakeForceFeedbackComponent.as` —— 运动事件 → 震屏/力反馈的标准订阅范式。

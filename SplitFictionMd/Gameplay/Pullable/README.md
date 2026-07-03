# Gameplay / Pullable 功能域总览

> 职责：**沿样条拉动物体**——玩家拉住交互点，把重物沿预设样条拖行，支持单人拉或双人协力拉。共 **8 文件**。
>
> **前提**：本文承接 [Gameplay 设计哲学](../DesignPhilosophy.md)与 Core 交互/样条/Crumb 框架。这里不重复哲学。

---

## 一句话定位

> **Pullable 把"拖重物"抽象成一维样条运动。** 玩家的推杆输入投影到样条切向变成 ±1，多名拉动玩家的输入取平均后驱动物体沿样条前进/后退；双人模式要求两人同时同向才动。物体位置经 Crumb 按样条相对位置同步。

---

## 文件清单（8 文件）

| 文件 | 关键类 | 基类 | 作用 |
| --- | --- | --- | --- |
| `Pullable.as` | `APullable` | `AHazeActor` | 单交互点可拉物体 |
| `DoublePullable.as` | `ADoublePullable` | `AHazeActor` | 双交互点（Mio/Zoe 各一），强制双人 |
| `PullableComponent.as` | `UPullableComponent` | `UActorComponent` | 核心引擎：样条运动 + 状态 + 网络同步 |
| `PlayerPullableInputCapability.as` | `UPlayerPullableInputCapability` | `UInteractionCapability` | 读玩家输入 → `ApplyPullInput` |
| `PlayerPullableMovementCapability.as` | `UPlayerPullableMovementCapability` | `UInteractionCapability` | 驱动运动 + 动画 + 把玩家吸在交互点 |
| `PlayerPullComponent.as` | `UPlayerPullComponent` | `UActorComponent` | 玩家侧动画数据 |
| `PullablePullBackCapability.as` | `UPullablePullBackCapability` | `UHazeCapability` | 无人拉时自动回拉（`NetworkMode=Crumb`） |
| `PullSheet.as` | `Pullable::PullSheet` | `UHazeCapabilitySheet` | 能力表 |

---

## 内部架构

### 核心引擎 `UPullableComponent`

参数：`AActor Spline`、`PullSpeed/PullAcceleration/PullDeceleration`、`bRequireBothPlayers`、`bCompleteAtEndOfSpline`、`bAutomaticallyPullBack`+`PullBackSpeed`。运行时：`TPerPlayer<FPullablePlayerState> PerPlayerState`、`FSplinePosition CurrentPosition`、`UHazeCrumbSyncedActorPositionComponent SyncPosition`。

**样条移动**：`BeginPlay` 用 `Spline::GetGameplaySpline(Spline, this)` 拿到样条，把物体贴到最近样条位置；每帧 `CurrentPosition.Move(ActivePullSpeed * DeltaTime)` 沿样条推进，返回 false 即到端点。

**输入处理**（`ApplyPullInput`）：玩家移动向量点乘样条切向，模长 >0.2 归一为 ±1，存进 `State.PullInput` 并镜像到 `SyncPullInput`。

**方向解算**（`GetActivePullDirection`）：把所有正在拉的玩家输入求和后除以拉动人数取平均。

### 单人 vs 双人

| 模式 | 条件 | 行为 |
| --- | --- | --- |
| 单人 | `bRequireBothPlayers=false` | 一人输入即可驱动；`StartPulling` 时 `Owner.SetActorControlSide(Player)` 把控制权交给拉的人 |
| 双人 | `bRequireBothPlayers=true` | 任一人未拉则方向强制为 0（`if (bAnyNotPulling && bRequireBothPlayers) return 0.0`）；两人输入取平均，必须同向才动；控制权不转交单人 |

`ADoublePullable` 强制 `bRequireBothPlayers=true`，并把两个交互点用 `UsableByPlayers = EHazeSelectPlayer::Mio/Zoe` 锁给各自玩家。

---

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 依赖 → | Core 样条 | `Spline::GetGameplaySpline`、`FSplinePosition.Move`、`GetClosestSplinePositionToWorldLocation` | 物体运动一维化到样条 |
| 依赖 → | Core `UInteractionComponent` | actor 声明交互组件+`Pullable::PullSheet`；能力 `SupportsInteraction` 判定 owner 有 `UPullableComponent` | 玩家经交互系统"抓住"拉点 |
| 依赖 → | Core Crumb | `UHazeCrumbSyncedFloatComponent`（每人输入）、`UHazeCrumbSyncedActorPositionComponent`（物体位置，按样条相对同步）、`CrumbFunction CrumbCompletePullable` | 双端一致 |
| 依赖 → | Core 能力屏蔽 | `PullSheet` 的 `Blocks.Add(Collision/Movement/GameplayAction)` | 拉动时屏蔽常规移动，再补方向输入能力 |
| 被使用 ← | 关卡蓝图 | 继承 `APullable`/`ADoublePullable`、绑定 `OnPullableCompleted/OnStartedPulling/OnStoppedPulling`、按实例设 `Spline` | Gameplay 树内无 .as 引用，纯数据驱动 |

> 注意：`GetPulledDistanceOnSpline()` 的网络同步不可靠（两个 actor 文件都注释警告需自行同步该值）。

---

## 关键文件

- `Gameplay/Pullable/PullableComponent.as` —— 核心引擎：样条运动、单/双人方向解算、Crumb 同步。
- `Gameplay/Pullable/DoublePullable.as` —— 双人协力变体（强制两人同向）。
- `Gameplay/Pullable/PlayerPullableInputCapability.as` —— 输入采集能力。
- `Gameplay/Pullable/PlayerPullableMovementCapability.as` —— 运动+动画+吸附能力。
- `Gameplay/Pullable/PullSheet.as` —— 能力表与屏蔽声明。

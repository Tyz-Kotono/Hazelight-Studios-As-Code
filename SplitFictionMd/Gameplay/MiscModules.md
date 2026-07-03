# Gameplay 杂项小模块（MiscModules）

本篇合并介绍 Gameplay 目录下 8 个体量较小、职责相对独立的模块。它们大多不构成一条完整的玩法链路，而是围绕 AI 移动、玩法触发体、玩家伤害/反馈、网络物理与无障碍等场景，为关卡设计与角色系统提供可复用的组件与 Actor。

阅读本篇前，建议先了解 `Gameplay/DesignPhilosophy.md` 中的三条主线约定：

- **能力四件套**：`ShouldActivate / ShouldDeactivate / OnActivated / OnDeactivated`（能力的启停协议），配合 `Setup / TickActive`。
- **标签互斥**：`CapabilityTags` 用于分组与互斥调度（本篇中路径跟随统一挂 `n"Pathfinding"` 标签）。
- **组件容器范式**：把每个玩家/Actor 的状态收进一个 `UActorComponent` 容器（常见 `GetOrCreate`），能力从容器读写、彼此解耦。

## 模块索引

| # | 模块 | 文件数 | 核心类型 | 一句话职责 |
| --- | --- | --- | --- | --- |
| 1 | Navigation | 17 | `UPathfollowingMoveToComponent` / `AWallclimbingNavigationVolume` / `UWallclimbingComponent` | AI 寻路与路径跟随（地面 / 飞行 / 攀墙三套后端） |
| 2 | Triggers | 10 | `UPlayerLookAtTriggerComponent` / `AVoxPlayerTrigger` / `UPlayerWorldRumbleTrigger` | 玩法触发体（看向 / VO / 震动 / 移动区 / 不可行走区） |
| 3 | PlayerHealth | 3 | `UDamageTriggerComponent` / `UDeathOnImpactComponent` / `UHealthBarInWorldComponent` | 环境伤害触发体与世界空间血条 |
| 4 | Physics | 1 | `ANetworkedPhysicsObject` | 带控制权动态切换的网络物理对象 |
| 5 | Accessibility | 3 | `UAccessibilityAutoJumpDashCapability` | 无障碍：自动跳跃 / 二段跳 / 冲刺辅助 |
| 6 | GamepadLight | 1 | `UPlayerGamepadLightCapability` | DualSense 手柄灯映射玩家色与血量 |
| 7 | Audio | 1 | `APlayerVOTriggerVolume` | 玩家 VO 触发体积（SoundDef 挂载/移除） |
| 8 | TopDownPlayerArrow | 2 | `UTopDownPlayerArrowCapability` / `ATopDownPlayerArrow` | 俯视视角下的玩家地面位置箭头 |

---

## 1. Navigation — AI 寻路与路径跟随

**职责与文件数**：17 个文件，是本篇最大模块。分两大子系统：

- `Pathfollowing/`：路径跟随框架 —— 一套统一的“MoveTo 请求/状态机 + 路径消费能力”，下挂 **地面（Ground）/ 飞行（Flying）/ 攀墙（Wallclimbing）** 三套具体寻路后端。
- `Wallclimbing/`：一套完全自研的**攀墙导航网格（navmesh）**，从引擎八叉树（NavOctree）离线烘焙出可在任意朝向墙面上行走的面片网格，并在其上做 A\* + 漏斗（funnel）平滑。

### 1.1 路径跟随框架（Pathfollowing 公共层）

核心是**请求方**与**执行方**的解耦：

- `UPathfollowingMoveToComponent`（组件容器）：维护一个按优先级 `EPathfollowingPriority` 排序的 `MoveTo` 请求队列（`FPathfollowingMoveTo`，含目的地、Instigator、优先级、完成回调）。同一 Instigator 只保留一条；最高优先级者生效，其余进入 `Queued`。对外提供 `MoveTo / StopMoteTo / GetStatus / WasSuccess / WasFailure / IsOngoing`。
  - 状态机 `EPathfollowingMoveToStatus`：`None → Pathfinding → Moving → AtDestination / AtFarAsWeCanGo`，以及 `CouldNotFindStart/End/Path`、`Stopped`、`Queued`。
  - 持有当前路径 `FNavigationPath`（`TArray<FVector> Points` + `PathIndex`），可由引擎 `UNavigationPath` 填充。
- 三个 `...PathfollowingCapability`（执行方）：都挂 `n"Pathfinding"` 标签、`TickGroup = BeforeMovement`（在“设定目的地”与“移动能力”之间运行），仅在**控制端**（`HasControl()`）激活。它们共享同一套骨架：
  - `NeedsNewPath()`：目的地移动超阈值、或路径失效、或（不精确路径时）定时重试才重算。
  - `FindPath()`：调用各自后端求路，产出 `EBasicPathfindingResult`（`AccuratePath / InaccuratePath / BadStart / BadDestination / NoPath`），并通过 `MoveToComp.ReportNewPath/ReportFailed` 回写状态。
  - `TickActive()`：沿 `Path.Points` 逐点推进，用 `IsAt()`（距离容差 + 越过检测防止过冲）判断到点，最终 `SetPathfindingDestination()` 把“本帧要去的点”交给下游移动能力消费。

> 注意：这些 `Capability` 只负责**算路与推进目的地**，真正把角色移动到目的地的是移动能力（如 `UCharacterPathfollowingMoveToCapability`，它读取 `MoveToComp.GetPathfindingDestination()` 并驱动 `UHazeMovementComponent`）。

三套后端的差异：

| 后端 | 求路来源 | 起终点投影 | 典型使用者 |
| --- | --- | --- | --- |
| Ground | 引擎 `UNavigationSystemV1::ProjectPointToNavigation` + `FindPathToLocationSynchronously` | 先竖直窄投影再全尺寸投影 | 普通地面 AI |
| Sidescroller Ground | 同上，但把自身/路点投影到 `UHazeSplineComponent` 上比较，约束在样条平面推进 | 依赖 `UPlayerSplineLockComponent` 提供的样条 | 横版关卡 AI |
| Flying | `Navigation::NavOctree*`（八叉树取最近点、找路、漏斗化） | `NavOctreeGetNearestLocationInTree`；并周期性回查自身在八叉树中的位置作兜底 | 飞行 AI |
| Wallclimbing | 自研 `AWallclimbingNavigationVolume.FindPath`（见下） | 沿 `Owner.ActorUpVector` 抬高 `OwnerSize` 做投影，起终点带法线 | 攀墙 AI |

### 1.2 攀墙导航网格（Wallclimbing）

这是模块中算法最重的部分，核心 Actor 是 `AWallclimbingNavigationVolume : AVolume`（`Wallclimbing/WallClimbingNavigationVolume.as`，700+ 行）。

- **烘焙（编辑器 `BuildNavmesh`）**：从引擎导航八叉树叶子（`Navigation::GetNavOctreeLeaves`）中，找“空节点”与“阻挡节点”交界处生成朝外的矩形面片 `FWallclimbingNavigationFace`；再经 `ApplyModifiers`（`ANavModifierVolume` 剔除）、`MergeFaces`（同法线共面矩形合并，哈希分桶降复杂度）、`FindNeighbours`（按对齐轴哈希边、找共线重叠邻边，建立双向邻接）。结果 `NavMesh`（面片数组）与 `HashedNavMeshPolys`（空间哈希）序列化保存，`BeginPlay` 时相对 Actor 位置转回世界空间（兼容世界流送偏移）。
- **运行时查询**：
  - `FindPoly / FindLocationOnNavmesh / FindClosestNavmeshPoly`：给定位置 + 期望法线 + 半径/容差，空间哈希邻域搜最近合适面片。
  - `FindPath`：`FindPathAStar`（面片邻接图上的 A\*，二分插入的开放表）→ `FunnelPath`（把跨面片路径“展开”到平面后做漏斗收窄，处理法线变化的转折点）→ `SmoothPath`（去除内凹拐角、按权重平滑法线），输出 `TArray<FWallClimbingPathNode>`（位置 + 法线）。
- **注册与静态工具**：`UWallclimbingNavigationVolumeSet`（`Game::GetSingleton` 单例）收集所有卷；`Wallclimbing::` 命名空间（`WallclimbingStatics.as`）提供 `GetNavigationVolume / FindLocationOnNavmesh / HasStraightPath` 等便捷入口。
- **组件与追踪**：`UWallclimbingComponent`（每个攀墙 AI 的容器，持有当前 `Navigation` 卷、`PreferredGravity`、当前 `Path`）；`UClimbableWallTrackerComponent` + `WallTracker::GetNearestWallNormal`（球体重叠取最近墙面法线，用于生成朝向）。

### 与其他模块的协同

- 建立在 Haze 引擎 `UHazeMovementComponent`、导航八叉树（`Navigation::NavOctree*`）与引擎 NavMesh（`UNavigationSystemV1`）之上。
- 遵循能力四件套 + `n"Pathfinding"` 标签（DesignPhilosophy），组件容器 `UPathfollowingMoveToComponent` / `UWallclimbingComponent` 用 `GetOrCreate` 惰性挂载。
- 只在控制端算路，路径与状态通过既有移动同步（如 `ApplyCrumbSyncedGroundMovement`）在远端复现。

### 关键文件

- `Gameplay/Navigation/Pathfollowing/PathfollowingMoveToComponent.as` — MoveTo 请求队列与状态机
- `Gameplay/Navigation/Pathfollowing/PathfollowingTypes.as` — 状态/优先级/结果枚举与委托
- `Gameplay/Navigation/Pathfollowing/Ground|Flying|Wallclimbing/*PathfollowingCapability.as` — 三套后端
- `Gameplay/Navigation/Wallclimbing/WallClimbingNavigationVolume.as` — 攀墙 navmesh 烘焙 + A\* + 漏斗
- `Gameplay/Navigation/Wallclimbing/WallclimbingComponent.as` / `WallclimbingStatics.as` / `WallclimbingNavigationSet.as`

---

## 2. Triggers — 玩法触发体

**职责与文件数**：10 个文件。一批放置在关卡中的触发体积/组件，用于在玩家进入/看向/靠近时驱动 VO、镜头震动、继承移动、屏蔽行走等玩法反馈。多数遵循“**控制端判定 + CrumbFunction 同步到远端**”的联网模式。

关键类型：

- `APlayerLookAtTrigger` + `UPlayerLookAtTriggerComponent`：**看向触发**。组件按玩家聚焦位置做距离、视野中心占比（`ViewCenterFraction`）、能力标签、自定义 `UHazePlayerCondition`、可选包围体（`TriggerVolume`）多重判定；满足 `LookDuration` 后广播 `OnBeginLookAt/OnEndLookAt`。三种复制策略 `EPlayerLookAtTriggerReplication`（`Local / Crumb / NetFunction`），并按玩家距离动态调节 `TickInterval` 省开销。
- `AVoxPlayerTrigger`（`AVolume`，软弃用，建议用 `AVoxAdvancedPlayerTrigger`）与 `AVoxAdvancedPlayerTrigger`：**VO 触发体**。按 `EVoxPlayerTriggerType`（任一/双人/仅一人/双人访问过等多种组合）驱动 `UVoxTriggerComponent` 播放 Mio/Zoe 各自的 VO 资源；支持按 `FInstigator` 启停、经 `UHazeVoxController` 注册启停回调。另有 `VoxDuoPlayerTrigger` / `VoxPlayerLookAtTrigger`。
- `UPlayerWorldRumbleTrigger`（`UHazeMovablePlayerTriggerComponent`）：**世界震动**。定义相机抖动类 + 力反馈，玩家进入范围后按到中心距离曲线 `DistanceShakeScale` 缩放强度；进入的震动登记到玩家侧 `UWorldRumbleContainerComponent`，由 `UWorldRumbleUpdateCapability` 统一驱动 `PlayCameraShake` 与 `SetFrameForceFeedback`。
- `APlayerInheritMovementZone`：包裹 `UPlayerInheritMovementComponent` 的便捷 Actor，配置好“落地后进入形状范围即继承被撞平台移动”的默认参数。
- `APlayerUnwalkableZone`：包裹 `UPlayerUnwalkableTriggerComponent`，标记不可行走区域。

### 与其他模块的协同

- 触发结果多以 `event` 委托对外广播，供关卡蓝图/序列器接线；VO 类走 Kuro/Haze VO 系统（`UHazeVoxController`、`SoundDef`）。
- 震动类把状态收进玩家侧容器组件再由能力消费，是 DesignPhilosophy“组件容器 + 能力”范式的典型；相机/力反馈标签为 `CameraTags::Camera` 等。

### 关键文件

- `Gameplay/Triggers/PlayerLookAtTrigger.as` / `PlayerLookAtTriggerComponent.as`
- `Gameplay/Triggers/VoxPlayerTrigger.as` / `VoxAdvancedPlayerTrigger.as`
- `Gameplay/Triggers/PlayerWorldRumbleTrigger.as`
- `Gameplay/Triggers/PlayerInheritMovementZone.as` / `PlayerUnwalkableZone.as`

---

## 3. PlayerHealth — 环境伤害触发与世界血条

**职责与文件数**：3 个文件。关卡侧的“伤害/致死”触发体与世界空间血条组件。

> 与 `Core/PlayerHealth` 的区别：Core 侧是玩家**血量系统本体**（`UPlayerHealthComponent`、死亡/复活/伤害屏效等）；本模块是**关卡放置的触发器**，只是调用玩家的 `DamagePlayerHealth / KillPlayer` 等接口来施加伤害，并不管理血量数据。

关键类型：

- `UDamageTriggerComponent`（`UHazeMovablePlayerTriggerComponent`）：玩家在体积内时按 `DamageInterval` 周期造成 `DamageAmount`（可批量 DoT）。丰富的附加效果：击退（`AddKnockbackImpulse`，方向可在“远离触发器 ↔ 触发器前向”间混合）、击倒（`ApplyKnockdown`）、踉跄（`ApplyStumble`）、伤害/死亡特效类。支持按玩家、按 `FInstigator` 启停与 `StartDisabled`。
- `UDeathOnImpactComponent`（`UMovementImpactCallbackComponent`）：玩家以移动撞击（地面/墙壁/天花板可分别开关）此物时直接 `KillPlayer`。默认本地触发。
- `UHealthBarInWorldComponent`（`USceneComponent`）：在世界空间挂 `UHealthBarWidget` 血条，按 `PlayerVisibility` 决定对哪名玩家显示、`bOnlyShowWhenHurt` 控制受伤才显示，每帧同步 `CurrentHealth/MaxHealth`。

### 与其他模块的协同

- 依赖 Core 玩家血量 API（`DamagePlayerHealth / DealBatchedDamageOverTime / KillPlayer / FPlayerDeathDamageParams / UDamageEffect / UDeathEffect`）。
- 复用 Haze 触发基类 `UHazeMovablePlayerTriggerComponent`（进入/离开回调 `OnPlayerEnteredTrigger/OnPlayerLeftTrigger`）；血条走玩家 Widget 系统（`AddWidget/RemoveWidget`）。

### 关键文件

- `Gameplay/PlayerHealth/DamageTriggerComponent.as`
- `Gameplay/PlayerHealth/DeathOnImpactComponent.as`
- `Gameplay/PlayerHealth/HealthBarInWorldComponent.as`

---

## 4. Physics — 网络物理对象

**职责与文件数**：1 个文件。`ANetworkedPhysicsObject : AHazeActor` —— 一个物理模拟且在双人网络下能动态切换控制权与同步率的可推动物体。

关键点：

- 组合 `UStaticMeshComponent`（`SimulatePhysics`）+ `UHazeCrumbSyncedActorPositionComponent`（位置同步）+ `UDisableComponent`（超距自动禁用）。
- `DefaultControlSide`（`Host / Mio / Zoe`）决定默认控制方；`bSwitchControlSideWhenPlayersWithinRelevancyDistance` 时按玩家到物体距离在 `PlayerRelevancyDistance` 内动态把控制权交给更近的玩家（速度非零时不切换，避免打断飞行中的物体）。
- `bIncreaseSyncRateWhenPlayersWithinRelevancyDistance` 时玩家靠近提升 `EHazeCrumbSyncRate`。切换控制权通过 `CrumbFunction`（`CrumbSetControlSide`）同步，并据 `HasControl()` 开关物理模拟；非控制端每帧从 `SyncedPosition` 拉取位置/旋转。

### 与其他模块的协同

- 纯建立在 Haze 网络层（`Network::IsGameNetworked/HasWorldControl`、Crumb 同步、`SetActorControlSide`）之上，独立于其他 Gameplay 模块。

### 关键文件

- `Gameplay/Physics/NetworkedPhysicsObject.as`

---

## 5. Accessibility — 自动跳跃/冲刺辅助

**职责与文件数**：3 个文件。为运动操作提供无障碍辅助，核心是“按住跳跃键即自动完成跳→二段跳→空中冲刺”。（另含 `AccessibilityChatWidget` / `AccessibilityTextToSpeechSubsystem`，后者为联网双人的文本转语音辅助，属独立子功能。）

关键类型：

- `UAccessibilityAutoJumpDashCapability`（`UHazePlayerCapability`，`TickGroup = Input`）：受控于 CVar（`Haze.Accessibility.AutoJumpDash_Mio/Zoe`）。激活后驱动状态机 `EAccessibilityAutoJumpState`（`HasJumped → AllowAirJump → AllowAirDash`），按时序自动放行二段跳/冲刺，目的是**最优水平距离**而非高度。
- `UAccessibilityAutoJumpDashComponent`（容器）：存状态与移动组件引用，`TEST` 下注册开发者输入热键切换；`Accessibility::AutoJumpDash::` 命名空间导出 `ShouldAutoAirJump / ShouldAutoAirDash / StopAutoAirJumpDash` 供玩家移动能力查询。

### 与其他模块的协同

- 依赖玩家移动/跳跃能力组件（`UPlayerMovementComponent / UPlayerJumpComponent / UPlayerAirJumpComponent`）；实际的“是否自动触发空中动作”由这些能力回调本模块命名空间函数决定。遵循能力四件套；开关经 `FConsoleVariable` 与设置界面联动。

### 关键文件

- `Gameplay/Accessibility/AccessibilityAutoJumpDash.as`
- `Gameplay/Accessibility/AccessibilityTextToSpeechSubsystem.as` / `AccessibilityChatWidget.as`

---

## 6. GamepadLight — 手柄灯色

**职责与文件数**：1 个文件。`UPlayerGamepadLightCapability`（`UHazePlayerCapability`，`TickGroup = LastDemotable`）把 DualSense 手柄灯条颜色映射到玩家身份色与血量。

关键点：始终激活；每帧据血量把玩家色向黑色插值（血越低越暗），并叠加随血量降低而加快的正弦脉动；复活时闪白、死亡但复活计时中做渐暗过渡。Zoe 使用固定色（`0xff92f400`），Mio 用 `GetColorForPlayer`。通过 `Player.ApplyGamepadLightColor(..., EInstigatePriority::Normal)` 施加，`OnDeactivated` 时 `ClearGamepadLightColor`。

### 与其他模块的协同

- 读取 `UPlayerHealthComponent`（血量/死亡/复活状态）与 `UPlayerDamageScreenEffectComponent`（显示血量覆盖，逻辑“借鉴自” `PlayerDamageScreenEffectsCapability`）；输出走 Haze 手柄灯 API。

### 关键文件

- `Gameplay/GamepadLight/PlayerGamepadLightCapability.as`

---

## 7. Audio — 玩家 VO 触发体积

**职责与文件数**：1 个文件。`APlayerVOTriggerVolume : AHazeVOPlayerTrigger` —— 玩家进入体积时挂载一组 `FSoundDefReference`（VO/呼吸等音频定义），离开时移除，是关卡语音氛围的主要放置工具。

关键点：

- 控制端处理进入/离开（`ActorBeginOverlap/ActorEndOverlap`），经 `CrumbAttachSoundDefs / CrumbRemoveSoundDefs` 同步。可配置 `TriggerForPlayer`、`bTriggerOnlyOnce`、`DeactivationDistanceRange`（离开后按距离延迟移除，需要时才 Tick）。
- SoundDef 可挂到玩家、`OverrideActorOwner` 或专门生成的 `APlayerVOTriggerEventHandlerActor`（`bLinkBothPlayersEventHandlers` 时把双人事件汇入同一 SoundDef）；处理关联 Actor 的 EffectEvent 链接与 Spawner 回调。
- 追踪每玩家在体积内的时间（`FPlayerVOTriggerVolumePlayerData`），绑定 `UPlayerHealthComponent` 的死亡/复活事件并对外广播 `OnPlayerDied/OnPlayerRespawned`。带编辑器细节定制与可视化组件。

### 与其他模块的协同

- 深度依赖 Haze/Kuro 音频系统（`UHazeVOSoundDef`、`SoundDef.SpawnSoundDefAttached`、`EffectEvent::LinkActorToReceiveEffectEventsFrom`）与玩家血量组件；与 Triggers 模块的 Vox 触发是不同实现路径（这里以 SoundDef 挂载为中心）。

### 关键文件

- `Gameplay/Audio/Volumes/PlayerVOTriggerVolume.as`

---

## 8. TopDownPlayerArrow — 俯视玩家位置箭头

**职责与文件数**：2 个文件。俯视视角关卡里在玩家脚下地面显示一个位置指示箭头。

关键类型：

- `ATopDownPlayerArrow`：一个带 `UStaticMeshComponent`（Plane）的简单 Actor，即箭头本体。
- `UTopDownPlayerArrowCapability`（抽象 `UHazePlayerCapability`，`TickGroup = AfterPhysics`）：`Setup` 时按玩家生成对应箭头类（`TPerPlayer<TSubclassOf<...>>`），玩家存活时激活。`TickActive` 中：在地面时箭头贴玩家位置；在空中时向下 `Trace` 找地面并把箭头贴到命中点，未命中则隐藏。玩家死亡隐藏，`OnRemoved` 销毁箭头。

### 与其他模块的协同

- 依赖 `UPlayerMovementComponent`（`IsOnAnyGround`、`MovementWorldUp`）与 Haze Trace（`FHazeTraceSettings.TraceWithPlayer`）。标准能力四件套 + 每玩家资源随能力生命周期创建/销毁。

### 关键文件

- `Gameplay/TopDownPlayerArrow/TopDownPlayerArrow.as`
- `Gameplay/TopDownPlayerArrow/TopDownPlayerArrowCapability.as`

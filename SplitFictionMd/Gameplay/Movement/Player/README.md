# Gameplay / Movement / Player —— 玩家支撑层

> 职责：Player 目录（424 文件）除动作库（[Moves](../Moves/README.md) 166 + [MovesContextual](../MovesContextual/README.md) 165）之外的**支撑子系统**——运动组件核心、相机、输入、抛射、边界、自动跑、标签词典等。它们不是"动作"，而是让动作能跑起来的地基与外围。
>
> 本文覆盖：Component / Camera / Input / LaunchTo / PlayerUtilities / Boundary / AutoRun / Tags / Audio / EffectHandlers / MoveIntoPlayer / MovesLevel / Settings。

---

## Component —— 运动组件核心（11）

整个 Movement 的基石，`UPlayerMovementComponent` 就在这里。

| 文件 | 作用 |
| --- | --- |
| `PlayerMovementComponent.as` | **核心**：Core `UHazeMovementComponent` 的玩家特化子类。附加输入平面锁(`InputPlaneLock`)、MoveIntoPlayer 数据、撞击回调分发(Ground/Wall/Ceiling)、跟随移动数据；死亡时清空累积事件 |
| `PlayerMovementDefaults.as` | 玩家默认运动/求解/重力/步进/扫掠设置（BeginPlay 里 `ApplyDefaultSettings`） |
| `PlayerMovementStatics.as` | 运动查询/操作静态库 |
| `PlayerMovementIncludes.as` | 汇总 include |
| `PlayerInheritMovementComponent.as` / `PlayerInheritVelocityComponent.as` | 继承移动平台/参考系的运动与速度 |
| `PlayerCrumbSyncRelativeComponent.as` | 相对参考系的 Crumb 位置同步 |
| `PlayerKnockBackComponent.as` | 击退状态 |
| `PlayerIgnoreActorCollisionComponent.as` | 临时忽略与特定 Actor 的碰撞 |
| `MovementImpactCallbackComponent.as` | 撞击回调（撞到某表面触发事件） |
| `PlayerUnwalkableTriggerComponent.as` | 不可行走面触发 |

`UPlayerMovementComponent` 是所有动作 Capability 的 `MoveComp`——四件套里 `TickActive` 的 `PrepareMove`/`ApplyMove` 都打在它上面。

---

## Camera —— 运动相机能力（10）

运动过程中对相机的程序化调整（区别于 Core/Camera 的相机系统本体）。

| 文件 | 作用 |
| --- | --- |
| `PlayerAdjustCameraDistanceBySpeedCapability.as` | 随速度拉远相机（速度 300~1000 之间插值弹簧臂设置） |
| `CameraAssistMovementActivationCapability.as` / `CameraAssistUpdateCapability.as` / `CameraAssistUpdateNewCapability.as` | 相机辅助跟随（新旧两套更新逻辑） |
| `PlayerAlignCameraWithWorldUpCapability.as` | 世界上方向变化时对齐相机（重力井/球面重力等） |
| `CameraLookTowardsSpline{Capability,Component,Statics}.as` | 引导相机看向样条方向 |
| `MovementPerspectiveStatics.as` | 运动视角模式（第三人称/横版/分屏）查询 |
| `PlayerCameraSettings.as` | 玩家相机设置 |

这些能力打 `CapabilityTags.Add(n"Camera")`、`DebugCategory = n"Movement"`，通常在 `LastMovement` tick 组，运动求解完成后调整相机。

---

## Input —— 移动输入处理（5）

把手柄摇杆原始输入转成可用的移动意图。

| 文件 | 作用 |
| --- | --- |
| `PlayerMovementSquareDirectionInputCapability.as` | **方形**输入映射（含摇杆回中检测 `FStickSnapbackDetector`） |
| `PlayerMovementOvalDirectionInputCapability.as` | **椭圆**输入映射 |
| `PlayerMovementDirectionFacingCapability.as` | 输入方向 → 角色朝向 |
| `MovementInputStatics.as` | 输入静态库（含 `CapabilityTickGroupOrder` 基准） |
| `MovementInputDebug.as` | 输入调试可视化 |

输入能力跑在 `BeforeMovement` tick 组，是每帧运动链的起点；`UPlayerMovementComponent.MovementCapabilityType` 决定当前用方形还是椭圆映射。

---

## LaunchTo —— 抛射到目标（5）

把玩家程序化地"发射"出去（弹射器、爆炸、剧情抛射）。

| 文件 | 作用 |
| --- | --- |
| `PlayerLaunchToComponent.as` | 抛射状态与参数（`FPlayerLaunchToParameters`：目标点/时长/动画） |
| `PlayerLaunchToMovementCalculator.as` | 抛物线/曲线轨迹计算 |
| `PlayerCrumbedLaunchToCapability.as` | **Crumb 联机**版抛射（`LaunchToPoint` 到点弧线，锁输入） |
| `PlayerLocalLaunchToCapability.as` | **本地模拟**版抛射 |
| `PlayerLaunchToStatics.as` | 触发接口 |

三种抛射类型（`EPlayerLaunchToType`）：`LaunchToPoint`（弧线到点，锁输入）、`LaunchWithImpulse`（给方向冲量、锁输入一段时间）、`LerpToPointWithCurve`（曲线插值到点并留退出速度）。三种联机模式（Crumbed / SimulateLocal / SimulateLocalImmediateTrajectory）权衡"双人一致"与"末端偏差"。

---

## Boundary —— 双人边界约束（3）

Split Fiction 是双人游戏，需约束两名玩家的相对位置。全部实现为 **Core Resolver 扩展**（`UMovementResolverExtension`），挂到运动求解器上直接钳制位移。

| 文件 | 作用 |
| --- | --- |
| `MaximumDistanceBetweenPlayersResolverExtension.as` | 两名玩家最大间距约束 |
| `LimitLeadingPlayerDistanceAlongSplineResolverExtension.as` | 沿样条限制"领跑者"超前距离 |
| `CameraFrustumBoundaryResolverExtension.as` | 把玩家约束在相机视锥内 |

机制与 [ResolverExtensions](../ResolverExtensions/README.md) 相同：`PrepareExtension` 取组件设置 → 每帧迭代里改写 `MovementDelta`。支持全部四种求解器（Simple/Stepping/Sweeping/Teleporting）。

---

## AutoRun —— 自动奔跑（3）

过场/引导段落让玩家自动前进。

| 文件 | 作用 |
| --- | --- |
| `PlayerAutoRunComponent.as` | 当前自动跑配置（`ActiveAutoRun`，`TInstigated`） |
| `PlayerAutoRunCapability.as` | 有激活配置时接管移动输入（`BeforeMovement` 组），可与冲刺叠加 |
| `PlayerAutoRunStatics.as` | 启停接口 |

`ShouldActivate` 就看 `AutoRunComp.ActiveAutoRun` 是否为默认值——外部设了目标就自动跑。

---

## PlayerUtilities —— 杂项玩家能力（28）

不属于某个具体动作的通用玩家功能，最大一块是 **CenterView（居中视角）**。

| 组 | 文件 | 作用 |
| --- | --- | --- |
| CenterView 核心 | `CenterView{Component,Settings,Statics}.as` + `CenterViewForwardPlayerCapability.as` + `CenterViewTarget*Capability.as`（Hold/Rotate/Tap/Toggle）+ `CenterViewTargetComponent.as` | 把视角居中/锁向目标（点按/长按/切换多种交互） |
| CenterView 教学 | `CenterViewTutorial*.as` + `CenterView{Hard,Soft}Lock*Capability.as` + `CenterView{Hold,Tap}ToCenter*.as` | 居中视角的教学引导变体 |
| 找队友 | `FindOtherPlayerCapability.as` + `OtherPlayerIndicator{Capability,Component,Widget}.as` | 屏幕外队友方向指示器 |
| 落向样条 | `FallTowardSplineZone.as` + `FallTowardSplineZoneVisualizer.as` + `FallTowardsSplineCapability.as` | 下落时引导玩家落向样条 |
| 其它 | `PlayerBlobShadowCapability.as`、`PlayerSyncLocationMeshOffsetCapability.as`、`PlayerInputPlaneLockTrigger.as` | Blob 阴影、网格偏移同步、输入平面锁触发体 |

---

## Tags —— 标签词典（3）

全模块标签矩阵的"字典表"，被所有动作 `default CapabilityTags.Add(...)` 引用。

| 文件 | 内容 |
| --- | --- |
| `BlockedWhileInTags.as` | `BlockedWhileIn::` 约 40 个"状态词"（Core Moves / Contextual Moves / Level Abilities 三段） |
| `PlayerMovementTags.as` | `PlayerMovementTags::` 分类词（CoreMovement/Dash/AirDash/Grapple…）+ `PlayerMovementExclusionTags::` 例外词 |
| `MovementDevToggles.as` | 开发期调试开关（如 `Dash::AutoAlwaysDash`、`Move::AutoRunInCircles`） |

标签矩阵运转机制见 [Moves/README.md](../Moves/README.md) 的"标签矩阵如何工作"。

---

## 其余支撑子目录

| 子目录 | 文件 | 作用 |
| --- | --- | --- |
| **Audio** (7) | `PlayerProxyEmitterSpatialization*.as`、`Player{Fullscreen,Sidescroller}ListenerCapability.as`、`SplitTraversal*.as` | 玩家音频代理发声与监听器的空间化（第三人称/横版/分屏/全屏各一套） |
| **EffectHandlers** (2) | `PlayerCoreMovementEffectHandler.as`、`PlayerSwimmingEffectHandler.as` | 运动特效事件分发中枢（`Trigger_AirDash_Started/Stopped` 等由动作 Capability 调用） |
| **MoveIntoPlayer** (6) | `MoveIntoPlayerMovement*.as`、`MoveIntoPlayerShape*.as`、`MoveIntoPlayerRotating*.as` | 让外部物体"移动进入玩家"的运动数据 + 求解器 + 形状匹配 + 旋转对齐（`UPlayerMovementComponent` 内建此数据） |
| **MovesLevel** (4) | `PlayerPolevault{Capability,ChargeCapability,AnticipationCapability,Component}.as` | **关卡专属动作**：撑杆跳（充能/预备/执行），只在特定关卡装配 |
| **Settings** (1) | `PlayerWallSettings.as` | 玩家墙面交互通用设置 |

---

## 关键文件

| 文件 | 作用 |
| --- | --- |
| `Component/PlayerMovementComponent.as` | 玩家运动组件（Core `UHazeMovementComponent` 特化），所有动作的 `MoveComp` |
| `Component/PlayerMovementDefaults.as` | 玩家默认运动/求解/重力设置 |
| `Tags/BlockedWhileInTags.as` + `Tags/PlayerMovementTags.as` | 标签矩阵词典 |
| `EffectHandlers/PlayerCoreMovementEffectHandler.as` | 运动特效事件分发 |
| `Boundary/MaximumDistanceBetweenPlayersResolverExtension.as` | 双人间距约束（Resolver 扩展范式） |
</content>

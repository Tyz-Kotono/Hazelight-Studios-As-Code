# Core / Camera / Camera

> 职责：相机系统主体——全局分屏/全屏混合单例、每位玩家的相机用户组件、各类相机类型与 Actor、视点混合、聚焦目标、兴趣点、能力、设置、卷体与调试。共 79 文件（含 17 个子目录）。

## 内部架构

整体遵循 **Actor → Component → Updater** 三段式：`AHazeCameraActor` 摆放体持有一个 `UHazeCameraComponent`（相机类型），其每帧位姿由配套 `UHazeCameraUpdater` 计算；相机 Actor 的 `PrepareUpdaterForUser` 负责把数据注入 Updater。每位玩家持有一份 `UHazeCameraUserComponent`（本项目派生为 `UCameraUserComponent`），承载输入、期望旋转、联机同步与设置。

### 核心类

| 类 | 基类 | 角色 |
| --- | --- | --- |
| `UCameraSingleton` | `UHazeCameraSingleton` | 全局单例，驱动分屏↔全屏的相机/投影混合（`FCameraBlendToSplitScreenData` / `FCameraBlendToFullScreenData`） |
| `UCameraUserComponent` | `UHazeCameraUserComponent` | 每位玩家的相机控制中枢：期望旋转、输入加速、转向速率、快照、联机同步 |

### 子目录与关键类

| 子目录 | 关键类 / 基类 | 职责 |
| --- | --- | --- |
| `Blend/` | `UCameraDefaultBlend` / `UCameraCurveControlBlend` / `UCameraFollowActorBlend` / `UCameraOrbitBlend`（均派生 `UHazeCameraViewPointBlendType`）；`CameraBlend` 命名空间 | 无状态的视点混合算法资产（线性/缓动/曲线/轨道/跟随） |
| `CameraActors/` | `ABallSocketCameraActor` / `AFocusCameraActor` / `APivotCameraActor` / `ASplineFollowCameraActor` / `ASplineFollowCustomRotationCameraActor` / `AStaticCameraActor`（均派生 `AHazeCameraActor`） | 可摆放相机体，重写 `PrepareUpdaterForUser` 注入数据 |
| `CameraTypes/` | 相机组件 `USpringArmCamera` / `UFocusTargetCamera` / `USplineFollowCamera` / `UBallSocketCamera` / `UFocusTargetCustomRotationCamera`，各配套 `UCameraXxxUpdater` | 相机组件 + 每帧位姿 Updater（弹簧臂/聚焦/样条/球窝） |
| `CameraFocusTarget/` | `UCameraWeightedTargetComponent` / `UCameraWeightedTargetOptionalComponent`（派生 `UHazeCameraResponseComponent`）；数据 `FFocusTargets` / `FHazeCameraWeightedFocusTargetInfo` | 加权聚焦目标的数据与持有组件，供 Updater 取「保持入框」位置 |
| `User/` | `UCameraControlCapability` / `UCameraUpdateCapability` / `UCameraControlReplicationCapability` / `UCameraPanFocusCameraCapability` / `UCameraVolumePlayerConditionCapability`（Capability）；`CameraPlayerStatics` / `CameraUserStatics`（mixin） | 用户侧能力：读取摇杆输入、驱动相机更新、复制旋转、聚焦相机平移 |
| `PointOfInterest/` | `UCameraPointOfInterest` / `UCameraPointOfInterestClamped`（派生 `UHazePointOfInterestBase`）；`UCameraPointOfInterestSettingsComponent`；`PointOfInterestStatics` | 兴趣点看向系统，支持清除/暂停/输入打断 |
| `ChaseAssistance/` | `UCameraAssistComponent`（`UActorComponent`）；`UCameraAssistType` / `UCameraFollowAssistTypeNew`（`UDataAsset`）；`UPlayerCameraAssistSettings` | 相机追随辅助（按速度/视角自动跟随玩家朝向） |
| `Modifiers/` | `UCameraImpulseCapability` / `UCameraModifierCapability`（`UHazeMarkerCapability`）；`UCameraImpulseSettings` | 冲量与修饰器（震动/动画/后处理）的门控开关 |
| `NonControlledRotation/` | `UCameraNonControlledTransitionCapability` / `UCameraMatchOthersCutsceneRotationCapability`（`UHazeCapability`） | 从非受控相机平滑过渡回受控相机、匹配另一玩家的过场视角 |
| `HideOverlappers/` | `UCameraHideOverlappersCapability`（`UHazeCapability`） | 通过射线检测隐藏遮挡相机视线的物体 |
| `FollowSplineRotation/` | `UCameraFollowSplineRotationComponent`（`UActorComponent`）；`ACameraFollowSplineRotationTrigger`（`APlayerTrigger`） | 随玩家沿样条移动旋转期望相机朝向 |
| `Settings/` | `UCameraUserSettings` / `UControlRotationSettings` / `UCameraInheritMovementSettings`（均 `UHazeComposableSettings`） | 可组合相机参数数据容器 |
| `Volumes/` | `ABothPlayerCameraVolume`（`AVolume`）；`ACameraInheritMovementVolume` / `AFocusCameraPanningVolume`（`APlayerTrigger`） | 用卷体在进入/双人入卷时推送相机设置 |
| `SplineFollowCustomRotation/` (+`FocusBlend/`) | `USplineFollowCustomRotationCameraResponseComponent`；`USegmentedSplineFocusCameraBlendComponent`；`USplineFocusCameraBlendCapability`（`UHazePlayerCapability`） | 沿样条分段插值相机的 FOV/灵敏度/入框/聚焦目标 |
| `CameraTags.as` | `namespace CameraTags` | 集中定义所有相机能力标签 FName 常量 |
| `Debug/` | `UCameraDevMenu` / `ADebugCameraActor` / `UDebugCameraCapability` 等 | 开发期调试相机、自由飞行、日志、动画检视 |
| `Deprecated/` | `AFocusTrackerCameraActor` / `AKeepInViewCameraActor` 等 | 已弃用的旧相机 Actor 与迁移辅助 |

### 内部数据流

1. **输入 → 期望旋转**：`UCameraControlCapability` 读摇杆，经 `UCameraUserComponent` 的加速曲线积分（`EvaluateDiscretizedAccelerationCurve`，做帧率无关处理）转成 `AddUserInputDeltaRotation`，写入 `InternalLocalDesiredRotation`。
2. **联机同步**：有控制端把期望旋转写入 `UHazeCrumbSyncedCameraComponent`；无控制端从中读取（见 `OnPostUpdate` 的 `HasControl()` 分支）。
3. **每帧位姿**：相机 Actor `PrepareUpdaterForUser` → Updater `OnCameraUpdate`，结合聚焦目标、设置、混合产出最终视点。
4. **分屏/全屏**：`UCameraSingleton.BlendToFullScreenUsingProjectionOffset` / `BlendToSplitScreenUsingProjectionOffset` 用 `AStaticCameraActor` 占位 + `SceneView` 分屏分隔/投影偏移逐帧推进。

## 协同关系

> 基于对整个 Core 与 Gameplay/LevelSpecific 的 grep 实证。

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 被调用 | Aiming / Audio Listeners / Movement / 数十个关卡 Capability | 1 | `UCameraUserComponent::Get(Player)` 是全项目最热的相机入口（grep 命中 40+ 文件）：取转向/灵敏度、设期望旋转、触发快照 |
| 被调用 | `UCameraSingleton` ← 引擎分屏逻辑 | 1 | `BlendToFullScreen/SplitScreen...` 由引擎 BlueprintOverride 调度 |
| 调用 | `AHazePlayerCharacter` | 1 | `ActivateCamera` / `DeactivateCameraByInstigator` / `ApplyViewSizeOverride` 切换与混合相机 |
| 调用 | `UHazeCrumbSyncedCameraComponent` | 2 | 期望旋转的联机同步（`UpdateValues` / `GetDesiredRotation` / `TransitionSync` / `BlockSync`） |
| 调用 | `UCameraXxxSettings`（Settings 子目录） | 5 | `ApplySettings` / `GetSettings`：用户设置、控制旋转、冲量、聚焦等分层参数 |
| 调用 | `UHazeCameraModifierManager` | 1+3 | Modifiers 能力按标签门控并 Block/Unblock 震动与修饰器 |
| 内部 | `CameraTags::*` | 3 | 各 Capability 用 `CameraTags::Camera` 等标签做隐式互斥与归类 |
| 内部 | Volumes → 相机设置 | 2+4+5 | 双人入卷用 `CrumbFunction` 同步，`OnBothEntered` 等委托广播，再 `Apply/Clear` 设置 |
| 内部 | HideOverlappers / 各 Trigger | 4 | `User.OnReset.AddUFunction` / `OnPlayerEnter.AddUFunction` 等委托订阅 |
| 被依赖 | `CameraBoundaryComponents` | 1 | 该并列模块强依赖本模块的 `ASplineFollowCameraActor` 与 `FocusTargetComponent` |

## 关键文件

- `CameraSingleton.as` —— 全局单例，分屏↔全屏相机与投影偏移混合的状态机。
- `User/CameraUserComponent.as` —— 每位玩家相机中枢：输入加速、期望/基准旋转、转向速率、快照、Crumb 同步。
- `CameraTags.as` —— 所有相机能力标签常量的集中定义。
- `User/` —— 用户侧能力与静态助手（控制输入、更新门控、复制、聚焦平移、卷体条件）。
- `CameraTypes/` + `CameraActors/` —— 相机类型组件、其 Updater 与对应可摆放 Actor。
- `Blend/` —— 无状态视点混合算法资产。
- `CameraFocusTarget/` + `PointOfInterest/` —— 加权聚焦目标与兴趣点看向。
- `Settings/` —— 可组合相机参数（`UHazeComposableSettings`）。
- `ChaseAssistance/` / `Modifiers/` / `NonControlledRotation/` / `HideOverlappers/` / `FollowSplineRotation/` —— 各类相机行为能力与设置。
- `Volumes/` —— 卷体驱动的相机设置推送。
- `SplineFollowCustomRotation/`（含 `FocusBlend/`）—— 样条分段相机混合。
- `Debug/` / `Deprecated/` —— 调试工具与遗留相机。

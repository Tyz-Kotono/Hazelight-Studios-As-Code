# LevelSpecific / Meltdown / SplitMechanics（打破分屏机制群）

> 路径：`SplitTraversal/`(99) + `ScreenWalk/`(67) + `SoftSplit/`(43) + `WorldSpin/`(26) + `SplitSlide/`(20)，另含 `ScreenPush/`(2)、`SplitBonanza/`(12)、`FakeSplitRendering/`(1)、`WorldLink/`(2)。合计约 255 文件。

## 职责

Split Fiction 的招牌呈现是**恒定的竖直分屏**——两名玩家永远各占半屏，各在自己的世界里。Meltdown 作为终章，把这条铁律当成可玩内容来解构：科幻与奇幻两个世界"故障熔毁"、边界坍塌，于是这一机制群用五种方式**打破分屏本身**——斜切融合、跨屏穿越、屏中屏行走、整世界翻转、高速滑行合屏。它们不新增渲染管线，而是集中编排 Core 的 `SceneView::` 分屏/相机 API 与相机组件，是本关"画面即玩法"的技术核心。

## 内部架构

### 共同底座：双世界坐标 + `SceneView::` 编排

五套机制共享同一套解法：

- **两个世界并存、用巨型偏移隔开**：科幻/奇幻两个 sublevel 的几何在世界坐标里被一个大常量分开（`SoftSplitOffset(200000,0,0)`、`SplitOffset(500000,0,0)`）。每个 Manager 提供 `Position_FantasyToScifi` / `Position_ScifiToFantasy` / `Position_Convert(Src,Tgt)` 换算，把"另一个世界"的相机/物体/玩家搬进本世界。世界标签是 `EHazeWorldLinkLevel`（Fantasy / SciFi）。
- **只编排 Core，不写渲染**：分屏模式切换 `SceneView::SetSplitScreenMode(EHazeSplitScreenMode::{Vertical / ManualViews / CustomMerge})`；手动多视图 `SetManualViews(FHazeComputedView[])`；视图计算 `ComputeView(FHazeViewParameters)`→`FHazeComputedView`；世界↔屏幕投影 `ProjectWorldToScreenPosition` / `ProjectWorldToViewpointRelativePosition` / `DeprojectScreenToWorld_Relative` / `_Absolute` / `DeprojectScreenToWorldInView_Absolute`；分辨率 `GetFullViewportResolution` / `GetConstrainedViewportResolution`；关卡可见性 `SetLevelRenderedForAnyView` / `SetLevelRenderedForPlayer`。
- **Manager 单例 + 玩家分身**：每套机制一个 `AHazeActor` Manager（挂 `UHazeListedActorComponent`），用 `UHazeRequestCapabilityOnPlayerComponent` 注入能力；用 `PlayerCopy` Actor 在对方世界渲染另一名玩家的影子。

### 五套机制

**SoftSplit（43）——斜切软分屏（招牌中的招牌）**
`ASoftSplitManager : ACustomMergeSplitManager`（Core 的"融合分屏"基类）。它把两个世界揉进**同一块屏幕**：Zoe 用真相机，Mio 侧 `Spawn` 一个 `AStaticCameraActor`（MioCam）用 `Position_FantasyToScifi(Zoe.ViewLocation)` 把 Zoe 的相机镜像进科幻世界。每帧把两名玩家投影到屏幕空间（`ProjectWorldToViewpointRelativePosition`），求中点 `CenterInScreenSpace` 与朝向 `ScreenDirectionRotation`，喂给基类的后处理融合材质 `SplitScreenInstance`：`SetVectorParameterValue("ScreenCenterPosition"/"ScreenSplitDirection")`、`SetScalarParameterValue("WobbleIntensity"/"BorderSize")`——于是分屏线不再竖直，而是**跟随两名玩家斜切、抖动**。配套：off-center 投影 `UHazeCameraViewPoint::ApplyOffCenterProjectionOffset`、`USoftSplitFakeLetterbox`（假黑边）、`ASoftSplitPlayerCopy`（+`USoftSplitMeshRenderingComponent`）、并主动关闭 Core `URenderingSettingsSingleton` 的 SSR/SSS（因为等于全屏渲染两遍）。障碍物做成"双份"变体（`SoftSplitMovingDoubleObstacle` / `...RotatingDoubleObstacle` / `SoftSplitDestructionDoubleActor`），同时存在于两个世界。

**SplitTraversal（99）——跨屏穿越**
`ASplitTraversalManager : AHazeActor`。核心是让物体/玩家**从半屏物理地穿到另一半屏**：分屏线在 `SplitPosition = 0.5`，用 `SceneView::GetPlayerScreenAtScreenPosition` 判断某点落在谁的屏幕，用 `ProjectWorldToScreenPosition` + `DeprojectScreenToWorldInView_Absolute` 把位置跨越隔离偏移换算。可在 `ManualViews`（`ActivateSplitSlide`：两屏共享一个 `AStaticCameraActor DuplicatedCamera`）与 `Vertical` 间切换。`ASplitTraversalPlayerCopy`（+`USplitTraversalMeshRenderingComponent`、`USplitTraversalCopyPartsCapability`）渲染对侧分身；`SplitTraversalThrowable` 抛物越界时广播 `FThrowableCrossedSplitScreen`，被金像手雷/能量核心/旋转猫像等消费。其余大量文件是穿越关卡里的谜题道具（`Turret`、`Pushable(Mushroom)`、`DragableFloatingPlatform`、`WateringCan`、`CarnivorousPlant`、`FlyingUmbrella`、`RevealingStaircase`、`Throwable` 等）。

**ScreenWalk（67）——屏中屏行走**
`AMeltdownScreenWalkManager : AHazeActor`。一名玩家在另一名玩家的"屏幕"上行走的透视机制：强制 `Vertical` 分屏，给 Zoe 施加 `ApplyViewSizeOverride(Fullscreen)` + `TopDown` 视角，同时用 `USceneCaptureComponent2D` 把 Mio 的视图（用其真实 `FHazeComputedView::ProjectionMatrix`）渲进 `UTextureRenderTarget2D`，再把这张 RT 贴到 `AMeltdownScreenWalkDisplayPlane`（材质参数 `TiltSceneTexture`）——于是 Mio 的画面成了 Zoe 脚下可走的平面。`ProjectSeethrough_OutsideToInside` / `_InsideToOutside` 用 `ProjectWorldToViewUV` / `DeprojectViewUVToWorld` 在外部世界与屏面之间换算坐标。配套能力 `MeltdownScreenWalkStompCapability` / `TransitionCapability`（`UHazePlayerCapability`）、组件 `...UserComponent` / `...ResponseComponent` / `...TransitionComponent`，及一批传送带/平台/打地鼠道具。

**WorldSpin（26）——整世界翻转**
`AMeltdownWorldSpinManager : AHazeActor`（同样持 `USceneCaptureComponent2D` + RT + `TPerPlayer<FHazeComputedView>`）。`UMeltdownWorldSpinnerPlayerCapability : UHazePlayerCapability` 让"旋转者"整体旋转共享世界（旋转角用 `UHazeCrumbSyncedFloatComponent` 联网同步），并对另一名玩家施加 `GravityDirectionOverride`——一人转动世界与重力、另一人在其中穿行；连天空 `AGameSky` 与方向光一起转。`UMeltdownWorldSpinSettings : UHazeComposableSettings` 存参数。

**SplitSlide（20）——高速滑行合屏**
高速滑板/滑水追逐段。`AMeltdownSplitSlideCameraTarget : AHazeActor` 驱动相机，`USplitSlideTransitionToHoverboardPlayerCapability : UHazePlayerCapability` 把玩家过渡到悬浮板；`MeltdownEffectLineWidget` 用 `ProjectWorldToScreenPosition` 在屏上画分屏线特效。段落本身大量走 SplitTraversal 的 `ActivateSplitSlide`（ManualViews 共享相机）路径。场景演出 Actor（`ShipDragon`、`Shark`、`CarFish`、`MissileBird`、`Tank`、`WhiteSpaceRift`、坍塌桥）均为 `AHazeActor`。

### 同系机制（同一套 `SceneView::` 手法，非本篇重点）
- `ScreenPush/MeltdownScreenPushManager.as`——全关 `SceneView::` 用得最重的文件（`SetLevelRenderedForAnyView` / `GetConstrainedViewportResolution` / `SetManualViews` / `ChangeSplitScreenPosition(ThirdScreen)`）。
- `SplitBonanza/SplitBonanzaManager.as`——`SetHazeGlobalTextureForViewPosition` + `ComputeView`。
- `Ending/MeltdownEndingManager.as`、`Underwater/MeltdownUnderwaterManager.as`——同样的 `ManualViews` + `ComputeView` 模式。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| → | Core `SceneView::` 分屏 API | 直接调用 | `SetSplitScreenMode(Vertical/ManualViews/CustomMerge)`、`SetManualViews`：SplitTraversalManager(336/508/582/688)、ScreenWalkManager(73/118)、SoftSplit(经基类) |
| → | Core `ACustomMergeSplitManager` | 继承 | `ASoftSplitManager : ACustomMergeSplitManager`（`Core/SceneView/CustomMergeSplitManager.as`），复用其 `ActivateCustomSplit` + 融合材质 `SplitScreenInstance` |
| → | Core 相机视点 | 投影控制 | `UHazeCameraViewPoint::ApplyOffCenterProjectionOffset` / `Clear...`（SoftSplit 斜切核心）、`ApplyViewSizeOverride(Fullscreen)`（ScreenWalk） |
| → | Core 场景捕获/渲染 | 屏中屏 | ScreenWalk/WorldSpin 用 `USceneCaptureComponent2D` + `Rendering::CreateRenderTarget2D` + `FHazeComputedView::ProjectionMatrix` |
| → | Core 渲染设置单例 | 性能门控 | SoftSplit `URenderingSettingsSingleton.EnableScreenSpaceReflections/SubsurfaceScattering` 关闭（全屏渲染两遍） |
| → | Core 相机 | 生成镜像相机 | SoftSplit / SplitTraversal `Spawn` `AStaticCameraActor` 做相机镜像/复制 |
| → | Gameplay 移动系统 | 重力覆盖 | WorldSpin 对另一玩家施 `GravityDirectionOverride`，联动 Core 移动管线 |
| ↔ | Core Crumb 联机 | 状态同步 | WorldSpin 旋转角走 `UHazeCrumbSyncedFloatComponent`；穿越/滑行能力 `NetworkMode=Crumb` |
| → | 世界坐标换算 | 跨屏对象 | 所有 Manager 的 `Position_Convert` + `EHazeWorldLinkLevel`；抛物越界事件 `FThrowableCrossedSplitScreen` |
| → | 玩家分身渲染 | 副本 | `ASoftSplitPlayerCopy` / `ASplitTraversalPlayerCopy`（+`MeshRenderingComponent`）在对侧世界渲染玩家 |

## 关键文件

- `SoftSplit/SoftSplitManager.as` — `ASoftSplitManager : ACustomMergeSplitManager`，斜切软分屏核心（坐标换算 + 融合材质参数驱动）
- `SplitTraversal/SplitTraversalManager.as` — `ASplitTraversalManager`，跨屏穿越 + `ActivateSplitSlide`（ManualViews 共享相机）
- `ScreenWalk/MeltdownScreenWalkManager.as` — `AMeltdownScreenWalkManager`，`SceneCapture`→RT→屏面贴图的屏中屏行走
- `WorldSpin/MeltdownWorldSpinManager.as` + `WorldSpin/MeltdownWorldSpinnerPlayerCapability.as` — 整世界+重力翻转
- `SplitSlide/MeltdownSplitSlideCameraTarget.as`、`SplitSlide/MeltdownEffectLineWidget.as` — 高速滑行相机与分屏线特效
- `../../Core/SceneView/CustomMergeSplitManager.as` — Core 融合分屏基类（`SetSplitScreenMode(CustomMerge)` + 每玩家 CustomMerge 材质 + MergeMask RT）
- `ScreenPush/MeltdownScreenPushManager.as` — 同系机制中 `SceneView::` 使用最全的参考实现

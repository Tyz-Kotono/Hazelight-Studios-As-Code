# Core / Rendering / SceneView

> 职责：双人分屏的视图布局管理——在垂直分屏、手动多屏视口、自定义合并蒙版、对角分屏、全屏叠加场景之间切换，并为第 3~5 路视图托管额外相机用户。共 5 个文件。

## 内部架构

各 Manager 都是 `AHazeActor`，通过 `SceneView::` 命名空间向引擎下达分屏模式与材质，多数与 Camera 域单例协同（投影偏移、相机激活）。

- **`AMultiScreenSplitManager`**：手动多视口布局。维护 `TArray<FHazeManualView>` 视口矩形，支持 `SnapViews`/`BlendViews`（`FManualViewBlend` 源→目标插值），`SetManualViews` + `SetSplitScreenMode(ManualViews)`。`ActivateCameraOnView` 为 0/1 路用玩家相机、2~4 路按需 `SpawnActor(AMultiScreenCameraUser)` 并指定 `EHazeSplitScreenPosition::Third/Fourth/FifthScreen`。
- **`ACustomMergeSplitManager`**：自定义合并基类。`ActivateCustomSplit` 设 `SetSplitScreenMode(CustomMerge)`、创建全屏分辨率的 `MergeMask` RenderTarget、为每位玩家建 `CustomMergeMaterial` MID 并 `SetSplitScreenCustomMergeMaterial`，同时给相机视点 `ApplyOffCenterProjectionOffset`；`Tick` 每帧把 `SplitScreenMaterial` 绘进 `MergeMask`。
- **`ADiagonalSplitManager : ACustomMergeSplitManager`**：对角分屏特化。`SetDiagonalAngle`/`SetSplitPercentage` 驱动 `OnStateUpdated` 计算分割线偏移，写分屏材质参数并为双人施加方向相关的离心投影偏移；可挂 `UDiagonalSplitOverlayWidget` 在全屏画分割线。
- **`ASplitOverlaySceneManager`**：用一个偏远处 `USceneCaptureComponent2D`（`PRM_UseShowOnlyList`）只渲染指定 Actor 到 `OverlayRT`，再用全屏 `USplitOverlaySceneWidget` 叠加显示；`USplitOverlayScreenSpacePositionComponent` 让 Actor 按屏幕坐标/深度摆到捕获偏移处。
- **`AMultiScreenCameraUser : AHazeAdditionalCameraUser`**：附加相机用户，承载第 3~5 路视图的相机组件。

数据流：`Manager 激活 → SceneView::SetSplitScreenMode / SetManualViews / SetSplitScreenCustomMergeMaterial → 相机视点投影偏移 + RenderTarget 蒙版/叠加 → 渲染`。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
| --- | --- | --- | --- |
| 调用 | 引擎 SceneView | 直接调用 | `SceneView::SetSplitScreenMode`、`SetManualViews`、`SetSplitScreenCustomMergeMaterial`、`GetFullViewportResolution` |
| 调用 | Camera 域 | 直接调用 | `Player.ActivateCamera`、`UHazeCameraViewPoint.ApplyOffCenterProjectionOffset/Clear`（分屏与相机协同） |
| 被调用 | 关卡管理器 | 直接调用 | Meltdown 的 SoftSplitManager / ScreenPushManager / WorldSpinManager 引用 `SceneView::` |
| 调用 | GUI | 直接调用 | `Widget::AddFullscreenWidget/RemoveFullscreenWidget` 叠加分割线/场景叠加 UI |
| 调用 | 引擎渲染 | 直接调用 | `Rendering::CreateRenderTarget2D/ResizeRenderTarget/DrawMaterialToRenderTarget`、`SceneCapture.CaptureScene` |

## 关键文件

- **MultiScreenSplitManager.as**：`AMultiScreenSplitManager`，手动多视口矩形布局 + 视口混合 + 多路相机激活。
- **CustomMergeSplitManager.as**：`ACustomMergeSplitManager`，自定义合并蒙版基类（MergeMask + 每玩家合并材质 + 离心投影偏移）。
- **DiagonalSplitManager.as**：`ADiagonalSplitManager` 对角分屏特化 + `UDiagonalSplitOverlayWidget` 分割线绘制。
- **SplitOverlaySceneManager.as**：`ASplitOverlaySceneManager` 场景捕获叠加 + 屏幕坐标定位组件 + 叠加 Widget。
- **MultiScreenCameraUser.as**：`AMultiScreenCameraUser`，第 3~5 路视图用的附加相机用户。

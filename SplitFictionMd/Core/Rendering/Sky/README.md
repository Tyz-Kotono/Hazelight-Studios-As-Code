# Core / Rendering / Sky

> 职责：游戏天空与环境光照——平行光、天光、指数高度雾、天穹、风、CSM 级联阴影，并支持光照配置切换、室内感知关级联、全局雨遮罩与雨阻挡。共 4 个文件。

## 内部架构

核心是单例式 `AGameSky`（经 `AGameSky::Get()` 从 `ActorList` 取唯一实例），把多个引擎光照组件聚合到一个 Actor 上，由属性驱动统一配置：

- **`AGameSky : AHazeActor`**（数百属性）：
  - 组件：`UDirectionalLightComponent`（平行光/光轴）、`USkyLightComponent`（天光）、`UExponentialHeightFogComponent`（雾）、`UStaticMeshComponent` Skydome/下半球/太阳、`UGlobalRainComponent`、`UDynamicWaterEffectControllerComponent`。
  - `FHazeLightingConfig` / `FFogParameters`：成套光照配置（可切换、可混合 `SetLightingConfigBlend`），含天光强度、平行光、雾、后处理体、HazeSphere、灯组。
  - `Init()`：把所有参数推到光照组件、Skydome MID（`SkyMaterialDynamic`）、`GlobalParameters`（SunColor/SunDirection/海洋不透明度…）、`WindParameters`（风强/阵风/风向）。
  - **CSM 级联**：`ESkyCascadeShadowSettings` × `Haze.GameSkyShadowCascadeQuality` 决定级联数与距离；`UpdateCascadeSettings()` 每帧按每位玩家相机-玩家距离缩放 CSM（`SceneView::SetCSMDistanceScaleForView`），并查询室内卷体决定是否关级联/平行光。
  - 暖屏/白空间（Whitespace/Glitch）、备用光照混合（`SetAlternateLightingEnabled*`）。
  - 编辑器内经 `UGameSkyTickInEditorComponent` 在 sequencer 中更新。
- **`UGlobalRainComponent : UActorComponent`**：场景捕获深度生成雨遮罩 RenderTarget（2048²），叠加 `ARainBlocker` 绘制的强制加/去雨区域，再 `SceneView::SetHazeGlobalTextureForViewPosition` 设为各分屏视口全局纹理；参数写 `GlobalParameters`（GlobalRain*）。
- **`AIndoorSkyLightingVolume : AVolume`**：室内卷体；`GetIndoorSkyLightingSettings()` 静态查询所有卷体（支持反转卷体语义），输出「是否关级联 / 关平行光」，供 `AGameSky` 调用。
- **`ARainBlocker : AHazeActor`** + `URainBlockerComponent`：带优先级（`opCmp` 排序）的强制加雨/去雨方块，被 `UGlobalRainComponent.CaptureRainMask` 绘进遮罩。

数据流：`属性/光照配置 → Init/Tick → 光照组件 + 材质参数集合 + CSM(按视口) → 渲染`；`室内卷体查询 → 关级联/平行光`。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
| --- | --- | --- | --- |
| 被调用 | 全局 | `GetSky()` / `AGameSky::Get()` | DynamicWaterEffect、EnvironmentClothSimSubsystem、StoneBossTransitionSkyBoxManager 等引用 |
| 调用 | Sky/室内卷体 | 直接调用 | GameSky.as `AIndoorSkyLightingVolume::GetIndoorSkyLightingSettings(...)` 决定关级联/平行光 |
| 调用 | SceneView | 直接调用 | `SceneView::SetCSMDistanceScaleForView`（CSM 按视口缩放）、`SetHazeGlobalTextureForViewPosition`（雨遮罩）、读 `IsPendingFullscreen` |
| 调用 | PostProcess | 直接调用 | 光照配置含 `AHazePostProcessVolume`，切换时启停其 `bEnabled`/`BlendWeight` |
| 委托 | 关卡激活 | 委托 | GameSky.as `Level.OnLevelActivated/Deactivated` 控制天光可见性 |
| 数据 | 世界材质 | 材质参数集合 | 写 `GlobalParameters`（SunColor/SunDirection/Ocean*）、`WindParameters`、`GlobalRain*` |

## 关键文件

- **GameSky.as**：`AGameSky` 天空总成（平行光/天光/雾/天穹/太阳/风）、`FHazeLightingConfig`/`FFogParameters`、光照配置混合、CSM 级联按视口缩放、编辑器 Tick 组件。
- **GlobalRain.as**：`UGlobalRainComponent`，场景捕获 + RainBlocker 合成雨遮罩 RenderTarget 并设为各视口全局纹理。
- **IndoorSkyLightingVolume.as**：`AIndoorSkyLightingVolume` 室内卷体 + `GetIndoorSkyLightingSettings` 查询（支持反转卷体）。
- **RainBlocker.as**：`ARainBlocker` + `URainBlockerComponent`，带优先级的强制加/去雨区域（含编辑器可视化）。

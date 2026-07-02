# Core / Rendering

> 职责：Split Fiction 的渲染与视图功能域，负责屏幕后处理、画质开关、天空/光照/雨、灯光封装、贴花拖尾、可绘制表面、换装溶解，以及双人分屏的视图布局。共 8 个子模块、约 26 个文件。

## 功能域概览

渲染域服务于双主角 Mio / Zoe（视口索引 0 / 1），围绕「每位玩家一套后处理栈 + 一个全局天空/画质单例 + 一套分屏视图管理」组织。绝大多数效果最终落到三类载体之一：

- **Uber Shader（巨型后处理材质）**：每位玩家一个 `UMaterialInstanceDynamic`，承载速度线、黑白、暗角、曝光、模板遮罩描边、瓦斯等所有屏幕特效。
- **材质参数集合（MaterialParameterCollection）**：`GlobalParameters` / `WindParameters` / `StencilParameters`，由 `UPostProcessingComponent`、`AGameSky`、`UGlobalRainComponent` 写入，被场景中海量材质读取。
- **控制台变量（CVar）**：`URenderingSettingsSingleton` 据画质/详情等级切换 SSS、SSR、动态分辨率、视距、纹理预算。

```
AHazePlayerCharacter (Mio / Zoe)
   └─ UPostProcessingComponent (每玩家一份)
        ├─ UberShaderMaterialDynamic ─── 速度线/黑白/暗角/曝光/瓦斯
        ├─ UStencilEffectViewerComponent ─ 每视口 15 个模板遮罩 → 描边/抠像/时间战
        └─ UOutlineViewerComponent
URenderingSettingsSingleton (全局单例, TInstigated 切 CVar)
AGameSky (单例, 平行光/天光/雾/天穹/风/CSM 级联)
   ├─ UGlobalRainComponent ── 雨遮罩 RenderTarget → SceneView 全局纹理
   └─ 查询 AIndoorSkyLightingVolume → 室内关级联/平行光
AMultiScreenSplitManager / ACustomMergeSplitManager (分屏视图布局)
```

## 八个子模块

| 子模块 | 职责 | 文件数 |
| --- | --- | --- |
| [PostProcess](./PostProcess/README.md) | 后处理体 + 预设、Uber Shader 后处理组件、模板遮罩描边、玩家异步描边、画质强制开关 | 8 |
| [Rendering](./Rendering/README.md) | 画质设置单例（TInstigated 切 CVar）、关卡级覆盖 Actor、玩家次表面散射裁剪能力 | 3 |
| [Sky](./Sky/README.md) | 游戏天空（平行光/天光/雾/天穹/风/CSM）、全局雨、室内光照卷体、雨阻挡器 | 4 |
| [Lights](./Lights/README.md) | 点光 / 聚光的薄封装（编辑器图标缩放） | 2 |
| [Decal](./Decal/README.md) | 随移动生成的拖尾贴花组件 | 1 |
| [PaintablePlane](./PaintablePlane/README.md) | CPU 数组 + GPU 渲染目标双份的可绘制平面/体积 | 1 |
| [Wardrobe](./Wardrobe/README.md) | 换装溶解组件（Glitch 着色 + Niagara 接缝）+ 手动溶解球 | 2 |
| [SceneView](./SceneView/README.md) | 多屏 / 自定义合并 / 对角 / 叠加分屏视图管理 + 额外相机用户 | 5 |

## 模块关系图

```
                ┌──────────────────────────────────────────────┐
                │              PostProcess (后处理核心)            │
                │  UPostProcessingComponent · UberShaderDynamic   │
                │  StencilEffect · Outline                        │
                └───────┬──────────────────────────┬────────────┘
                        │ UberShaderEnablement       │ ShowLeft/RightViewportStencils
                        │ (模板可见 → 开启巨型着色器) │ ← SceneView::IsFullScreen()
                        ▼                            ▼
   ┌──────────────────────────┐        ┌──────────────────────────────┐
   │    Rendering (画质单例)    │        │       SceneView (分屏视图)      │
   │  URenderingSettingsSingleton│      │  Manager → SetSplitScreenMode  │
   │  →切 SSS/SSR/动态分辨率 CVar │ ←─── │  IsPendingFullscreen (cutscene)│
   └──────────────────────────┘  查询  └───────────────┬──────────────┘
                                                        │ 雨遮罩纹理写入各视口
                ┌───────────────────────────────────────┘
                ▼
   ┌──────────────────────────┐  查询   ┌──────────────────────────┐
   │       Sky (天空/雨/光)     │ ──────→ │ IndoorSkyLightingVolume   │
   │  AGameSky · GlobalRain     │ 室内关  │ (室内关级联/平行光)        │
   │  → CSM 按视口缩放 (SceneView)│ 级联   └──────────────────────────┘
   └──────────────────────────┘

   Lights / Decal / PaintablePlane / Wardrobe ── 相对独立的资源/工具型模块,
   通过引擎光照、贴花、RenderTarget、Glitch 材质参数与渲染管线协作。
```

## 域内五大协作机制

| 机制 | 典型用法 | 实证 |
| --- | --- | --- |
| 直接调用 `Type::Get` | `UPostProcessingComponent::Get(Player)`、`AGameSky::Get()`、`Game::GetSingleton(URenderingSettingsSingleton)` | PostProcessing.as、GameSky.as、LevelRenderingSettingsActor.as |
| Crumb / 联机同步 | 与 Camera 域协同的视图状态（SceneView 经相机用户驱动） | MultiScreenSplitManager.as |
| 标签阻塞 | 玩家描边读 `IsCapabilityTagBlocked(CapabilityTags::Outline / BlockedByCutscene)` | StencilEffect.as |
| 委托 | `Level.OnLevelActivated`、`SequenceResponseComponent.OnSequenceEnd` 控制天光/暖屏 | GameSky.as、TestPortalWarpVolume.as |
| 可组合设置 / TInstigated | `TInstigated<ERenderingSettingMode>`、`UPlayerOutlineSettings : UHazeComposableSettings`、`UberShaderEnablement.Apply/Clear` | RenderingSettingsSingleton.as、PlayerOutlineSettings.as |

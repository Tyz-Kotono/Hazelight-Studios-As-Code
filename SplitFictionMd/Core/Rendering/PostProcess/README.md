# Core / Rendering / PostProcess

> 职责：屏幕后处理的核心载体——巨型后处理材质（Uber Shader）+ 模板遮罩描边 + 后处理体预设，统一承载速度线、黑白、暗角、曝光、玩家描边等全部屏幕特效。共 8 个文件。

## 内部架构

核心是挂在每位玩家身上的 `UPostProcessingComponent`（一玩家一份），它持有 `UberShaderMaterialDynamic`（一个 MID），并把所有效果合成进 `GlobalPostProcess` 后通过 `Player.AddCustomPostProcessSettings` 注入。

关键类与数据结构：

- **`UPostProcessingComponent : UActorComponent`**：后处理总线。
  - `UberShaderMaterialDynamic`：巨型后处理材质 MID，承载速度线（`speedEffectData_*`）、瓦斯（`gasData_Strength`）、模板描边可见性（`ShowLeft/RightViewportStencils`）、视口索引等。`AddToRoot()` 防止被 GC（渲染线程直接引用）。
  - `TInstigated<>` 字段：`PostProcessMaterial`、`CurrentSpeedEffect`、`BlackAndWhiteStrength`、`UberShaderEnablement`、`CurrentCameraParticles`——均以「发起者 + 优先级」叠加管理。
  - `UpdatePostProcess()`：按需把 Uber Shader / 自定义材质打包成 `FWeightedBlendable` 注入；只有 `UberShaderEnablement` 或速度效果激活时才挂载巨型着色器（性能优化）。
  - `Tick`：每帧把双人位置、速度、骨骼插槽、相机姿态写入 `GlobalParameters`（草丛避让等世界材质读取）和 `GlobalNiagaraParams_Inst`（Mio_/Zoe_ 前缀）。
  - 暗角 / 曝光用 `FHazeAcceleratedFloat` 平滑过渡。
- **`AHazePostProcessVolume : APostProcessVolume`** + **`UHazePostProcessPreset : UDataAsset`**：后处理体，可按 `EPostProcessDistanceCheckType`（Camera / Player / Custom）决定混入点；预设可在编辑器一键保存/复用。
- **模板遮罩系统**（StencilEffect.as）：`UStencilEffectViewerComponent` 每视口维护最多 15 个模板槽（`StencilEffectSlots[16]`），将 custom-depth stencil 按位打包（Mio 用高 4 位、Zoe 用低 4 位）；`FStencilEffectStateInstigated` 按优先级解析每个目标组件当前生效的槽。玩家描边用相机到对方玩家骨骼的异步射线（`AsyncLineTraceByChannel`）逐根检测遮挡，据覆盖率淡入淡出描边。
- **描边外观**（Outlines.as）：`FOutline` / `UOutlineDataAsset` 定义颜色、填充/边框透明度、显示模式、贴图索引；`Outline::` 命名空间把「在 Actor/组件上应用描边」转发到 `StencilEffect::`。
- **画质强制开关**（PostProcessShaderEnablement.as）：放进关卡即对双人 `UberShaderEnablement.Apply(true)`，强制常开巨型着色器。

数据流：`效果 API（SpeedEffect:: / PostProcessing:: / StencilEffect::）→ TInstigated 叠加 → UpdatePostProcess / Uber Shader 参数 → AddCustomPostProcessSettings → 渲染`。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
| --- | --- | --- | --- |
| 被调用 | 全局大量能力 | `UPostProcessingComponent::Get/GetOrCreate` | 40+ 文件引用 `UPostProcessingComponent::`/`SpeedEffect::`/`PostProcessing::`（如 WingSuitMovementCapability、BattlefieldHoverboard…） |
| 调用 | SceneView | 直接调用 `SceneView::IsFullScreen()` | StencilEffect.as `UpdateViewportStencilAssociation()` 决定左右视口是否显示模板 |
| 调用 | Rendering 域 | 标签阻塞 | StencilEffect.as 读 `IsCapabilityTagBlocked(CapabilityTags::Outline / BlockedByCutscene)` 隐藏玩家描边 |
| 调用 | Effects（Niagara） | 直接调用 | `KillParticleManager::`、`InfluenceSystem::AddPointsForMeshBones` 管理相机粒子影响点 |
| 配置 | 玩家描边可见性 | 可组合设置 | `UPlayerOutlineSettings : UHazeComposableSettings`（PlayerOutlineSettings.as） |
| 数据 | 世界材质 / Niagara | 材质参数集合 | 写 `GlobalParameters`（Player0/1Position…）、`StencilParameters`、`GlobalNiagaraParameters` |

## 关键文件

- **PostProcessing.as**：`UPostProcessingComponent` 核心 + `SpeedEffect::` / `PostProcessing::` 效果 API + 速度线/暗角/曝光/黑白/相机粒子逻辑。
- **HazePostProcessVolume.as**：`AHazePostProcessVolume` + `UHazePostProcessPreset` + 编辑器预设保存定制化。
- **StencilEffect.as**：每视口 15 模板遮罩打包、`UStencilEffectViewerComponent`、玩家异步描边遮挡检测、`StencilEffect::Apply/ClearStencilEffect`。
- **Outlines.as**：`FOutline`/`UOutlineDataAsset` 描边数据 + `Outline::` 应用/清除命名空间（转发到模板系统）。
- **PlayerOutlineSettings.as**：玩家描边是否可见的可组合设置资产。
- **StencilCutout.as**：`AStencilCutout` 静态网格抠像体（不渲主通道，仅写模板，挖空身后描边）。
- **PostProcessShaderEnablement.as**：`UUberPostProcessShaderEnablementComponent`，关卡内强制常开巨型着色器。
- **TestPortalWarpVolume.as**：传送门吸入特效后处理体（溶解材质 + Niagara + 网格碰撞渐禁），偏关卡测试用途。

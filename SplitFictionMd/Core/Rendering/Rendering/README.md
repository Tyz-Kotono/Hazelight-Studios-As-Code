# Core / Rendering / Rendering

> 职责：全局画质设置——按画质/详情等级动态切换 SSS、SSR、动态分辨率、视距、纹理预算等控制台变量，并提供关卡级覆盖与玩家次表面散射的距离裁剪。共 3 个文件。

## 内部架构

核心是单例 `URenderingSettingsSingleton`，用 `TInstigated<>` 收集各方诉求（关卡 Actor、cutscene 等），每帧 `UpdateInstigatedSettings()` 解析后写入 CVar：

- **`URenderingSettingsSingleton : UHazeSingleton`**：
  - `TInstigated<ERenderingSettingMode>` 控制 SSS（`r.SubsurfaceScattering`）、SSR（`ShowFlag.ScreenSpaceReflections`）。
  - `TInstigated<float>` 控制动态分辨率节流（普通/高配两档，由 `Haze.DynamicResThrottlingSpec` CVar 选档）、视距缩放（`r.ViewDistanceScale.SecondaryScale`）、纹理预算（`r.Streaming.Boost`）。
  - `ERenderingSettingMode`（On/Off/NeedMedium/High/UltraShaderQuality/NeedMedium/HighDetailMode）经 `ResolveSetting()` 结合 `Game::IsShaderQualityAtLeast*` 求值。
  - `Tick` 检测「全屏 + 有玩家参与 cutscene」状态切换 cutscene 设置；关卡间 `ResetStateBetweenLevels()` 清空发起者。
  - 设计约束（注释强调）：不要覆盖由设备配置 ini 或选项菜单设定的 CVar。
- **`ALevelRenderingSettingsActor : AHazeActor`**：放进关卡即在 `BeginPlay` 向单例 `Apply` 覆盖、`EndPlay` `Clear`，支持覆盖 SSS 与动态分辨率两档。
- **`UPlayerSubsurfaceEnablementCapability : UHazePlayerCapability`** + **`UPlayerRenderingSettingsComponent`**：按视口距离裁剪次表面散射——当对方玩家/附加网格离本视口相机超过 FOV 映射出的阈值时调 `Mesh.SetAvoidSubsurfaceInView(ViewIndex, true)`，省 SSS 开销；带 `BlockedByCutscene` 标签。

数据流：`关卡 Actor / cutscene → TInstigated.Apply → UpdateInstigatedSettings → Console::SetConsoleVariable*`。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
| --- | --- | --- | --- |
| 被调用 | 关卡覆盖 Actor | `Game::GetSingleton(URenderingSettingsSingleton)` + TInstigated | LevelRenderingSettingsActor.as BeginPlay/EndPlay Apply/Clear |
| 调用 | SceneView | 直接调用 | RenderingSettingsSingleton.as Tick 读 `SceneView::IsPendingFullscreen()` 判定 cutscene |
| 调用 | 引擎渲染 | CVar | `Console::SetConsoleVariableInt/Float` 切 SSS/SSR/动态分辨率/视距/纹理 |
| 标签 | cutscene | 标签阻塞 | `UPlayerSubsurfaceEnablementCapability` 默认带 `CapabilityTags::BlockedByCutscene` |
| 调用 | 玩家网格 | 直接调用 | `Mesh.SetAvoidSubsurfaceInView(ViewIndex, bool)` 按视口裁剪 SSS |

## 关键文件

- **RenderingSettingsSingleton.as**：`URenderingSettingsSingleton` 画质单例，TInstigated 解析后切 SSS/SSR/动态分辨率/视距/纹理 CVar。
- **LevelRenderingSettingsActor.as**：`ALevelRenderingSettingsActor` 关卡级画质覆盖 Actor（Apply/Clear 单例诉求）。
- **PlayerSubsurfaceEnablementCapability.as**：`UPlayerSubsurfaceEnablementCapability` + `UPlayerRenderingSettingsComponent`，按视口距离裁剪玩家/附加网格的次表面散射。

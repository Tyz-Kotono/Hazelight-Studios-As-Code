# Editor / 可视化 — Visualization

> 职责：把不可见的玩法数据画到编辑器视口，做到所见即所得。
>
> 覆盖子模块：**EditorVisualizers**（19 文件，最大子模块）、**VisualizationModes**（12 文件）。

---

## 一、EditorVisualizers/ — 组件视口可视化（19 文件）

为各类玩法组件在编辑器视口绘制辅助图形（连线、包围盒、网格、骨骼预览）。基于 `UHazeScriptComponentVisualizer`：声明 `default VisualizedClass = U某组件;`，当选中带该组件的 actor 时，引擎自动调用 `VisualizeComponent()`。

### 基础设施

#### `EditorVisualizationSubsystem`（核心）
这是整个可视化体系的底层引擎，继承 `UHazeEditorSubsystem`。提供"画一帧"风格的 API：
- `DrawMesh(Instigator, Transform, StaticMesh)` — 画静态网格
- `DrawAnimation(Transform, AnimSequence, Time, ...)` — 画骨骼动画预览（支持 root motion、自动找预览网格）
- `DrawSolidBox(Instigator, Location, Rotation, Extents, Color, Opacity)` — 画实心盒
- `DrawVisualizationComponent(...)` — 通用：临时生成任意组件类型

**自动回收机制**：内部用 `LastUsedFrame`（帧号）追踪每个临时 actor/组件。`Tick()` 里凡是上一帧没再被请求的（`LastUsedFrame < GFrameNumber - 1`）就销毁。调用方只需"每帧请求一次"，无需手动管理生命周期。同时提供 `DrawSolidBox` mixin，让任意 Visualizer 都能直接画实心盒+线框边。

#### `EditorBillboardComponent`
编辑器图标（广告牌）组件，`bIsEditorOnly = true`。`ConstructionScript` 里按名字从 `/Game/Editor/EditorBillboards/` 或引擎资源加载贴图，给空 actor 一个可见图标。游戏世界中自动跳过。

### 具体可视化器（示例）

| 文件 | 可视化内容 |
|---|---|
| `RespawnPointVolumeVisualizer` | 重生点连线，**按可用玩家上色**（仅 Mio 蓝 / 仅 Zoe 红 / 双人橙）；含细节面板 `URespawnPointVolumeDetails` 用即时 UI 提示"无可用重生点" |
| `ScenePointComponentVisualizer` / `ScenepointAnimationComponentVisualizer` | AI 场景点（站位/入场）及其动画预览 |
| `TraversalScenepointComponentVisualizer` | AI 穿越场景点 |
| `InteractionVisualizer` | 玩家交互体范围 |
| `CameraSettingsComponentVisualizer` | 相机设置组件 |
| `HazeVoxTriggerVisualizers` | 语音(VOX)触发体 |
| `PlayerLookAtTriggerComponentVisualizer` | 玩家看向触发体 |
| `BasicAIProjectileLauncherComponentVisualizer` | AI 投射物发射器轨迹 |
| `MovingActorSplineComponentVisualizer` | 移动 actor 的样条路径 |
| `TeamLazyTriggerShapeComponentVisualizer` | 队伍触发形状 |
| `SanctuarySnakeSplineEffectComponentVisualizer` | Sanctuary 关卡蛇形样条特效（关卡专属可视化也归在此） |
| `DummyVisualizationComponentVisualizer` | 通用调试图元（球/柱，配合 `Core/Visualization`） |
| `VisualizePlayerPose` | 玩家姿势预览 |
| `HazeActorSpawnPatternVisualizer` | AI 生成器的生成模式分布 |

---

## 二、VisualizationModes/ — 视口调试着色模式（12 文件）

整屏按某个属性给组件着色，用于排查美术 / 性能问题。

**统一模式**：每个类实现 `GetMaterialVisualizationIndex(UObject Object)`，根据规则返回一个可视化索引（0 = 不影响），引擎据此用对应调试材质替换渲染，从而把场景"染色"。

例如 `VisualizeTextureDensity`：只处理静态网格组件，跳过透明/半透明材质和名字以 "swatch" 开头的材质，其余返回索引 1 参与纹理密度着色。

### 全部模式

| 文件 | 可视化维度 |
|---|---|
| `VisualizeTextureDensity` | 纹理密度（贴图分辨率是否合理） |
| `VisualizeMaterialSlots` | 材质槽分布 |
| `VisualizeNonUniformScale` | 非均匀缩放（找出被拉伸变形的网格） |
| `VisualizeShadowPriority` | 阴影优先级 |
| `VisualizeDecalRecieve` | 贴花接收设置 |
| `VisualizeComponentTags` | 组件标签 |
| `VisualizePallette` | 调色板 |
| `VisualizeSwatchMaterials` | swatch 材质 |
| `VisualizeTexturePhysMat` | 纹理物理材质 |
| `VisualizeTilerLayers` | Tiler 层 |
| `VisualizeTilerLayersEnabled` | Tiler 层启用状态 |
| `VisualizeDeveloperAssets` | 开发资产（配合质检体系标识泄漏） |

---

## 与运行时模块的关系

- EditorVisualizers 可视化的对象，多来自 `Core`（RespawnPoint、Camera、Interaction、Vox、Spline、Visualization）与 `Gameplay/AI`（ScenePoint、Spawner、Projectile、Traversal）。
- `VisualizeDeveloperAssets` 与 [Validation.md](./Validation.md) 的开发资产质检体系配套。

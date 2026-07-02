# Editor / 关卡编辑 — Level Editing

> 职责：提供拉伸缩放、绘制样条、合并网格、批量资产处理、细节面板定制等高效内容编辑工具。
>
> 覆盖子模块：**LevelEditor**（10）、**SplineEditor**（8）、**Utilities**（7）、**Props**（1）、**HazeActorSpawner**（1）。

---

## 一、LevelEditor/ — 关卡编辑杂项工具（10 文件）

混合了交互式工具、工具窗口与右键菜单扩展。

### 交互式视口工具

#### `ScaleStretchTool`（重点）
继承 `UEditorScriptableClickDragTool`，是一个**拉伸缩放工具**：
- 工具栏注册：`ToolCategory = "Placement"`、`ToolName = "Stretch"`、带自定义图标 `StretchTool.png`。
- 用 `UHazeScaleStretchPropertySet`（继承 `UScriptableInteractiveToolPropertySet`）保存工具设置，`OnScriptSetup` 时恢复、`OnScriptShutdown` 时保存。
- 拖拽包围盒的**边 / 面 / 顶点**来缩放选中组件（`DraggingEdge` / `DraggingFace` / `DraggingVertex`），激活时临时切换编辑器 Widget 模式为 None。

### 工具窗口（`UEditorUtilityWidget`）

| 文件 | 功能 |
|---|---|
| `ActorReferencesUtilityWidget` | 可视化 actor 之间的引用关系（含"被关卡蓝图引用"标记），用即时 UI 列表展示，可跟随选择 |
| `RenderedTrianglesUtilityWidget` | 统计渲染三角面数（性能检查） |

### 右键菜单扩展（`UScriptActorMenuExtension`）

| 文件 | 功能 |
|---|---|
| `CopyActorsAsArrayActorMenuExtension` | 把选中 actor 的路径复制为 `(path1,path2,...)` 数组格式，方便粘贴到属性数组 |
| `ReplaceStaticMeshActorWithAmbientMovementMenuExtension` | 把静态网格 actor 替换为带环境运动（AmbientMovement）的版本 |
| `RespawnPointActions` | 重生点相关右键操作 |

### 其它工具 / 子系统

| 文件 | 功能 |
|---|---|
| `CursorPlacement` | 光标处放置 actor 的辅助 |
| `HazeVoxTriggerReplacer` | 批量替换语音(VOX)触发体 |
| `PlayerScaleReference` | 在场景中放置玩家比例参考（美术对照尺寸） |
| `EditorCustomizationsSubsystem` | 关卡编辑器定制的子系统入口 |

---

## 二、SplineEditor/ — Haze 样条编辑工具包（8 文件）

完整的样条编辑套件，是项目里最复杂的编辑工具之一。被 `Props/PropLine`、`KineticActors`、各关卡轨道等大量系统复用。

### 核心入口

#### `HazeSplineEditor`
继承 `UHazeScriptComponentVisualizer`，`VisualizedClass = UHazeSplineComponent`。负责样条点的**可视化与交互**：
- `PrepareAssets()` 预加载点/切线手柄网格与多种状态材质（选中/悬停/选中+悬停/覆盖切线），资源在 `/Game/Editor/SplineEditor/`。
- 处理左右键点击、点的悬停高亮、切线手柄拖拽、点交换拖拽（`SwapDrag*`）、点复制等。
- 通过 `GetGlobalSplineSelection()` 共享全局选择状态。

#### `HazeSplineDrawTool`（绘制工具）
继承交互式工具，可在**几何表面上直接画样条**。用 `UHazeSplineDrawPropertySet` 暴露丰富参数：
- 点间距 `MinPointSpacing`、是否直线段、是否自动从最近端绘制；
- 表面模式：射线打到世界表面 / 用上一点高度 / 自定义平面（origin+rotation）；
- 点朝向（背离表面）、Shift 拖动是否沿表面移动、橡皮擦范围等。

### 配套类

| 文件 | 职责 |
|---|---|
| `HazeSplineBuilder` | 样条构建逻辑 |
| `HazeSplineSelection` | 全局选择状态（哪些点被选中/悬停） |
| `HazeSplineEditOperations` | 编辑操作（增删点、设切线等） |
| `HazeSplineDetails` | 样条组件的细节面板 |
| `MovingActorSplineDetails` | 移动 actor 样条的细节面板 |
| `HazeSplineContextMenu` | 样条右键上下文菜单 |

---

## 三、Utilities/ — 资产右键 Action 工具（7 文件）

均为 `UScriptAssetMenuExtension`，通过 `SupportedClasses.Add(...)` 限定适用资产类型，用 `UFUNCTION(CallInEditor)` 暴露操作。

| 文件 | 功能 |
|---|---|
| `MakeBreakable` | 适用于 `UStaticMesh`。用 **GeometryScript** 把网格按连通分量拆分（`SplitMeshByComponents`）、写顶点色、再合并写回，转成可破坏网格。含 `ClearVertexColor` 等操作 |
| `HazeLightActionUtility` | 光源资产/actor 批量操作（基类） |
| `HazePointLightActionUtility` | 点光源操作 |
| `HazeSpotLightActionUtility` | 聚光源操作 |
| `HazeDecalActionUtility` | 贴花批量操作 |
| `HazeTextureActionUtilities` | 纹理批处理 |
| `FixTilerGlobalParameters` | 修复 Tiler 全局材质参数 |

---

## 四、Props/ — PropLine 细节面板定制（1 文件）

#### `PropLineDetailCustomization`
继承 `UHazeScriptDetailCustomization`，`DetailClass = APropLine`。是细节面板定制的典范，功能丰富：
- **隐藏无关分类**（LOD、RayTracing、TextureStreaming 等），有预设时隐藏手动设置项。
- **即时 UI 按钮**（`AddImmediateRow` + 链式绘制）：
  - 🛠️ Merge Mesh / Refresh Merged Mesh —— 把 prop line 上所有网格**分段合并**成单一替换网格（按距离聚类成 segment，逐段 `MergeComponentsToStaticMesh` 生成到 `/Game/Environment/Generated/MergedPropLines/`），优化 draw call。
  - 🎛️ Save Preset —— 把当前设置存为 `UPropLinePreset` 资产供复用。
  - ✏️ Draw Spline —— 激活 `UHazeSplineDrawTool` 在地形上画样条。
- **错误提示**：检测无效网格（origin 全在前端、bounds 为 0）并红字警告。
- **自动重命名**：根据预设名或网格名把 actor 标签设为 `PropLine <名字>`。
- 所有修改包进 `FScopedTransaction` 支持撤销。

---

## 五、HazeActorSpawner/ — 生成器细节定制（1 文件）

#### `HazeActorSpawnerDetailCustomization`
`UHazeScriptDetailCustomization`，为 AI 生成器 `AHazeActorSpawnerBase`（见 `Gameplay/AI/Spawning`）提供生成模式（spawn pattern）的细节面板编辑。配合 [Visualization.md](./Visualization.md) 的 `HazeActorSpawnPatternVisualizer` 一起用：面板调参 + 视口看分布。

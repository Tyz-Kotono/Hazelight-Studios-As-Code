# Core / Systems / Props

> 职责：环境道具的批量摆放与生成——沿样条铺设网格的道具线、网格阵列、Haze 道具材质组件、生成预设与编辑器子系统。共 10 文件（编辑器生成为主）。

## 内部架构

核心是两类「**程序化摆放 actor**」+ 一套「**网格生成管线**」+ 支撑组件/子系统。

### 关键类型

| 类型 | 基类 / 形态 | 角色 |
| --- | --- | --- |
| `APropLine` | `AHazeBaseProp` | 沿编辑器样条（`UPropLineSplineComponent`）铺一排网格，支持分段/分布/拉伸/合并网格 |
| `APropArray` | `AHazeBaseProp` | 在 X/Y/Z 三维栅格上摆放网格，带随机偏移/旋转/缩放/错切 |
| `APropLineProgression` | `AHazeActor` | 由外部 `ProgressionActor` 驱动某 `APropLine` 的渐进显隐/强度 |
| `UHazePropStaticMeshComponent` | `UStaticMeshComponent` | 给蓝图 actor 用的 Haze 道具网格组件，写自定义图元数据（按世界位置随机值 + 总体缩放）供材质 tiler |
| `AHazePrefabActor` | `AHazeActor`（Abstract） | 环境几何 BP 基类，构造期按 `bAffectsNavigation` 设置网格导航属性 |
| `AHazeNiagaraActor` | `AHazeActor` | 环境 Niagara 体，按裁剪距离联动 `UDisableComponent`（分屏单侧可单独停渲染） |
| `UTagContainerComponent` | `USphereComponent` | 编辑器期标签容器（不可见、无碰撞），把标签拷给生成出的道具组件 |
| `UPropLineEditorSubsystem` | `UHazeEditorSubsystem` | 维护 `PropLinesPendingUpdate`，把旧道具线迁移到实例组件并触发更新 |

### 道具线数据流（`APropLine`）

1. **配置**：`Preset`（`UPropLinePreset` 数据资产）或内联 `FPropLineSettings`；`Type`（StaticMeshes / CurvedPlacement / SplineMeshes）+ `MeshDistribution` + `MeshStretching` 决定铺设方式（数据结构集中在 `PropLineData.as`）。
2. **更新 `UpdatePropLine`**：应用 preset → `PropSpline.UpdateSpline`（`UPropLineSplineComponent` 按段类型强制直/曲切线）→ `PrepareSegments` 对齐段数 → 组装 `FPropLineMeshGenerationContext`（`PropLineMeshGeneration.as`）→ `Generate()` 复用现有实例组件、按段生成 `UHazePropComponent` / `UHazePropSplineMeshComponent`；存在 `MergedMeshes` 时直接放合并网格。未用到的旧组件销毁。
3. **实例组件迁移**：首次非构造脚本更新切到 `bIsUsingInstanceComponents`；`ConstructionScript` 在编辑器中把未迁移的道具线排入子系统 `PropLinesPendingUpdate`。
4. **Gameplay 样条**：仅当 `bGameplaySpline` 勾选才在烘焙时复制出可供运行时使用的 `UHazeSplineComponent`；否则样条为编辑器专用，运行时取用会经 `CookChecks::EnsureSplineCanBeUsedOutsideEditor` / `CheckUsableGameplaySpline` 报错。

### 阵列数据流（`APropArray`）

`ConstructionScript` 三重循环按 `CopiesX/Y/Z` 调 `PlaceProp`：用基于 actor 位置哈希的 `FRandomStream` 选网格、算栅格偏移+错切、按 `EPropArrayRotationType` 施加静态/随机区间旋转、可选随机缩放，受 `MaxInstances` 上限保护。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
| --- | --- | --- | --- |
| 被引用 | `Prefab/`（`FPrefabPropLine` 等） | 数据嵌入 | 预制体快照内嵌道具线设置、段、合并网格、样条点（`PrefabAsset.as`），并通过 `UHazePropComponent::AreSettingsEqual` 比较 |
| 被引用 | `Spline/GameplaySplineCheck.as` | 直接调用 | 复用 `APropLine` / `CookChecks` 校验 gameplay 样条可用性 |
| 生成 | `UHazePropComponent` / `UHazePropSplineMeshComponent` | `Create` / `GetOrGenerateComponent` | 道具线与阵列在构造期生成这些引擎道具组件 |
| 调用 | `Editor::CopyAllComponentTags` / `DestroyAndRenameInstanceComponentInEditor` | 直接调用 | 标签拷贝与未用组件清理 |
| 依赖 | `Disable/UDisableComponent` | 组件组合 | `AHazeNiagaraActor` 按裁剪距离自动停用 |
| 委托 | `FPropLineUpdatedEvent OnPropLineUpdated` | 委托 | 道具线更新后广播给监听者 |

## 关键文件

- `PropLine.as` — `APropLine` 道具线 actor + `UPropLineSplineComponent`（按段强制切线、gameplay 样条守卫）+ `CookChecks` 烘焙校验 + `FPropLineUpdateScope`。
- `PropLineData.as` — 道具线数据：`UPropLinePreset`、`FPropLineMesh`/`FPropLineSettings`、`EPropLineType` / `EPropLineDistributionType` / `EPropLineStretchType` 等枚举与尺寸计算。
- `PropLineMeshGeneration.as` — `FPropLineMeshGenerationContext` 生成管线：`Generate()` 与 `GetOrGenerateComponent` 复用/生成网格组件。
- `PropLineProgression.as` — `APropLineProgression`：由外部 actor 驱动道具线的渐进强度/显隐。
- `PropLineEditorSubsystem.as` — `UPropLineEditorSubsystem`：迁移旧道具线到实例组件并批量更新。
- `PropArray.as` — `APropArray` 网格阵列 actor + `EPropArrayRotationType`，三维栅格 + 随机偏移/旋转/缩放/错切，位置哈希做随机种子。
- `HazePropStaticMeshComponent.as` — `UHazePropStaticMeshComponent`：写按世界位置的随机值与总体缩放到自定义图元数据，供 Haze 道具材质。
- `TagContainerComponent.as` — `UTagContainerComponent`：编辑器期不可见标签容器，标签随生成拷给道具组件。
- `HazePrefabActor.as` — `AHazePrefabActor`：环境几何 BP 基类，按 `bAffectsNavigation` 设置网格导航（注释提示与 `APrefabRoot` 无关，待改名）。
- `HazeNiagaraActor.as` — `AHazeNiagaraActor`：环境 Niagara 体，按裁剪距离联动 `UDisableComponent` 实现分屏单侧停渲染。

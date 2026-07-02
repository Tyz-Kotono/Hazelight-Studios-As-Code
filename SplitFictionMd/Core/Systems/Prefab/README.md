# Core / Systems / Prefab

> 职责：编辑器预制体系统——把一组环境 actor（道具/灯光/Niagara/雾球/道具线/音源/贴花）采集为 `UPrefabAsset` 数据，并在实例上回放、编辑、合并网格。共 7 文件（编辑器期为主）。

## 内部架构

整体是「**资产数据 ↔ 关卡实例**」双向同步，配合一个编辑器子系统驱动更新。

### 关键类型

| 类型 | 基类 / 形态 | 角色 |
| --- | --- | --- |
| `APrefabRoot` | `AHazeBasePrefab` | 关卡中的预制体实例根，持有 `UPrefabAsset`、`FPrefabState`、`FPrefabInstanceSettings` |
| `UPrefabAsset` | `UHazeBasePrefabAsset` | 预制体资产：`FPrefabData Data`、`ChangeId`、子预制体变更 id、是否自动生成合并网格 |
| `FPrefabData` | struct | 预制体内容快照：Props / SpotLights / PointLights / ChildPrefabs / NiagaraSystems / HazeSpheres / PropLines / SpotSounds / Decals + 合并网格设置 |
| `FPrefabState` | struct（`PrefabEditing.as`） | 编辑态：是否可编辑、guid↔actor 映射、组件映射、实例设置 |
| `UPrefabEditorSubsystem` | `UHazePrefabEditorSubsystem` | 关卡变更时更新所有改动预制体、绘制编辑视口 UI、提供「选中转预制体」控制台命令 |
| `ABrokenPrefabRoot` | `AHazeActor` | 损坏/无法解析预制体的占位 actor |
| `UPrefabDetailCustomization` | `UHazeScriptDetailCustomization` | `APrefabRoot` 的细节面板定制 |

### 数据流

1. **采集**：`SaveChangesAndStopEditing` → `Prefab::GatherDataFromEditablePrefab`（`PrefabParts.as` 内各 `GatherDataFromEditablePrefabPart` 按类型抽取灯光/道具等参数）→ 生成新 `FPrefabData`，与旧数据 `IsChanged` 比较后写回 `UPrefabAsset` 并 `BumpPrefabChangeId`。
2. **应用**：`UpdatePrefabToData` → `Prefab::UpdatePrefabToData(this, Asset.Data, State)` 在实例上重建子物件；`UpdatePrefabIfChanged` 通过 `NeedsPrefabUpdate(Asset, LastUpdatedGuid)` 检测资产更新。
3. **编辑**：`StartEditing` / `DiscardChangesAndStopEditing` / `SaveChanges...` 切换 `FPrefabState.bEditable`，编辑期把子 actor 解封为可改。
4. **合并网格**：`bAutoGenerateMergedMesh` 时 `Prefab::GenerateMergedMesh` 把整组网格烘成单一 `StaticMesh` 写入 `Data.MergedMeshSettings`。
5. **变更检测**：`FPrefabData::IsChanged` 与各 `FPrefab*::IsChanged` 逐字段（Transform/Guid/灯光参数/道具设置…）对比，避免无谓的资产写盘。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
| --- | --- | --- | --- |
| 依赖 | `Props/`（`UHazePropComponent`、`UPropLinePreset`、`FPropLineSettings`、`EPropLineType`…） | 数据嵌入 | `FPrefabData.PropLines` 内嵌 `FPrefabPropLine`，复用 Props 模块的道具线设置与网格元素结构 |
| 依赖 | `Lights` / `Decal` / Niagara / 音频 / `Shapes`(HazeSphere) | 数据采集 | 各 `FPrefab*` 结构 `SetFromLight` / `Editor_AssignToLight` 等在引擎组件与快照间拷贝参数 |
| 调用 | `Prefab::` 命名空间（`PrefabParts.as`） | 直接调用 | `Gather/Update/StartEditing/StopEditing/GenerateMergedMesh/BumpPrefabChangeId` 等核心过程 |
| 依赖 | `SourceControl` / `UEditorLoadingAndSavingUtils` | 直接调用 | 新建/改动资产时检出与保存包 |
| 关联 | `Props/HazePrefabActor.as`（`AHazePrefabActor`） | 命名相邻 | 源码注释明确「TODO: 改名以免与 `APrefabRoot` 混淆」，二者并非同一体系 |

## 关键文件

- `PrefabRoot.as` — `APrefabRoot` 实例根：持有资产/状态/实例设置，实现编辑、采集、应用、合并网格、缩略图与放置回调。
- `PrefabAsset.as` — `UPrefabAsset` 资产 + `FPrefabData` 及全部 `FPrefab*` 子结构（道具/灯光/Niagara/雾球/道具线/音源/贴花），含逐字段 `IsChanged`。
- `PrefabParts.as` — `Prefab` 命名空间的采集/应用过程：从可编辑预制体各部件抽取数据、回写实例、生成合并网格。
- `PrefabEditing.as` — `FPrefabState` 等编辑态结构（可编辑标记、guid↔actor 与组件映射）。
- `PrefabEditorSubsystem.as` — `UPrefabEditorSubsystem`：关卡变更触发更新、视口编辑 UI、「选中转预制体」控制台命令。
- `PrefabDetails.as` — `UPrefabDetailCustomization`：`APrefabRoot` 细节面板定制。
- `BrokenPrefabRoot.as` — `ABrokenPrefabRoot`：无法解析预制体时的占位 actor。

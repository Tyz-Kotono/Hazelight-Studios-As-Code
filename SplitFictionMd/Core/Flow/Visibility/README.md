# Core / Flow / Visibility（每玩家显隐）

> 职责：以体积控制目标演员/组件/关卡对单名玩家（Mio 或 Zoe）的可见性，支持多体积叠加的可见性仲裁；并提供玩家整体隐藏能力与 temporal log 回溯时的隐藏。共 3 个文件。

## 内部架构

### 显隐体积
- **`AVisibilityVolume`（AHazeActor）**：每帧检测两名玩家视点/位置是否在体积内，据此对目标设置每玩家渲染可见位。
  - **模式** `EVisibilityVolumeMode`：`OnlyVisibleWhenInside`（仅当玩家在内时目标可见）、`HideTargetsWhileInside`（玩家在内时强制隐藏目标）、`None`（自身无功能、仅供他者引用）。
  - **目标来源**：`TargetVolumes`（引用其他体积的内含演员，可跨关卡）、`TargetActors`、`TargetLevels`（整个子关卡）、以及自身体积内的演员（`bTargetContainedActors`，可限静态/完全包含/排除项）。编辑器期 `UpdateContainedActors()` 预扫包含演员。
  - **激活判定** `EVisibilityVolumeApplyType`：按玩家相机位（`PlayerCameraInside`）或玩家本体位（`PlayerActorInside`）测试是否在盒内。
  - **可见性仲裁** `EVisibilityResult`：`Visible` / `Invisible` / `MaybeVisible` / `ForceInvisible`。`UpdateVisibilityForTargetActors` 遍历全局 `TListedActors<AVisibilityVolume>`，综合其他体积对同一目标的结论决定最终显隐——`ForceInvisible` 最高，`Visible` 次之，`MaybeVisible` 仅在无人将其隐藏时可见。
  - **应用**：对目标组件调 `PrimComp.SetComponentRenderVisibilityBit(Bit, bHide)`，对关卡调 `Player.SetLevelRenderedForPlayer(LevelRef, bRender)`，其中 `Bit` 按玩家取 `EHazeVisibilityBit::HideFromMio` 或 `HideFromZoe`。
  - 用 `UHazeListedActorComponent` 登记以支持全局遍历仲裁；`TPerPlayer<bool> IsVolumeActive` 缓存激活态，仅在变化时重算。

### 玩家整体隐藏能力
- **`UPlayerVisibilityCapability`（UHazeMarkerCapability）**：`Visibility` 标签的标记能力。被标记屏蔽（`OnMarkerBlocked`）时隐藏玩家本体及其所有挂接子组件/子演员（递归，跳过另一玩家与 WorldSettings），解除时还原。整体隐藏经 `AddActorVisualsBlock(this)` / `AddComponentVisualsBlocker(this)`。

### 调试回溯隐藏
- **`UTemporalLogScrubbableVisible`（UHazeTemporalLogScrubbableComponent）**（TemporalLogScrubVisibilityComponent.as）：回溯 temporal log 帧时隐藏该演员；若为玩家则 `BlockCapabilities(CapabilityTags::Visibility, this)` 并隐藏挂接演员，停止回溯时还原。`UPlayerVisibilityCapability` 在 `!RELEASE` 下持有它。

### 数据流
体积每帧测玩家在内 → 激活态变化 → `UpdateVisibilityForTargetActors` 跨体积仲裁 → 对目标组件/关卡按 Mio/Zoe 设可见位。能力侧则以标记阻塞整体隐藏玩家。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 调用 | 渲染（组件可见位） | 直接调用 | `PrimComp.SetComponentRenderVisibilityBit(EHazeVisibilityBit::HideFromMio/HideFromZoe, ...)` |
| 调用 | 渲染（关卡可见） | 直接调用 | `Player.SetLevelRenderedForPlayer(LevelRef, bRender)` |
| 仲裁 | 其他 `AVisibilityVolume` | 列表遍历 | 遍历 `TListedActors<AVisibilityVolume>` 综合各体积结论 |
| 阻塞 | 玩家本体/子物体 | 视觉阻塞 | `UPlayerVisibilityCapability` 调 `AddActorVisualsBlock(this)` / `AddComponentVisualsBlocker(this)` |
| 标签阻塞 | Visibility 能力 | 标签阻塞 | 回溯组件调 `BlockCapabilities(CapabilityTags::Visibility, this)` 暂停玩家显隐能力 |

## 关键文件

- `VisibilityVolume.as`：`AVisibilityVolume`，每玩家显隐体积与多体积可见性仲裁；含编辑器子系统重建包含演员。
- `PlayerVisibilityCapability.as`：`UPlayerVisibilityCapability`，标记阻塞时整体隐藏玩家及挂接物体。
- `TemporalLogScrubVisibilityComponent.as`：`UTemporalLogScrubbableVisible`，回溯调试帧时隐藏演员/玩家。

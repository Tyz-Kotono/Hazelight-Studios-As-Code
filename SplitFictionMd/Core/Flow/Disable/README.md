# Core / Flow / Disable（按需禁用）

> 职责：当两名玩家都远离时自动禁用演员（及其关联演员）以节省开销，由全局单例分帧轮转批量更新；并提供分组禁用演员便于按半径批量管理。共 4 个文件。

## 内部架构

### 组件本体
- **`UDisableComponent`（UActorComponent）**：挂在演员上的自动禁用控制器。
  - 字段：`bAutoDisable`、`AutoDisableRange`（默认 5000）、`AutoDisableLinkedActors`（连带禁用的关联演员）、`bActorIsVisualOnly`（仅视觉则本地判定、不联网）、`bStartDisabled` + `StartDisabledInstigator`。
  - 距离判定 `UpdateAutoDisable(ClosestDistanceSq)`：取到两玩家中较近距离，超过 `AutoDisableRange²` 则应禁用。
  - **联机**：非视觉演员只在控制端判定，经 `CrumbUpdateAutoDisableState`（`CrumbFunction`）同步；并限频（两次状态翻转间隔需 > 0.5s）。视觉演员本地直接 `UpdateAutoDisableState`。
  - 实际禁用经 `Owner.AddActorDisable(this)` / `RemoveActorDisable(this)`，对关联演员同样处理（`AppliedLinkedDisables` 记录）。
  - 支持运行时增删关联演员（`LateAdd/RemoveAutoDisableLinkedActor`）与对外的整组启停（`AddActorDisableToActorAndLinkedActors`）。

### 全局单例（分帧轮转）
- **`UDisableComponentSingleton`（UHazeSingleton）**：集中管理所有启用自动禁用的组件。
  - 注册：组件 `BeginPlay` 调 `RegisterToSingleton()` 加入 `AutoDisableComponents`，记录 `SingletonRegistrationIndex`；`EndPlay` 时以“尾元素交换删除”高效注销。
  - **分帧轮转**：每帧只更新 `Math::Max(总数/30, 15)`（且不超过总数）个组件，用 `UpdateIndex` 滚动遍历，摊平开销；暂停或缺玩家时跳过。
  - 新注册组件（`NewlyAddedAutoDisableComponents`）当帧立即更新一次，避免延迟。

### 分组与调试
- **`AGroupedDisableActor`（AHazeActor）**：自带 `UDisableComponent`（默认开启、范围 10000）的分组禁用锚点。编辑器可按半径 `AddActorsInRadius()` 批量把范围内指定类型演员加入 `AutoDisableLinkedActors`。
- **DisableStatics（mixin）**：`TemporalLogAllDefaultDisableLogic`，把演员/组件的禁用、阻塞（Tick/视觉/碰撞）状态写入 temporal log，仅 `#if TEST`。
- **`DisableComponentVisualizer.as`**：编辑器可视化禁用范围/关联演员。

### 数据流
组件 `BeginPlay` 注册单例 → 单例每帧轮转算到玩家距离 → 超范围则控制端 Crumb 同步 `UpdateAutoDisableState` → `Owner.AddActorDisable` + 连带关联演员；玩家靠近则反向移除。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 注册 | `UDisableComponentSingleton` | 单例 | `Game::GetSingleton(UDisableComponentSingleton)`，加入 `AutoDisableComponents` 并记录索引 |
| 调用 | 宿主演员 | 直接调用 | `Owner.AddActorDisable(this)` / `RemoveActorDisable(this)` 落实禁用 |
| 同步 | 远端 | Crumb | `CrumbUpdateAutoDisableState(bShouldBeDisabled)` 联网同步禁用状态（视觉演员除外） |
| 读取 | Progress | 直接调用 | `BeginPlay` 用 `Progress::HasActivatedAnyProgressPoint()` 决定首帧是否更新 |
| 内嵌 | `AGroupedDisableActor` | 组合 | 分组演员以 `DefaultComponent UDisableComponent` 默认开启自动禁用 |
| 被使用 | Audio/CameraBoundary 等 | 组合 | 多模块以 `UDisableComponent` 实现远距禁用（如 `PlayerAudioVolumeBase`、相机样条边界碰撞组件） |

## 关键文件

- `DisableComponent.as`：`UDisableComponent`（远距自动禁用 + 关联演员 + 联机限频）与 `UDisableComponentSingleton`（分帧轮转单例）。
- `GroupedDisableActor.as`：`AGroupedDisableActor`，自带禁用组件的分组锚点，可按半径批量关联演员。
- `DisableStatics.as`：`TemporalLogAllDefaultDisableLogic` 调试 mixin，记录禁用/阻塞状态。
- `DisableComponentVisualizer.as`：编辑器禁用范围与关联演员可视化。

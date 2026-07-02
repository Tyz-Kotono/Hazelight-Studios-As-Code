# Core / Camera / CameraBoundaryComponents

> 职责：为样条跟随相机提供动态碰撞边界——在相机更新时移动一组阻挡碰撞盒，防止玩家越界跑出相机取景范围。共 1 文件。

## 内部架构

| 类 | 基类 | 角色 |
| --- | --- | --- |
| `UCameraSplineFollowBoundaryCollisionComponent` | `UHazeCameraResponseComponent` | 主体：随相机/聚焦目标动态摆放并启停碰撞盒 |
| `ACameraSplineFollowBoundaryCollisionActor` | `AHazeActor` | 被生成的阻挡体（含 `UBoxComponent`，剔除可行走/攀爬等标签） |
| `UCameraSplineFollowBoundaryCollisionComponentVisualizer` | `UHazeScriptComponentVisualizer` | 编辑器内绘制碰撞盒预览 |

关键数据结构：
- `FCameraSplineFollowBoundaryColliderData` —— 单个碰撞盒配置（位置/旋转相对类型、样条距离偏移、盒尺寸、碰撞配置名、生成的 `CollisionActor`）。
- `FCameraSplineFollowBoundaryCollisionContextData` —— 运行/编辑共用的上下文（世界 Up、样条、相机 Actor、玩家、UserComp），统一取相机变换与聚焦目标位置。
- 多个枚举定义碰撞盒的位置/旋转参考系（聚焦目标 / 样条最近点 / 相机 / 水平相机）。

数据流：`BeginPlay` 校验宿主为 `ASplineFollowCameraActor` 并 `SpawnColliders()`（每个盒生成一个 Actor，初始 `AddActorDisable(this)`）。引擎在相机更新/快照时回调 `OnCameraUpdateForUser` / `OnCameraSnapForUser` → `HandleMoveColliders` 按上下文重算每个盒的 `FTransform` 并赋给对应 Actor。`UpdateCollidersEnabled` 依据「是否有禁用 Instigator、激活的 UserComp 数量、是否有人待全屏」决定整体启停（`IsCollisionEnabled`：双人各占半屏时禁用，仅在单人/待全屏时启用）。

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 调用 | `ASplineFollowCameraActor`（主模块 `Camera/CameraActors`） | 1 | `Cast<ASplineFollowCameraActor>(Owner)`，读 `GetSplineToUse()` 与 `FocusTargetComponent` |
| 调用 | `UCameraWeightedTargetComponent`（主模块 `CameraFocusTarget`） | 1 | `FocusTargetComponent.GetFocusTargets(Player)` / `GetEditorPreviewTargets()` 取聚焦中心 |
| 调用 | `UHazeSplineComponent` | 1 | 取样条最近点/距离/旋转用于摆放碰撞盒 |
| 调用 | `ACameraSplineFollowBoundaryCollisionActor` | 1 | 生成、设变换、`AddActorDisable/RemoveActorDisable` 启停 |
| 被调用 | 引擎相机更新管线 | 1 | 经 `UHazeCameraResponseComponent` 回调 `OnCameraUpdateForUser` / `OnCameraSnapForUser` / `OnCameraActivated/Deactivated` |
| 被调用 | 外部脚本 | 1 | `DisableCollision(Instigator)` / `EnableCollision(Instigator)`（Instigator 计数门控） |

> 强依赖关系：本模块是相机主模块的扩展插件，`devCheck` 强制宿主为 `ASplineFollowCameraActor`，不可独立使用。

## 关键文件

- `CameraSplineFollowBoundaryCollisionComponent.as` —— 样条跟随相机的动态碰撞边界组件、被生成的阻挡 Actor 及编辑器可视化。

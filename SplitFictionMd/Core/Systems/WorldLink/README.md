# Core / Systems / WorldLink

> 职责：编辑器中跨「双世界」actor 互链——在 Mio/Zoe 两套世界间建立成对引用，并按世界锚点同步定位。共 3 文件（编辑器期）。

## 内部架构

Split Fiction 双人玩法常需在两套并行世界放置「对应」actor。本模块在编辑器里维护这种成对关系并自动同步位置。

### 关键类型

| 类型 | 基类 | 角色 |
| --- | --- | --- |
| `UHazeWorldLinkComponent` | `UActorComponent` | 持有 `TSoftObjectPtr<AActor> LinkedActor`，负责建立/解除双向链接 |
| `UHazeEditorWorldLinkPositionComponent` | `USceneComponent` | actor 移动时按世界锚点把链接对象同步到镜像位置 |
| `AHazeWorldLinkSyncedPosition` | `AHazeActor` | 现成的链接 actor：组合上述两个组件 + 公告板 |
| `UHazeWorldLinkComponentVisualizer` | `UHazeScriptComponentVisualizer` | 在两端之间画黄色连线 |

### 数据流

- **建链 `LinkUpWorlds`**：校验 `LinkedActor` 有效；若目标无链接组件则沿 `AttachParentActor` 上溯找可链接父级（`GetFirstLinkableParent`）；排除自引用；让对端先 `DropReferenceMutual` 解旧链；`DropAllWorldLinkReferences(Owner)` 清掉所有指向自己的旧引用；最后令对端 `LinkedActor` 指回自己，形成双向引用。
- **解链 `DropReferenceMutual`**：先令对端清掉对自己的引用，再重置自身。
- **同步定位 `OnActorModifiedInEditor`**（位置组件）：取自身世界位置最近锚点 `WorldLink::GetClosestAnchor` 与其对侧 `GetOppositeAnchor`，按「相对锚点的偏移」把链接对象镜像到另一世界对应位置，随后调用 `LinkUpWorlds` 刷新链接。

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 调用 | `WorldLink::GetClosestAnchor` / `GetOppositeAnchor` | 直接调用 | 依赖世界锚点系统换算两世界对应坐标 |
| 调用 | `Editor::GetAllEditorWorldActorsOfClass` | 直接调用 | `DropAllWorldLinkReferences` 遍历清理旧引用 |
| 自洽 | 三文件互相协作 | 组件组合 | `AHazeWorldLinkSyncedPosition` 把链接组件、定位组件、可视化器组合为可放置 actor |
| 概念相邻 | `Network/` | 区分 | 本模块是**编辑器期**内容关联，与运行时联机同步无关 |

## 关键文件

- `HazeWorldLinkComponent.as` — `UHazeWorldLinkComponent`：`LinkUpWorlds` / `DropReferenceMutual` / `GetFirstLinkableParent` 维护双向软引用；`DropAllWorldLinkReferences` 全局清理。
- `HazeEditorWorldLinkPositionComponent.as` — `UHazeEditorWorldLinkPositionComponent`：actor 改动时按锚点把链接对象镜像到另一世界并触发重链。
- `HazeWorldLinkSyncPositionActor.as` — `AHazeWorldLinkSyncedPosition` 组合 actor + `UHazeWorldLinkComponentVisualizer`（两端黄线可视化）。

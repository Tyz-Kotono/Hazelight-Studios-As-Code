# Core / Flow / Progress（进度检查点）

> 职责：作为关卡检查点触发体，任一玩家首次进入时激活指定进度点；默认禁用、需由关卡蓝图显式启用，且经 Crumb 保证联机仅触发一次。共 1 个文件。

## 内部架构

- **`AProgressPointTrigger`（AVolume）**：检查点触发体积。
  - 关键字段：`ActivateProgressPoint`（`FHazeProgressPointRef`，要激活的进度点）、`bTriggerForMio` / `bTriggerForZoe` 门控、私有 `bEnabled`（默认 `false`）与 `bTriggeredProgressPoint`（单次防重）。
  - **默认禁用**：必须由关卡蓝图调用 `EnableProgressPointTrigger()` 才生效；启用瞬间会用 `TraceOverlappingComponent` 重检已在体积内的控制端玩家，立即补触发。
  - **联机单次**：进入由控制端（`Player.HasControl()`）经 `CrumbTriggerProgressPoint`（`UFUNCTION(CrumbFunction)`）同步到双端，`bTriggeredProgressPoint` 确保进度点只激活一次。
  - 激活调用 `Progress::ActivateProgressPoint(Progress::GetProgressPointRefID(ActivateProgressPoint))`。
  - 编辑器侧 `UProgressPointTriggerDetails` 自动重命名 Actor 标签（`ProgressPointTrigger → 目标 (关卡)`）并提示两条注意事项：必须从关卡蓝图启用、目标进度点须先 prepare。

### 数据流
关卡蓝图 `EnableProgressPointTrigger()` → 玩家进入（控制端校验 + Mio/Zoe 门控）→ `CrumbTriggerProgressPoint` 双端同步 → `Progress::ActivateProgressPoint` 激活 → 标记 `bTriggeredProgressPoint` 不再触发。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 调用 | 进度系统 Progress | 直接调用 | `CrumbTriggerProgressPoint` 调 `Progress::ActivateProgressPoint(Progress::GetProgressPointRefID(...))` |
| 同步 | 远端 | Crumb | `UFUNCTION(CrumbFunction) CrumbTriggerProgressPoint` 把进度激活同步双端 |
| 被启用 | 关卡蓝图 | 委托/直接调用 | `EnableProgressPointTrigger()` 须由关卡蓝图调用（详情面板明确警示） |
| 重检 | 玩家胶囊 | TraceOverlappingComponent | 启用时 `Player.CapsuleComponent.TraceOverlappingComponent(BrushComponent)` 补触发已在内玩家 |

## 关键文件

- `ProgressPointTrigger.as`：`AProgressPointTrigger`，默认禁用的检查点触发体，单次激活进度点；含编辑器详情自定义 `UProgressPointTriggerDetails`。

# Core / SplineAndSpace / Collision

> 职责：玩家碰撞开关、形状/射线查询的便捷封装，以及为大规模球体重叠测试服务的静态空间索引。共 5 个文件。

## 内部架构（关键类/基类/数据结构/数据流）

本模块是"碰撞与空间查询"的工具集，彼此相对独立，可分三组：

- **玩家碰撞开关**：`PlayerCollisionCapability.as`（`UPlayerCollisionCapability : UHazeMarkerCapability`，标签 `CapabilityTags::Collision`）。当 marker 被阻塞（`OnMarkerBlocked`）时，把玩家胶囊切到 `PlayerCharacterOverlapOnly` 碰撞档（机制 3：标签阻塞 + 碰撞 profile 覆盖），并递归遍历附着在玩家身上的组件/Actor，对它们调用 `AddActorCollisionBlock`/`AddComponentCollisionBlocker`；`OnMarkerUnblocked` 时全部恢复。会主动跳过另一玩家（另一玩家碰撞由其自身管理——双人 Mio/Zoe 实证）。
- **查询便捷封装**（mixin）：
  - `OverlapStatics.as`：在 `UHazeCapsuleCollisionComponent`/`UBoxComponent` 上挂 `IsPointInside`/`IntersectsSphere`，内部转发到 `FHazeShapeSettings::Make*`（Shapes 域）与 `Overlap::QueryShapeOverlap`。
  - `PlayerTraceHelpers.as`：`namespace Trace` + mixin。`InitFromPlayer`/`TraceWithPlayer` 构造"以玩家身份移动"的 `FHazeTraceSettings`（即便玩家碰撞当前被阻塞，仍按正常碰撞返回命中）；`IgnorePlayers` 一键忽略 `Game::Mio`/`Game::Zoe`；底层走 `UHazeMovementComponent` + `MovementTrace::Init`。
- **静态空间索引**：`StaticSparseSphereGrid.as`（`FStaticSparseSphereGrid`）。一个"烘焙并扁平化的四叉树"，运行时 `GetOverlappingSphere`/`HasOverlappingSphere` 用先 X 后 Y 的二分查找快速判断测试球是否与海量实例球相交；`#if EDITOR` 的 `Generate`/`Build_*` 负责离线构建（按坐标排序、递归分箱）。
- **海洋波数据接入**：`OceanWaves.as`（`namespace OceanWaves`）。通过唯一的 `AOceanWavePaint` Actor（`TListedActors`）请求/查询某 `FInstigator` 在某世界坐标处的波高数据（`RequestWaveData`/`IsWaveDataReady`/`GetLatestWaveData`，机制 4：以 Instigator 为键的异步请求-回读）。

## 协同关系（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 调用 | `Shapes/ShapeStatics`（`FHazeShapeSettings`） | 直接调用 | `OverlapStatics` 的 `IsPointInside` 转发到 Shapes 域构造形状 |
| 调用 | 引擎 `Overlap::QueryShapeOverlap` | 直接调用 | 形状-球体相交测试底座 |
| 调用 | `Movement` 的 `UHazeMovementComponent`/`MovementTrace` | 直接调用 `Type::Get` | `Trace::TraceWithMovementComponent` 以运动组件代表 Actor 做射线 |
| 被调用 | `PlayerHealth/PlayerHealthStatics` | `Overlap::` | 血量域复用重叠查询（grep `Overlap::` 命中） |
| 被调用 | `Targetable/TargetableHelpers` | `Trace::`/`IgnorePlayers` | 可锁定目标域复用玩家射线助手（grep 命中） |
| 被调用 | `Aiming/PlayerAimingStatics`、`Audio/Volumes/FoliageDetectionVolume`、`Movement/SplineLock/SplineLockStatics` 等 | `GetOverlappingSphere` | 多域消费 `FStaticSparseSphereGrid` 做快速球体重叠（grep 命中 10 文件） |
| 依赖 | `AOceanWavePaint`（唯一实例） | `TListedActors` + Instigator 请求 | 波数据查询要求场景中恰有一个该 Actor |
| 阻塞 | 玩家胶囊 `CapsuleComponent` | 标签阻塞 + profile 覆盖 | `OnMarkerBlocked` 切 `PlayerCharacterOverlapOnly`，`PlayerTraceHelpers` 对此档位特判 |

## 关键文件（逐文件一句话）

- `PlayerCollisionCapability.as`：marker 被阻塞时把玩家及其附属物切到"仅重叠"碰撞档并递归屏蔽，解除时恢复。
- `OverlapStatics.as`：胶囊/盒体的"含点"与"与球相交"便捷 mixin，转发到 Shapes 与 `Overlap::`。
- `PlayerTraceHelpers.as`：以玩家/运动组件身份构造射线设置的 `Trace` 命名空间与 mixin，含忽略双人玩家。
- `StaticSparseSphereGrid.as`：烘焙扁平四叉树，运行时二分查询海量球体重叠（编辑器内构建）。
- `OceanWaves.as`：通过唯一 `AOceanWavePaint` 按 Instigator 请求/回读海洋波高数据。

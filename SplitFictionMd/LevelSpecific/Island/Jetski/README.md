# LevelSpecific / Island / Jetski（水上摩托）

> 路径：`LevelSpecific/Island/Jetski/`（83 文件）

## 职责

Island 的招牌载具玩法。玩家骑水上摩托穿越淹水走廊、下潜避障、被追赶时靠橡皮筋追赶（rubberbanding）保持双人同框，中途穿插巨型海怪（触手 / 章鱼 / 乌贼）的过场剧本。载具是一台 **四态样条载具**：`Air / Ground / Water / Underwater`（`EJetskiMovementState`）四种移动状态随所处环境自动切换，沿关卡样条 `JetskiSpline` 前进，波浪由 Perlin 噪声驱动。

## 内部架构

- **载具实体** `AJetski : AHazeActor`（Abstract）：网格 + 偏移组件 + 水面特效贴花 + 驾驶员挂点 `DriverAttachment` 的表现载体。`access` 私有授权给移动解算器 / 死亡 / 动画。核心枚举 `EJetskiMovementState`（四态）与 `EJetskiUp`（多种"上方向"来源：地面法线 / 水面 / 波浪法线 / 样条等）。
- **玩家挂接** `UJetskiDriverComponent : UActorComponent`：持有 `JetskiClass`，`SpawnActor` 生成水上摩托并把玩家绑到 `DriverAttachment`。玩家侧能力集 `Player/Capabilities/`：`JetskiDriverCapability`（主控，`ApplySettings(JetskiPlayerHealthSettings)`）、Attach / Animation / Death / Dead / BlockRespawn / Sequence。教学 `Player/Tutorial/`（加速 / 下潜）。
- **移动系统**：
  - `Components/JetskiMovementComponent : UHazeMovementComponent` — 移动组件底座；`JetskiWaterPlaneComponent` / `JetskiWaterSampleComponent` 采样水面高度。
  - `Movement/`：`JetskiMovementResolver`（解算）+ `JetskiMovementData` + `JetskiMovementSettings`。
  - `Capabilities/Movement/`：按四态各一套——`JetskiAirMovementCapability` / `JetskiAirDiveMovementCapability` / `JetskiGroundMovementCapability` / `JetskiWaterMovementCapability` / `JetskiUnderwaterMovementCapability`(+State/+FollowSpline)。`JetskiBlockDiveCapability` 门控下潜。
  - `Steering/`：`JetskiSteeringRotateCapability`（自由转向）/ `JetskiSteeringSplineCapability`（贴样条转向）。
- **样条与区域** `Spline/`：`JetskiSpline` 主路径，一族 zone 组件按段开启行为——`DiveZone`（可下潜）/ `UnderwaterZone`（强制水下）/ `ForceThrottle`（强制油门）/ `IgnoreRubberBanding`（关闭追赶）/ `AllowAligningWithCeiling`（贴顶）。`RespawnSpline/` 死亡后沿样条重生。`OverrideWaterHeight/` 局部改写水面高度。
- **波浪与浮动** `JetskiPerlinWaves`（Perlin 波浪）+ `Bobbing/`（`JetskiBobbingCapability` + Component 上下颠簸）。
- **相机** `Camera/` + `Capabilities/Camera/`：样条相机 `JetskiSplineCameraCapability` / 自由相机 `JetskiFreeCameraCapability`，配 `JetskiCameraOverrideSpline` / `Volume`。
- **视觉事件** `Capabilities/VisualEvents/`：空中 / 水面 / 水下三态特效；`Capabilities/Audio/` 水下音频。
- **关卡剧本** `LevelEvents/`：一次性演出——大浪 `JetskiBigWave`、酸液泄漏、碎窗、关门、走廊涌水、坠船、移动尖刺、定时坠物；以及海怪 `JetskiSquidArm`(+Curved/+SequenceActor) / `JetskiUnderwaterSquid` / `OctopusTentacleSequenceActor`。
- **联机** `JetskiSyncComponent : UHazeCrumbSyncedStructComponent`（`SyncRate = Standard`）同步载具状态。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| 依赖 | Core 移动 | `UHazeMovementComponent` | `JetskiMovementComponent : UHazeMovementComponent`，四态移动能力的底座 |
| 依赖 | Core 联机 | Crumb 同步 | `JetskiSyncComponent : UHazeCrumbSyncedStructComponent`；`JetskiDriverCapability` 走 Crumb |
| 依赖 | Core/Flow 设置 | ApplySettings | `JetskiDriverCapability.OnActivated` → `Player.ApplySettings(JetskiPlayerHealthSettings, this)` |
| 生成 | 载具实体 | SpawnActor | `UJetskiDriverComponent::SpawnJetski` → `SpawnActor(JetskiClass, ...)` 并绑定 `DriverAttachment` |
| 依赖 | 样条 zone 组件 | 分段门控 | `Spline/` 各 zone 组件切换下潜 / 强制油门 / 关闭橡皮筋等段落行为 |
| 触发 | LevelEvents 剧本 | 过场演出 | 大浪 / 海怪触手 / 坠船等由 `LevelEvents/` 的 SequenceActor 编排 |

## 关键文件

- `LevelSpecific/Island/Jetski/Jetski.as` — 载具实体基类 `AJetski` + `EJetskiMovementState`/`EJetskiUp` 枚举
- `LevelSpecific/Island/Jetski/Player/Components/JetskiDriverComponent.as` — 玩家挂接 + 载具生成
- `LevelSpecific/Island/Jetski/Player/Capabilities/JetskiDriverCapability.as` — 玩家主控能力
- `LevelSpecific/Island/Jetski/Components/JetskiMovementComponent.as` — 移动组件底座
- `LevelSpecific/Island/Jetski/Movement/JetskiMovementResolver.as` — 四态移动解算
- `LevelSpecific/Island/Jetski/Spline/` — 主样条 + 分段 zone 组件
- `LevelSpecific/Island/Jetski/LevelEvents/` — 大浪 / 海怪 / 坠船等剧本
- `LevelSpecific/Island/Jetski/JetskiSyncComponent.as` — 载具状态联机同步

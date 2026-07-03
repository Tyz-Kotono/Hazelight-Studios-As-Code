# LevelSpecific / Dentist / Player（牙齿玩家能力）

> 路径：`LevelSpecific/Dentist/Player/`（98 文件）

## 职责

这一关玩家不是常规角色，而是被变成一颗**会跳的牙齿**（Tooth）。本目录实现牙齿这套该关专属的玩家玩法：自定义地面/空中移动、跳跃（含二段/三段翻转）、冲刺、砸地（打 Boss 弱点的核心手段）、被弹射炮/双炮发射、被 Boss 劈成两半后的**分裂**（一半玩家控制、一半 AI 逃跑）、以及落水死亡。整套完全替换了 Gameplay 的默认玩家移动，走自定义 Resolver。

## 内部架构

### 变身入口 `UDentistToothPlayerComponent`

牙齿身份中枢。`SpawnAndAttachTooth`：覆盖胶囊尺寸（`Dentist::CollisionRadius/Height`）、把 `MeshOffsetComponent` 对齐到胶囊中心以便**整体旋转牙齿网格**、生成视觉体 `ADentistTooth`（带活动眼球 `GooglyEye`）、调高重力/终端速度/台阶尺寸、施加 blob 阴影设置。核心 API `SetMeshWorldRotation / AddMeshWorldRotation`（用 `FHazeAcceleratedQuat` 平滑）供所有牙齿能力旋转网格。还记录被工具约束的状态标志（`bHooked / StruckByHammerFrame / bDrilled`）。

### 移动四件套 `Movement/`

自定义移动栈，遵循 Move 系统的 Data/Resolver/Settings 分层：

- `UDentistToothMovementResolver : USteppingMovementResolver`（+ `DentistToothMovementData` / `Settings`）——牙齿自己的步进移动解算，处理不稳定边缘不可走等。
- `DentistToothGroundMovementCapability` / `DentistToothAirMovementCapability`——地面/空中移动能力。
- `DentistToothMovementResponseComponent`——被外部（如砸地、被撞）影响的响应入口。
- 表现层能力 `Capabilities/`：`Animation`（读动画值）、`InputTilt`（输入倾斜牙齿）、`MeshRotation`（网格朝向）、`Outline`、`Possess`、`Sequence`。

### 动作能力群

每个动作一组能力四件套（Component + Capabilities + Settings，多带 EventHandler）：

- **Jump** `Jump/`：`JumpInput → Jump → JumpTo` + `Rotation/`（`DoubleJump` / `TripleJump` 翻转）。
- **Dash** `Dash/`：`DashInput → Dash → DashLand → DashBackflip`，带 `DashAutoAimComponent`（自动瞄准）与专属 `DashMovementResolver`；`DashCandleTutorial/` 是用蜡烛教冲刺的教学。
- **GroundPound** `GroundPound/`：`Anticipation → Drop → Recover`，`DentistGroundPoundTargetableComponent` 标记可砸目标（Boss 双手弱点即用它）。**这是对 Boss 输出伤害的主要途径。**
- **Bounce** `Bounce/`：踩弹跳道具（樱桃/棒棒糖等）的弹跳。
- **Ragdoll** `Ragdoll/`：被击飞后布娃娃 + 布娃娃状态下的空/地移动。
- **Cannon / DoubleCannon**：被单炮/双炮发射的锁定→发射→落地流程（配合 `LevelActors/Cannon`）。
- **WaterDeath** `WaterDeath/`：落水死亡与复位（`WaterLandscapeHandle`）。

### 分裂系统 `Split/`（该关最独特）

Boss 用刮匙+锤子把牙齿劈成两半时触发。分三块：

- `Split/Player/`：`UDentistToothSplitCapability`（`NetworkMode = Crumb`）在 `bShouldSplit` 时把玩家一分为二——玩家控制左半（`SplitTooth Player`），生成 AI 控制右半（`ADentistSplitToothAI`），用 `Trajectory::CalculateVelocityForPathWithHeight` 把两半抛向 `ADentistSplitToothTarget` 的落点。`SplitRecombine` 负责重新合并。
- `Split/SplitTooth/AI/`：逃跑 AI，自带 `EDentistSplitToothAIState` 状态机（Splitting / Idle 乱跳 / Startled 转身 / Scared 逃跑 / Recombining），一组状态能力 + `CircleConstraint`（限制活动范围）。
- `Split/SplitTooth/Shared/`：玩家半与 AI 半共用的移动能力（`AirMovement` / `GroundIdle` / `GroundLunge`）与组件。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| → | Core Move 系统 | 复用 + 覆盖 | `UDentistToothMovementResolver : USteppingMovementResolver`；`UMovementGravity/Stepping/TerminalVelocity Settings` 全走 Move 系统 API |
| → | 玩家网格/胶囊 | 变身覆盖 | `Player.CapsuleComponent.OverrideCapsuleSize`、`MeshOffsetComponent.SnapToRelativeLocation`（牙齿整体旋转） |
| → | Boss 弱点 | 砸地输出 | `GroundPound` + `DentistGroundPoundTargetableComponent` 命中 Boss 手部 → Boss 侧 `OnGroundPoundedOn` |
| ← | Boss 工具 | 被约束/触发分裂 | `bHooked/bDrilled/StruckByHammerFrame` 由刮匙/钻头/锤子设置；锤子劈牙 → `SplitComp.bShouldSplit` |
| → | 联机 | Crumb | `UDentistToothSplitCapability` `NetworkMode = Crumb`，分裂状态全网同步 |
| → | 轨迹工具 | 抛物线 | 分裂抛射用 `Trajectory::CalculateVelocityForPathWithHeight` |
| ← | 关卡机关 | 发射/弹跳 | `Cannon/DoubleCannon` 能力配合 `LevelActors/Cannon`；`Bounce` 配合弹跳甜品道具 |

## 关键文件

- `Player/DentistToothPlayerComponent.as` — 牙齿变身中枢：生成牙齿、覆盖胶囊、网格整体旋转
- `Player/DentistTooth.as` — 牙齿视觉体 `ADentistTooth`（含活动眼球）
- `Player/Movement/DentistToothMovementResolver.as` — 牙齿自定义步进移动解算
- `Player/GroundPound/DentistToothGroundPoundStateCapability.as` + `DentistGroundPoundTargetableComponent.as` — 砸地（打 Boss 弱点核心）
- `Player/Split/Player/DentistToothSplitCapability.as` — 玩家一分为二（Crumb 同步）
- `Player/Split/SplitTooth/AI/DentistSplitToothAI.as` — 逃跑的 AI 半，自带状态机
- `Player/Split/SplitTooth/DentistSplitToothTarget.as` — 分裂两半的落点目标
- `Player/DentistSettings.as` — 牙齿玩法数值/常量

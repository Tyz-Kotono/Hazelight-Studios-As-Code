# Core / Audio / Movement

> 职责：玩家移动相关音频管线——脚步/手部触地与滑行（Footstep）以及角色发力呼吸/吭声及其恢复（Effort/Recovery）。共 8 个文件。

## 内部架构（关键类、基类、数据流）

本目录围绕两条独立的音频线展开，共享 `UHazeMovementAudioComponent`（引擎层组件，提供移动标签 `GetActiveMovementTag`、阻塞开关 `RequestBlock/UnBlock`、各类能力开关 `CanPerformMovement(EMovementAudioFlags)`）作为数据源。

### 一、脚步线（Footstep / Hand / Slide）

- **`UMovementAudioEventHandler`（`MovementAudioEventHandler.as`）**：继承自引擎 `UHazeEffectEventHandler`。它是一组 `BlueprintEvent` 空实现的“接收口”，由动画/移动系统通过 Effect Event 机制 **Trigger**（与同仓 `UFoliageDetectionEventHandler`、`UActorSpotAudioTriggerVolumeEventHandler` 同模式）。事件覆盖：脚步触地左右（`OnFootstepTrace_Left/Right`）、手部触地左右（`OnHandTrace_Left/Right`）、脚/手滑行的 Start/Stop/Loop/Tick，以及甩臂 `StartArmswing/StopArmswing`。蓝图子类负责实际播放声音。
- **参数结构体（同文件）**：`FPlayerFootstepParams`、`FPlayerHandImpactParams`、`FPlayerHandSlideAudioParams`、`FPlayerFootSlide(Start/Stop/Tick)Params` 等，承载每次事件的物理材质（`UPhysicalMaterialAudioAsset`）、材质事件（`UHazeAudioEvent`）、增益/音高、冲击点法线、速度等。
- **`FootstepAudioTypes.as`**：纯类型层。各角色的脚步枚举（玩家 `EFootType/EHandType/EHandTraceAction`，以及 Dragon/TundraMonkey/TreeGuardian/FantasyOtter/Pig 等坐骑/生物专属枚举），外加可组合的射线检测设置 `FAudioFootTraceSettings`（trace 形状、socket、长度随速度映射、触发冷却、增益/音高、共享冷却等）。
- **`MovementAudioStatics.as`**：`MovementAudio` 命名空间的静态库。集中定义各角色的 socket/bone 名常量；封装能力查询（`CanPerformFootsteps/Armswing/Breathing/Efforts`）、移动状态查询（`GetMovementState`）与阻塞请求（`RequestBlock/RequestUnBlock`）转发到 `UHazeMovementAudioComponent`。
- **地面材质获取**：脚步取地面物理材质并非本目录实现，而是调用 `Audio::Materials::GetGroundAudioPhysMat(Actor)`（`Core/Audio/AudioStatics.as`），它经由 `UHazeMovementComponent` 的地面接触 `GetGroundContact()` 做射线命中，再取 `UPhysicalMaterial.AudioAsset`，回退到 `UPhysicalMaterialAudioAsset` 的 DefaultObject。

### 二、发力线（Effort / Recovery）

数据驱动的“疲劳/喘息”模型：玩家持续移动累积 exertion（劳累值 0~100），停下后随时间恢复，并据强度播放不同呼吸/吭声。

- **`UPlayerEffortAudioComponent`（`PlayerEffortAudioComponent.as`，继承 `UActorComponent`）**：核心状态机与数据中枢。`BeginPlay` 中订阅 `MoveAudioComp.OnMovementTagChanged` 委托，在移动标签进入时 `PushEffortData(Group, Tag)` 从设置资产匹配出 `FEffortData` 作为当前发力；维护 `Exertion`、`EffortCurveAlpha/RecoveryCurveAlpha`、按强度分桶的待恢复队列 `Recoveries`、压力来源 `StressInstigators`。提供阈值/曲线取值（`GetEffortMaxClampValue`、`GetValueFromCurve`）、消费/恢复流转（`ConsumeEffort/PushRecoveryData/ConsumeRecoveredEffort`）。还暴露 mixin 静态 API：`RequestPlayerStress/UnRequestPlayerStress/PlayerIsInStress`。通过 `access` 限定符把曲线 alpha 等私有成员只开放给两个 Capability。
- **`UPlayerEffortAudioCapability`（`PlayerEffortAudioCapability.as`，继承 `UHazePlayerCapability`）**：TickGroup=Audio，Order=0。当组件 `HasPendingEffort()` 时激活，将发力置 `Handling`，按 `Immediate`/`Continuous` 推送类型逐帧沿 `EffortCurve` 累积劳累值（含坡度倾斜倍率 `GetSlopeTiltMultiplier`），移动中断或达上限时消费并转为待恢复。`OnLogState` 输出 TemporalLog 调试。
- **`UPlayerEffortRecoveryAudioCapability`（`PlayerEffortRecoveryAudioCapability.as`，继承 `UHazePlayerCapability`）**：TickGroup=Audio，Order=1（排在发力之后）。当组件 `HasPendingRecoveries()` 时激活，按 `RecoveryCurve` 与按强度区间选取的恢复倍率逐帧降低劳累值；含落地/空中、激活延迟、与当前更高强度发力的避让等门控（`CanTickRecovery`），恢复完毕调用 `ConsumeRecoveredEffort`。
- **`UPlayerEffortAudioSettings`（`PlayerEffortAudioSettings.as`，继承 `UHazePlayerEffortAudioSettings`）**：可组合设置资产。定义全局 ExertionFactor/RecoveryFactor、各强度（Low/Medium/High/Critical）的阈值与区间与恢复倍率、曲线倍率、坡度倍率，以及默认 `EffortCurve`/`RecoveryCurve`。
- **`PlayerEffortAudioTypes.as`**：本文件内容为整体注释（历史类型定义），当前 `FEffortData`、`FRecoveryEffortDatas`、`EEffortAudioIntensity/State/PushType` 等实际由引擎/C++ 层提供。

## 协同关系

| 方向 | 对象 | 机制 | 说明（基于实证） |
| --- | --- | --- | --- |
| 入 | 动画/移动系统 → `UMovementAudioEventHandler` | 委托（Effect Event Trigger） | 继承 `UHazeEffectEventHandler`，由播放系统 Trigger 其 `OnFootstepTrace_*` / `OnHandTrace_*` / Slide / Armswing 等 BlueprintEvent。 |
| 入 | `UHazeMovementAudioComponent.OnMovementTagChanged` → `UPlayerEffortAudioComponent` | 委托 event/Broadcast | `BeginPlay` 中 `AddUFunction(..., n"OnMovementTagChanged")`，标签进入时触发 `PushEffortData`。 |
| 入 | `UPlayerHealthComponent.OnReviveTriggered` → `UPlayerEffortAudioComponent` | 委托 event/Broadcast | 订阅复活事件 `OnPlayerRespawn` → `ClearRecoveries` 重置劳累。 |
| 出 | `UPlayerEffortAudioComponent` → `UPlayerEffortAudioSettings` | 可组合设置 ApplySettings | `BeginPlay` 调 `PlayerOwner.ApplyDefaultSettings(EffortsAsset)`；Recovery Capability 用 `UPlayerEffortAudioSettings::GetSettings(Player)` 读取。 |
| 出 | 两个 Capability → `UPlayerEffortAudioComponent` | 直接调用 Type::Get | `Setup` 中 `UPlayerEffortAudioComponent::Get(Player)` 取组件并读写其状态（受 `access` 限定）。 |
| 内 | Effort Capability ↔ Recovery Capability | 标签阻塞 CapabilityTags | 两者同 `CapabilityTags.Add(n"PlayerMovementEffortAudio")`，以 TickGroupOrder 0/1 协调先发力后恢复。 |
| 出 | `MovementAudioStatics` / Capability → `UHazeMovementAudioComponent` | 直接调用 Type::Get | 查询能力/移动标签、`RequestBlock/UnBlock`（带 Instigator）控制移动音频流。 |
| 出 | 脚步逻辑 → `Audio::Materials::GetGroundAudioPhysMat` | 直接调用（静态库） | 取地面 `UPhysicalMaterialAudioAsset` 填充 `FPlayerFootstepParams`。 |
| 出 | 外部系统 → 玩家压力 | 直接调用（mixin 静态） | `RequestPlayerStress / UnRequestPlayerStress / PlayerIsInStress` 经组件 `StressInstigators` 标记是否处于压力。 |

## 关键文件（逐文件一句话）

- **`FootstepAudioTypes.as`**：脚步/手部的各角色枚举与可组合射线检测设置 `FAudioFootTraceSettings`。
- **`MovementAudioEventHandler.as`**：`UMovementAudioEventHandler`（继承 `UHazeEffectEventHandler`）及各类脚步/滑行/手部事件参数结构体，作为动画系统触发的音频接收口。
- **`MovementAudioStatics.as`**：`MovementAudio` 命名空间静态库，集中 socket 常量与对 `UHazeMovementAudioComponent` 的能力查询/阻塞封装。
- **`PlayerEffortAudioCapability.as`**：`UHazePlayerCapability` 子类，逐帧累积玩家劳累值（含坡度倍率），将发力消费为待恢复。
- **`PlayerEffortAudioComponent.as`**：`UActorComponent` 子类，发力/恢复状态机与数据中枢，订阅移动标签与复活委托，暴露压力相关 mixin API。
- **`PlayerEffortAudioSettings.as`**：`UHazePlayerEffortAudioSettings` 子类，发力线的可组合设置资产（阈值、恢复倍率、曲线）。
- **`PlayerEffortAudioTypes.as`**：发力线历史类型定义（现整体注释，实类型由引擎层提供）。
- **`PlayerEffortRecoveryAudioCapability.as`**：`UHazePlayerCapability` 子类，TickGroupOrder=1，按恢复曲线逐帧降低劳累值并处理门控。

# Core / Audio / SpotSound

> 职责：在关卡里摆放的定点环境声/场景音源系统，通过池化发射器播放 Event 或生成 SoundDef，并支持多种空间定位模式（基础/多点/平面/样条/视线体积）。共 10 个文件。

## 内部架构（关键类、基类、数据流）

### 组件主体：USpotSoundComponent
`USpotSoundComponent`（继承 `UHazeSpotSoundComponent`）是整个系统的核心运行时组件，承载所有播放逻辑：

- **两种播放路径**：`UpdateSoundAsset()` 从 `AssetData` 解析出资产类型——
  - **Event 路径**：通过 `GetAudioComponentAndEmitter()` 调用 `Audio::GetPooledEmitter()` 取得池化发射器 `Emitter`，再 `SetupEmitter()` 配置后 `Emitter.PostEvent(Event, Ambience)`；停止时 `Audio::ReturnPooledEmitter()` 归还。
  - **SoundDef 路径**：构造 `FSpawnSoundDefSpotSoundParams` 并调用 `SoundDef::SpawnSoundDefSpot()`，由 SoundDef 运行时自行管理发射器（Spot 上不持有 Emitter）。对没有 HazeActor 宿主的场合（如 Volume）会临时 Spawn 一个 `WorkaroundOwner` 作为父级。
- **发射器配置 `SetupEmitter()`**：统一应用衰减缩放（`SetAttenuationScaling`）、Zone 遮挡联动（`GetZoneOcclusion`）、声音方向/距离/仰角追踪（`GetSoundDirection`/`GetDistanceToTarget`/`GetElevationAngle`）、空间声像（`SetSpatialPanning`）、默认 RTPC 与节点属性。这些开关由可组合的 `FSpotSoundEmitterSettings` 结构体驱动。
- **生命周期**：`BeginPlay` 时若 `bPlayOnStart` 则 `Start()`；支持依赖延迟启动（`SetPendingActor` + Tick 轮询等待 LinkedMeshOwner 加载）；`OnActorEnabled/Disabled` 在禁用区重新触发循环声。

### 模式多态体系：USpotSoundModeComponent
`Start()`/`Stop()`/`Tick()` 会委托给 `ModeComponent`（若存在），形成多态分发：

- `USpotSoundModeComponent`（`Abstract` 基类，继承 `UHazeSpotSoundModeComponent`）：定义虚函数 `Start/Stop/SetupMode/TickMode`，并在自身 `BeginPlay` 时反向把自己注册到 `ParentSpot.ModeComponent` 并触发 `ModeComponentStart()`。
- `USpotSoundBasicComponent`（Basic）：最简实现，仅持有一份 `FSpotSoundEmitterSettings`。Basic 模式实际由 `USpotSoundComponent.InternalStart()` 直接处理。
- `USpotSoundMultiComponent`（Multi）：管理多发射点 `MultiEmitters`。可由 SoundDef 的多 Emitter 数据驱动，或在 `bMultipleEmitterMode` 下每点独立 `FSpotSoundEmitterSettings`；用 `Audio::GetPooledAudioComponent` 取音频组件、`SetMultipleSoundPositions` 设置多点位置。
- `USpotSoundPlaneComponent`（Plane）：把声音吸附到关联网格表面——每帧对每个监听器用 `GetClosestPointOnCollision` 求最近点并 `SetMultipleSoundPositions`。
- `USpotSoundSplineComponent`（Spline）：把声音吸附到样条曲线——对每监听器求 `GetClosestSplinePositionToWorldLocation`，支持 `StartLoops` 多循环事件与 `AkMultiPositionType` 定位类型。

### 体积与预制
- `ASpotSoundPlaneVolume` / `ASpotSoundPlaneLookAtVolume`：`AVolume` 子类，作为 Plane 模式的绘制边界；后者内置完整的"视线追踪"逐帧算法（基于玩家视线方向投射、限幅 `LookAtSpread`、插值移动发射器位置）。
- `ASpotSound`：包装 `USpotSoundComponent` 的关卡 Actor，暴露 `Start/Stop/PostSpotEvent`。
- `APrefabSpotSound`：与 Prefab 系统对接的 Actor，`ApplyOnComponent()` 把预制参数批量写入内部 `USpotSoundComponent.Settings`。

数据流概览：`AssetData → UpdateSoundAsset → (Event | SoundDef) → ModeComponent 多态 Start → GetPooledEmitter/SpawnSoundDefSpot → SetupEmitter（衰减/追踪/RTPC/Zone）→ 逐帧 TickMode 更新空间位置`。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
| --- | --- | --- | --- |
| 出 | 池化发射器系统 | 直接调用 Type::Get | `Audio::GetPooledEmitter()` / `Audio::ReturnPooledEmitter()` / `Audio::GetPooledAudioComponent()` 取还发射器与音频组件（SpotSoundComponent.as、SpotSoundMultiComponent.as）。 |
| 出 | SoundDef 运行时 | 直接调用 Type::Get | `SoundDef::SpawnSoundDefSpot(Params)` 生成定点 SoundDef；`SoundDef::GetSoundDefEmitters()` 读取多发射点数据（SpotSoundComponent.as、SpotSoundMultiComponent.as）。 |
| 出 | 遮挡 Zone | 可组合设置 ApplySettings | `bLinkToZone` 时 `AudioComponent.GetZoneOcclusion(...)` 把 Spot 与 `LinkedZone` 联动（SpotSoundComponent.SetupEmitter）。 |
| 出 | 监听器（每玩家） | 直接调用 Type::Get | `Audio::GetListeners()` / `Emitter.GetListeners()` 取得监听器列表，Plane/Spline 据此逐帧计算各玩家最近声源位置（SpotSoundPlaneComponent.as、SpotSoundSplineComponent.as）。 |
| 入 | 模式组件 ↔ 主组件 | 委托 event/Broadcast | 主组件 `Start/Stop/Tick` 委托给 `ModeComponent`；模式组件 `BeginPlay` 反向注册 `ParentSpot.ModeComponent` 并订阅 `Emitter.OnEventStarted` 重启位置追踪（SpotSoundModeComponent.as、SpotSoundPlaneComponent.as）。 |
| 入 | Prefab 系统 | 可组合设置 ApplySettings | `APrefabSpotSound.ApplyOnComponent()` 把预制参数（衰减/RTPC/节点属性/Zone）写入 `USpotSoundComponent.Settings`（PrefabSpotSound.as）。 |
| 入 | 关卡专用音频 | 直接调用 Type::Get | 关卡脚本通过 `Cast<USpotSoundComponent>(GetAttachParent())` 取 Spot 的 Emitter/AudioComponent 做特化（LevelSpecific/Meltdown/SoftSplit/Audio/SoftSplitAudioSpotSoundComponent.as）。 |
| 入 | SoundDef 资产 | 直接调用 Type::Get | 多个 `*_SoundDef.as`（如 SpotSound_Waterfall_CloseDistant、Spot_Tracking_SoundDef）作为 Spot 播放内容被引用（Audio/SoundDefinitions/）。 |

## 关键文件（逐文件一句话）

- **SpotSound.as**：`ASpotSound` 关卡 Actor，包装 `USpotSoundComponent` 并暴露 `Start/Stop/PostSpotEvent` 蓝图接口。
- **SpotSoundComponent.as**：核心组件 `USpotSoundComponent` 及配置结构 `FSpotSoundEmitterSettings`，实现 Event/SoundDef 两路播放、池化发射器取还、追踪/RTPC/Zone 配置与模式委托。
- **SpotSoundModeComponent.as**：抽象基类 `USpotSoundModeComponent`，定义模式多态接口并在 BeginPlay 时反向注册到父 Spot。
- **SpotSoundBasicComponent.as**：`USpotSoundBasicComponent`（Basic 模式），仅承载一份发射器设置，实际播放由主组件直接处理。
- **SpotSoundMultiComponent.as**：`USpotSoundMultiComponent`（Multi 模式），管理多发射点位置与各自设置，支持 SoundDef 多 Emitter 数据驱动及单独启停。
- **SpotSoundPlaneComponent.as**：`USpotSoundPlaneComponent`（Plane 模式），把声源逐帧吸附到关联网格表面的最近点。
- **SpotSoundSplineComponent.as**：`USpotSoundSplineComponent`（Spline 模式），把声源逐帧吸附到样条曲线最近位置，支持多循环事件与多定位类型。
- **SpotSoundPlaneVolume.as**：`ASpotSoundPlaneVolume`，无重叠事件的简单体积，用于绘制 Plane 模式的边界。
- **SpotSoundPlaneLookAtVolume.as**：`ASpotSoundPlaneLookAtVolume`，内置玩家视线追踪算法的体积，把发射器位置投射并插值到玩家注视点。
- **PrefabSpotSound.as**：`APrefabSpotSound`，与 Prefab 系统对接的 Actor，`ApplyOnComponent()` 将预制参数批量套用到内部 Spot 组件。

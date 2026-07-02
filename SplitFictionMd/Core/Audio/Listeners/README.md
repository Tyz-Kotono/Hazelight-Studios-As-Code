# Core / Audio / Listeners

> 职责：一组玩家 Capability，竞争控制每玩家的 Wwise 监听器（UHazeAudioListenerComponent，即 Player.PlayerListener），决定空间音频从哪个位置/朝向收听。共 3 个文件。

## 内部架构（关键类、基类、数据流）

三个类均继承自 `UHazePlayerCapability`，`TickGroup = Audio::ListenerTickGroup`（即 `EHazeTickGroup::BeforeGameplay`），每帧在 `TickActive` 中对监听器 `SetWorldTransform`。

- **UPlayerDefaultListenerCapability**（默认监听器，常驻）
  - 标签：`Audio::Tags::Listener` + `Audio::Tags::DefaultListener`。
  - `ShouldActivate` 恒为 `true`、`ShouldDeactivate` 恒为 `false`，是兜底监听器。
  - 数据流：`TickActive` 先调 `Audio::SetPanningBasedOnScreenPercentage(Player)` 做分屏声像；随后判断是否由玩家控制相机——是则 `MoveTowardsInBetweenCameraTransform`（按移速 `MoveComp.Velocity` 在耳朵 `Audio::GetEarsLocation` 与相机视点之间插值），否则 `MoveTowardsDefaultTransform`（拉回耳朵 `Audio::GetEarsTransform`）。位移用 `FHazeAcceleratedFloat` 平滑加速，并检测监听器被外部改动后重置。

- **UPlayerCutsceneListenerCapability**（过场监听器）
  - 标签：`Audio::Tags::Listener` + `Audio::Tags::CutsceneListener`。
  - `ShouldActivate` 条件：`Player.bIsParticipatingInCutscene`。
  - 激活时 `BlockCapabilities(Audio::Tags::Listener, this)` 阻塞其它监听器（含默认监听器）；若 `LevelSequenceActor.bIsCutscene` 还额外阻塞 `Audio::Tags::ReflectionTracing`。
  - 数据流：按 `SceneView::GetPlayerViewSizePercentage` 分屏占比，把监听器对齐到当前相机 / 对方玩家相机变换；过场临近结束（`GetTimeRemaining` < 阈值）时调 `Audio::UpdateListenerTransform` 渐变回常规位置。

- **UPlayerDebugCameraListenerCapability**（调试相机监听器）
  - 标签：`n"Debug"` + `n"Audio"`，`DebugCategory = n"Audio"`（注意：**不带** `Audio::Tags::Listener` 标签）。
  - `ShouldActivate` 条件：`Player.IsAnyCapabilityActive(CameraTags::CameraDebugCamera)`。
  - 激活时主动 `BlockCapabilities(Audio::Tags::Listener, this)` 抢占，并缓存/恢复音频与混响组件相对位置；`TickActive` 把监听器对齐到 `Player.GetViewTransform()`。

监听器引用获取方式有两种：`Player.PlayerListener`（默认/调试）或 `UHazeAudioListenerComponent::Get(Player)`（过场）。

## 协同关系

| 方向 | 对象 | 机制 | 说明（基于实证） |
| --- | --- | --- | --- |
| 出 | UHazeAudioListenerComponent (Player.PlayerListener) | 直接调用 | 三类均通过 `Listener.SetWorldTransform(...)` 写入监听器变换，是 Wwise 空间音频的"耳朵"位置/朝向来源 |
| 出 | Audio:: 静态库 | 直接调用 | 调用 `Audio::GetEarsTransform/GetEarsLocation`、`Audio::UpdateListenerTransform`、`Audio::SetPanningBasedOnScreenPercentage`、`Audio::DebugListenerLocations`（AudioStatics.as） |
| 双向 | 其它 Listener Capability | 标签阻塞 | 默认监听器带 `Audio::Tags::Listener`；过场与调试监听器在 `OnActivated` 调 `BlockCapabilities(Audio::Tags::Listener, this)`，靠共享标签把默认监听器挤下台，`OnDeactivated` 解除 |
| 出 | 反射追踪 Capability | 标签阻塞 | 过场监听器在真过场时 `BlockCapabilities(Audio::Tags::ReflectionTracing, this)`，抑制反射追踪（参见 PlayerAudioReflectionTraceCapability） |
| 入 | 关卡特定监听器（复用同标签） | 标签阻塞 | `PlayerSidescrollerListenerCapability` / `PlayerFullscreenListenerCapability`（Gameplay/Movement/Player/Audio）、`WingSuitPlayerListenerCapability`、Prison 系 `Drone*` / `Magnetic*` / `RemoteHacking*`、Summit `PlayerAcidDragonListener` / `PlayerTailDragonListener`、Skyline `Car*` / `GravityBike*` 等均带 `Audio::Tags::Listener`，并 `BlockCapabilities(Audio::Tags::DefaultListener, this)` 抢占默认监听器；标签常量见 `Audio::Tags`（如 `Summit::AcidListener`、`LevelSpecificListener`） |
| 入 | 相机 / 移动系统 | 直接调用 | 读取 `UCameraUserComponent`、`UHazeMovementComponent`、`Player.GetCurrentlyUsedCamera()`、`Player.GetViewTransform()` 决定监听器跟随位置 |
| 入 | 过场系统 | Crumb/直接调用 | 通过 `Player.bIsParticipatingInCutscene` 与 `Player.GetActiveLevelSequenceActor()`（AHazeLevelSequenceActor）驱动过场监听器激活与渐变 |

## 关键文件（逐文件一句话）

- **PlayerDefaultListenerCapability.as**：常驻兜底监听器，按移速在玩家耳朵与相机之间平滑插值，并做分屏声像。
- **PlayerCutsceneListenerCapability.as**：过场期间将监听器对齐相机/对方相机，按分屏占比与剩余时长渐变，并阻塞默认监听器与反射追踪。
- **PlayerDebugCameraListenerCapability.as**：调试相机激活时抢占监听器并对齐调试视点，退出时恢复音频/混响组件位置。

# Core / Audio / Capabilities

> 职责：以 Haze 玩家能力（Capability）驱动运行时音频状态——角色身份、死亡滤波、Game Over、分屏声像，通过 RTPC 与发射器更新落地。共 4 个文件。

## 内部架构（关键类、基类、数据流）

四个能力均继承 `UHazePlayerCapability`，由 Capability 系统按 `ShouldActivate` / `ShouldDeactivate` 自动激活/停用，运行时通过 `Player`（AHazePlayerCharacter）访问角色身份（`IsMio()`）、组件与设置。涉及音频的设置类继承自 `UHazeComposableSettings`。

- **UPlayerDefaultRtpcCapability**：一次性能力（`bHasActivated` 守门）。`OnActivated` 根据 `Player.IsMio()` 取值 -1/1，调用 `Player.PlayerAudioComponent.SetRTPCOnEmitters` 写入 `Rtpc_Shared_Player_IsMio_IsZoe`，向该玩家所有发射器标记角色身份，随即立刻停用。

- **UPlayerGameOverAudioCapability**：监听 `UPlayerHealthComponent.bIsGameOver`，激活/停用时分别将全局 RTPC `Rtpc_Global_IsGameOver` 设为 1/0（`AudioComponent::SetGlobalRTPC`）。

- **UPlayerFilterAudioOnDeathCapability**：最复杂。`CapabilityTags.Add(n"Audio")`，`TickGroup=Audio`。Setup 取双玩家 `UPlayerHealthComponent`、`UPlayerHealthSettings`、可组合的 `UPlayerDefaultAudioDeathSettings`，并通过 `Game::GetSingleton` 取得死亡滤波管理器 `UPlayerAudioFilterDeathManager`。死亡（或双人死亡触发 GameOver，经 `ShouldTriggerGameOver` 判定并写入 `FFilterAudioOnDeathCapabilityActivationParams.bIsGameOver`）时，激活：委托管理器启动滤波（`PlayerFilterActivated` / `StartFilteringForPlayer`）、设全局 RTPC `Rtpc_PlayerDead_Mio/Zoe`、按需播放屏幕空间死亡 SoundDef（`FSoundDefReference`）。复活由 `HealthComp.OnReviveTriggered` 委托回调 `OnPlayerRespawn` → `PlayerRespawned` 停止滤波并清零 RTPC。同文件定义设置类与两个预设 asset（关闭滤波 / 关闭效果与 GameOver 事件）。

- **UPlayerSetPanningCapability**：`TickGroup=Audio`，始终激活。缓存 `Player.PlayerAudioComponent`，每帧比较 `AudioComponent.Panning` 与 `PreviousPanningValue`，变化时调用 `Audio::UpdatePlayerPanning()` 触发所有带声像 RTPC（`Rtpc_SpeakerPanning_LR`）的发射器更新。

## 协同关系

| 方向 | 对象 | 机制 | 说明（基于实证） |
| --- | --- | --- | --- |
| 出 | UHazeAudioPlayerComponent（发射器） | 直接调用 | `PlayerDefaultRtpc` 经 `Player.PlayerAudioComponent.SetRTPCOnEmitters` 向该玩家发射器写角色身份 RTPC |
| 出 | Audio:: 静态库（全局 RTPC） | 直接调用 | GameOver/Death 能力经 `AudioComponent::SetGlobalRTPC` 写 `Rtpc_Global_IsGameOver`、`Rtpc_PlayerDead_Mio/Zoe` |
| 出 | Audio:: 静态库（声像） | 直接调用 | `PlayerSetPanning` 调 `Audio::UpdatePlayerPanning()` 刷新发射器声像；与 Listener 侧 `SetPanningBasedOnScreenPercentage` 同属 `AudioStatics.as` 分屏声像体系 |
| 出 | UPlayerAudioFilterDeathManager | 委托/单例 | 死亡能力经 `Game::GetSingleton` 取管理器并调用 `PlayerFilterActivated`/`StartFilteringForPlayer`/`StopFilteringForPlayer`（见 `Audio\Death\PlayerAudioFilterDeathManager.as`） |
| 入 | UPlayerHealthComponent | 委托 | 死亡能力订阅 `OnReviveTriggered`（`AddUFunction`）回调 `OnPlayerRespawn`；GameOver/Death 能力轮询 `bIsGameOver`/`bIsDead` 决定激活 |
| 入 | UPlayerDefaultAudioDeathSettings | 可组合设置 | 死亡能力经 `GetSettings` 读取可组合设置（`UHazeComposableSettings`），由 asset 预设覆盖开关与淡入淡出时长 |
| 出 | FSoundDefReference（SoundDef） | 直接调用 | 死亡能力经 `SpawnSoundDefAttached`/`RemoveFromActor` 播放与移除屏幕空间死亡音 |

## 关键文件（逐文件一句话）

- `PlayerDefaultRtpcCapability.as`：玩家激活时一次性向其发射器写入 Mio/Zoe 角色身份 RTPC。
- `PlayerGameOverAudioCapability.as`：随玩家 GameOver 状态置位/清零全局 `Rtpc_Global_IsGameOver`。
- `PlayerFilterAudioOnDeathCapability.as`：玩家死亡/双人 GameOver 时委托死亡滤波管理器做音频滤波、设死亡 RTPC 并播放屏幕空间死亡音，复活时还原。
- `PlayerSetPanningCapability.as`：每帧检测玩家声像变化并触发所有带声像 RTPC 的发射器更新（分屏声像）。

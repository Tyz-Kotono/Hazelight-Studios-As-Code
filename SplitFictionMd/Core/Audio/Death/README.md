# Core / Audio / Death

> 职责：处理玩家死亡 / 受伤 / Game Over 时的音频表现——死亡滤波与运行时效果、伤害与死亡 VO 声音定义的引用计数挂载、以及可组合的 VO 开关设置。共 3 个文件。

## 内部架构（关键类、基类、数据流）

本目录由「Manager（全局效果驱动）+ Component（每玩家声音挂载）+ Settings（可组合开关）」三类构成，自身不含 Capability，触发逻辑位于上层 `Audio/Capabilities/PlayerFilterAudioOnDeathCapability.as`。

- **`UPlayerAudioFilterDeathManager : UHazeSingleton`**（PlayerAudioFilterDeathManager.as）
  全局单例，统一管理两名玩家共享的死亡滤波 / 失真 / 比特粉碎 / 卡顿（Stutter）/ Game Over 等运行时音效。
  - 通过 `UHazeAudioRuntimeEffectSystem`（`Game::GetSingleton` 取得）`StartControlled` 启停一系列 `UHazeAudioEffectShareSet` 效果实例；通过 `AudioComponent::SetGlobalRTPC` 推动 Wwise RTPC。
  - 辅助结构 `FPlayerAudioDeathInterpData` 负责单个 RTPC 的插值淡入淡出（From/To/Timer/Duration + Tick）；`FDeathFilterPlayerData` 保存每玩家（按 `EHazePlayer` 索引，`PlayerDatas` 定长 2）的激活态、Fade、Filter 数据。
  - Game Over 时订阅 `UHazeAudioMusicManager.OnMusicBeat`（`OnGameOverMusicBeat`）按节拍递进卡顿时长，并通过 `Audio::StartOrUpdateUserStateControlledBusMixer` 切换 `GameOverBusMixer`。
  - 入口方法 `PlayerFilterActivated` / `StartFilteringForPlayer` / `StopFilteringForPlayer` 全部由 Capability 调用；`Tick` 内驱动所有插值与效果 Alpha，效果全部归零后 `ReleaseAllEffects`。

- **`UPlayerDeathDamageAudioComponent : UActorComponent`**（PlayerDeathDamageAudioComponent.as）
  挂在玩家 Actor 上，按 SoundDef 类做引用计数（`TMap<UClass, FPlayerAudioDeathOrDamageData>`，内部为 `TSet<FInstigator>`），分别管理死亡（DeathSoundDefs）与伤害（DmgSoundDefs）声音。
  - `AttachSoundDef` / `RemoveSoundDef`：首个 instigator 时经 `FSoundDefReference.SpawnSoundDefAttached(Owner)` 真正生成声音；归零时经 `USoundDefContextComponent.RemoveSoundDefByClass` 移除。
  - 广播 `OnNewDeathEffect` / `OnNewDamageEffect`（`event FOnNewDeathEffect/FOnNewDamageEffect`）通知计数变化。
  - 受限访问块 `access:InternalAudio`（仅 `UCharacter_Player_Health_SoundDef`）提供 `AttachInvulnerableSoundDef` / `RemoveInvulnerableSoundDef`，处理无敌期间被规避的伤害音。

- **`UVODamageDeathSettings : UHazeComposableSettings`**（PlayerVODamageDeathSettings.as）
  最简可组合设置，仅含 `bDamageEnabled` / `bDeathEnabled` 两个开关，并提供 `DamageDeathVoDisabled` 资产将两者关闭。被 `VO_DeathEffect_Default_SoundDef` 通过 `GetSettings` 读取，决定死亡/伤害 VO 的 `ShouldActivate` / `ShouldDeactivate`。

数据流概览：玩家死亡 → `UPlayerFilterAudioOnDeathCapability.ShouldActivate` 命中 → `OnActivated` 调用 Manager 启动全局滤波/Game Over 效果；与此并行，`DeathEffect` / `DamageEffect` 在生效/移除时调用 Component 的 Attach/Remove 挂载具体 VO 与效果 SoundDef，VO SoundDef 再读 `UVODamageDeathSettings` 决定是否发声。

## 协同关系

| 方向 | 对象 | 机制 | 说明（基于实证） |
| --- | --- | --- | --- |
| 入 | `UPlayerFilterAudioOnDeathCapability` | 直接调用 Type::Get | Capability `Setup` 中 `Game::GetSingleton(ManagerClass)` 取得 Manager，`OnActivated`/`PlayerRespawned` 调用 `PlayerFilterActivated`、`StartFilteringForPlayer`、`StopFilteringForPlayer` 驱动死亡滤波。 |
| 入 | `UPlayerFilterAudioOnDeathCapability` | 标签阻塞 CapabilityTags | Capability `default CapabilityTags.Add(n"Audio")` 且 `TickGroup = EHazeTickGroup::Audio`，是 Death 效果的唯一触发入口（本目录无自带 Capability）。 |
| 入 | `UPlayerHealthComponent` | 委托 event/Broadcast | Capability 订阅 `HealthComp.OnReviveTriggered`（`OnPlayerRespawn`），复活后通知 Manager 停止滤波。 |
| 入 | `DeathEffect` / `DamageEffect`（PlayerHealth） | 直接调用 Type::Get | `UPlayerDeathDamageAudioComponent::GetOrCreate(Player)` 后在 `OnApplied`/`OnRemoved` 调用 `AttachSoundDef`/`RemoveSoundDef` 引用计数挂载死亡与伤害 SoundDef。 |
| 入 | `UCharacter_Player_Health_SoundDef` | 直接调用 Type::Get | 经 `access:InternalAudio` 调用 Component 的 `AttachInvulnerableSoundDef`/`RemoveInvulnerableSoundDef` 处理无敌规避伤害音。 |
| 出 | 监听 Component 的对象 | 委托 event/Broadcast | Component 在计数变化时 `OnNewDeathEffect`/`OnNewDamageEffect` 广播 `(Effect, Count)`。 |
| 出 | `UHazeAudioMusicManager` | 委托 event/Broadcast | Manager Game Over 时 `MusicManager.OnMusicBeat.AddUFunction(this, n"OnGameOverMusicBeat")` 按节拍推进卡顿，复活/超时后 `UnbindObject`。 |
| 出 | Wwise（Ak 桥接） | 直接调用 Type::Get | Manager 通过 `AudioComponent::SetGlobalRTPC` / `PostGlobalEvent`、`EffectSystem.StartControlled`、`Audio::StartOrUpdateUserStateControlledBusMixer` 推动底层效果与混音。 |
| 出 | `USoundDefContextComponent` / `FSoundDefReference` | 直接调用 Type::Get | Component 经 `SpawnSoundDefAttached` 生成、`RemoveSoundDefByClass` 移除运行时 SoundDef。 |
| 入 | `VO_DeathEffect_Default_SoundDef` | 可组合设置 ApplySettings | VO SoundDef `ParentSetup` 中 `UVODamageDeathSettings::GetSettings(HazeOwner)`，依 `bDeathEnabled` 决定 VO 是否激活；`DamageDeathVoDisabled` 资产可整体关闭。 |
| 入 | Capability | 可组合设置 ApplySettings | Capability 读 `UPlayerDefaultAudioDeathSettings::GetSettings`（定义同在 Capability 文件），其 `bCanActivate`/`bDisableEffectsAndGameOverEvents` 及各淡入淡出时长被传入 Manager。 |

## 关键文件（逐文件一句话）

- **PlayerAudioFilterDeathManager.as**：全局单例 `UPlayerAudioFilterDeathManager`，按玩家与节拍插值驱动死亡滤波、卡顿、失真及 Game Over 运行时音效与 RTPC。
- **PlayerDeathDamageAudioComponent.as**：玩家身上的 `UPlayerDeathDamageAudioComponent`，对死亡/伤害 SoundDef 做引用计数挂载/移除并广播计数变化事件。
- **PlayerVODamageDeathSettings.as**：可组合设置 `UVODamageDeathSettings`（`bDamageEnabled`/`bDeathEnabled`）及关闭资产，供死亡/伤害 VO SoundDef 读取以开关发声。

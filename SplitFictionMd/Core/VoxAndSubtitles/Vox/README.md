# Core / VoxAndSubtitles / Vox

> 职责：单例驱动、基于车道（Lane）的语音（VO / Bark）排程与播放系统——校验、优先级仲裁、排队、暂停/恢复、字幕与面部动画驱动。共 18 个文件（含子目录 `RuntimeAssets/`、`Triggers/`）。

## 内部架构（关键类/基类/数据结构/数据流）

整个系统是两级单例 + 一棵运行期实例树：

```
设计师 / 触发组件
   │ HazePlayVox(VoxAsset, Actors)
   ▼
UHazeVoxController（单例：守门人）          ← 校验 / 角色模板匹配 / play-once / 冷却 / CanTrigger
   │ PlayLocal(RuntimeAsset, VoiceLineIndex)
   ▼
UHazeVoxRunner（单例：播放器）              ← 6 条车道，低序车道阻塞高序车道
   │ Lane.Play(...)
   ▼
UVoxLane（车道：槽位 + 队列 + 优先级仲裁）
   │
   ▼
UVoxRuntimeAsset（一次播放的 live 实例：状态机 Stopped/Queued/Playing/TailingOut）
   │
   ▼
UVoxVoiceLine（单句状态机：PreDelay→Playing→PostDelay→TailingOut；Wwise 发射器 / 面部动画 / RTPC / 字幕）
```

### 守门人 `UHazeVoxController`（HazeVoxController.as）
- 基类 `UHazeVoxControllerBaseSingleton`，经 `Game::GetSingleton` 取得。
- 持有每资产持久态映射 `TMap<FName, UVoxSharedRuntimeAsset>`、暂停 Actor 列表 `PausedActors`、持久 play-once 集合 `PersistentPlayedOnceVoxAssets`、网络随机种子盐 `RandSeedSalt`。
- `Play()` 入口：若调用方 `CallingVOSoundDef.HasControl()` 则走 `CrumbInternalPlay`（联网 Crumb），否则 `InternalPlayLocal` 本地播放。
- `InternalPlayLocal()` 执行校验链：管理器是否激活 → VoxAsset 有效 → 本地化 Bnk 支持 → 语音行非空且有音频事件 → `MatchActorsAndCheckCanTrigger` 得到 `EVoxControllerCanTriggerResult`（`CanTrigger` / `BlockedButCanQueue` / `FullyBlocked`）。
- `CanAssetTrigger()` 判定：InGame、外部暂停、SharedAsset.CanTrigger（play-once/冷却）、所需角色模板齐全、目标 Actor 是否被暂停、是否被更低序车道阻塞（含 `EHazeVoxLaneInterruptType` 打断模式）。
- 暂停体系：`PauseActor/ResumeActor` 以 `FInstigator` 计数，玩家死亡（`UPlayerHealthComponent.bIsDead`）自动暂停。
- 游戏暂停经 `Game::IsPausedForAnyReason` 检测，转为 `SetExternalPause` 硬暂停 Runner。

### 播放器 `UHazeVoxRunner`（HazeVoxRunner.as）
- 基类 `UHazeVoxRunnerBaseSingleton`。`InitLanes()` 创建 6 条固定车道（序号即优先级）：

| 序号 | 车道 | 槽位数 | 排序类型 |
|---|---|---|---|
| 0 | First | 1 | Priority |
| 1 | Second | 1 | Priority |
| 2 | Third | 1 | Priority |
| 3 | Generics | 3 | Priority |
| 4 | EnemyCombat | 10 | PriorityAge |
| 5 | Efforts | 10 | PriorityAge |

- **阻塞规则**：`FindBlockingLane` / `StopBlockedLanes`——低序车道（如 First）正在播放某 Actor 时，会阻塞 / 停止高序车道（如 Generics、Efforts）上同一 Actor 的语音。
- 管理共享 RTPC（对话 ducking、屏蔽对侧玩家 Efforts），以 `FInstigator` 计数引用。
- 每帧 Tick：推进每条车道、把空闲车道的队首取出播放、重算阻塞关系。

### 车道 `UVoxLane`（HazeVoxLane.as）
- 持有 `RuntimeSlots`（正在播放）、`TailingOutAssets`（尾音衰减中）、`QueuedAssets`（排队）。
- `TestPriority` / `FindInsertIndex` 按 `EVoxLaneSlotSortType`（Priority / PriorityAge / Age）做优先级 + 年龄仲裁；槽位满时挤掉低优先级。
- `QueueAsset` 按优先级排序插入队列；`GetNextInQueue` 出队时回头向 Controller 复查 CanTrigger。

### 运行期实例 `UVoxRuntimeAsset`（RuntimeAssets/HazeVoxRuntimeAsset.as）
- 一次播放的 live 实例，状态机 `EVoxRuntimeState`。
- 持有 `ActiveVoiceLines`（一组 `UVoxVoiceLine`）；对话（Dialogue）类型逐行串联，`GetNextDialogueIndex` 驱动下一句。
- 命名空间 `VoxAssetHelpers::FindVoiceLineActor` 把角色模板映射到具体 Actor。
- 对话且含双玩家 / 玩家+NPC 时启用 Conversation RTPC ducking。

### 单句 `UVoxVoiceLine`（HazeVoxVoiceLine.as）
- 单句状态机 `ERuntimeVoiceLineState`：PreDelay → Playing → PostDelay → TailingOut。
- 经 `UHazeAudioEmitter.PostEvent` 发射 Wwise VO 事件，驱动面部动画（`PlayFaceAnimation`）、RTPC、字幕（持有 `UVoxSubtitles`）。
- 前后延迟（PreDelay/PostDelay/OverlapOffset/QueuePreDelay）、提前截断（cutoffEarly）、暂停 seek（`PauseSeekMS`）、硬暂停/恢复。

### 每资产持久态 `UVoxSharedRuntimeAsset`（HazeVoxSharedRuntimeAsset.as）
- 跨多次播放保留：行选择顺序（Ordered/Shuffle/Random 加权）、`bPlayedOnce`/`bAllPlayedOnce`、`Cooldown`。
- 用 `FRandomStream`（种子 = 资产名哈希 ^ 网络种子盐）保证双机同步。

### 限流（挂在 Mio 上的玩家组件）
- `UGlobalVoTimersPlayerComponent`（GlobalVoTimers.as）：命名计时器节流（如 Combat/Reload）。
- `UGlobalVoRateLimitPlayerComponent`（GloalVoRateLimits.as）：时间窗口内事件计数限流。

### 触发组件（Triggers/）
设计师在关卡放置组件，外部（触发体/蓝图）调用 `OnStarted`/`StartTrigger`，组件内部按条件调用 `HazePlayVox` / `HazePlayVoxSoftActors`：
- `UVoxTriggerComponent`：基础 Bark，支持 Mio/Zoe 专属资产、TimeInTrigger、最大触发次数、延迟。
- `UVoxAdvancedPlayerTriggerComponent`：增加距离检查（玩家间 / 到目标 Actor）选择主/备资产、重复模式。
- `UVoxDuoPlayerTriggerComponent`：双人掷骰决定 Mio/Zoe 谁说（AlwaysBoth / IndividualRoll / RollBetween）。

### 死亡暂停
- `UPlayerPauseVoxOnDeathCapability` + `UPlayerPauseVoxOnDeathComponent`：玩家死亡延迟 0.2s 后调 `Controller.PauseActor`，复活时 `ResumeActor` 并广播死亡/复活委托。

## 协同关系（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
|---|---|---|---|
| 调出 | `UHazeVoxController::Get()` | 1 直接调用 Type::Get | `HazePlayVox` / 各触发组件 / 死亡暂停 capability 统一入口（HazeVoxStatics.as、Triggers/\*、PlayerPauseVoxOnDeathCapability.as） |
| 调出 | `UHazeVoxRunner::Get()` | 1 直接调用 | Controller 把 RuntimeAsset 交给 Runner.PlayLocal；VoiceLine/RuntimeAsset 调用 Runner 的 RTPC 接口 |
| 调出 | `UVoxSubtitles` → `Subtitle::ShowSubtitle` | 1 直接调用（C++ 桥） | VoiceLine 经 HazeVoxSubtitles.as 把字幕推给 Subtitles 模块的 `USubtitleManagerComponent` |
| 调出 | `AHazeActor.GetVoxCharacterTemplate()` | 5 可组合设置 + mixin | Controller 匹配 Actor 上的 `UHazeVoxCharacterTemplateComponent`（PlayerCharacter.as 持有该组件） |
| 调出 | Crumb 网络函数 | 2 Crumb | `CrumbInternalPlay` / 各触发组件 `CrumbPlayVoxAsset` 经 `CrumbFunction` 联机同步播放 |
| 调出 | Wwise `UHazeAudioEmitter` / `AudioComponent::SetGlobalRTPC` | 1 直接调用 | VoiceLine 发射 VO 事件，Runner 设全局对话 ducking / Efforts 静音 RTPC |
| 调出 | `Game::GetSingleton` / `Game::GetMio/GetZoe/GetPlayer` | 1 直接调用 | 单例获取与双人玩家解析 |
| 调出 | `UPlayerHealthComponent` | 1 直接调用 | Controller/Capability 据 `bIsDead`/`bIsGameOver` 暂停-恢复 Actor |
| 调出 | `EHazeVoxLaneName` → 字幕优先级 | 5 可组合设置 | `LaneToSubtitlePriority`（HazeVoxSubtitles.as）按车道映射字幕优先级 |
| 调入 | 标签 `Vox` capability | 3 标签 | `UPlayerPauseVoxOnDeathCapability` 以 `CapabilityTags.Add(n"Vox")` 注册 |
| 调入 | `OnVoxAssetTriggered` / `OnPlayerTriggersEnabled/Disabled` | 4 委托 event/Broadcast | 触发组件与 Controller 向外广播事件 |
| 调入 | `BindVoxOnPlayerRespawn/Death` | 4 委托 | 外部绑定玩家复活/死亡回调 |

## 关键文件（逐文件一句话）

| 文件 | 说明 |
|---|---|
| `HazeVoxController.as` | 单例守门人：校验、角色匹配、play-once/冷却/CanTrigger 判定、暂停管理、Crumb 联网播放。 |
| `HazeVoxRunner.as` | 单例播放器：初始化 6 条车道、车道间阻塞、推进队列、管理共享 RTPC。 |
| `HazeVoxLane.as` | 车道：槽位/排队/尾音容器，按 Priority/PriorityAge/Age 做优先级仲裁与插入。 |
| `RuntimeAssets/HazeVoxRuntimeAsset.as` | 一次播放的 live 实例状态机，管理一组语音行并串联对话。 |
| `HazeVoxVoiceLine.as` | 单句状态机：Wwise 发射、面部动画、RTPC、前后延迟、暂停 seek、字幕。 |
| `HazeVoxSharedRuntimeAsset.as` | 每资产持久态：行选择（顺序/洗牌/加权随机）、play-once、冷却、同步随机流。 |
| `HazeVoxSubtitles.as` | `UVoxSubtitles` 桥接：清洗 [CC] 文本、按车道定优先级、把字幕推给字幕系统。 |
| `HazeVoxStatics.as` | 全局函数门面：`HazePlayVox`、`HazeVoxStop/Start`、`PauseActor` 等。 |
| `HazeVoxHelpers.as` | `VoxHelpers`：InGame 判定、随机种子洗牌等工具。 |
| `HazeVoxCharacterTemplateComponent.as` | 挂在 Actor 上保存 `UHazeVoxCharacterTemplate`，提供 `GetVoxCharacterTemplate` mixin。 |
| `HazeVoxDebug.as` | 调试 CVar、调试配置、时间线/Telemetry 数据结构与 `BuildLaneDebugName`。 |
| `GlobalVoTimers.as` | `UGlobalVoTimersPlayerComponent`：命名计时器节流（Combat/Reload 等）。 |
| `GloalVoRateLimits.as` | `UGlobalVoRateLimitPlayerComponent`：时间窗口事件计数限流。 |
| `PlayerPauseVoxOnDeathComponent.as` | 存玩家死亡/复活委托列表 + 绑定/解绑全局函数。 |
| `PlayerPauseVoxOnDeathCapability.as` | `Vox` 标签能力：死亡延迟暂停语音、复活恢复并触发委托。 |
| `Triggers/VoxTriggerComponent.as` | 基础 Bark 触发组件（Mio/Zoe 专属资产、延迟、次数）。 |
| `Triggers/VoxAdvancedPlayerTriggerComponent.as` | 进阶触发：距离检查选主/备资产、重复模式。 |
| `Triggers/VoxDuoPlayerTriggerComponent.as` | 双人触发：掷骰决定 Mio/Zoe 谁说。 |

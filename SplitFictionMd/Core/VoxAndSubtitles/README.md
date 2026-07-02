# Core / VoxAndSubtitles —— 语音与文字可达性功能域总览

> 本功能域负责游戏的语音（VO / Bark）排程播放、屏幕字幕显示，以及无障碍文字转语音。包含 3 个并列子模块：**Vox**（18 文件，核心）、**Subtitles**（1 文件）、**Accessibility**（1 文件）。游戏为双人合作，两名玩家分别为 **Mio** 与 **Zoe**。

## 三模块关系

```
                 ┌─────────────────────────────────────────────┐
                 │                    Vox                       │
                 │  单例驱动、基于车道的 VO 排程与播放（核心）  │
                 └───────────────┬──────────────┬──────────────┘
                                 │              │
              VoiceLine 播放时   │              │  （无代码耦合，
              推送字幕            │              │   仅同域并列）
                                 ▼              ▼
                 ┌────────────────────┐   ┌────────────────────────┐
                 │     Subtitles      │   │     Accessibility      │
                 │ 每玩家双槽字幕显示 │   │ Haze.TestNarration TTS │
                 └────────────────────┘   └────────────────────────┘
```

- **Vox → Subtitles**：唯一的实质代码协同。Vox 的 `UVoxVoiceLine` 在播放每句台词时，经 `HazeVoxSubtitles.as` 的桥接类 `UVoxSubtitles` 调用 C++ 命名空间 `Subtitle::ShowSubtitle / ShowSubtitlesFromAsset / ClearSubtitlesBy*`，最终路由到玩家身上的 `USubtitleManagerComponent`。车道（Lane）经 `LaneToSubtitlePriority` 映射为字幕优先级（First=High、Second=Medium、其余=Low）。
- **Accessibility**：与前两者无代码调用关系，仅是同属「语音/文字可达性」主题的独立 TTS 测试命令，故并列归档。

## Vox 排程流程图（Controller → Runner → Lane → VoiceLine）

```
设计师 HazePlayVox(VoxAsset, Actors)  ／  触发组件（VoxTrigger / Advanced / Duo）
        │
        ▼
┌──────────────────────── UHazeVoxController（单例守门人）────────────────────────┐
│  · 校验 VoxAsset（有效 / 非空 / 有音频事件 / 本地化支持）                         │
│  · 把 Actors 匹配到 UHazeVoxCharacterTemplate（角色模板齐全？）                   │
│  · CanTrigger 判定 →  CanTrigger / BlockedButCanQueue / FullyBlocked            │
│       检查项：play-once、冷却、外部暂停、Actor 被暂停、是否被低序车道阻塞        │
│  · 经 Crumb 联机同步（CallingVOSoundDef.HasControl 时）                          │
└───────────────┬───────────────────────────────────┬────────────────────────────┘
   CanTrigger    │                     BlockedButCanQueue（且 bQueueOnPlay）
                 ▼                                     ▼
        Runner.PlayLocal(RuntimeAsset)         Lane.QueueAsset(排队，限时存活)
                 │                                     │（车道空闲时回头复查 CanTrigger 再出队）
                 ▼                                     │
┌──────────────────── UHazeVoxRunner（单例播放器）────┴───────────────────────────┐
│  6 条车道（序号即优先级，低序阻塞高序）：                                          │
│  First(0) > Second(1) > Third(2) > Generics(3) > EnemyCombat(4) > Efforts(5)     │
│  · StopBlockedLanes：低序车道占用某 Actor 时，停掉高序车道上同 Actor 的语音       │
│  · 管理共享 RTPC（对话 ducking、屏蔽对侧玩家 Efforts）                            │
└───────────────────────────────────┬────────────────────────────────────────────┘
                                     ▼
┌──────────────────────────── UVoxLane（车道）────────────────────────────────────┐
│  RuntimeSlots（播放中）／ TailingOutAssets（尾音）／ QueuedAssets（队列）         │
│  · TestPriority / FindInsertIndex：按 Priority / PriorityAge / Age 仲裁、挤占     │
└───────────────────────────────────┬────────────────────────────────────────────┘
                                     ▼
┌────────────────── UVoxRuntimeAsset（一次播放的 live 实例）───────────────────────┐
│  状态机：Stopped → Queued → Playing → TailingOut                                 │
│  · 对话类型逐行串联（GetNextDialogueIndex）                                       │
│  · 行选择 / play-once / 冷却 由 UVoxSharedRuntimeAsset（每资产持久态）提供        │
└───────────────────────────────────┬────────────────────────────────────────────┘
                                     ▼
┌────────────────────────── UVoxVoiceLine（单句状态机）───────────────────────────┐
│  PreDelay → Playing → PostDelay → TailingOut                                     │
│  · Wwise 发射器 PostEvent（VO）   · 面部动画 PlayFaceAnimation                   │
│  · RTPC（ducking / 静音 Efforts） · 前后延迟 / 重叠偏移 / 暂停 seek               │
│  · 字幕 → UVoxSubtitles → Subtitles 模块                                          │
└─────────────────────────────────────────────────────────────────────────────────┘
```

辅助支撑：
- **限流**：`UGlobalVoTimersPlayerComponent`（命名计时器节流）、`UGlobalVoRateLimitPlayerComponent`（时间窗口计数限流），均挂在 Mio 上、由设计师在触发逻辑前查询。
- **死亡暂停**：`UPlayerPauseVoxOnDeathCapability`（`Vox` 标签能力）+ `UPlayerPauseVoxOnDeathComponent`：玩家死亡暂停其语音、复活恢复并广播委托。

## 五种协作机制小结

| 机制 | 在本功能域的体现 |
|---|---|
| 1 直接调用 Type::Get | `UHazeVoxController::Get()` / `UHazeVoxRunner::Get()` 单例；字幕经 `Subtitle::` C++ 桥 |
| 2 Crumb（联网同步） | `CrumbInternalPlay` 及各触发组件 `CrumbPlayVoxAsset` 经 `CrumbFunction` 同步双机播放 |
| 3 标签阻塞 | 车道序号即优先级，低序车道阻塞高序车道；`Vox` 能力标签 |
| 4 委托 event/Broadcast | `OnVoxAssetTriggered`、`OnPlayerTriggersEnabled/Disabled`、死亡/复活委托 |
| 5 可组合设置 | VoxAsset 上的车道/优先级/play-once/冷却/延迟等数据驱动；字幕 CVar、限流计时器名 |

## 子模块文档

- [Vox/README.md](./Vox/README.md) —— 语音排程与播放核心（18 文件）。
- [Subtitles/README.md](./Subtitles/README.md) —— 每玩家字幕显示（1 文件）。
- [Accessibility/README.md](./Accessibility/README.md) —— TTS 朗读测试命令（1 文件）。

# Core / Audio / Zones

> 职责：空间化音频环境区域系统——环境声、混响、遮挡、传送门、水下，由 `UAudioZoneManager` 统一驱动每帧的玩家/监听器相关度计算并写入 Wwise RTPC。共 8 个文件。

## 内部架构（关键类、基类、数据流）

### 中心调度：`UAudioZoneManager`（继承自 C++ 基类 `UHazeAudioZoneManager`）
每帧 `Tick` 中遍历由 C++ 基类维护的 `ActiveZones`（区域进入/离开监听器范围时由 C++ 自动注册到此列表），对每个区域调用 `Process(Zone, ...)`：

- 读取 `Zone.ListenerOverlaps`（每个监听器与区域的重叠记录，含 `ObjectRelevance` 重叠相关度），结合 `Zone.GetZoneRelevance()` 计算出该区域的最终相关度 `FinalRelevance`。
- 通过 `Zone.SetZoneRTPC(FinalRelevance, PowerOf)` 把相关度（经曲线幂运算）下发给具体区域子类，区域再把它平滑（`MoveZoneRtpcToTarget`）后写入发射器的 Wwise RTPC，从而实时改变音量/淡入淡出。
- 混响走另一条线：`ProcessingQueue` 中的 `UHazeAudioReverbComponent`（每个发射器/监听器身上的混响组件）按其 `ZoneOverlaps`（携带 `AttenuationDistance`、`ZonePriority`）逐区域计算 aux send 值，优先级越高的区域会压低低优先级区域的混响发送量（`GetLowerPrioReverbSendValue`）。
- VO 特例：环境/混响区可携带 `PlayerVoGameAuxSendVolume` 等参数，Manager 累加每个玩家的 VO aux send 后写到玩家 VO 发射器（`Audio::GetPlayerVoEmitter`）。

### 区域家族：`AHazeAudioZone`（C++ 基类）的 AngelScript 子类
所有区域共享基类的 `ZoneType`、`Priority`、`ListenerOverlaps`、`ZoneRTPCValue/ZoneFadeTargetValue`、`MoveZoneRtpcToTarget`、Brush 碰撞体（碰撞档案 `AudioZone`）等。`AudioZone::OnBeginPlay`（ZoneStatics）统一根据 `RtpcCurve` 设置曲线幂 `ZoneRTPCCurvePower`。

- `AAmbientZone`：环境声主类。从对象池 `Audio::GetPooledAudioComponent` 取环境发射器播放 `ZoneAsset.QuadEvent`，按相关度驱动 `AmbZoneFade` RTPC；支持双人左右声像 `AmbZonePanning`（由 Manager 按 Mio/Zoe 在区域内的相关度计算并调用 `UpdatePanning`）；还驱动 `RandomSpots`（随机点缀音效，FireForget 投放在监听器周围）。
- `AWaterZone`：继承 `AAmbientZone`，仅 `ZoneType=Water`，复用环境声逻辑表达水下音效；Manager 通过全局 RTPC `Rtpc_Shared_Camera_InWater` 判断双摄像机是否都在水中以静音其他环境区。
- `AReverbZone`：混响区，提供 `GetSendLevel`/`GetReverbBus`（可被实例覆盖或取自 `ZoneAsset`）供 Manager 的混响发送计算使用。
- `AOcclusionZone`：遮挡区，固定 `Relevance=1`，依赖基类重叠/淡变把遮挡量平滑到目标值。
- `APortalZone`：传送门区，连接相邻区域。`ZoneType=Portal` 被 Manager 的相关度处理跳过，转而充当连接通道：通过几何计算（`CalculateBoundsForZones`/`GetNearestPlane`）求出连接面，用 `ZoneRTPCValue` 作为缩放系数影响相邻环境区的相关度曲线幂（`SetZoneRTPC` 的 `PowerOfOverride`）。
- `ASimplifiedOcclusionZone`：独立的轻量遮挡，**不**继承 `AHazeAudioZone`（继承自 `AVolume`），不进 Manager 流程。自身 Tick 中按玩家监听器到包围盒的距离/`Attenuation` 算遮挡值，直接对 `LinkedActors` 上收集到的 SpotSound/SoundDef 音频组件设置 `Rtpc_Shared_Occlusion_Fade`。

### 静态库：`ZoneStatics.as`（`namespace AudioZone`）
提供 `OnBeginPlay`（曲线初始化）、`GetVoGameAuxVolumeValues`（向上转型取环境/混响区的 VO 参数）、重叠取玩家/VO 发射器工具，以及全套调试可视化 `DrawZone`/`FColorCoding`。

### 与发射器/SoundDef 的遮挡联动
发射器侧通过 `AudioComponent.GetZoneOcclusion(bFollowRelevance, LinkedZone, bAutoSetRtpc)`（C++ 桥接）把自身与某个区域绑定，让区域相关度直接调制该发射器的遮挡 RTPC。`USpotSoundComponent.SetupEmitter`/`UpdateZoneOcclusionTracking` 在 `bLinkToZone` 时调用它；众多 SoundDef（如 `Spot_Tracking_SoundDef`、隧道列车、爆破机等）也在自身 emitter 上调用 `GetZoneOcclusion` 实现进出隧道/区域的遮挡。

## 协同关系

| 方向 | 对象 | 机制 | 说明（基于实证） |
| --- | --- | --- | --- |
| 出 | `UHazeAudioEmitter` / 环境发射器 | 委托 ApplySettings（SetRTPC/PostEvent） | `AAmbientZone` 从池取发射器后 `PostEvent(ZoneAsset.QuadEvent)` 并按相关度写 `AmbZoneFade`、`AmbZonePanning` RTPC（AmbientZone.as:143,176,230,396） |
| 出 | 玩家 VO 发射器 | 直接调用 Type::Get + ApplySettings | Manager 累加 VO aux send 后 `Audio::GetPlayerVoEmitter(Player).SetNodeProperty(...)`（AudioZoneManager.as:84-92） |
| 入/出 | 各 `AHazeAudioZone` 子类 | 委托（基类回调 SetZoneRTPC/Process） | Manager 遍历 `ActiveZones` 调 `Process`→`Zone.SetZoneRTPC(FinalRelevance, PowerOf)`，子类 Tick 平滑后下发（AudioZoneManager.as:74-181, OcclusionZone.as:29, ReverbZone.as:69） |
| 入 | `UHazeAudioReverbComponent` | 可组合设置 ApplySettings | Manager 按 `ZoneOverlaps` 逐区域算 aux send 并 `UpdateObjectAuxSendValue`/`SetAuxSendValue`（AudioZoneManager.as:248,294） |
| 出 | 相邻 `AAmbientZone` | 可组合设置（曲线幂覆盖） | `APortalZone` 用 `ZoneRTPCValue` 作缩放，经 `ListenerOverlap.RTPCCurvePowerOverride` 改邻区相关度曲线（PortalZone.as, AudioZoneManager.as:152-156） |
| 出 | SpotSound / SoundDef 发射器 | 直接调用 Type::Get（GetZoneOcclusion 桥接） | 发射器 `AudioComponent.GetZoneOcclusion(bFollowRelevance, LinkedZone, bAutoSetRtpc)` 让区域相关度调制其遮挡 RTPC（SpotSoundComponent.as:322,360；Spot_Tracking_SoundDef.as:87 等多处 SoundDef） |
| 出 | `LinkedActors` 上的 SpotSound/SoundDef 音频组件 | 直接调用 ApplySettings | `ASimplifiedOcclusionZone` 按距离算遮挡后 `SetRTPCOnEmitters(Rtpc_Shared_Occlusion_Fade)`（SimplifiedOcclusionZone.as:99-110，组件来自 `GatherAudioComponents`） |
| 入 | 全局 RTPC `Rtpc_Shared_Camera_InWater` | Crumb（全局缓存 RTPC 读取） | Manager 读 `GetCachedGlobalRTPC` 判断双摄像机入水以静音其他环境区（AudioZoneManager.as:123-128） |
| 出 | 发射器对象池 | 直接调用 Type::Get | `AAmbientZone` 经 `Audio::GetPooledAudioComponent` 借/还环境发射器（AmbientZone.as:127,203,419） |

## 关键文件（逐文件一句话）

- `AudioZoneManager.as`：`UAudioZoneManager` 中心调度——每帧遍历活跃区域算相关度并下发 RTPC、处理混响发送与玩家 VO aux send。
- `AmbientZone.as`：`AAmbientZone` 环境声主类——池化发射器播放环境四声道、双人声像、随机点缀音效，按相关度驱动淡入淡出 RTPC。
- `WaterZone.as`：`AWaterZone` 继承环境区表达水下音效，配合全局入水 RTPC 控制其他环境区静音。
- `ReverbZone.as`：`AReverbZone` 混响区，提供可覆盖的 SendLevel 与 ReverbBus 供 Manager 计算 aux send。
- `OcclusionZone.as`：`AOcclusionZone` 遮挡区，固定满相关度，依赖基类淡变把遮挡量平滑到目标。
- `PortalZone.as`：`APortalZone` 传送门区，几何计算连接面、以缩放系数覆盖相邻环境区的相关度曲线幂。
- `SimplifiedOcclusionZone.as`：`ASimplifiedOcclusionZone`（继承 `AVolume`，不入 Manager 流程）按距离对 LinkedActors 的 SpotSound/SoundDef 直接设置遮挡 RTPC。
- `ZoneStatics.as`：`namespace AudioZone` 静态库——曲线初始化、VO 参数取值、重叠工具与区域调试可视化。

# Core / Audio / Volumes

> 职责：定义音频触发体积（AVolume 子类），在玩家或指定 Actor 进出体积时播放/停止事件、SoundDef 或触发植被检测逻辑。共 7 个文件。

## 内部架构（关键类、基类、数据流）

体积分两条继承线，均派生自引擎 `AVolume`：

**1. 玩家音频体积线（PlayerAudioVolumeBase 家族）**

- `APlayerAudioVolumeBase`（`AVolume`，`Abstract`）：玩家专用触发体积基类。碰撞配置为 `TriggerOnlyPlayer`。核心数据流：引擎回调 `ActorBeginOverlap/ActorEndOverlap`（以及私有 `OnPlayerEnter/OnPlayerExit`）先做玩家过滤（`IsEnabledForPlayer`，按 `bTriggerForMio/bTriggerForZoe` 区分双角色 Mio/Zoe）、冷却检查（`IsCooldownReady`/`CooldownTime`）、`bTriggerOnce` 一次性禁用，再调用可被子类 override 的 `PlayOnEnter(Player)` / `PlayOnExit(Player)`。发射器通过 `SetupEmitter()`（可 override）懒创建：从 `Audio::GetPooledAudioComponent` 取池化组件并 `GetEmitter(this)` 得到 `VolumeEmitter`，随后 `PostSetupEmitter` 应用 RTPC、节点属性、声道 Panning（`SetPlayerPanning`/`SetSpatialPanning`）。资产以 `FHazeSpotSoundAssetData` 形式解析为 `Event`（`UHazeAudioEvent`）或 `SoundDefRef`（`FSoundDefReference`）。
  - `APlayerAudioTriggerVolume`：override `PlayOnEnter/PlayOnExit`，支持多个进入/退出资产（`OnEnterAssets/OnExitAssets`），按 `Position`（OnPlayer 或 VolumeCenter）选择在玩家 `PlayerAudioComponent` 或 `VolumeEmitter` 上 PostEvent；可 `bStopEventsOnExit` 跟踪 PlayingID 并淡出停止；SoundDef 走 `SpawnSoundDefOneshot`。
  - `APassbyishTriggerVolume`：override `SetupEmitter`（把发射器放到自定义 `PassbyTransform` 位置）与 `PlayOnEnter`，以 `EHazeAudioEventPostType::Passby` 投递事件，模拟“掠过音”。编辑期由 `UPassbyishTriggerEditorComponent` + Visualizer 提供声源位置拖拽与可视化。

**2. 独立体积**

- `AActorSpotAudioTriggerVolume`（`AVolume`）：面向指定 `ActorClass`（非玩家）的定点音体积。在 `ActorBeginOverlap/ActorEndOverlap` 中为重叠 Actor `GetOrCreate` 一个 `USpotSoundComponent`，配置 `FSpotSoundEmitterSettings`、可选挂接 `AAmbientZone`、按 `EmitterFollow` 决定发射器跟随谁，然后 `Start()` 播放进入事件/SoundDef，退出时停止或播放退出事件。
- `AFoliageDetectionVolume`（`AVolume`）：植被检测体积，碰撞 `TriggerOnlyPlayer`。重叠玩家后开 Tick，用预烘焙的 `FStaticSparseSphereGrid`（编辑器 `BuildFoliageVolume` 从静态网格/Instanced Foliage 生成）查询玩家胶囊体所在植被类型（Grass/Bush/Plant）。类型变化时调 `OnPlayerOverlapFoliageChange`，封装 `FFoliageDetectionData`（含 `MaterialOverridesPerType` 物理材质覆盖）。

`PlayerVOTriggerVolume.as` 当前仅含一个空命名空间占位函数 `GetSoundDefVOTriggerDummy`（VO 配音触发体积尚未实现）。

## 协同关系

| 方向 | 对象 | 机制 | 说明（基于实证） |
| --- | --- | --- | --- |
| 出 | UFoliageDetectionEventHandler | 委托 event/Broadcast | `FoliageDetectionVolume.as:75` 调 `UFoliageDetectionEventHandler::Trigger_FoliageOverlapEvent(Player, Data)`，触发其 `BlueprintEvent FoliageOverlapEvent`（定义于 `Core/Audio/Foliage/FoliageDetectionEffectEventHandler.as:12-17`，基类 `UHazeEffectEventHandler`）。 |
| 入 | Foliage SoundDef（如 Foliage_SoundDef / Player_Movement_Basic_SoundDef） | 直接调用 Type::Get | SoundDef 以 `EventClassAndFunctionNames.Add(n"FoliageOverlapEvent", UFoliageDetectionEventHandler)` 注册回调（`Audio/SoundDefinitions/.../Foliage_SoundDef.as:43`、`Player_Movement_Basic_SoundDef.as:296`），消费上述植被事件播放脚步/植被音。 |
| 出 | Audio:: 静态库 / UHazeAudioEmitter | 直接调用 Type::Get | 各体积 `SetupEmitter` 调 `Audio::GetPooledAudioComponent` 取池化组件并 `GetEmitter(this)` 得到 `VolumeEmitter`，在其上 PostEvent / SetRTPC / SetNodeProperty。 |
| 出 | 玩家 PlayerAudioComponent | 直接调用 Type::Get | `PlayerAudioTriggerVolume` 在 `Position==OnPlayer` 时于 `Player.PlayerAudioComponent.PostEvent` 在玩家身上播事件。 |
| 出 | USpotSoundComponent / FSoundDefReference | 可组合设置 ApplySettings | `ActorSpotAudioTriggerVolume` 为重叠 Actor `GetOrCreate` SpotSound 组件并赋 `Settings`/Zone 配置后 `Start`；`SoundDefRef.SpawnSoundDefOneshot` 派生一次性 SoundDef。 |
| 出 | AHazePlayerCharacter | 标签阻塞/禁用 | `PlayerAudioVolumeBase` 用 `AddActorDisable(this)` 实现 `bTriggerOnce` 一次性禁用，并按 Mio/Zoe 过滤触发对象。 |
| 双向 | UPassbyishTriggerEditorComponent（+Visualizer） | 直接调用 Type::Get | `PassbyishTriggerVolume` 持有该编辑器组件；Visualizer 反向 Cast 回体积读写 `SoundPosition/PassbyTransform`，提供声源位置编辑（仅编辑器）。 |

## 关键文件（逐文件一句话）

- `PlayerAudioVolumeBase.as`：玩家音频体积抽象基类，统一处理进出重叠、玩家过滤、冷却、池化发射器创建与 `PlayOnEnter/PlayOnExit` 钩子。
- `PlayerAudioTriggerVolume.as`：玩家进/出体积时按位置（玩家身上或体积中心）播放多组进入/退出事件与 SoundDef，可退出时淡出停止。
- `PassbyishTriggerVolume.as`：将发射器置于自定义位置、以 Passby 投递类型播放“掠过音”的玩家触发体积。
- `PassbyishTriggerVolumeEditorComponent.as`：仅编辑器组件与 Visualizer，提供掠过音声源位置的拖拽、可视化与拷贝/粘贴到光标。
- `ActorSpotAudioTriggerVolume.as`：针对指定 Actor 类的定点音体积，进出时为该 Actor 创建/驱动 SpotSound 组件播放事件或 SoundDef。
- `FoliageDetectionVolume.as`：用预烘焙稀疏球网格检测玩家所处植被类型，类型变化时 Trigger 植被重叠事件给 SoundDef 消费。
- `PlayerVOTriggerVolume.as`：玩家 VO 配音触发体积占位文件，目前仅含空 dummy 命名空间函数。

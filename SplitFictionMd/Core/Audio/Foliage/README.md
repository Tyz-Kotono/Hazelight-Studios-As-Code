# Core / Audio / Foliage

> 职责：定义"玩家穿过植被"的效果事件契约（数据结构 + 事件处理器基类），把检测体积的触发解耦给音频侧消费。共 1 个文件。

## 内部架构（关键类、基类、数据流）

本子目录只承担"事件契约"层，不做检测、也不直接播放声音，是检测端与音频端之间的中转接口。

- `FFoliageDetectionData`（结构体）：单次植被重叠事件的载荷。
  - `bIsOverlappingFoliage`：进入/离开植被的状态切换标志。
  - `Type`（`EFoliageDetectionType`，定义在 `Volumes/FoliageDetectionVolume.as`，取值 Grass/Bush/Plant）：植被密度类型。
  - `MaterialOverride`（`UPhysicalMaterialAudioAsset`）：按类型覆盖的物理材质音频资产，供 SoundDef 选音。
- `UFoliageDetectionEventHandler`（抽象类，基类 `UHazeEffectEventHandler`）：植被重叠效果事件处理器。
  - 暴露 `FoliageOverlapEvent(FFoliageDetectionData)`（`BlueprintEvent` + `AutoCreateBPNode`）作为可重写入口。
  - 继承自 `UHazeEffectEventHandler` 后，框架自动生成静态 `Trigger_FoliageOverlapEvent(Player, Data)` 触发函数，供外部按玩家广播。

数据流：
`AFoliageDetectionvolume.Tick` 检测每个玩家是否进入某类植被
→ 状态变化时调用 `OnPlayerOverlapFoliageChange`，组装 `FFoliageDetectionData`（含按类型查得的 `MaterialOverride`）
→ `UFoliageDetectionEventHandler::Trigger_FoliageOverlapEvent(Player, Data)`
→ 玩家身上的 `UHazeEffectEventHandlerComponent` 派发
→ 各 SoundDef 重写的 `FoliageOverlapEvent` 收到数据并播放对应植被音效。

## 协同关系

| 方向 | 对象 | 机制 | 说明（基于实证） |
| --- | --- | --- | --- |
| 入 | `AFoliageDetectionVolume`（`Core/Audio/Volumes/FoliageDetectionVolume.as`） | 委托 event/Broadcast | 体积在 `Tick` 中检测重叠并调用 `UFoliageDetectionEventHandler::Trigger_FoliageOverlapEvent(Player, Data)`（第 75 行），单向触发本目录定义的事件。 |
| 出 | `Foliage_SoundDef` / `Player_Movement_Basic_SoundDef` / `Movement_Foliage_SoundDef`（`Audio/SoundDefinitions/...`） | 委托 event/Broadcast | SoundDef 通过 `EventClassAndFunctionNames.Add(n"FoliageOverlapEvent", UFoliageDetectionEventHandler)` 订阅，并重写 `FoliageOverlapEvent(FFoliageDetectionData)` 消费事件播放音效。 |
| 出 | `UHazeEffectEventHandlerComponent`（玩家身上） | 直接调用 Type::Get | SoundDef 经 `UHazeEffectEventHandlerComponent::Get(Player)` 取得派发组件，作为事件由 Trigger 到 SoundDef 的中转。 |
| 入 | `UPhysicalMaterialAudioAsset` | 可组合设置 ApplySettings | 体积按 `EFoliageDetectionType` 索引 `MaterialOverridesPerType`，将材质资产填入 `FFoliageDetectionData.MaterialOverride` 随事件下发，供音频端选音。 |

## 关键文件（逐文件一句话）

- `FoliageDetectionEffectEventHandler.as`：定义植被重叠事件载荷 `FFoliageDetectionData` 与抽象事件处理器基类 `UFoliageDetectionEventHandler`，作为检测体积与音频 SoundDef 之间的事件契约。

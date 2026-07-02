# Core / Audio

> 职责：Split Fiction 的**运行时音频管线**——把游戏世界状态（玩家位置、移动、死亡、环境、过场）转换为 Wwise(Ak) 声音的整套运行时设施。单模块功能域，共 99 文件，分 15 个子目录，是 Core 中最大的功能域。

---

## 一、定位：运行时管线 vs 内容定义

引擎里有两个名字相近但职责完全不同的 Audio 目录，务必区分：

| 目录 | 角色 | 规模 | 内容 |
| --- | --- | --- | --- |
| **顶层 `Audio/SoundDefinitions/`** | **内容层**（声音"是什么"） | 约 1099 文件 | 每个 `*_SoundDef.as` 定义一种声音的资源、参数、播放规则。设计/音频师维护。 |
| **`Core/Audio/`（本目录）** | **运行时管线**（声音"怎么放") | 99 文件 | 发射器/监听器/区域/触发体积/调试等运行时机制。把 SoundDef 内容在正确时机、正确空间位置投放出去。 |

本目录的静态库与工具大多挂在 `Audio::` 命名空间下（见 `AudioStatics.as`），底层桥接 Wwise 的 `Ak` 系统（`UHazeAudioComponent`、`FAkSoundPosition` 等）。

---

## 二、运行时三角色

整条管线围绕三个核心运行时对象组织：

| 角色 | 类型 | 实例化方式 | 职责 |
| --- | --- | --- | --- |
| **发射器 Emitter** | `UHazeAudioEmitter` | **池化**：`Audio::GetPooledEmitter(Params)` 取、`Audio::ReturnPooledEmitter()` 归还 | 声音的"发声口"。承载一个 `UHazeAudioComponent`，负责 PostEvent、设置衰减/RTPC/节点属性、多位置(MultiPosition)、空间声像。 |
| **监听器 Listener** | `UHazeAudioListenerComponent` | **每玩家一个**：`Player.PlayerListener` | 声音的"耳朵"。决定从哪个位置/朝向收听空间音频。由一组 Listener Capability 通过标签竞争控制其变换。 |
| **SoundDef 运行时对象** | `UHazeSoundDefBase` 及其行为模块 | 由 `SoundDef::SpawnSoundDefSpot` 等生成 | 顶层 SoundDefinitions 内容在运行时的活动实例；可挂载 `SoundDefObjects/` 中的行为模块（包络/多普勒/经过音等）动态调制。 |

典型投放链路（以玩家发射器播一次性事件为例，`AudioStatics.as:319`）：
`Audio::PostEventOnPlayer(Player, Event)` → `Player.PlayerAudioComponent.GetEmitter()` → `Emitter.PostEvent(Event)` → 设置 NodeProperty / RTPC。

---

## 三、分屏声像（Split-Screen Panning）

游戏为双人合作分屏，声音需按玩家 **Mio / Zoe** 在屏幕上的位置做声像（panning）。相关算法集中在 `AudioStatics.as`：

- `GetPlayerPanningValue` / `GetScreenPositionRelativePanningValue`：按世界坐标投影到视口，算左右声像。
- `SetPanningBasedOnScreenPercentage` / `SetSidescrollerScreenPositionRelativePanning`：按各玩家视口占比（全屏/横版分屏模式）分配声像；Mio 偏左(-1)、Zoe 偏右(+1)。
- 由 `Capabilities/PlayerSetPanningCapability` 与各 Listener Capability 每帧驱动。

---

## 四、五种协作机制

本域内部及与外部（Gameplay / LevelSpecific）的协同统一遵循 Haze 的五种机制：

1. **直接调用 `Type::Get`** —— 如 `UHazeAudioListenerComponent::Get(Player)`、`AudioDebugManager::Get()`。
2. **Crumb（`HasControl` + `CrumbFunction`）** —— 网络/控制权一致性。
3. **标签阻塞 `CapabilityTags` / `BlockCapabilities`** —— Listener 与 Reflection 的多 Capability 竞争（如 `Audio::Tags::Listener`、`ReflectionTracing`）。
4. **委托 `event` / `Broadcast`** —— 死亡/伤害/移动/植被的事件分发（`UHazeEffectEventHandler` 体系）。
5. **可组合设置 `ApplySettings`** —— `UHazeComposableSettings` 子类按 Instigator 叠加（呼吸、发力、代理发射器激活等）。

---

## 五、15 子目录索引

| 子目录 | 文件数 | 一句话职责 |
| --- | --- | --- |
| [AnimNotifies](./AnimNotifies/README.md) | 2 | 动画通知帧投放一次性音频事件（通用 / 区分 Mio·Zoe 双事件）。 |
| [Capabilities](./Capabilities/README.md) | 4 | 玩家级音频能力：默认 RTPC、死亡滤波、GameOver 音、分屏声像。 |
| [Death](./Death/README.md) | 3 | 玩家死亡/受伤音频：滤波管理器、伤害死亡音组件、VO 死亡设置。 |
| [Debug](./Debug/README.md) | 约33 | 音频调试系统：管理器 + 19 个领域调试处理器 + Widget + 网络调试（多在 TEST/EDITOR 下）。 |
| [Foliage](./Foliage/README.md) | 1 | 植被穿行音效的事件契约层（由 Volumes 触发、SoundDef 订阅）。 |
| [InstanceLimiting](./InstanceLimiting/README.md) | 5 | 限制同一 SoundDef 同时播放实例数：可扩展条件（范围内监听器/最大数/夹角内）。 |
| [Listeners](./Listeners/README.md) | 3 | 监听器变换控制 Capability：默认（耳朵↔相机插值）、过场、调试相机；按标签竞争。 |
| [Movement](./Movement/README.md) | 8 | 移动音频：脚步线（EventHandler + 物理材质）与发力/喘息线（Capability + 组件 + 设置）。 |
| [Physics](./Physics/README.md) | 2 | 物理材质音频资产 + 玩家材质追踪组件（材质→脚步等音效参数）。 |
| [Reflection](./Reflection/README.md) | 5 | 声音早期反射：射线追踪组件 + 三种追踪策略 Capability（默认/全屏/静态，标签竞争）。 |
| [Settings](./Settings/README.md) | 2 | 可组合音频设置：呼吸参数、默认代理发射器激活参数。 |
| [SoundDefObjects](./SoundDefObjects/README.md) | 4 | 挂在 SoundDef 上的运行时调制模块：AHR 包络、交叉淡变、多普勒、经过音。 |
| [SpotSound](./SpotSound/README.md) | 10 | 关卡定点环境声组件：Event/SoundDef 两路 + Basic/Multi/Plane/Spline 模式多态 + 区域遮挡联动。 |
| [Volumes](./Volumes/README.md) | 7 | 音频触发体积：玩家进出体积基类及派生（事件/VO/掠过音）、植被检测体积、定点 Actor 触发。 |
| [Zones](./Zones/README.md) | 8 | 空间音频区域：管理器 + 环境/遮挡/混响/传送门/水区家族 + 查询静态库 + 遮挡联动。 |

> 子目录文件数合计 67；其余约 32 文件为本目录根级的全局设施（见下）。

---

## 六、根级全局设施（不属于任何子目录）

| 文件 | 角色 |
| --- | --- |
| `AudioStatics.as` | 核心静态库（`Audio::` 命名空间）：发射器/监听器辅助、分屏声像、dB/振幅换算、全局 RTPC、`PostEventOnPlayer`、Music 入口等。 |
| `AudioVOStatics.as` | VO（配音）相关 RTPC 读取（如 Zoe·Gaia 嗓音电平、广播音量）。 |
| `BreathingStatics.as` | 呼吸音类型定义（吸/呼气、张/闭嘴事件）。 |
| `MovementAudioStatics.as`（在 Movement/） | 移动音静态工具（见 Movement 子目录）。 |
| `AudioLevelScriptActor.as` / `MusicLevelScriptActor.as` / `VOLevelScriptActor.as` | 关卡脚本 Actor：在关卡载入/切换时驱动环境声、音乐、VO 的入口。 |
| `HazeAmbientSound.as` | 通用环境声 Actor（`AHazeAmbientSound`），摆放即播放的环境音源。 |

---

## 七、与外部功能域的关系（实证）

- **Gameplay/Movement/Player/Audio/**：定义 `PlayerSidescrollerListenerCapability`、`PlayerFullscreenListenerCapability` 等，复用本域 `Audio::Tags::Listener` 标签竞争监听器控制权。
- **LevelSpecific/\*/Audio/**：各关卡专属监听器（Prison 无人机/远程黑客、Summit 龙、Coast 翼装等）同样复用 Listener 标签，并通过 `Audio::PostEventOnPlayer` 等投放音频。
- **PlayerHealth**：死亡/伤害委托驱动 `Death/` 与 `Capabilities/PlayerFilterAudioOnDeathCapability`。
- **顶层 `Audio/SoundDefinitions/`**：通过 `SpotSound`、`Foliage`、`InstanceLimiting`、`SoundDefObjects` 等机制被运行时实例化与调制。

# Core / Audio / Settings

> 职责：定义音频域的"可组合设置"资产（呼吸音频、默认代理发射器激活），供各处通过 ApplySettings 叠加生效、由消费方 GetSettings 读取。共 2 个文件。

## 内部架构（关键类、基类、数据流）

本目录两个类均继承自 Haze 的可组合设置基类 `UHazeComposableSettings`，其本质是"纯数据 + 叠加机制"：调用方将设置实例 `ApplySettings(实例, Instigator[, 优先级])` 应用到 `AHazePlayerCharacter`，多个 Instigator 的设置按优先级叠加；消费方通过 `类名::GetSettings(Player)` 拿到当前生效的合并结果；退出时用 `ClearSettingsOfClass` / `ClearSettingsWithAsset` 按 Instigator 撤销。

- **UPlayerBreathingAudioSettings**：玩家呼吸音频参数。字段 `bForceOpenMouth`（强制张嘴）、`OpenMouthFactor`（张嘴系数）、`Low/Mid/HighExertionCycleThreshold`（低/中/高强度呼吸的循环阈值，默认 25/50/75）。数据流：`PlayerVOTriggerVolume` 在其 `Settings Override` 字段配置实例并对进入体积的玩家应用；`PlayerMovementAudioComponent` 提供 `ApplyBreathingSettings` / `RemoveBreathingSettingsOverride` mixin 作为应用/撤销入口；呼吸类 SoundDef（`VO_Player_Breathing_SoundDef`、`VO_Player_Skydive_Breathing_SoundDef`）在运行时 `GetSettings` 读取阈值驱动播放。

- **UPlayerDefaultProxyEmitterActivationSettings**：控制"默认代理发射器（proxy emitter）"何时/如何激活。字段 `bCanActivate`（是否允许默认激活能力）、`bIncludeVOInDefaultProxies`（默认代理是否含 VO）、`CameraDistanceActivationBufferDistance`（相对 IdealCameraDistance 的激活缓冲距离，默认 300cm）、`DefaultAttenuation` / `SideScrollerAttenuation`（默认/横版衰减）。数据流：`PlayerProxyEmitterSpatializationBaseCapability` 在 `Setup()` 中 `GetSettings(Player)` 读取并据此做代理发射器空间化与激活判定；关卡脚本 `AudioLevelScriptActor` 持有默认配置；多处用 `asset ... of` 声明预设实例并在能力中 `ApplySettings` 覆盖（如 `PlayerProxyEmitterDisablingCapability` 用 `Override` 优先级禁用、Tundra 各变身能力、Summit 小龙攀爬能力）。

## 协同关系

| 方向 | 对象 | 机制 | 说明（基于实证） |
| --- | --- | --- | --- |
| 入站 | `Gameplay/Audio/Volumes/PlayerVOTriggerVolume.as` | 可组合设置 ApplySettings | 体积持有 `UPlayerBreathingAudioSettings BreathingSettings` 字段，对进入玩家应用，离开时 `ClearSettingsOfClass(UPlayerBreathingAudioSettings, this)` 撤销 |
| 入站 | `Gameplay/Movement/Audio/PlayerMovementAudioComponent.as` | 可组合设置 ApplySettings | mixin `ApplyBreathingSettings` 调 `Player.ApplySettings(Settings, Instigator)`；`RemoveBreathingSettingsOverride` 调 `ClearSettingsOfClass` |
| 出站 | `Audio/SoundDefinitions/VO_Player_Breathing_SoundDef.as`、`VO_Player_Skydive_Breathing_SoundDef.as` | 可组合设置 GetSettings | 运行时 `UPlayerBreathingAudioSettings::GetSettings(PlayerOwner)` 读取阈值/张嘴参数驱动呼吸播放 |
| 出站 | `Gameplay/Movement/Player/Audio/PlayerProxyEmitterSpatializationBaseCapability.as` | 可组合设置 GetSettings | `Setup()` 中 `UPlayerDefaultProxyEmitterActivationSettings::GetSettings(Player)`，据缓冲距离/衰减做代理激活与空间化 |
| 入站 | `Gameplay/Movement/Player/Audio/PlayerProxyEmitterDisablingCapability.as` | 可组合设置 ApplySettings | `asset ... of` 预设 `bCanActivate=false`，`OnActivated` 以 `EHazeSettingsPriority::Override` 应用以临时禁用代理 |
| 入站 | `LevelSpecific/Tundra/Audio/*ShapeshiftAudioCapability.as`、`LevelSpecific/Summit/Audio/PlayerBabyDragonTailClimbAudioCapability.as` | 可组合设置 ApplySettings | 各变身/攀爬能力用 `asset ... of UPlayerDefaultProxyEmitterActivationSettings` 声明专属预设并叠加覆盖 |
| 入站 | `Audio/AudioLevelScriptActor.as` | 直接持有 | 关卡脚本持有 `UPlayerDefaultProxyEmitterActivationSettings DefaultProxyActivationSettings` 作为默认配置 |
| 出站 | `Audio/Debug/DebugTypes/AudioDebugProxy.as` | 可组合设置 GetSettings | 调试可视化通过 `Player.GetSettings(...)` + Cast 读取当前代理激活设置 |
| 同域 | `Core/Audio/BreathingStatics.as` | 直接调用 Type::Get（数据结构） | 提供 `FBreathingTagData`（吸/呼气事件、随机偏移），与呼吸设置配套描述呼吸音频事件数据，由呼吸 SoundDef 消费 |

## 关键文件（逐文件一句话）

- `PlayerBreathingAudioSettings.as`：可组合设置，定义玩家呼吸音频的张嘴开关/系数与低中高强度循环阈值。
- `PlayerDefaultProxyEmitterActivationSettings.as`：可组合设置，控制默认代理发射器是否激活、激活缓冲距离、VO 包含与默认/横版衰减。

# Core / Systems / Settings

> 职责：可组合设置的应用——`BeginPlay` 在 actor 上批量套用 `UHazeComposableSettings`，以及把音频设置应用为全局 RTPC。共 2 文件。

## 内部架构

本模块对应协作五机制中的 **「可组合设置（ApplySettings）」**：设置以 `UHazeComposableSettings` 数据资产形式存在，由组件在合适时机推送到目标 actor。

### 关键类

| 类 | 基类 | 角色 |
| --- | --- | --- |
| `UDefaultSettingsComponent` | `UActorComponent` | `BeginPlay` 时把配置的 `DefaultSettings` 数组逐个 `ApplyDefaultSettings` 到宿主 `AHazeActor` |
| `UGameSettingsApplicator` | `UHazeGameSettingsApplicatorBase` | 玩家音频/扬声器设置 → 全局 Wwise RTPC 的应用器 |
| `UHazeAudioDefaultMenuSettings` | `UHazeAudioDefaultMenuSettingsBase` | 提供音频菜单默认值（音量上限、扬声器类型、动态范围、声道配置） |

### 数据流

- **`UDefaultSettingsComponent.BeginPlay`**：`Cast<AHazeActor>(Owner)` 后遍历 `TArray<UHazeComposableSettings> DefaultSettings`，对每项调用 `HazeOwner.ApplyDefaultSettings(Settings)`——把可组合设置注入引擎的设置合成栈。非 `AHazeActor` 宿主会 `ensure` 报错。
- **`UGameSettingsApplicator`**：各 `ApplyAudio*`（主/语音/音乐/SFX 音量、扬声器类型、声道、动态范围、夜间/主播模式）重写把值写成 `AudioComponent::SetGlobalRTPC`（RTPC id 集中在 `Audio` 命名空间常量）。`GetValidDynamicRangeBasedOnSpeakerType` 用 `SettingsToDynamicRange` 表按扬声器类型钳制动态范围；`ChangeDefaultSettingIfDefaultSet` 仅在用户未改过该项时更新默认值。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
| --- | --- | --- | --- |
| 调用 | `AHazeActor::ApplyDefaultSettings` | 直接调用（机制5） | `UDefaultSettingsComponent` 推送 `UHazeComposableSettings` |
| 同机制 | `Triggers/ApplySettingsTrigger.as`、各 `*Settings.as`（Movement/Camera/Aiming/PlayerHealth…，26+ 处） | 可组合设置 | 全项目大量 `UHazeComposableSettings` 派生类共用同一应用通道；本组件提供「随 actor 默认套用」的入口 |
| 调用 | `AudioComponent::SetGlobalRTPC`、`Audio::SetPanningRule` | 直接调用 | 把设置落为 Wwise 全局参数 |
| 依赖 | `GameSettings::`（Get/Set/Description） | 直接调用 | 读取/更新游戏设置项与默认值 |
| 依赖 | `Game::PlatformName` / `Audio::GetSpeakerConfiguration` | 直接调用 | 按平台与硬件推断默认扬声器/声道 |

## 关键文件

- `DefaultSettingsComponent.as` — `UDefaultSettingsComponent`：`BeginPlay` 遍历 `DefaultSettings` 数组对宿主 `AHazeActor` 调用 `ApplyDefaultSettings`。
- `GameSettingsApplicator.as` — `UGameSettingsApplicator`（音频/扬声器设置→全局 RTPC，含夜间/主播模式与动态范围钳制）、`UHazeAudioDefaultMenuSettings`（菜单默认值）、`Audio` 命名空间 RTPC 常量。

# Core / Audio / Debug

> 职责：运行时音频调试系统——通过控制台变量开关、统一的“调试类型处理器”插件化架构、世界/视口可视化与文本绘制、过滤器与开发菜单，按领域（玩家/组件/SoundDef/Zone/Spot/事件/音乐/性能等约 20 个领域）可视化音频管线状态。约 27 个文件，绝大多数代码位于 `#if TEST` / `#if EDITOR` 守卫内，发布版无开销。

## 内部架构（关键类、基类、数据流）

### 1. 枚举与开关层（AudioDebug.as）
- 定义全部调试领域枚举：`EHazeAudioDebugType`（约 20 个领域 + `NumOfTypes`）、`EDebugAudioWorldVisualization`（世界可视化子集）、`EDebugAudioViewportVisualization`（视口可视化全集）、`EDebugAudioFilter`（过滤分类）、`EDebugAudioOutputBlock`（网络静音侧）。三个枚举共享前缀顺序，故可用同一位下标互转。
- `namespace AudioDebug`：所有开关逻辑的入口。每个领域对应一个 `HazeAudio.DebugXxx` 控制台变量；世界/视口可视化各用一个位标志 CVar（`DebugWorldVisualizationFlags` / `DebugViewportVisualizationFlags`）。提供 `IsEnabled(...)`（按枚举重载）、`ToggleDebugging`、`SetConsoleCvar`、`CheckConsoleVars`（每帧同步控制台与位标志）、`IsAnyDebugFlagSet`、`GetActorLabel` 等。这是其它模块查询“某领域是否开启”的统一 API。

### 2. 管理器层（AudioDebugManager.as）
- `UAudioDebugManager : UHazeAudioDebugManager`——核心调度器（C++ 基类的 AngelScript 扩展）。持有 `TArray<UAudioDebugTypeHandler> DebugTypeHandlers`（按 `EHazeAudioDebugType` 下标定长）。
- `Setup()`：反射枚举 `UAudioDebugTypeHandler` 的全部子类，逐个实例化并按 `Type()` 注册进数组——这是 DebugTypes 的插件化注册机制。
- `Tick()`：每帧调 `AudioDebug::CheckConsoleVars()` 同步开关，再遍历所有 Handler，按世界/视口开关位调用 `Visualize()`（世界绘制）与 `Draw()/DrawCustom()`（视口 widget 绘制），并触发 `OnWorldToggled/OnViewToggled` 跳变回调。
- 双玩家绘制：内嵌 `FAudioDebugDrawHelper`（封装 `UHazeImmediateWidget` 即时模式绘制），Mio/Zoe 各一份，按视口可见性选择 `GetPreferredDrawer`。
- 注册表：`RegisterSpot/UnregisterSpot`（SpotSound 自注册），并经基类提供 `GetRegisteredComponents/GetRegisteredSoundDefs`。
- 过滤入口：`IsFiltered(...)`、`IsFilterEmpty(...)` 委托给 Config 中的过滤器。

### 3. 配置与过滤层（AudioDebugConfig.as、AudioDebugFilter.as）
- `UHazeAudioDevMenuConfig` / `UHazeAudioDebugConfig`：`Config = EditorPerProjectUserSettings` 持久化配置，存世界/视口过滤器、杂项标志与位标志，`Save()` 写入 ini。`namespace AudioDebug` 提供 `GetMenuConfig/GetConfig`（取 DefaultObject 单例）。
- `FAudioDebugFilter`：按领域（`EDebugAudioFilter`）的文本过滤器，支持 `or/and/not` 布尔操作符，解析为 `FAudioDebugTypeFilters{Any/All/Exclusions}`，核心方法 `IsNameFiltered`。`FAudioDebugMiscFlags`：一组可视化开关（显示 RTPC、AuxSends、衰减、禁用的 SoundDef 等）。

### 4. 处理器基类（AudioDebugHandler.as）—— DebugTypes 的统一模式
- `UAudioDebugTypeHandler : UObject`：所有领域处理器的基类。约定虚函数 `Type()`（返回所属领域枚举）、`GetTitle()`、生命周期 `Setup/Shutdown/OnWorldToggled/OnViewToggled`，以及三类绘制：`Visualize()`（世界）、`Draw()/DrawCustom()`（视口 widget）、`Menu()`（开发菜单额外 UI）。标志位 `bUseCustomDrawing`、`bUseViewportDrawer` 控制绘制路径。
- `namespace AudioDebug::GetHandlerOfType` 用瞬态包按类查找/创建单例实例。

### 5. DebugTypes/（19 个文件，统一模式）
每个文件一个 `UAudioDebugXxx : UAudioDebugTypeHandler` 子类，覆盖一个领域，仅重写所需虚函数。按覆盖领域分类：
- 世界+视口共享：`Players` `Components` `SoundDefs` `Zones` `Spots` `Splines` `Delays` `Gameplay`
- 视口为主：`Events` `Banks` `Music` `BusMixers` `NodeProperties` `Network` `Proxy` `Loudness`
- 独立/特殊：`Cutscenes` `Effects` `Performance`（`Performance` 按需注册 `RegisterOnOutputDeviceMetering/RegisterResourceMonitoring` 回调以降低开销）

### 6. Widget 层（AudioGraphWidget.as、AudioViewportWidget.as）
- `UAudioGraphWidget : UHazeAudioGraphWidget`：折线图绘制，`OnPaint` 把 `GraphEntries` 采样画成曲线 + 鼠标游标取值（用于响度/性能等时序数据）。
- `UAudioViewportWidget : UHazeAudioViewportOverlayWidget`：视口叠加容器，含 `DynamicContent`（即时绘制宿主）与垂直框增删子控件。

### 7. DebugCapabilities/ 与网络调试
- `UAudioGameplayDebugCapabilityBase : UHazeCapability`：Capability 基类，`ShouldActivate/Deactivate` 直接挂钩 `AudioDebug::IsEnabled(Gameplay)`——游戏玩法相关音频调试 Capability 的统一父类。
- `HazeAudioDebugNetworkCapability.as` 中 `UPlayerAudioDebugNetworkCapability : UHazePlayerCapability`：仅当 `UHazeAudioNetworkDebugManager::IsNetworkSimulating()` 且玩家为 Mio 时激活，读 `DisableAudioOutputs` CVar，调 `UHazeAudioNetworkDebugManager::Get(this).SetNetworkAudioOutput(...)` 静音 Remote/Control 侧，用于网络音频差异排查。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
| --- | --- | --- | --- |
| 入 | SpotSoundComponent | 直接调用 Type::Get | `SpotSoundComponent.as:367,381` 在 `BeginPlay/EndPlay` 调 `AudioDebugManager::Get()` 后 `RegisterSpot/UnregisterSpot` 自注册到管理器 |
| 出 | DebugTypes 各 Handler | 直接调用（反射注册） | `AudioDebugManager.as:115` 反射 `UAudioDebugTypeHandler` 全部子类并按 `Type()` 注册，Tick 中回调其 `Visualize/Draw` |
| 双 | 监听器 Capability | 标签阻塞 CapabilityTags | `PlayerDefaultListenerCapability.as:136` 等以 `AudioDebug::IsEnabled(Players)` 决定是否启用调试绘制分支 |
| 双 | Gameplay 调试 Capability | 标签阻塞 CapabilityTags | `AudioGameplayDebugCapabilityBase.as:6,12` 的 `ShouldActivate/Deactivate` 由 `AudioDebug::IsEnabled(Gameplay)` 驱动 |
| 入 | SpotSoundPlaneLookAtVolume / Zones / Components | 直接调用 | 多处以 `AudioDebug::IsEnabled(Spots/...)` 查询开关后决定自身可视化（`SpotSoundPlaneLookAtVolume.as:109,168` 等） |
| 出 | 控制台 CVar | 可组合设置 ApplySettings | `AudioDebug::SetConsoleCvar/ToggleDebugging` 写 `HazeAudio.Debug*` CVar，`CheckConsoleVars` 每帧同步位标志与开关状态 |
| 出 | 网络调试管理器 | 直接调用 Type::Get | `HazeAudioDebugNetworkCapability.as:28` 调 `UHazeAudioNetworkDebugManager::Get(this).SetNetworkAudioOutput(...)`；`AudioDebugNetwork.as:216` 取另一端管理器对比 |
| 出 | 配置单例 | 可组合设置 ApplySettings | Handler/Manager 经 `AudioDebug::GetMenuConfig/GetConfig` 读写过滤器与杂项标志并 `SaveConfig` 持久化 |

## 关键文件（逐文件一句话）

- `AudioDebug.as`：调试领域枚举 + `namespace AudioDebug` 开关 API（CVar 与位标志的开/关/同步/互转），全系统状态入口。
- `AudioDebugManager.as`：`UAudioDebugManager` 调度器——反射注册所有 Handler、每帧 Tick 驱动可视化/绘制、双玩家即时绘制、Spot 注册表。
- `AudioDebugConfig.as`：`UHazeAudioDevMenuConfig` / `UHazeAudioDebugConfig` 持久化配置单例（过滤器、杂项标志、位标志）。
- `AudioDebugFilter.as`：`FAudioDebugFilter` 按领域文本过滤器（or/and/not 布尔解析）+ `FAudioDebugMiscFlags` 可视化开关集。
- `AudioDebugHandler.as`：`UAudioDebugTypeHandler` 基类——定义 DebugTypes 的统一虚函数协议与单例查找。
- `AudioGraphWidget.as`：`UAudioGraphWidget` 时序折线图绘制（采样曲线 + 游标取值）。
- `AudioViewportWidget.as`：`UAudioViewportWidget` 视口叠加容器（即时绘制宿主）。
- `HazeAudioDebugNetworkCapability.as`：`UPlayerAudioDebugNetworkCapability` 网络模拟下按 CVar 静音 Remote/Control 侧音频输出。
- `DebugCapabilities/AudioGameplayDebugCapabilityBase.as`：玩法音频调试 Capability 基类，激活条件挂钩 Gameplay 开关。
- `DebugTypes/`（19 个 `UAudioDebugXxx : UAudioDebugTypeHandler` 子类，各覆盖一领域）：
  - 共享世界+视口：`Players` `Components` `SoundDefs` `Zones` `Spots` `Splines` `Delays` `Gameplay`。
  - 视口为主：`Events` `Banks` `Music` `BusMixers` `NodeProperties` `Network` `Proxy` `Loudness`。
  - 独立/特殊：`Cutscenes` `Effects` `Performance`（按需注册性能/资源监控回调）。

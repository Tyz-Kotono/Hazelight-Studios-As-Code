# Core / Audio / Reflection

> 职责：通过对环境几何做射线追踪，估算每个玩家在三个方向（Upwards / NorthWest / NorthEast）上的声音反射，并写回一组全局 RTPC / 反混响 AuxBus 喂给 Wwise（影响混响与空间感）。共 5 个文件。

## 内部架构（关键类、基类、数据流）

- **UAudioReflectionComponent**（继承 C++ 基类 `UHazeAudioReflectionComponent`）：每个玩家一个的反射组件，缓存追踪结果并把数据翻译成 RTPC。
  - 内部维护 `CacheByTraceType`（按 `EHazeAudioReflectionTraceType` 分通道的 `FChannelRuntimeData`，记录上次命中、物理材质、距离 Alpha）。
  - `bIsMio` 决定使用哪一套 RTPC ID（Mio / Zoe）。
  - 追踪本身（`TraceSingle` / `IsBusy` / `OnTraceDone` 委托 / `InitTraceSettings` / `AddActorToIgnore` / `SetReflectionSendToReverbBus`）由 C++ 基类提供，.as 子类只负责消费命中结果。
  - 数据流：`OnZoneChanged()` 在区域切换时读 `UHazeAudioReflectionDataAsset` 写静态 RTPC 与混响 AuxBus；`UpdateReflectionChannel()` 在每次追踪命中后按距离 Alpha + 物理材质硬度（Soft/Hard 频率乘子）算动态 RTPC。

- **AudioReflection 命名空间静态库**（`AudioReflectionStatics.as`）：把 (玩家, 方向, RtpcType) 映射到具体 Wwise 全局 RTPC ID 并写入。
  - 预构建 `MiosReflectionRtpcs` / `ZoesReflectionRtpcs`（按方向 × 10 个 RTPC 的固定顺序，顺序硬编码不可错位）。
  - `SetReflectionRtpc(...)` 多个重载最终调用 `AudioComponent::SetGlobalRTPC`；`ReverbSendLevel` 类型走 `Component.SetReflectionSendToReverbBus` 改混响总线发送。
  - 一组 `BlueprintPure` 辅助函数（`GetRuntimeReflectionData` / `GetReflectionTraceAlpha` / `IsTraceBlocking` 等）供外部只读查询缓存。

- **UPlayerAudioReflectionTraceCapability**（继承 `UHazePlayerCapability`，TickGroup = AfterGameplay）：默认追踪策略。
  - `Setup()` 通过 `UAudioReflectionComponent::GetOrCreate(Player)` 取组件，`OnActivated()` 绑定 `ReflectionComponent.OnTraceDone` 到 `OnTraceFinished`。
  - `TickActive()` 用监听器位置 + 方向向量发起 3 次 `TraceSingle`；`OnTraceFinished()` 读当前混响区域反射资产，调 `OnZoneChanged` / `UpdateReflectionChannel` 把命中写回。
  - **Fullscreen / Static 是它的子类**，复写追踪来源或激活条件。

数据流总览：`TraceCapability.TickActive` 发起追踪 → 基类完成后触发 `OnTraceDone` → `OnTraceFinished` → `UAudioReflectionComponent` 更新缓存并调 `AudioReflection::SetReflectionRtpc` → `AudioComponent::SetGlobalRTPC` / 反混响 AuxBus → Wwise。

## 协同关系

| 方向 | 对象 | 机制 | 说明（基于实证） |
| --- | --- | --- | --- |
| 出 | `AudioComponent::SetGlobalRTPC` / Wwise | 直接调用 Type::Get | `AudioReflectionStatics.as` 把反射估值写入全局 RTPC 与反混响 AuxBus，是本目录最终产物。 |
| 出/入 | `UAudioReflectionComponent` | 直接调用 Type::Get | Trace Capability 用 `GetOrCreate` / `Get` 拿组件；组件由 C++ 基类 `UHazeAudioReflectionComponent` 提供追踪 API。 |
| 入 | `UHazeAudioReflectionComponent.OnTraceDone` | 委托 event/Broadcast | 默认 Capability `OnActivated` 绑定 `OnTraceFinished`，追踪完成后回调以更新通道。 |
| 出 | 同玩家 Default 追踪（`DefaultReflectionTracing`） | 标签阻塞 CapabilityTags | Fullscreen Capability 激活时 `BlockCapabilities(DefaultReflectionTracing)`，全屏时接管追踪。 |
| 出 | 同玩家 Default + Fullscreen 追踪 | 标签阻塞 CapabilityTags | Static Capability 同时 Block `DefaultReflectionTracing` 与 `FullscreenReflectionTracing`，区域有静态反射资产时接管为最高优先级。 |
| 入 | 三种追踪 Capability（`ReflectionTracing` 父标签） | 标签阻塞 CapabilityTags | `PlayerCutsceneListenerCapability` 在 Cutscene 中 `BlockCapabilities(ReflectionTracing)`，一次性阻塞全部反射追踪。 |
| 入 | `AHazeAudioZone` / `UHazeAudioReflectionDataAsset` | 可组合设置 ApplySettings | `OnZoneChanged` 从区域反射资产 `GetTraceValues` 读 Static/Dynamic RTPC 配置并应用；区域提供混响 AuxBus。 |
| 入 | `UPhysicalMaterialAudioAsset` | 可组合设置 ApplySettings | 命中体的物理材质硬度（Soft/Hard）决定频率乘子，参与动态 RTPC 计算。 |
| 入 | `WingSuit` / `GravityBikeFree` 等关卡 Capability | 直接调用 Type::Get | 关卡专属能力对组件调 `AddActorToIgnore` / `SetMovementComponentOverride`，调整追踪忽略对象与世界上方向。 |

## 关键文件（逐文件一句话）

- **AudioReflectionComponent.as**：每玩家反射组件，缓存追踪命中并按区域资产 + 物理材质把反射估值翻译成静态/动态 RTPC。
- **AudioReflectionStatics.as**：`AudioReflection` 命名空间静态库，按 (玩家, 方向, RtpcType) 固定顺序映射并写入 Wwise 全局 RTPC / 反混响发送，附只读查询辅助函数。
- **PlayerAudioReflectionTraceCapability.as**：默认追踪策略 Capability，按监听器方向发起三方向射线追踪并经 `OnTraceDone` 回调更新组件（Fullscreen/Static 的基类）。
- **PlayerAudioReflectionTraceFullscreenCapability.as**：全屏单人镜头时启用，改用玩家旋转作追踪来源，并 Block 默认追踪。
- **PlayerAudioReflectionTraceStaticCapability.as**：区域提供静态反射资产时启用，仅在换区时更新发送，并同时 Block 默认与全屏追踪（最高优先级）。

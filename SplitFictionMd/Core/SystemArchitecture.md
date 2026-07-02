# Core / System 架构 — System Architecture

> 本文分析 Core 中 **System 级**（全局骨架）`.as` 文件的架构。
>
> 区别于普通的 Component（挂 actor、多实例）与 Capability（每帧行为、多实例），System 级文件是**全局唯一、跨 actor / 跨玩家协调**的骨架。

---

## 一、如何识别"System 级"文件

由继承的引擎基类标识：

| 基类 | 含义 |
|---|---|
| `UHazeSingleton` / `UHaze*BaseSingleton` | 游戏运行时全局单例 |
| `UHazeEditorSubsystem` / `UHazePrefabEditorSubsystem` | 编辑器子系统 |
| `UHaze*Manager`（引擎封装的单例族） | 领域管理器 |

### Core 中的清单（16 个）

| 文件 | 类 : 基类 | 角色 |
|---|---|---|
| `TimeDilation/TimeDilationEffectSingleton.as` | `UTimeDilationEffectSingleton : UHazeSingleton` | 运行时·效果聚合 |
| `Fades/FadeSingleton.as` | `UFadeSingleton : UHazeSingleton` | 运行时·加载屏流程 |
| `Rendering/RenderingSettingsSingleton.as` | `URenderingSettingsSingleton : UHazeSingleton` | 运行时·分层设置 |
| `Disable/DisableComponent.as` | `UDisableComponentSingleton : UHazeSingleton` | 运行时·性能注册表 |
| `Audio/Death/PlayerAudioFilterDeathManager.as` | `: UHazeSingleton` | 运行时·音频死亡 |
| `Camera/CameraSingleton.as` | `UCameraSingleton : UHazeCameraSingleton` | 运行时·分屏协调 |
| `Vox/HazeVoxRunner.as` | `UHazeVoxRunner : UHazeVoxRunnerBaseSingleton` | 运行时·语音播放 |
| `Vox/HazeVoxController.as` | `UHazeVoxController : UHazeVoxControllerBaseSingleton` | 运行时·语音守门 |
| `Audio/Zones/AudioZoneManager.as` | `UAudioZoneManager : UHazeAudioZoneManager` | 运行时·音区 |
| `Audio/Debug/AudioDebugManager.as` | `: UHazeAudioDebugManager` | 调试 |
| `Wardrobe/MeshDissolveComponent.as` | `: UHazeMeshDissolveManagerComponent` | 运行时·溶解 |
| `Sequencer/SequencerRenderSingleton.as` | `: UHazeSequenceRenderBaseSingleton` | 运行时·序列注册表 |
| `Prefab/PrefabEditorSubsystem.as` | `: UHazePrefabEditorSubsystem` | 编辑器 |
| `Props/PropLineEditorSubsystem.as` | `: UHazeEditorSubsystem` | 编辑器 |
| `Visibility/VisibilityVolume.as` | `UVisibilityVolumeEditorSubsystem : UHazeEditorSubsystem` | 编辑器 |

> 另：`Team/HazeTeamManager`、`Network` 等也具系统性质，但更偏"服务对象工厂"，不在本次单例族重点内。

---

## 二、统一架构骨架（5 大支柱）

所有 System 级类共享同一套生命周期与协作约定，这是 Core 的"系统层契约"。

### 支柱 1 — 全局访问：`::Get()`
单例由引擎托管，任何地方用静态 `Type::Get()` 拿到唯一实例：
```cpp
UTimeDilationEffectSingleton Singleton = UTimeDilationEffectSingleton::Get();
```
脚本侧从不 `new` 单例，也不持有其引用——每次现取。

### 支柱 2 — 集中 Tick：聚合循环
单例重写 `UFUNCTION(BlueprintOverride) void Tick(float DeltaTime)`，在**一个循环里推进所有活动条目**。以 TimeDilation 为例：
```cpp
TArray<FActiveTimeDilation> ActiveEffects;          // 全局活动表
void Tick(float DeltaTime){
  for (...ActiveEffects...)
      ActiveEffect.Timer += Time::UndilatedWorldDeltaSeconds;
  float Wanted = GetWantedWorldTimeDilation();       // 聚合(取最低)
  if (Wanted != AppliedWorldDilation)
      Time::SetWorldTimeDilation(Wanted);
}
```
**这是单例存在的根本理由**：把"多个来源想要的效果"聚合成"世界唯一的最终结果"。聚合策略各异——TimeDilation 取**最低倍率**、Fade 取**最强 alpha**、Rendering 取**分层叠加**。

### 支柱 3 — 关卡边界生命周期
单例跨关卡存活，因此必须主动清状态：
```cpp
void ResetStateBetweenLevels(){
    ActiveEffects.Empty();
    Time::SetWorldTimeDilation(1.0);
}
```
`Rendering` 还用 `Shutdown()` 在 PIE 结束时把 SSS/SSR 恢复回编辑器预览态——**单例负责"善后"**。编辑器子系统则用 `Initialize` / `Deinitialize` 配对。

### 支柱 4 — Instigator（发起者）记账
多个系统/触发体会同时请求同一全局资源，每条活动记录都带 `FInstigator`，按"谁请求谁撤销"管理，避免一个系统关掉另一个系统的效果：
```cpp
StartTimeDilationEffect(Actor, Effect, Instigator);
StopTimeDilationEffect (Actor,         Instigator);  // 仅撤销自己那条
```
Rendering 更进一步用 `TInstigated<T>` 模板容器，天然支持"多发起者叠加 + 各自 Empty"。

### 支柱 5 — 薄 Statics 门面（namespace + mixin）
单例本体**不直接暴露给调用方**，而是包一层 `namespace` / `mixin` 静态 API（见 `TimeDilationStatics.as`）：
```cpp
namespace TimeDilation {
    void StartWorldTimeDilationEffect(...){ ...::Get().Start...(); }
}
mixin void StartActorTimeDilationEffect(AHazeActor Actor, ...){
    ...::Get().Start...();
}
```
调用点写 `Actor.StartActorTimeDilationEffect(...)` 或 `TimeDilation::StartWorld...()`，可读、可 BlueprintCallable，且把 `::Get()` 细节隐藏。**"单例存数据/逻辑，Statics 当门面"是 Core 的固定搭配。**

---

## 三、按职责分成 4 类单例

骨架统一，但按"协调什么"分为四种典型：

### A. 效果聚合器（最常见）
持有 `TArray<F活动条目>`，每帧聚合成单一全局输出。
- `TimeDilationEffectSingleton` —— 多个慢动作请求 → 世界/每 actor 最终时间膨胀。
- `RenderingSettingsSingleton` —— 多发起者 → SSS/SSR/动态分辨率/视距等 CVar。
- `FadeSingleton` + `FadeManagerComponent` —— 加载屏与淡化的全局状态。

> 形态：**少量全局状态 + Tick 聚合 + Instigator 记账**。

### B. 性能注册表 / 调度器
组件向单例**注册自己**，单例集中调度以省性能。
- `DisableComponentSingleton` —— 所有 `UDisableComponent` 注册进来，单例每帧只 tick **一部分**：
  ```cpp
  int UpdateCount = Math::Max(Math::IntegerDivisionTrunc(DisableCompCount, 30), 15);
  UpdateIndex = (UpdateIndex + 1) % DisableCompCount;   // 环形轮转
  ```
  把"N 个组件各自每帧算距离"压成"单例每帧算 15~N/30 个"。

> 形态：**注册表 + 分帧轮转（time-slicing）**，把 O(N) 每帧成本摊薄的标准手法。

### C. 跨玩家/跨视图协调器
处理只有"全局视角"才能做的事——尤其是双人分屏。
- `CameraSingleton` —— 持有 `SplitBlend` / `FSBlend` 两个状态机，编排**分屏↔全屏**的相机与投影偏移混合（需同时操作 Mio 与 Zoe 两台相机，单个组件无法胜任）。
- `Vox` 双单例 —— `HazeVoxController`（守门：校验/匹配/判定 trigger/queue/block）+ `HazeVoxRunner`（播放：6 条车道的跨车道阻塞）。**职责分离**：一个决定"能不能播"，一个决定"怎么播"。

> 形态：**内嵌状态机 + 跨实体编排**。

### D. 领域管理器 / 注册表
引擎给特定领域提供的专用单例基类，脚本侧补具体策略。
- `AudioZoneManager`（混响/遮挡/环境声区合成）、`MeshDissolveManagerComponent`、`SequencerRenderSingleton`（弱指针注册龙 actor 供序列取用）、`PlayerAudioFilterDeathManager`。

### E.（旁支）编辑器子系统
`UHazeEditorSubsystem` 家族，仅编辑器存活，用 `Initialize/Deinitialize`、`OnEditorLevelsChanged`、`Tick`（画视口 overlay UI）。
- `PrefabEditorSubsystem`、`PropLineEditorSubsystem`、`VisibilityVolumeEditorSubsystem`。常配 `FConsoleCommand` 暴露控制台命令。

---

## 四、架构全景图

```
                 调用方（Capability / Component / Trigger / Level 脚本）
                        │  Actor.StartActorXxx(...)  /  Xxx::DoThing()
                        ▼
        ┌──────────────────────────────────────────────────┐
        │  Statics 门面 (namespace + mixin) —— 隐藏 ::Get()    │
        └──────────────────────────────────────────────────┘
                        │  Type::Get()
                        ▼
        ┌──────────────────────────────────────────────────┐
        │  System 单例 (UHazeSingleton 等)                    │
        │   · TArray<F活动条目> + FInstigator 记账             │
        │   · Tick(): 推进所有条目 → 聚合 → 应用全局输出         │
        │   · ResetStateBetweenLevels / Shutdown 善后          │
        │   分类: A效果聚合 B性能注册表 C跨玩家协调 D领域管理器   │
        └──────────────────────────────────────────────────┘
                        │  Time::Set... / Console::Set... / 操作两台相机...
                        ▼
                  引擎全局状态（世界时间膨胀 / CVar / 分屏视图 / Wwise）
```

---

## 五、核心结论

Core 的 System 层是一套**纪律极强的单例框架**，5 条契约缺一不可：

1. **`::Get()` 全局访问**，从不持有引用；
2. **集中 Tick + 聚合**，把"多来源意图"收敛为"唯一全局结果"（这是单例的存在理由）；
3. **`ResetStateBetweenLevels` / `Shutdown` 关卡边界善后**，因单例跨关卡存活；
4. **`FInstigator` / `TInstigated<T>` 记账**，保证多请求方互不踩踏；
5. **薄 Statics 门面**，对外只露 `namespace` / `mixin`，隐藏单例细节。

四种职责变体（效果聚合 / 性能注册表分帧 / 跨玩家协调状态机 / 领域管理器）都是这套骨架的具体应用。

> **判断一个 Core 文件是否"System 级"**：看它是否继承 `UHazeSingleton` / `UHaze*Subsystem` / `UHaze*Manager` 并实现 `Tick` + `ResetStateBetweenLevels`。

---

## 相关文档

- 各单例的具体功能见所属功能域文档：[Rendering.md](./Rendering.md)、[SequencerAndTime.md](./SequencerAndTime.md)（TimeDilation）、[Flow.md](./Flow.md)（Fades/Disable/Visibility）、[Camera.md](./Camera.md)、[VoxAndSubtitles.md](./VoxAndSubtitles.md)、[Audio.md](./Audio.md)、[Systems.md](./Systems.md)。
- 联机相关的 Crumb 概念见 [Systems.md](./Systems.md) 与 [README.md](./README.md)。

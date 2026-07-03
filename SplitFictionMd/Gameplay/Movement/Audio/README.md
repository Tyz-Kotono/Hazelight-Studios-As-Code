# Gameplay / Movement / Audio —— 移动音频

> 职责：玩家运动产生的**音频**——脚步、手部/杆/梯攀爬摩擦、滑行、划水、植被沙沙、手臂摆动。共 **25 文件**。核心手法是"**表面追踪(trace) → 查物理材质 → 播对应音效**"。
>
> 注意：本目录是 Movement 域内的**玩家运动音频**；引擎级音频系统本体在 [Core/Audio](../../../Core/Audio/README.md)，本目录的类多继承自 Core 的 `UHazeMovementAudioComponent` 等基类。

---

## 与 Core/Audio 的关系

| | Core/Audio | Movement/Audio（本目录） |
| --- | --- | --- |
| 定位 | 引擎级音频框架（发声器、监听器、物理材质音频资产、实例限制…） | 玩家**运动事件**触发音频的业务逻辑 |
| 基类 | 提供 `UHazeMovementAudioComponent`、`UPhysicalMaterialAudioAsset` 等 | 继承并特化为玩家脚步/滑行/划水 |
| 触发源 | 通用 | 运动 Capability + AnimNotify |

本目录**消费** Core 的物理材质音频体系：trace 打到地面 → 拿到 `UPhysicalMaterialAudioAsset` → 交给 Core 播放。

---

## 内部结构

### 发声组件与能力（脚步/滑行/植被/摆臂）

| 文件 | 作用 |
| --- | --- |
| `PlayerMovementAudioComponent.as` | **中枢**：继承 Core `UHazeMovementAudioComponent`；管理左右脚/手 trace 数据、滑行手状态、呼吸设置(mixin)、脚步与滑行 trace 事件（`OnFootTrace`/`OnFootSlideTrace`） |
| `PlayerFoliageAudioComponent.as` | 穿行植被的沙沙声 |
| `PlayerFootSlideAudioCapability.as` / `PlayerHandSlideAudioCapability.as` | 脚/手滑行摩擦音 |
| `Armswing/PlayerArmswingAudioCapability.as` + `PlayerArmswingMovementAudioSetting.as` | 手臂摆动音（跑动时衣物/摆臂） |
| `AudioDebugMovement.as` | 运动音频调试 |

### Trace —— 表面追踪子系统（`Audio/Trace/`）

运动音频的核心：向脚/手/杆/梯下方打射线，取表面物理材质决定播什么声。

| 文件 | 作用 |
| --- | --- |
| `FootstepTraceComponent.as` / `PlayerFootstepTraceComponent.as` | 脚步 trace 状态 |
| `PlayerFootstepTraceCapability.as` | 脚步 trace 能力（`Audio` tick 组，带时光日志调试扩展） |
| `PlayerAudioHandTraceCapability.as` / `PlayeAudioPoleTraceCapability.as` / `PlayerAudioLadderTraceCapability.as` | 手/爬杆/爬梯的表面 trace |
| `PlayerWallRunTransitionAudioTraceCapability.as` / `PlayerWallScrambleTransitionAudioTraceCapability.as` | 墙跑/攀墙进出的过渡音 trace |

### TraceSettings —— 按角色分的 trace 参数（`Audio/Trace/TraceSettings/`）

不同可玩角色/生物的脚/手 trace 参数（骨骼名、射线长度…）各成一个 Settings，继承 `AudioTraceSettings.as`：

`AudioPlayerFootTraceSettings` / `AudioPlayerHandTraceSettings`（人类玩家）、`AudioDragonFootTraceSettings`、`AudioFantasyOtterFootTraceSettings`、`AudioTundraMonkeyFootTraceSettings`、`AudioTundraTreeGuardianFootTraceSettings`（各变身/关卡生物形态）。

### Water —— 划水音频（`Audio/Water/`）

`AudioWaterMovementVolume.as`（+ Details 编辑器扩展 + EventHandler）：进入水体运动体积时触发划水/入水音效。

---

## 数据流

```
运动 Capability / AnimNotify（如落脚、手贴墙）
        │  触发 trace 请求
        ▼
Audio/Trace/*Capability  —— 向脚/手/杆/梯下方打射线（参数来自对应 TraceSettings）
        │  命中表面
        ▼
拿到 Core 的 UPhysicalMaterialAudioAsset（草地/金属/水…）
        │
        ▼
PlayerMovementAudioComponent 广播事件 → Core/Audio 播放对应音效
```

---

## 关键文件

| 文件 | 作用 |
| --- | --- |
| `PlayerMovementAudioComponent.as` | 运动音频中枢，管理脚/手 trace 数据与发声事件 |
| `Trace/PlayerFootstepTraceCapability.as` | 脚步 trace 能力（trace 范式典型） |
| `Trace/TraceSettings/AudioTraceSettings.as` | trace 参数基类（各角色形态派生） |
| `Water/AudioWaterMovementVolume.as` | 划水音频触发体积 |
</content>

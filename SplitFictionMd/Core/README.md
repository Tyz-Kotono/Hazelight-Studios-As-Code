# Core — 引擎核心框架（模块级文档）

> 路径：`SplitFiction/Core`（513 文件，58 个子模块）
>
> 通用底层系统，不绑定任何具体关卡。是整个游戏脚本的地基——上层的 `Gameplay`、`Animation`、`LevelSpecific` 都建立在 Core 提供的能力、组件、单例与静态库之上。
>
> **本目录采用"功能域 / 模块"两级结构**：每个功能域是一个文件夹（含 `README.md` 总览），其下每个子模块再各有一个文件夹（含 `README.md`，分析**模块内部架构**与**与其他模块的协同**）。

---

## 贯穿全模块的核心概念

理解以下几个概念，就理解了 Core 的运作方式：

### 1. Capability（能力）—— 行为单元
继承 `UHazeCapability` / `UHazePlayerCapability`，生命周期为 `Setup → ShouldActivate → OnActivated → TickActive → ShouldDeactivate → OnDeactivated`。通过 `CapabilityTags`（能力标签）互相阻塞、通过 `TickGroup` 控制每帧执行顺序。

### 2. Component（组件）—— 状态载体
继承 `UActorComponent` / `USceneComponent`，挂在 actor 上持有状态、提供服务。能力读写组件。

### 3. Singleton（单例）—— 全局协调者
继承 `UHazeSingleton` 等，全局唯一，负责跨 actor / 跨玩家调度。详见 [SystemArchitecture.md](./SystemArchitecture.md)。

### 4. Composable Settings（可组合设置）—— 分层参数
继承 `UHazeComposableSettings`，多来源按优先级 `Apply` 叠加，取最高优先级生效。

### 5. Crumb（面包屑）—— 确定性联机核心 ⭐
确定性分屏协作：每端各控一名玩家（**Mio** / **Zoe**）。Crumb 是带时间戳的命令，经有序 crumb trail 在两端同时间点回放。`HasControl()` 判控制端，`CrumbFunction` 有序传递。

### 6. Mio / Zoe —— 双主角
枚举 `EHazePlayer::Mio` / `EHazePlayer::Zoe`，大量系统用 `TPerPlayer<T>` 存每玩家数据。

---

## 横切分析文档（先读这几篇）

| 文档 | 主题 |
|---|---|
| [DesignPhilosophy.md](./DesignPhilosophy.md) ⭐ | **设计哲学（先读这篇）**：4 大支柱 + 确定性联机第一性原理 + 一帧全局数据流 |
| [SystemArchitecture.md](./SystemArchitecture.md) | **System 级骨架**：单例 5 大支柱与 4 类职责 |
| [Communication.md](./Communication.md) | **模块间协作机制**：5 类机制使用量统计与主次 |

---

## 模块索引（功能域 → 模块文件夹）

> 每个模块链接指向 `<功能域>/<模块>/README.md`，含内部架构 + 协同分析。

### 🎥 [Camera/](./Camera/) — 相机框架（82 文件）
[Camera](./Camera/Camera/) · [CameraShake](./Camera/CameraShake/) · [CameraShakeForceFeedback](./Camera/CameraShakeForceFeedback/) · [CameraBoundaryComponents](./Camera/CameraBoundaryComponents/)

### 🔊 [Audio/](./Audio/) — 运行时音频管线（99 文件）
[Audio](./Audio/Audio/)（单模块功能域，内部按 15 子目录分节）

### 🏃 [Movement/](./Movement/) — 移动框架（58 文件）
[Movement](./Movement/Movement/) · [MoveTo](./Movement/MoveTo/)

### 🧍 [PlayersAndHealth/](./PlayersAndHealth/) — 玩家与生命（64 文件）
[Players](./PlayersAndHealth/Players/) · [PlayerHealth](./PlayersAndHealth/PlayerHealth/) · [Offset](./PlayersAndHealth/Offset/) · [Team](./PlayersAndHealth/Team/)

### 🗣️ [VoxAndSubtitles/](./VoxAndSubtitles/) — 语音与字幕（20 文件）
[Vox](./VoxAndSubtitles/Vox/) · [Subtitles](./VoxAndSubtitles/Subtitles/) · [Accessibility](./VoxAndSubtitles/Accessibility/)

### 🎯 [InteractionAndAiming/](./InteractionAndAiming/) — 交互与瞄准（30 文件）
[Targetable](./InteractionAndAiming/Targetable/) · [Interaction](./InteractionAndAiming/Interaction/) · [Aiming](./InteractionAndAiming/Aiming/)

### 📐 [SplineAndSpace/](./SplineAndSpace/) — 样条与空间（27 文件）
[Spline](./SplineAndSpace/Spline/) · [Collision](./SplineAndSpace/Collision/) · [Navigation](./SplineAndSpace/Navigation/) · [Volumes](./SplineAndSpace/Volumes/) · [Attachment](./SplineAndSpace/Attachment/) · [Shapes](./SplineAndSpace/Shapes/) · [Sorting](./SplineAndSpace/Sorting/)

### 🚦 [Flow/](./Flow/) — 流程控制（31 文件）
[Triggers](./Flow/Triggers/) · [Tutorial](./Flow/Tutorial/) · [RespawnPoint](./Flow/RespawnPoint/) · [Progress](./Flow/Progress/) · [Disable](./Flow/Disable/) · [Fades](./Flow/Fades/) · [Visibility](./Flow/Visibility/)

### 🎬 [SequencerAndTime/](./SequencerAndTime/) — 序列与时间（20 文件）
[Sequencer](./SequencerAndTime/Sequencer/) · [TimeDilation](./SequencerAndTime/TimeDilation/) · [Time](./SequencerAndTime/Time/) · [Bink](./SequencerAndTime/Bink/)

### 🎮 [MiniGames/](./MiniGames/) — 小游戏与体感（20 文件）
[ButtonMash](./MiniGames/ButtonMash/) · [StickSpin](./MiniGames/StickSpin/) · [StickWiggle](./MiniGames/StickWiggle/) · [ForceFeedback](./MiniGames/ForceFeedback/)

### 🖼️ [Rendering/](./Rendering/) — 渲染与视图（26 文件）
[PostProcess](./Rendering/PostProcess/) · [Rendering](./Rendering/Rendering/) · [Sky](./Rendering/Sky/) · [Lights](./Rendering/Lights/) · [Decal](./Rendering/Decal/) · [PaintablePlane](./Rendering/PaintablePlane/) · [Wardrobe](./Rendering/Wardrobe/) · [SceneView](./Rendering/SceneView/)

### 🔢 [MathAndUtilities/](./MathAndUtilities/) — 数学与工具（8 文件）
[Math](./MathAndUtilities/Math/) · [Curves](./MathAndUtilities/Curves/) · [FloatCurves](./MathAndUtilities/FloatCurves/) · [Visualization](./MathAndUtilities/Visualization/)

### ⚙️ [Systems/](./Systems/) — 系统设施（28 文件）
[Network](./Systems/Network/) · [Prefab](./Systems/Prefab/) · [Settings](./Systems/Settings/) · [WidgetPool](./Systems/WidgetPool/) · [WorldLink](./Systems/WorldLink/) · [Props](./Systems/Props/)

---

## 每个模块文档的统一结构

每个 `<模块>/README.md` 遵循以下模板：

1. **职责** —— 一句话定位 + 文件数。
2. **内部架构** —— 关键类、基类、核心数据结构、内部数据流。
3. **协同关系**（重点）—— 表格列出：
   - *它调用谁*（依赖的其他模块/组件）
   - *谁调用它*（被哪些模块消费）
   - 协作走哪种机制（直接调用 / Crumb / 标签阻塞 / 委托 / 设置注入，见 [Communication.md](./Communication.md)）
4. **关键文件** —— 逐文件一句话。

---

## 架构总结

Core 是一套高度统一的框架：**行为**用 Capability、**状态**用 Component、**全局协调**用 Singleton、**联机**用 Crumb、**双人**用 Mio/Zoe、**参数**用 Composable Settings、**工具**用 `Xxx::` 静态库。58 个子模块都是这套概念的具体应用。

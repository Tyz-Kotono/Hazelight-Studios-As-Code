# Core / Audio / SoundDefObjects

> 职责：定义可挂载到 SoundDef 运行时对象上的"行为模块"，由 SoundDef 每帧驱动以动态调制声音（音量包络、层交叉淡变、多普勒音高、经过音事件）。共 4 个文件。

## 内部架构（关键类、基类、数据流）

本目录的四个 `.as` 文件目前均**整体处于注释状态**——它们是 AngelScript 原型，真正生效的实现位于 C++/蓝图侧。每个脚本类通过 `UFUNCTION(BlueprintOverride)` 重写 C++ 基类暴露的回调，因此可以从被注释的类声明与回调签名准确还原出架构。

**统一基类模式**：四个行为类各自继承一个同名的 C++ 基类，构成一一对应的"脚本子类 ← C++ 基类"关系：

| 行为类 | C++ 基类 | 作用 |
| --- | --- | --- |
| `USoundDefAHREnvelope` | `USoundDefAHREnvelopeObject` | Attack-Hold-Release 音量包络 |
| `USoundDefCrossfade` | `USoundDefCrossfadeObject` | 多层声音按位置交叉淡变 |
| `USoundDefDoppler` | `USoundDefDopplerObject` | 按相对速度计算多普勒值 |
| `USoundDefPassby` | `USoundDefPassbyObject` | 物体掠过监听器时触发经过音 |

这些 `...Object` 基类即任务背景中所述的 SoundDef 行为对象基类家族：它们作为可组合模块被挂在顶层 SoundDef 运行时实例上，由 SoundDef 在生命周期中调用其回调，并把计算结果（包络 Alpha、层音量、多普勒值、经过音参数）回写给声音引擎（RTPC / 层音量 / 事件参数），从而实现"挂在 SoundDef 上调制声音"的核心模式。

**被 SoundDef 调用的回调（初始化 / 每帧）与数据流**：

- **AHR 包络** —— 由外部驱动 `BP_Start(bStartFromZero, bIsLooping)` / `BP_Stop()` 控制状态机进入 Attack/Hold/Release，每帧调用 `Evaluate(DeltaSeconds, FAHREnvelopeEvaluationData)`：按当前阶段与曲线（`EaseIn`/`EaseOut` + 曲线指数）求出 `EnvelopeAlphaValue`（0~1 音量系数），阶段结束时 `AdvanceToNextStage()` 推进；阶段切换/结束广播 `ExecuteOnTickActive`、`EnteredLoopingHold`、`ExecuteOnReleaseFinished`。输出：归一化音量 Alpha。
- **交叉淡变** —— 每帧调用 `EvaluateLayers(InPosition)`：按位置（如样条/参数轴）遍历 `Layers`，`CalculateLayerAlpha` 计算每层 Alpha（与相邻层重叠时按 `FadeIn/FadeOutCurvePower` 互相淡入淡出，并处理全局首尾 FadeIn/FadeOut），通过 `SetLayerAlpha(LayerId, Alpha)` 回写层音量；进出层时广播 `TriggerOnLayerEnter` / `TriggerOnLayerExit`。输出：各层音量 Alpha。
- **多普勒** —— 初始化 `SetupDopplerObject()` 记录各玩家初始位置；每帧 `CalculateDopplerValue(DeltaSeconds)`：取观察目标（`EHazeDopplerObserverTargetType`：双监听器/双玩家/Mio/Zoe），用 `DopplerAudioComp` 与目标的距离变化求相对速度，映射到 `[-1,1]` 并按归一化距离的幂次缩放，得到 `CurrentDopplerValue`（驱动音高的多普勒系数），再 `UpdateTrackedPositions()` 缓存位置。输出：多普勒值（音高调制）。
- **经过音** —— 初始化 `SetupPassbyObject()`；每帧 `QueryPassby(DeltaSeconds)`：用相对速度、朝向点积（`MinVelocityAngle`）、距离与冷却（`CooldownTime`）判定目标是否正掠过 `PassbyAudioComp`，命中即填充 `FPassbyResult`（观察者角度/距离/归一化距离/相对速度等）并 `TriggerPassby` 广播一次性事件，进入冷却。输出：经过音触发事件 + 参数。

## 协同关系

> 实证说明：这些行为类在本仓 AngelScript 侧仅出现在各自文件内且整体被注释（`Grep "SoundDef(AHREnvelope|Crossfade|Doppler|Passby)"` 仅命中本目录 4 文件），真正实现与挂载在 C++/蓝图侧；下表方向以注释代码中的实际调用为据。仓内不存在 `Audio/SoundDefinitions/` AngelScript 目录（内容定义在引擎/资产侧）。

| 方向 | 对象 | 机制 | 说明（基于实证） |
| --- | --- | --- | --- |
| SoundDef → 本目录 | `USoundDef<X>Object` 各行为类 | 4 委托 event/Broadcast | SoundDef 通过 `BlueprintOverride` 回调（`Evaluate`/`EvaluateLayers`/`CalculateDopplerValue`/`QueryPassby` 等）每帧驱动行为对象，是 C++ → 脚本的虚回调分发 |
| 本目录 → SoundDef | 交叉淡变 / 包络 | 5 可组合设置 ApplySettings | 行为类把计算结果回写给宿主：`SetLayerAlpha(LayerId, Alpha)` 设层音量、包络输出 `EnvelopeAlphaValue` 供 SoundDef 调制音量 |
| 本目录 → 玩家 | 多普勒 / 经过音 | 1 直接调用 Type::Get | 通过 `Game::GetPlayers()` / `Game::GetMio()` / `Game::GetZoe()` 取玩家与监听器位置，计算相对速度（`DopplerAudioComp` / `PassbyAudioComp` 为宿主发射器组件） |
| 本目录 → 监听 / 表现 | 经过音 / 交叉淡变 | 4 委托 event/Broadcast | 命中时广播：经过音 `TriggerPassby(FPassbyResult)`、交叉淡变 `TriggerOnLayerEnter/Exit`、包络 `ExecuteOnReleaseFinished` 等，供上层响应 |
| 触发体 → SoundDef | `APassbyishTriggerVolume` | 1 直接调用 Type::Get | `PlayOnEnter` 中 `SoundDefRef.SpawnSoundDefOneshot(...)` 生成可 Tick 的 SoundDef 一次性实例，间接使用经过音类行为（同 `EHazeDopplerObserverTargetType` 枚举） |

## 关键文件（逐文件一句话）

- **SoundDefAHREnvelope.as** —— Attack-Hold-Release 状态机包络，按曲线逐帧求 `EnvelopeAlphaValue` 调制音量，支持循环 Hold 与可中断的起停。
- **SoundDefCrossfade.as** —— 多层声音按位置参数交叉淡变，逐层计算并 `SetLayerAlpha` 回写层音量，处理相邻层重叠与全局首尾淡变。
- **SoundDefDoppler.as** —— 跟踪玩家/监听器位置，按与发射器的相对速度和距离幂次算出 `CurrentDopplerValue` 用于音高多普勒调制。
- **SoundDefPassby.as** —— 用相对速度、朝向点积、距离与冷却判定物体掠过监听器，命中即填充 `FPassbyResult` 并 `TriggerPassby` 触发一次性经过音。

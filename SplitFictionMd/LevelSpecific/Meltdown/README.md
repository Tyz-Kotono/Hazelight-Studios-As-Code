# LevelSpecific / Meltdown（故障熔毁·终章）

> 路径：`LevelSpecific/Meltdown/`（545 文件，科幻 / Mio 线，终章）
>
> Meltdown 是游戏的收尾关卡：科幻世界与奇幻世界的边界开始"故障熔毁"，两条叙事线在同一块屏幕上撞在一起。它的招牌不是某个新载具或新武器，而是**对分屏这一 Split Fiction 招牌呈现方式本身的解构**——整关围绕"打破分屏"设计了一整组机制：把两个玩家的画面斜切、融合、翻转、互穿。玩法上还有一把贯穿全关的**故障枪**（把奇幻/科幻物件"格式化"）与一场**三阶段 Boss 战**。本关不发明底层范式，是 Core 的 `SceneView::` 相机/渲染 API 与 Gameplay 能力四件套在"元层面（画面本身即玩法）"上的集中运用。上层设计见 [Core 设计哲学](../../Core/DesignPhilosophy.md)、[Gameplay 设计哲学](../../Gameplay/DesignPhilosophy.md)。

---

## 目录构成

| 类别 | 目录 | 说明 |
|---|---|---|
| **通用基础设施** | `Editor/`、`Transition/`、`Ending/`、`Audio/`(散落各机制)、`PrototypeCubes/` | 编辑器子系统、段落转场、结局编排、音频挂接。沿用 Core/Gameplay 框架，**本文档一笔带过** |
| **打破分屏机制群** ⭐ | `SplitTraversal/`(99)、`ScreenWalk/`(67)、`SoftSplit/`(43)、`WorldSpin/`(26)、`SplitSlide/`(20)、`ScreenPush/`(2)、`SplitBonanza/`(12)、`FakeSplitRendering/`(1)、`WorldLink/`(2) | 本关的绝对核心：斜切软分屏、跨屏穿越、屏中屏行走、世界旋转、高速滑行 |
| **故障武器** ⭐ | `GlitchShooting/`(23)、`GlitchBazooka/`(6)、`GlitchBeam/`(4)、`GlitchSword/`(6) | 故障枪/火箭筒/光束/剑，把物件"格式化"，也是打 Boss 的手段 |
| **三阶段 Boss** ⭐ | `BossBattle/`(64)、`BossPhaseOne/`(10)、`BossPhaseTwo/`(27)、`BossPhaseThree/`(106) | 共享 `AMeltdownBoss` 的三阶段终极战 |
| **场景段落** | `Underwater/`(17)、`RopeHang/`(4) | 水下段、绳索悬挂段等一次性桥段 |

---

## 招牌机制分篇

| 机制 | 文件数 | 一句话 | 文档 |
|---|---:|---|---|
| [打破分屏机制群 SplitMechanics](./SplitMechanics/) | 255 | 软分屏斜切融合 / 跨屏穿越 / 屏中屏行走 / 世界旋转 / 高速滑行——把分屏本身变成玩法 | [SplitMechanics/](./SplitMechanics/) |
| [三阶段 Boss](./Boss/) | 207 | 共享 `AMeltdownBoss`，一阶段竞技场近战 → 二阶段多世界弹幕 → 三阶段传送门召唤+飞行/跳伞多模态 | [Boss/](./Boss/) |
| [故障射击 GlitchShooting](./GlitchShooting/) | 39 | 把奇幻/科幻物件"格式化"的故障武器四件套（枪/火箭筒/光束/剑），兼作 Boss 输出手段 | [GlitchShooting/](./GlitchShooting/) |

---

## Meltdown 通用范式（读一处懂全关）

1. **两个世界并存于一套坐标，用巨型偏移隔开**
   打破分屏的所有机制都基于同一招：科幻世界与奇幻世界在世界坐标里被一个巨大的常量偏移分开（`SoftSplit` 用 `SoftSplitOffset(200000,0,0)`、`SplitTraversal` 用 `SplitOffset(500000,0,0)`），各机制的 Manager 都提供 `Position_FantasyToScifi` / `Position_ScifiToFantasy` / `Position_Convert` 一组坐标换算，把"另一个世界"的物体/相机搬到本世界渲染或参与碰撞。`EHazeWorldLinkLevel`（Fantasy / SciFi）是贯穿全关的世界标签。

2. **不改渲染，只编排 Core 的 `SceneView::`**
   机制自身不写渲染管线，而是驱动 Core 的分屏/相机 API：`SceneView::SetSplitScreenMode(Vertical / ManualViews / CustomMerge)`、`SetManualViews`、`ComputeView`→`FHazeComputedView`、世界↔屏幕投影（`ProjectWorldToScreenPosition` / `DeprojectScreenToWorld_*`）。软分屏走 Core 的 `ACustomMergeSplitManager` 基类 + 后处理融合材质；屏中屏走 `USceneCaptureComponent2D` + `UTextureRenderTarget2D`。详见 [SplitMechanics](./SplitMechanics/)。

3. **每个 Manager 是一个 `AHazeListedActorComponent` 单例**
   打破分屏的每套机制由一个场景中的 Manager Actor 主导（`ASoftSplitManager`、`ASplitTraversalManager`、`AMeltdownScreenWalkManager`、`AMeltdownWorldSpinManager`…），挂 `UHazeListedActorComponent` 供 `TListedActors<T>().GetSingle()` 全局取用，用 `UHazeRequestCapabilityOnPlayerComponent` 向玩家注入本段所需能力。

4. **玩家分身（PlayerCopy）**
   当两个世界合并到一块屏幕，需要在"对方世界"里显示另一名玩家的影子——`ASoftSplitPlayerCopy` / `ASplitTraversalPlayerCopy`（配 `MeshRenderingComponent`）在另一侧世界渲染玩家副本，坐标由 Manager 的换算函数每帧同步。

5. **能力四件套 + Crumb 联机**
   故障武器、Boss 攻击、跨屏移动仍是标准 Capability + Component + Settings(+EffectHandler) 四件套，`NetworkMode = Crumb`，与 Gameplay 一致。

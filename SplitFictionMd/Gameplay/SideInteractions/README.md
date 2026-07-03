# Gameplay / SideInteractions 功能域总览

> 职责：**可选支线小游戏合集**——三个互不相干的趣味场景装置：兄弟长椅、打水漂、泰坦尼克摆姿势。共 **24 文件**，无根级文件，每个小游戏一个独立子目录。
>
> **前提**：本文承接 [Gameplay 设计哲学](../DesignPhilosophy.md)。这里不重复哲学。

---

## 一句话定位

> **SideInteractions 没有共同框架，是"用通用积木拼出的小游戏目录"。** 三者不共享基类、不互相引用；唯一共性是都从同一批通用零件搭起：`AnimationInteractions` 的交互组件、Core 交互能力、`FauxPhysics`、能力/相机系统、样条、Crumb。代码量与玩法交互度成正比（Titanic 摆姿势 2 文件 → Bench 坐+相机 7 文件 → SkippingStones 完整投掷小游戏 15 文件）。

---

## 目录结构（24 文件）

| 子目录 | 文件数 | 玩法 |
| --- | --- | --- |
| `BrothersBench` | 7 | 两人坐上长椅两端，触发对话与景观相机 |
| `SkippingStones` | 15 | 捡石头、瞄准、蓄力、投掷，让石头在水面打水漂 |
| `TitanicBoat` | 2 | 泰坦尼克船头"我在飞"双人摆姿势（船随重心摇晃） |

---

## 三个小游戏

### BrothersBench（长椅，7 文件）

- **玩法**：两人各坐一端，双方坐下后按友谊等级播对话、淡入音频总线；长椅相机滑向就座玩家，延时后混切到景观相机。六张椅（`EBrothersBenchID`：Skyline/Tundra/Island/Summit/Prison/Sanctuary）。
- **关键类**：`ABrothersBench : AHazeActor`。两个座位直接是 `UThreeShotInteractionComponent LeftInteraction/RightInteraction`（复用 `AnimationInteractions`）。坐下/起身经组件委托 `OnEnterBlendedIn → OnSitDown`、`OnCancelPressed → OnGetUp`，无自定义检测。相机用 `UHazeCameraComponent`+`UCameraOrbitBlend`。玩家侧 `UHazePlayerCapability` 读 `ActiveInteraction.Owner` 判定是否长椅；无物理、无 Pickup，能力 `NetworkMode=Crumb`。

### SkippingStones（打水漂，15 文件）

- **玩法**：进石堆交互 → 捡石 → 瞄准（相机夹角+兴趣点）→ 蓄力 → 投掷；石头自定义弹道飞行、撞水面弹跳计数、可溅水/超时/砸到对方。子目录分 `Interaction/`、`Stone/`、`System/Capabilities/`。
- **关键类**：`ASkippingStones : AHazeActor`（石堆站，直接用 **Core** `UInteractionComponent`，非 ThreeShot）；`ASkippingStone : AHazeActor`（投出的石头，**手写弹道**：`Acceleration::ApplyAccelerationToVelocity`+阻力+逐次 trace 解算+`Bounce()` 恢复系数，**非** FauxPhysics）；`ASkippingStonesWaterSpline : ASplineActor`（定义水域）。
- **构建**：入口能力 `USkippingStonesCapability : UInteractionCapability`；每人状态机 `ESkippingStonesState{ Pickup, Aim, Throw }` 存于 `USkippingStonesPlayerComponent`，各阶段一个 `UHazePlayerCapability`。石头**手动联网生成**（`SpawnActor`+`MakeNetworked`+`SetActorControlSide`，非共享 Pickup 系统），关键动作走 `CrumbFunction`（`CrumbHitPlayer/CrumbPickupStone/CrumbThrowStone`）。调参集中在 `SkippingStonesSettings.as`（`SkippingStones::` 命名空间）。

### TitanicBoat（摆姿势，2 文件）

- **玩法**：船头双人摆"我在飞"姿势——Rose 先就位，Jack 位才启用；船体随两人重心物理倾斜摇晃。
- **关键类**（两个独立 Actor）：`ATitanic : AHazeActor` 组合 `UThreeShotInteractionComponent RoseInteractionComp` + `UOneShotInteractionComponent JackInteractionComp`（Jack 位开局 `Disable`，Rose 开始才 `Enable`）；`ATitanicBoat : AHazeActor` 纯 FauxPhysics，无逻辑：`UFauxPhysicsConeRotateComponent RotateComp` + `UFauxPhysicsPlayerWeightComponent PlayerWeightComp`。最声明式的一个。

---

## 协同关系（跨模块复用）

| 通用积木 | 使用者 | 说明 |
| --- | --- | --- |
| `UThreeShotInteractionComponent`（`AnimationInteractions`） | Bench（左右座）、Titanic（Rose） | 三段式动画交互 |
| `UOneShotInteractionComponent`（`AnimationInteractions`） | Titanic（Jack） | 一次性姿势 |
| `UInteractionComponent` / `UInteractionCapability`（Core） | SkippingStones（石堆站 + `USkippingStonesCapability`） | 直接用 Core 交互 |
| `UFauxPhysics*`（`FauxPhysics`） | Titanic（锥旋转 + 玩家体重） | 船体摇晃 |
| Core Crumb / `MakeNetworked` | SkippingStones（`CrumbFunction`）、Bench/SkippingStones 能力（`NetworkMode=Crumb`） | 联机一致；Titanic 无联机代码 |
| `ASplineActor.Spline` | SkippingStones（水域 `TListedActors`） | 判定水面（非 `UHazeSplineComponent` 类型名） |
| Pickup | SkippingStones **自带** bespoke 捡石（非共享 `Gameplay/Pickups`） | 命名巧合，无复用 |

> 三者互不引用、无共享基类/根文件。都是 `AHazeActor` 派生的 `UCLASS(Abstract)`（供蓝图派生内容），普遍用 `EventHandler` 类解耦音频/VFX，交互经委托（`OnInteractionStarted/Stopped`、`OnEnterBlendedIn`、`OnCancelPressed`）连接。

---

## 关键文件

- `Gameplay/SideInteractions/BrothersBench/BrothersBench.as` —— 长椅本体（双座 ThreeShot + 相机混切）。
- `Gameplay/SideInteractions/SkippingStones/Stone/SkippingStone.as` —— 手写弹道 + 水面弹跳的石头。
- `Gameplay/SideInteractions/SkippingStones/System/Capabilities/SkippingStonesCapability.as` —— 打水漂入口能力（`UInteractionCapability`）。
- `Gameplay/SideInteractions/TitanicBoat/TitanicBoat.as` —— 纯 FauxPhysics 摇晃船（锥旋转 + 玩家体重）。
- `Gameplay/SideInteractions/TitanicBoat/Titanic.as` —— ThreeShot(Rose)+OneShot(Jack) 姿势交互与门控。

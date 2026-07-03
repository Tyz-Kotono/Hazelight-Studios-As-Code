# Gameplay / AnimationInteractions 功能域总览

> 职责：**动画式交互物的通用套件**——玩家走近后播一段过场动画的可放置对象，分一次性、三段式、双人协同三种。共 **11 文件**，是关卡里"坐下/开门/摆姿势"等交互的复用积木。
>
> **前提**：本文承接 [Gameplay 设计哲学](../DesignPhilosophy.md)与 **Core / Interaction**（`UInteractionComponent`、`UInteractionCapability`）。这里不重复哲学。

---

## 一句话定位

> **AnimationInteractions 把 Core 交互框架封成三种"开箱即用"的动画交互模板。** 玩家进交互后被吸附到位、播 slot 动画；子类只需覆写 `SupportsInteraction` 认领自己的组件类型（`ShouldActivate` 在 Core 基类里是 `final`）。三种模板对应三种动画节奏：一次性播完即退、三段式可控退出、双人协同完成。

---

## 文件清单（11 文件）

每套遵循同一模式：**Actor（可放置对象）+ Component（数据/事件，继承 `UInteractionComponent`）+ Capability（玩家侧逻辑，继承 `UInteractionCapability`）+ EffectEventHandler（蓝图特效桥，`UHazeEffectEventHandler` Abstract）**。

| 套件 | 文件 | 关键类 |
| --- | --- | --- |
| One-shot（一次性） | `OneShotInteractionActor/Component/Capability/EffectEventHandler.as` | `AOneShotInteractionActor`、`UOneShotInteractionComponent`、`UOneShotInteractionCapability` |
| Three-shot（三段式） | `ThreeShotInteractionActor/Component/Capability/EffectEventHandler.as` | `AThreeShotInteractionActor`、`UThreeShotInteractionComponent`、`UThreeShotInteractionCapability` |
| Double（双人） | `DoubleInteractionActor/Capability/EffectEventHandler.as` | `ADoubleInteractionActor`、`UDoubleInteractionCapability` |

（Double 把动画阶段拆成多个 Capability，数据结构内联在 Actor 文件，故无独立 Component 文件。）

---

## 内部架构

### Core 基础（复用点）

- `UInteractionCapability : UHazePlayerCapability`（Core）：默认 tag `Interaction`/`BlockedByCutscene`/`BlockedWhileDead`，`NetworkMode = Crumb`。`ShouldActivate` 是 **`final`**（当有 `ActiveInteraction` 且 `SupportsInteraction` 为真才激活）——所以三套模板都只覆写 `SupportsInteraction`。`LeaveInteraction()` → `KickPlayerOutOfInteraction`。
- `UInteractionComponent : UTargetableComponent`（Core）：三套的 Component 继承它，只改 `bPlayerCanCancelInteraction`、`MovementSettings = FMoveToParams::SmoothTeleport()`、`InteractionSheet`。事件 `OnInteractionStarted/Stopped`、`CanPlayerCancel`/`SetPlayerIsAbleToCancel` 被子类直接用。

### One-shot：播完即自动退出

`UOneShotInteractionComponent` 默认 `bPlayerCanCancelInteraction = false`，`TPerPlayer<FOneShotSettings> OneShotSettings`（每人一段 `UAnimSequence` + `BlendTime/BlendType` + 音频）。能力 `OnActivated`：算目标点 `FMoveToDestination::CalculateDestination` → attach 玩家 → `Mesh.PlaySlotAnimation(...)`（带 `OnAnimationBlendedIn`/`OnAnimationBlendingOut` 委托）。动画混出 → 解 attach → `LeaveInteraction()` → 落地 `SnapToGround`。**单段动画、不可取消**。

### Three-shot：进入/循环/退出状态机

`UThreeShotInteractionComponent` 默认 `bPlayerCanCancelInteraction = true`，`FThreeShotSettings` 含 `EnterAnimation`/`MHAnimation`（循环保持段）/`ExitAnimation`。能力用状态机 `EThreeShotState{ Initial, EnterBlendedIn/Out, MHBlendedIn/Out, ExitBlendedIn/Out, Finished }`，`AdvanceToState` 单调递进并在每个边界同时 `Trigger_*` 特效 + `Broadcast` 委托。流程：进交互 block `n"InteractionCancel"`（自己接管取消以播完退出动画）→ Enter 动画 → MH 循环（此时才 `SetPlayerIsAbleToCancel(true)`）→ 玩家按 Cancel（`CrumbCancelThreeShot`）→ Exit 动画 → `LeaveInteraction`。

### Double：双人协同完成

`ADoubleInteractionActor` 需双人同时交互，含 `UNetworkLockComponent CompletionLock` 防 desync，`EDoubleInteractionExclusiveMode`（Left/Right 归 Mio/Zoe）。动画阶段拆成 4 个 Capability（Enter/MH/Cancel/Completed，均 `NetworkMode=Crumb`），用 `FDoubleInteractionState.PlayerState[Player]` 的 bool 位推进；完成走 `CrumbLockInDoubleInteract`/`CrumbCompleteDoubleInteract`。

### 一次性 vs 三段式对比

| | One-shot | Three-shot |
| --- | --- | --- |
| 可取消 | 否 | 是（自行接管 Cancel 保证退出动画完整） |
| 动画 | 单段，播完自动退 | 进入 → 循环保持 → 退出三段 |
| 典型用途 | 梯子、拉杆 | 坐下/起身、长时停留交互 |

---

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 依赖 → | Core `UInteractionCapability` | 覆写 `SupportsInteraction`（`ShouldActivate` 为 final）、`LeaveInteraction` | 认领机制 + Crumb 网络 |
| 依赖 → | Core `UInteractionComponent` | `InteractionSheet`（`AddCapability` + `Blocks Movement/GameplayAction`）、`OnInteractionStarted/Stopped`、`CanPlayerCancel` | sheet 注入能力、取消 API |
| 依赖 → | Core 动画/移动 | `PlaySlotAnimation` + `FHazeAnimationDelegate`、`AttachToComponent`、`SnapToGround`、`FMoveToDestination` | 统一走 slot 动画 + 吸附 |
| 依赖 → | Core Crumb | `CrumbFunction`（ThreeShot 取消、Double 锁定/完成）、`UNetworkLockComponent` | 关键状态双端一致 |
| 被使用 ← | `Gameplay/Movement/…/Ladder/Actors/Ladder.as` | 组合 `UOneShotInteractionComponent` | 梯子交互 |
| 被使用 ← | `Gameplay/SideInteractions/BrothersBench/BrothersBench.as` | 组合 `UThreeShotInteractionComponent`（左右座坐下/起身） | 支线长椅 |
| 被使用 ← | `Gameplay/SideInteractions/TitanicBoat/Titanic.as` | Rose 用 ThreeShot、Jack 用 OneShot | 泰坦尼克摆姿势 |
| 被使用 ← | `Gameplay/SideGlitch/SideGlitchInteractionActor.as` | 双 `UThreeShotInteractionComponent`（Mio/Zoe） | 支线故障入口 |

> 注意：`AOneShotInteractionActor`/`ADoubleInteractionActor` 这些 Actor 类型多是设计师直接放置/蓝图派生使用；被跨模块复用的主要是它们的 **Component 类型**。关卡里许多同名 `*DoubleInteraction*` 是各自独立实现（直接继承 Core 交互），并非复用本目录的 `ADoubleInteractionActor`。

---

## 关键文件

- `Gameplay/AnimationInteractions/OneShotInteractionCapability.as` —— 一次性动画交互（播完自动退）。
- `Gameplay/AnimationInteractions/ThreeShotInteractionCapability.as` —— 三段式状态机（进入/循环/退出，`AdvanceToState`）。
- `Gameplay/AnimationInteractions/ThreeShotInteractionComponent.as` —— 被 SideInteractions/SideGlitch 大量复用的三段式组件。
- `Gameplay/AnimationInteractions/DoubleInteractionActor.as` —— 双人协同交互（`UNetworkLockComponent` + 4 能力）。

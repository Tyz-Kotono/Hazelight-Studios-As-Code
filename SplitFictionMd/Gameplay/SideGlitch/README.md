# Gameplay / SideGlitch 功能域总览

> 职责：**横版故障支线入口**——双人协力按住按键，触发"故障/传送门"进入一段横版支线小游戏（side story）。共 **4 文件**。
>
> **前提**：本文承接 [Gameplay 设计哲学](../DesignPhilosophy.md)与 **Core / Interaction**、Progress（进度点）系统。这里不重复哲学。

---

## 一句话定位

> **SideGlitch 是"双人 button-hold → 淡白屏 → 激活进度点进入横版支线"的入口装置，自身不含相机/传送/横版切换实现。** 交互层完全复用 `AnimationInteractions` 的 `UThreeShotInteractionComponent`；SideGlitch 只做三件事：进度聚合、网络触发、特效派发。真正的横版转场交给 Haze 的 ProgressPoint 系统与关卡脚本。

---

## 文件清单（4 文件）

| 文件 | 关键类 | 基类 | 作用 |
| --- | --- | --- | --- |
| `SideGlitchInteractionActor.as` | `ASideGlitchInteractionActor` | `AHazeActor` | 主逻辑：双人 button-hold 入口 |
| `SideGlitchInteractionEffectHandler.as` | `USideGlitchInteractionEffectHandler` | `UHazeEffectEventHandler`（Abstract） | 5 个蓝图特效事件桥 + 参数结构 |
| `SideGlichGiants.as` | `ASideGlichGiants` | `AHazeActor`（Abstract） | 空占位基类（注意拼写 "Glich"） |
| `SidePortalInteractionActor_Exit.as` | `ASidePortalInteractionActor_Exit` | `AHazeActor`（Abstract） | 出口传送门空占位基类 |

顶层另定义 `event FOnSideGlitchEntered` 与 `enum ESideGlitchState{ None, Aborted, Completed }`。

---

## 内部架构

### `ASideGlitchInteractionActor` 流程

1. 挂两个按角色区分的交互组件 `MioInteraction`/`ZoeInteraction`（均 `UThreeShotInteractionComponent`，`UsableByPlayers = Mio/Zoe`，`bShowForOtherPlayer=true`）。
2. 玩家进交互（`OnEnterBlendedIn → OnPlayerEntered`）后启动**按住式 button mash**（`EButtonMashMode::ButtonHold`，`Duration=1.0`），并 `SetButtonMashAllowCompletion(this, false)` 由本 Actor 手动判完成，UI 挂在 `Mio/ZoeButtonMashAttachComp`。
3. `Tick` 读双人 `GetButtonMashProgress`，用二阶贝塞尔 `BezierCurve::GetLocation_2CP` 映射到 blend space 驱动"用力"动画。
4. **当 Mio 与 Zoe 进度都 ≥0.95 且 `HasControl()`** → `NetTriggerSidePortal()`（需双人协作）。
5. 触发后播 `Trigger_OnPlayersEnteringSideStory`；若 `OnSideGlitchEntered` 被绑则广播（交关卡脚本），否则默认 `bFadeToWhiteWhenEntering` 淡白屏 → 0.5s 后 `ActivateSideStory()`。
6. `ActivateSideStory()` 经**进度点**加载支线：`Progress::PrepareProgressPoint` + `Progress::ActivateProgressPoint(ID, false)`（ID 由 `FHazeProgressPointRef SideGlitchProgressPoint` 配置）。
7. 退出由外部调 `OnSideStoryAborted()`（放弃 → `Aborted`）或 `OnSideStoryCompleted()`（完成 → `Completed` 并 `AddActorDisable(this)` 防重入）。`SideGlitchState` 标注供配音（VO）使用。

---

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 依赖 → | `AnimationInteractions` `UThreeShotInteractionComponent` | 复用其 `OnEnterBlendedIn/BlendingOut/ExitBlendedIn` 事件 | 交互层不自造 |
| 依赖 → | Core Progress | `Progress::PrepareProgressPoint` / `ActivateProgressPoint` | 横版转场交给进度点系统 |
| 依赖 → | Core button mash | `EButtonMashMode::ButtonHold`、`GetButtonMashProgress`、`SetButtonMashAllowCompletion` | 双人蓄力 |
| 依赖 → | Core 网络 | 单个 `UFUNCTION(NetFunction) NetTriggerSidePortal` + `HasControl()` | **无 Crumb**，网络极简 |
| 被使用 ← | 关卡脚本/蓝图 | 绑 `OnSideGlitchEntered`、配 `SideGlitchProgressPoint`、蓝图继承三个 Abstract 基类 | Gameplay 树内**无 .as 引用** |
| 无关 ✗ | 相机/传送/横版切换 | —— | 本模块无相机或模式切换代码；"portal" 仅体现在命名与淡白屏，唯一 "Teleport" 是继承默认 `SmoothTeleport()` 吸附 |

---

## 关键文件

- `Gameplay/SideGlitch/SideGlitchInteractionActor.as` —— 双人 button-hold 入口，进度聚合 + `NetTriggerSidePortal` + 淡白屏 + 激活进度点。
- `Gameplay/SideGlitch/SideGlitchInteractionEffectHandler.as` —— 蓝图特效事件桥（进入支线等）。
- `Gameplay/SideGlitch/SideGlichGiants.as` / `SidePortalInteractionActor_Exit.as` —— 供数据侧扩展的空占位基类。

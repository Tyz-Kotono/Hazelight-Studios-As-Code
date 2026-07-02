# Core / Camera

> 职责：Split Fiction 的相机系统功能域，负责双人分屏相机的控制、跟随、混合、聚焦、震动与边界碰撞。共 4 个子模块、约 82 个文件。

## 功能域概览

相机系统服务于双主角 Mio / Zoe（`TPerPlayer<T>`），核心是「每位玩家一套相机栈」加上一个全局单例 `UCameraSingleton`，后者管理**分屏 ↔ 全屏**之间的投影混合与画面过渡。

整体运行链路（主干 Actor → Component → Updater 模式）：

```
AHazeCameraActor (相机摆放体)
   └─ UHazeCameraComponent (相机类型: 弹簧臂/聚焦/样条跟随/球窝/静态)
        └─ UHazeCameraUpdater (每帧位姿计算)
             ▲ PrepareUpdaterForUser 注入数据
UHazeCameraUserComponent (每位玩家一份, 派生 UCameraUserComponent)
   ├─ 输入 / 期望旋转 / 联机同步 (Crumb)
   ├─ Capabilities (控制/更新/复制/隐藏遮挡/非受控旋转...)
   └─ ComposableSettings (用户设置/控制旋转/冲量...)
UCameraSingleton (全局, 分屏↔全屏混合 + 投影偏移)
```

## 四个子模块

| 子模块 | 职责 | 文件数 |
| --- | --- | --- |
| [Camera](./Camera/README.md) | 相机系统主体：单例、用户组件、相机类型/Actor、混合、聚焦、POI、能力、设置、卷体、调试 | 79 |
| [CameraShake](./CameraShake/README.md) | 基于距离衰减的可移动相机震动组件 | 1 |
| [CameraShakeForceFeedback](./CameraShakeForceFeedback/README.md) | 相机震动 + 手柄力反馈一体触发组件 | 1 |
| [CameraBoundaryComponents](./CameraBoundaryComponents/README.md) | 样条跟随相机的动态碰撞边界（阻挡玩家越界出框） | 1 |

## 模块关系图

```
            ┌─────────────────────────────────────────┐
            │            Camera (主模块)                │
            │  UCameraSingleton / UCameraUserComponent  │
            │  CameraTypes · Blend · FocusTarget · POI  │
            └───────────┬───────────────────┬──────────┘
                        │ ASplineFollowCameraActor
                        │ (样条跟随相机)
                        ▼
            ┌───────────────────────────┐
            │  CameraBoundaryComponents  │  挂在样条相机 Actor 上,
            │  (UHazeCameraResponseComp) │  随相机更新移动阻挡碰撞盒
            └───────────────────────────┘

   CameraShake / CameraShakeForceFeedback  ── 独立组件, 通过
   Player.PlayCameraShake / PlayWorldCameraShake 调用引擎震动,
   与主模块无直接类型依赖 (并列于 Camera 域)。
```

- **CameraBoundaryComponents** 强依赖主模块：它派生自 `UHazeCameraResponseComponent`，且 `devCheck` 要求宿主必须是主模块的 `ASplineFollowCameraActor`，并读取其 `FocusTargetComponent`。
- **CameraShake / CameraShakeForceFeedback** 与主模块**无类型耦合**，只通过 `AHazePlayerCharacter` 的引擎接口 `PlayCameraShake` / `PlayWorldCameraShake` / `PlayForceFeedback` 触发，按 Mio/Zoe 分别施加。它们的主要消费者是关卡 Actor 与 AnimNotify（如 `AnimNotifyCameraShakeAndForceFeedback`、`KineticCameraShakeForceFeedbackComponent`）。

## 协作机制使用概览（5 类机制在本域的分布）

| 机制 | 说明 | 主要出现位置 |
| --- | --- | --- |
| 1 直接 Get/GetOrCreate | `UCameraUserComponent::Get(Player)` 等 | **全域普遍**（被 Aiming / Audio / Movement / 大量关卡 Capability 消费） |
| 2 Crumb 联机 | `HasControl()` + `CrumbFunction` | UserComponent 旋转同步、Volumes 双人入卷、POI 输入清除 |
| 3 能力标签 | `CameraTags::*` / `BlockCapabilities` | User / Modifiers / NonControlledRotation / HideOverlappers / Debug |
| 4 委托广播 | `event` / `AddUFunction` / `Broadcast` | Volumes（OnBothEntered 等）、HideOverlappers、各 Trigger |
| 5 可组合设置 | `ApplySettings` / `GetSettings` | Settings / ChaseAssistance / POI / SpringArm / FocusTarget（**数据流主干**） |

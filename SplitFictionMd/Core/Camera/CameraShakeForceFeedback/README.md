# Core / Camera / CameraShakeForceFeedback

> 职责：一次性触发「相机震动 + 手柄力反馈」的场景组件，支持世界空间半径衰减或直接对玩家施加。共 1 文件。

## 内部架构

| 类 | 基类 | 角色 |
| --- | --- | --- |
| `UCameraShakeForceFeedbackComponent` | `USceneComponent` | 同时播放相机震动与力反馈，可按世界半径衰减或全屏施加 |
| `UCameraShakeForceFeedbackScriptVisualizer` | `UHazeScriptComponentVisualizer` | 编辑器内绘制内/外半径线框（绿/红） |

关键属性：`CameraShake`（`TSubclassOf<UCameraShakeBase>`）+ 缩放、`ForceFeedback`（`UForceFeedbackEffect`）+ 缩放、`bPlayInWorld`、`InnerRadius/OuterRadius`、`SamplePosition`。

数据流：`ActivateCameraShakeAndForceFeedback(TargetPlayer = nullptr)`：
- 未指定玩家 → 同时对 Mio 与 Zoe 施加；指定则只对单人。
- `bPlayInWorld == true`：`Player.PlayWorldCameraShake(...)`（按 `InnerRadius/OuterRadius/SamplePosition` 衰减）+ `ForceFeedback::PlayWorldForceFeedback(...)`（按 `EHazeSelectPlayer` 选择目标）。
- `bPlayInWorld == false`：直接 `Player.PlayCameraShake` + `Player.PlayForceFeedback`。

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 调用 | `AHazePlayerCharacter` | 1 | `PlayWorldCameraShake` / `PlayCameraShake` / `PlayForceFeedback` |
| 调用 | `ForceFeedback::PlayWorldForceFeedback` | 1 | 世界空间力反馈（半径衰减 + `EHazeSelectPlayer`） |
| 调用 | `Game::Mio` / `Game::Zoe` | 1 | 默认对双主角同时施加 |
| 被调用 | AnimNotify / 关卡 Actor | 1 | 如 `AnimNotifyCameraShakeAndForceFeedback`、`Gameplay/KineticActors/KineticCameraShakeForceFeedbackComponent` 等触发 `ActivateCameraShakeAndForceFeedback` |

> 注：与相机主模块无类型依赖，仅复用引擎震动与力反馈接口；是相机域内的独立反馈工具组件。

## 关键文件

- `CameraShakeForceFeedbackComponent.as` —— 相机震动 + 力反馈一体触发组件及其编辑器可视化。

# Core / Camera / CameraShake

> 职责：基于玩家与震源距离衰减的「可移动相机震动」组件，按 Mio/Zoe 分别开启或关闭。共 1 文件。

## 内部架构

| 类 | 基类 | 角色 |
| --- | --- | --- |
| `UMovableCameraShakeComponent` | `UActorComponent` | 在半径内按距离施加相机震动，每帧调整强度 |
| `UMovableCameraShakeVisualizer` | `UHazeScriptComponentVisualizer` | 编辑器内绘制震动半径线框（紫色） |
| `FMovableCamShakeSettings` | struct | 震动配置：半径、震源偏移、`UCameraShakeBase` 类、缓出指数/缩放或曲线、是否对 Mio/Zoe 生效 |

数据流：组件默认不 Tick，由 `ActivateMovableCameraShake()` 打开 Tick。每帧对每位玩家：
1. 按 `bActiveForMio/bActiveForZoe` 过滤；
2. `GetShakeScale(PlayerLocation)` 用「玩家到震源距离 / 半径」算归一化强度（曲线资产或 `Math::EaseOut`）；
3. 强度 > 0 时通过 `Player.PlayCameraShake(...)` 启动并缓存 `CamShakeInstance[Player]`（`TPerPlayer` 思路用 `TPerPlayer<bool>` / `TPerPlayer<UCameraShakeBase>`），后续仅更新 `ShakeScale`；强度归零时 `Player.StopCameraShakeInstance`。

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 调用 | `AHazePlayerCharacter` | 1 | `PlayCameraShake` / `StopCameraShakeInstance` / `StopCameraShakeByInstigator`（引擎相机震动接口） |
| 调用 | `Game::Players` / `Game::Mio` / `Game::Zoe` | 1 | 遍历双主角并按角色过滤 |
| 被调用 | 关卡 Actor 蓝图/脚本 | 1 | 通过 `ActivateMovableCameraShake` / `DeactivateMovableCameraShake` 控制（UFUNCTION，供外部触发） |

> 注：与相机主模块（`Camera/`）**无类型依赖**，仅复用引擎层 `UCameraShakeBase` 与玩家震动接口；属于相机域内的独立工具组件。

## 关键文件

- `MovableCameraShakeComponent.as` —— 距离衰减式可移动相机震动组件及其编辑器可视化。

# LevelSpecific / Coast / WingSuit — 翼装滑翔

> 路径：`LevelSpecific/Coast/WingSuit/`（24 文件）＋ `WingSuitBots/`（2）

## 职责

追逐序列的第二段：玩家穿上翼装从列车飞出，进入自由滑翔飞行。核心是空气动力式的失重滑翔手感——加速、俯冲换速、桶滚、借抓钩转向，并在飞行边界内被橡皮筋式拉回航道。段末衔接滑水。

## 内部架构

- **载具与挂接** — `AWingSuit : AHazeActor` 为翼装本体，`UWingSuitPlayerComponent : UActorComponent` 保存每玩家状态，`AWingsuitManager : AHazeActor` 管理整段流程与边界。
- **核心飞行能力** — `UWingSuitMovementCapability : UHazePlayerCapability` 负责水平/垂直加速、俯冲换速、桶滚、橡皮筋回拉与镜头晃动。围绕它的能力：
  - `WingSuitPlayerInputCapability` — 输入采集
  - `WingSuitGrappleCapability` — 抓钩点借力转向
  - `WingSuitPlayerCameraCapability` — 飞行镜头
  - `WingSuitFlyingOffRampCapability` — 从下坡/坡道起飞进入滑翔
  - `WingsuitPlayerDeathCapability` — 撞击死亡与重生
- **Audio/** — 监听器与反射 trace 能力（Wwise 空间音频）。
- **WingSuitBots/** — `AWingSuitBots : AHazeActor` 伴飞/领航僚机。
- **形态互转** — `CoastWingsuitToWaterskiTransitionCapability`（与滑水段的反向转换成对）实现翼装落水无缝切滑水。

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
|---|---|---|---|
| 依赖 | Gameplay | `UPlayerMovementComponent` + `USweepingMovementData` | 滑翔移动底座 |
| 依赖 | Gameplay | 抓钩点组件 | 空中借力转向 |
| 依赖 | Audio | KuroAudio/Wwise 监听/反射 | Audio 子目录空间音频 |
| 衔接 | Coast/Train | 飞出列车 | 承接火车段 |
| 衔接 | Coast/Waterski | `...ToWaterskiTransitionCapability` | 落水切换为滑水 |

## 关键文件

- `LevelSpecific/Coast/WingSuit/WingSuit.as` — 翼装本体
- `LevelSpecific/Coast/WingSuit/Capabilities/WingSuitMovementCapability.as` — 核心滑翔能力
- `LevelSpecific/Coast/WingSuit/WingsuitManager.as` — 段落管理
- `LevelSpecific/Coast/WingSuit/Capabilities/`（Input/Grapple/Camera/OffRamp/Death）
- `LevelSpecific/Coast/WingSuitBots/` — 伴飞僚机

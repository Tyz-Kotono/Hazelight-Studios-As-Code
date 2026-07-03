# LevelSpecific / Prison / Pinball — 弹球台 Boss 战

> 路径：`LevelSpecific/Prison/Pinball/`（201 文件）
>
> Prison 的招牌高潮：整个战斗场景是一张巨型弹球台。玩家驾驶磁力无人机（MagnetDrone）在台面上活动，同时台上滚动着一颗大球（BossBall）；用柱塞、拨片、挡板把球打向弹球 Boss 的弱点。这是"弹球物理 + 无人机驾驶 + Boss 战"的三合一玩法。

承接 [Prison 总览](../README.md)。本机制大量复用 [Drones 无人机](../Drones/) 的磁力驾驶，请先读该篇。

---

## 职责

把标准弹球台元素（球、柱塞、拨片、挡板、斜坡、门）实现为可联机的物理玩法，并接入 Boss 战：球撞 Boss 造成伤害、Boss 反击、球出界重置。

---

## 内部架构

### 三大主体

| 主体 | 关键类 | 基类 | 说明 |
|---|---|---|---|
| **球（BossBall）** | `APinballBossBall` | `AHazeActor` | 台面上的大球，承载弹射/移动/受击。挂 `UPinballBallComponent`（共享球逻辑）+ 一组能力（见下） |
| **弹球 Boss** | `APinballBoss` | `AHazeActor` | 被攻击目标，状态机驱动（Idle/Following/ChargeAttack/Ball/Dying），有可开合的核心（Core）弱点 |
| **磁力无人机（玩家）** | `Pinball/MagnetDrone/*` | 复用 Drones | 玩家操作单元，在台上移动、吸附、把球打出去 |

### 球的能力表（CapabilitySheet 组织）

`PinballBossBall.as` 用多张 `UHazeCapabilitySheet` 装配球的行为，是能力四件套的典型应用：

- **控制端 Sheet**：`AirMove`（空中移动）、`Launched`（被弹射）、`LaunchedTrail`（拖尾）、`Move`、`LaunchTrajectory`（弹道）。
- **共享 Sheet**：`UpdateMeshRotation`（网格滚动）、`LaunchedOffset`（弹射偏移矫正）。
- **远端 Sheet（Proxy）**：`Prediction*` 系列，见"网络"。

球的核心状态收束在 `UPinballBossBallLaunchedComponent`：记录是否被弹射、弹射数据 `FPinballBallLaunchData`、近期弹射历史、弹道与网络预测时间。

### 台面元素（Actors/）

| 元素 | 类 | 说明 |
|---|---|---|
| 柱塞 | `Plunger/`（`APinballPlunger`） | 蓄力发射球 |
| 拨片 | `Paddle/` | 玩家控制的弹球拨片 |
| 挡板/旋转器 | `Pinball_Spinner`、`PinballFlaper` | 台面反弹装置 |
| 弹跳垫 | `BouncePad/` | 定向弹射 |
| 门 | `PinballGate/` | 通道开合 |
| 可黑客弹球物 | `HackablePinball/`（`AHackablePinball` + Manager + ResponseComponent） | 远程黑客改变台面 |
| 死亡/重置 | `PinballDeathVolume`、`PinballBoss_RisingDeathVolume`、`GlobalReset/` | 球出界与全局重置 |
| 辅助 | `PinballAutoAim`、`PinballFocusTarget`、`PinballDofManager`、`PinballTimer` | 自动瞄准、聚焦、景深、计时 |

### 网络（Network/）

弹球对实时性与联机同步要求高，`Pinball/Network/` 是独立的重头模块：`BossBall/Predictabilities/`、`MagnetDrone/Predictabilities/`（Attached/Attraction/Launched/Movement/Rail）、`Proxy/`、`Prediction/`、`RecordTransform/`。球与无人机在远端用 Predictability 做客户端预测 + 代理回放，避免弹射轨迹抖动。

### 数据流

```
玩家(磁力无人机) → 柱塞/拨片 蓄力
   → UPinballBallComponent.OnLaunched(FPinballBallLaunchData)
   → BossBall Launched 能力接管弹道飞行
   → 命中 Actors/BossDamageZone → PinballBoss 受击/开核心
   → 出界 → DeathVolume → GlobalReset 重置球
```

---

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 复用 | `Drones/` 磁力驾驶 | 无人机 | `Pinball/MagnetDrone/Magnet/`（Aim/Attached/Attraction/StartAttract）整套照搬无人机磁力体系；能力标签含 `PrisonTags::Drones`（见 `PinballBossBallLaunchedCapability.as`） |
| 依赖 | `UHazeMovementComponent` | Core/Move | 球与无人机移动均基于 Haze 移动组件（`SetupMovementData(UPinballMagnetDroneMovementData)`） |
| 引用 | `UPinballLauncherComponent` | 本机制 | `FPinballBallLaunchData.LaunchedBy` 记录发射者，柱塞/拨片实现该组件 |
| 输出 | `APinballBoss` 弱点 | 本机制 Boss | 球命中 `Actors/BossDamageZone` 触发 Boss 开核心受伤 |
| 标签互斥 | `Pinball::Tags::Pinball` / `PinballMovement` / `PinballLaunched` | Core/Tags | 球的移动/弹射状态用标签互斥切换（见能力 `CapabilityTags`） |

---

## 关键文件

- `Pinball/BossBall/PinballBossBall.as` — 球主体与 CapabilitySheet 装配。
- `Pinball/BossBall/Capabilities/Launch/PinballBossBallLaunchedComponent.as` — 球弹射核心状态容器。
- `Pinball/Shared/Components/PinballBallComponent.as` — 共享球逻辑与弹射数据结构（`FPinballBallLaunchData`）。
- `Pinball/PinballBoss/PinballBoss.as` — 弹球 Boss 状态机（`EPinballBossState`）。
- `Pinball/MagnetDrone/PinballMagnetDroneCapability.as` — 玩家操作单元（复用无人机磁力）。
- `Pinball/Network/` — 联机预测/代理模块。
- `Pinball/PinballSettings.as` / `PinballStatics.as` / `PinballTags.as` — 设置/静态/标签四件套。

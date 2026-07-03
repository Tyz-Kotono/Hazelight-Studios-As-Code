# LevelSpecific / Prison — 太空监狱越狱（科幻线）

> 路径：`LevelSpecific/Prison/`（1032 文件）
>
> Mio 科幻线的越狱关卡：一座高科技太空监狱。核心体验围绕**黑客改造关卡物件**与**双人驾驶无人机**展开——从牢区潜行、无人机磁力攀爬、垃圾处理场逃生，到最终的巨型弹球台 Boss 战。

本文承接 [LevelSpecific 总览](../README.md) 与 [Gameplay 设计哲学](../../Gameplay/DesignPhilosophy.md)（能力四件套 Capability+Component+Settings+Statics、标签互斥、组件容器范式）。Prison 不发明新范式，是这套范式在"载具驾驶 + 场景改造"玩法族上的集中应用，哲学不再重复。

---

## 关卡主题与流程骨架

科幻线叙事以"逃出监狱"为主线，机制随场景推进层层叠加：

1. **牢区（CellBlock）** — 越狱开场：牢门、囚犯运输平台、垃圾滑道、垃圾压缩机等可黑客物件构成的初段。
2. **潜行（Stealth）** — 顶视/侧视潜行段：躲避守卫巡逻、监控摄像头视锥，用纸箱伪装、击晕机器人。
3. **无人机（Drones）** — 核心载具：磁力无人机（MagnetDrone）与蜂群无人机（SwarmDrone）双形态，磁力吸附攀爬、蜂群滑翔与劫持。
4. **磁场（MagneticField）** — 玩家借磁场充能/斥力做大跨度弹射位移，贯穿多个无人机与 Boss 段。
5. **最高安保区（MaxSecurity）** — 激光地狱：激光墙、激光切割机、可黑客机械，高压穿越段。
6. **弹球（Pinball）** — 招牌高潮：玩家化身弹球台上的球与拨片操作者，用弹球物理攻击弹球 Boss。
7. **Boss（PrisonBoss）** — 持刀狱警 Boss 多阶段战，回收磁力/黑客/弹球能力，终局进入 Boss 脑内（Brain）。

详见下方招牌机制索引。

---

## 招牌机制索引

| 机制 | 文件数 | 一句话职责 | 文档 |
|---|---:|---|---|
| **弹球（Pinball）** | 201 | 巨型弹球台 Boss 战：球体物理、拨片/柱塞/挡板、磁力无人机操作、弹球 Boss | [Pinball/](./Pinball/) |
| **无人机（Drones）** | 193 | 磁力无人机 + 蜂群无人机双形态：磁力吸附攀爬、蜂群滑翔/劫持/载船 | [Drones/](./Drones/) |
| **Boss（PrisonBoss）** | 111 | 持刀狱警 Boss：十余种攻击、追杀处决、脑内终局 | [Boss/](./Boss/) |
| **潜行体系（Stealth+）** | 42+52+23 | 潜行躲避 + 最高安保激光 + 牢区可黑客物件 | [Stealth/](./Stealth/)（含 MaxSecurity / CellBlock） |
| LevelActors | 267 | 通用可放置物件库：各类可黑客机关、磁场物件、竞技场 | 见下表说明 |
| MagneticField | 18 | 玩家磁场弹射系统：充能、斥力、发射位移 | 见下表说明 |
| SideInteractions | 32 | 支线互动：牢房弹射、股市、鸽子发射器、无人机跳舞等 | 见下表说明 |
| GarbageRoom | 22 | 垃圾处理场：垃圾车、逃生段 | 见下表说明 |

### 小机制 / 通用目录说明

| 目录 | 关键类 | 说明 |
|---|---|---|
| LevelActors | `AMagneticFieldPhysicsObject`、`DroneHackables/*`（`AHackableCraneArm`、`ASniperTurret` 等）、`RemoteHackables/*`（`ATazerBot`、`ATelescopeRobot`、`ARobotVacuum` 等）、`AArena*` | 全关最大目录（267），通用可放置物件库。分两大族：**无人机可黑客物**（`DroneHackables/`，无人机磁力/攻击触发）与**远程可黑客物**（`RemoteHackables/`，玩家远程黑客操控）。都是独立 actor，按需查子目录名 |
| MagneticField | `AMagneticFieldRadiusVolume`、`UMagneticFieldPlayerComponent`、`UMagneticFieldPlayerChargeCapability`、`UMagneticFieldPlayerLaunchMovementCapability` | 玩家磁场移动系统：进入磁场体积后充能，触发斥力/发射做大跨度位移。被无人机段、Boss 段共用 |
| SideInteractions | `CellEjection/`、`StockMarket/`、`PigeonLauncher/`、`SwarmDroneDancing/`、`OilPuddle/`、`MagnetAnnoyingDrone/` | 可选支线互动小玩具，不影响主线 |
| GarbageRoom | `AGarbageTruck`、`GarbateTruck/` | 垃圾处理场景剧本 |

---

## 通用目录（一笔带过，不建分篇）

- **`AI/`（34）** — 通用敌人库：守卫（Guard）、守卫机器人（GuardBot/Zapper）及穿越 AI。敌人多继承自 Gameplay 的 AI 框架并挂 `UHazeCapabilityComponent`（Behaviours）。AI 框架见 [Gameplay/AI](../../Gameplay/README.md)。
- **`Audio/`（8）** — 音频挂接，基于 KuroAudio/Wwise，见 [Audio](../../Audio.md)。
- 其余零散目录：`Nosedive/`、`PrisonDefense/`（ShootEmUp 小段）、`RemoteHacking/`（远程黑客共享逻辑）、`LevelScriptActors/`、`Tags/`。

---

## 机制间关系

- **弹球 ⊃ 无人机**：弹球段的玩家操作单元 `Pinball/MagnetDrone/` 是磁力无人机的特化复用——弹球本质是"磁力无人机在弹球台里控制球"，共用 `Drones/` 的磁力吸附与移动能力。
- **无人机 ↔ LevelActors**：磁力无人机吸附/攻击 `LevelActors/DroneHackables/` 系列可黑客物（起重臂、狙击炮塔、电板等），构成关卡谜题。
- **Boss ↔ 磁场 / 黑客**：`PrisonBoss` 的磁力猛砸（`MagneticSlam`）、可黑客磁力弹（`PrisonBossHackableMagneticProjectile`）、斥力面（`PrisonBossRepelSurface`）复用 `MagneticField/` 与远程黑客系统。
- **潜行 ↔ 蜂群无人机**：`Stealth/` 段中出现 `SwarmBoat`、蜂群无人机等（`LevelActors/Stealth/SwarmBoat`），潜行与无人机机制交叠。
- **弹球 Boss = 弹球台终局**：`Pinball/PinballBoss/` 与 `Pinball/BossBall/` 是弹球高潮的一体两面——球体（BossBall）撞击 Boss 弱点。

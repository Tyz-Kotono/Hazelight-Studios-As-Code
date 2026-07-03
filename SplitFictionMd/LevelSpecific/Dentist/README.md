# LevelSpecific / Dentist（牙医 Boss 竞技场）

> 路径：`LevelSpecific/Dentist/`（286 文件，奇幻 / Zoe 线）
>
> Dentist 是一整关围绕**单场 Boss 战**构建的竞技场。反派"The Dentist"是一台多臂机械牙医，站在一座旋转蛋糕（`Cake`）竞技场中央，用一整套牙科工具（钻头、假牙、纸杯、牙刷、牙膏、刮匙、锤子）依次攻击玩家；玩家这一关被变成一颗**会跳的牙齿**（Tooth），拥有该关专属的一套轻量玩家能力（跳/冲刺/砸地/被劈成两半）。整关不是"走关卡"，而是"打一个多阶段脚本化 Boss"。本层不发明新范式，是 Core/Gameplay 的**能力四件套**（Capability + Component + Settings + Statics）、**标签互斥**、**组件容器**在 Boss 编排与玩家变身上的应用。上层设计见 [Core 设计哲学](../../Core/DesignPhilosophy.md)、[Gameplay 设计哲学](../../Gameplay/DesignPhilosophy.md)。

---

## 目录构成

| 类别 | 目录 | 说明 |
|---|---|---|
| **招牌 Boss** ⭐ | `Boss/`(123) | 牙医 Boss 本体、七套工具、动作队列编排、阶段状态机、终结技 |
| **招牌玩家** ⭐ | `Player/`(98) | 玩家变身"牙齿"的专属移动/跳跃/冲刺/砸地/分裂能力四件套 |
| **关卡机关** | `LevelActors/`(65) | 竞技场内一次性道具：弹射炮、弹跳樱桃/棒棒糖、易碎地面、蜡烛、糖果机、间歇泉球、华夫饼等，多为奇幻甜品主题 |

---

## 招牌机制分篇

| 机制 | 文件数 | 一句话 | 文档 |
|---|---:|---|---|
| [牙医 Boss](./Boss/) | 123 | 多臂机械牙医，七套工具轮番攻击，三队列动作队列驱动的多阶段状态机 + 终结技 | [Boss/](./Boss/) |
| [牙齿玩家 Player](./Player/) | 98 | 玩家变身会跳的牙齿，专属跳/冲刺/砸地/分裂能力，砸地是打 Boss 弱点的核心手段 | [Player/](./Player/) |

---

## Dentist 通用范式（读一处懂全部）

1. **一切围绕单个 `ADentistBoss` 编排**
   Boss 是全关中枢。它持有一张 `TMap<EDentistBossTool, ADentistBossTool>` 管理七套工具，持有 3 个 `UHazeActionQueueComponent`（动作队列）驱动全部攻击时序，用一个 `EDentistBossState` 大状态机推进战斗（`Start → RestrainedInChair → ToothBrush → Dentures → Cup → Cake 旋转 → … → Defeated → Chase`）。玩家能力、相机、工具都由 Boss 的动作队列 `Capability()/Idle()/Event()` 逐帧插入。

2. **玩家 = 牙齿变身，能力挂在玩家组件上**
   玩家不是常规角色，而是套上 `UDentistToothPlayerComponent` 变成牙齿：覆盖胶囊尺寸、把网格挂到 `MeshOffsetComponent` 上做整体旋转、生成 `ADentistTooth` 视觉体（含活动眼球）。该关一整套移动能力（`DentistTooth*`）替换了 Gameplay 的默认移动，走自定义 `UDentistToothMovementResolver : USteppingMovementResolver`。

3. **弱点 = 砸地打手**
   Boss 的两只手（`LeftHand/RightHandWeakpointMesh` + `UBasicAIHealthComponent`）是可攻击弱点；玩家用**砸地**（GroundPound）命中即 `CrumbArmTakeDamage`。假牙咬手、纸杯抓人、刮匙+锤子把牙齿劈成两半（Split）等都是围绕"制造弱点窗口 → 玩家砸地"的循环。

4. **联机统一 Crumb**
   Boss 的伤害、死亡、工具交换、纸杯排序、玩家分裂等状态改变都走 `CrumbFunction`/`NetworkMode = Crumb`，与 Core/Flow 的联机一致性模型一致。

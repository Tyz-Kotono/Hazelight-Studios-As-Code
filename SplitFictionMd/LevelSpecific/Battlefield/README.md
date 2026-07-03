# LevelSpecific / Battlefield（科幻战区）

> 路径：`LevelSpecific/Battlefield/`（254 文件，科幻 / Mio 线）
>
> Battlefield 是一场高速**悬浮板穿越战区**的关卡。两名玩家骑悬浮板（Hoverboard）在被外星舰队轰炸的战场上竞速前进：磨轨、跑墙、抓钩、摆荡、腾空做特技，同时躲避火炮、导弹、激光、炮塔组成的密集火力，并穿过一连串脚本化大场面（外星巡洋舰、飞船残骸、坦克脚踩踏、终局战列巡洋舰主炮）。核心是**悬浮板 = 一整套载具移动系统**，其余系统都服务于"在高速移动中制造威胁与奇观"。本层不发明新范式，是 Core/Gameplay 的**能力四件套**（Capability + Component + Settings + Statics）、**标签互斥**、**组件容器**、**Move 系统 Resolver** 在竞速载具上的大规模应用。上层设计见 [Core 设计哲学](../../Core/DesignPhilosophy.md)、[Gameplay 设计哲学](../../Gameplay/DesignPhilosophy.md)。

---

## 目录构成

| 类别 | 目录 | 说明 |
|---|---|---|
| **招牌载具** ⭐ | `Hoverboard/`(133) | 悬浮板：地/空移动、磨轨、跑墙、抓钩、摆荡、跳跃、特技、竞速橡皮筋、相机 |
| **武器系统** ⭐ | `ProjectileSystems/`(7)、`MissileSystem/`(7)、`Turrets/`(5)、`LaserSystems/`(5)、`BaseSystems/`(5) | 抛射物/导弹/炮塔/激光四类武器 + 共享基类，威胁玩家的火力 |
| **场景剧本** ⭐ | `Scenarios/`(50) | 一次性脚本化大场面：外星巡洋舰、飞船残骸、坦克脚、终局主炮、慢动作段 |
| **通用/其它** | `Objects/`(17)、`SplineMovingObjects/`(7)、`ArtillerySystem/`、`BurrowingAliens/`、`IceCavern/`、`Audio/` | 可动物件、沿样条移动的战机、火炮、钻地外星人、冰洞、音频。**本文档一笔带过** |

---

## 招牌机制分篇

| 机制 | 文件数 | 一句话 | 文档 |
|---|---:|---|---|
| [悬浮板 Hoverboard](./Hoverboard/) | 133 | 竞速载具全栈：磨轨/跑墙/抓钩/摆荡/跳跃/特技 + 自定义 Resolver + 竞速橡皮筋 | [Hoverboard/](./Hoverboard/) |
| [武器系统 WeaponSystems](./WeaponSystems/) | 29 | 抛射物/导弹/炮塔/激光四类威胁，共享 BaseSystems 基类，池化 + 沿样条 | [WeaponSystems/](./WeaponSystems/) |
| [场景剧本 Scenarios](./Scenarios/) | 50 | 一次性脚本化大场面：外星巡洋舰、终局主炮、坦克脚、慢动作段 | [Scenarios/](./Scenarios/) |

---

## Battlefield 通用范式（读一处懂全部）

1. **悬浮板 = 玩家组件 + 生成实体 + 能力群**
   悬浮板不常驻，而是把状态存在玩家身上的 `UBattlefieldHoverboardComponent`（持有所有子系统 Settings + 运行态），骑乘能力 `RidingCapability` 在 `Setup` 时 `SpawnActor` 出 `ABattlefieldHoverboard` 实体并挂到玩家手骨。全部移动子系统（磨轨/跑墙/抓钩/摆荡…）各自一个 Component + Capabilities + Settings。

2. **自定义 Move Resolver**
   悬浮板走 `UBattlefieldHoverboardMovementResolver : USteppingMovementResolver`，重写 `ProjectMovementUponImpact` 让高速载具在斜坡/边缘上不被"吸下去"。所有移动能力都是 `UHazePlayerCapability`，用 `CapabilityTags::Movement` + 自定义标签（`BattlefieldGrinding` / `BattlefieldHoverboard`）互斥。

3. **武器系统 = 池化 + 共享基类**
   抛射物/导弹都用 PoolManager 预生成对象池（`ABattlefieldProjectilePoolManager` 预建 50 个），避免运行时频繁 spawn。四类武器共享 `BaseSystems/` 的 `AttackComponent` / `FollowSpline` / `Bobbing` 基础组件，很多攻击沿样条飞行（`ProjectileFollowSplineComponent`）。

4. **竞速橡皮筋（Rubberbanding）**
   为保证双人竞速不脱节，`LevelRubberbanding/` + `Grinding` 内的 `Rubberbanding` 能力把落后玩家沿关卡样条拉近，`PreferredAheadPlayerSettingVolume` 等体积微调。

5. **联机统一 Crumb**
   移动能力（如 `GrindingCapability`）与场景剧本状态改变统一 `NetworkMode = Crumb`，与 Core/Flow 一致。

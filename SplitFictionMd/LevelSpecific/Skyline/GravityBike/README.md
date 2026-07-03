# LevelSpecific / Skyline / GravityBike（重力摩托）

> 路径：`LevelSpecific/Skyline/GravityBike/`（414 文件，Skyline 最大招牌机制）
>
> 赛博城市高速载具玩法：玩家骑重力摩托在墙面/管道/斜坡上贴面飞驰，做特技、打敌人、飙车追逐。是 Skyline 的核心驾驶体验。

承接 [Skyline 关卡总览](../README.md)。载具复用 Gameplay 的 Move 系统（`UFloatingMovementResolver`）、能力四件套与标签互斥范式，不重复哲学，见 [Gameplay 设计哲学](../../../Gameplay/DesignPhilosophy.md)。

---

## 职责

重力摩托是一辆 `AHazeActor` 载具，不是角色。玩家（`AHazePlayerCharacter`）通过"驾驶员"能力（`UHazePlayerCapability`）附着到载具上并向其转发输入，载具自身用一套能力集（`UHazeCapability`，挂在载具的 `UHazeCapabilityComponent` 上）跑转向、油门、悬浮、对齐、特技、武器等逻辑。

核心分成两大形态，各自独立实现（类名前缀 `Free` / `Spline`）：

| 形态 | 目录 | 文件 | 说明 |
|---|---|---:|---|
| **Free（自由驾驶）** | `Free/` | 148 | 开放式驾驶，玩家完全控制转向/油门，可在半管、斜坡、通风管上自由走位、漂移、做特技。用于自由探索与竞速段。 |
| **Spline（样条驾驶）** | `Spline/` | 221 | 在轨（on-rails）驾驶，摩托被约束在样条 `UHazeSplineComponent` 上，玩家只控制左右摆位与油门。用于剧本化追逐战、双人（驾驶员+乘客）段、敌人围攻段。 |
| **Fullscreen** | `Fullscreen/` | 11 | 全屏（非分屏）演出用的重力摩托关卡道具（破碎阳台、吊车等）。 |
| **Phone** | `Phone/` | 33 | 摩托手机 UI / 小游戏（车载娱乐系统、小部件）。 |
| **Shared** | `Shared/` | 1 | `GravityBikeWheelComponent` 车轮组件，Free/Spline 共用。 |

> Free 与 Spline 是**两套平行实现**，不共享 Actor 基类，仅共用 `UGravityBikeWheelComponent`（`Shared/GravityBikeWheelComponent.as`）与相同的设计范式。

---

## 内部架构

### 通用骨架（Free 与 Spline 同构）

两套形态的主 Actor 结构高度一致，都由"载具 Actor + 移动解算器 + 一堆能力"组成：

```
AGravityBikeFree / AGravityBikeSpline (AHazeActor)
├── USphereComponent Sphere            // 碰撞体（半径48，PlayerCharacter profile）
├── UHazeSkeletalMeshComponentBase     // 车身网格
├── UGravityBikeWheelComponent x2      // 前/后轮（Shared）
├── UHazeCapabilityComponent           // 能力容器
├── UGravityBike*MovementComponent     // 移动组件
├── UGravityBike*SyncComponent         // 联机状态同步（High 频率）
├── UHazeCrumbSyncedActorPositionComponent  // 位置面包屑同步
├── USplineLockComponent               // 样条锁定
└── FGravityBike*Input Input           // 输入结构（转向/油门，带联机取值）
```

移动核心是 `UGravityBikeFreeMovementResolver : UFloatingMovementResolver`（`Free/Movement/GravityBikeFreeMovementResolver.as`）——直接继承 Gameplay 的浮空移动解算器，处理落地冲量、撞墙对齐（AlignWithWall）、撞墙致死（DeathFromWall）、地面/空中状态切换等。Spline 侧对应 `UGravityBikeSplineMovementResolver`。

**重力对齐**是招牌手感：`FHazeAcceleratedQuat AccUpVector`（Free）/ `AccBikeUp`+`AccGlobalUp`+`AccTurnReference`（Spline）平滑插值车身"上方向"，使摩托能贴合墙壁、管道内壁、斜坡行驶而不突兀翻转。`BeginPlay` 里通过 `UMovementGravitySettings::SetGravityAmount` 设定重力系数。

### Free 形态子系统

| 子系统 | 目录 | 关键类 / 说明 |
|---|---|---|
| 驾驶员 | `Free/Driver/` | `UGravityBikeFreeDriverCapability`（`UHazePlayerCapability`，附着玩家、请求 Locomotion）、`UGravityBikeFreeDriverComponent`（按 Mio/Zoe 生成并 MakeNetworked 载具）、相机 |
| 转向/油门 | `Free/Capabilities/` | `UGravityBikeFreeSteeringCapability`、`GravityBikeFreeThrottleCapability`（速度相关的转向角、漂移增幅）、输入、拖尾、撞墙、动画、爆炸 |
| 移动 | `Free/Movement/` | 地面/空中移动能力、状态、与地面接触对齐（`AlignWithGroundContactCapability`） |
| 悬浮 | `Free/Hover/` | `GravityBikeFreeHoverComponent` 悬浮/俯仰/横滚冲量（落地反馈） |
| 特技 | `Free/Trick/` `Free/KartDrift/` `Free/HalfPipe/` `Free/QuarterPipe/` | 空中特技、卡丁漂移、半管/四分管（Aim/BackFlip/Land/Movement 子状态） |
| 加速/跳跃 | `Free/Boost/` `Free/Jump/` | 助推（`BoostComponent`）、跳跃与各类跳跃触发器（Circle/Ramp/Jump Trigger） |
| 武器 | `Free/Weapons/` | 机枪（`MachineGun`）+ 导弹（`MissileLauncher`），带瞄准/锁定/十字准星/充能样条/可拾取（`Pickup`）；目标用 `GravityBikeWeaponTargetableComponent` |
| 对齐/自动转向 | `Free/Alignment/` `Free/AutoSteer/` | 空中/地面对齐、自动转向辅助（`AutoSteerTargetComponent`） |
| 附件 | `Free/Blade/` `Free/Vent/` | 车载刀片、通风管穿行 |

### Spline 形态子系统

Spline 形态额外承载**敌人系统**与**双武器（鞭/刀）交互**，是追逐战主场：

| 子系统 | 目录 | 关键类 / 说明 |
|---|---|---|
| 玩家 | `Spline/Player/` | 驾驶员（`GravityBikeSplineDriverCapability`）+ 乘客（`Passenger`）双人；乘客负责用武器 |
| 转向/移动 | `Spline/Steering/` `Spline/Capabilities/Movement/` | `SteeringComp` 求目标转向增量旋转，`AccTurnReference` 做转向参考延迟；地面/空中移动+对齐 |
| 敌人 ⭐ | `Spline/Enemies/` | 敌方载具与单位：`Bike`（敌摩托，含驾驶员/乘客/手枪/掉落）、`Car`（敌车+炮塔 Turret）、`AttackShip`（攻击飞船）、`Enforcer`（执法者+可投掷物）、`Flying`、`Missile`（导弹+发射器）。都用 `UHazeCapabilitySheet` 组装能力集 |
| 重力鞭 ⭐ | `Spline/Whip/` | 乘客用鞭：`Grab`（瞄准/套索 Lasso/拉拽 Pull）、`Throw`（投掷/瞄准/回弹）、`Throwable`（可投掷物/被抓/被扔状态）、`SideScroller` |
| 重力刀 ⭐ | `Spline/Blade/` | 乘客用刀：投掷（`Throwing`/`Thrown`）、桶滚锚点（`Barrel` 抓取到桶）、表面抓取（`GrappleToSurface`）、重力触发器 |
| 加速/跳跃/悬浮 | `Spline/Boost/` `Spline/Jump/` `Spline/Hover/` | 与 Free 对应但样条化，Boost 带 FOV 效果 |
| 触发器 | `Spline/Trigger/` `Spline/AlongSplineTriggers/` `Spline/AutoJump/` `Spline/AutoSteer/` | 沿样条布置的触发器：应用设置、自动跳跃（Trigger→Target）、自动转向体积 |
| 其它 | `Spline/RedirectVelocity/` `Spline/InheritMovement/` `Spline/TimeDilationZone/` | 速度重定向、继承运动、时间膨胀区 |

> `Spline/Whip` 与 `Spline/Blade` 是摩托**追逐战专用**的鞭/刀实现，与独立步行武器 [GravityWeapons](../GravityWeapons/)（`GravityWhip/` `GravityBlade/`）是**不同实现**——前者绑定摩托乘客与样条，后者是玩家步行时的能力。

### 数据流（一帧）

```
玩家输入 → Driver Capability 转发 → FGravityBikeInput（转向/油门，SyncComp 联机取值）
  → Steering/Throttle Capability 计算 AccSteering / 目标速度
  → MovementResolver（继承 UFloatingMovementResolver）解算：碰撞、落地冲量、撞墙对齐/致死
  → HoverComp 施加俯仰/横滚冲量 + AccUpVector 平滑上方向（贴墙/管道）
  → 动画数据 FGravityBikeAnimationData → 网格
  → EventHandler 广播落地/离地/撞墙事件（音频/特效挂接）
```

---

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| 继承 | `UFloatingMovementResolver` | Gameplay/Move | `Free/Movement/GravityBikeFreeMovementResolver.as:1` 直接继承 |
| 使用 | `UMovementGravitySettings` | Gameplay/Move | `Free/GravityBikeFree.as:192` `SetGravityAmount(this, GravityFactor)` 设重力 |
| 使用 | `UHazeSplineComponent` / `USplineLockComponent` | Core/Spline | Spline 形态约束载具在样条上（`Spline/GravityBikeSpline.as`） |
| 附着 | `AHazePlayerCharacter` | Gameplay | 玩家经 `UHazePlayerCapability` 驾驶员能力附着并请求 Locomotion（`Free/Driver/Capabilities/GravityBikeFreeDriverCapability.as:49`） |
| 联机 | `UHazeCrumbSyncedActorPositionComponent` + SyncComp | Core/网络 | 位置面包屑 + 高频状态同步，敌人预测用 SyncComp（`Spline/GravityBikeSpline.as:276`） |
| 组装 | `UHazeCapabilitySheet` | Gameplay | Spline 敌人用能力表组装（`Spline/Enemies/Bike/GravityBikeSplineBikeEnemy.as:1`） |
| 依赖 | Skyline `AI/GravityWhippable` / `AI/GravityBladeReactions` | Skyline/AI | 敌人对鞭/刀的受击反应行为（供 Spline 追逐战鞭刀命中使用） |
| 触发 | `SkylineHighwayCombatStatics` | Skyline 根 | 高速路战斗执法者批量移除（`LevelSpecific/Skyline/SkylineHighwayCombatStatics.as`） |
| 区分 | 独立 `GravityWhip/` `GravityBlade/` | Skyline/GravityWeapons | 步行版鞭/刀，与 `Spline/Whip` `Spline/Blade` 摩托版**不同实现** |

---

## 关键文件

| 文件 | 作用 |
|---|---|
| `LevelSpecific/Skyline/GravityBike/Free/GravityBikeFree.as` | Free 主载具 Actor（`AGravityBikeFree : AHazeActor`），重力对齐/落地/撞墙核心 |
| `LevelSpecific/Skyline/GravityBike/Free/Movement/GravityBikeFreeMovementResolver.as` | Free 移动解算器（继承 `UFloatingMovementResolver`） |
| `LevelSpecific/Skyline/GravityBike/Free/Driver/GravityBikeFreeDriverComponent.as` | 按 Mio/Zoe 生成并联机化载具 |
| `LevelSpecific/Skyline/GravityBike/Free/Capabilities/GravityBikeFreeSteeringCapability.as` | 转向能力（速度相关转向角、漂移） |
| `LevelSpecific/Skyline/GravityBike/Free/Weapons/MachineGun/GravityBikeMachineGun.as` | Free 车载机枪 |
| `LevelSpecific/Skyline/GravityBike/Spline/GravityBikeSpline.as` | Spline 主载具 Actor（在轨、驾驶员+乘客、自动瞄准、样条切换） |
| `LevelSpecific/Skyline/GravityBike/Spline/Steering/GravityBikeSplineSteeringComponent.as` | 样条转向参考解算 |
| `LevelSpecific/Skyline/GravityBike/Spline/Enemies/Bike/GravityBikeSplineBikeEnemy.as` | 敌方摩托（能力表 + 状态机） |
| `LevelSpecific/Skyline/GravityBike/Spline/Whip/GravityBikeWhip.as` | 追逐战乘客用重力鞭 |
| `LevelSpecific/Skyline/GravityBike/Spline/Blade/GravityBikeBlade.as` | 追逐战乘客用重力刀 |
| `LevelSpecific/Skyline/GravityBike/Shared/GravityBikeWheelComponent.as` | Free/Spline 共用车轮组件 |
</content>

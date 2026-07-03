# Gameplay / AI / Basic（单体 AI 核心框架）

> 职责：单个敌人的完整实现——组件容器、行为决策中枢、行为树、攻击体系、多形态移动、生命/伤害、感知/瞄准。AI 模块最大最核心的部分，共 114 文件，分 12 个子目录。

## 一、目录结构（子系统）

| 子目录 | 文件数 | 职责 |
|---|---|---|
| （根） | 5 | `ABasicAICharacter` 及三个形态变体、`BasicAIUpdateCapability` |
| Behaviour | 5 + 11 子目录 | 行为决策中枢 + 全部具体行为（下详） |
| Attacks | 9 | 武器/近战/投射物/发射器/开火模式 |
| Movement | 15 | 地面/飞行/爬墙/样条/传送移动能力 + 目的地/飞行组件 |
| Health | 10 | 生命/伤害/击倒/死亡/血条 |
| Perception | 6 | 目标选择、视线、目标轨迹 |
| Animation | 6 | 动画特性、AnimNotify、开枪叠加动画 |
| Effects | 4 | 伤害/投射物/武器事件特效 |
| Networking | 3 | 控制端切换、匹配目标控制端 |
| Settings | 2 | `UBasicAISettings`、传送设置 |
| ResourceHandling | 1 | 资源管理器 |

Behaviour 下的 11 个子目录（具体行为，均为 `UBasicBehaviour`）：`Attacking`、`CompoundBehaviours`、`Entrance`、`Fleeing`、`Gentleman`、`Idling`、`Maneuvering`、`Perception`、`Reactions`、`Recovering`、`Stunned`。

## 二、ABasicAICharacter 组件构成

`Gameplay/AI/Basic/BasicAICharacter.as` 中 `ABasicAICharacter : AHazeCharacter`（Abstract），是所有敌人的基类。碰撞按「频繁移动物」优化：`bGenerateOverlapEvents = false`、胶囊用 `EnemyIgnoreCharacters`、Mesh 无碰撞。

组件（见总览 README 表）核心九件：`BehaviourComponent`、`TargetingComponent`、`AnimComp`、`DestinationComp`、`HealthComp`、`RespawnComp`、`DisableComp`（`AutoDisableRange=30000`）、`VoiceComp`、`CapabilityComp`。

默认能力五件：`BasicAIUpdateCapability`、`BasicAIDeathCapability`、`BasicAIAnimationMovementCapability`、`BasicAIRequestOverrideFeatureCapability`、`BlockBehavioursWhenControlledByCutsceneCapability`。

`BeginPlay` 绑定 `HealthComp.OnDie/OnTakeDamage` 转发到蓝图事件；`BlockBehaviour/UnblockBehaviour` 用 `BasicAITags::Behaviour` 标签整体阻塞行为能力（如过场控制时）。

### 三个形态变体

| 变体 | 追加能力/组件 | 说明 |
|---|---|---|
| `ABasicAIFlyingCharacter` | `FlyingPathfollowingCapability`、`BasicAIFlyingMovementCapability`、`BasicAIFlyAlongSplineMovementCapability` + `UBasicAIFlightComponent` + `MoveToComp` | 飞行敌，`BaseMovementTag = Flying`，重生时重置飞行组件 |
| `ABasicAIGroundMovementCharacter` | 地面移动能力 | 普通地面敌 |
| `ABasicAIWallclimbingCharacter` | 爬墙移动能力 | 沿墙面移动敌 |

变体只是「换一套移动能力 + 移动组件」，行为决策/攻击/生命完全共享——这是组件化的直接收益。

## 三、UBasicBehaviourComponent 行为决策中枢

`Gameplay/AI/Basic/Behaviour/BasicBehaviourComponent.as` 是 AI 的「大脑仲裁器」。它本身不含具体逻辑，而是管理**需求资源的抢占**，让众多行为能力互斥有序。

### 需求资源仲裁（核心机制）

`BasicBehaviourTypes.as` / `BasicBehaviourRequirements.as` 定义 5 种可抢占资源（位标志）：

```
enum EBasicBehaviourRequirement { Movement, Focus, Animation, Weapon, Perception }
```

- 每个 `UBasicBehaviour` 声明自己需要哪些资源（`Requirements.Add`）以及优先级（`Priority`，默认 `1000 - 注册序号`，越早注册越高）。
- 激活前调 `CanClaimRequirements(flags, prio, instigator)`：按优先级降序遍历已占用记录，若更高优先级者已占用所需资源则不能激活。
- 激活时 `ClaimRequirements` 抢占（含 `RequirementsBlocks`：占而不用，仅阻止更低优先级者）。
- 每个 Instigator 只能占一格（`ReleaseRequirements` 释放）。

这保证了：一个敌人同一时刻只能有一个行为占用「移动」，不会边攻击边逃跑；高优先级行为（如被击倒 Stunned）能抢占低优先级（如巡逻）。

### 生命周期与联机

- `BeginPlay`：加入队伍（`TeamName`/`TeamClass`，默认 `AITeams::Default`），应用 `DefaultSettings`。
- `OnActorDisabled`：若已死则 `Unspawn()`（广播 `OnUnspawn`，对接生成池回收）；并阻塞所有 `Behaviour`/`CompoundBehaviour` 标签能力。
- `OnActorEnabled`：解除阻塞、重新标记 spawned。
- `Reset`（重生时由 `BasicAIUpdateCapability` 绑定调用）。
- 事件：`OnUnspawn`/`OnEnabled`/`OnDisabled`。

### 状态更新驱动

`BasicAIUpdateCapability.as`（Tag `Update`，`TickGroup=BeforeGameplay`，即在评估行为能力之前）：`Setup` 时把重生组件的 `OnRespawn` 绑定到 Behaviour/Health/Destination/Anim 的 `Reset`；每帧 `TickActive` 调 `DestinationComp.Update()` + `AnimComp.Update()`，刷新意图缓冲供下游能力消费。

### UBasicBehaviour 基类

`Behaviour/BasicBehaviour.as`，`UHazeChildCapability` 派生（作为复合能力的子节点运行）：
- `ShouldActivate`：冷却结束 + 能抢到需求资源。
- `ShouldDeactivate`：冷却被设置 或 抢不到资源。
- `OnActivated/OnDeactivated`：抢占/释放需求 + 重置冷却 + 清动画特性。
- `Setup` 缓存 6 个协作组件（Behaviour/Target/Destination/Anim/Perception/Settings）。

## 四、行为树（CompoundBehaviours）

复杂敌人挂一个 `UHazeCompoundCapability`，在 `GenerateCompound()` 里用 Core 复合节点把行为编排成树。以 `CompoundBehaviours/BasicBehaviourMeleeCompoundCapability.as` 为例：

```
Selector
├─ Try RunAll(Stunned)   : Knockdown, HurtReaction
├─ Try RunAll(Combat)    : Sequence[StatePicker(Finisher/Dual/Single 近战) → FallBack],
│                          CircleAdvance, Chase, GentlemanCircle, TrackTarget, RaiseAlarm
└─ Try RunAll(Idle)      : FindTarget, ScenepointEntrance, Roam
```

`RunAll` 内的子行为靠**需求资源仲裁**天然互斥；`StatePicker` 在多种攻击间挑选。各行为按目录归类：

| 行为目录 | 代表 | 作用 |
|---|---|---|
| Attacking | `BasicMeleeAttackBehaviour`、`BasicMeleeChargeBehaviour`、`BasicSingleRangedAttackBehaviour` | 近战/冲锋/远程攻击 |
| Maneuvering | `BasicChaseBehaviour`、`BasicCircleStrafeBehaviour`、`BasicCrowdEncircleBehaviour`、`BasicEvadeBehaviour`、`BasicShuffleBehaviour`、`BasicFitnessCircleStrafeBehaviour` +飞行/爬墙变体 | 追击、环绕、围攻、闪避、走位 |
| Perception | `BasicFindTargetBehaviour`、`BasicFindPriorityTargetBehaviour`、`BasicFindProximityTargetBehaviour`、`BasicFindBalancedTargetBehaviour`、`BasicRaiseAlarmBehaviour` | 找目标、报警 |
| Gentleman | `BasicGentlemanCircleBehaviour`、`BasicGentlemanWaitBehaviour` | 未持攻击令牌时的环绕/等待 |
| Idling | `BasicRoamBehaviour`、`BasicFlyingRoamBehaviour` | 巡逻游荡 |
| Entrance | `BasicAIEntranceAnimationBehaviour`、`BasicScenepointEntranceBehaviour`、`BasicSplineEntranceBehaviour` | 登场动画/入场 |
| Fleeing | `BasicStartFleeingBehaviour`、`BasicFleeAlongSplineBehaviour` | 逃跑 |
| Recovering | `BasicFallBackBehaviour`、`BasicRestBehaviour`、`BasicTeleportRetreatBehaviour`、`BasicFlyingDriftBehaviour` | 撤退/休整 |
| Stunned | `BasicKnockdownBehaviour`、`BasicHurtReactionBehaviour`、`BasicFallingBehaviour`、`BasicPushedBackBehaviour` | 受击硬直 |
| Reactions | `BasicSpotTargetReactionBehaviour` | 发现目标反应 |

近战攻击行为示例（`Attacking/BasicMeleeAttackBehaviour.as`）：声明需要 `Weapon/Movement/Focus`；`ShouldActivate` 检查武器冷却、有效目标、随机攻击概率、在 `AttackRange` 内；`OnActivated` 调 `AnimComp.RequestFeature(MeleeCombat, ...)` 播动画；持续超 `AttackDuration` 后设武器与行为冷却。

## 五、攻击体系（Attacks）

| 文件 | 作用 |
|---|---|
| `BasicAIWeapon.as` | `ABasicAIWeapon`（Abstract）武器演员基类，带附着 socket 覆盖 |
| `BasicAIWeaponWielderComponent.as` | 持械者组件 |
| `BasicAIMeleeWeaponComponent.as` | 近战武器（含冷却），被近战行为查询 |
| `BasicAIProjectile.as` / `BasicAIProjectileComponent.as` | 投射物演员/组件 |
| `BasicAIHomingProjectileComponent.as` | 追踪弹 |
| `BasicAIProjectileLauncherComponent.as` / `BasicAINetworkedProjectileLauncherComponent.as` | 本地/联网发射器 |
| `AIFirePattern.as` | 开火模式：`FAIFirePattern`（每发间隔数组）+ `UAIFirePatterns`（数据资产）+ `UAIFirePatternManager`（洗牌不重复消费） |

**开火模式复用**：`UAIFirePatternManager` 挂在关卡脚本演员（或退化到玩家）上，被同类敌人**共享**——洗牌后依次消费模式、用尽再重洗，避免全场敌人同步开火、又保证多样性。

## 六、移动体系（Movement）

意图缓冲设计：行为**每帧向 `UBasicAIDestinationComponent` 写入**移动/朝向意图（`Focus`、`Speed`，每帧清零，故行为需持续写），移动能力**消费**这些意图执行实际位移。`FHazeBasicAITarget` 封装「盯着一个演员或一个世界坐标」（带世界/局部偏移）。

移动能力按形态分：地面（`BasicAIGroundMovementCapability`、`BasicAISimpleGroundMovementCapability`）、飞行（`BasicAIFlyingMovementCapability` + `UBasicAIFlightComponent`）、爬墙（`BasicAIWallclimbingMovementCapability`）、样条（`BasicAIClimbAlongSplineMovementCapability`、`BasicAIFlyAlongSplineMovementCapability` + `UBasicAIRuntimeSplineComponent`）、传送（`BasicAIAnimationTeleportingMovementCapability`、`BasicAITeleportAlongRuntimeSplineCapability`）、动画位移（`BasicAIAnimationMovementCapability`，根运动）。

## 七、生命与感知

**Health**（`Health/BasicAIHealthComponent.as`）：`MaxHealth` 默认 1（偏好调伤害而非血量）。`TakeDamage` 仅在 Instigator 控制端生效、经 `CrumbTakeDamage` 联机同步、受 `TakeDamageCooldown` 限频。死亡分阶段：`OnStartDying`（死亡不可逆时）→ `OnDie`；远端提前 `OnRemotePreDeath` 做视觉。支持 `bImmortal`（血量卡在 0.001）、`SetStunned`、`SetInvulnerable`。事件丰富：`OnTakeDamage/OnHealthChange/OnStartDying/OnDie`。其余：击倒(`BasicAIKnockdownComponent`)、死亡能力(`BasicAIDeathCapability`)、血条(`HealthBarWidget`/`BossHealthBarWidget`)、死亡追踪(`DeathTrackerActor`)。

**Perception**（`Perception/`）：`UBasicAITargetingComponent` 是目标核心——潜在目标列表（默认双人玩家 + 手配非玩家）、`SetTarget`（联机 Crumb）、视线检测限频缓存（`HasVisibleTarget`/`HasGeometryVisibleTarget` 经 `PerceptionComp.Sight.VisibilityExists`）、警戒目标（`RaiseAlarm`/`FindAlarmTarget`）、`FindClosestTarget`/`FindAllTargets`。**切换目标时自动桥接绅士组件**：从旧目标移除、加到新目标的 `UGentlemanComponent` 作为 opponent（见 Gentleman/）。`UBasicAIPerceptionComponent` 只是持有 `Sight`；`SeeThroughPlayersPerceptionSight` 可穿透玩家视线；`TargetTrailComponent` 记目标轨迹。

## 八、数据流（感知 → 决策 → 攻击/移动）

```
每帧 BeforeGameplay:
  BasicAIUpdateCapability.TickActive
    └─ DestinationComp.Update() + AnimComp.Update()   // 清空意图缓冲

感知:  TargetingComponent 选目标 + 视线检测（限频缓存）
                │  切目标时挂接 Gentleman 组件
                ▼
决策:  CompoundCapability 行为树逐节点评估
         每个 UBasicBehaviour.ShouldActivate:
           冷却结束? + BehaviourComponent 抢到需求资源(Movement/Weapon/Focus...)?
                │ 激活胜出行为
                ▼
执行:  ├─ 攻击行为 → AnimComp.RequestFeature(播攻击动画) + Weapon/FirePattern
       └─ 移动行为 → DestinationComp.MoveTowards/RotateTowards(写意图)
                          ▼
       Movement 能力消费 DestinationComp 意图 → 实际位移(寻路/飞行/爬墙)

伤害:  玩家攻击 → HealthComp.TakeDamage(Crumb 同步) → OnStartDying → OnDie
                          ▼
       BasicAIDeathCapability + OnActorDisabled → Unspawn → 生成池回收
```

## 关键文件

- `BasicAICharacter.as`：`ABasicAICharacter` 敌人基类（组件容器 + 默认能力）。
- `BasicAIFlyingCharacter.as` / `BasicAIGroundMovementCharacter.as` / `BasicAIWallclimbingCharacter.as`：三形态变体（换移动能力集）。
- `BasicAIUpdateCapability.as`：`Update` 标签能力，驱动状态刷新 + 绑定重生 Reset。
- `Behaviour/BasicBehaviourComponent.as`：行为决策中枢，需求资源抢占仲裁。
- `Behaviour/BasicBehaviour.as`：行为能力基类（`UHazeChildCapability`）。
- `Behaviour/BasicBehaviourRequirements.as` / `BasicBehaviourTypes.as`：需求资源位标志、优先级、冷却。
- `Behaviour/CompoundBehaviours/BasicBehaviourMeleeCompoundCapability.as`：近战敌行为树范例。
- `Attacks/AIFirePattern.as`：开火模式（共享、洗牌消费）。
- `Attacks/BasicAIWeapon.as`：武器演员基类。
- `Movement/BasicAIDestinationComponent.as`：移动/朝向意图缓冲 + `FHazeBasicAITarget`。
- `Health/BasicAIHealthComponent.as`：生命/伤害/死亡（Crumb 同步、分阶段死亡）。
- `Perception/BasicAITargetingComponent.as`：目标选择、视线、警戒、绅士组件桥接。

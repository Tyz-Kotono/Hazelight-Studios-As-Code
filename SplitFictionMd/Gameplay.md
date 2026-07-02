# Gameplay — 通用玩法系统

> 路径：`SplitFiction/Gameplay`（831 文件）
>
> 跨关卡复用的玩法机制。

---

## AI/ — 敌人 AI 框架

敌人 AI 框架：角色、行为状态机、感知、攻击、血量、移动协调。基于 `BasicAICharacter` + 行为/能力树。

| 子目录 | 功能 |
|---|---|
| **Basic** | 核心 AI：角色、行为状态机、攻击、血量、移动、特效（`BasicAICharacter`, `BasicBehaviour`） |
| **Gentleman** | 攻击令牌协调，让多个 AI 公平地同步攻击（"绅士规则"）（`UGentlemanComponent`） |
| **Balancing** | 难度调节，在血量阈值降低攻击令牌（`UBalancingSettings`） |
| **Fitness** | 评分哪个 AI/目标最适合执行某动作（如走位）（`UFitnessUserComponent`） |
| **ScenePoints** | 放置的位置标记，AI 用于入场/站位（`AScenepointActor`） |
| **Spawning** | actor 生成池、生成器、重生、生成模式（`HazeActorSpawner*`） |
| **Traversal** | AI 在区域间用特殊方式穿越（`TraversalAreaActor`, `TraversalManager`） |
| **Weakpoints** | AI 身上可破坏的弱点部位（`UBasicAIWeakpointComponent`） |
| **Audio** | AI 死亡追踪、群控音频（`AIEnemyGlobalDeathTracker`） |
| **VoiceOver** | AI 语音/吼叫（`BasicAIVoiceOverComponent`） |
| **Volumes** | AI 触发/击杀体 |
| **Debug** | AI 调试菜单与显示 |

---

## Movement/ — 玩家移动玩法层

| 子目录 | 功能 |
|---|---|
| **Player** | 主体：移动组件、空中冲刺/跳跃、蹲伏、相机、发射、输入（`UPlayerMovementComponent`） |
| **Audio** | 基于射线的脚步/植被/滑行移动音频（`PlayerFootstepTraceCapability`） |
| **Helpers** | 预测与摇杆轻拨工具 |
| **ResolverExtensions** | 移动解算器扩展（圆形约束、样条碰撞） |
| **Triggers** | 移动状态触发体（`GroundedOnlyPlayerTrigger`） |
| **Tutorials** | 单招式移动教程提示（`PlayerAirDashTutorialCapability`） |

---

## Combat/ — 战斗系统

伤害系统、受击反应、战斗标签/设置（`DamageStatics`, `DamageTypes`）。

| 子目录 | 功能 |
|---|---|
| **ContextualCombat** | 情境战斗激活点的目标选择（`ContextualCombatTargetingCapability`） |
| **HitStop** | 命中时的帧冻结/打击停顿（`UCombatHitStopComponent`） |
| **PlayerHurtReactions** | 玩家受伤反应：加法击中、击倒、踉跄（各含 capability + component + statics） |
| **Tags** | 在特定状态下门控战斗的游戏标签（`PlayerCombatTags`） |

---

## Pickups/ — 拾取系统

拾取/搬运/放下物体。

| 子目录 | 功能 |
|---|---|
| **BaseActors** | 基础拾取/放下 actor（`APickupBase`） |
| **Capabilities** | 拾取/放下/移动逻辑 |
| **Components** | 物体与玩家上的拾取状态 |
| **Data** | 容器与标签 |
| **Visualizers** | 编辑器放下可视化器 |

---

## 其它玩法模块

| 子模块 | 功能 |
|---|---|
| **Navigation** | AI 寻路与路径跟随（地面/飞行/攀墙）。含 Pathfollowing（跟随计算路径）、Wallclimbing（可攀爬墙导航体 `UWallclimbingComponent`, `AWallClimbingNavigationVolume`） |
| **FauxPhysics** | 轻量级伪物理运动（力/旋转/平移/样条/弹簧），非真实模拟（`UFauxPhysicsForceComponent`）。含 Components（各效果驱动器）、Constraints（弹簧约束） |
| **KineticActors** | 移动/旋转/样条跟随平台，带网络同步与编辑器可视化（`AKineticMovingActor`） |
| **Physics** | 网络物理对象，控制端切换 + 位置同步（`ANetworkedPhysicsObject`） |
| **Pullable** | 玩家沿样条拉动的物体（单/双玩家）（`APullable`, `PullableComponent`） |
| **Projectiles** | 抛射物近距检测（擦身/闪避事件）（`UProjectileProximityDetectorComponent`） |
| **Destructible** | 烘焙顶点动画破坏道具（由材质帧驱动）（`ABakedDestructionActor`） |
| **SideInteractions** | 可选小游戏/支线：BrothersBench（兄弟长椅）、SkippingStones（打水漂）、TitanicBoat（各含完整 capability/component 集） |
| **SideGlitch** | 横版"故障"/传送门交互序列与巨人布景（`ASideGlitchInteractionActor`） |
| **AnimationInteractions** | 可放置交互物，交互时播放一次性/三段动画 + 效果事件（`AOneShotInteractionActor`） |
| **GamepadLight** | 由玩家颜色/血量状态驱动 DualSense 手柄灯色（`UPlayerGamepadLightCapability`） |
| **TopDownPlayerArrow** | 俯视视角下指示玩家位置的屏幕箭头（`ATopDownPlayerArrow`） |
| **Accessibility** | 玩家辅助选项（自动跳跃/冲刺、TTS 聊天），读取 CVar 自动化或辅助输入（`UAccessibilityAutoJumpDashCapability`） |
| **Audio** | 玩家语音触发体，跟踪进入/死亡/重生以播放 VO（`APlayerVOTriggerVolume`） |
| **PlayerHealth** | 玩家伤害/死亡组件与世界内血条（`DamageTriggerComponent`, `DeathOnImpactComponent`） |
| **Triggers** | 玩家触发体：VO、看向、震动、移动区、不可行走区（`PlayerLookAtTrigger`, `PlayerWorldRumbleTrigger`, `Vox*Trigger`） |

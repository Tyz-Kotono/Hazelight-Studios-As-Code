# Gameplay — 通用玩法系统

> 路径：`Gameplay/`（831 文件，20 个子模块）
>
> 建立在 [Core](../Core/README.md) 框架之上的**跨关卡复用玩法层**。Core 提供引擎级能力（移动求解、相机、联机…），Gameplay 在其上实现**具体的可玩内容**：玩家的每一个动作、敌人的每一种行为、拾取/战斗/物理互动等。
>
> 本目录仿 Core 采用**功能域 / 模块**两级结构，每模块含内部架构 + 数据流 + 协同分析。

---

## 先读：设计哲学

> 📐 **[DesignPhilosophy.md](./DesignPhilosophy.md) —— 先读这篇。** Gameplay 层的三大范式：Move 系统（能力四件套 + 标签矩阵）、AI 角色框架（组件容器 + 行为能力）、以及它们如何全部落在 Core 的"能力+确定性联机"地基上。

---

## 体量分布（关键认知）

Gameplay 的 831 文件高度集中在两大核心，占 **83%**：

| 子模块 | 文件数 | 占比 | 定位 |
|---|---:|---:|---|
| **Movement** | 468 | 56% | 玩家移动与所有动作（Move 系统） |
| **AI** | 218 | 26% | 敌人 AI 框架 |
| SideInteractions | 24 | | 支线小游戏 |
| Combat | 19 | | 伤害/受击/战斗标签 |
| Navigation | 17 | | AI 寻路/攀爬 |
| FauxPhysics | 15 | | 伪物理运动 |
| Pickups | 13 | | 拾取/搬运 |
| AnimationInteractions | 11 | | 动画交互物 |
| Triggers | 10 | | 玩法触发体 |
| Pullable | 8 | | 可拉动物 |
| 其余 10 个 | 各 1-6 | | 见下方索引 |

> **理解 Gameplay = 先理解 Movement 的 Move 系统 + AI 的角色框架。** 其余模块都是同一套能力范式的小规模应用。

---

## 模块索引

### 🏃 [Movement/](./Movement/) — 玩家移动与动作（468 文件）
玩家的全部动作能力。核心是 `Player/Moves`（166，基础动作）+ `Player/MovesContextual`（165，情境动作）。
[Movement 总览](./Movement/) · [Player/Moves](./Movement/Moves/) · [Player/MovesContextual](./Movement/MovesContextual/) · [Player 支撑层](./Movement/Player/) · [Audio](./Movement/Audio/) · [ResolverExtensions](./Movement/ResolverExtensions/) · [Tutorials](./Movement/Tutorials/)

### 🤖 [AI/](./AI/) — 敌人 AI 框架（218 文件）
核心是 `Basic`（114，AI 角色 + 行为 + 攻击）。
[AI 总览](./AI/) · [Basic](./AI/Basic/) · [Spawning](./AI/Spawning/) · [Traversal](./AI/Traversal/) · [Gentleman](./AI/Gentleman/) · [ScenePoints](./AI/ScenePoints/) · [Fitness](./AI/Fitness/) · [Balancing/Weakpoints/Audio/VoiceOver/Volumes](./AI/Support/)

### ⚔️ 玩法机制模块
| 模块 | 文档 | 说明 |
|---|---|---|
| Combat | [Combat/](./Combat/) | 伤害系统、受击帧冻结、玩家受伤反应、战斗标签 |
| Pickups | [Pickups/](./Pickups/) | 拾取/搬运/放下 |
| Pullable | [Pullable/](./Pullable/) | 沿样条拉动物体 |
| FauxPhysics | [FauxPhysics/](./FauxPhysics/) | 轻量伪物理（力/旋转/弹簧） |
| KineticActors | [KineticActors/](./KineticActors/) | 移动/旋转平台 |
| Destructible | [Destructible/](./Destructible/) | 烘焙顶点动画破坏 |
| Projectiles | [Projectiles/](./Projectiles/) | 抛射物近距检测 |
| AnimationInteractions | [AnimationInteractions/](./AnimationInteractions/) | 一次性动画交互物 |
| SideInteractions | [SideInteractions/](./SideInteractions/) | 支线小游戏（BrothersBench/SkippingStones/TitanicBoat） |
| SideGlitch | [SideGlitch/](./SideGlitch/) | 横版故障/传送门 |

### 🔧 其余小模块 → [MiscModules.md](./MiscModules.md)
Navigation · Triggers · PlayerHealth · Physics · Accessibility · GamepadLight · Audio · TopDownPlayerArrow

---

## 与 Core 的关系

```
Core（引擎地基）                    Gameplay（玩法实现）
─────────────────                  ──────────────────────
UHazeMovementComponent      ←────  Movement/Player/Moves 的每个动作
  + Resolver 管线                    调 PrepareMove/ApplyMove 做位移
UHazeCapability 生命周期     ←────  每个 Move / AI 行为都是一个能力
UTargetableComponent        ←────  MovesContextual 情境激活点
Crumb / HasControl          ←────  AI 攻击、玩家动作的联机同步
UInteractionComponent       ←────  Pickups / AnimationInteractions
UDisableComponent           ←────  AI 远距自动禁用
```

**Gameplay 不发明新范式，它是 Core 能力框架的大规模具体应用。** 想理解某个玩法为何这样写，先看 [Core/DesignPhilosophy](../Core/DesignPhilosophy.md)。

# Gameplay / AI（敌人与智能体）

> 职责：Split Fiction 所有非玩家敌人的完整框架。承接 `Gameplay/DesignPhilosophy.md` 的组件化哲学——AI 角色与玩家同构（组件容器 + 能力清单），但额外叠加了一整套「多智能体协调层」来编排群体敌人的攻击、站位与难度。共 218 文件，Gameplay 第二大模块，分 12 个子目录。

## 一、总览与哲学承接

AI 遵循与玩家一致的范式：**演员本身很薄，只是一堆组件的容器；行为逻辑全部拆进 Capability（能力）**。差异只在于：
- 玩家的能力由输入驱动；AI 的能力由感知与决策驱动。
- 玩家是单体；AI 是群体，因此需要玩家侧不存在的**协调层**（谁能攻击、站哪里、放多难）。

因此本模块可分为两大板块：

| 板块 | 子目录 | 说明 |
|---|---|---|
| **单体 AI 框架** | Basic(114) | 单个敌人的组件、能力、行为树、攻击、移动、生命 |
| **多智能体协调层** | Gentleman(14)、Fitness(5)、ScenePoints(10)、Balancing(2)、Spawning(37)、Traversal(20) | 群体层面的令牌、打分、站位、难度、生成、穿越 |
| **周边支持** | Audio(4)、VoiceOver(3)、Volumes(4)、Weakpoints(2)、Debug(2) | 音频、语音、触发盒、弱点、调试 |

## 二、12 子目录关系图

```
                     ┌─────────────────────────────────────────┐
                     │            ABasicAICharacter             │
                     │  (Basic：组件容器 + 能力清单，见 Basic/)  │
                     └───────────────┬─────────────────────────┘
                                     │ 感知→决策→攻击/移动
          ┌──────────────┬──────────┼───────────┬──────────────┐
          ▼              ▼          ▼           ▼              ▼
      Gentleman       Fitness   ScenePoints  Balancing     Traversal
      (攻击令牌)      (站位打分)  (站位标记)   (难度令牌)    (区域穿越)
          │              │          │           │
          └──── 都挂在「目标」(玩家/队伍)身上，协调多个 AI ────┘

  Spawning(生成池/波次) ──创建/回收──▶ ABasicAICharacter
  Audio/VoiceOver/Volumes/Weakpoints/Debug ──▶ 周边支持（见 Support/）
```

- **Basic** 是核心：单个敌人。其余目录都是围绕 Basic 的横切系统。
- **Gentleman / Fitness / ScenePoints / Balancing**：协调层，通常以组件形式挂在**目标**（玩家或队伍）上，被多个 AI 共享查询——「同一时刻允许几个 AI 攻击你」「哪个站位最优」「你血量低了就少放几个敌人」。
- **Spawning**：AI 的生命周期工厂——生成池复用演员、生成器 + 波次模式控制何时生成多少。
- **Traversal**：AI 在断裂区域间的特殊移动（弧线跳、抛物线、传送）。

## 三、AI 角色框架（组件容器 + 能力清单，对比玩家）

以 `Gameplay/AI/Basic/BasicAICharacter.as` 的 `ABasicAICharacter : AHazeCharacter` 为例，其组件构成（`DefaultComponent`）：

| 组件 | 职责 | 玩家对应 |
|---|---|---|
| `UBasicBehaviourComponent` | 行为决策中枢：注册行为、按优先级仲裁「需求资源」（移动/朝向/动画/武器/感知） | 无（玩家由输入直接驱动） |
| `UBasicAITargetingComponent` | 目标选择与可见性（视线检测、警戒目标、绅士组件桥接） | `UAimingComponent` 大致对应 |
| `UBasicAIAnimationComponent` | 动画特性请求 | 玩家 Locomotion |
| `UBasicAIDestinationComponent` | 移动/朝向意图缓冲（每帧由行为写入，移动能力消费） | 玩家 MoveTo |
| `UBasicAIHealthComponent` | 生命/伤害/死亡（联机 Crumb 同步） | `UPlayerHealthComponent` |
| `UHazeActorRespawnableComponent` | 重生/回收（对接生成池） | 玩家复活 |
| `UDisableComponent` | 远距自动禁用（默认 30000 范围） | 同用（Core/Flow/Disable） |
| `UBasicAIVoiceOverComponent` | 语音 ID 分配 | 玩家 Vox |
| `UHazeCapabilityComponent` | 能力容器 | 同用 |

默认能力清单（`CapabilityComp.DefaultCapabilities`）：`BasicAIUpdateCapability`（驱动状态更新）、`BasicAIDeathCapability`、`BasicAIAnimationMovementCapability`、`BasicAIRequestOverrideFeatureCapability`、`BlockBehavioursWhenControlledByCutsceneCapability`。复杂敌人再叠加一个 `UHazeCompoundCapability`（行为树，见 Basic/）。

**关键区别**：玩家能力各自独立跑；AI 的行为能力（`UBasicBehaviour`）必须先通过 `BehaviourComponent` 抢占「需求资源」才能激活，从而保证一个敌人同一时刻不会同时「攻击 + 逃跑」。详见 Basic/README。

## 四、多智能体协调层概述

玩家没有、AI 独有的系统，用于把「一群各自为战的敌人」编排成「有章法的战斗」：

| 系统 | 机制 | 挂载位置 | 详见 |
|---|---|---|---|
| **Gentleman（绅士规则）** | 令牌信号量：目标身上有限数量的「攻击令牌」，AI 须先抢到令牌才能进攻，实现「围而不攻、轮流上」 | 目标（玩家/队伍） | Gentleman/ |
| **Fitness（站位打分）** | 对候选站位打分（视野、拥挤度），选最优环绕方向 | AI + 目标 | Fitness/ |
| **ScenePoints（站位标记）** | 关卡里手放的站位/入场/动画点，AI 申领占用 | 关卡 | ScenePoints/ |
| **Balancing（难度令牌）** | 按玩家血量动态增减令牌上限，血少则少放敌人 | 玩家 | Support/ |
| **Spawning（生成）** | 生成池复用 + 生成器 + 波次模式 | 关卡 | Spawning/ |

这些系统大量复用 Core 的**队伍（HazeTeam）**做群体分组，用 **Instigator/信号量**做资源仲裁。

## 五、与 Core 的关系

| Core 系统 | AI 如何使用 |
|---|---|
| **Disable（按需禁用）** | 每个 AI 自带 `UDisableComponent`，玩家远离时自动禁用省开销；禁用即触发 Unspawn 回收 |
| **Respawn（`UHazeActorRespawnableComponent`）** | 生成池的复用基石：Unspawn 归还、Respawn 重置生命/行为/动画/目标 |
| **复合能力（`UHazeCompoundCapability`）** | AI 行为树的运行时：`UHazeCompoundSelector/RunAll/Sequence/StatePicker` 编排各 `UBasicBehaviour` |
| **Navigation（Pathfinding）** | 移动能力寻路、Traversal 用 `Pathfinding::HasPath` 判定可达 |
| **Team（`UHazeTeam`）** | 队伍分组：AI 队伍、生成器队伍、绅士成本队伍、Traversal 区域队伍均为 `UHazeTeam` 派生 |
| **Settings（`UHazeComposableSettings`）** | `UBasicAISettings` 等成组可覆盖设置 |
| **Instigator / `TInstigated<T>`** | 行为需求抢占、生成器激活、禁用叠加均用 Instigator 语义 |

## 子目录文档索引

- `Basic/README.md` — 单体 AI 核心框架（114 文件）
- `Spawning/README.md` — 生成池 / 生成器 / 波次（37）
- `Traversal/README.md` — 区域间特殊穿越（20）
- `Gentleman/README.md` — 攻击令牌协调（14）
- `ScenePoints/README.md` — 站位标记（10）
- `Fitness/README.md` — 站位打分（5）
- `Support/README.md` — Balancing / Weakpoints / Audio / VoiceOver / Volumes / Debug 合篇（15）

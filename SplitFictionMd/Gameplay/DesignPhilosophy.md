# Gameplay 设计哲学 — Design Philosophy

> 本文回答：Gameplay 层（831 文件）是按什么范式组织的？数据如何流动？核心框架长什么样？
>
> **前提**：Gameplay 完全建立在 [Core 的设计哲学](../Core/DesignPhilosophy.md) 之上——组合优于继承、行为(Capability)与状态(Component)分离、有序帧管线、确定性联机。本文只讲 Gameplay **在此之上新增的三个范式**，不重复 Core 的四大支柱。

---

## 一句话概括

> **Gameplay 把"一个玩家能做的每件事、一个敌人的每种行为"都拆成独立的能力单元，用密集的标签矩阵声明它们之间的互斥关系，让数百个动作在同一个玩家身上无冲突地共存与切换。**

三个核心范式：

1. **Move 系统** —— 每个玩家动作 = 能力四件套（Capability + Component + Settings + Statics）。
2. **标签互斥矩阵** —— 动作之间的"谁能打断谁、谁不能与谁共存"完全用标签声明，而非硬编码判断。
3. **AI 角色框架** —— 敌人 = 组件容器 + 行为能力清单，与玩家角色同构。

---

## 二、范式一：Move 系统 —— 动作即四件套

### 每个动作是一个标准四件套
玩家的每一个动作（空中冲刺、二段跳、蹲伏、抓钩、攀爬…）都是 `Movement/Player/Moves/<动作名>/` 下的一组文件，遵循严格命名：

```
Moves/AirDash/
├── PlayerAirDashCapability.as   ← 行为：激活条件 + 每帧逻辑（继承 UHazePlayerCapability）
├── PlayerAirDashComponent.as    ← 状态：冷却、次数、方向等（继承 UActorComponent）
├── PlayerAirDashSettings.as     ← 参数：速度、时长等（继承 UHazeComposableSettings）
└── PlayerAirDashStatics.as      ← 工具：命名空间静态函数 / mixin
```

- **Capability** 决定"什么时候能冲刺、冲刺时每帧做什么"。
- **Component** 存"还能冲几次、上次冲刺时间"——因为能力停用会丢中间量，需持久的存 Component（Core 支柱二）。
- **Settings** 是可组合参数，关卡/触发体可分层覆盖（Core 支柱四）。
- **Statics** 供其他能力查询/触发本动作。

`Moves`(166 文件) 是基础动作，`MovesContextual`(165 文件) 是**情境动作**——需要世界里有激活点才能做的动作（抓钩点、弹跳点、攀爬面），它们用 `UTargetableComponent`（Core 的可瞄准框架）在世界里注册激活点，玩家靠近时才解锁对应能力。

### 动作走 Core 的移动管线
Move 的 Capability 不自己改位置，而是调 Core 的移动组件：

```cpp
UPlayerMovementComponent MoveComp;      // Core 的移动组件
USteppingMovementData Movement;         // Core 的移动数据类型
// TickActive 里：PrepareMove → 累加速度/重力 → ApplyMove（Resolver 求解写回）
```

**Gameplay 定义"想怎么动"，Core 的 Resolver 管线负责"实际怎么动（碰撞/去穿透）"。** 这条分工是理解整个 Movement 的关键。

---

## 三、范式二：标签互斥矩阵 —— 声明式动作协调

这是 Gameplay 最精妙的部分，也是它区别于普通动作系统的核心。

### 问题
一个玩家身上同时挂着 **50+ 个动作能力**（PlayerCharacter 装配清单）。任一帧只有部分能激活，且它们互相冲突：冲刺时不能攀爬、游泳时不能二段跳、抓钩时不能蹲…… 如果用 `if` 硬判断，就是 O(N²) 的意大利面。

### 解法：三层标签声明
以 `PlayerAirDashCapability` 为例，它用**纯声明**表达自己与所有其他动作的关系：

```cpp
// ① 我是谁——自我分类标签
default CapabilityTags.Add(CapabilityTags::Movement);
default CapabilityTags.Add(PlayerMovementTags::CoreMovement);
default CapabilityTags.Add(PlayerMovementTags::AirDash);

// ② 我在这些状态下不能激活——被阻塞标签矩阵
default CapabilityTags.Add(BlockedWhileIn::WallScramble);
default CapabilityTags.Add(BlockedWhileIn::Swimming);
default CapabilityTags.Add(BlockedWhileIn::Grapple);
default CapabilityTags.Add(BlockedWhileIn::Ladder);
default CapabilityTags.Add(BlockedWhileIn::LedgeGrab);
// …共 11 条

// ③ 例外——即使被阻塞也放行的排除标签
default BlockExclusionTags.Add(n"ExcludeAirJumpAndDash");
```

### `BlockedWhileIn` 命名空间：全局状态词典
`Movement/Player/Tags/BlockedWhileInTags.as` 定义了整套"状态标签"（约 30 个）：

```cpp
namespace BlockedWhileIn {
    const FName Swimming = n"BlockedWhileInSwimming";
    const FName Grapple  = n"BlockedWhileInGrapple";
    const FName Ladder   = n"BlockedWhileInLadder";
    const FName Swing    = n"BlockedWhileInSwing";
    // …约 30 个状态
}
```

当"游泳"能力激活时，它 `BlockCapabilities(BlockedWhileIn::Swimming)`；所有声明了 `BlockedWhileIn::Swimming` 标签的动作**本帧自动失活**——冲刺、二段跳等无需知道"游泳"的存在。

### 为什么这是好设计
- **加新动作不改老动作**：新增"喷气背包"，只要给它打上合适的 `BlockedWhileIn::*` 标签，就自动融入互斥网络。
- **冲突关系集中可读**：一个动作能不能在某状态下做，看它的标签声明即可，不必追踪散落的 `if`。
- **是 Core 支柱三的极致应用**：Core 用标签做能力互斥，Gameplay 把它扩展成一张覆盖数百动作的**声明式状态机矩阵**。

---

## 四、范式三：AI 角色框架 —— 与玩家同构

敌人角色和玩家角色是**同一套骨架**：组件容器 + 能力清单。

`AI/Basic/BasicAICharacter.as`：

```cpp
class ABasicAICharacter : AHazeCharacter
{
    // —— 组件：状态载体 ——
    UBasicBehaviourComponent   BehaviourComponent;   // 行为决策状态
    UBasicAITargetingComponent TargetingComponent;   // 瞄准目标
    UBasicAIHealthComponent    HealthComp;           // 血量
    UBasicAIDestinationComponent DestinationComp;    // 移动目标点
    UHazeActorRespawnableComponent RespawnComp;      // 重生（复用 Core）
    UDisableComponent DisableComp;                   // 远距禁用（复用 Core）
      default DisableComp.AutoDisableRange = 30000.0;

    // —— 能力清单：行为单元 ——
    default CapabilityComp.DefaultCapabilities.Add(n"BasicAIUpdateCapability");
    default CapabilityComp.DefaultCapabilities.Add(n"BasicAIDeathCapability");
    default CapabilityComp.DefaultCapabilities.Add(n"BasicAIAnimationMovementCapability");
    // …
}
```

### AI 的行为组织：Behaviour + Capability
- **`UBasicBehaviourComponent`** 是行为决策中枢（选择当前该做什么：追击/攻击/逃跑）。
- 具体行为仍是**能力**——攻击、移动、死亡各是一个 Capability。
- 复杂 AI 用 Core 的**复合能力**（`UHazeCompoundCapability`，行为树式 Selector/Sequence，见 [Examples](../Examples.md)）组织。

### AI 特有的协调子系统（Gameplay 独有）
玩家只有 2 个（Mio/Zoe），AI 可能几十个同屏，因此 Gameplay 加了玩家层没有的**群体协调**：

| 子模块 | 解决什么 | 机制 |
|---|---|---|
| **Gentleman** (14) | 多个 AI 别一起打玩家 | 攻击令牌协调（"绅士规则"），要攻击先拿令牌 |
| **Fitness** (5) | 哪个 AI 最适合做某动作 | 按站位打分选最优 |
| **Balancing** (2) | 难度调节 | 血量阈值降低攻击令牌数 |
| **ScenePoints** (10) | AI 站位/入场 | 世界里放置的位置标记 |
| **Spawning** (37) | 成批生成/波次 | 生成池 + 模式（Single/Wave/Interval） |

这些是"多智能体"才需要的层，玩家动作系统里没有对应物。

---

## 五、一次玩家动作的完整数据流

以"空中按下冲刺键"为例，串起 Gameplay + Core：

```
Input 阶段（Core）
  采样手柄 → WasActionStarted(Dash) 为真
        ↓
Gameplay 判定（各 Move 能力的 ShouldActivate）
  PlayerAirDashCapability.ShouldActivate():
    · 检查 AirDashComponent 还有冲刺次数吗？（读 Component 状态）
    · 检查我的 BlockedWhileIn 标签是否被当前状态阻塞？（标签矩阵）
    · 都通过 → 返回 true
        ↓
能力激活 + 标签阻塞传播
  OnActivated(): BlockCapabilities(冲突标签) → 攀爬/游泳等本帧失活
  AirDashComponent.RemainingDashes--   （写回 Component 状态）
        ↓
ActionMovement 阶段（Core 移动管线）
  TickActive(): PrepareMove → AddVelocity(冲刺方向) → ApplyMove
  → Core 的 Resolver 扫掠求解、处理墙壁碰撞 → 写回 transform
        ↓
【控制端】以上真实计算；【远端】NetworkMode=Crumb 回放（Core 支柱四）
        ↓
后续阶段：相机跟随（Core）、冲刺音效（Movement/Audio）、拖尾特效
```

关键：**Gameplay 负责"决策与意图"（能不能冲、往哪冲、消耗次数），Core 负责"执行与一致性"（碰撞求解、联机回放）。**

---

## 六、总结

Gameplay 层在 Core 的能力框架上，长出三个自己的范式：

1. **Move 四件套** —— 每个动作是自包含的 Capability+Component+Settings+Statics，可独立开发、自由装配到玩家身上。
2. **标签互斥矩阵** —— 数百个动作用 `BlockedWhileIn::*` 声明式表达互斥，加新动作不碰老动作。
3. **AI 同构框架** —— 敌人复用"组件容器+能力清单"骨架，额外加多智能体协调层（令牌/打分/生成）。

三者都没有背离 Core：Move 走 Core 的移动管线、AI 复用 Core 的 Disable/Respawn、一切状态变更走 Crumb。**Gameplay 是 Core 哲学的大规模验证——数百个能力、数千次标签声明，证明了"组合 + 声明式互斥 + 确定性联机"这套范式可以撑起一整个游戏的可玩内容。**

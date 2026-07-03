# Gameplay / AI / Gentleman（攻击令牌协调机制）

> 职责：多智能体协调层的核心——「绅士规则」。同一时刻允许多少个 AI 攻击同一目标由「令牌信号量」控制：AI 必须先向目标身上的绅士组件抢到令牌才能进攻，抢不到就环绕/等待。这就是敌人「围而不攻、轮流上前」观感的来源。共 14 文件（含 Queue 子目录 4 文件）。

> 命名梗：代码里满是维多利亚式绅士用语（"Queensberry rules"、"Pip pip!"、"one's place in society"）——绅士决斗规矩，敌人不能一拥而上。

## 一、核心：UGentlemanComponent（令牌信号量）

`Gentleman/GentlemanComponent.as`。**挂在目标（玩家或队伍）身上**，被围攻它的多个 AI 共享查询。

### 令牌（Token）信号量
- `TokenSemaphore: TMap<FName, FGentlemanSemaphore>`：每种令牌一个信号量，记录各 claimant 占了几个。
- `MaxAllowedClaimants: TMap<FName, int>`：每种令牌的上限。
- 核心 API：`IsTokenAvailable` / `CanClaimToken` / `ClaimToken` / `ReleaseToken`（可带冷却）。
- **冷却对象池**：`ReleaseToken(Cooldown>0)` 时用一个 `UGentlemanCooldown` 占位对象继续占着令牌，到点在 `Tick` 里释放并回收——实现「攻击后一段时间内令牌不立即回流」，制造攻击节奏。

令牌类型（`GentlemanActions.as`）：`Cost`、`Ranged`、`Melee`、`Attack`。近战/远程有不同「分值」（`GentlemanScore::Melee=2.0`、`Shooting=1.0`）。

### 对手追踪
- `Opponents: TArray<AHazeActor>`：谁正在盯着我打。由 `UBasicAITargetingComponent.SetTargetLocal` 在切目标时自动 `AddOpponent/RemoveOpponent`（见 Basic/Perception）。
- `GetNumOpponents` / `GetNumOtherOpponents`：供围攻走位行为查询「围我的还有几个」。

### 目标有效性
- `TargetInvalidators` + `SetInvalidTarget/ClearInvalidTarget/IsValidTarget`：让目标临时「不可被选为目标」（如玩家进入安全区）。

## 二、成本令牌（Cost）细化

`GentlemanCostComponent.as`（**挂在 AI 上**）：用一种统一的「成本令牌」量化不同攻击的开销，向目标的绅士组件申领。
- `EGentlemanCost` 枚举把攻击分档（`XXXSmall=1 … Large=30`），`ClaimToken(Claimant, Cost)` 用 `int(Cost)` 作为申领的令牌数——**贵的攻击占更多令牌**，从而目标同时只能承受有限「总攻击成本」。
- 切目标时（`OnChangeTarget`）释放旧目标所有 Cost 令牌、切到新目标的绅士组件。
- 支持个人冷却（`PersonalCooldownTime`）、延迟释放（`PendingReleaseToken`，下一 Tick 才真正还，避免同帧抢还抖动）。

配套：`GentlemanCostSettings.as`（成本设置）、`GentlemanCostTeam.as`（成本队伍 `UGentlemanCostTeam`，AI `BeginPlay` 加入）、`GentlemanCostComponent` 加入 `n"GentlemanCostTeam"`。

## 三、令牌争夺与队列

- `GentlemanStealTokenComponent.as` + `GentlemanTokenHolderComponent.as`：**令牌抢夺**。持有者给每种令牌打分（`TokenScores`），`CanStealToken(name, StealScore)` 当挑战者分更高时可抢——让「更该攻击的 AI」夺取令牌。
- `Queue/` 子目录：**排队机制**，让等待的 AI 有序轮候而非乱抢。
  - `GentlemanQueueManagerComponent.as`：`UGentlemanQueue`（`JoinQueue/LeaveQueue/MoveQueue`）+ 按名管理多队列。
  - `GentlemanCostQueueComponent.as`：成本+队列结合。
  - `BasicGentlemanQueueSwitcherBehaviour.as` / `BasicGentlemanFitnessQueueSwitcherBehaviour.as`：根据排队位次（及 Fitness 打分）切换行为，队首的上前攻击、其余环绕。

## 四、行为侧接入

Basic 的 Gentleman 行为（见 `Gameplay/AI/Basic/Behaviour/Gentleman/`）：
- `BasicGentlemanWaitBehaviour`：未持令牌时原地等待。
- `BasicGentlemanCircleBehaviour`：未持令牌时环绕目标（`Requirements.AddBlock(Weapon)`——占着武器需求但不用，阻止低优先级攻击；靠近到 `GentlemanStepBackRange` 内会后撤，走位失败反向重试）。

攻击行为激活前会先查 `GentlemanCostComponent.ClaimToken`，抢到才真正进攻，否则退回环绕行为。

## 五、辅助

- `PlayerAISafeZone.as`：`APlayerAISafeZone` 触发盒——玩家进入时 `GentlemanComp.SetInvalidTarget`，使玩家临时不被任何 AI 选为目标。
- `GentlemanStatics.as`：便捷函数。
- `GentlemanLogCapability.as`：调试日志（temporal log）。

## 数据流

```
AI 选中目标(TargetingComponent) → 自动 AddOpponent 到目标的 GentlemanComponent
攻击行为想进攻:
  GentlemanCostComponent.ClaimToken(target 的 GentlemanComp, EGentlemanCost)
    ├─ 令牌够(未超上限) → 抢到 → 执行攻击 → 结束后 ReleaseToken(带冷却→冷却对象占位)
    └─ 令牌不够 → 退回 BasicGentlemanCircle/Wait 行为(环绕/排队)
Balancing(Support): 玩家血低 → 降低目标令牌上限 → 同时可攻击的 AI 变少(见 Support/)
玩家进安全区 → SetInvalidTarget → AI 放弃该目标
```

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
|---|---|---|---|
| 挂载 | 玩家/队伍(目标) | 组合 | `UGentlemanComponent` 挂在被攻击者身上，供众 AI 共享 |
| 桥接 | `UBasicAITargetingComponent` | 事件 | 切目标时 Add/RemoveOpponent、Claim/Release |
| 消费 | Gentleman 行为 | 行为 | 抢不到令牌 → 环绕/等待/排队 |
| 分档 | `EGentlemanCost` | 令牌数 | 贵攻击占更多令牌，限制总攻击成本 |
| 调节 | Balancing | 令牌上限 | 按玩家血量动态增减上限（Support/） |
| 打分 | Fitness | 组合 | Queue 切换行为结合站位打分（Fitness/） |
| 队伍 | `UGentlemanCostTeam` | 队伍 | AI 加入成本队伍便于群体管理 |
| 失效 | `APlayerAISafeZone` | Invalidator | 安全区内玩家不被选为目标 |

## 关键文件

- `GentlemanComponent.as`：`UGentlemanComponent` 令牌信号量 + 对手追踪 + 目标有效性 + 冷却对象池（核心）。
- `GentlemanActions.as`：令牌名（Cost/Ranged/Melee/Attack）与分值常量。
- `GentlemanCostComponent.as`：AI 侧成本令牌申领，`EGentlemanCost` 分档、延迟释放、个人冷却。
- `GentlemanStealTokenComponent.as` / `GentlemanTokenHolderComponent.as`：令牌抢夺（按分数）。
- `Queue/GentlemanQueueManagerComponent.as`：排队管理（多命名队列）。
- `Queue/BasicGentlemanQueueSwitcherBehaviour.as`：按队列位次切换攻击/环绕行为。
- `PlayerAISafeZone.as`：玩家安全区（令牌目标失效）。

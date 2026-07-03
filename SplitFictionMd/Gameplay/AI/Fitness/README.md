# Gameplay / AI / Fitness（站位打分）

> 职责：给 AI 的候选站位「打分」，衡量某个位置对当前目标玩家而言好不好——核心指标是「在玩家视野正前方且距离适中」。行为据此选择环绕方向、决定是否移动到更优位置。让敌人倾向于站在玩家看得见、打得着、又不太挤的地方，提升战斗可读性。共 5 文件。

## 一、核心：UFitnessUserComponent（打分器）

`Fitness/FitnessUserComponent.as`（挂在 AI 上）。关键函数 `GetFitnessScoreAtLocation(Player, Location)`：

```
Score = (10000 / max(Dist,1)) * max(ViewDir · ToAIDir, 0)^3
```

- **距离项**：越近分越高（反比）。
- **视野项**：位置越靠近玩家视线正前方分越高，且用 3 次方强烈偏向正前方（侧后方几乎 0 分）。

配套判定：`IsFitnessOptimalAtLocation`（是否达到最优阈值）、`IsFitnessBetterAtLocation`、`ShouldMoveToLocation`（比较当前 vs 目标位置的优劣，决定是否值得移动过去，避免抖动）。阈值来自 `FitnessSettings.as`（`OptimalThresholdMax` 等）。

## 二、站位方向优化

`FitnessStrafingComponent.as`（挂在 AI 上）：决定环绕时**往左还是往右走**。
- `OptimizeStrafeDirection`：在左右各偏移 100 单位处分别打分（`GetFitnessScoreAtLocation`），走向分更高的一侧；两侧都够好或相等则随机。
- `SetClosestToViewStrafeDirection`：按 AI 在玩家视野哪一侧决定方向。
- `RandomizeStrafeDirection` / `FlipStrafeDirection`：随机 / 翻转。

## 三、能力与队列排序

- `BasicOptimizeFitnessStrafingCapability.as`：周期性调用打分优化环绕方向的能力。
- `FitnessTargetCostQueueSortingCapability.as`：**把 Fitness 打分接入 Gentleman 排队**——按站位适合度对攻击队列排序，让站位更好的 AI 优先获得攻击令牌。

## 四、与行为/协调层接入

Basic 的 Fitness 走位行为（Basic/Behaviour/Maneuvering）：`BasicFitnessCircleStrafeBehaviour`、`BasicFlyingFitnessCircleStrafeBehaviour`、`BasicWallClimbingFitnessCircleStrafeBehaviour`——三形态各一，都用 Fitness 打分选环绕方向。Gentleman 的 `BasicGentlemanFitnessQueueSwitcherBehaviour` 结合打分与排队切换行为。

## 数据流

```
AI 走位行为激活:
  FitnessStrafingComponent.OptimizeStrafeDirection()
    → 左右候选位各 GetFitnessScoreAtLocation(玩家视野+距离评分)
    → 选高分侧 → DestinationComp.MoveTowards 环绕
Gentleman 排队:
  FitnessTargetCostQueueSortingCapability → 按 Fitness 分排序攻击队列 → 站位好的先拿令牌
```

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
|---|---|---|---|
| 消费 | Fitness 走位行为 | 行为 | `BasicFitnessCircleStrafeBehaviour` 三形态用打分选方向 |
| 接入 | Gentleman 队列 | 排序能力 | `FitnessTargetCostQueueSortingCapability` 按适合度排令牌队列 |
| 读取 | 目标玩家 | 直接调用 | 打分基于玩家 `ViewRotation` + 距离 |
| 设置 | `FitnessSettings` | 可组合设置 | 最优阈值等参数 |
| 执行 | `UBasicAIDestinationComponent` | 组合 | 选出的方向写入移动意图 |

## 关键文件

- `FitnessUserComponent.as`：`UFitnessUserComponent` 打分器（距离 × 视野正前方^3）。
- `FitnessStrafingComponent.as`：环绕方向优化（左右打分取优）。
- `BasicOptimizeFitnessStrafingCapability.as`：周期性优化环绕方向的能力。
- `FitnessTargetCostQueueSortingCapability.as`：按 Fitness 分排序 Gentleman 攻击队列。
- `FitnessSettings.as`：最优阈值等参数。

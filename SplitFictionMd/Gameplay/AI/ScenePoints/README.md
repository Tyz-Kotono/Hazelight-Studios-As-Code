# Gameplay / AI / ScenePoints（站位标记）

> 职责：关卡设计师在场景里手工摆放的「站位/入场/动画」标记点，AI 申领占用后前往站定或播放定点动画。是把「设计师意图」注入 AI 行为的锚点——比如「敌人从这几个门口登场」「弓箭手站在这些高台上」。共 10 文件。

## 一、核心：UScenepointComponent（申领式站位点）

`ScenePoints/ScenePointComponent.as`。一个带占用管理的场景点：
- 参数：`Radius`（到位判定半径）、`AlignAngleDegrees`（朝向对齐容差）、`CooldownDuration`（释放后冷却）、`MaxNumberOfUsers`（可同时占用人数，默认 1）。
- 申领：`CanUse` / `Use` / `Release`（释放触发冷却）—— **多个 AI 不会挤到同一个点**。
- 到位判定：`IsAt(Actor, PredictTime)`（支持用速度预测「即将经过」）、`IsAlignedWith`（朝向对齐）。

### FScenepointContainer（一组点的轮换分配）
把一批场景点当资源池分配，避免重复选同一个：
- `UseBestScenepoint` / `UseRandomInViewScenepoint` / `UseRandomScenepoint` / `UseNextScenepoint`。
- 内部维护 `UnusedScenepoints`，用尽后重填并排除上次用过的，实现「尽量不重复」。
- `ScenepointStatics::GetRandomInView`：优先选在任一玩家视野内的点（登场演出优先出现在镜头里）。

## 二、场景点演员

`ScenePointActor.as`：`AScenepointActorBase`（Abstract，`GetScenepoint()` 接口）+ `AScenepointActor`（内嵌 `UScenepointComponent`，编辑器带箭头/广告牌可视化）。
- `SplineScenepointActor.as`：沿样条的场景点（巡逻路径/入场路线）。
- `AnimationScenepointActor.as`：定点动画点。

## 三、专用组件

| 文件 | 职责 |
|---|---|
| `ScenepointUserComponent.as` | AI 侧：持有 `EntryScenepoint`（登场点引用） |
| `ScenepointTargetComponent.as` | 场景点的目标关联 |
| `ScenepointTeamComponent.as` | 场景点分队伍，供群体查询 |
| `ScenepointAnimationComponent.as` | 定点动画驱动 |
| `ScenepointVolume.as` | 体积内的场景点集合 |
| `ScenepointStatics.as` | 随机/视野内选点等便捷函数 |

## 四、与行为/生成的接入

- **入场**：生成模式 `HazeActorSpawnPatternEntryScenepoint`（Spawning/）指定敌人在哪个场景点登场；Basic 行为 `BasicScenepointEntranceBehaviour`（Basic/Behaviour/Entrance）驱动 AI 前往登场点并播入场动画。
- **站位**：远程/环绕行为可申领场景点作为射击/站位位置。
- **穿越点**：Traversal 的 `UTraversalScenepointComponent` 派生自本体系——穿越点本质也是场景点。

## 数据流

```
设计师: 关卡摆放 AScenepointActor / SplineScenepointActor（分组/分队伍）
AI 需要站位/登场:
  FScenepointContainer.UseBestScenepoint()（优先视野内、不重复）
    → Scenepoint.CanUse? → Use()（占用，计入 MaxNumberOfUsers）
    → 行为驱动 DestinationComp 前往 → IsAt() + IsAlignedWith() 判定到位对齐
    → 用完 Release()（触发 CooldownDuration）
```

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
|---|---|---|---|
| 消费 | Basic 入场/站位行为 | 行为 | `BasicScenepointEntranceBehaviour` 等前往并占用 |
| 指定 | Spawning | 生成模式 | `SpawnPatternEntryScenepoint` 指定登场点 |
| 派生 | Traversal | 继承 | 穿越点 `UTraversalScenepointComponent` 基于场景点 |
| 分组 | `ScenepointTeamComponent` | 队伍 | 场景点成组供群体查询 |
| 视野 | Core SceneView | 直接调用 | `GetRandomInView` 优先镜头内点 |
| 占用 | 申领语义 | Users 列表 | `MaxNumberOfUsers` + 冷却，避免拥挤 |

## 关键文件

- `ScenePointComponent.as`：`UScenepointComponent` 申领式站位点 + `FScenepointContainer` 轮换分配 + `ScenepointStatics`。
- `ScenePointActor.as`：`AScenepointActorBase` / `AScenepointActor` 场景点演员。
- `SplineScenepointActor.as` / `AnimationScenepointActor.as`：样条/动画场景点。
- `ScenepointUserComponent.as`：AI 侧登场点引用。
- `ScenepointTeamComponent.as` / `ScenepointVolume.as`：分组与体积集合。

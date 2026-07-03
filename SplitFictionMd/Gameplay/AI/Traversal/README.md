# Gameplay / AI / Traversal（AI 穿越）

> 职责：让 AI 在导航网格无法直连的断裂区域之间做「特殊移动」——弧线跳跃、抛物线飞跃、传送。核心是把关卡切成若干「穿越区域（TraversalArea）」，区域间用「穿越点（Scenepoint）+ 穿越方式（Method）」建立可跨越的连接，AI 找到所在区域后按方式跨越。共 20 文件。

## 一、内部架构

```
        UTraversalManager (UHazeTeam，全局单例队伍)
          │  成员 = 所有 ATraversalAreaActor
          │  缓存 Scenepoint↔Area 映射、分帧路径检测
          ▼
   ATraversalAreaActorBase (一块可停留/过渡区域)
          │  含若干 UTraversalScenepointComponent (穿越点)
          │  PointsByMethod: 每种 Method 一组点
          ▼
   UTraversalMethod (穿越方式：弧线/抛物线/传送)
          │  CanTraverse / IsInRange / AddTraversalPath
          ▼
   AI 侧: UTraversalComponentBase + BasicAIFindTraversalAreaCapability
          玩家侧: PlayerTraversalComponent + PlayerFindTraversalAreaCapability
```

| 层 | 文件 | 职责 |
|---|---|---|
| **管理器** | `TraversalManager.as`、`TraversalStatics.as` | 全局区域队伍、区域查找与路径缓存 |
| **区域** | `TraversalAreaActor.as` | `ATraversalAreaActorBase` 可停留/过渡区，持有分方式的穿越点 |
| **方式** | `TraversalMethods/TraversalMethod.as` + `TraversalComponentBase.as`、`TraversalScenepointComponent.as` | 穿越方式基类 + AI 穿越组件基类 + 穿越点 |
| **具体方式** | `ArcTraversal/`、`TrajectoryTraversal/`、`TeleportTraversal/` | 弧线、抛物线、传送三种实现 |
| **接入** | `BasicAIFindTraversalAreaCapability.as`、`BasicAITraversalSettings.as`、`PlayerFindTraversalAreaCapability.as`、`PlayerTraversalComponent.as` | AI/玩家 定位所在区域的能力 |

## 二、管理器（区域查找 + 分帧路径检测）

`TraversalManager.as` 的 `UTraversalManager : UHazeTeam`：所有穿越区域都是它的**队伍成员**。
- `FindTraversalArea(Scenepoint)`：为某个场景点找到它所属的穿越区域。**代价高的寻路检测被分帧摊平**——每次最多检查 100 个区域、每次只做一次 `Pathfinding::HasPath` 路径校验，用 `FindAreaCache` 记录进度；查全一轮后把 `Range` 翻倍再来。命中后写入 `ScenepointAreaCache` 缓存。
- `FindClosestTraversalScenepoint(Location)` / `...OnSameElevation`：从缓存里就近取点。
- `CanClaimTraversalCheck/ClaimTraversalCheck`：按帧号让同一帧只有一个 AI 做穿越检查，进一步限流。

## 三、区域与穿越方式

`ATraversalAreaActorBase`（`TraversalAreaActor.as`）：一块区域，`bTransitArea` 标记「只过路不停留」。核心是 `PointsByMethod`（`TMap<UClass, FTraversalScenepoints>`）——**按穿越方式分组**存穿越点，`TraversalBounds` 是球形包围。编辑器重建时刷新各区域点与包围。

`UTraversalMethod`（`TraversalMethod.as`，Abstract）：定义「怎么从一个点到另一个点」：
- `MinRange/MaxRange`：可穿越距离区间（区域可覆盖）。
- `CanTraverse / AddTraversalPath`：子类实现具体连接判定与建路。
- `IsInRange`：距离区间检测（支持区域级覆盖）。
- `IsTraversable(Edge)`：用射线判断导航边是「墙」还是「可跳下的落差」。
- `IsDestinationCandidate`：只接受本方式偏好类的场景点（"one must know one's place in society"）。

三种具体方式各含「组件 + 场景点 + 类型/几何」文件：

| 方式目录 | 文件 | 移动形态 |
|---|---|---|
| `ArcTraversal/` | `ArcTraversalComponent`、`ArcTraversalScenepoint`、`ArcTraversalTypes`、`TraversalArc` | 弧线跳跃 |
| `TrajectoryTraversal/` | `TrajectoryTraversalComponent`、`TrajectoryTraversalScenepoint`、`TrajectoryTraversalTypes`、`TraversalTrajectory` | 抛物线飞跃 |
| `TeleportTraversal/` | `TeleportTraversalComponent`、`TeleportTraversalScenepoint` | 传送 |

## 四、AI/玩家接入

`BasicAIFindTraversalAreaCapability.as`：AI 用来确定自己所在区域。`Setup` 取 `UTraversalComponentBase` + `Traversal::GetManager()`，绑重生 `OnReset`；`ShouldActivate` 当 `TraversalComp.CurrentArea == nullptr`（尚未定位）时激活，带随机初始冷却错峰；逐步扩大 `Range` 搜索。玩家侧有对称的 `PlayerFindTraversalAreaCapability` + `PlayerTraversalComponent`——**穿越系统玩家和 AI 共用**，同一套区域/方式既服务敌人也服务玩家。

## 数据流

```
关卡: 布置 ATraversalArea + 穿越点(按 Method 分组) → 全部入 TraversalManager 队伍
运行: AI/玩家 FindTraversalAreaCapability 激活(CurrentArea 空)
   → TraversalManager.FindTraversalArea(就近点，分帧 HasPath 校验) → 命中缓存
   → TraversalComponent.CurrentArea 设定
   → 需要跨区时: Method.CanTraverse/IsInRange 选点 → Arc/Trajectory/Teleport 组件执行移动
```

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
|---|---|---|---|
| 分组 | `UHazeTeam` | 队伍 | `UTraversalManager` 派生自队伍，成员为所有区域 |
| 寻路 | Core Navigation | 直接调用 | `Pathfinding::HasPath` 校验区域可达，分帧摊平 |
| 复用 | ScenePoints | 组合 | 穿越点 `UTraversalScenepointComponent` 派生自场景点体系 |
| 重生 | `UHazeActorRespawnableComponent` | 事件 | 重生时 `OnReset` 清空 CurrentArea 重新定位 |
| 共用 | Player | 对称能力 | 玩家用同一套区域/方式做特殊移动 |
| 限流 | 帧号仲裁 | Instigator/帧 | 同帧只允许一个查询做昂贵的区域检测 |

## 关键文件

- `TraversalManager.as`：`UTraversalManager`（区域队伍 + 分帧路径检测 + 场景点↔区域缓存）。
- `TraversalAreaActor.as`：`ATraversalAreaActorBase` 穿越区域（按方式分组的穿越点、过渡区标记）。
- `TraversalMethods/TraversalMethod.as`：穿越方式基类（距离区间、可穿越判定、目标点候选）。
- `TraversalMethods/TraversalComponentBase.as`：AI 穿越组件基类（持 CurrentArea）。
- `BasicAIFindTraversalAreaCapability.as`：AI 定位所在区域的能力。
- `ArcTraversal/` `TrajectoryTraversal/` `TeleportTraversal/`：三种具体穿越方式实现。
- `PlayerTraversalComponent.as` / `PlayerFindTraversalAreaCapability.as`：玩家侧对称接入。

# Gameplay / AI / Spawning（生成系统）

> 职责：AI 敌人的生命周期工厂——用「生成池」复用演员避免频繁 new/destroy，用「生成器 + 生成模式（Pattern）」声明式地控制何时、生成多少、如何生成，并配合重生组件把「死亡」变成「归还池中待复用」。共 37 文件，分 3 层。

## 一、内部架构（三层）

```
        AHazeActorSpawnerBase (关卡里放置的生成器演员)
              │  挂载
        UHazeActorSpawnerComponent  ← 激活开关(bIsActive, Instigator)
              │  + 若干 UHazeActorSpawnPattern (声明式模式，可叠加)
              ▼
        UHazeActorSpawnerCapability (控制端每帧驱动，AfterGameplay)
              │  从池取演员
              ▼
        SpawnPool (本地/联网) ──复用──▶ AHazeActor(带 RespawnableComponent)
```

| 层 | 文件 | 职责 |
|---|---|---|
| **生成器** | `HazeActorSpawnerBase.as`、`HazeActorSpawnerComponent.as`、`HazeActorSpawnerCapability.as` | 放置演员 + 激活开关组件 + 驱动能力 |
| **生成模式** | `SpawnPatterns/` 24 文件 | 声明「生成什么/多少/条件」，可叠加组合 |
| **生成池** | `HazeActorLocalSpawnPoolComponent.as`、`HazeActorNetworkedSpawnPoolComponent.as`、`HazeActorLocalSpawnPoolEntryComponent.as`、`HazeActorSpawnPoolReserve.as` | 复用演员实例 |
| **重生** | `HazeActorRespawnableComponent.as` | 演员侧「可回收」标记 + 重置广播 |

## 二、生成池（复用核心）

`HazeActorLocalSpawnPoolComponent.as`：按 `SpawnClass` 建一个池，挂在关卡脚本演员上（持久层/动态生成层有特殊归属规则，见 `HazeActorLocalSpawnPoolStatics::GetOrCreateSpawnPool`）。
- `Spawn(Params)`：若 `Respawnables` 有空闲实例则 `TeleportActor` 复位复用；否则 `SpawnActor` 真正实例化。
- `UnSpawn(Respawnable)`：把演员放回 `Respawnables` 待复用（不销毁）。
- `HazeActorNetworkedSpawnPoolComponent` 是其联网版；生成器能力用的正是联网池。

**这是「死亡即回收」的落点**：AI 死亡 → `UDisableComponent` 禁用 → `BasicBehaviourComponent.OnActorDisabled` 判死 → `OnUnspawn` → `RespawnableComponent` → 池 `UnSpawn`。

## 三、重生组件

`HazeActorRespawnableComponent.as`（演员侧，AI 基类自带）：
- `OnUnspawn`（可回收了）、`OnRespawn`（复用前重置）、`OnPostRespawn`。
- `OnSpawned(Spawner, Params)`：记录生成器/时间，广播 `OnRespawn`，并触发网络位置同步 `TransitionSync`。
- 由 `BasicAIUpdateCapability` 把 `OnRespawn` 绑定到 Health/Behaviour/Destination/Anim 的 `Reset`——复用时把敌人状态清干净。

## 四、生成器与驱动能力

`HazeActorSpawnerComponent.as`：激活开关用 `TInstigated<bool> bIsActive`（`ActivateSpawner/DeactivateSpawner`，Instigator + 优先级）。用 `TeamName`（= 生成器路径名）给生成物建队伍，从而提供群体操作：`KillSpawnedAI`、`MakeSpawnedAIsFlee`、`DisableSpawnedActors`、`ResetSpawnPatterns`、按 Tag 激活/停用模式。

`HazeActorSpawnerCapability.as`（`AfterGameplay`，仅控制端）：`Setup` 收集并**按 `UpdateOrder` 升序排序**所有模式、为每个 SpawnClass 建联网池、订阅 `OnSpawnedBySpawner`。`ShouldActivate` 要求有控制端 + 生成器激活 + 有模式 + `bNeedsUpdate`。逐模式（低序在前）累积一个 `FHazeActorSpawnBatch`，高序模式可覆盖低序的生成参数（如改设置/加入队伍）。

## 五、生成模式（Pattern，24 个）

`SpawnPatterns/HazeActorSpawnPattern.as` 定义基类 `UHazeActorSpawnPattern : USceneComponent`：`bStartActive`、`UpdateOrder`（First→Last，决定累积/覆盖次序）、`bCanEverSpawn`、激活用 `TInstigated<bool>`。多个模式**叠加**在一个生成器上组合出复杂行为。分三类：

| 类别 | 代表模式 | 作用 |
|---|---|---|
| **生成量** | `Single`、`Interval`、`Wave` | 单个 / 定时 / 波次 |
| **激活条件** | `ActivateOnDeaths`、`ActivateOnDelay`、`ActivateOnRandomDelay`、`ActivateOnDoneSpawning`、`ActivateOtherSpawner`、`ActivateOwnPatterns`、`OnDeaths`、`OtherActivateOn*` | 按死亡数/延迟/其他生成器完成等触发链式生成 |
| **生成物配置** | `ApplySettings`、`ApplyWorldUp`、`AggroPlayer`、`BlockCapabilityTag`、`EntryAnimation`、`EntryScenepoint`、`EntrySpline`、`FleeSpline`、`JoinTeam`、`SpawnAtScenepoint` | 生成后立即配置：套设置、指定登场动画/入场点/样条、加入队伍、锁定玩家 |

波次示例 `HazeActorSpawnPatternWave.as`：`WaveSize`（每波数量，超量分批 `MaxPerBatch=5`/`BatchInterval=0.5s` 避免尖峰）、`RespawnDuration`（一波全灭后隔多久起新波）、`bInfiniteSpawn` / `MaxTotalWaves`。波次靠追踪本波生成物 Unspawn 计数归零来判定「全灭」。

`Spawners/` 是封装好的现成生成器演员：`HazeActorSingleSpawner`、`HazeActorIntervalSpawner`、`HazeActorWaveSpawner`（内嵌 `SpawnPatternWave` + `EntryScenepoint`）、`HazeActorSpawner`。

## 六、数据流

```
关卡放置 Spawner → ActivateSpawner(Instigator)
   → SpawnerCapability(控制端) 逐 Pattern(按 UpdateOrder) 累积 Batch
   → 从联网 SpawnPool 取/建演员 → RespawnableComponent.OnSpawned → OnRespawn 重置状态
   → 高序 Pattern 配置(设置/入场/队伍) → 敌人加入生成器队伍参战
敌人死亡 → Disable → OnUnspawn → SpawnPool.UnSpawn(归还) → 供下一波复用
一波全灭计数归零 → 等待 RespawnDuration → 起新波（未达 MaxTotalWaves）
```

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
|---|---|---|---|
| 复用 | `SpawnPool` | 组合 | 演员实例复用，避免频繁创建销毁 |
| 回收触发 | `UDisableComponent` / `BasicBehaviourComponent` | 事件 | 死后禁用 → `Unspawn` → 归还池 |
| 重置 | `BasicAIUpdateCapability` | 事件绑定 | `OnRespawn` → Health/Behaviour/Destination/Anim `Reset` |
| 分组 | `UHazeTeam` | 队伍 | 生成物入生成器队伍，支持群体 Kill/Flee/Disable |
| 激活 | `TInstigated<bool>` | Instigator | 生成器/模式激活开关 |
| 入场 | ScenePoints / Spline | 组合 | `EntryScenepoint`/`EntrySpline` 模式指定登场位置 |
| 联网 | Crumb / 位置同步 | 网络 | 控制端生成，`TransitionSync` 同步位置 |

## 关键文件

- `HazeActorLocalSpawnPoolComponent.as` / `HazeActorNetworkedSpawnPoolComponent.as`：生成池（本地/联网），复用核心。
- `HazeActorRespawnableComponent.as`：演员侧可回收标记 + 重置广播。
- `HazeActorSpawnerComponent.as`：生成器激活开关 + 队伍级群体操作。
- `HazeActorSpawnerCapability.as`：控制端驱动，按 UpdateOrder 累积生成批次。
- `SpawnPatterns/HazeActorSpawnPattern.as`：模式基类（叠加/次序）。
- `SpawnPatterns/HazeActorSpawnPatternWave.as`：波次模式（分批、全灭判定、最大波数）。
- `Spawners/HazeActorWaveSpawner.as` 等：封装好的现成生成器演员。

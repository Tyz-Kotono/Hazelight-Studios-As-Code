# Core / Flow / Triggers（触发体）

> 职责：提供一组联机安全的体积触发器，监测玩家或演员进入/离开并广播事件，是关卡逻辑与外部系统（能力、设置、教学）的通用接入点。共 7 个文件。

## 内部架构

### 基类与派生体系
- **`APlayerTrigger`（AVolume）**：模块核心基类。监测各玩家是否在体积内，提供 `OnPlayerEnter` / `OnPlayerLeave` 事件。多数其他触发体继承自它。
  - 每玩家状态 `TPerPlayer<FPlayerTriggerPerPlayerData>`（`bIsPlayerInside` + `DisableInstigators`）。
  - `bTriggerForMio` / `bTriggerForZoe` 门控；`bStartDisabled` + `StartDisabledInstigator` 支持入场即禁用。
  - 进入/离开经 `CrumbPlayerEnter` / `CrumbPlayerLeave` 联机同步；`bTriggerLocally` 可改为纯本地。
  - `protected TriggerOnPlayerEnter/Leave()` 为虚函数，子类覆盖以扩展行为（不破坏事件广播）。
- **`ABothPlayerTrigger`（AVolume）**：仅当“两名玩家同时在内”时广播 `OnBothPlayersInside`，离开其一即 `OnStopBothPlayersInside`。状态用 `TPerPlayer<bool> PlayersInsideTrigger` + `bBothPlayersInside`，仅控制端跟踪。
- **`AActorTrigger`（AVolume）**：按 `ActorClasses`（类）或 `SpecificActors`（实例）筛选 `AHazeActor`，进入/离开广播 `OnActorEnter` / `OnActorLeave`；以 `ActorsInsideTrigger` 数组去重。
- **`AConditionalPlayerTrigger`（AVolume, Abstract）**：在 `APlayerTrigger` 语义上增加持续条件 `IsPlayerConditionMet(Player)`，重叠期间每帧 Tick 重算，条件满足才视为“在内”。子类需覆写条件。

### 行为型派生（均继承 `APlayerTrigger`）
- **`ACapabilityBlockVolume`**：进入时对玩家 `BlockCapabilities(Tag, this)`，离开时 `UnblockCapabilities`，按 `BlockTags` 屏蔽能力。
- **`ACapabilitySheetVolume`**：内含 `UHazeRequestCapabilityOnPlayerComponent`，进入时启动 `PlayerSheets`/`MioSheets`/`ZoeSheets` 指定的能力表，离开时停止。
- **`AApplySettingsTrigger`**：进入时按 `SettingsToApply`（资产 + 优先级 + 适用玩家）调用 `ApplySettings`，离开/EndPlay 时按 instigator 清除。

### 数据流
引擎重叠事件 → 控制端校验（`HasControl` + `IsEnabledForPlayer`）→ Crumb 同步 → `ConditionalOnPlayerEnter/Leave` 去重 → `TriggerOnPlayerEnter/Leave` 虚函数 → 事件广播 / 子类扩展。启停状态变化时由 `UpdateAlreadyInsidePlayers()` 经 `TraceOverlappingComponent` 重检补漏。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 被继承 | `ATutorialVolume` / `AForceRespawnOtherPlayerTrigger` / `ARespawnOnSplineNearOtherPlayerVolume` | 继承 | 这些类 `class X : APlayerTrigger`，复用进入/离开 + 启停语义 |
| 调用 | 能力系统 | 标签阻塞 | `ACapabilityBlockVolume` 调 `Player.BlockCapabilities(Tag, this)` / `UnblockCapabilities` |
| 持有 | `UHazeRequestCapabilityOnPlayerComponent` | 直接调用 | `ACapabilitySheetVolume` 调 `RequestComp.StartInitialSheetsAndCapabilities(Player, this)` |
| 调用 | 设置系统 | ApplySettings | `AApplySettingsTrigger` 调 `Actor.ApplySettings(Asset, this, Priority)` 与 `ClearSettingsByInstigator(this)` |
| 广播 | 关卡蓝图/其他系统 | 委托 | `OnPlayerEnter` / `OnBothPlayersInside` / `OnActorEnter` 供外部绑定 |

## 关键文件

- `PlayerTrigger.as`：基类 `APlayerTrigger`，每玩家进入/离开监测与启停，全模块触发模式样板。
- `BothPlayerTrigger.as`：`ABothPlayerTrigger`，仅两名玩家同时在内时触发。
- `ActorTrigger.as`：`AActorTrigger`，按类或实例筛选演员的进入/离开触发。
- `ConditionalPlayerTrigger.as`：抽象基类 `AConditionalPlayerTrigger`，带每帧持续条件判定的玩家触发。
- `CapabilityBlockVolume.as`：`ACapabilityBlockVolume`，进入时按标签屏蔽玩家能力。
- `CapabilitySheetVolume.as`：`ACapabilitySheetVolume`，进入时为玩家请求能力表。
- `ApplySettingsTrigger.as`：`AApplySettingsTrigger`，进入/离开时应用与清除可组合设置。

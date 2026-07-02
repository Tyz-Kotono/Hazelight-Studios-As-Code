# Core / Flow / RespawnPoint（重生点）

> 职责：定义世界中的重生点及其启停体积，玩家死亡时由 PlayerHealth 重生流程按优先级与距离选点；并提供“靠近队友重生”“强制队友重生”等特殊重生覆盖。共 5 个文件。

## 内部架构

### 重生点本体
- **`ARespawnPoint`（AHazeActor）**：单个重生点。
  - 优先级 `ERespawnPointPriority`（NoRespawn → Lowest/Low/Normal/High/Highest）；`bCanMioUse` / `bCanZoeUse` 门控。
  - 每玩家启用状态 `TPerPlayer<FRespawnPointPlayerState>`（`EnableInstigators` 引用计数），经 `EnableForPlayer` / `DisableForPlayer` 启停，`IsEnabledForPlayer` 判定。
  - 支持双人分别落点（`SecondPosition` / `FinalSpawnPositions`）、贴地（`bSnapToGround` 胶囊射线）、贴样条（`bSnapToSpline`）、关卡出生点（`bIsLevelSpawnPoint` 创建 `UHazePlayerSpawnPointComponent`）、重生相机旋转。
  - 委托：`OnRespawnAtRespawnPoint`、`OnPlayerTeleportToRespawnPoint`、`OnRespawnPointEnabled/Disabled`；虚函数 `IsValidToRespawn` / `ShouldRecalculateOnRespawnTriggered` 供派生扩展。
  - 用 `UHazeListedActorComponent` 登记，便于重生流程经 `TListedActors<ARespawnPoint>` 全局遍历。

### 启停体积
- **`ARespawnPointVolume`（AVolume）**：玩家进入时启用 `EnabledRespawnPoints` 中的重生点。
  - `bSticky`（粘性：进入后持续生效直到进入另一粘性体积）、`bSharedByBothPlayers`（任一玩家进入即为双方启用）、`bOnlyTriggerOnce`、`bOnlyTriggerWhenPlayerGrounded`、Mio/Zoe 门控、`bStartDisabled`。
  - `DisableBacktrackingToVolumes`：首次进入即永久禁用回溯体积，避免折返后回退重生点。
  - 每玩家 `FRespawnPointVolumePerPlayerData`；用 `TraceOverlappingComponent` 重检；落地条件未满足时开 Tick 等待。

### 特殊重生覆盖
- **`ARespawnNearOtherPlayerVolume`（AVolume）**：进入时给玩家绑定 `FOnRespawnOverride` 委托，死亡时在 `UsableRespawnPoints` 中选离“队友”最近且最高优先级者重生。
- **`ARespawnOnSplineNearOtherPlayerVolume`（APlayerTrigger）**：本地触发；死亡时在样条上选离队友最近点重生，可匹配队友速度（`bMatchOtherPlayerSpeedOnRespawn`）。
- **`AForceRespawnOtherPlayerTrigger`（APlayerTrigger）**：任一玩家首次进入时，若队友已死则用 `UPlayerRespawnComponent.ApplyRespawnOverrideLocation` 强制其在指定重生点复活。

### 数据流
体积进入 → 启用重生点（`EnableForPlayer(this)`）→ 玩家死亡 → `PlayerRespawnComponent` 遍历已启用重生点按优先级+距离选点（或被 override 委托接管）→ 取 `GetPositionForPlayer` 落位 → `OnRespawnTriggered` 广播。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 被消费 | PlayerHealth/`UPlayerRespawnComponent` | 直接调用 + 列表遍历 | `PlayerRespawnComponent.as` 遍历 `TListedActors<ARespawnPoint>`，比 `RespawnPriority` 与距离，调 `IsEnabledForPlayer` / `IsValidToRespawn` / `GetPositionForPlayer` |
| 驱动 | `ARespawnPoint` | 直接调用 | `ARespawnPointVolume.EnableRespawnPoints` 调 `RespawnPoint.EnableForPlayer(Player, this)` |
| 粘性回调 | PlayerHealth mixin | 直接调用 | 体积调 `ApplyEnterStickyRespawnPointVolume` / `ApplyRemoveRespawnPointVolumeSticky`（定义于 `PlayerRespawnComponent.as`） |
| 覆盖 | `UPlayerRespawnComponent` | 委托 | Near/Spline 体积调 `Player.ApplyRespawnPointOverrideDelegate(this, ...)` / `ClearRespawnPointOverride`；Force 调 `ApplyRespawnOverrideLocation` |
| 继承 | Triggers/`APlayerTrigger` | 继承 | Spline 与 Force 体积继承自 `APlayerTrigger` |

## 关键文件

- `RespawnPoint.as`：`ARespawnPoint`，重生点本体（优先级/双人落点/贴地贴样条/启停）。
- `RespawnPointVolume.as`：`ARespawnPointVolume`，进入即启用其重生点，支持粘性/共享/单次/回溯禁用。
- `RespawnNearOtherPlayerVolume.as`：`ARespawnNearOtherPlayerVolume`，死亡时选离队友最近的重生点。
- `RespawnOnSplineNearOtherPlayerVolume.as`：在样条上选离队友最近点重生，可匹配队友速度。
- `ForceRespawnOtherPlayerTrigger.as`：`AForceRespawnOtherPlayerTrigger`，首次进入强制已死队友重生。

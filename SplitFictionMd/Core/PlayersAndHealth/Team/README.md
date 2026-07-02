# Core / PlayersAndHealth / Team

> 职责：提供通用的「按名称分组」队伍系统，让任意 `AHazeActor` 加入/离开以 `FName` 标识的队伍并共享成员列表与动作时间戳，共 3 文件。与玩家本体松耦合，主要服务于 AI 协作。

## 内部架构

- **`UHazeTeam`（`HazeTeam.as`）**：单支队伍对象（`UObject`）。
  - 数据：`TArray<AHazeActor> Members`、`TMap<FName, float> LastReportedActionTimes`（按动作标签记录最近一次时间）、`Originator`（首个成员）。
  - 接口：`AddMember`/`RemoveMember`/`IsMember`/`GetMembers`、`ReportAction`/`GetLastActionTime`、事件 `OnJoined`/`OnLeft`（`FHazeTeamMemberEventSignature`），以及可重写的 `OnMemberJoined`/`OnMemberLeft` BlueprintEvent。
- **`UHazeTeamManager`（`HazeTeamManager.as`）**：全局单例（`Game::GetSingleton`），持有 `TMap<FName, UHazeTeam> Teams`。
  - `JoinTeam(Member, TeamName, TeamClass?)`：队伍不存在则按指定类新建（默认 `UHazeTeam`），存在则校验类匹配后加入。
  - `GetJoinedTeam`/`GetTeam`/`LeaveTeam`：查询与离队；队伍空则自动移除。
- **`HazeTeamStatics.as`**：mixin/命名空间门面，`JoinTeam`/`GetJoinedTeam`/`LeaveTeam`（mixin on `AHazeActor`）与 `HazeTeam::GetTeam`，内部均通过 `Game::GetSingleton(UHazeTeamManager)` 转发。

> 注：这是 `AHazeActor` 级别的通用分组机制，并非玩家专属。实测主要消费者集中在 `Gameplay/AI`（生成、感知、行为、Gentleman 成本队伍）与各 `LevelSpecific` 关卡 AI/机关。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|------|------|------|------|
| 单例 | `UHazeTeamManager` | `Game::GetSingleton` | `HazeTeamStatics.as` 所有 mixin 经此取管理器 |
| 调用← | mixin `JoinTeam`/`LeaveTeam`/`GetJoinedTeam` | mixin 直接调用 | `Gameplay/AI`（`HazeActorSpawner*`、`BasicBehaviourComponent`、`GentlemanCostTeam`）、多关卡 AI（Island/Coast/Dentist 等）40+ 文件消费 |
| 委托→ | `OnJoined`/`OnLeft` | event/Broadcast | 成员变动通知订阅者 |
| 扩展 | `OnMemberJoined`/`OnMemberLeft` | BlueprintEvent 重写 | 派生队伍类自定义入队/离队逻辑 |
| 计时 | `ReportAction`/`GetLastActionTime` | 共享时间戳 | 队伍内协调动作节奏（如 AI 轮流攻击） |

## 关键文件

- `HazeTeam.as`：单支队伍对象，成员列表、动作时间戳与入/离队事件。
- `HazeTeamManager.as`：全局单例，按 `FName` 创建/查询/销毁队伍并校验队伍类。
- `HazeTeamStatics.as`：`JoinTeam`/`LeaveTeam`/`GetJoinedTeam`/`GetTeam` 门面 mixin。

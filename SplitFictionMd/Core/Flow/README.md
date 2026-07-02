# Core / Flow（流程控制）

> 职责：以触发体、组件与单例驱动关卡推进、教学、重生、检查点、按需禁用、淡入淡出与显隐，是连接玩家行为与关卡事件的中枢。共 7 子模块、33 个源文件。

## 子模块总览

| 子模块 | 文件数 | 核心职责 | 入口类型 |
|---|---|---|---|
| [Triggers](Triggers/README.md) | 7 | 体积触发器：玩家/双人/演员进入离开、能力屏蔽与能力表、可组合设置应用 | `APlayerTrigger` 及其派生 |
| [Tutorial](Tutorial/README.md) | 8 | 教学提示系统：屏幕/世界空间提示、提示链、取消提示、教学体积与能力 | `UTutorialComponent` |
| [RespawnPoint](RespawnPoint/README.md) | 5 | 重生点与重生体积：按优先级/距离选点、靠近队友重生、强制队友重生 | `ARespawnPoint` |
| [Progress](Progress/README.md) | 1 | 检查点触发体：首次进入激活进度点（默认禁用、Crumb 单次） | `AProgressPointTrigger` |
| [Disable](Disable/README.md) | 4 | 按需禁用：双玩家远离时自动禁用演员，单例分帧轮转，分组禁用 | `UDisableComponent` |
| [Fades](Fades/README.md) | 3 | 屏幕淡化与加载屏淡化：每玩家淡入淡出、全屏淡化、加载屏过渡 | `UFadeManagerComponent` |
| [Visibility](Visibility/README.md) | 3 | 每玩家显隐：体积控制目标演员/关卡对 Mio 或 Zoe 单独可见性 | `AVisibilityVolume` |

## 共有触发体模式

Flow 中大量体积类（`APlayerTrigger`、`ABothPlayerTrigger`、`AActorTrigger`、`ARespawnPointVolume`、`AProgressPointTrigger` 等）共享一套联机安全的触发约定，理解一处即可读懂全部：

1. **每玩家 `FInstigator` 启停**
   触发体不维护单一 bool 开关，而是用 `TArray<FInstigator> DisableInstigators`（或 `EnableInstigators`）记录“谁让我禁用/启用”。只有当某玩家的 instigator 集合为空时该玩家才算启用。这允许多个系统独立地启停同一触发体而互不覆盖（引用计数语义）。常见 API：`EnablePlayerTrigger(Instigator)` / `DisablePlayerTrigger(Instigator)` / `EnableForPlayer` / `DisableForPlayer`。

2. **Mio / Zoe 门控**
   普遍存在 `bTriggerForMio` / `bTriggerForZoe`，在 `IsEnabledForPlayer()` 中先判定玩家身份（`Player.IsMio()`）再决定是否响应，使同一触发体可对两名玩家差异化生效。

3. **`HasControl()` + `CrumbFunction` 联网**
   重叠事件只在控制端（`Player.HasControl()`）处理，随后通过 `UFUNCTION(CrumbFunction)` 的 Crumb（如 `CrumbPlayerEnter`/`CrumbPlayerLeave`）把进入/离开广播到远端，保证两端状态一致。`bTriggerLocally = true` 可绕过联网直接本地广播（用于纯表现或确定性本地逻辑）。

4. **`TraceOverlappingComponent` 重检**
   由于禁用、流式加载或触发体移动可能漏掉引擎的 `ActorBeginOverlap`/`ActorEndOverlap`，每个触发体提供 `UpdateAlreadyInsidePlayers()`（或 `UpdateContainment()`），在启停状态变化时用 `Player.CapsuleComponent.TraceOverlappingComponent(BrushComponent)` 主动重检谁在体积内，补发遗漏的进入/离开。`EndPlay` 时立即强制离开，不等 Crumb。

## 协作 5 机制（跨模块通用）

Flow 模块之间、以及与外部系统（PlayerHealth、Progress、能力系统、设置系统）协作的标准方式：

| # | 机制 | 说明 | 典型实例 |
|---|---|---|---|
| 1 | **直接调用 `Type::Get`** | 通过 `UXxx::Get(Owner/Player)` 取得目标组件后直接调用 | TutorialStatics 用 `UTutorialComponent::Get(Player)`；FadeStatics 用 `UFadeManagerComponent::GetOrCreate(Player)` |
| 2 | **Crumb（联机广播）** | `CrumbFunction` 把控制端决策同步到远端 | 所有触发体的 `CrumbPlayerEnter`；`CrumbUpdateAutoDisableState` |
| 3 | **标签阻塞（Tag Block）** | `BlockCapabilities(Tag, Instigator)` 按能力标签屏蔽 | `ACapabilityBlockVolume`；可视化遮挡用 `CapabilityTags::Visibility` |
| 4 | **委托（Delegate / Event）** | 广播事件或绑定重生覆盖回调 | `OnPlayerEnter`/`OnRespawnAtRespawnPoint`；`ApplyRespawnPointOverrideDelegate` |
| 5 | **可组合设置 ApplySettings** | `ApplySettings(Asset, Instigator, Priority)` 分优先级叠加设置，离开时按 instigator 清除 | `AApplySettingsTrigger` |

## 模块协同关系（概览）

| 方向 | 关系 | 机制 |
|---|---|---|
| RespawnPoint → PlayerHealth | `PlayerRespawnComponent` 遍历 `TListedActors<ARespawnPoint>`，按 `RespawnPriority` + 到玩家距离选点并消费 `GetPositionForPlayer` | 直接调用 + 委托 |
| Triggers → Tutorial | `ATutorialVolume` 继承 `APlayerTrigger`，进入时调用教学静态函数 | 继承 + 直接调用 |
| Triggers → 能力系统 | `ACapabilityBlockVolume` / `ACapabilitySheetVolume` 在进入离开时屏蔽能力或请求能力表 | 标签阻塞 / 请求组件 |
| Triggers → 设置系统 | `AApplySettingsTrigger` 在进入离开时应用/清除可组合设置 | ApplySettings |
| Progress → 进度系统 | `AProgressPointTrigger` 首次进入激活进度点 | Crumb + 直接调用 |
| Disable → 全局单例 | `UDisableComponent` 注册到 `UDisableComponentSingleton`，由单例分帧轮转更新远近禁用 | 单例注册 |
| Visibility → 渲染 | `AVisibilityVolume` 对目标组件设置 `EHazeVisibilityBit::HideFromMio/Zoe` | 直接调用渲染位 |

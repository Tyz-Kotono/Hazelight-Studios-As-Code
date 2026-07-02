# Core / InteractionAndAiming / Interaction

> 职责：在 Targetable 地基上构建“近身可交互物”玩法——把交互物注册成 `n"Interaction"` 分类的目标，玩家按键后经 move-to → 网络锁 → 能力 sheet 的流程进入/退出交互。共 8 个文件。

## 内部架构

### 关键类与基类
- **`UInteractionComponent : UTargetableComponent`**（InteractionComponent.as）：交互物组件，**子类化 Targetable**。
  - `default TargetableCategory = n"Interaction"`，重写 `CheckTargetable`：先判 Focus/Action 区域命中、互斥(`MutuallyExclusiveInteractions`)、`FInteractionCondition` 条件，再调 `Targetable::ScoreCameraTargetingInteraction` 评分。
  - 区域：`FocusArea`（远，参与考虑/网络 hint）/`ActionArea`（近，可触发）；由 `FHazeShapeSettings` 自动生成 trigger shape，overlap 回调维护 `EnteredFocus/ActionAreas`。
  - 交互体：`InteractionCapabilityClass`/`InteractionSheet`/`MovementSettings(FMoveToParams)`/`bIsImmediateTrigger`。
  - 状态：`StartInteracting/StopInteracting`（控制端）广播 `OnInteractionStarted/Stopped`；交互期间 `Disable(n"InteractionActive")` 防重入。
  - 网络：`NetworkLock : UNetworkLockComponent`，按距离/区域计算 owner hint（`UpdateNetworkHints`）；跨 actor 互斥用全局锁 `n"GlobalInteractionLock"`。
- **`UPlayerInteractionsComponent : UActorComponent`**（PlayerInteractionsComponent.as）：玩家侧轻状态——`ActiveInteraction`、`bIsNearInteraction`、默认 sheet（`DefaultInteractionSheet`/`DefaultValidationSheet`）、`InteractionWidgetClass`。

### 能力链（均 `NetworkMode = Crumb`）
- **`UInteractionEnterCapability`**：PreTick 渲染所有可交互浮窗；`ShouldActivate` 按下 `n"Interaction"` 动作 + 取 `GetPrimaryTarget(UInteractionComponent)` + 抢 `NetworkLock`；`OnActivated` 执行 move-to（瞬移或异步）→ 拿锁后 validation → `StartInteraction`。
- **`UInteractionExitCapability`**：`ActiveInteraction` 存在即启动其 sheet/能力，退出时 `StopInteracting`；TickActive 维护 Cancel 提示。
- **`UInteractionCancelCapability`**：按 `ActionNames::Cancel` 且 `CanPlayerCancel` → 停止交互。
- **`UInteractionCapability`**（基类）：交互期间在玩家上运行的具体玩法能力基类，`SupportsInteraction` 由子类决定响应哪些交互。

### 数据流
注册（Focus/Action overlap）→ EnterCapability PreTick 展示浮窗 → 按键取 primary → 抢锁/move-to → `StartInteracting` 置 `ActiveInteraction` → ExitCapability 起 sheet → Cancel/条件失效/`StopInteracting` 退出释放锁。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 继承自 | `UTargetableComponent` | 子类化 | `UInteractionComponent : UTargetableComponent`，`default TargetableCategory = n"Interaction"`（InteractionComponent.as:17、23） |
| 调用入 | `UPlayerTargetablesComponent` | 直接调用 | EnterCapability `GetPrimaryTarget(UInteractionComponent)` / `ShowWidgetsForTargetables` / `GetMostVisibleTargetAndResult`（InteractionEnterCapability.as:54-95） |
| 调用出 | `UNetworkLockComponent` | 网络锁 | `NetworkLock.Acquire/Release/IsAcquired`、`ApplyOwnerHint`；跨 actor 用 `GetOrCreate(Game::Mio, n"GlobalInteractionLock")`（InteractionComponent.as:197；EnterCapability.as:104） |
| 调用出 | `UMoveToComponent` / `MoveTo::` | 直接调用 + 委托 | `MovePlayerTo(..., FOnMoveToEnded(this, n"OnMoveComplete"))`、`MoveTo::ApplyMoveToInstantly`（EnterCapability.as:135、157） |
| 调用出 | 玩家能力系统 | 直接调用 | `Player.StartCapabilitySheet(...)`、`StartSingleRequestedCapability`（ExitCapability.as:67-71） |
| 网络同步 | 控制/远端 | Crumb | Enter/Exit/Cancel 能力 `NetworkMode = EHazeCapabilityNetworkMode::Crumb` |
| 事件 | 业务监听者 | 委托 Broadcast | `OnInteractionStarted/Stopped.Broadcast`（`FOnInteractionEvent`）；`FInteractionCondition` delegate（InteractionComponent.as:667、680） |
| 标签 | move-to / cutscene | 标签阻塞 | `BlockExclusionTags n"UsableDuringMoveTo"`、`CapabilityTags n"BlockedByCutscene"` |
| 消费方 | Audio 域 | 直接调用 | `PlayerEffortRecoveryAudioCapability` 引用交互组件（grep 命中） |
| 双人 | Mio / Zoe | 可组合设置 | `TPerPlayer<FInteractionPerPlayerData>`；网络锁保证两人不同时占用同一交互 |

## 关键文件

| 文件 | 说明 |
|---|---|
| InteractionComponent.as | 交互物组件 `UInteractionComponent`（继承 Targetable）+ 区域/互斥/条件/网络锁 |
| PlayerInteractionsComponent.as | 玩家侧状态：`ActiveInteraction`、默认 sheet、widget 类 |
| InteractionEnterCapability.as | 进入流程：取 primary → 抢锁 → move-to → validation → 开始交互 |
| InteractionExitCapability.as | 持有/退出交互 sheet 与能力，维护 Cancel 提示 |
| InteractionCancelCapability.as | 响应 Cancel 输入，主动退出交互 |
| InteractionCapability.as | 交互期间运行的玩法能力基类，`SupportsInteraction` 钩子 |
| InteractionWidget.as | 交互浮窗 `UInteractionWidget : UTargetableWidget`（含全屏双人偏移逻辑） |
| InteractionComponentVisualizer.as | 编辑器可视化 Focus/Action 形状（仅 EDITOR） |

# LevelSpecific / Dentist / Boss（牙医 Boss 战）

> 路径：`LevelSpecific/Dentist/Boss/`（123 文件）

## 职责

实现整关唯一的 Boss——"The Dentist"，一台三对机械臂的机器人牙医。它站在旋转蛋糕竞技场（`ADentistBossCake`）中央，按脚本化的多阶段战斗，用七套牙科工具依次攻击两名玩家（变身的牙齿），最后被玩家砸掉双手弱点、触发终结技并进入追逐段。这是一个由**动作队列 + 大状态机**驱动的、高度编排的 Boss，而非通用 AI。

## 内部架构

### Boss 本体 `ADentistBoss : AHazeActor`

- **工具容器**：`TMap<EDentistBossTool, ADentistBossTool> Tools` 持有全部工具实例；`BeginPlay` 通过 `TListedActors<>` 收集场景中放置的工具并 `AttachTool` 挂到对应机械臂骨骼（`LeftUpperAttach` 等）。工具枚举 `EDentistBossTool`：Drill / MioChair / ZoeChair / Dentures / ToothBrush / CupLeft/Middle/Right / Scraper / Hammer / ToothPasteTube。
- **弱点与血量**：左右手各有 `WeakpointMesh` + `UBasicAIHealthComponent` + `UDentistBossArmWeakpointHealthBarComponent` + `UDentistGroundPoundAutoAimComponent`（吸引玩家砸地）。`OnGroundPoundedOn` → `CrumbArmTakeDamage` 扣手部血量。整体血量 `CurrentHealth = TotalHealthPerArm * 2`。
- **动作队列**：`TArray<UHazeActionQueueComponent> ActionQueueComps`（`QueueNum = 3`）。三条并行队列让 Boss 能同时"播动画/移工具/等玩家"。
- **状态机**：`EDentistBossState CurrentState`（Start / RestrainedInChair / ToothBrushOne / DentureSpawning / CupSpawning / SpinningCake / … / Defeated / Chase）。
- **动画/IK 中枢**：大量 `TInstigated<>` 变量（`CurrentAnimationState` / `CurrentIKState` / `LeanBlendSpaceValues` / `LookAtEnabled`）+ 六臂 IK 变换，供 AnimBP 读取；`Tick` 里全部塞进 `TEMPORAL_LOG` 便于调试。
- **能力表** `DentistBossSheet`：挂 `ActionSelection` / `LookAtPlayer` / `MalfunctioningEyes` / 左右臂 `ArmDeath` / `ArmSwatAwayPlayer` / `LeanBlendSpace` 等常驻能力，用 `InitialStoppedSheets` + `ToggleActivated` 控制启停。

### 编排核心 `UDentistBossActionSelectionCapability`

全 Boss 战的"导演"。`ShouldActivate` 要求 Boss 激活、有控制权、且三条动作队列全空——即上一段动作演完才选下一段。`OnActivated` 是一个巨大的 `switch(CurrentState)`，每个 case 把一串动作**入队**（不是立即执行），例如 `RestrainedInChair`：等玩家挣脱椅子 → 关摄像机 → 等钻头钻完 → 收钻头 → `SetState(ToothBrushOne)`。类里封装了约 80 个入队辅助方法（`SetAnimState / ToggleTool / DrillAttackCurrentTarget / CupGrabPlayer / ScraperAttack / ToothPasteAttack …`），每个都是往 `ActionQueueComps[i]` 塞 `.Capability()` / `.Idle()` / `.Event()`。还带一整排 `Dev_*` 调试直跳。

### 七套工具 `DentistTools/`

每套工具是 `ADentistBossTool` 子类 + 一组该工具专属能力（挂在工具或 Boss 上，通过动作队列 `.Capability()` 激活）：

- **Drill 钻头**：钻蛋糕、旋转竞技场、抖动/倾斜玩家相机（`DrillImpaleCake / DrillRotateCake / DrillShakePlayer`）。
- **Dentures 假牙**：会跳的独立单位，咬 Boss 自己的手制造弱点；可被玩家骑乘（`DentureRiding/` 一整套：交互/咬手指/相机/跳跃/旋转）。
- **Cup 纸杯**：三选一抓人小游戏，`DentistBossCupManager` + `DentistBossCupSortingComponent`（联机排序）控制"猜杯"逻辑。
- **ToothBrush / ToothPasteTube 牙刷+牙膏**：横扫攻击 + 抛射牙膏团（`ToothPasteGlob`，走 `HazeActorNetworkedSpawnPool` 池化）。
- **Scraper + Hammer 刮匙+锤子**：刮匙钩住玩家、锤子敲击、最终把牙齿**劈成两半**（触发玩家 Split）。
- **Chair 椅子**：开场把两名玩家束缚在牙科椅上（`ChairRestraint / ChairWaitUntilEscaped`）。

### 追逐与终结 `Player/` `SideStoryExit/`

`DentistBossPlayerDrillFinisherCapability` + `FinisherDoubleInteractActor`（双人交互）触发终结技；`DentistChaseBoss` / `DentistCutsceneBoss` 是击败后追逐段与过场用的独立 Boss 变体；`DentistBossChaseGameOverCapability` 处理追逐失败。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| → | 动作队列系统 | 时序编排 | `ActionSelectionCapability` 全靠 `Dentist.ActionQueueComps[i].Capability/Idle/Event`（`UHazeActionQueueComponent`）驱动 |
| → | 牙齿玩家 | 请求能力 / 砸地 | `RequestComp.PlayerCapabilities.Add(n"DentistBossPlayerWiggleRotationCapability")`；`ToothMovementResponseComp.OnGroundPoundedOn` → 扣弱点血 |
| → | 玩家能力屏蔽 | 标签互斥 | `BlockCapabilities(EHazeSelectPlayer::Both, CapabilityTags::Movement/Outline, CameraTags::CameraControl, n"StickWiggle")` 在演出段冻结玩家 |
| → | AI 血量系统 | 复用 | 双手 `UBasicAIHealthComponent`；`ProgressToState` 里 `LeftHandHealthComp.Die()` 推进阶段 |
| → | 相机系统 | 直接调用 | `ToggleCamera` 切 `StartCamera / HookedCamera / FinisherCamera`（`AHazeCameraActor` + `UHazeCameraComponent`） |
| → | 联机 | Crumb | `CrumbArmTakeDamage / CrumbDie / CrumbSwapTools / CrumbStartCupSorting` 均 `UFUNCTION(CrumbFunction)` |
| → | 生成池 | 池化投射物 | 牙膏团 `ToothPastePool = HazeActorNetworkedSpawnPoolStatics::GetOrCreateSpawnPool(...)` |
| ← | 玩家分裂 | 触发 Split | 锤子 `HammerSplitToothCapability` / `bHammerSplitPlayer` 触发玩家侧 Tooth Split |

## 关键文件

- `Boss/DentistBoss.as` — Boss 本体 `ADentistBoss`：工具容器、弱点、动作队列、状态/动画/IK 数据、伤害与死亡
- `Boss/Capabilities/Actions/DentistBossActionSelectionCapability.as` — 战斗导演：状态机 switch + 全部入队辅助方法（1440 行）
- `Boss/DentistBossSettings.as` / `DentistBossTimings.as` — 数值与时序常量（Settings 四件套）
- `Boss/DentistTools/DentistBossTool.as` — 工具基类 `ADentistBossTool`
- `Boss/DentistTools/Cup/DentistBossCupManager.as` + `Cup/DentistBossCupSortingComponent.as` — 猜杯小游戏与联机排序
- `Boss/DentistTools/Dentures/DentureRiding/` — 假牙骑乘子系统（9 文件）
- `Boss/Capabilities/DentistBossPlayerDrillFinisherCapability.as` — 终结技
- `Boss/DentistChaseBoss.as` — 击败后追逐段 Boss 变体

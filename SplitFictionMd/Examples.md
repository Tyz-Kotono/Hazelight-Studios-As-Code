# Examples — 官方教学代码库

> 路径：`Examples/`（69 文件）
>
> 本模块与其他所有模块本质不同：它**不参与游戏运行**，是 Hazelight 为开发者准备的**官方"教科书 + 可运行范本"**。全部文件以 `Example_*` / `Template_*` 命名，配有密集讲解注释，多数顶部挂 wiki 链接（`// See http://wiki.hazelight.se/...`）。
>
> 文中所有代码引用均使用 **AS 相对路径**（从 `SplitFiction` 起，如 `Examples/Capabilities/Example_Capability.as`）。

---

## 一、它是什么、为什么读它

| 维度 | 其他模块 | Examples |
|---|---|---|
| 参与游戏运行 | 是 | **否**（不被真实关卡引用） |
| 目的 | 实现功能 | **教开发者用引擎特性** |
| 注释 | 极简 | **密集讲解式**，每函数一段 doc |
| 命名 | 功能名 | 一律 `Example_*` / `Template_*` |

**核心价值：它是本套引擎设计意图的"官方注脚"。** 前面各模块文档里归纳出的机制（Capability 生命周期、Crumb 联机、EffectEvent、TInstigated、ComposableSettings、Targetable…），在这里都能找到官方**明示的最小正确用法**——文档若与 Examples 冲突，以 Examples 为准。

---

## 二、三层结构

Examples 按抽象层次分三层：**语言 → 框架 → 系统对接**。

```
Examples/
├── 层1 语言语法      Angelscript/         (14) —— 与游戏无关，纯 AngelScript 语法
├── 层2 框架核心      Capabilities/ (9)  Features/ (14) —— Haze 引擎最核心抽象
└── 层3 系统对接      AI/Aiming/Animation/Camera/Movement/Network/Physics/
                     Rendering/Editor/Development/Disables/Interaction/Math/Spawner (32)
                                       —— 每个具体系统的 "hello world"
```

---

## 三、层1 · AngelScript 语言语法（`Angelscript/`，14 文件）

给"没写过 Haze AngelScript 的人"的入门，与游戏逻辑无关。

| 文件 | 教什么 | 要点 |
|---|---|---|
| `Examples/Angelscript/Example_Actor.as` | 脚本类继承 `AHazeActor` 并暴露给蓝图 | `BlueprintOverride`(BeginPlay/EndPlay/ConstructionScript…)；无 `UFUNCTION` 的方法热重载更快 |
| `Examples/Angelscript/Example_Array.as` | `TArray<T>` | `Add/Num/Contains/FindIndex/RemoveAt`；`Contains/Find/Remove` 都是线性查找 |
| `Examples/Angelscript/Example_Map.as` | `TMap<K,V>` | `[]` 键不存在会**抛异常**，先 `Contains`；`[]`/迭代返回引用可直接改 |
| `Examples/Angelscript/Example_Struct.as` | `struct` 自动绑定 Unreal | 改 struct 任何属性（无论是否 UPROPERTY）都需 full reload |
| `Examples/Angelscript/Example_Enum.as` | 脚本枚举自动绑定 | `EExampleEnum(0)` int 转枚举，无需额外标记 |
| `Examples/Angelscript/Example_Delegates.as` | `delegate`(单播) vs `event`(多播) | event 只能作 UPROPERTY 不能当参数；绑定目标须是 `UFUNCTION`；`n""` 字面 FName 编译期解析更快 |
| `Examples/Angelscript/Example_Functions.as` | 函数暴露为蓝图节点 | 函数上方注释成为节点 tooltip；`&out` 参数变输出针脚 |
| `Examples/Angelscript/Example_FormatString.as` | `f"..."` 格式化字符串 | `{expr =}` 打印表达式名；Python 风格 `:.3 / :010d / :#x / :>40`；枚举 `:n` 只打名字 |
| `Examples/Angelscript/Example_ConstructionScript.as` | ConstructionScript 动态建组件、算派生属性 | `UBillboardComponent::Create(this, n"Name")` |
| `Examples/Angelscript/Example_Overlaps.as` | Actor/Component 重叠检测 | Actor 覆写 `ActorBeginOverlap`；组件是**事件**（`OnComponentBeginOverlap`），须 BeginPlay 里绑 UFUNCTION |
| `Examples/Angelscript/Example_AccessSpecifiers.as` | `access` 关键字细粒度访问控制 | 比 private/protected 更细；`inherited` 才含子类；通配 `* (editdefaults, readonly)` |
| `Examples/Angelscript/Pickups/Example_Pickup.as` | 拾取系统搭建（WIP） | `UPickupComponent` + `UInteractionComponent`；`UPlayerPickupComponent::Get(Player).PickUp(...)` |
| `Examples/Angelscript/Specifiers/Example_FunctionSpecifiers.as` | 函数说明符全集 | `BlueprintPure/BlueprintEvent/NetFunction(+Unreliable)/DevFunction/CallInEditor`；`NetFunction` 默认可靠有序、名字加 Net 前缀 |
| `Examples/Angelscript/Specifiers/Example_PropertySpecifiers.as` | 属性说明符/meta 全集 | `DefaultComponent/RootComponent/Attach`；meta `EditCondition(+Hides)/ClampMin-Max/ArraySizeEnum/MakeEditWidget` |

---

## 四、层2 · 框架核心（精华）

### 4.1 Capabilities/（9 文件）—— 引擎最核心的抽象

能力系统是整个 Haze 框架的行为单元。这 9 个文件构成一条**难度递进的学习路径**：

```
1. Template_Capability      空骨架，认识 6 个生命周期钩子
2. Example_Capability       真实条件 + Block/Tag 完整讲解
3. Example_PlayerCapability 玩家专属基类（自动暴露 Player）
4. Example_AdvancedCapability  Tick 组抢占、InterruptsCapabilities
5. Example_NetworkCapabilities Crumb 网络 + 参数同步（引入"两端"概念）
6. Example_RequestCapabilities Actor 驱动动态请求能力
7. Example_CapabilitySheets    sheet 批量启停编排
8. Example_CompoundCapability   行为树式组合（最抽象）
9. Template_MoveCapability      集大成实战移动模板
```

| 文件 | 教什么 | 关键 API / 要点 |
|---|---|---|
| `Examples/Capabilities/Template_Capability.as` | 可复制的最简能力骨架 | 全套 `BlueprintOverride` 空实现，`ShouldActivate` 返回 true |
| `Examples/Capabilities/Example_Capability.as` | 能力入门：完整生命周期 | `Setup/ShouldActivate/ShouldDeactivate/OnActivated/OnDeactivated/TickActive/PreTick`；`Owner.BlockCapabilities(tag, Instigator=this)` 配对 Unblock；`PreTick` 无视激活状态且最先跑 |
| `Examples/Capabilities/Example_PlayerCapability.as` | 玩家专属能力 | `UHazePlayerCapability` 自动暴露 `Player`；只能挂玩家 |
| `Examples/Capabilities/Example_AdvancedCapability.as` | Tick 组进阶与抢占 | `SeparateInactiveTick(group, order)`、`InterruptsCapabilities(tag)`；本地能力**不能打断**网络能力 |
| `Examples/Capabilities/Example_NetworkCapabilities.as` | 能力网络同步 | `NetworkMode = EHazeCapabilityNetworkMode::Crumb`；带 struct 的 `ShouldActivate(Params&)`；`Should*` 只在控制端跑但两端同步激活；Crumb 能力远端不能被 Block，只会变 `IsQuiet()` |
| `Examples/Capabilities/Example_RequestCapabilities.as` | Actor 向玩家动态请求能力 | `UHazeRequestCapabilityOnPlayerComponent.PlayerCapabilities.Add(...)`；BouncePad 示例展示 Actor→玩家组件→能力全流程 |
| `Examples/Capabilities/Example_CapabilitySheets.as` | 能力表启停 | `Player.StartCapabilitySheet/StopCapabilitySheet(sheet, Instigator=this)`；不能启动未被请求过的 sheet（会报错） |
| `Examples/Capabilities/Example_CompoundCapability.as` | 复合能力（行为树式） | `GenerateCompound()` + `RunAll(.Add)/Sequence(.Then)/Selector(.Try)/StatePicker(.State)`；`UHazeChildCapability` |
| `Examples/Capabilities/Template_MoveCapability.as` | 实战移动能力模板 | `UPlayerMovementComponent` + `SetupSteppingMovementData` + `PrepareMove/AddGravityAcceleration/ApplyMoveAndRequestLocomotion`；`HasControl()` 分支 vs `ApplyCrumbSyncedGroundMovement` |

### 4.2 Features/（14 文件）—— 引擎横切机制的最小示例

这 14 个正是前面分析 Core 时反复遇到的横切机制的"官方标准用法"。

| 文件 | 教什么 | 关键要点（注释强调的坑/警告） |
|---|---|---|
| `Examples/Features/Example_EffectEvents.as` | 玩法→表现的 pub-sub 事件分发 | `UHazeEffectEventHandler` + `Trigger_<Event>(actor, params)`；⚠️ 注释明写 **"DO NOT USE FOR GAMEPLAY"**（仅表现层）；事件最多一个参数 struct |
| `Examples/Features/Example_TInstigated.as` | 发起者+优先级分层值 | `TInstigated<T>`：`Apply(value, Instigator, priority)/Clear/Get`；所有 instigator 清除后回落 `SetDefaultValue` |
| `Examples/Features/Example_ComposableSettings.as` | 多资产按勾选+优先级合并设置 | `GetSettings` 返回**活引用**（后续 Apply 会立即反映）；struct 属性需打 `Meta=(ComposedStruct)` |
| `Examples/Features/Example_TargetableComponent.as` | 可瞄准/可交互框架 | 重写 `CheckTargetable(FTargetableQuery&)`；每个 `TargetableCategory` 只有一个 primary；返回 false = 完全不可见不可用 |
| `Examples/Features/Example_TemporalLog.as` | 按时间轴记录状态+可视化调试 | `TEMPORAL_LOG(this)` 链式 `.Value/.Capsule/.Line/.Status/.Section/.Page`；每 actor 只能一个 `Status`；`OnLogState`(每帧) vs `OnLogActive`(仅激活帧) |
| `Examples/Features/Example_Timer.as` | 定时器库 | `Timer::SetTimer(obj, n"Func", Delay)`；⚠️ `ClearTimer` 中止定时器，`Invalidate` **只断句柄不清定时器** |
| `Examples/Features/Example_Timeline.as` | 不依赖 actor 的时间轴曲线 | `FHazeTimeLike`：`Duration/bLoop/bFlipFlop`、`Play/Reverse`；绑定的更新/完成函数**必须是 `UFUNCTION()`** |
| `Examples/Features/Example_ActionQueue.as` | 委托/能力串成顺序流程 | `FHazeActionQueue`：`Idle/Capability/Event/Duration/Parallel/ScrubTo`；**含 Capability 动作的队列不可 Scrub**；`Duration` 委托每帧收 0→1 Alpha |
| `Examples/Features/Example_StructQueue_BossAttacks.as` | struct 队列驱动 Boss 攻击 | `FHazeStructQueue`：每个 struct 类型对应一个能力；能力靠 `Start` 领队首、`Finish` 推进；攻击能力用 `NetworkMode=Crumb` |
| `Examples/Features/Example_ConstrainedPhysicsValue.as` | 纯数学模拟"物理感"单浮点值 | `FHazeConstrainedPhysicsValue`：`SpringTowards/AddImpulse/Update`；**不是真物理**、无碰撞；**必须每帧 `Update`** |
| `Examples/Features/Example_ListedActor.as` | 挂组件即可被全局列表查询 | `UHazeListedActorComponent` + `TListedActors<T>`（可迭代）/`.GetSingle()`；免手动维护管理器列表 |
| `Examples/Features/Example_SoftObjectPtr.as` | 软引用跨 sublevel 不强制加载 | `TSoftObjectPtr<T>`/`TSoftClassPtr<T>`：`.Get()`（未加载时**可能 null**）/`.LoadAsync(回调)` |
| `Examples/Features/Example_ContentBrowserFilter.as` | 内容浏览器自定义过滤器 | `UHazeScriptContentBrowserFilter`；⚠️ 先比较 name/string 再考虑加载资产；用 `#ifdef false` 包住避免污染菜单 |
| `Examples/Features/Example_ContentBrowserContextMenuAction.as` | 内容浏览器右键菜单条目 | `UHazeCustomContextMenuAction`；`ShouldAdd*` 要廉价，先路径比较 |

---

## 五、层3 · 各系统对接示例（32 文件）

每个子目录是对应系统的"hello world"，可对照 Core/Gameplay 各模块文档深入。

### AI/（5）—— 复合能力搭建行为树 AI
- `Examples/AI/ExampleRunAwayAI.as` — 用 `UHazeCompoundCapability` + `GenerateCompound()` 搭 AI 行为树（`Selector.Try()`/`Sequence.Then()`），Actor 把 CompoundCapability 加进 `DefaultCapabilities`，`NetworkMode=Crumb`。
- `Examples/AI/ExampleAIIdleBehavior.as` — 行为树子能力(待机浮动)：`UHazeChildCapability`、`ActiveDuration`、`CompoundNetworkSupport`。
- `Examples/AI/ExampleAIRunAwayFromPlayerBehavior.as` — 逃离玩家：距离阈值(<1000 进/>1500 退)实现**迟滞**。
- `Examples/AI/ExampleAIStartleBehavior.as` — 受惊：`ActiveDuration>=1.0` 定时退出。
- `Examples/AI/ExampleAIStunnedBehavior.as` — 眩晕：`Player.IsAnyCapabilityActive(n"Jump")` 触发，持续 3 秒。

### Aiming/（2）
- `Examples/Aiming/ExampleAimingCapability.as` — 进入瞄准模式：`UPlayerAimingComponent::Get`、`StartAiming/StopAiming`、`FAimingSettings`(bShowCrosshair/bUseAutoAim)。
- `Examples/Aiming/ExampleAimingAutoTarget.as` — 自定义自动锁定：`UAutoAimTargetComponent` 覆写 `CheckTargetable`。

### Animation/（4）—— ABP / Feature / DataAsset
- `Examples/Animation/Example_BaseAnimInstance.as` — NPC 单 ABP 基类：`UHazeAnimInstanceBase`、`BlueprintUpdateAnimation`、`GetAnimBoolParam`。
- `Examples/Animation/Example_FeatureAnimInstance.as` — 可切换子动画：`UHazeFeatureSubAnimInstance`、`CanTransitionFrom/OnTransitionFrom`、`AnimNotify_` 前缀。
- `Examples/Animation/Example_LocomotionFeature.as` — 动画数据资产：`UHazeLocomotionFeatureBase`、`FLocomotionFeature*Data`、命名规范。
- `Examples/Animation/Example_BoneFilterAsset.as` — 骨骼过滤模板参考（全文注释，列各预设骨骼权重表）。

### Camera/（2）
- `Examples/Camera/Example_CameraSettings.as` — 应用/清除相机设置：`UCameraSettings::GetSettings`、`FOV/PivotOffset.Apply/Clear`、`EHazeCameraPriority`（默认互斥需唯一优先级）。
- `Examples/Camera/Example_PointOfInterest.as` — 相机聚焦兴趣点：`SetFocusToActor/Component`、`ApplyPointOfInterest/ApplyClampedPointOfInterest`。

### Movement/（2）
- `Examples/Movement/Example_MovementCapability.as` — 移动能力：`UHazeMovementComponent::Get`、`EHazeTickGroup`(BeforeMovement/InfluenceMovement/…)、`HasMovedThisFrame`、`PrepareMove/ApplyMove/AddGravityAcceleration`。
- `Examples/Movement/Example_CustomMovement.as` — 自定义移动数据/解算器：`USteppingMovementData`(DefaultResolverType) 配对 `USteppingMovementResolver`(RequiredDataType)。

### Network/（1）
- `Examples/Network/Example_SyncedValues.as` — Crumb 同步值：`UHazeCrumbSyncedFloat/Vector/RotatorComponent`、`HasControl()` 写 `Value`、`OverrideSyncRate/SnapRemote/BlockSync`；自定义结构用 `UHazeCrumbSyncedStructComponent` 实现 `InterpolateValues`。

### Physics/（1）
- `Examples/Physics/Example_PhysicsTracing.as` — 射线/形状检测：`Trace::InitChannel/InitFromPlayer`、`FHazeTraceSettings`(IgnoreActor/UseBoxShape)、`QueryTraceSingle/Multi`、`Overlap::QueryShapeOverlap`、`TraceDebug::`。

### Rendering/（3）
- `Examples/Rendering/Example_MaterialParameters.as` — 材质参数：`CreateDynamicMaterialInstance` + `SetScalar/Vector/TextureParameterValue`。
- `Examples/Rendering/Example_RenderTargets.as` — GPU 绘制：`Rendering::CreateRenderTarget2D`、Canvas 画形状 / 材质整绘 / 双缓冲反馈循环三法。
- `Examples/Rendering/Example_Textures.as` — CPU 更新纹理：`CreateTexture2D` + `UpdateTexture2D(TArray<FVector4f>)`；仅适合小纹理。

### Development/（3）—— 开发工具
- `Examples/Development/Example_DevInput.as` — 注册调试输入：`FHazeDevInputInfo` 或 `UHazeDevInputHandler`。
- `Examples/Development/Example_DevMenu.as` — 开发菜单：`UHazeDevMenuEntryWidget`(WBP) vs `UHazeDevMenuEntryImmediateWidget`(即时)；`DevMenu::RequestImmediateDevMenu`。
- `Examples/Development/Example_DevToggle.as` — 调试开关：`FHazeDevToggleBool/Group/Category/Preset`。

### Editor/（3，均 `#if EDITOR`）
- `Examples/Editor/Example_EditorMenuExtensions.as` — 右键/工具栏菜单：`UScriptActorMenuExtension`/`UScriptAssetMenuExtension`、`UFUNCTION(CallInEditor)`；带 Actor 参数会对每个选中项调一次。
- `Examples/Editor/Example_EditorSubsystemInput.as` — 编辑器子系统输入/绘制：`UHazeEditorSubsystem`、`OnLevelEditorKeyInput/Click`、覆盖层画 UI。
- `Examples/Editor/Example_SelectableVisualizer.as` — 组件可视化器/可拖拽手柄：`UHazeScriptComponentVisualizer`、`SetHitProxy`、`HandleInputDelta`。

### Disables/（2）
- `Examples/Disables/Example_Disable.as` — Actor 整体禁用：`AddActorDisable/RemoveActorDisable`（引用计数）+ `OnActorDisabled/OnActorEnabled`。
- `Examples/Disables/Example_FunctionallityBlocks.as` — 分项屏蔽：`AddActorTickBlock/VisualsBlock/CollisionBlock`（带 instigator）。

### Interaction/（1）
- `Examples/Interaction/Example_InteractionWithCapability.as` — 交互驱动能力：`UInteractionComponent`(InteractionCapability/MovementSettings) + `UInteractionCapability` 重写 `SupportsInteraction()`。

### Math/（2）
- `Examples/Math/Example_RuntimeSpline.as` — 运行时样条：`FHazeRuntimeSpline`(AddPoint/Tension/Looping/GetLocation)，可每帧新建。
- `Examples/Math/Example_StatelessMovement.as` — 无状态时间驱动运动：`Time::PredictedGlobalCrumbTrailTime` + `Math::Wrap/GetMappedRangeValueClamped`，**网络安全**的往返运动。

### Spawner/（1）
- `Examples/Spawner/Example_HazeActorSpawner.as` — Actor 生成器：`AHazeActorSpawnerBase` + `UHazeActorSpawnPattern*`(Single/Interval/Wave/JoinTeam) 组合编排波次；`UHazeActorRespawnableComponent`。

---

## 六、"想学 X 看哪个文件"速查

| 我想学… | 看这个 | 对应正式文档 |
|---|---|---|
| 写一个能力 | `Examples/Capabilities/Template_Capability.as` → `Example_Capability.as` | [Core/README](./Core/README.md) 核心概念 |
| 写移动能力 | `Examples/Capabilities/Template_MoveCapability.as` | [Core/Movement](./Core/Movement/) |
| 联机同步值 | `Examples/Network/Example_SyncedValues.as` | [Core/Systems/Network](./Core/Systems/Network/)、[Communication](./Core/Communication.md) |
| 玩法触发音效/特效 | `Examples/Features/Example_EffectEvents.as` | Core/Audio（EffectEvent 系统） |
| 分层参数/设置 | `Examples/Features/Example_TInstigated.as`、`Example_ComposableSettings.as` | [Communication](./Core/Communication.md) 机制 5 |
| 相机设置 | `Examples/Camera/Example_CameraSettings.as` | [Core/Camera](./Core/Camera/) |
| AI 行为树 | `Examples/AI/ExampleRunAwayAI.as` | Gameplay/AI |
| 时序日志调试 | `Examples/Features/Example_TemporalLog.as` | [GUI/Debug/TemporalLog](./GUI/Debug/TemporalLog/) |
| 编辑器工具 | `Examples/Editor/*` | [Editor/Patterns](./Editor/Patterns.md) |
| 开发菜单/开关 | `Examples/Development/*` | [GUI/Debug](./GUI/Debug/) |
| 物理射线检测 | `Examples/Physics/Example_PhysicsTracing.as` | [Core/SplineAndSpace/Collision](./Core/SplineAndSpace/Collision/) |

---

## 七、结论

Examples 是这套引擎的**权威学习入口**与**设计意图注脚**：
- **层1** 教语言，**层2**（Capabilities + Features）是精华——覆盖能力系统学习路径与全部横切机制的标准用法，**层3** 教各系统对接。
- 它的注释里藏着大量**官方警告与最佳实践**（如 EffectEvent "别用于 gameplay"、ConstrainedPhysicsValue "非真物理"、Timer 的 Clear vs Invalidate、SoftObjectPtr 可能为 null），这些是踩坑经验的结晶。
- 阅读顺序建议：先 `Angelscript/` 打语言基础 → `Capabilities/` 按 1-9 递进 → 需要某机制时查 `Features/` → 做具体系统时看层3 对应目录。

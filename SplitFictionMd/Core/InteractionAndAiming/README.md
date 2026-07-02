# Core / InteractionAndAiming 功能域总览

> 玩家与世界中可瞄准/可交互目标之间的统一目标查询体系：Targetable 提供地基，Interaction 与 Aiming 在其上各自构建“近身交互”与“自动瞄准”两套玩法。

## 一、三模块定位

| 子模块 | 文件数 | 角色 | 核心组件 |
|---|---|---|---|
| Targetable | 8 | 地基：每帧分类目标查询/评分/选 primary，驱动 Widget 与 Outline | `UTargetableComponent`(USceneComponent)、`UPlayerTargetablesComponent` |
| Interaction | 8 | Targetable 的子类化应用：近身交互（move-to→网络锁→sheet） | `UInteractionComponent : UTargetableComponent`、`UPlayerInteractionsComponent` |
| Aiming | 14 | Targetable 的独立消费者：自动瞄准修正射击 | `UPlayerAimingComponent`、`UAutoAimTargetComponent : UTargetableComponent` |

## 二、地基关系图

```
                    ┌─────────────────────────────────────────────┐
                    │            Targetable（地基）                  │
                    │                                               │
                    │  UTargetableComponent (USceneComponent)       │
                    │   · TargetableCategory(FName) 分类             │
                    │   · CheckTargetable(Query) 评分钩子            │
                    │        ▲ 子类化              ▲ 子类化           │
                    │        │                     │                │
                    │  UPlayerTargetablesComponent（玩家级中枢）       │
                    │   · 按分类注册/反注册                            │
                    │   · 每帧 GetQueryThisFrame：排序→评分→选 primary │
                    │   · CurrentAimingRay / OverrideTargetableAimRay │
                    │   · ShowWidgetsForTargetables (UTargetableWidget)│
                    │   · ShowOutlinesForTargetables(UTargetableOutline)│
                    └───────┬──────────────────────────┬────────────┘
                            │ 分类 n"Interaction"         │ 分类 n"AutoAim"
                            │                            │
            ┌───────────────▼──────────┐   ┌─────────────▼─────────────────┐
            │   Interaction（交互）       │   │      Aiming（瞄准）             │
            │                          │   │                               │
            │ UInteractionComponent    │   │ UAutoAimTargetComponent       │
            │   : UTargetableComponent │   │   : UTargetableComponent      │
            │                          │   │                               │
            │ UInteractionEnterCap →   │   │ UPlayerAimingComponent（独立）  │
            │   GetPrimaryTarget       │   │  推入 CurrentAimingRay /        │
            │   → move-to → 网络锁      │   │  OverrideTargetableAimRay      │
            │   → sheet 流程            │   │  读回 n"AutoAim" primary →修正射击│
            │ UPlayerInteractionsComp  │   │ Capabilities/ 输入采集 + 同步    │
            └──────────────────────────┘   └───────────────────────────────┘
```

核心数据流（每帧）：`UPlayerAimingUpdateCapability` 与各交互能力在 PreTick/Tick 调用 `UPlayerTargetablesComponent`；后者跑一次 `GetQueryThisFrame`（按距离排序候选→逐个 `CheckTargetable` 评分→挑出每分类的 primary），结果被三方共享：交互能力取 `n"Interaction"` primary 启动交互，瞄准组件取 `n"AutoAim"` primary 弯折射击方向，Widget/Outline 系统据此渲染。

## 三、Targetable→Interaction / Aiming 的两种复用方式

| 复用方式 | Interaction | Aiming |
|---|---|---|
| 与 Targetable 的关系 | `UInteractionComponent` 直接继承 `UTargetableComponent`（默认分类 `n"Interaction"`） | `UAutoAimTargetComponent` 继承 `UTargetableComponent`（默认分类 `n"AutoAim"`）；`UPlayerAimingComponent` 是**独立**的 UActorComponent 消费者 |
| 评分钩子 `CheckTargetable` | 检查 Focus/Action 区域命中、互斥、条件，再调 `Targetable::ScoreCameraTargetingInteraction` | 计算 aim ray 弯折角与遮挡，调 `Targetable::RequireAimToPointNotOccluded` |
| 查询入口 | 能力调 `PlayerTargetablesComp.GetPrimaryTarget(UInteractionComponent)` | 组件调 `TargetablesComp.GetPrimaryTargetForCategory(n"AutoAim")` |
| AimRay 来源 | 默认相机视线 | `UPlayerAimingComponent` 把 `GetPlayerAimingRay()` 写入 `TargetablesComp.CurrentAimingRay` 及 `OverrideTargetableAimRay` |

## 四、本功能域中的五种协作机制

| 机制 | 域内实例（grep 实证） |
|---|---|
| 1. 直接调用 `Type::Get` | 各能力 `UPlayerTargetablesComponent::GetOrCreate(Player)`、`UPlayerAimingComponent::Get(Player)`、`UAutoAimTargetComponent` Cast 等 |
| 2. Crumb 网络 | `UInteractionEnter/Exit/CancelCapability` 全部 `NetworkMode = Crumb`；`UAiming2DSyncDirectionCapability` 用 `UHazeCrumbSyncedRotatorComponent` + `CrumbFunction` 同步瞄准方向 |
| 3. 标签阻塞 | `Targetable::RequireCapabilityTagNotBlocked`；`UAimingMouseDirectionalOnlyInputCapability2D` `BlockCapabilities(n"MouseCursorInput")`；交互能力的 `BlockExclusionTags n"UsableDuringMoveTo"` |
| 4. 委托 event/Broadcast | `UInteractionComponent.OnInteractionStarted/Stopped`（`FOnInteractionEvent`）、`FInteractionCondition` delegate、move-to 回调 `FOnMoveToEnded` |
| 5. 可组合设置 ApplySettings | `UPlayerAimingSettings : UHazeComposableSettings`，`Player.ApplyDefaultSettings(DefaultAimingSettings)` |

双人协作：所有 Targetable 按 `EHazeSelectPlayer UsableByPlayers` 区分 Mio/Zoe；查询、Widget、Outline 全程 `TPerPlayer` / `Game::Mio`、`Game::Zoe` 分别处理；交互用 `UNetworkLockComponent` 保证两名玩家不会同时占用同一交互。

## 五、子模块文档

- [Targetable/README.md](./Targetable/README.md)
- [Interaction/README.md](./Interaction/README.md)
- [Aiming/README.md](./Aiming/README.md)

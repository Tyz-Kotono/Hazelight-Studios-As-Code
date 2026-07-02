# Core / PlayersAndHealth

> 职责：定义双人协作游戏的玩家本体、生命/伤害/死亡/重生系统、玩家偏移访问器与通用队伍机制，是 Split Fiction 所有玩法系统的承载根基。

Split Fiction 是双人合作游戏，两名玩家分别是 `EHazePlayer::Mio`（品红）与 `EHazePlayer::Zoe`（青柠），均为 `APlayerCharacter : AHazePlayerCharacter`。本功能域包含 4 个子模块：

| 子模块 | 文件数 | 职责 |
|--------|--------|------|
| [Players](Players/README.md) | 22（含 Debug 子目录 14） | 玩家 Actor 本体、配色、变体换装、高光、视角模式、聚焦设置、调试工具 |
| [PlayerHealth](PlayerHealth/README.md) | 38 | 生命值、伤害、死亡、重生、Game Over 全流程与各类致死体积/特效 |
| [Offset](Offset/README.md) | 1 | 玩家相机/根/网格偏移组件的 mixin 访问器 |
| [Team](Team/README.md) | 3 | 通用按名分组队伍系统（`UHazeTeam` + `UHazeTeamManager`），主要服务于 AI |

## 4 模块关系

```
                    APlayerCharacter : AHazePlayerCharacter
                    (Players 模块定义本体)
                              │
        ┌─────────────────────┼──────────────────────────┐
   挂载组件                  注册能力                    访问器/数据
        │                     │                          │
  UPlayerHealthComponent  健康能力集（PlayerHealth）   GetCameraOffsetComponent
  UPlayerRespawnComponent   PlayerDeathCapability       等 mixin（Offset）
  UPlayerVariantComponent   PlayerRespawnCapability
  UPlayerMovement-          PlayerGameOverCapability
    PerspectiveModeComp     PlayerHealthRegeneration...
        │
   PlayerColor / PlayerHighlight / PlayerFocus（Players）

   Team 模块独立：AI 等系统调用 Player.JoinTeam(n"...") 分组协作，
   与玩家本体松耦合（AHazeActor 级别，非玩家专属）。
```

- **Players** 是核心：`APlayerCharacter` 在默认值里挂载所有健康组件并在 `DefaultCapabilities` 里注册 PlayerHealth 模块的全部能力。
- **PlayerHealth** 依赖 Players：所有能力/组件以 `AHazePlayerCharacter` 为 Owner，通过 `UPlayerHealthComponent::Get(Player)` 协同。
- **Offset** 是 Players 的轻量补充：仅暴露玩家内置偏移组件的便捷访问 mixin。
- **Team** 与玩家本体松耦合：作用于 `AHazeActor` 级别，全游戏共用（实测主要消费者为 `Gameplay/AI` 与各 `LevelSpecific` 关卡），并非玩家专属。

## 伤害 / 死亡 / 重生流程图

```
[伤害来源]
  ├─ UDealPlayerDamageComponent.DealDamage(Target, ...)   （需追踪多种伤害类型时）
  └─ mixin DamagePlayerHealth(Player, Damage, ...)        （便捷直调）
            │
            ▼
  UPlayerHealthComponent::DamagePlayer()           [仅 HasControl 侧]
            │
   CanTakeDamage()? ── 否（god模式/无敌/死亡/过场）→ 触发 AvoidedDamage 事件，返回
            │ 是
   WouldDieFromDamage()?
       ├─ 否 ─► CrumbDamagePlayer()  → Health.Damage() 扣血(0~1归一化)
       │         生成 UDamageEffect、广播 OnPlayerTookDamage、加无敌帧
       │         InfiniteHealth 无障碍 / SecondChance 二次机会会拦截致死
       └─ 是 ─► CrumbKillPlayer()    → bIsDead=true，生成 UDeathEffect
                  阻塞 TagsBlockedDuringDeath/UntilRespawn，广播 OnDeathTriggered
                                       │
                                       ▼
              ┌─────────────── 死亡阶段 ───────────────┐
              │  UPlayerDeathCapability                  │
              │   OnActivated → StartDying() 触发死亡特效 │
              │   OnDeactivated → FinishDying()           │
              └──────────────────────────────────────────┘
                                       │ bHasFinishedDying
                                       ▼
              ┌────────────── 重生阶段 ──────────────────┐
              │  UPlayerRespawnCapability （淡入淡出状态机）│
              │   FadingOutBeforeRespawn → BlackScreen →   │
              │   FadingInAfterRespawn → None              │
              │   黑屏时 CrumbTriggerRespawn:               │
              │     UPlayerRespawnComponent.PrepareRespawn- │
              │       Location() 选最近/Override 重生点      │
              │     HealthComp.Revive() 复活、重置血量       │
              └─────────────────────────────────────────────┘

  双双死亡且 bGameOverWhenBothPlayersDead：
     UPlayerGameOverCapability（host/Mio 控制）
       → 黑白滤镜 + 淡出 → Save::RestartFromLatestSave()
                       （或 CustomGameOverOverride 自定义）
```

## 5 大协作机制（全功能域统一约定）

1. **直接调用 `Type::Get`**：如 `UPlayerHealthComponent::Get(Player)`、`UPlayerRespawnComponent::Get(Player)`，全游戏 77+ 处消费。
2. **Crumb 网络同步**：致命/扣血/重生等改变状态的操作在 `HasControl()` 侧执行 `CrumbXxx` 函数（`UFUNCTION(CrumbFunction)`），自动同步到远端。
3. **标签阻塞**：死亡时 `Player.BlockCapabilities(Tag, this)`，复活时解阻；`CanDie()` 检查 `IsCapabilityTagBlocked(n"Death")`，`ShouldActivate` 检查 `n"Respawn"` 等。
4. **委托 event / Broadcast**：`OnPlayerTookDamage`、`OnDeathTriggered`、`OnPlayerRespawned`、`OnJoined/OnLeft` 等供外部订阅。
5. **可组合设置 ApplySettings**：`UPlayerHealthSettings`、`UDeathRespawnEffectSettings`、`UPlayerHighlightSettings` 等继承 `UHazeComposableSettings`，通过 `Player.ApplySettings(...)` / `GetSettings(Player)` 叠加生效。

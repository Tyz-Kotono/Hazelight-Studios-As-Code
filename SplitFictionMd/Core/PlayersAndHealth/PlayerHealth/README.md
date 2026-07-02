# Core / PlayersAndHealth / PlayerHealth

> 职责：实现玩家生命值、伤害、死亡、重生、Game Over 的完整流程，以及各类致死体积、伤害/死亡特效与 UI，是全游戏受击玩法的统一入口，共 38 文件。

## 内部架构

### 核心组件与数据结构
- **`UPlayerHealthComponent`（`PlayerHealthComponent.as`）**：生命系统中枢，挂在 `APlayerCharacter` 上。
  - 持有 `FHealthValue Health`、状态位 `bIsDead`/`bIsRespawning`/`bIsGameOver`/`bHasStartedDying`/`bHasFinishedDying`、`EGodMode`、无敌列表 `DamageInvulnerabilities`、活动伤害特效 `DamageEffects`。
  - 关键入口：`DamagePlayer()`（判 `CanTakeDamage`→`WouldDieFromDamage`→分流扣血或致死）、`KillPlayer()`、`HealPlayer()`、`Revive()`、`TriggerRespawn()`、`TriggerGameOver()`、批量伤害 `DealBatchedDamage()`（每 0.15s 结算）。
  - 状态修改一律走 `UFUNCTION(CrumbFunction)`：`CrumbKillPlayer`/`CrumbDamagePlayer`/`CrumbHealPlayer`/`CrumbTriggerRespawn`/`CrumbGameOver`。
  - 委托：`OnPlayerTookDamage`、`OnDeathTriggered`、`OnReviveTriggered`、`OnStartDying`/`OnFinishDying`、`CustomGameOverOverride`。
- **`FHealthValue`（`HealthValue.as`）**：归一化 0~1 血量结构，分离「当前血量 / 最近损失 / 最近治疗 / 再生」用于 UI 平滑显示；`Damage`/`Heal`/`Regenerate`/`Reset`/`Update`。
- **`UPlayerRespawnComponent`（`PlayerRespawnComponent.as`）**：管理粘性重生点（`ARespawnPointVolume`/`ARespawnPoint`）、重生覆盖（位置 `TInstigated<FRespawnLocation>` 与委托 `FOnRespawnOverride`）；`PrepareRespawnLocation()` 按优先级+最近距离选点，无点时按设置阻塞或原地兜底。

### 能力（Capabilities，TickGroupOrder 串联流程）
- **`UPlayerDeathCapability`**：`bIsDead && !bHasFinishedDying` 时激活，触发死亡特效、`StartDying()`/`FinishDying()`。无 `Death` 标签（只阻止不打断死亡）。
- **`UPlayerRespawnCapability`**：淡入淡出状态机 `ERespawnState`（FadingOut→BlackScreen→FadingIn→None），黑屏时 `CrumbTriggerRespawn` 调 `PrepareRespawnLocation` + `Revive` + 传送。
- **`UPlayerRespawnMashCapability`**：连打/长按/自动加速重生计时器，同步 `UHazeCrumbSyncedVectorComponent`。
- **`UPlayerGameOverCapability`**：双双死亡且 `bGameOverWhenBothPlayersDead` 时由 host/Mio 触发，黑白滤镜+淡出后 `Save::RestartFromLatestSave()` 或自定义。
- **`UPlayerHealthRegenerationCapability`**：脱战 `RegenerationDelay` 后 `Health.Regenerate()`。
- **`UPlayerHealthDisplayCapability`**、**`UPlayerDamageScreenEffectsCapability`**、**`UPlayerGroundImpactDeathCapability`**：血条显示、受击屏幕特效、落地砸死。

### 特效/事件处理器（`UHazeEffectEventHandler` 派生）
- **`UDeathEffect`**（`DeathEffect.as`）：死亡特效基类，含 `EDeathEffectType`、阻塞标签、相机抖动/力反馈、`Died/FinishedDying/RespawnStarted/RespawnTriggered` 事件；`UDeathAnimatedEffect` 为动画派生。
- **`UDamageEffect`**（`DamageEffect.as`）：受击特效，`EDamageEffectType`、按玩家覆盖 Niagara、`Activate/Deactivate`、最大持续时长。
- **`UPlayerDamageEffectHandler`**（`PlayerDamageEffectHandler.as`）：玩家级伤害事件分发器，`DamageTaken/Healed/HealthUpdated/PlayerDied/PlayerRevived/HealthRegenerated/AvoidedDamageByInvulnerable` 一系列 `Trigger_` 事件；`URespawnEffectHandler`、`UPlayerDamageEventHandler` 类似。

### 致死/伤害体积与外部入口
- **`UDealPlayerDamageComponent` + mixin `DealTypedDamage`**（`DealPlayerDamageComponent.as`）：给伤害源挂的组件，按 `EDamageEffectType/EDeathEffectType` 映射特效后调 `HealthComp.DamagePlayer`。
- **`ADeathVolume`/`AInverseDeathVolume`/`APlayerKillOnAnyImpactVolume`/`ASquishTriggerBox`** + `UDeathVolumeComponent`/`UDeathTriggerComponent`/`UInverseDeathVolumeManager`：进入/离开即 `KillPlayer`。
- **`PlayerHealthStatics.as`**：对外 mixin/全局函数门面——`DamagePlayerHealth`、`HealPlayerHealth`、`KillPlayer`、`IsPlayerDead/Respawning`、`AddDamageInvulnerability`、`PlayerHealth::RespawnPlayerSkipTimer/ForceRespawnPlayerInstantly/KillPlayersInRadius/TriggerGameOver` 等。
- **设置**：`UPlayerHealthSettings`、`UDeathRespawnEffectSettings`（均 `UHazeComposableSettings`）。
- **无障碍/调试**：`CombatModifier.as`（`ECombatModifier::InfiniteHealth`，CVar 控制）、`PlayerHealthDevToggles.as`（god/jesus 模式）、`ChallengeModeStatics.as`（Boss Rush）、`DamageFlashStatics.as`。
- **UI**：`HealthWheelWidget`、`PlayerHealthOverlayWidget`、`PlayerRespawnMashOverlayWidget`、`PlayerDamageScreenEffectComponent`。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|------|------|------|------|
| 调用← | `UPlayerHealthComponent::Get` | `Type::Get` | 全游戏 77+ 处消费（AI 攻击、Boss、关卡机关、音频等） |
| 调用← | mixin `DamagePlayerHealth(Player, ...)` | mixin 便捷调用 | 45+ 处伤害源（`LevelSpecific` 各关卡 boss/弹幕/陷阱）实证 |
| 网络 | `CrumbKillPlayer`/`CrumbDamagePlayer`/`CrumbTriggerRespawn` 等 | Crumb（HasControl+CrumbFunction） | 状态变更仅控制侧执行并同步远端 |
| 阻塞 | `n"Death"`/`n"Respawn"`/`n"Movement"`/`n"GameplayAction"` 等 | 标签阻塞 | `CanDie`/`ShouldActivate` 检查；死亡时 Block、复活解阻 |
| 委托→ | `OnPlayerTookDamage`/`OnDeathTriggered`/`OnPlayerRespawned` | event/Broadcast | 音频（`PlayerFilterAudioOnDeath`）、Vox（`PlayerPauseVoxOnDeath`）、AI 等订阅 |
| 设置 | `UPlayerHealthSettings`/`UDeathRespawnEffectSettings` | ApplySettings/GetSettings | `BeginPlay` 中 `ApplySettings(DeathRespawnSettings, ...)` |
| 调用→ | `ARespawnPoint`/`ARespawnPointVolume`（RespawnPoint 模块） | 直接调用 | `PrepareRespawnLocation` 遍历 `TListedActors<ARespawnPoint>` 选点 |
| 调用→ | `Save::RestartFromLatestSave` | 直接调用 | `UPlayerGameOverCapability.FinishGameOver` |
| 配色 | `PlayerColor::*`（Players 模块） | 命名空间常量 | 重生/受击闪烁色、调试绘制 |

## 关键文件

- `PlayerHealthComponent.as`：生命中枢，伤害/死亡/治疗/重生/GameOver 全部 Crumb 入口。
- `HealthValue.as`：归一化 0~1 血量结构，分离显示用最近损失/治疗/再生。
- `PlayerRespawnComponent.as`：粘性重生点、重生覆盖、最近点选取。
- `PlayerRespawnCapability.as`：重生淡入淡出状态机，黑屏时触发复活+传送。
- `PlayerDeathCapability.as`：死亡阶段驱动，触发死亡特效与 Start/FinishDying。
- `PlayerRespawnMashCapability.as`：连打/长按/自动加速重生计时。
- `PlayerGameOverCapability.as`：双双死亡 Game Over，黑白+淡出后读档重启。
- `PlayerHealthRegenerationCapability.as`：脱战自动回血再生。
- `PlayerHealthDisplayCapability.as` / `PlayerDamageScreenEffectsCapability.as`：血条与受击屏幕特效。
- `PlayerGroundImpactDeathCapability.as`：落地砸死（含组件）。
- `DealPlayerDamageComponent.as`：伤害源组件 + `DealTypedDamage` mixin，按类型映射特效。
- `PlayerHealthStatics.as`：对外伤害/治疗/重生/GameOver 全局门面 mixin。
- `PlayerRespawnStatics.as`：粘性重生点/重生覆盖的全局 mixin。
- `DeathEffect.as` / `DeathAnimatedEffect.as`：死亡特效基类与动画派生（`EDeathEffectType`）。
- `DamageEffect.as`：受击特效（`EDamageEffectType`）。
- `PlayerDamageEffectHandler.as` / `RespawnEffectHandler.as` / `PlayerDamageEventHandler.as`：伤害/重生事件分发器。
- `PlayerHealthSettings.as` / `DeathRespawnEffectSettings.as`：可组合健康与死亡重生特效设置。
- `DeathVolume.as` / `InverseDeathVolume.as` / `InverseDeathVolumeManager.as` / `PlayerKillOnAnyImpactVolume.as` / `SquishTriggerBox.as` / `DeathVolumeComponent.as` / `DeathTriggerComponent.as`：各类致死体积/触发器。
- `CombatModifier.as`：无障碍无限血修饰（CVar）。
- `PlayerHealthDevToggles.as`：god/jesus 模式开关。
- `ChallengeModeStatics.as`：Boss Rush 挑战模式。
- `DamageFlashStatics.as` / `DeathStaticCamera.as`：受击闪白、死亡静态相机。
- `HealthWheelWidget.as` / `PlayerHealthOverlayWidget.as` / `PlayerRespawnMashOverlayWidget.as` / `PlayerDamageScreenEffectComponent.as`：血量与重生/受击 UI。

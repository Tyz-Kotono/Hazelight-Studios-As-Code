# Gameplay / Combat 功能域总览

> 职责：Split Fiction 中**伤害的产生与承受**——集中式伤害入口、受击帧冻结、玩家三档受伤反应（加法击中/踉跄/击倒）、以及情境战斗目标点。共 **19 文件**。
>
> **前提**：本文承接 [Gameplay 设计哲学](../DesignPhilosophy.md)（能力四件套、标签互斥矩阵）与 Core。这里不重复哲学，只讲 Combat 自身的架构、数据流与协同边界。

---

## 一句话定位

> **Combat 不是一个"战斗系统"，而是一组正交的伤害基础设施。** 伤害的施加统一走 `Damage::` 命名空间；伤害的表现（玩家怎么被打得踉跄/倒下、命中瞬间的顿帧）各自是独立能力，靠标签与网络回放接入玩家。真正的敌人血量/攻击逻辑在 `Gameplay/AI`，玩家血量在 `Gameplay/PlayerHealth`。

---

## 目录结构

| 位置 | 文件数 | 内容 |
| --- | --- | --- |
| 根 | 3 | `DamageStatics.as`（伤害入口）、`DamageTypes.as`（伤害类型枚举）、`DamageSettings.as`（AI 伤害可组合设置） |
| `ContextualCombat/ActivationPoints` | 3 | 情境战斗目标点：可被瞄准的战斗交互点与其瞄准能力 |
| `HitStop` | 2 | 命中顿帧组件 + 触发它的 AnimNotify |
| `PlayerHurtReactions` | 9 | 玩家三档受伤反应，各自四件套的 Capability + Component + Statics |
| `Tags` | 2 | 战斗标签词典（GloryKill 相关） |

---

## 内部架构

### 1. 伤害入口：`Damage::` 命名空间

`Combat/DamageStatics.as` 是纯自由函数命名空间（无类），是全项目**施加伤害的唯一收口**：

| 函数 | 作用 |
| --- | --- |
| `PlayerRadialDamage` | 对 `Game::Players` 范围内玩家扣血（走 `Player.DamagePlayerHealth`） |
| `AIRadialDamageToTeam` | 对某 `UHazeTeam` 成员施加范围伤害 |
| `AITakeDamage` | 单体 AI 伤害，路由到 `UBasicAIHealthComponent.TakeDamage` |
| `GetRadialDamageFactor` | 距离衰减系数（内半径 + 指数） |

`Combat/DamageTypes.as` 定义 `enum EDamageType`（`MeleeBlunt/MeleeSharp/Projectile/Acid/Energy/Impact/Fire/Explosion/…`），是贯穿 AI、投射物、弱点、击杀体的伤害类型词汇表。

### 2. 命中顿帧：`UCombatHitStopComponent`

`Combat/HitStop/CombatHitStopComponent.as` 中 `UCombatHitStopComponent : UActorComponent`。原理：把 owner 及其挂接 Actor 的 `ActorTimeDilation` 设为近零（`0.00001`）持续一小段时间，制造"命中定格"的打击感。支持按 instigator 分键的叠加、禁用/恢复、自动续期。触发唯一来自动画：`AnimNotify_CombatStaticHitStop.as`（`UAnimNotify_CombatStaticHitStop : UAnimNotify`，受 CVar `Haze.Skyline.UseCombatTweaks` 控制）。

### 3. 玩家受伤三档反应（`PlayerHurtReactions/`）

三档从轻到重，每档都是标准四件套（Capability 行为 + Component 状态 + Statics 入口），且 **`NetworkMode = Crumb`** 保证双端一致：

| 档 | 关键类 | 基类 | 表现 |
| --- | --- | --- | --- |
| 加法击中 | `UPlayerAdditiveHitReactionCapability` | `UHazePlayerCapability` | 播放加法层 flinch 动画（`RequestAdditiveFeature n"HitReaction_Addative"`），**不打断移动**，带冷却 |
| 踉跄 | `UPlayerStumbleCapability` | `UHazeCapability` | 有方向的趔趄：地面走动画移动比、空中走弹道速度；期间 block `GameplayAction` |
| 击倒 | `UPlayerKnockdownCapability` | `UHazeCapability` | 全身物理击倒：接管移动/旋转、换碰撞档 `PlayerCharacterKnockedDown`、给伤害无敌、播 `ULocomotionFeatureKnockDown`，处理起身与空中恢复 |

对应 Component（`...Component.as`）以 `access`（AngelScript 访问限定）把状态写入权仅开放给自家 Capability；Statics（`...Statics.as`）导出 `mixin ApplyStumble/ApplyKnockdown` 与 `BP_*` 蓝图入口，并对短时间内重复请求去重。

### 4. 情境战斗目标点（`ContextualCombat/`）

`UContextualCombatTargetableComponent : UContextualMovesTargetableComponent`（空标记子类，把 Actor 标记为可锁定的战斗目标）；`UContextualCombatTargetingCapability : UHazePlayerCapability` 通过 `UPlayerTargetablesComponent` 为附近目标显示锁定 UI。复用 Movement 的情境目标点框架。

---

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 被调用 ← | `Gameplay/AI/Debug/AIDevMenu.as` | `Damage::AITakeDamage(Actor, Dmg, Game::Mio, EDamageType::Default)` | 调试菜单是 `Damage::` 命名空间在 Gameplay 里的直接调用者 |
| 被调用 ← | `Gameplay/AI/Volumes/AIKillTrigger.as` 等 | `EDamageType` 词汇 + `UBasicAIHealthComponent.TakeDamage` | 击杀体、弱点、投射物共享 `EDamageType` 枚举 |
| 被调用 ← | `Gameplay/PlayerHealth/DamageTriggerComponent.as` | `Player.ApplyKnockdown(...)` / `Player.ApplyStumble(...)` | 玩家受伤触发器按配置调出击倒/踉跄反应 |
| 被调用 ← | `Gameplay/Movement/Player/Component/PlayerKnockBackComponent.as` | `Player.ApplyKnockdown(...)` | 击退组件复用击倒反应 |
| 依赖 → | Core 血量系统 | `Player.DamagePlayerHealth`、`UPlayerHealthComponent`、`UBasicAIHealthComponent` | 伤害最终落到血量组件，本模块不存血量 |
| 依赖 → | Core Team | `HazeTeam::GetTeam` → `UHazeTeam` | 范围伤害按队伍圈定目标 |
| 依赖 → | Core 能力/网络 | `BlockCapabilities(CapabilityTags::GameplayAction)`、`Movement.ApplyCrumbSyncedGroundMovement()` | 受伤反应期间屏蔽玩法动作，Crumb 回放 |
| 依赖 → | Core 动画 | `RequestAdditiveFeature`、`ULocomotionFeatureKnockDown` | 加法层与击倒 locomotion 特征 |

> 注：Combat 内部**未**引用 `UInteractionComponent`、`UHazeSplineComponent`，也没有通用 `ApplyDamage`——伤害一律经血量组件 API。

---

## 标签词典（`Tags/`）

两个单常量命名空间，都围绕"处决（GloryKill）"：

- `PlayerCombatTags.as`：`PlayerCombatTags::GloryKill = n"GloryKill"`（状态标签）
- `CombatBlockedWhileInTags.as`：`CombatBlockedWhileIn::GloryKill = n"BlockedWhileInGloryKill"`（处决时屏蔽其他能力的 block 标签）

受伤反应自身用到的标签（`n"HitReaction"`、`n"AdditiveHitReaction"`、`n"Knockdown"`、`n"Stumble"`）就近声明在各 Capability 文件内，未收进本目录。

---

## 关键文件

- `Gameplay/Combat/DamageStatics.as` —— 全项目伤害施加的唯一入口（`Damage::` 命名空间）。
- `Gameplay/Combat/DamageTypes.as` —— `EDamageType` 伤害类型枚举，跨 AI/投射物/弱点通用。
- `Gameplay/Combat/HitStop/CombatHitStopComponent.as` —— 命中顿帧（时间膨胀近零）。
- `Gameplay/Combat/PlayerHurtReactions/Knockdown/PlayerKnockdownCapability.as` —— 全身物理击倒（含无敌/起身/Crumb）。
- `Gameplay/Combat/PlayerHurtReactions/Stumble/PlayerStumbleCapability.as` —— 方向性踉跄。
- `Gameplay/Combat/PlayerHurtReactions/AdditiveHurt/PlayerAdditiveHitReactionCapability.as` —— 不打断移动的加法层 flinch。

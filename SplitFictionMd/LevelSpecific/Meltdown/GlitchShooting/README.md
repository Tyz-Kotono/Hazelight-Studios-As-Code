# LevelSpecific / Meltdown / GlitchShooting（故障射击）

> 路径：`GlitchShooting/`(23) + `GlitchBazooka/`(6) + `GlitchBeam/`(4) + `GlitchSword/`(6)。合计 39 文件。

## 职责

Meltdown 的专属武器玩法。世界"故障熔毁"，玩家获得一把能把奇幻/科幻物件"格式化（glitch）"的故障武器——射击可清除/改变故障物件、推进谜题，也是**打三阶段 Boss 弱点的唯一输出手段**。武器有四种形态：主力**故障枪**（GlitchShooting）、**故障火箭筒**（GlitchBazooka）、**故障光束**（GlitchBeam）、**故障剑**（GlitchSword）。四者同构：`AHazeActor` 武器实体 + `UHazePlayerCapability` 能力链 + `UActorComponent` 状态组件（+ `UHazeComposableSettings` 参数 + `UHazeEffectEventHandler` 表现）。

## 内部架构

### 故障枪 GlitchShooting（23）——核心形态

- **武器实体** `AMeltdownGlitchShootingWeapon : AHazeActor`。
- **能力链**（均 `UHazePlayerCapability`，`NetworkMode=Crumb`）：`UMeltdownGlitchShootingWeaponCapability`（持枪/装备）、`...AimCapability`（瞄准，走 `UPlayerAimingComponent`）、`...FireCapability`（开火/蓄力节奏，检 `ActionNames::WeaponFire`、地面/冲刺/冲量门控）、`...StrafeCapability`（横移）、`...PostCutsceneCapability`（过场后恢复）。
- **状态组件**（`UActorComponent`）：`UMeltdownGlitchShootingUserComponent`（持有 `ProjectileClass`、指示器、`bGlitchShootingActive`）、`...ResponseComponent`（命中响应，挂在被打的物件/Boss 上）、`UMeltdownGlitchinessResponseComponent`、`UMeltdownGlitchShootingAccumulationComponent`（故障累积）。
- **参数** `UMeltdownGlitchShootingSettings : UHazeComposableSettings`（`ChargeCooldown` 等）。
- **投射物与表现**：`AMeltdownGlitchShootingProjectile` + `...ProjectileEffectHandler`；子弹用 `UHazeActorLocalSpawnPoolComponent` 对象池生成。
- **拾取/升级**：`AMeltdownGlitchShootingPickup : ADoubleInteractionActor`（双人交互拾取）、`AMeltdownGlitchShootingPowerup`（+EffectHandler）。
- **UI/反馈**：`UMeltdownGlitchShootingCrosshair : UCrosshairWidget`、`AMeltdownPlayerGlitchIndicator`、`AMeltdownGlitchShootingOrbEffect`。
- 本目录还托管共享 Boss：`AMeltdownBoss` / `AMeltdownBossWeakPoint` / `UMeltdownBossHealthComponent`（见 [Boss](../Boss/)）。

### 三种衍生形态（同构四件套）

| 形态 | 目录 | 实体 | 能力（`UHazePlayerCapability`） | 状态组件 |
|---|---|---|---|---|
| 火箭筒 | `GlitchBazooka/`(6) | `AMeltdownGlitchBazooka` | Equip / Fire / PostCutscene | `UMeltdownGlitchBazookaUserComponent` |
| 光束 | `GlitchBeam/`(4) | `AMeltdownGlitchBeam` | Aim / Fire | `UMeltdownGlitchBeamUserComponent` |
| 剑 | `GlitchSword/`(6) | `AMeltdownGlitchSword` | Attack / Equip / PostCutscene | `UMeltdownGlitchSwordUserComponent` |

火箭筒/剑各带投射物 Actor（`...Projectile`）；三者与故障枪走同一套 `NetworkMode=Crumb` + 命中响应约定。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| → | Core 瞄准 | 直接调用 | `...FireCapability` / `...AimCapability` 依赖 `UPlayerAimingComponent`（Setup 里 `::Get(Player)`） |
| → | Core 移动 | 门控 | `FireCapability` 读 `UPlayerMovementComponent`（地面判定、`HasImpulse`），冲刺时禁开火 |
| → | Core 对象池 | 子弹生成 | `HazeActorLocalSpawnPoolStatics::GetOrCreateSpawnPool(ProjectileClass, Player)` |
| → | 三阶段 Boss | 输出手段 | 命中触发 Boss 的 `UMeltdownGlitchShootingResponseComponent.OnGlitchHit`（Boss 唯一受伤途径） |
| → | Core 双人交互 | 拾取 | `AMeltdownGlitchShootingPickup : ADoubleInteractionActor` |
| → | Core Settings | 参数分层 | `UMeltdownGlitchShootingSettings : UHazeComposableSettings`（`GetSettings(Player)`） |
| ↔ | Core Crumb 联机 | 状态同步 | 所有开火/装备能力 `NetworkMode=Crumb` |

## 关键文件

- `GlitchShooting/MeltdownGlitchShootingWeapon.as` — 武器实体 `AMeltdownGlitchShootingWeapon`
- `GlitchShooting/MeltdownGlitchShootingFireCapability.as` — 开火/蓄力节奏 + 移动门控（核心逻辑）
- `GlitchShooting/MeltdownGlitchShootingUserComponent.as` — 玩家状态中枢组件
- `GlitchShooting/MeltdownGlitchShootingResponseComponent.as` — 命中响应（挂在物件/Boss 上）
- `GlitchShooting/MeltdownGlitchShootingSettings.as` — 参数 `UHazeComposableSettings`
- `GlitchShooting/MeltdownGlitchShootingPickup.as` — 双人交互拾取
- `GlitchBazooka/` `GlitchBeam/` `GlitchSword/` — 三种衍生形态（同构四件套）

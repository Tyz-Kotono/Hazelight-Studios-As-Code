# LevelSpecific / Skyline / 重力武器（GravityWeapons）

> 路径：`LevelSpecific/Skyline/GravityBlade/`（87，重力刀）+ `LevelSpecific/Skyline/GravityWhip/`（38，重力鞭）
>
> Skyline 的两件人形近战/操控武器。重力刀（GravityBlade）= 近战连招 + 重力抓钩 + 处决；重力鞭（GravityWhip）= 抓取/投掷物体 + 抓点位移 + 处决。两者共享"重力操控 + 自动瞄准 + GloryKill 处决"范式，故合并成篇。

承接 [Skyline 关卡总览](../README.md)。能力四件套/标签范式见 [Gameplay 设计哲学](../../../Gameplay/DesignPhilosophy.md)，不重复。

---

## 职责

两把武器都以能力（`UHazePlayerCapability`）形式挂玩家 `UHazeCapabilityComponent`，武器本体是 actor（`AGravityBladeActor` / `AGravityWhipActor`），"用户组件"（UserComponent）作玩家侧状态容器。抓取/抓钩基于 Core 目标系统（`UAutoAimTargetComponent` / `UTargetableComponent`），处决走共用的 GloryKill 语义。

---

## 一、重力刀 GravityBlade（87）

近战连招剑，附带把玩家拉向目标的重力抓钩。

- **本体/状态**：`GravityBladeActor`、`GravityBladeUserComponent`（`UActorComponent`）、`GravityBladeSettings`（命名空间）、`GravityBladeDebugComponent`。
- **持握/表现能力（`Capabilities/`）**：`SheatheCapability`（收/出鞘）、`VisibilityCapability`（可见性）、`CameraCapabilty`（相机，含 `UHazeCameraSpringArmSettingsDataAsset`）。
- **战斗（`Combat/`）**：
  - 攻击（`Combat/Capabilities/Attacks/`）：`GroundAttack` / `AirAttack` / `AttackTraceHit`（命中判定）。
  - 冲刺（`Combat/Capabilities/Rush/`）：`RushCapability` + `RushCameraCapability` + `RushStatics`。
  - 处决（`Combat/Capabilities/GloryKill/`，含 `Enforcer/`）+ `Combat/GloryKill/` 数据、`Combat/OpportunityAttack/`（趁机攻击）、`Combat/Notifies/`（AnimNotify）、`Combat/Widgets/`。
- **抓钩（`Grapple/`）**：`GravityBladeGrappleComponent : UAutoAimTargetComponent`（可被抓的目标）、`GrappleUserComponent`、`GrappleSettings`、`GrappleResponseComponent`（响应事件 `FGravityBladeThrowSignature` / `FGravityBladePullSignature`）、`GrappleData` / `GrappleAnimationData`、准星 `GrappleCrosshairWidget` / `GrappleTargetWidget`；能力在 `Grapple/Capabilities/`（含 `Eject/` 弹出）。
- **重力位移组件**：`GravityBladeGravityShiftComponent`（重力方向切换，被 DroneBoss 等复用）。
- `Tutorial/`、`EventHandlers/`。

## 二、重力鞭 GravityWhip（38）

抓取远处物体/敌人并投掷或拉近，支持多种抓取模式。

- **本体/状态**：`GravityWhipActor`、`GravityWhipUserComponent`（`UActorComponent`）、`GravityWhipResponseComponent`（挂被抓物）、`GravityWhipImpactResponseComponent`（命中响应）。
- **目标系统**：`GravityWhipTargetComponent : UTargetableComponent`（可抓目标）、`GravityWhipSlingAutoAimComponent : UAutoAimTargetComponent`（投掷自动瞄准）、准星 `GravityWhipCrosshairWidget : UCrosshairWithAutoAimWidget`。
- **能力（`Capabilities/`）**：`AimCapability`（瞄准）、`TargetCapability`（选目标）、`GrabCapability` / `GrabAnimationCapability` / `GrabPointCapability`（抓取，`Capabilities/GrabModes/` 存多种抓取模式枚举 `EGravityWhipGrabMode`）、`AirGrabCapability`（空中抓）、`SlingAimCapability` / `SlingObjectMovementCapability`（投掷瞄准/被投物移动）、`StrafeCapability`、`GloryKillCapability`（处决，含 AnimNotify `DamageNotify` / `ComboWindowNotify` / `LockedMovement`）、`VisibilityCapability`。
- `Responses/`、`EventHandlers/`、`Tutorial/`（含 `SecondTutorial/`）。

---

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 依赖 | `AHazeActor` | Core | `AGravityBladeActor`、`AGravityWhipActor` 武器本体 |
| 依赖 | `UHazePlayerCapability` / `UHazeCapabilityComponent` | Gameplay | 所有攻击/抓取/处决能力挂玩家容器（能力四件套） |
| 依赖 | `UAutoAimTargetComponent` / `UTargetableComponent` | Core 目标系统 | 抓钩/抓取目标选取与自动瞄准 |
| 共用 | GloryKill 处决语义 | Skyline | 重力刀 `Combat/GloryKill/` 与重力鞭 `GloryKillCapability` 共享处决范式 |
| 被复用 | `UGravityBladeGravityShiftComponent` / `GrappleComponent` | Skyline Boss(DroneBoss) | DroneBoss 挂重力刀的重力场 + 抓钩组件供玩家操控 |
| 依赖 | `UHazeComposableSettings` | Gameplay | `GravityBladeGrappleSettings` 等设定集 |
| 依赖 | `UAnimNotify` / `UAnimNotifyState` | Core Anim | 攻击/处决判定绑动画时间轴 |
| 特效 | `UHazeEffectEventHandler` | Core | 两武器 `EventHandlers/` 处理命中表现 |

---

## 关键文件

- 重力刀本体：`GravityBlade/GravityBladeActor.as`、`GravityBlade/GravityBladeUserComponent.as`
- 重力刀战斗：`GravityBlade/Combat/Capabilities/Attacks/GravityBladeCombatGroundAttackCapability.as`、`GravityBlade/Combat/Capabilities/Rush/GravityBladeCombatRushCapability.as`、`GravityBlade/Combat/Capabilities/GloryKill/`
- 重力刀抓钩：`GravityBlade/Grapple/GravityBladeGrappleComponent.as`、`GravityBlade/Grapple/Capabilities/`
- 重力鞭本体：`GravityWhip/GravityWhipActor.as`、`GravityWhip/GravityWhipUserComponent.as`、`GravityWhip/GravityWhipTargetComponent.as`
- 重力鞭能力：`GravityWhip/Capabilities/GravityWhipGrabCapability.as`、`GravityWhip/Capabilities/GravityWhipSlingObjectMovementCapability.as`、`GravityWhip/Capabilities/GravityWhipGloryKillCapability.as`

# LevelSpecific / Summit / 龙剑（DragonSword）

> 路径：`LevelSpecific/Summit/DragonSword/`（53 文件，含蓄力回旋镖 3 文件）
>
> 玩家落地后的近战连招武器：地面/空中/冲刺/冲锋多形态连击，配合蓄力回旋镖（Boomerang），用于石兽终局的破甲与弱点攻击。

承接 [Summit 总览](../README.md)。能力四件套/标签范式见 [Gameplay 设计哲学](../../../Gameplay/DesignPhilosophy.md)，不重复。

---

## 职责

龙剑是一把挂在玩家身上的武器 actor，攻击逻辑全部以能力（`UHazePlayerCapability`）形式挂在玩家 `UHazeCapabilityComponent` 上。它是 Summit 唯一的纯人形近战武器，主要服务石兽（StoneBeast）终局的破甲/弱点战。

---

## 内部架构

### 武器本体与挂载

- **`ADragonSword : AHazeActor`**（Abstract）— 极简武器 actor：`UHazeOffsetComponent` + `UStaticMeshComponent SwordMesh`（无碰撞，纯视觉）+ `UHazeMovementAudioComponent`，持 `AHazePlayerCharacter Player` 引用。
- **`UDragonSwordUserComponent : UActorComponent`** — 玩家侧武器状态容器。
- **持握/表现能力（`Capabilities/`）**：`DragonSwordWieldingCapability`（`UHazePlayerCapability`，装备/持握）、`DragonSwordOnBackCapability`（收剑背上）、`DragonSwordMarkerCapability`（目标标记）、`DragonSwordOutlineCapability`（轮廓）、`DragonSwordCameraCapabilty`（相机）。

### 战斗系统（`Combat/`，主体）

一套完整的连击框架，数据与能力分离：

- **数据层**：`DragonSwordCombatSettings`、`DragonSwordCombatAttackData` / `AttackType` / `AttackAnimationData` / `ComboData` / `AnimData` — 攻击类型、连击表、动画数据。
- **用户/输入/响应组件**：`DragonSwordCombatUserComponent`、`DragonSwordCombatInputComponent`、`DragonSwordCombatTargetComponent`（目标选取）、`DragonSwordCombatResponseComponent`（受击响应，挂在被攻击物上）。
- **控制能力（`Combat/Capabilities/`）**：`PrimaryInputCapability`（输入）、`AttackActivationCapability`（触发）、`AttackStateCapability`（攻击状态机）、`ComboCapability`（连击窗口）、`AttackRecoilCapability`（击退回弹）、`AimCapability`（瞄准）、`BlockJumpCapability`、`CombatCameraCapability`。
- **攻击招式（`Combat/Capabilities/Attacks/`）**：`GroundAttack` 地面 / `AirAttack` 空中 / `AirAttackGroundPound` 空中砸地 / `DashAttack` 冲刺 / `SprintAttack` 疾跑攻击 / `GroundChargeAttack` 地面蓄力。
- **冲锋位移（`Combat/Capabilities/Rush/`）**：`GroundRush` / `GroundDashRush` / `AirRush` / `AirDashRush` + `RushStatics`——攻击附带的突进位移。
- **AnimNotify（`Combat/Notifies/`）**：`ComboWindow`（连击窗口）、`HitWindow`（命中判定窗口）、`SettleWindow`（收招窗口）——把攻击判定绑到动画时间轴。
- **事件/UI**：`EventHandlers/`（`CombatEventHandler`、`PlayerEventHandler`）、`Widgets/`（Mio/Zoe 可攻击目标提示）、`CrosshairWidget`。

### 蓄力回旋镖（Boomerang，3 文件）

蓄力攻击的远程变体，位于 `Combat/Capabilities/Attacks/`：

- `GroundChargeAttackBoomerangCapability`（投掷）、`GroundChargeAttackBoomerangRecallCapability`（召回），配合基础的 `GroundChargeAttackCapability`（蓄力）。命名空间 `DragonSwordBoomerang` 存投掷距离/距离修正等参数（`MinThrowDistance`、`DistanceModifierUpdateInterval`），投出的实例走 `MakeNetworked` 联机。

### 钉地（`PinToGround/`）

`DragonSwordPinToGround*` 系列（`Capability` / `ActiveCapability` / `ExitCapability` + `Component` + `PinnableGroundComponent` + `EffectHandler`）——把剑/目标钉在地面的专用交互，`UDragonSwordPinnableGroundComponent` 标记可被钉的地面。

---

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 依赖 | `AHazeActor` | Core | `ADragonSword : AHazeActor`（Abstract），纯视觉武器 actor |
| 依赖 | `UHazePlayerCapability` / `UHazeCapabilityComponent` | Gameplay | 所有攻击/持握逻辑挂玩家能力容器（能力四件套） |
| 依赖 | `UHazeComposableSettings` | Gameplay | `DragonSwordCombatSettings` 等设定集 |
| 依赖 | `UAnimNotify` | Core Anim | Combo/Hit/Settle 窗口把判定绑到动画时间轴 |
| 联机 | `MakeNetworked` | Core Network | 回旋镖投掷实例生成时联网 |
| 服务 | 石兽破甲/弱点 | Summit StoneBoss | 龙剑连招 + 回旋镖用于石兽终局破 `StoneBreakable*`、弱点 QTE |
| 双人 | Mio / Zoe | Dragons 约定 | `Widgets/` 分 Mio/Zoe 可攻击目标提示，延续双人语义 |
| 特效 | `UHazeEffectEventHandler` | Core | `PinToGround` / Combat EventHandler 处理命中表现 |

---

## 关键文件

- 本体：`DragonSword/DragonSword.as`、`DragonSword/DragonSwordUserComponent.as`、`DragonSword/Capabilities/DragonSwordWieldingCapability.as`
- 战斗核心：`DragonSword/Combat/DragonSwordCombatSettings.as`、`DragonSword/Combat/Capabilities/DragonSwordCombatComboCapability.as`、`DragonSword/Combat/Capabilities/DragonSwordCombatAttackStateCapability.as`
- 招式：`DragonSword/Combat/Capabilities/Attacks/DragonSwordCombatGroundAttackCapability.as`、`DragonSword/Combat/Capabilities/Rush/DragonSwordCombatGroundRushCapability.as`
- 回旋镖：`DragonSword/Combat/Capabilities/Attacks/DragonSwordCombatGroundChargeAttackBoomerangCapability.as`、`...BoomerangRecallCapability.as`、`...GroundChargeAttackCapability.as`
- 钉地：`DragonSword/PinToGround/DragonSwordPinToGroundCapability.as`

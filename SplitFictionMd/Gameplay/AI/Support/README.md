# Gameplay / AI / Support（周边支持合篇）

> 职责：AI 模块中六个小的横切/支持子系统合并成篇——难度平衡、弱点、音频群控、语音、触发盒、调试。它们各自独立、体量小，共 15 文件。

---

## 一、Balancing（难度令牌，2 文件）

按玩家血量动态调整敌人「攻击令牌上限」——玩家越危险，同时能攻击他的敌人越少，实现自适应难度（DDA）。

- `BalancingSettings.as`：`UBalancingSettings : UHazeComposableSettings`。`TokenReductionHealthLimitFactors` 是「血量阈值 → 削减令牌数」的表（默认 `{0.1, 1}`：血量降到 10% 时减 1 个令牌）。
- `BalancingTokenHealthLimitPlayerCapability.as`：挂玩家上的能力。每帧读 `UPlayerHealthComponent.Health.CurrentHealth`，找到匹配的血量阈值档，据此调整玩家身上 `UGentlemanComponent` 的令牌数（代码中调令牌部分标 TODO）。

**协同**：直接操纵 Gentleman 令牌上限 → 影响同时可攻击 AI 数量（见 Gentleman/）。

---

## 二、Weakpoints（弱点，2 文件）

大型敌人/Boss 身上可被针对性破坏的部件。

- `BasicAIWeakpointComponent.as`：`UBasicAIWeakpointComponent : USceneComponent`。挂载点，其下挂可碰撞体和 `UTargetableComponent`（供自动瞄准）。生命周期：`DamageWeakpoint`（受击广播 `OnWeakpointTakeDamage`）、`DestroyWeakpoint`（`bIsDestroyable` 时销毁并广播）、`ReviveWeakpoint`、`AddEnable/AddDisableWeakpoint`（连带启停 Targetable）。事件：`OnWeakpointHit/TakeDamage/Destroyed`。
- `namespace Weakpoints`：`FindWeakpointFromHit` / `FindAndDamageWeakpointFromHit` / `FindAndDestroyWeakpointFromHit`——从命中的图元反查弱点组件并施加伤害/破坏。

**协同**：接 Core 的 `UTargetableComponent`（自动瞄准）；伤害逻辑由使用方（Boss 蓝图）管理，本组件只做「数据 + 事件容器」。

---

## 三、Audio（音频群控与死亡追踪，4 文件）

- `AICharacterAudioCrowdControlComponent.as` + `Manager`：**人群音频群控**。按 `GroupTag` 把同类敌人分组（Manager 挂 Mio 上做全局单例），`GetFullCrowdSize/GetNumInGroup` 让音频系统知道「这群敌人有多大」，据此调整战斗音效强度/密度。AI 基类自带 `CrowdControlComp`。
- `AIEnemyGlobalDeathTrackerManagerComponent.as` + `AIEnemyGlobalDeathTrackerCapability.as`：**全局死亡计数**。按 Tag 统计近期死亡数（`RegisterDeath` 计数，`DeathTimers` 到点递减衰减），`GlobalAIEnemy::GetTrackedAIEnemyDeathsOfTag` 查询——用于音乐/演出对「刚刚死了一波」做反应。
- `AIEnemySpawnerTrackerVolume.as`：追踪体积内生成器状态。

**协同**：单例挂 `Game::GetMio()`；供 Core/Audio 音乐系统查询战斗烈度。

---

## 四、VoiceOver（语音，3 文件）

- `BasicAIVoiceOverComponent.as`：AI 基类自带 `VoiceComp`。`BeginPlay` 向 `UBasicAIVoiceOverManager` 单例按 `Owner.Class` 申领一个唯一 `VoiceOverID`——**同类敌人分配不同语音 ID**，避免多个同款敌人同时喊同一句台词。
- `BasicAIVoiceOverManager.as`：单例，`CreateNewVoiceID` 按类分配递增 ID。
- `BasicAIVoiceOverStatics.as`：便捷函数。

**协同**：接 Core 的 Vox/字幕系统。

---

## 五、Volumes（AI 触发盒，4 文件）

关卡里作用于 AI 的体积/触发器：

| 文件 | 作用 |
|---|---|
| `AITrigger.as` | `AAITrigger`：AI 进出触发（`OnTriggerEnter/Exit`），用 `UTeamLazyTriggerShapeComponent` 检测本队伍成员；有成员靠近时全速 Tick，否则低频 Tick（`FarTickInterval=0.2`）省开销 |
| `AIKillTrigger.as` | 进入即杀死 AI 的体积 |
| `ActorSettingsTrigger.as` | `AActorSettingsTrigger`：AI 进入体积时应用一组 `UHazeComposableSettings`（离开时清除），用于分区改敌人参数 |
| `TeamLazyTriggerShapeComponent.as` | 按队伍的惰性形状触发组件（远时降频） |

**协同**：接 Core 的 Team、Settings、Shape/Trigger 体系。

---

## 六、Debug（调试，2 文件）

- `AIDevMenu.as`：`UAIDevMenu` 开发菜单。可对选中 AI/队伍施加伤害、设不死/高血、暂停行为的各子系统（移动/攻击/朝向/感知）、可视化目标/敌人/队友关系。AI `BeginPlay`（`#if TEST`）请求此菜单。
- `AIDebugDisplayComponent.as`：`UAIDebugDisplayComponent`，`#if TEST` 时挂到 AI 上做屏幕调试显示。

**协同**：接 GUI/Debug 的 DevMenu、TemporalLog。

---

## 汇总协同关系

| 子系统 | 主要对接 | 机制 |
|---|---|---|
| Balancing | Gentleman 令牌 / PlayerHealth | 按血量调令牌上限 |
| Weakpoints | Core Targetable | 部件伤害 + 事件容器 |
| Audio | Core/Audio、Mio 单例 | 群体规模 / 死亡计数供音乐反应 |
| VoiceOver | Core Vox、单例 | 按类分配唯一语音 ID |
| Volumes | Core Team/Settings/Shape | AI 进出体积改参数/杀死 |
| Debug | GUI DevMenu / TemporalLog | 仅 TEST/EDITOR 调试 |

## 关键文件

- `Balancing/BalancingTokenHealthLimitPlayerCapability.as`：按玩家血量削减令牌（自适应难度）。
- `Weakpoints/BasicAIWeakpointComponent.as`：弱点部件（伤害/破坏/复活 + Targetable 联动）。
- `Audio/AIEnemyGlobalDeathTrackerManagerComponent.as`：全局死亡计数（衰减）。
- `Audio/AICharacterAudioCrowdControlComponent.as`：人群音频分组。
- `VoiceOver/BasicAIVoiceOverComponent.as`：唯一语音 ID 分配。
- `Volumes/AITrigger.as` / `ActorSettingsTrigger.as`：AI 触发盒（惰性 Tick / 应用设置）。
- `Debug/AIDevMenu.as`：AI 开发调试菜单。

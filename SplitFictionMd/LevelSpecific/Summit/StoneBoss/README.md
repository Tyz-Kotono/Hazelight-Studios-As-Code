# LevelSpecific / Summit / 石头 Boss（StoneBoss / StoneBeast）

> 路径：`LevelSpecific/Summit/StoneBoss/`（195 文件）
>
> Summit 的最终战：巨型石兽（StoneBeast，一条被晶体/金属侵蚀的巨龙）。战斗横跨多个大阶段，回收本关全部能力（成年龙飞行、龙剑近战/回旋镖、QTE），是整关玩法的收束点。

承接 [Summit 总览](../README.md)。

---

## 内部架构：按阶段分区

StoneBoss 目录按战斗流程切成几个大区，每区一套 actor + 能力：

### 1. `Chase/` — 追逐段（骑成年龙逃离石兽/风暴）
玩家骑成年龙在坍塌的山洞/风暴中逃，石兽在后追。
- **`StormChase/`**（最大子区）— 海量环境障碍与关卡事件：落石/落柱/落晶体（`FallingObstacles/`）、破裂石缝（`BreakingStoneGaps/`）、洞穴尖刺、上升岩、生长金属藤、龙卷风（`Tornado/StormDragon*`）、火山喷发、瀑布、天花板崩塌等，各配 `EffectHandler`/`Manager`/`EventHandler` 三件套驱动。
- **`StormDragonChase/`** — 风暴龙追击者本体（`AStormDragonChase`）：跟随样条、多重闪电攻击、释放魔法球（`ASummitStormMagicSphere`）。
- 其余：`CrystalCloud/`、`ElectricTrap/`、`SplineObstacleSpawn/`。

### 2. `StormPeak/` — 峰顶战（石兽峰形态，分相位攻击）
- **`AStoneBossPeak`**：持 `EStoneBossPeakPhase{ Phase1, Phase2, Phase3, Vulnerable }` 相位枚举，用 `UHazeCapabilityComponent` 挂五个攻击/目标能力：`TargetingCapability`、`ProximityAttack`、`ProjectileAttack`、`CrystalBreathAttack`、`ActivateShieldAmplifier`。
- 护盾放大器 `AStoneBossPeakShieldAmplifier`（3 个保护者，`TListedActors` 收集），打掉后进入 `Vulnerable`（`CrumbActivateVulnerableState` 联机同步）。弹药 `AStoneBossPeakProjectile`。

### 3. `Finale/` — 终局（爬上石兽本体逐段击破，本区最大）
- **`StoneBossFinale`** — 终局根 actor，做整体旋转/俯仰（Roll/Pitch）编排，给两名玩家施加冲量。
- **`StoneBeastHead/`** — 石兽头部：`AStoneBeastAdulDragonBase : AHazeCharacter`（**成年龙形态放大特化**，头里持 Acid/Tail 两条龙引用）+ 头部滚动/摇晃/等待能力状态机、相机聚焦。
- **`StoneBeastTail/`** — 尾部多段（Segment）跟随链：Lead/Follow/Imitation/Return 能力做蛇形尾巴运动。
- **血量与弱点**：
  - `StoneBeastHealthManager` — 简单整型血条（Max=4），`DamageStoneBeast`/`SetWeakpointTarget` 驱动 Boss 血条 UI（`UBossHealthBarWidget`）。
  - `Weakpoint/` — 弱点系统 + `QTE/` 用龙剑刺击弱点的按键 QTE（Player 侧一整套：DrawBack/HitSuccess/HitFail/Release/Sync 能力 + ButtonMash 组件 + AnimNotify 刺击窗口）。
  - `HeadWeakpoint/` — 头部弱点，闪电打击。
- **龙剑可破物**：`SwordBreakables/`（`AStoneBreakableActor` + 血量/再生组件）、`Destructibles/StoneBeastCrystalJungle`（回旋镖攻击目标）、`Finale/ClimbObstacles/`（攀爬障碍：晶体尖刺爆裂/滑落）。
- 辅助：`Debris/`（坠落平台 QTE）、`Neck/`（脖子附着/自定义相机）、`Spline/`（玩家沿石兽身体的重生样条 `StoneBeastPlayerSpline`）、`FullscreenCameraManager/`、`StoneBeastLevelSegmentActors/`。

### 4. `StormBackgroundSystems/` — 背景氛围
远景风暴山峦旋转、闪电（`StoneBeastStormLightningManager`）、生成山脉——纯演出。

---

## 数据流要点

- **相位机**：各阶段（Peak/Head）都用**枚举 State + CapabilityComponent 挂攻击能力**的模式，能力靠 `ShouldActivate` 读当前相位决定出招。
- **三件套演出**：环境障碍普遍是 `Actor` + `EffectHandler`（表现）+ `Manager`（编排/生成），与 Gameplay 的 EventHandler 范式一致。
- **联机**：相位切换、伤害用 `CrumbFunction` 同步（如 `CrumbActivateVulnerableState`）。
- **血量抽象**：Boss 血是"弱点计数"而非连续 HP（`MaxStoneBeastHealth = 4`，每破一个弱点 -1）。

---

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 复用 | `AAdultDragon` 骑乘/飞行 | [Dragons](../Dragons/) | `Chase/` 段玩家骑成年龙逃离 |
| 特化 | `AStoneBeastAdulDragonBase : AHazeCharacter` | [Dragons](../Dragons/) | 石兽头部 = 放大版成年龙，持 Acid/Tail 龙引用 |
| 被攻击 | `ADragonSword` / 回旋镖 | [DragonSword](../DragonSword/) | `SwordBreakables/`、`StoneBeastCrystalJungle`、弱点 QTE 用龙剑刺击 |
| 依赖 | `UHazeCapabilityComponent` + State 枚举 | Gameplay | Peak/Head 相位机挂攻击能力 |
| 依赖 | `UBossHealthBarWidget` | GUI | `StoneBeastHealthManager` 全屏 Boss 血条 |
| 依赖 | `UHazeListedActorComponent` / `TListedActors` | Core | 收集护盾放大器、段 actor |
| 联机 | `CrumbFunction` | Core Network | 相位/伤害同步 |

---

## 关键文件

- 峰顶：`StoneBoss/StormPeak/StoneBossPeak.as`、`StoneBoss/StormPeak/Capabilities/`
- 终局根：`StoneBoss/Finale/StoneBossFinale.as`、`StoneBoss/Finale/StoneBeastHealthManager/StoneBeastHealthManager.as`
- 石兽本体：`StoneBoss/Finale/StoneBeastHead/StoneBeastAdulDragonBase.as`、`StoneBoss/Finale/StoneBeastHead/StoneBeastHead.as`、`StoneBoss/Finale/StoneBeastTail/`
- 弱点 QTE：`StoneBoss/Finale/Weakpoint/QTE/`、`StoneBoss/Finale/Weakpoint/StoneBossWeakpoint.as`
- 龙剑可破物：`StoneBoss/Finale/SwordBreakables/StoneBreakableActor.as`、`StoneBoss/Finale/Destructibles/StoneBeastCrystalJungle.as`
- 追逐：`StoneBoss/Chase/StormDragonChase/StormDragonChase.as`、`StoneBoss/Chase/StormChase/`
</content>

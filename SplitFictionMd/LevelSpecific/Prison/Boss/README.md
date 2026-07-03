# LevelSpecific / Prison / Boss — 持刀狱警 Boss 战

> 路径：`LevelSpecific/Prison/Boss/`（111 文件）
>
> Prison 的剧情 Boss：一名持巨型剪刀/刀的狱警。多阶段战斗，招式繁多（横斩、波斩、旋风、锯齿、甜甜圈、克隆、抓取投掷、磁力猛砸、地面拖痕、齐射等），最终深入 Boss 脑内（Brain）完成击杀。回收全关磁力/黑客/追杀能力。

承接 [Prison 总览](../README.md)。

---

## 职责

实现 Boss 本体、其攻击招式群、AI 决策（Brain）、追杀处决与终局脑内阶段。

---

## 内部架构

### Boss 本体

`APrisonBoss : AHazeCharacter`（`PrisonBoss.as`）——挂多个武器/头/手部挂点组件与一批 `TSubclassOf<>` 效果配置（受击/死亡/相机震动/血条）。行为由 `PrisonBossSheet` 装配的状态能力驱动，`EPrisonBossAttackType` 枚举定义攻击类型。

配套本体文件：`PrisonBossAnimInstance` / `PrisonBossAnimationData`（动画）、`PrisonBossAttackDataComponent`（攻击数据）、`PrisonBossSettings`（设置四件套）、`PrisonBossChaseExecutioner`（追杀处决）、`PrisonBossChokeManager` / `PrisonBossVersusButtonMashManager`（掐脖/搓按键 QTE）、`PrisonBossVoActor`（语音）。

### 攻击招式（Capabilities/Attacks/）

每种招式一个子目录，是能力四件套的密集应用。招式与其对应 Actor：

| 招式（Capabilities/Attacks/） | 对应 Actor（Actors/） | 说明 |
|---|---|---|
| `HorizontalSlash` / `WaveSlash` | `PrisonBossHorizontalSlashActor` / `PrisonBossWaveSlashActor` | 横斩 / 波斩 |
| `Scissors` / `DashSlash` | `PrisonBossScissorsAttack` | 剪刀 / 冲刺斩 |
| `Spiral` / `ZigZag` / `Donut` | `PrisonBossZigZagAttack` / `PrisonBossDonutAttack` | 旋风 / 锯齿 / 甜甜圈弹幕 |
| `Volley` | `PrisonBossVolleyProjectile` | 齐射弹幕 |
| `GroundTrail` | `PrisonBossGroundTrailAttack` | 地面拖痕危险区 |
| `Clone` | `PrisonBossClone` | 分身 |
| `GrabPlayer` / `GrabDebris` | `PrisonBossMagneticDebris` | 抓玩家 / 抓碎片投掷 |
| `MagneticSlam` | `PrisonBossRepelSurface` | 磁力猛砸（复用磁场） |
| `HackableMagneticProjectile` | `PrisonBossHackableMagneticProjectile`（+ `...MovementCapability : URemoteHackableBaseCapability`） | 可黑客磁力弹：玩家能黑掉弹体反打 |
| `PlatformDangerZone` | `PrisonBossPlatformDangerZone` | 平台危险区 |

其余能力目录：`Movement/`（移动）、`Player/TakeControl/`（夺取玩家控制的过场）。

### Boss 大脑（Brain/）— 终局脑内阶段

击杀前进入 Boss 脑内的谜题/攻击阶段：`PrisonBossBrain`（主控）、`PrisonBossBrainNeuron` + `NeuronManager`（神经元）、`BrainEye` + `EyeCable` + `EyePulseAttack`（眼球攻击）、`BrainHackablePanel` / `BrainButtonCover`（可黑客面板）、`BrainGroundDrawAttack`、`BrainBridge` / `BrainPlatform` / `BrainCable`（场景）。

### 其它

`Brain/`（大脑）、`Components/`、`EffectEventHandlers/`（表现事件）、`Audio/`。

---

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 复用 | `MagneticField/` | 磁场 | `MagneticSlam`、`PrisonBossRepelSurface`、`PrisonBossMagneticDebris` 走磁场斥力/物理体系 |
| 复用 | 远程黑客系统 | LevelActors/RemoteHackables | `PrisonBossHackableMagneticProjectileMovementCapability : URemoteHackableBaseCapability`；脑内 `BrainHackablePanel` 可黑客 |
| 依赖 | `AHazeCharacter` + CapabilitySheet | Core | Boss 本体与招式全用能力四件套 + 标签互斥切换攻击状态 |
| 依赖 | QTE / 相机 | Gameplay | 掐脖、搓按键、追杀处决借 Gameplay 通用系统 |

---

## 关键文件

- `Boss/PrisonBoss.as` — Boss 本体、挂点、`EPrisonBossAttackType`、CapabilitySheet 装配。
- `Boss/Capabilities/Attacks/` — 招式能力群（按招式分子目录）。
- `Boss/Actors/` — 招式对应的场景 Actor。
- `Boss/Brain/PrisonBossBrain.as` — 脑内终局主控。
- `Boss/PrisonBossChaseExecutioner.as` — 追杀处决。
- `Boss/PrisonBossSettings.as` — Boss 设置四件套。

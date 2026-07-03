# Gameplay / Movement / Moves —— 基础动作库

> 职责：玩家的**基础运动动作库**——凡是"随时随地都能做、不依赖世界里某个激活点"的动作都在这里。共 **166 文件**，分成 25 个动作分组。
>
> 上游背景：Move 四件套范式与标签矩阵见 [Gameplay 设计哲学](../../DesignPhilosophy.md) 与 [Movement 总览](../README.md)。本文讲**有哪些动作、四件套怎么装、以 AirDash 做完整范例、标签矩阵如何运转**。

---

## 与 MovesContextual 的一句话区别

| | Moves（本目录） | MovesContextual |
| --- | --- | --- |
| 触发 | 玩家自己按键即可（空中按冲刺就空冲） | 需世界里有 `UTargetableComponent` 激活点（靠近抓钩点才能抓钩） |
| 例子 | 跳、蹲、冲刺、滑铲、墙跑、攀边 | 抓钩、荡绳、爬杆、栖点、梯子、重力井 |

---

## 25 个动作分组

| 分组 | 文件数 | 说明 |
| --- | --- | --- |
| AirDash | 4 | 空中冲刺（标准四件套，本文范例） |
| AirJump | 5 | 二段跳（含落地重置能力 GroundedReset） |
| AirMotion | 4 | 空中基础运动（下落/空中位移/发射冲量静态库） |
| Crouch | 5 | 蹲伏（含强制蹲体积 ForceCrouchVolume、蹲伏相机） |
| FloorMotion | 5 | 地面基础走跑（所有地面运动的底座 + Tags） |
| FloorSlowdown | 3 | 地面减速（急停/贴边减速） |
| Jump | 4 | 地面跳（含跳跃计数器 JumpCounter） |
| Land | 7 | 落地（高速落地/致命落地/眩晕落地/落地追踪/落地体积） |
| LedgeGrab | 14 | 抓边悬挂（进入/攀爬/下坠进入/转墙跑/取消/冲刺/多套 Settings） |
| LedgeMantle | 17 | 翻越矮墙上沿（大量分动作能力 + Tags/Settings） |
| PlayerActionMode | 2 | 动作模式组件与体积（切换玩家运动风格） |
| RollDash | 4 | 翻滚冲刺（地面滚动式冲刺） |
| RollDashJump | 1 | 翻滚冲刺接跳 |
| Skydive | 6 | 跳伞式下落（含相机能力/Tags/静态库） |
| Slide | 11 | 滑铲（移动/相机/俯仰钳制/多种触发体积/重生点） |
| SlideDash | 4 | 滑铲冲刺（含启动能力 Startup） |
| SlideJump | 4 | 滑铲跳 |
| SplineLock | 6 | 锁定到样条（最佳样条挑选/橡皮筋/求解器扩展/触发区） |
| Sprint | 10 | 冲刺奔跑（激活/停用/转身/相机/触发体积/Tags/静态库） |
| StepDash | 4 | 步伐冲刺（地面短冲） |
| Strafe | 11 | 横移锁定（地面/空中/跳跃各一套四件套） |
| UnwalkableSlide | 3 | 不可行走坡面下滑 |
| Vault | 7 | 翻越障碍（Component/Settings/Tags + 分动作） |
| WallRun | 18 | 墙跑（进入/评估/攀边/转身/跳出/转移/相机 + 多套 Settings） |
| WallScramble | 7 | 攀墙（爬/跳/退出 + Statics/Tags） |

> 观察：**分动作越多的动作，文件越多**。AirDash 是最纯粹的四件套（4 文件），而 WallRun/LedgeMantle/LedgeGrab 是"一个状态、多个分动作能力"的复合体（14~18 文件）——但仍共享**一套** Component/Settings。

---

## 四件套结构

一个动作分组内的文件按后缀归类：

| 后缀 | 基类 | 角色 | 生命周期 |
| --- | --- | --- | --- |
| `*Capability` | `UHazePlayerCapability` | **行为**：激活条件 + 每帧逻辑 | 能力激活/停用，局部变量随之丢弃 |
| `*Component` | `UActorComponent` | **状态**：跨激活保留的量 | 随玩家常驻，`GetOrCreate` 获取 |
| `*Settings` | `UHazeComposableSettings` | **参数**：可分层覆盖的数值 | 关卡/触发体注入覆盖 |
| `*Statics` | 命名空间 / mixin | **工具**：查询/触发接口 | 无状态，供外部调用 |
| `*Tags` | 命名空间 | **标签**：本动作的分类/子状态标签 | 编译期常量 |
| `*Volume` / `RespawnPoint` | Actor | 关卡放置物：强制进入某动作、限速、重生 | 世界实例 |

---

## 范例：AirDash 空中冲刺（完整拆解）

AirDash 是最标准的四件套，读懂它就懂了整个 Moves。

### ① Component —— 状态载体
`Gameplay/Movement/Player/Moves/AirDash/PlayerAirDashComponent.as`

```cpp
class UPlayerAirDashComponent : UActorComponent
{
    UPlayerAirDashSettings Settings;      // 指向本动作参数
    EAirDashDirection DashDirection;      // 前/左/右/后
    bool bCanAirDash = false;             // 核心状态：还能不能空冲（落地后重置）
    TArray<FAirDashAutoTarget> AutoTargets;             // 世界里注册的自动吸附目标
    TInstigated<FAirDashDirectionConstraint> DirectionConstraint;  // 方向锥约束
    // StartDash/StopDash/IsAirDashing/GetTimeSince… —— 状态读写接口
}
```

关键点：**`bCanAirDash` 存在 Component 而非 Capability**，因为它必须跨越"能力停用"存活——落地后由 `PreTick` 重置为 true，起跳一次消耗为 false。这正是 Component 与 Capability 分工的教科书案例。

### ② Settings —— 可组合参数
`PlayerAirDashSettings.as`（`UHazeComposableSettings`）

```cpp
float DashDuration = 0.1666;    float DashDistance = 250.0;
float DashAccelerationDuration = 0.05;   float DashDecelerationDuration = 0.05;
float InputBufferWindow = 0.08;          float RedirectionWindow = 0.0;
float ForwardDashAngle = 50.0;  float BackwardDashAngle = 65.0;  // 角度阈值决定前冲/横移/后撤
float GravityDurationAtEnd = 0.1;        // 冲刺末段恢复重力
```

关卡设计师可对特定区域注入不同 Settings 分层覆盖这些数值，无需改代码。

### ③ Statics —— 工具接口
`PlayerAirDashStatics.as`（mixin）

```cpp
mixin void ResetAirDashUsage(AHazePlayerCharacter Player)   { …bCanAirDash = true;  }
mixin void ConsumeAirDashUsage(AHazePlayerCharacter Player) { …bCanAirDash = false; }
```

供其他系统（如某个关卡机关、某个 buff）重置或消耗空冲资格，而不必知道 Component 内部结构。

### ④ Capability —— 行为逻辑
`PlayerAirDashCapability.as`（`UHazePlayerCapability`），四个关键回调：

- **`ShouldActivate()`**：一连串否决检查——不在地面、`bCanAirDash` 为真、刚起跳不久不行、输入缓冲窗内按了冲刺键（或无障碍自动空冲）。
- **`OnActivated()`**：`BlockCapabilities(BlockedWhileIn::Dash)` 传播互斥；消耗输入；计算方向（输入方向 → 自动目标吸附 → 方向锥约束）；判定前冲/横移/后撤并设目标朝向；构造 `FDashMovementCalculator`；触发动画 `n"StartAirDash"` 与特效 `Trigger_AirDash_Started`。
- **`TickActive()`**：三段式驱动 Core 管线——`PrepareMove` → `AddDeltaWithCustomVelocity`（由计算器算出本帧位移/速度）+ 末段 `AddGravityAcceleration` → `ApplyMove`；`HasControl` 侧真算，远端 `ApplyCrumbSyncedGroundMovement` 回放。
- **`OnDeactivated()`**：`StopDash`、按计算器退出速度钳制水平速度、`Trigger_AirDash_Stopped`、`UnblockCapabilities`。

### 数据流一图

```
按下冲刺键（空中）
   │  ShouldActivate: bCanAirDash? 未被 BlockedWhileIn 阻塞? 输入缓冲窗内?
   ▼
OnActivated
   │  BlockCapabilities(BlockedWhileIn::Dash) ──► 攀爬/墙跑/游泳本帧失活
   │  Component.StartDash() + bCanAirDash=false（写状态）
   │  算方向 → 自动吸附 → 方向锥 → 判前/横/后
   ▼
TickActive（每帧，走 Core 管线）
   PrepareMove(Movement) → AddDelta(方向×计算器位移) → ApplyMove → Core Resolver 扫掠写回
   ▼
OnDeactivated（计算器结束/落地/撞墙）
   钳制退出速度、UnblockCapabilities、恢复
```

---

## 标签矩阵如何工作

以 `PlayerAirDashCapability` 的类默认声明为例（三层）：

```cpp
// ① 我是谁 —— 分类标签（供别人 IsAnyCapabilityActive 查询 / 供 Debug 归组）
default CapabilityTags.Add(CapabilityTags::Movement);
default CapabilityTags.Add(PlayerMovementTags::CoreMovement);
default CapabilityTags.Add(PlayerMovementTags::Dash);
default CapabilityTags.Add(PlayerMovementTags::AirDash);

// ② 我在这些状态下不能激活 —— 被阻塞矩阵（共 11 条）
default CapabilityTags.Add(BlockedWhileIn::WallScramble);
default CapabilityTags.Add(BlockedWhileIn::WallRun);
default CapabilityTags.Add(BlockedWhileIn::Swimming);
default CapabilityTags.Add(BlockedWhileIn::Grapple);
default CapabilityTags.Add(BlockedWhileIn::Ladder);
default CapabilityTags.Add(BlockedWhileIn::PoleClimb);
default CapabilityTags.Add(BlockedWhileIn::Perch);
default CapabilityTags.Add(BlockedWhileIn::LedgeGrab);
default CapabilityTags.Add(BlockedWhileIn::Swing);
default CapabilityTags.Add(BlockedWhileIn::Vault);
default CapabilityTags.Add(BlockedWhileIn::LedgeMantle);

// ③ 例外放行 —— 即使被阻塞也不失活
default BlockExclusionTags.Add(n"ExcludeAirJumpAndDash");
```

运转机制（三步）：

1. **声明**：每个动作在类默认里静态列出"我怕哪些状态"（`BlockedWhileIn::*`）。
2. **传播**：某动作激活时 `OnActivated` 里调 `Player.BlockCapabilities(BlockedWhileIn::X, this)`；停用时 `UnblockCapabilities`。全库共 55+ 处这种调用。
3. **仲裁**：能力系统每帧比对——凡是持有当前被阻塞标签、且 `BlockExclusionTags` 未放行的能力，`ShouldActivate` 前就被拦下。

标签词典集中在 `Gameplay/Movement/Player/Tags/`：

| 文件 | 内容 |
| --- | --- |
| `BlockedWhileInTags.as` | `BlockedWhileIn::` 约 40 个"状态词"（Core Moves / Contextual / Level） |
| `PlayerMovementTags.as` | `PlayerMovementTags::` 分类词（CoreMovement/Dash/AirDash/Grapple…）+ `PlayerMovementExclusionTags::` 例外词 |
| `MovementDevToggles.as` | 开发调试开关（如 `AutoAlwaysDash`） |

**好处**：加"喷气背包"只需给它打 `BlockedWhileIn::*` 标签就自动融入互斥网络；查一个动作能否在某状态下做，看它的标签声明即可，不必追踪散落的 `if`。

---

## 关键文件

| 文件 | 作用 |
| --- | --- |
| `Player/Moves/AirDash/PlayerAirDash{Capability,Component,Settings,Statics}.as` | 标准四件套范例 |
| `Player/Moves/FloorMovement/PlayerFloorMotion*.as` | 所有地面走跑的底座 |
| `Player/Moves/WallRun/*`（18） | 复合动作范例：一套状态、多分动作能力 |
| `Player/Tags/BlockedWhileInTags.as` | 全局状态标签词典 |
| `Player/Tags/PlayerMovementTags.as` | 分类/例外标签词典 |
</content>

# Gameplay / Movement 功能域总览

> 职责：Split Fiction 中**玩家角色的全部运动能力**——从走跑跳蹲滑等基础动作，到抓钩、攀爬、游泳、荡绳等情境动作，再到相机、输入、边界、音频等支撑层。这是 Gameplay 层最大的模块，共 **468 文件**。
>
> **前提**：本文承接 [Gameplay 设计哲学](../DesignPhilosophy.md)（Move 四件套、标签互斥矩阵）与 [Core / Movement](../../Core/Movement/README.md)（底层求解内核）。这里不重复哲学，只讲 Movement 模块自身的**目录地图、装配范式与分工边界**。

---

## 一句话定位

> **Movement 定义玩家"想怎么动"（激活条件、方向、消耗、互斥），Core 的 Resolver 管线负责"实际怎么动"（碰撞扫掠、去穿透、联机回放）。** 数百个动作能力挂在同一个玩家身上，靠标签矩阵无冲突地共存与切换。

---

## 六个子目录

| 子目录 | 路径 | 文件数 | 定位 |
| --- | --- | --- | --- |
| Player | `Gameplay/Movement/Player` | 424 | 核心：玩家的所有动作能力 + 支撑层（相机/输入/组件/边界…） |
| Audio | `Gameplay/Movement/Audio` | 25 | 移动音频：脚步、植被、滑行、划水的 trace 与发声 |
| ResolverExtensions | `Gameplay/Movement/ResolverExtensions` | 9 | 接入 Core Resolver 的运动约束扩展（圆形约束/样条碰撞） |
| Tutorials | `Gameplay/Movement/Tutorials` | 7 | 单招式教学提示（空冲/二段跳/游泳/荡绳…） |
| Helpers | `Gameplay/Movement/Helpers` | 2 | 通用工具（预测辅助、摇杆急拨追踪） |
| Triggers | `Gameplay/Movement/Triggers` | 1 | 触发体（仅地面玩家触发器） |

其中 **Player 占九成以上**，是理解 Movement 的主战场。

---

## Player 的 14 个子目录地图

Player 内部分为「**动作**」与「**支撑**」两大类：

### 动作类（占绝大多数文件）

| 子目录 | 文件数 | 内容 | 文档 |
| --- | --- | --- | --- |
| Moves | 166 | **基础动作库**：空冲/二段跳/蹲/跑/冲刺/滑铲/翻滚/墙跑/攀边/游泳/梯子等，无需世界激活点即可做 | [Moves/README.md](../Moves/README.md) |
| MovesContextual | 165 | **情境动作库**：抓钩/荡绳/爬杆/栖点/重力井等，需世界里注册激活点（Targetable）才能做 | [MovesContextual/README.md](../MovesContextual/README.md) |
| MovesLevel | 4 | **关卡专属动作**：撑杆跳（Polevault）等只在特定关卡使用的动作 | 见 Player/README |
| MoveIntoPlayer | 6 | 让外部物体"移动进入玩家"的移动数据/求解器（形状匹配、旋转对齐） | 见 Player/README |

### 支撑类（非动作，服务于动作）

| 子目录 | 文件数 | 内容 | 文档 |
| --- | --- | --- | --- |
| PlayerUtilities | 28 | 杂项玩家能力：居中视角(CenterView)、找队友指示、Blob 阴影、落向样条等 | [Player/README.md](../Player/README.md) |
| Component | 11 | **运动组件核心**：`UPlayerMovementComponent`、击退、继承速度、碰撞忽略、默认设置 | [Player/README.md](../Player/README.md) |
| Camera | 10 | 运动相关的相机能力：随速调距、辅助跟随、对齐世界上方向、看向样条 | [Player/README.md](../Player/README.md) |
| Audio | 7 | 玩家音频代理发声/监听器空间化（第三人称/横版/分屏） | [Player/README.md](../Player/README.md) |
| Input | 5 | 移动输入形状（方形/椭圆）、朝向、调试 | [Player/README.md](../Player/README.md) |
| LaunchTo | 5 | 把玩家抛射到目标点/带冲量/曲线插值（含 Crumb 联机） | [Player/README.md](../Player/README.md) |
| Boundary | 3 | 双人边界约束（相机视锥/领跑距离/双人最大间距），走 Resolver 扩展 | [Player/README.md](../Player/README.md) |
| AutoRun | 3 | 自动奔跑（过场/引导段落） | [Player/README.md](../Player/README.md) |
| Tags | 3 | **标签词典**：`BlockedWhileIn::`、`PlayerMovementTags::`、Dev 开关 | 本文下方 |
| EffectHandlers | 2 | 核心移动/游泳的特效事件分发（`Trigger_AirDash_Started` 等） | [Player/README.md](../Player/README.md) |
| Settings | 1 | 玩家墙面通用设置 | 见 Player/README |

---

## Move 四件套范式速览

每个动作是 `Player/Moves/<动作名>/` 或 `MovesContextual/<动作名>/` 下的一组文件，遵循严格命名（以 AirDash 为例）：

```
Moves/AirDash/
├── PlayerAirDashCapability.as   ← 行为：ShouldActivate + TickActive（继承 UHazePlayerCapability）
├── PlayerAirDashComponent.as    ← 状态：bCanAirDash、方向、自动目标（继承 UActorComponent）
├── PlayerAirDashSettings.as     ← 参数：DashDistance/DashDuration…（继承 UHazeComposableSettings）
└── PlayerAirDashStatics.as      ← 工具：ResetAirDashUsage / ConsumeAirDashUsage（mixin）
```

- **Capability** 决定"何时能做、做时每帧怎么动"。能力停用会丢弃自身局部状态。
- **Component** 存需要跨激活保留的量（还能冲几次、上次冲刺时间），因为能力停用会清空局部变量。
- **Settings** 是可组合参数，关卡/触发体可分层覆盖。
- **Statics** 供其他能力查询/触发本动作（如 `ConsumeAirDashUsage`）。

复杂动作会有**多个 Capability**协作（如 WallRun 有 18 个文件：进入/评估/跳出/攀边/转身/相机各一个能力），但只有一套 Component/Settings。四件套是最小骨架，不是硬性文件数。

> 详见 [Moves/README.md](../Moves/README.md) 的 AirDash 完整拆解。

---

## 标签互斥矩阵（词典在 `Player/Tags/`）

`Player/Tags/BlockedWhileInTags.as` 定义了全套"状态标签"（约 40 个 `BlockedWhileIn::*`），`PlayerMovementTags.as` 定义了"分类标签"（`CoreMovement`/`AirDash`/`Grapple`…）。

- 每个动作能力用 `CapabilityTags.Add(BlockedWhileIn::Swimming)` 声明"游泳时我失活"。
- 游泳能力激活时调 `Player.BlockCapabilities(BlockedWhileIn::Swimming, this)`，所有声明该标签的动作**本帧自动失活**——彼此无需知道对方存在。
- `BlockExclusionTags` 是例外放行（如 `ExcludeAirJumpAndDash`）。

全模块共 **55+ 处** `BlockCapabilities` 调用编织成这张矩阵。加新动作只需打标签，不改老动作。

```
BlockedWhileIn 命名空间（节选，Player/Tags/BlockedWhileInTags.as）
├── Core Moves:   AirMotion / FloorMotion / Jump / AirJump / Dash / Slide /
│                 WallScramble / WallRun / LedgeGrab / Sprint / Strafe / Vault …
├── Contextual:   Crouch / Swimming / Grapple / Ladder / Swing / GravityWell /
│                 PoleClimb / Perch / PerchSpline / ApexDive
└── Level:        ShapeShiftForm
```

---

## 与 Core 移动管线的分工

Move 的 Capability **不自己写 transform**，而是驱动 Core 的移动组件：

```cpp
UPlayerMovementComponent MoveComp;    // Player/Component/PlayerMovementComponent.as，继承 Core 的 UHazeMovementComponent
USteppingMovementData Movement;       // Core 的运动数据类型（六种模式之一）

// TickActive 内的典型三段式：
if (MoveComp.PrepareMove(Movement))              // ① 拍下初始状态，锁定组件
{
    Movement.AddDeltaWithCustomVelocity(...);    // ② 累加本帧位移/速度/重力/旋转
    Movement.RequestFallingForThisFrame();
    MoveComp.ApplyMove(Movement);                // ③ 交给 Core Resolver 扫掠求解并写回
}
```

| 层 | 谁 | 负责 |
| --- | --- | --- |
| 意图/决策 | **Gameplay / Movement** | 能不能做、往哪做、消耗次数、动画触发、互斥传播 |
| 执行/一致性 | **Core / Movement** | 碰撞扫掠、去穿透、沿面滑动、地面跟随、Crumb 联机回放 |

`UPlayerMovementComponent`（`Player/Component/`）是 Core `UHazeMovementComponent` 的玩家特化子类，附加了 MoveIntoPlayer、输入平面锁、撞击回调等玩家专属逻辑。

联机：两名玩家（Mio/Zoe）各有独立 `UPlayerMovementComponent`，求解在 `HasControl` 侧真算，远端以 `NetworkMode=Crumb` 回放。

---

## 子目录文档

- [Moves/README.md](../Moves/README.md) — 基础动作库（166）
- [MovesContextual/README.md](../MovesContextual/README.md) — 情境动作库（165）
- [Player/README.md](../Player/README.md) — Player 支撑层
- [Audio/README.md](../Audio/README.md) — 移动音频（25）
- [ResolverExtensions/README.md](../ResolverExtensions/README.md) — 运动求解器扩展（9）
- [Tutorials/README.md](../Tutorials/README.md) — 单招式教程（7）
</content>
</invoke>

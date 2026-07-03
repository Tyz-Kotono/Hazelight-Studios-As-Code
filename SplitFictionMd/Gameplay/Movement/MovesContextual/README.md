# Gameplay / Movement / MovesContextual —— 情境动作库

> 职责：玩家的**情境运动动作库**——需要世界里预先注册"激活点"才能做的动作：抓钩、荡绳、爬杆、栖点跳跃、梯子攀爬、重力井等。共 **165 文件**，12 个动作分组。
>
> 上游背景：四件套与标签矩阵见 [Movement 总览](../README.md) 与 [Moves/README.md](../Moves/README.md)。本文讲**激活点机制（Targetable）、与基础动作的本质区别、以 Grapple 为代表**。

---

## 与基础动作（Moves）的区别

| | Moves | MovesContextual（本目录） |
| --- | --- | --- |
| 触发前提 | 玩家自身状态满足即可 | 世界里必须有一个 **激活点** 且玩家在其激活范围内 |
| 激活点载体 | 无 | `UContextualMovesTargetableComponent`（Core Targetable 框架） |
| 世界侧内容 | 顶多几个 Volume | 每个动作带一批 **Actor**（抓钩点、梯子、爬杆、栖点样条…） |
| UI | 一般无 | 靠近时显示可交互 **Widget** 提示 |
| 例子 | 空冲、跳、滑铲 | 抓钩到点、荡绳、爬梯、栖点跳、重力井发射 |

本质：**基础动作是"能力"，情境动作是"能力 + 世界激活点 + 瞄准/UI 系统"的组合。**

---

## 12 个动作分组

| 分组 | 文件数 | 内容 |
| --- | --- | --- |
| ActivationPoints | 6 | **激活点与瞄准系统的公共底座**（本文核心，下详） |
| Grapple | 54 | **抓钩**（最大分组）：多种钩点 Actor + 瞄准/进入/冲刺/弹跳/多种目标类型 |
| GrappleV2 | 1 | 抓钩 V2 的新钩子 Actor（迭代中） |
| Ladder | 21 | 梯子：Actor + 攀爬/上下/冲刺/进入（地面/顶部）/退出/悬挂/跳出 |
| LookAt | 1 | 情境动作看向能力（角色朝向激活点） |
| PerchPoint | 15 | 栖点：点 Actor + 跳上/跳下/降落/停驻/2D 停驻/相机 + 多组件 |
| PerchSpline | 9 | 栖点样条：沿样条的空中运动/冲刺/掉落/跳/移动 |
| PoleClimb | 16 | 爬杆：杆 Actor + 进入/移动/冲刺/连冲/跳/转身/上下栖点/取消 |
| SciFi | 9 | 科幻专属：**重力井**（导管/电梯移动/发射/进入引导/相机） |
| Swimming | 14 | 游泳：水面/水下/俯冲(ApexDive)/冲刺/跳出/上浮 + 水流体积 |
| Swing | 17 | 荡绳：空中荡（相机/取消/冲击/跳/绳/移动）+ 墙上荡（相机/取消/跳/移动） |
| TriggerVolumes | 2 | 情境动作触发体积（进入区域强制启用某情境动作） |

> 结构惯例：分组内常再分 `Actors/`（世界放置物）、`Capabilities/`（能力，常按子动作细分）、`Components/`。例如 Ladder 有 `Ladder/Actors/Ladder.as` + `Ladder/Capabilities/LadderClimbCapability.as` 等 14 个能力。

---

## 激活点机制（ActivationPoints）

这是 MovesContextual 区别于 Moves 的核心系统，位于 `MovesContextual/ActivationPoints/`（6 文件）。

### `UContextualMovesTargetableComponent` —— 世界侧激活点
`ActivationPoints/ContextualMovesTargetableComponent.as`，继承 Core 的 `UTargetableComponent`：

```cpp
class UContextualMovesTargetableComponent : UTargetableComponent
{
    default TargetableCategory = n"ContextualMoves";
    default UsableByPlayers = EHazeSelectPlayer::Both;   // 双人皆可

    float ActivationRange = 1500.0;        // 可激活半径
    float AdditionalVisibleRange = 800.0;  // 提前可见（尚不可激活）的额外范围
    // 一批激活条件开关：
    //   bTestCollision       —— 是否要求对相机不被遮挡
    //   bShouldValidateWorldUp —— 玩家世界上方向是否需与点匹配
    //   bRestrictToForwardVector —— 玩家是否必须在点"正面"
    //   AirActivationSettings   —— 仅空中/仅地面/皆可
    //   HeightActivationSettings —— 仅上方/仅下方/皆可（含可见但不可用变体）

    bool CheckTargetable(FTargetableQuery& Query) const override { … }  // 每帧打分裁决
}
```

它把 Core 通用的 Targetable 打分接口（`ApplyVisibleRange` / `ApplyTargetableRangeWithBuffer` / `ScoreLookAtAim` / `RequireNotOccludedFromCamera`）包装成"情境动作激活点"的语义：**在范围内 + 满足空中/高度/朝向/遮挡条件 → 成为候选目标**。

关卡设计师把各种 `AGrapplePoint`、`ALadder`、`APoleClimbActor` 摆进世界，它们内部就挂着这类 Targetable 组件。

### `UContextualMovesTargetingCapability` —— 玩家侧瞄准
`ActivationPoints/ContextualMovesTargetingCapability.as`，玩家常驻能力：

```cpp
void TickActive(float DeltaTime)
{
    FTargetableWidgetSettings WidgetSettings;
    WidgetSettings.TargetableClass = UContextualMovesTargetableComponent;  // 只关心情境激活点
    WidgetSettings.MaximumVisibleWidgets = 1;                              // 一次只提示最优的一个
    PlayerTargetablesComponent.ShowWidgetsForTargetables(WidgetSettings);  // 显示交互提示 Widget
}
```

它每帧向 Core 的 `UPlayerTargetablesComponent` 查询"当前最优情境激活点"，并显示对应 Widget（`UContextualMovesWidget`，Mio/Zoe 各有皮肤）。

### 完整链路

```
关卡放置 AGrapplePoint（内含 UContextualMovesTargetableComponent 激活点）
        │
        ▼  每帧
Core Targetable 系统对所有激活点打分（范围/朝向/遮挡/空中/高度条件）
        │
        ▼
UContextualMovesTargetingCapability 取最优点 → 显示交互 Widget 提示
        │
        ▼  玩家按交互键
对应情境动作 Capability 的 ShouldActivate 通过（读到"有可用激活点"）
        │
        ▼
BlockCapabilities(BlockedWhileIn::Grapple…) 传播互斥 → 走 Core 移动管线执行
```

三个支撑文件：`ContextualMovesTargetableSettings.as`（激活点参数）、`ContextualMovesTargetableVisualizer.as`（编辑器可视化范围）、`ContextualMovesWidget.as` + `PlayerContextualMovesTargetingComponent.as`（Widget 与玩家侧状态）。

---

## 代表：Grapple 抓钩（54 文件，最大分组）

Grapple 展示了情境动作"一个机制、多种世界物"的规模：

- **`Grapple/Actors/`**：多种钩点各是一个 Actor——`GrappleHookPoint`（标准钩点）、`GrappleBashPoint`（弹跳）、`GrapplePerchPoint`（钩到栖点）、`GrappleSlidePoint`、`GrappleWallrunPoint`、`GrappleToPolePoint`、`GrappleWallScramblePoint` 等。每个 Actor 内含 `UGrapplePointComponent` 与激活点组件，并广播 `OnPlayerInitiatedGrappleToPoint` / `OnGrappleHookReachedGrapplePoint` / `OnPlayerFinishedGrapplingToPoint` 等事件供关卡挂接。
- **`Grapple/BashCapability/`**：弹跳钩的瞄准/快速进入等能力。
- 抓钩过程仍遵循标签矩阵：激活时 `BlockCapabilities(BlockedWhileIn::Grapple)`，让空冲、二段跳等基础动作本帧失活（见 Moves 里 AirDash 声明了 `BlockedWhileIn::Grapple`）。

抓钩的 `BlockedWhileIn::Grapple` 与 `BlockedWhileIn::GrappleEnter` 两个状态标签，以及 `PlayerMovementExclusionTags::ExcludeGrapple` 例外标签，都在 `Player/Tags/` 词典里。

---

## 关键文件

| 文件 | 作用 |
| --- | --- |
| `ActivationPoints/ContextualMovesTargetableComponent.as` | 世界侧激活点（继承 Core Targetable），定义可激活条件 |
| `ActivationPoints/ContextualMovesTargetingCapability.as` | 玩家侧瞄准，每帧取最优点并显示交互 Widget |
| `ActivationPoints/PlayerContextualMovesTargetingComponent.as` | 玩家侧瞄准状态与 Widget 配置 |
| `Grapple/Actors/GrappleHookPoint.as` | 标准抓钩点 Actor 范例（事件广播模式） |
| `Ladder/Actors/Ladder.as` + `Ladder/Capabilities/*` | 情境动作"Actor + 多子动作能力"结构范例 |
| `Swimming/PlayerSwimming*.as` | 游泳分组（水面/水下/俯冲多能力 + 水流体积） |
</content>

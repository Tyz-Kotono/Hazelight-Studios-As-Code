# Core / 模块间协作机制 — Communication

> 本文回答一个问题：Split Fiction **没有中央事件总线**，那么 Core 的 513 个文件靠什么互相协作？
>
> 结论基于对全 `Core/` 目录的 grep 使用量统计 + 代表文件实证。与 [SystemArchitecture.md](./SystemArchitecture.md)（单例骨架）、[EventSystem 相关讨论] 互补。

---

## 一、五类机制的使用量（全 Core 统计）

| 机制 | 命中次数 | 涉及文件 | 定位 |
|---|---:|---:|---|
| **直接获取 + 调用** `::Get()` / `::GetOrCreate()` | **522** | **174** | 🥇 绝对主导 |
| **Crumb 联机** `CrumbFunction` / `HasControl()` | 149 | 51 | 联机一致性骨干 |
| **能力标签阻塞** `BlockCapabilities` / `CapabilityTags` | 131 | 63 | 隐式互斥协调 |
| **委托 / 广播** `AddUFunction` / `Broadcast` / `event` | 114 | 51 | 局部观察者 |
| **可组合设置** `ApplySettings` / `GetSettings` | 97 | 63 | 分层参数注入 |

> 关键数据：**"直接获取并调用"(522 次) 以约 5:1 压倒 "委托广播"(114 次)**。Core 的协作主干是"拿到对象直接调"，而非"发事件等订阅"。

---

## 二、机制详解（按主次）

### 1. 直接获取 + 调用 —— 主干（522 次）

任何模块要与另一模块协作，**先 `Type::Get(Owner)` 拿到组件/单例，再直接调方法**：

```cpp
auto EffectEventComp = UHazeEffectEventHandlerComponent::GetOrCreate(Player);
auto FadeManager     = UFadeManagerComponent::Get(Player);
HealthComp           = UPlayerHealthComponent::Get(Player);
```

- `::Get()` —— 取已存在的组件（取不到返回 null）。
- `::GetOrCreate()` —— 取不到就创建（保证存在）。

**为什么主导**：确定性分屏要求每帧执行顺序固定。直接调用是同步的、即时的、顺序可预测的，没有"事件何时被处理"的不确定性。这是 Haze 框架最根本的协作姿势——**组件即服务，按需现取现用**。

### 2. Crumb 联机 —— 跨机器一致性（149 次）

凡是会改变游戏状态、需两端一致的操作，都走 Crumb：

```cpp
if (HasControl())              // 我是这个对象的控制端吗？
    CrumbKillPlayer(...);      // 经 crumb trail 有序回放到对端
```

重灾区：`PlayerHealthComponent`(12)、`NetworkLockComponent`(15)、各 Trigger(4~5)——**状态变更类模块几乎都包一层 Crumb**。这解释了为什么不用事件广播：异步事件的执行时机难以跨机器对齐，而 Crumb 是带时间戳的有序回放。详见 [Systems.md](./Systems.md) 的 Crumb 概念。

### 3. 能力标签阻塞 —— 隐式互斥（131 次）

模块间**不直接通信也能协调**：能力带标签，激活时阻塞带冲突标签的其它能力：

```cpp
// PlayerSequenceBlockMovementCapability：过场时阻塞移动
// CameraNonControlledTransitionCapability(4)：接管相机时挡住玩家控制
```

这是**声明式协作**：A 不需要知道 B 存在，只要 A 阻塞了 `Movement` 标签，所有 `Movement` 能力自动失活。MoveTo、Interaction、Sequencer、QTE 小游戏都靠它"占用"玩家。

### 4. 委托 / 广播 —— 局部观察者（114 次）

**真正的 pub-sub 在 Core 里存在，但只用于"一对多通知、且不影响确定性逻辑"的局部场景**：

```cpp
// PlayerHealthComponent.as 顶部声明 event
event void FOnPlayerTookDamage(AHazePlayerCharacter Player, float DamageAmount);
event void FPlayerHealthComponentStartDyingSignature();
```

谁关心玩家受伤（UI 闪屏、音频滤波、VO）就 `AddUFunction` 订阅。

> ⚠️ 注意：它是**某个组件自己的局部 event，不是全局总线**——订阅者必须先拿到那个组件实例（又回到机制 1）。`HazeVoxController`(4)、`HazeTeam`(2) 同理。

### 5. 可组合设置 —— 分层参数注入（97 次）

不是"调用"也不是"事件"，而是**数据叠加**。多个来源按优先级把设置 `Apply` 到同一对象，系统取最高优先级生效：

```cpp
Player.ApplySettings(DefaultProxyActivationSettings, this, EHazeSettingsPriority::Gameplay);
auto Settings = UPlayerAimingSettings::GetSettings(Player);  // 读合并结果
```

相机、移动、瞄准、音频、玩家高亮都用它调参。`ApplySettingsTrigger`、`CameraInheritMovementVolume` 让关卡设计师在不写代码的情况下注入参数——**参数协作而非逻辑协作**。

---

## 三、协作全景图

```
                        ┌─ 一个 AHazeActor / AHazePlayerCharacter ─┐
                        │  挂着一堆 Component（各模块的服务端点）     │
                        └───────────────────────────────────────────┘
                                        ▲
        ┌───────────────┬───────────────┼───────────────┬──────────────┐
        │①直接获取调用   │②Crumb        │③标签阻塞       │④局部委托      │⑤设置注入
        │ ::Get()→调方法 │ HasControl()  │ Block标签       │ event/订阅    │ ApplySettings
        │ (522次/主干)   │ +CrumbFunc    │ (隐式互斥)      │ (一对多通知)  │ (分层参数)
        │ 同步·确定      │ (跨机器一致)  │ (声明式)        │ (局部·非关键) │ (数据驱动)
        └───────────────┴───────────────┴───────────────┴──────────────┘

           没有中央事件总线。协作 = 拿到对象直接调 + Crumb 保同步 + 标签防冲突
```

---

## 四、核心结论

Core 的模块协作呈现**清晰的优先级**：

1. **默认用直接调用**（522）——拿到组件就调，同步、确定、顺序可控。Haze 的第一性原则。
2. **状态变更包 Crumb**（149）——凡改游戏状态必经 `HasControl()` + `CrumbFunction`，保证两端一致。
3. **冲突用标签隐式解决**（131）——能力互斥不靠通信，靠声明标签。
4. **事件仅用于局部非关键通知**（114）——UI/音频订阅受伤、死亡等表现层信号，且是组件局部 event 而非全局总线。
5. **参数用设置叠加**（97）——分层 Apply，数据驱动，免代码协作。

**一句话**：Split Fiction 的 Core 刻意**回避全局事件总线**，转而用"组件即服务 + 直接调用"做主干、Crumb 保联机、标签防冲突——一切设计都服务于**确定性分屏联机**这个最高约束。事件广播被降级为"局部、表现层、不影响逻辑确定性"的次要手段。

> 这正是为什么"中心 EventSystem 分发一切"的设想不成立：**异步广播模型与确定性联机天然冲突**。
>
> 对照：玩法效果（命中/射击 → 音效/VFX）才用脚本可见的 pub-sub（EffectEvent 系统），因为那是纯表现层、不影响逻辑确定性。生命周期（加载/建角色/游戏开始）则由引擎 C++ 掌管，AS 通过 `Progress::` / `Game::` / `BeginPlay` 轮询接入。

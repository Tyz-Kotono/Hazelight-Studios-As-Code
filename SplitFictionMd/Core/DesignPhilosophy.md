# Haze 设计哲学 — Design Philosophy

> 本文回答一个比"有什么"更根本的问题：**Split Fiction 的 Haze 引擎脚本层，是按什么范式设计的？为什么是这个范式？一帧里数据如何流动？**
>
> 前面 100+ 篇文档描述"有什么模块、怎么协作"；本篇只讲**范式本身与其第一性原理**，是理解全部文档的钥匙。
>
> 全文只正面阐述 Haze 自身的模型，不做外部框架类比——因为这套模型有它自己的内在一致性，类比反而会失真。

---

## 一、一句话范式

> **Actor 是能力的容器；行为写在 Capability 里，状态存在 Component 里；每帧由一套有序的 Tick 阶段驱动全部 Capability；一切可变状态都以"控制端计算、Crumb 回放到远端"的方式保持双机确定性一致。**

拆成四条支柱，后面逐条论证：

1. **组合而非继承** —— Actor 不靠类继承获得行为，靠"装配一列 Capability"。
2. **行为与状态分离** —— Capability(行为) 与 Component(状态) 是两种正交的东西。
3. **有序帧管线** —— 没有中央调度器发事件，而是 TickGroup 定义的固定阶段顺序。
4. **确定性联机是最高约束** —— 上面三条的具体形态，都是被"确定性分屏联机"这一条推导出来的。

---

## 二、支柱一：Actor = 能力容器（组合优于继承）

### 代码事实
`Core/Players/PlayerCharacter.as` 里，玩家角色**不是**一个写满逻辑的大类，而是一份**装配清单**：

```cpp
class APlayerCharacter : AHazePlayerCharacter
{
    UPROPERTY(DefaultComponent)
    UHazeCapabilityComponent CapabilityComponent;

    default CapabilityComponent.DefaultCapabilities.Add(n"CameraControlCapability");
    default CapabilityComponent.DefaultCapabilities.Add(n"CameraUpdateCapability");
    default CapabilityComponent.DefaultCapabilities.Add(n"PlayerDeathCapability");
    // …共 50 条
}
```

玩家角色 = **50 个能力名字**的集合。角色"会做什么"完全由这张清单决定，而非由类层次决定。

### 量化印证（全库统计）
- **3665 个** Capability 类（继承 `UHazeCapability` / `UHazePlayerCapability` / `UHazeChildCapability`）
- **1661 个** Component 文件
- 一个玩家装配 **50 个**能力

行为被切成了数千个**小而独立的能力单元**，通过"往 `CapabilityComponent` 里加名字"来组合。这就是为什么 [Examples](../Examples.md) 里学习路径的第一课是 `Template_Capability.as`——写游戏 = 写能力 + 装配能力。

### 能力的注册是"数据"，不是"代码"
`DefaultCapabilities.Add(n"CameraControlCapability")` 用的是 **FName 字符串**而非类型引用。这意味着能力清单是可被关卡蓝图、`UHazeRequestCapabilityOnPlayerComponent`（见 `Examples/Capabilities/Example_RequestCapabilities.as`）在**运行时动态增删**的——关卡里放一个弹床 actor，就能给路过的玩家临时挂上"弹跳能力"。**能力集是活的数据，不是死的继承树。**

---

## 三、支柱二：行为(Capability) 与 状态(Component) 正交分离

两种东西，职责严格分工：

| | Capability（行为） | Component（状态） |
|---|---|---|
| 基类 | `UHazeCapability` | `UActorComponent` / `USceneComponent` |
| 含什么 | 每帧逻辑（`TickActive`）、激活条件 | 数据字段 + 读写方法 |
| 生命周期 | 会激活/停用/被阻塞 | 一直存在 |
| 数量级 | 3665 | 1661 |

典型协作：能力在 `Setup()` 里 **拿到组件引用**，之后每帧读写它：

```cpp
UHazeMovementComponent MoveComp;
void Setup()        { MoveComp = UHazeMovementComponent::Get(Owner); }  // 拿状态载体
bool ShouldActivate() const { return MoveComp.Velocity.Size() > 100.0; } // 读状态决定行为
```

### 关键设计：能力无状态持久性，状态无行为主动性
- Capability 停用后再激活，不保证记得上次的中间量——**需要跨激活保留的值，必须存到 Component**。这条纪律在 `Examples/Features/Example_TemporalLog.as` 的 `OnLogState`(每帧记) vs `OnLogActive`(仅激活记) 上有直接体现。
- Component 不含"每帧主动跑"的逻辑——它是被能力驱动的**被动数据服务**。

这种分离让同一个 Component 能被多个 Capability 共享读写（如 `UHazeMovementComponent` 被移动、跳跃、冲刺、SplineLock 等多个能力共用），也让 Capability 能被自由装卸而不丢状态。

---

## 四、支柱三：有序帧管线（没有中央事件总线）

### 为什么不是"事件驱动"
前面 [Communication.md](./Communication.md) 用数据证明过：全 Core 里"直接获取并调用"(`::Get()` 522 次) 以 5:1 压倒"委托广播"(114 次)；[加载屏链路](./Flow/Fades/) 也证明生命周期是**轮询 `Progress::`/`Game::` 状态**而非订阅事件。

**Haze 刻意回避"中央事件总线分发一切"的模型。** 取而代之的是——

### TickGroup：一套固定的全局执行阶段
每个 Capability 声明自己属于哪个 `TickGroup`，引擎按**固定顺序**逐阶段执行。实测代码里的阶段序列：

```
Input          输入采样
  ↓
BeforeGameplay / BeforeMovement    预处理、意图收集
  ↓
Gameplay       主玩法逻辑（大多数能力在此）
  ↓
InfluenceMovement → ActionMovement → Movement → LastMovement
               移动求解管线（见下）
  ↓
AfterGameplay / AfterPhysics
  ↓
Audio          音频（读取本帧最终状态来定位监听器/发声）
  ↓
LastDemotable / PostWork    收尾
```

同一阶段内再用 `TickGroupOrder` 排序。**这就是"调度核心"**——不是一个发事件的对象，而是一张**确定的时间表**。任何能力想在别的能力之前跑，就把自己放到更早的阶段/序号，而非"监听某事件"。

### 这样做的收益
- **顺序可预测** → 两台机器的执行顺序天然一致（见支柱四）。
- **无隐藏控制流** → 读代码就知道谁先谁后，不必追踪"谁订阅了谁"。
- **阻塞是声明式的** → 能力冲突用 `BlockCapabilities(tag)` 标签互斥解决（131 次使用），A 阻塞 `Movement` 标签，所有移动能力本帧自动失活，A 无需知道它们是谁。

---

## 五、支柱四：确定性联机是"第一因"

前三条支柱不是独立的品味选择，而是**同一个约束推导出的必然结果**。这个约束是：

> **Split Fiction 是双人分屏协作，每端各控一名玩家（Mio / Zoe），两台机器必须对同一世界得出逐帧一致的模拟结果。**

### 从约束到设计的推导链

```
约束：两机必须逐帧一致（确定性）
   │
   ├─→ 需要"谁负责算某对象"的明确归属      →  HasControl() 机制（全库 3466 处分支）
   │      控制端算，远端不算，避免两边各算出分歧
   │
   ├─→ 需要状态变更按相同顺序在两端重演     →  Crumb（带时间戳的有序命令回放）
   │      NetFunction 立即发；CrumbFunction 经 crumb trail 有序传递
   │
   ├─→ 需要执行顺序在两端相同             →  有序 TickGroup 管线（支柱三）
   │      固定时间表 → 两机同序；异步事件总线做不到这点
   │
   └─→ 需要行为可被精确重放/回滚以校验     →  能力/求解器的自包含设计（支柱一、二）
          Resolver 只在自身算、不依赖外部隐藏状态 → 支持 temporal-log 重跑
```

### 代码印记：`HasControl()` 无处不在
全库 **3466 处** `HasControl()` 分支。典型形态（`Examples/Network/Example_SyncedValues.as`）：

```cpp
void Tick(float DeltaTime)
{
    if (HasControl())            // 我是控制端 → 我来算
    {
        SyncedFloat.Value = ComputeNewHeight();   // 算出并写入同步值
    }
    else                         // 我是远端 → 我不算，只读同步过来的值
    {
        ActorLocation.Z = SyncedFloat.Value;      // 自动经 crumb trail 平滑插值
    }
}
```

**"控制端计算、远端读取回放"是贯穿全库的第一性写法。** 理解了这一条，就理解了为什么 Movement 的 Resolver 要自包含、为什么能力有 `NetworkMode = Crumb`、为什么生命周期靠轮询而非事件——它们都是"让两台机器保持一致"的不同侧面。

---

## 六、一帧的完整数据流（端到端）

把四条支柱合起来，一帧的主脉络是：

```
┌─ Input 阶段 ──────────────────────────────────────────────┐
│  采样手柄/键鼠 → 写入输入状态                                │
└───────────────────────────────────────────────────────────┘
        ↓
┌─ Gameplay 阶段 ───────────────────────────────────────────┐
│  各 Capability.ShouldActivate() 判定激活/停用               │
│  激活的能力 TickActive()：读 Component 状态 → 决策           │
│  冲突用 BlockCapabilities(tag) 标签互斥                     │
│  【控制端】才做状态变更，经 CrumbFunction 排入 crumb trail   │
└───────────────────────────────────────────────────────────┘
        ↓
┌─ Movement 管线（InfluenceMovement→…→LastMovement）────────┐
│  能力调 PrepareMove() → 生成 UBaseMovementData（本帧快照）  │
│  → Resolver 自包含求解（扫掠/去穿透）→ ApplyResolve 写回     │
│  【远端】不算，用 ApplyCrumbSyncedGroundMovement 回放        │
└───────────────────────────────────────────────────────────┘
        ↓
┌─ 相机 / 后续阶段 ─────────────────────────────────────────┐
│  相机能力读取本帧最终 transform → UCameraSingleton 分屏混合  │
└───────────────────────────────────────────────────────────┘
        ↓
┌─ Audio 阶段 ──────────────────────────────────────────────┐
│  监听器能力把监听器定位到本帧最终位置 → 发射器发声          │
│  （放在最后：确保读到的是本帧已定稿的世界状态）              │
└───────────────────────────────────────────────────────────┘
        ↓
   两端各自完成本帧；crumb trail 在后续帧把控制端的变更
   在远端同一逻辑时刻回放 → 两机收敛到一致状态
```

关键点：**数据单向沿 tick 阶段流动**（输入→玩法→移动→相机→音频），后面的阶段读取前面阶段定稿的状态。没有"回头广播"，因此没有环、没有顺序歧义——这正是确定性所要求的。

---

## 七、这套哲学如何解释全部文档

回头看前面每一篇文档，都是这四条支柱的局部投影：

| 你读到的文档 | 它其实在讲哪条支柱 |
|---|---|
| [Core/README](./README.md) 的 6 概念 | 支柱一、二（Capability/Component/Singleton/Settings） |
| [Communication.md](./Communication.md)（5 类协作机制） | 支柱三、四（为何直接调用+Crumb 主导、事件仅局部） |
| [SystemArchitecture.md](./SystemArchitecture.md)（单例骨架） | 支柱三（Tick 聚合 + 关卡边界善后） |
| [Movement](./Movement/) 的 Data+Resolver 管线 | 支柱四（Resolver 自包含 → 可重跑校验） |
| [Systems/Network](./Systems/Network/)（Crumb trail） | 支柱四（确定性的底层实现） |
| [各 MiniGames](./MiniGames/) 的 `NetworkMode=Crumb` | 支柱四 |
| [Examples](../Examples.md) 的能力学习路径 | 支柱一、二（写游戏=写能力+装配） |

---

## 八、一句话总结

> **Haze 不是"面向对象地写游戏对象"，而是"把行为拆成数千个可组合的能力单元，用一张有序的帧时间表驱动它们，并让一切状态变更都以控制端计算、Crumb 回放的方式跨机确定一致"。**
>
> 组合优于继承、行为与状态分离、有序帧管线——这三条都是被**"确定性分屏联机"**这一条最高约束推导出来的。抓住"确定性"这个第一因，整套引擎的所有设计取舍都变得可解释。

# Animation — 动画系统

> 路径：`SplitFiction/Animation`（919 文件）
>
> 基于 Hazelight Haze 框架，结合 Capability/Component 玩法系统与数据驱动的 "Feature" 动画系统。

---

## 架构分层

数据驱动动画的关键在于区分以下几层：

### BaseAnimInstances/
绑定到骨骼网格 AnimBP 的具体 `UAnimInstance` 子类；每帧读取游戏状态并暴露 AnimGraph 消费的瞬态属性（风、重力、过场标志等）。按关卡 + Generic 组织。

- 含每关卡子文件夹：`AIGeneric`, `Battlefield`, `Coast`, `Dentist`, `Desert`, `GameShowArena`, `Generic`, `Giants`, `Island`, `Meltdown`, `MoonMarket`, `PigWorld`, `Prison`, `Sanctuary`, `SketchBook`, `Skyline`, `Summit`, `Tundra`

### Features/
数据驱动动画模块，每个是**一对**：
- 一个 `ULocomotionFeatureBase` 子类 —— 持有标签化动画资产/混合空间的**纯数据资产**，无逻辑。
- 一个配对的 `UHazeFeatureSubAnimInstance`（FeatureAnimInstance）—— 读取该数据并计算每 Feature 的运行时值。

按领域和关卡分组：
- 通用：`Locomotion`（移动）、`Traversal`（穿越）、`Generic`（通用：Freeze/HitReactions/KnockDown/PickUp/Stumble/Swing 等）、`Tutorial`、`AIGeneric`（AI 通用：AimStrafe/Death/Flee/MeleeCombat/Shooting 等）
- 每关卡：`Battlefield`（悬浮板系列）、`Coast`（滑水/翼装）、`Desert`、`Giants`、`Island`、`Prison`、`Sanctuary`、`Skyline`、`Summit`、`Tundra`、`MoonMarket`、`PigWorld`、`Meltdown`、`Sketchbook`、`Village`、`KiteTown`、`OilRig`、`SolarFlare`、`Spacewalk`、`DentistsNightmare`、`FreakyWeek`、`GameShow`、`Boneyard` 等

### Capabilities/
`UHazeCapability` / `UHazePlayerCapability` 单元，以 `ShouldActivate`/`OnActivated`/`Tick` 生命周期驱动玩法侧动画行为（如看向开关），活动时阻塞其它能力。

### AnimNotifies/
`UAnimNotify` / `UAnimNotifyState` 标记，放置在动画时间轴上，在特定帧触发效果（相机抖动、力反馈、看向 alpha、生成网格），通过访问网格 owner 上的组件实现。

### Components/
`UActorComponent` 服务，提供共享的每 actor 动画工具（足部 IK 射线、坡度对齐、倾斜、网格生成），供 FeatureAnimInstance 与 Capability 消费。含 Giants 子目录。

### Actors/
动画相关 actor。

### Ragdoll/
布娃娃物理：组件（`URagdollComponent`）+ 应用/清除物理布娃娃的 Capability + 门控何时允许布娃娃的 AnimNotify。

### Debug/
动画调试工具。

---

## 数据流

```
Capabilities / Components   →  驱动玩法状态
        ↓
BaseAnimInstances           →  读取游戏状态
FeatureAnimInstances        →  读取 Feature 数据资产
        ↓
        AnimGraph           ←  Feature 数据资产提供实际动画资源
        ↓
AnimNotifies                →  在特定帧触发定时事件
```

1. **Capabilities/Components** 驱动玩法状态。
2. **BaseAnimInstances** 和 **FeatureAnimInstances** 读取该状态喂给 AnimGraph。
3. **Feature 数据资产** 供给实际动画资源。
4. **AnimNotifies** 触发定时事件。

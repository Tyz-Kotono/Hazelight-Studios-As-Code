# Gameplay / Destructible 功能域总览

> 职责：**烘焙顶点动画破坏道具**——把预先烘焙好的破碎顶点动画按帧回放，做出"碎裂"效果的关卡道具。共 **4 文件**。
>
> **前提**：本文承接 [Gameplay 设计哲学](../DesignPhilosophy.md)（能力四件套）。这里不重复哲学。

---

## 一句话定位

> **Destructible 不做物理模拟，也不接伤害系统。** 破碎是一段**预烘焙进纹理**的顶点动画：材质里存 `Position Texture`（每行一帧的顶点位置），运行时由能力按 24fps 递增 `Display Frame` 标量参数来"擦洗"播放。由外部（蓝图/关卡）显式调用触发，与 Combat / 网络均无耦合。

---

## 文件清单（4 文件）

| 文件 | 关键类 | 基类 | 作用 |
| --- | --- | --- | --- |
| `BakedDestructionActor.as` | `ABakedDestructionActor` | `AHazeActor` | 破坏道具本体：完整/破碎两套 Mesh + 材质帧驱动 |
| `BakedDestructionTimeIterateCapability.as` | `UBakedDestructionTimeIterateCapability` | `UHazeCapability` | 按时间推进破碎动画帧 |
| `BakedDestructionEffectHandler.as` | `UBakedDestructionEffectHandler` | `UHazeEffectEventHandler`（Abstract） | 破碎特效/音频事件桥（`OnDestroyObjectTriggered`） |
| `GenericDestructibleSettings.as` | `GenericDestructibleSettings` 命名空间 | —— | `DefaultFrameRate = 24` |

---

## 内部架构

### `ABakedDestructionActor`

组件：`OriginalComp`（完整外观）+ `BrokenComp`（破碎外观，开局 `SetHiddenInGame(true)`），均为 `UHazePropComponent`；`CapabilityComp` 默认加 `BakedDestructionTimeIterateCapability`。状态：`TArray<UMaterialInstanceDynamic> DynamicMaterials`、`MaxDisplayFrame`、`bRunTimeBased`、`SpeedMultiplier`。

**初始化**（`BeginPlay`）：给 `BrokenComp` 每个材质建动态实例，设 `n"Auto Playback"=0`（关掉材质自播、交给 AS 控制）、`n"Display Frame"=1.0`，并读 `n"Position Texture"` 的高度作为 `MaxDisplayFrame`（纹理行数=动画长度）。

**触发（两条入口，都露 `BrokenComp` 藏 `OriginalComp`）**：
- `StartDestructible(SpeedMultiplier)`：置 `bRunTimeBased=true`，交给能力做时间驱动回放。
- `StartCustomDestructible()`：走蓝图 `BP_ActivateDestruction()`，并触发 `UBakedDestructionEffectHandler::Trigger_OnDestroyObjectTriggered`（携带 `WorldLocation`，"主要给音频"）。

### 帧推进能力

`UBakedDestructionTimeIterateCapability`（Tick 组 `Gameplay`，order 100）：`ShouldActivate` 仅当 `bRunTimeBased`；`TickActive`：

```
FrameCount += DefaultFrameRate(24) * SpeedMultiplier * DeltaTime;
FrameCount = Clamp(FrameCount, 0, MaxDisplayFrame);
SetDynamicMaterialDisplayFrameTimeBased(FloorToInt(FrameCount));  // 写所有材质的 Display Frame
```

`FrameCount >= MaxDisplayFrame` 时 `ShouldDeactivate`。此外提供手动 `SetDynamicMaterialDisplayFrame(All)` 供蓝图逐帧控制。

---

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 依赖 → | 材质系统 | `UMaterialInstanceDynamic` + `n"Auto Playback"`/`n"Display Frame"`/`n"Position Texture"` | 破碎数据烘焙进纹理，材质参数驱动 |
| 依赖 → | Core 能力 | `UHazeCapability` + `UHazeCapabilityComponent` | 时间驱动回放 |
| 被使用 ← | 关卡蓝图 | `StartDestructible` / `StartCustomDestructible` / `BP_ActivateDestruction` / 继承 `UBakedDestructionEffectHandler` | Gameplay 树内无 .as 引用，纯外部触发 |
| 无关 ✗ | Combat / 伤害 | —— | 无 `Damage::`/`TakeDamage`/`EDamageType`，不接伤害系统 |
| 无关 ✗ | 网络 | —— | 无 Crumb/复制代码，同步依赖外部触发时机 |

---

## 关键文件

- `Gameplay/Destructible/BakedDestructionActor.as` —— 破坏道具本体，材质帧驱动 + 两条触发入口。
- `Gameplay/Destructible/BakedDestructionTimeIterateCapability.as` —— 24fps 时间驱动帧推进。
- `Gameplay/Destructible/BakedDestructionEffectHandler.as` —— 破碎特效/音频事件桥。

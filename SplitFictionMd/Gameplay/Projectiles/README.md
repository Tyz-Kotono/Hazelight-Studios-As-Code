# Gameplay / Projectiles 功能域总览

> 职责：**抛射物近距检测**——让关卡里的目标（尤其是 AI 闪避者）知道"有投射物靠近了"。共 **3 文件**（全在子目录 `ProjectileProximityDetection`）。
>
> **前提**：本文承接 [Gameplay 设计哲学](../DesignPhilosophy.md)。这里不重复哲学。

---

## 一句话定位

> **这不是一个投射物系统，而是一个"投射物邻近事件总线"。** 每个玩家有一个管理器（Manager）登记其发射的活跃投射物；关心此事的目标挂检测器（Detector）登记形状区域；管理器每帧暴力两两做点-在-形状内测试，命中即广播事件。纯几何、纯本地、无物理重叠、无网络。

---

## 文件清单（3 文件）

| 文件 | 关键类 | 基类 | 作用 |
| --- | --- | --- | --- |
| `ProjectileProximityDetectorComponentBase.as` | `UProjectileProximityDetectorComponentBase` | `USceneComponent`（abstract） | 定义虚函数 `CheckProximity(AActor)` |
| `ProjectileProximityDetectorComponent.as` | `UProjectileProximityDetectorComponent` | 上者子类 | 具体检测器：形状 + `OnProximity` 事件 |
| `ProjectileProximityManagerComponent.as` | `UProjectileProximityManagerComponent` | `UActorComponent` | 每玩家的登记中枢 + 匹配循环 |

---

## 内部架构

### 检测器 `UProjectileProximityDetectorComponent`

- `FHazeShapeSettings DetectionShape`（默认球，半径 100）——邻近判定区域。
- `EHazeSelectPlayer DetectPlayerProjectiles = Both`——监听哪名玩家的投射物流。
- `ProjectileProximityTime`——最近一次命中时间戳（供消费方轮询）。
- 自身 Tick 关闭；`BeginPlay`→`Register()` 遍历 `Game::Players`，对匹配玩家 `UProjectileProximityManagerComponent::GetOrCreate(Player).RegisterProximityDetector(this)`。
- 检测核心是**点-在-形状内测试**（非物理重叠）：

```
void CheckProximity(AActor Projectile) override {
    if (DetectionShape.IsPointInside(WorldTransform, Projectile.ActorLocation))
        OnProjectileProximity(Projectile);  // 记录时间 + OnProximity.Broadcast(Projectile)
}
```

### 管理器 `UProjectileProximityManagerComponent`

每玩家一个（`GetOrCreate(Player)`）。维护 `TArray<AActor> Projectiles` 与 `TArray<Detector> Detectors`，四个登记方法用 `AddUnique`/`RemoveSwap`。只有两表都非空才开 Tick，匹配引擎即暴力双重循环：

```
for (Projectile : Projectiles)
    for (Detector : Detectors)
        Detector.CheckProximity(Projectile);   // 注释：Optimize as needed!
```

**整体流程**：发射能力 `GetOrCreate` 玩家管理器并传给投射物 → 投射物发射时 `RegisterProjectile`、销毁时 `UnregisterProjectile` → 检测器（目标身上）登记 → 每帧两两测试 → 命中广播 `OnProximity` + 打时间戳。

---

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 依赖 → | Core `FHazeShapeSettings` | `IsPointInside(WorldTransform, Loc)` | 几何邻近判定 |
| 依赖 → | Core `Game::Players`/`EHazeSelectPlayer` | 按玩家挂管理器 | 检测的是"某玩家发射的投射物" |
| 被使用 ← 发射侧 | `LevelSpecific/Desert/SandHand/…`、`Sanctuary/Dark|LightProjectile`、`Summit/…IceBow` 箭矢/`WindJavelin` | 发射能力 `GetOrCreate(Player)` → 投射物 `RegisterProjectile` | 均为玩家武器/道具 |
| 被使用 ← 检测侧 | `LevelSpecific/Island/AI/IslandDodger`、`Sanctuary/AI/Dodger`（`SphereRadius=600`） | `OnProximity.AddUFunction` + 轮询 `ProjectileProximityTime` 决定闪避 | **AI 用它来躲玩家投射物** |
| 无关 ✗ | `Gameplay/AI/Basic` 通用投射物 | —— | 通用 AI 投射物走独立的 `BasicAINetworkedProjectileLauncher`，**不接**本系统 |
| 无关 ✗ | 网络 | —— | 无 Crumb/HasControl，双端各自本地确定性运行 |

> Gameplay 树内无引用；全部真实用例在 `LevelSpecific/`。注意 `IceBowShootCapability`/`IceBowShootWindCapability` 两个变体的 `GetOrCreate` 被注释掉，当前传空管理器。

---

## 关键文件

- `Gameplay/Projectiles/ProjectileProximityDetection/ProjectileProximityDetectorComponent.as` —— 检测器：形状 + `OnProximity` 事件 + 时间戳。
- `Gameplay/Projectiles/ProjectileProximityDetection/ProjectileProximityManagerComponent.as` —— 每玩家登记中枢与两两匹配循环。
- `Gameplay/Projectiles/ProjectileProximityDetection/ProjectileProximityDetectorComponentBase.as` —— 抽象基类（`CheckProximity` 虚接口）。

# Gameplay / Pickups 功能域总览

> 职责：可搬运物体的**拾取 / 携带 / 放下**——走近箱子捡起、扛在身上移动、就地放下或塞进指定插槽。共 **13 文件**。
>
> **前提**：本文承接 [Gameplay 设计哲学](../DesignPhilosophy.md)（能力四件套、组件容器范式）与 Core 交互框架。这里不重复哲学，只讲 Pickups 自身的装配与协同边界。

---

## 一句话定位

> **Pickups 不自己做检测与输入，而是完全架在 Core 的 `UInteractionComponent` / `UInteractionCapability` 之上。** 物体挂交互组件+`UPickupComponent`，玩家身上挂 `UPlayerPickupComponent` 作为状态中枢，四个能力靠一组事件（PickUpStarted/PickedUp/PutDownStarted/PutDown）解耦协作。

> ⚠️ **原型状态**：本模块布满 `// Eman TODO`，核心对齐/放下逻辑多处被注释，且 Gameplay 树内**无任何下游使用者**（很可能尚未接入正式关卡）。文档描述其设计意图与骨架。

---

## 目录结构

| 子目录 | 文件数 | 内容 |
| --- | --- | --- |
| `BaseActors` | 2 | `APickupBase`（可捡物体）、`APutdownInteractionBase`（放置插槽） |
| `Capabilities` | 5 | 捡取/放下的桥接与表现能力 |
| `Components` | 3 | `UPickupComponent`（物体侧）、`UPlayerPickupComponent`（玩家侧中枢）、`UPutdownInteractionComponent`（插槽） |
| `Data` | 2 | 事件参数/枚举/设置结构（`PickupContainers.as`）、标签（`PickupTags.as`） |
| `Visualizers` | 1 | 插槽编辑器可视化 |

---

## 内部架构

### 两类 Actor

- **`APickupBase : AHazeActor`** —— 被捡的物体。组合 `Mesh` + `UPickupComponent` + 一个通用 `UInteractionComponent`（`InteractionSheet = Pickup::PickupInteractionSheet`）。默认捡取类型 `EPickupType::Light`。
- **`APutdownInteractionBase : AHazeActor`** —— 放置插槽（如底座）。组合 `Mesh` + `UPutdownInteractionComponent`。

### 三个组件

- **`UPickupComponent : UActorComponent`**（物体侧配置）。`BeginPlay` 找到 owner 上匹配 `PickupInteractionSheet` 的交互组件并接上，注册交互条件 `CanPlayerPickUp`（玩家未持物、插槽允许取出时才可捡）。`GetAttachBoneForType`：Light→`n"Backpack"`，Heavy→`n"YoMamma"`。
- **`UPlayerPickupComponent : UActorComponent`**（玩家侧中枢 + 事件源）。持 `CurrentPickup`、`bCarryingPickup`（`access` 限定仅捡放能力可写），广播 `OnPickUpStarted/OnPickedUp/OnPutDownStarted/OnPutDown`，持有 `ULocomotionFeaturePickUp` 动画特征。
- **`UPutdownInteractionComponent : UInteractionComponent`**（插槽，**直接继承 Core 交互组件**）。含 `PickupCompatibility`(Light/Heavy/Both)、`PickupSocketTransform`（MakeEditWidget），注册条件 `CanPlayerInteract`（玩家持物且兼容才可放）。

### 四个能力

| 能力 | 基类 | 职责 |
| --- | --- | --- |
| `UPickupInteractionCapability` | `UInteractionCapability` | 桥接：交互触发 → `PlayerPickupComponent.PickUp()` → `Disable` 交互防抢 |
| `UPickupCapability` | `UHazeCapability` | 表现：对齐玩家 → 按动画时机 attach 到骨骼 → 广播 `OnPickedUp` |
| `UPickupLocomotionCapability` | `UHazeCapability` | 携带期间的 `AddLocomotionFeature` / `RequestOverrideFeature(n"PickUp")` |
| `UPutdownCapability` | `UHazeCapability` | 自由放下：按 Cancel 就地脱手（`DetachPickupActor`） |
| `UPutdownInteractionCapability` | `UInteractionCapability` | 插槽放下：摆到 `PickupSocketTransform`，广播 `OnPickupPlacedInSocket` |

### 捡 / 搬 / 放数据流

```
走近 APickupBase
  → Core 交互系统识别 UInteractionComponent（PickupInteractionSheet 给玩家挂 UPickupInteractionCapability）
  → 交互条件 CanPlayerPickUp 通过 → 触发
  → UPickupInteractionCapability.OnActivated → PlayerPickupComponent.PickUp() 设 CurrentPickup + Disable 交互
  → CurrentPickup!=null 激活 UPickupCapability（对齐+attach 到 Backpack/YoMamma）与 UPickupLocomotionCapability（切携带动画）
  → 放下两条路径：
     · 自由放下：Cancel → UPutdownCapability → LetGo() 就地
     · 插槽放下：走近 APutdownInteractionBase → UPutdownInteractionCapability → 摆入插槽
  → 广播 OnPutDown → 桥接能力收尾（重新 Enable 交互 / 清携带动画）
```

---

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 依赖 → | Core `UInteractionComponent` | `UPutdownInteractionComponent : UInteractionComponent`；`APickupBase` 组合通用交互组件 | 拾取的检测/交互完全复用 Core |
| 依赖 → | Core `UInteractionCapability` | 两个 `*InteractionCapability` 继承之，重写 `SupportsInteraction`，`Super::OnActivated` | 桥接能力接入交互能力流 |
| 依赖 → | Core `UHazeCapabilitySheet` | `Pickup::PickupInteractionSheet` / `PutdownInteractionSheet` 声明在组件文件末尾 | sheet 把能力注入玩家 |
| 依赖 → | Core `UPlayerInteractionsComponent` | `KickPlayerOutOfInteraction`、`Interaction.Disable/Enable(this)` | 交互控制与多人防抢 |
| 依赖 → | Core 动画 | `ULocomotionFeaturePickUp`、`RequestOverrideFeature` | 携带动画 |
| 被使用 ← | （无） | —— | Gameplay 树内无外部引用；`SkippingStones` 的同名 `PickupCapability` 是命名巧合，与本模块无关 |

> Pickups 内**无 Crumb / 网络同步 / Spline**——多人一致性目前仅靠 `Interaction.Disable(this)` 间接处理，印证其原型性质。

---

## 关键文件

- `Gameplay/Pickups/Components/PlayerPickupComponent.as` —— 玩家侧捡放状态中枢与事件源。
- `Gameplay/Pickups/Components/PickupComponent.as` —— 物体侧配置，接 Core 交互组件、注册捡取条件。
- `Gameplay/Pickups/Components/PutdownInteractionComponent.as` —— 插槽（继承 `UInteractionComponent`）。
- `Gameplay/Pickups/Capabilities/PickupInteractionCapability.as` —— 交互系统 ↔ 捡取系统的桥接能力。
- `Gameplay/Pickups/Capabilities/PickupCapability.as` —— 对齐 + attach 到骨骼的捡起表现。
- `Gameplay/Pickups/Data/PickupContainers.as` —— 事件参数、`EPickupType`/`EPutdownType`/兼容性枚举、`FPickupSettings`。

# LevelSpecific / Skyline / Hotwire（热接线）

> 路径：`LevelSpecific/Skyline/Hotwire/`（9 文件）
>
> 双人协作小游戏：两名玩家各操一支"探针"沿一根缓慢旋转的圆柱表面移动，两探针之间的连线扫过圆柱上的"节点"并将其点亮/熄灭，用于开锁式的热接线谜题。

承接 [Skyline 关卡总览](../README.md)。复用 Gameplay 能力四件套与标签互斥，不重复哲学。

---

## 职责

Hotwire 是一个聚焦式（Mio 默认全屏）的双人黑客小游戏。核心是一个自转的圆柱 Actor，圆柱面上散布节点；两名玩家各控一支探针在柱面上走位，当两探针都"接通"后，二者连线附近的节点被批量切换。逻辑集中在一个编排 Actor + 每玩家的能力集上。

---

## 内部架构

```
ASkylineHotwire (AHazeActor)  — 编排者，单例（TListedActors 查找）
├── RotationPivot（以 RotationSpeed 自转）
├── UHazeCameraComponent           // 聚焦相机
├── UHazeRequestCapabilityOnPlayerComponent  // 向玩家推送能力
├── UHazeCapabilityComponent       // 默认挂 SkylineHotwireConnectCapability
├── Nodes[]  / ConnectedTools[]    // 节点与已接通探针
├── GetProjectedPointOnCylinder()  // 柱面投影数学
├── IsConnected()                  // 两探针均在场
└── ConnectNodes()                 // 切换连线 ConnectionWidth 内的节点

ASkylineHotwireNode (AHazeActor)   // 可切换节点，ConstructionScript 吸附到柱面
ASkylineHotwireTool (AHazeActor)   // 每玩家探针，OnConnect/OnDisconnect 事件
```

玩家侧能力（均为 `UHazePlayerCapability`）：

| 能力 | 职责 |
|---|---|
| `USkylineHotwireUserCapability` | 入口：屏蔽移动/可见/碰撞/GameplayAction 标签，生成本方探针，把探针 `OnConnect/OnDisconnect` 接到 Hotwire 的处理函数，附着探针 |
| `USkylineHotwireUserInputCapability` | 每帧读 `AttributeVectorNames::LeftStickRaw` 驱动探针沿自转柱面移动 |
| `USkylineHotwireUserToolConnectCapability` | 按 `ActionNames::PrimaryLevelAbility` 屏蔽移动标签并 `Tool.Connect()`，松开则 `Disconnect()` |

`USkylineHotwireUserComponent`（`UActorComponent`）持有 Mio/Zoe 探针类并做延迟生成。

### 数据流

```
玩家进入 → UserCapability 激活（找到单例 ASkylineHotwire）→ 屏蔽常规移动 → 生成本方探针
  → UserInputCapability 左摇杆驱动探针沿柱面走位
  → PrimaryLevelAbility → UserToolConnectCapability → Tool.Connect() → OnConnect 广播
  → ASkylineHotwire.HandleToolConnect 收集到 ConnectedTools
  → 两探针都接通（IsConnected）→ ConnectNodes() 切换连线 ConnectionWidth 内所有节点
```

---

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| 推送 | `UHazeRequestCapabilityOnPlayerComponent` | Gameplay | `ASkylineHotwire` 向玩家推送 Hotwire 能力集 |
| 查找 | `UHazeListedActorComponent` / `TListedActors` | Core | 玩家能力经 `TListedActors` 定位单例 Hotwire Actor |
| 屏蔽 | `CapabilityTags::Movement/Visibility/Collision/GameplayAction` | Gameplay | 进入小游戏时屏蔽常规角色能力 |
| 输入 | `AttributeVectorNames::LeftStickRaw` / `ActionNames::PrimaryLevelAbility` | Core | 摇杆走位 + 主键接通 |
| 自包含 | —— | —— | grep `SkylineHotwire` 无外部引用，机制完全独立 |

---

## 关键文件

| 文件 | 作用 |
|---|---|
| `LevelSpecific/Skyline/Hotwire/SkylineHotwire.as` | 编排者：圆柱自转、节点切换、柱面投影数学 |
| `LevelSpecific/Skyline/Hotwire/SkylineHotwireUserCapability.as` | 玩家入口：探针生成、事件接线、标签屏蔽 |
| `LevelSpecific/Skyline/Hotwire/SkylineHotwireUserInputCapability.as` | 摇杆驱动探针沿柱面移动 |
| `LevelSpecific/Skyline/Hotwire/SkylineHotwireUserToolConnectCapability.as` | 接通/断开动作 |
| `LevelSpecific/Skyline/Hotwire/SkylineHotwireTool.as` | 每玩家探针 Actor |
| `LevelSpecific/Skyline/Hotwire/SkylineHotwireNode.as` | 可切换节点（吸附柱面） |

---

## 附：JetpackCombat（喷气背包战斗竞技场，7 文件）

> `LevelSpecific/Skyline/JetpackCombat/` —— 玩家借喷气背包与重力鞭作战的竞技场。规模小，此处表格带过。

玩家用**重力鞭**把悬停的执法者（Enforcer）敌人甩向一块由可破坏"分区"（zone）拼成的巨型广告牌；击毁足够分区即摧毁广告牌。这里定义的是**战斗竞技场**，玩家的喷气背包位移能力本身在别处（`Animation/Features/Island/Jetpack/`）。

| 文件 | 作用 / 基类 |
|---|---|
| `JetpackCombatZoneManager.as` | `AHazeActor` 协调者：聚合所有分区算包围盒、驱动 Boss 血条、供 AI 查询最近完好/空闲分区、达到阈值广播 `OnBillboardDestroyed` |
| `JetpackCombatZone.as` | `AHazeActor` 单个可破坏广告牌单元，`OnExplode`/`OnTelegraphExplosion` 事件 |
| `JetpackCombatZoneGround.as` | `AHazeActor` 地面危害：预警后发射放射状火箭齐射 |
| `JetpackCombatZoneRocket.as` | `AHazeActor` 火箭弹（复用 `UBasicAIProjectileComponent`），命中施加 `FStumble` 击退 |
| `JetpackCombatSign.as` | `AHazeActor` 假物理广告牌结构（`UFauxPhysics*` 组件），`Collapse()` 塌落 |
| `JetpackBillboardHealthBarComponent.as` | `UActorComponent` 包裹 `UBossHealthBarWidget` 的 Boss 血条 |
| `JetpackCombatMioFallTube.as` | `AHazeActor` Mio 下落管（令 Zoe 移动忽略其碰撞） |

**协同**：复用 Gameplay 的 `UBasicAIProjectileComponent`、`UBossHealthBarWidget`、`UFauxPhysics*`（假物理）；广告牌分区作为共享数据模型被 Skyline `AI/Enforcer/Hovering/*` 与 `AI/Enforcer/Weapon/StickyBombLauncher/*` 大量引用（执法者被鞭甩上广告牌、贴弹、死亡）；玩家瞄准经 `UGravityWhipSlingAutoAimComponent`（见 [GravityWeapons](../GravityWeapons/)）。
</content>

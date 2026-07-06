# Effects — 视觉特效库

> 路径：`Effects/`（58 文件，9 个功能子目录）
>
> 运行时**视觉特效**的 actor / 组件 / 效果处理器集合——攻击预警、环境 VFX、Chaos 破坏物理、氛围生物、溶解过渡等。与 [Environment](./Environment.md)（静态场景美术）互补：Effects 偏**动态**，多由玩法或时间驱动。

---

## 一、定位

| 模块 | 性质 |
|---|---|
| **Effects** | 运行时**动态**视觉表现（会动、会触发、会响应玩法） |
| **Environment** | 静态/半静态**场景美术**物件（摆好即生效） |

Effects 大量复用两套 Core/Gameplay 机制：
- **EffectEvent 系统**（`UHazeEffectEventHandler`）——玩法事件触发表现，见 Core/Audio 的 EffectEvent 分析。
- **`VFXStatics`**（`VFX::` / `Niagara::` 命名空间 + mixin）——通用工具：找 Niagara 挂点、立即失活等。

---

## 二、按子目录分类

### 1. BP/（14 文件）—— 蓝图转换特效
从蓝图迁移来的杂项特效 actor，覆盖面最广。多是 `AHazeActor` + Niagara/材质。

| 文件 | 特效 |
|---|---|
| `Geyser.as` | 间歇泉 |
| `Waterfall_Thin.as` / `FlowingWater_Plane.as` | 瀑布 / 流水面 |
| `EnergyBeamCylinder.as` | 能量光束 |
| `Glitch_01.as` / `Whitespace_Crack.as` / `Environment_Whitespace_MeltdownPhaseOne.as` | 故障 / 白空间裂缝（Meltdown 终章美术） |
| `InfluenceSystem.as` / `NiagaraInfluence.as` | Niagara 影响力系统（粒子受外力扰动） |
| `Piranha_Tundra.as` | Tundra 食人鱼 |
| `YarnTrail.as` | 毛线拖尾（MoonMarket 猫） |
| `IcePalaceVertigoSnowVolume.as` | 冰宫眩晕雪体 |
| `ToggleCameraParticles.as` | 相机粒子开关 |
| `CS_StonebeastMask.as` | 石兽遮罩 |

### 2. Environment/（17 文件）—— 环境 VFX【最大子目录】
场景级的动态视觉效果 + 通用 VFX 工具。

| 文件 | 说明 |
|---|---|
| `VFXStatics.as` | **通用 VFX 工具库**：`VFX::` / `Niagara::` 命名空间 + mixin（`FindRelevantAttachActorForNiagara`、`FindRelevantAttachMeshForNiagara`、`DeactivateImmediately`）。被全库 VFX 复用 |
| `DynamicWaterEffect.as` (+`...CutsceneOverride`) | 动态水（角色扰动水面） |
| `Waterfall.as` (+`WaterfallEffectHandler.as`) | 瀑布 + 效果处理器 |
| `FloodFill.as` | 洪水填充 |
| `RainBoxComponent.as` / `PreviewWeatherEffectSubsystem.as` | 降雨盒 / 天气预览子系统 |
| `SpotlightVFX.as` / `DaClubVisualizer.as` | 聚光 / 夜店可视化（Skyline） |
| `PlacedParticles.as` / `KillParticleVolume.as` | 放置粒子 / 粒子清除体 |
| `MeshSplineFollower.as` / `HelixRuntimeSpline.as` | 网格沿样条 / 螺旋样条（复用 Core/Spline） |
| `MovementVFXEventHandler.as` | 移动 VFX 事件处理器（接 EffectEvent） |
| `GlitchLoopingActor.as` / `TrailTester.as` | 故障循环 / 拖尾测试 |

### 3. FX_Chaos/（13 文件）—— Chaos 破坏物理
基于 UE Chaos 场系统（Field System）的物理破坏特效。

- **`BaseFieldSystemActor.as`** —— `ABaseFieldSystemActor : AFieldSystemActor`，所有场系统基类（含 `#if EDITOR` 编辑器图标）。
- **`FieldSystemStatics.as`** —— 场系统静态工具。
- **`HazeGeometryCollectionActor.as`** —— 几何集合 actor（可破碎网格）。
- **Impact/**（7）—— 冲击场：`BulletImpactFieldSystemActor`（子弹冲击）、`AcidImpactFieldSystemActor`（酸）、`RamImpactFieldSystemActor`（撞击）、`EmittingImpactFieldSystemActor` / `EmittingSphereFieldSystemActor`（持续发射）、`BaseImpactFieldSystemActor`（基类）、`ExampleImpactFieldSystemActor`（示例）。
- **Level/**（3）—— 关卡级场：`AnchorFieldSystemActor`（锚定）、`DisableFieldSystemActor`（禁用）、`Gameplay/ChaosLaserCutterActor`（激光切割）。

> 模式：一个 `AFieldSystemActor` 子类 = 一种"力场"，作用于附近的几何集合产生破碎/位移。

### 4. Critters/（7 文件）—— 氛围生物
增添生机的小动物，**带联机同步**（两端要一致）。

- **`BaseCritterActor.as`** —— `ABaseCritterActor : AHazeActor`（Abstract），青蛙/老鼠等的共同基类。含 `LinkedCritter` 镜像联机、`RandomMeshSizeScaler`（带 bias 的随机尺寸）、`COM`（群体质心）。持有 `TArray<UBaseCritterComponent>`。
- 具体：`CritterFrogs` / `CritterRats`（球面约束群）、`CritterFlying`（飞行）、`CritterFleeing`（逃跑）、`CritterSurface`（表面爬行）、`CritterLine`（列队）。

> 是"基类定共性 + 子类定行为"的清晰范例，`LinkedCritter` 主从镜像解决联机一致。

### 5. Attacks/（1）—— 攻击预警
- **`TelegraphDecalComponent.as`** —— `UTelegraphDecalComponent : UDecalComponent`，敌人攻击前在地面显示的**预警贴花**。带 `ETelegraphDecalType`（Fantasy / Scifi 两种风格）、半径、自动显隐。用 `access:EditOnly` 细粒度访问控制。配合 Gameplay/AI 的攻击 telegraphing。

### 6. Dissolve/（2）—— 溶解过渡
- `MaterialTransform.as` / `TestDissolve.as` —— 材质溶解/变换效果（编辑器可 tick 预览）。

### 7. 单文件效果处理器（3）
| 文件 | 说明 |
|---|---|
| `Movement/PlayerShellEffectHandler.as` | 玩家移动的外壳网格效果，继承 `UPlayerCoreMovementEffectHandler`（接 Core/Movement 的效果处理链） |
| `Projectile/VisualEffectProjectile.as` | `AVisualEffectProjectile`，纯视觉（无玩法碰撞）的抛射物 |
| `WardrobeChange/SequencerWardrobeChangeActor.as` (+`WardrobeChangeTest`) | 序列驱动的换装特效 actor（配合 Core/Wardrobe 溶解） |

---

## 三、贯穿的实现模式

| 模式 | 说明 | 体现 |
|---|---|---|
| **EffectHandler 接玩法事件** | `UHazeEffectEventHandler` 派生，玩法触发→表现 | WaterfallEffectHandler、MovementVFXEventHandler、AirVent |
| **基类 + 子类特化** | 基类定共性，子类定具体效果 | BaseCritterActor→各生物、BaseFieldSystemActor→各力场 |
| **命名空间静态工具** | `VFX::` / `Niagara::` mixin | VFXStatics 被全库复用 |
| **数据/风格枚举** | 一套效果两种美术风格 | TelegraphDecal 的 Fantasy/Scifi |
| **联机镜像** | 主从 actor 保证两端一致 | Critter 的 `LinkedCritter` |
| **`#if EDITOR` 辅助** | 编辑器图标/预览 tick | FieldSystem 图标、Dissolve 预览 |

---

## 四、协同关系

| 方向 | 对象 | 说明 |
|---|---|---|
| ← | Gameplay/AI | AI 攻击前调 TelegraphDecal 显示预警 |
| ← | Core/Movement 效果处理链 | PlayerShellEffectHandler 接入移动效果 |
| ← | EffectEvent 系统 | Waterfall/MovementVFX 等 Handler 被玩法事件触发 |
| → | Core/Spline | MeshSplineFollower / HelixRuntimeSpline 沿样条 |
| → | Core/Wardrobe | WardrobeChange 配合溶解 |
| → | 引擎 Chaos / Niagara | FX_Chaos 场系统、各 Niagara 特效 |
| → | `VFXStatics` (`VFX::`) | 找挂点、失活等通用工具 |

---

## 五、小结

Effects 是**动态视觉表现层**：用 EffectHandler 承接玩法事件、用基类/子类组织同类特效、用 `VFXStatics` 提供通用工具。技术上覆盖 Niagara 粒子、Chaos 破坏物理、贴花预警、材质溶解、联机同步的氛围生物。它专注"看起来对",玩法逻辑在 Gameplay，静态美术在 Environment。

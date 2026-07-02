# Core / Audio / Physics

> 职责：把 Unreal 物理材质映射为音频参数（脚步/弹道/碎屑等标签与音效），并在运行时按玩家脚下材质选取对应音效。共 2 个文件。

## 内部架构（关键类、基类、数据流）

### UPhysicalMaterialAudioAsset（PhysicalMaterialAudioAsset.as）
- 基类：`UHazePhysicalMaterialAudioAsset`（引擎层基类，提供 `FootstepData / WeaponProjectileData / AbilityProjectileData / ForceDebrisData / ExplosionDebrisData` 等数据块）。
- 数据资产（DataAsset），把一种物理材质的音频配置集中描述：硬度 `HardnessType`、摩擦 `FrictionType`，以及各类玩法事件能否触发（跳弹 `CanCauseRicochets`、碎屑 `CanCauseDebris`、近场 `CanCauseCloseProximity`）与衰减缩放。
- 核心数据流：`GetMaterialTag(TagType)` 按用途（Footstep / ProjectileBulletImpact / AbilityProjectileDebris / AbilityBlastDebris / ExplosionDebris）从对应数据块取出一个 `FName` 标签——这是“材质 → 音频”的关键句柄。
- 同文件 `namespace AudioPhysMaterial` 提供一层带空指针保护的 `BlueprintPure` 包装函数，供蓝图/外部安全查询上述能力与标签。
- 枚举：`EHazeAudioPhysicalMaterialHardnessType / FrictionType / DebrisType / TagType`。

### UPlayerAudioMaterialComponent（PlayerAudioMaterialComponent.as）
- 基类：`UActorComponent`，每玩家一个，运行时负责“材质标签 → 具体音效事件”的解析。
- 关键数据：`MaterialAssets`（`TMap<FName 标签, UHazeAudioMaterialReferenceAsset>`，标签即来自物理材质的 `FootstepTag`）。引用资产内含按移动类型分桶的 `FootSets / HandSets`（`FHazeMaterialSet`，按 `EFootType` 索引 Left/Right/Release 事件）。
- 覆盖机制：`OverrideMaterial`（全局覆盖）+ 4 槽 `MovementMaterialOverrides`（Slide / 左脚 / 右脚 / 手，按 `EFootType` 索引），均带 `FInstigator` 有效性校验，发起者失效时自动清除。
- 数据流主链：`GetMaterialEvent(标签, 移动类型, 脚类型, ...)` → 先 `CheckMovementOverrideMaterialTag` 应用覆盖 → `MaterialAssets.Find` 取引用资产（缺失则回退 `GlobalAudioDataAsset.DefaultAudioPhysMat`）→ 在 `FootSets/HandSets` 内按移动类型与脚类型取出 `UHazeAudioEvent` 返回。

整体数据流：物理材质 `.AudioAsset` → `UPhysicalMaterialAudioAsset.FootstepData.FootstepTag` → `UPlayerAudioMaterialComponent.GetMaterialEvent` → `UHazeAudioEvent`（最终播放）。

## 协同关系

| 方向 | 对象 | 机制 | 说明（基于实证） |
| --- | --- | --- | --- |
| 入 | `Audio::Materials::GetGroundAudioPhysMat`（AudioStatics.as） | 直接调用 Type::Get | 通过 `UHazeMovementComponent::Get` 取地面接触做 trace，将 `PhysMat.AudioAsset` Cast 为 `UPhysicalMaterialAudioAsset` 返回（AudioStatics.as:59-83）。 |
| 入 | `UPlayerFootstepTraceCapability`（Gameplay/Movement/Audio/Trace） | 直接调用 Type::Get | `MaterialComp = UPlayerAudioMaterialComponent::Get(Player)`，trace 命中后取 `PhysMat.AudioAsset` 转 `UPhysicalMaterialAudioAsset` 填入 `FootstepParams`（行 102、264、276）。 |
| 出 | `UHazeAudioEvent` / 脚步音效 | 直接调用 | `MaterialComp.GetMaterialEvent(AudioPhysMat.FootstepData.FootstepTag, MovementState, ...)` 解析出脚步与 Scuff 事件（行 276、285）。 |
| 入 | `UPlayerVariantComponent`（Core/Players） | 可组合设置 ApplySettings | `AudioMaterialComponent.SetMaterials(MaterialAssets)` 注入按角色变体配置的材质引用表（PlayerVariantComponent.as:215）。 |
| 入 | `SketchbookAudioLevelScriptActor` / `PlayerFootstep(Slide)SplitTraversalAudioCapability` | 可组合设置 ApplySettings + 委托 Instigator | 关卡侧调用 `SetAllMaterialOverride` / `SetMovementMaterialOverride(Foot, Mat, FInstigator(this))` 临时覆盖脚下材质，由 `FInstigator` 标识发起者并自动失效清理。 |
| 出 | 玩法系统（弹道/碎屑/跳弹） | 直接调用（蓝图包装） | `AudioPhysMaterial::CanCause*` / `GetMaterialTag` / `GetProjectileImpactAttenuationScale` 供武器、爆炸、能力 SoundDef 查询材质能力与标签。 |

## 关键文件（逐文件一句话）

- `PhysicalMaterialAudioAsset.as`：定义 `UPhysicalMaterialAudioAsset` 数据资产及配套枚举/蓝图包装，把单种物理材质的硬度、摩擦与各类玩法音频标签集中描述。
- `PlayerAudioMaterialComponent.as`：每玩家 `UPlayerAudioMaterialComponent`，按材质标签查 `GetMaterialEvent` 解析脚步/手部音效，并支持带 Instigator 的全局与分脚材质覆盖。

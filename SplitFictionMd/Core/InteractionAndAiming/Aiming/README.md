# Core / InteractionAndAiming / Aiming

> 职责：基于 Targetable 地基实现玩家瞄准与自动瞄准（auto-aim）——把瞄准射线推入 targetables 系统、读回 `n"AutoAim"` 分类的 primary 目标来弯折射击方向，并管理准星(Crosshair)与 2D/3D 多种输入模式。共 14 个文件（含 Capabilities 子目录 6 个）。

## 内部架构

### 关键类与基类
- **`UPlayerAimingComponent : UActorComponent`**（PlayerAimingComponent.as）：瞄准核心，**Targetable 的独立消费者**（不继承它）。
  - `StartAiming/StopAiming(Instigator, FAimingSettings)`：按 instigator 管理多份 `FActiveAiming`，按需创建准星、应用瞄准灵敏度。
  - `UpdateAiming()`（每帧）：对每份 aiming 调 `CalculateAim` → 写 `TargetablesComp.CurrentAimingRay = GetPlayerAimingRay()` → `TargetablesComp.UpdateTargeting()`。
  - `CalculateAim`：向 targetables `OverrideTargetableAimRay(n"AutoAim", Ray)`，再读 `GetPrimaryTargetForCategory(n"AutoAim")`；若 primary 是 `UAutoAimTargetComponent` 则用 `GetAutoAimTargetPointForRay` 求弯折落点，得出 `AimDirection`。
  - 覆盖入口：`ApplyAimingRayOverride`（输入能力写入）/`ApplyAimingTargetOverride`（强制目标）——均 `TInstigated<>` 按优先级。
  - 2D 约束：`ApplyAiming2DPlane/CameraPlane/SplineConstraint`，`Get2DConstraintPlaneNormal`/`Get2DAimingCenter` 供输入能力使用。
- **`UAutoAimTargetComponent : UTargetableComponent`**（AutoAimTargetComponent.as）：可选的自动瞄准目标，**子类化 Targetable**，`default TargetableCategory = n"AutoAim"`。
  - `CheckTargetable`：距离/角度预剔除 → `GetAutoAimTargetPointForRay`（沿 `TargetShape` 求最近点）→ 计算射线弯折角 `AngularBend`，超 `CalculateAutoAimMaxAngle` 则不可选 → 按弯折角与距离评分 → `RequireAimToPointNotOccluded` 遮挡检测。
- **`UPlayerAimingSettings : UHazeComposableSettings`**（PlayerAimingSettings.as）：可组合设置（屏幕偏移、2D interp 速度、准星 lerp 时长等）。

### 数据结构（AimingData.as）
`FAimingSettings`（是否准星/auto-aim/灵敏度/2D 准星设置）、`FAimingResult`（Ray/AimOrigin/AimDirection/AutoAimTarget/Point）、`FAimingRay`（`EAimingMode` Free3D/Directional2D/Cursor2D + Origin/Direction/约束面）、`FAimingConstraint2D`（Plane/Spline/CameraPlane）。

### Capabilities 子目录
- **`UPlayerAimingUpdateCapability`**：每帧（TickGroupOrder=-100，最早）调 `AimComp.UpdateAiming()`，是瞄准系统的心跳。
- 2D 输入采集（各写 `ApplyAimingRayOverride`）：`UAimingGamepadInputCapability2D`（右摇杆方向）、`UAimingMouseInputCapability2D`（鼠标光标→平面交点）、`UAimingMouseDirectionalOnlyInputCapability2D`（仅方向、屏蔽光标）。
- `UAiming2DResetToFacingCapability`：无输入时把 2D 瞄准重置回玩家朝向。
- `UAiming2DSyncDirectionCapability`：Crumb 网络同步 2D 瞄准方向到远端。

### 准星渲染
`UCrosshairContainer`（全屏容器，管理 2D/3D 准星位置、auto-aim lerp、方向箭头、软件光标）+ `UCrosshairWidget`/`UCrosshairWithAutoAimWidget`。

### 数据流
输入能力写 OverrideAimingRay → `UpdateAiming` 取最终 ray 推入 targetables → targetables 用该 ray 评分 `n"AutoAim"` 目标选 primary → 读回 primary 弯折 `AimDirection` → 准星跟随、射击系统使用 `GetAimingTarget`。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 继承自 | `UTargetableComponent` | 子类化 | `UAutoAimTargetComponent : UTargetableComponent`，`default TargetableCategory = n"AutoAim"`（AutoAimTargetComponent.as:12-14） |
| 调用入↔出 | `UPlayerTargetablesComponent` | 直接调用 | 写 `CurrentAimingRay`、`OverrideTargetableAimRay(n"AutoAim",Ray)`、读 `GetPrimaryTargetForCategory(n"AutoAim")`、调 `UpdateTargeting`（PlayerAimingComponent.as:345-393） |
| 调用出 | `UAutoAimTargetComponent` | 直接调用 | primary 若为该类型则 `GetAutoAimTargetPointForRay(Ray)` 求落点（PlayerAimingComponent.as:401-403） |
| 调用出 | 遮挡 trace | 直接调用 | `CheckTargetable` 内 `Targetable::RequireAimToPointNotOccluded`（AutoAimTargetComponent.as:223） |
| 心跳 | `UPlayerAimingComponent` | 直接调用 | `UPlayerAimingUpdateCapability.TickActive → AimComp.UpdateAiming()`（PlayerAimingUpdateCapability.as:39） |
| 输入→组件 | `ApplyAimingRayOverride` | 直接调用 + 优先级 | 四个输入能力将 `FAimingRay` 经 `TInstigated` 写入组件（各 InputCapability2D） |
| 网络同步 | 远端玩家 | Crumb | `UAiming2DSyncDirectionCapability` 用 `UHazeCrumbSyncedRotatorComponent` + `CrumbFunction` 同步方向（Aiming2DSyncDirectionCapability.as:9、87） |
| 标签 | 鼠标光标能力 | 标签阻塞 | `UAimingMouseDirectionalOnlyInputCapability2D.BlockCapabilities(n"MouseCursorInput")`（该文件:50） |
| 设置 | 玩家 | 可组合设置 | `Player.ApplyDefaultSettings(DefaultAimingSettings)`、`UPlayerAimingSettings::GetSettings`（PlayerAimingComponent.as:316） |
| 双人 | Mio / Zoe | 直接判断 | 鼠标瞄准互斥检查 `OtherPlayerAimComp.bIsUsingMouseAiming`；准星按 `Player.IsMio()` 取色（MouseInputCapability2D.as:36） |

## 关键文件

| 文件 | 说明 |
|---|---|
| PlayerAimingComponent.as | 瞄准核心组件：管理 aiming、推/拉 targetables aim ray、2D 约束 |
| AimingData.as | `FAimingSettings`/`FAimingResult`/`FAimingRay`/`FAimingConstraint2D` 等数据 |
| AutoAimTargetComponent.as | 自动瞄准目标 `UAutoAimTargetComponent`（继承 Targetable，弯折评分） |
| PlayerAimingSettings.as | `UPlayerAimingSettings`（可组合设置）+ 2D/准星相关 CVar |
| PlayerAimingStatics.as | 2D 约束 apply/clear 的便捷 mixin（含 spline 解析） |
| CrosshairContainer.as | 准星全屏容器：位置/lerp/方向箭头/软件光标/双人分屏 |
| CrosshairWidget.as | 准星 widget 基类及带 auto-aim 指示的变体 |
| AutoAimTargetVisualizer.as | 编辑器可视化 auto-aim 角度/形状/距离（仅 EDITOR） |
| Capabilities/PlayerAimingUpdateCapability.as | 每帧心跳，调用 `UpdateAiming` |
| Capabilities/AimingGamepadInputCapability2D.as | 手柄右摇杆 2D 方向输入 |
| Capabilities/AimingMouseInputCapability2D.as | 鼠标光标 2D 瞄准（平面交点） |
| Capabilities/AimingMouseDirectionalOnlyInputCapability2D.as | 仅方向的鼠标 2D 瞄准（屏蔽光标） |
| Capabilities/Aiming2DResetToFacingCapability.as | 无输入时重置 2D 瞄准回玩家朝向 |
| Capabilities/Aiming2DSyncDirectionCapability.as | Crumb 同步 2D 瞄准方向到远端 |

# Core / Rendering / Wardrobe

> 职责：换装时的溶解过渡——以球形区域驱动 Glitch（白空间）着色器在「旧装 → 新装」之间溶解，并在接缝处生成 Niagara 特效。共 2 个文件。

## 内部架构

- **`UMeshDissolveComponent : UHazeMeshDissolveManagerComponent`**（基类提供 `StartMesh`/`EndMesh`/`Alpha`/`DissolveType`/`DissolveDirection`）：
  - `OnStartDissolve`：可选在新装插槽（Hips/Base）生成循环 Niagara 接缝特效 `Seam_VFX`，并把旧网格设为始终更新骨骼姿态。
  - `Tick`：按 `EHazeDissolveType`：
    - `Sphere`：`UpdateAutomaticSphereDissolve` 根据插槽位置 + `DissolveDirection` 自动推进溶解球半径（随 `Alpha`）。
    - `ManualSphere`：`UpdateManualSphereDissolve` 读 `GetManualSphereComponent()`（即 `AWardrobeDissolveSphere`）的球心/半径。
  - `UpdateSphereDissolve` → `UpdateWhiteSpaceBlend`：把球心/半径/翻转/边宽写进新旧网格材质的 `Glitch_*` 参数（`Glitch_Center`/`Glitch_Radius`/`Glitch_Flip`/`HazeToggle_Glitch_Enabled`），并同步给接缝 Niagara（`DissolveLocation`/`DissolveRadius`）。
  - `OnEndDissolve`：停 Niagara、关 `HazeToggle_Glitch_Enabled`。
- **`AWardrobeDissolveSphere : AHazeActor`**：带无碰撞 `UHazeSphereCollisionComponent` 的定位球，作为 ManualSphere 模式的溶解区域驱动体。

数据流：`Alpha / 手动球 → 溶解球心半径 → Mesh 材质 Glitch_* 参数 + Niagara 接缝 → 渲染溶解`。

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 调用 | Niagara | 直接调用 | `Niagara::SpawnLoopingNiagaraSystemAttached` + `SetNiagaraVariable*` 驱动接缝特效 |
| 调用 | 角色网格 | 直接调用 | `Mesh.SetVector/ScalarParameterValueOnMaterials(n"Glitch_*")` 写 Glitch 着色器参数 |
| 引用 | 手动溶解球 | 直接调用 | `GetManualSphereComponent()` 读取 `AWardrobeDissolveSphere` 的球心/半径 |

## 关键文件

- **MeshDissolveComponent.as**：`UMeshDissolveComponent`，自动/手动球形溶解，写网格 Glitch 参数 + 生成 Niagara 接缝特效。
- **WardrobeDissolveSphere.as**：`AWardrobeDissolveSphere`，手动溶解模式的定位球（无碰撞球组件）。

# Core / Rendering / PaintablePlane

> 职责：可绘制的平面/体积表面——同时维护 CPU 端颜色数组与 GPU 端渲染目标，供 gameplay 读写「在世界某处涂了什么」。共 1 个文件。

## 内部架构

- **`APaintablePlane : AActor`**（假定轴对齐以省性能）：
  - 类型 `EPaintablePlaneType`：`Plane`（2D，RGBA16f 双缓冲 RenderTarget）或 `Volume`（3D `UTextureRenderTargetVolume`，目前体积绘制被注释停用）。
  - 双份数据：
    - **CPU**：`TArray<FLinearColor> CPUSideData`，按分辨率 `ResolutionX/Y/Z` 线性索引；`GetColorAtPoint`/`SetColorAtPoint`/`SetColorInBox` 直接读写。
    - **GPU**：`PaintTexture2DTarget1/2` 乒乓缓冲 + 输出贴图 `PaintTexture2DMaterialOut`，`SetColorInBox` 时用 `DrawBox2DMaterial` 绘制再 `CopyTexture2DMaterial` 拷到输出。
  - 坐标变换族：世界 ↔ 纹理 ↔ 数组索引（`WorldLocationToTextureLocation` 等）。
  - `ApplyVolumeDataToMaterial/Mesh*`：把输出贴图与平面变换参数注入场景网格材质，让其采样绘制结果。
  - `Initialize`（ConstructionScript + BeginPlay 调用）按类型创建缓冲；CPU 像素数 >10000 时打印性能警告（注释实测 12³ 约 20µs）。

数据流：`SetColorInBox → CPU 数组 + GPU 乒乓 RenderTarget → ApplyVolumeDataToMaterial → 场景材质采样`。

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 调用 | 引擎渲染 | 直接调用 | `Rendering::CreateRenderTarget2D/Volume`、`DrawMaterialToRenderTarget`、`ClearRenderTarget2D` |
| 提供 | 场景网格 | 直接调用 | `ApplyVolumeDataToMesh(es)` 向 `AStaticMeshActor` 材质注入绘制贴图与变换参数 |

## 关键文件

- **PaintablePlane.as**：`APaintablePlane`，CPU 数组 + GPU 乒乓 RenderTarget 的可绘制平面/体积，含坐标变换与材质注入。

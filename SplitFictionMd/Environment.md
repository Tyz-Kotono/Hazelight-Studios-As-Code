# Environment — 环境美术 Actor 库

> 路径：`Environment/`（28 文件，扁平目录）
>
> 一批**可放置的环境美术 actor**——光照、动态装饰（布料/缆绳/秋千）、海浪、遮挡、可破坏物等。它们是关卡设计师从内容浏览器拖进场景的"积木"，负责氛围与细节表现，不含玩法逻辑。

---

## 一、定位

Environment 与 [Effects](./Effects.md) 的区别：
- **Effects** = 运行时**动态特效**（攻击预警、生物、破坏物理、溶解），多由玩法触发。
- **Environment** = 静态/半静态的**场景美术物件**，摆好即生效，服务于"这个世界长什么样"。

所有 actor 都继承 `AHazeActor`（少数继承 `AHazePrefabActor`），普遍用 `#if EDITOR` 编辑器组件辅助摆放、`UDisableComponent`（Core）做远距性能禁用。

---

## 二、按类型分类

### 1. 光照类（8 个）—— 最大类别
可放置的各类光源 actor，多是 `AHazeActor` + `UStaticMeshComponent` 派生的发光组件。

| 文件 | 说明 |
|---|---|
| `DistantLight.as` | `ADistantLight` + `UDistantLightComponent`：远景光（模拟远处光源，不实际照明） |
| `LensFlare.as` / `LensFlareComponent.as` | `ALensFlare` + `ULensFlareComponent` + `UDataAssetLensFlare`：镜头光晕（数据资产驱动） |
| `Godray.as` | `AGodray` + `UGodrayComponent` + `UDataAssetGodray`：体积光/丁达尔效应 |
| `ToggleableLights.as` | 可开关的灯组 |
| `Police_Car_Lights.as` / `Police_Car_Uplights.as` | 警车灯（科幻关卡专用美术） |
| `Light_Scifi_Generic_01_Large_A.as` | 科幻通用大灯 |
| `Oilrig_Trench_Light_01.as` | OilRig 关卡沟槽灯 |
| `SurveillanceTower_01.as` | 监控塔（含灯） |

> 光照类普遍走"`AHazeActor` + 发光 `UStaticMeshComponent` + 可选 `UDataAsset` 参数"的组合。数据资产化让美术复用一套参数。

### 2. 动态装饰类（5 个）—— 有物理感的软体
用数学模拟出"会动"的装饰物，是 Environment 里技术含量最高的部分。

| 文件 | 关键类 / 机制 |
|---|---|
| `EnvironmentClothSim.as` + `EnvironmentClothSimSubsystem.as` | **GPU 布料模拟**：`UEnvironmentClothSimComponent`（继承 `UStaticMeshComponent`）注册进 `UEnvironmentClothSimSubsystem`（`UScriptWorldSubsystem`）。子系统把所有布料的 pin mask 打包进一张大 render target（`AddCloth` 时 `DrawTexture` 到网格对应格），GPU 统一模拟。是**注册表 + 批量 GPU**模式 |
| `EnvironmentCable.as` | `AEnvironmentCable` + `UEnvironmentCableComponent`（继承 `UHazeTEMPCableComponent`）：垂坠缆绳 |
| `EnvironmentChain.as` | `AEnvironmentChain`：链条 |
| `EnvironmentSwing.as` | `AEnvironmentSwing`：可摆动物 |
| `AmbientMovement.as` | `AAmbientMovement`（继承 `AHazePrefabActor`）：给静态网格加环境摆动（风吹感），含 `#if EDITOR` 编辑器组件 |

#### 驱动它们的力：`EnvironmentForceSingleton`
`EnvironmentForceSingleton.as` 是一个 `UHazeSingleton`：Cable/Chain/Swing 在 BeginPlay 时**注册**进来，单例每帧 `Tick` 把统一的力（风等）传播给所有已注册装饰物。
> 注：文件注释里作者留了自我怀疑——`@TODO: 不太确定单例是不是对的做法……更省的方案是写进一个公共源让大家读`。是了解设计取舍的好例子。

### 3. 水体类（2 个）
| 文件 | 说明 |
|---|---|
| `OceanWaveExample.as` | `AOceanWaveExample`：海浪示例 |
| `OceanWavePaint.as` | `AOceanWavePaint` + `UOceanWavePaintComponent` + `UOceanWavePaintTimeLoggerComponent`（时序日志可 scrub）：绘制式海浪 |

### 4. 渲染优化类（4 个）
| 文件 | 说明 |
|---|---|
| `Occluder.as` | `AOccluder`：遮挡体（剔除优化） |
| `LandscapeVirtualTextureController.as` | 地形虚拟纹理控制 |
| `ParallaxRoomCapture.as` | 视差房间（假室内景，用捕获贴图省几何） |
| `TangentBaker.as` | 切线烘焙工具 |

### 5. 可破坏 / 交互类（3 个）
| 文件 | 说明 |
|---|---|
| `Breakable.as` | `ABreakableActor` + `UBreakableComponent` + `UDataAssetBreakable`：可破坏物。数据资产含破碎力/散射、Niagara 破碎粒子、破碎音效（`FSoundDefReference`，接 Core/Audio）、破碎后力 |
| `AirVent.as` | `AAirVent` + `UAirVentEffectEventHandler`（`UHazeEffectEventHandler`）：通风口，用 EffectEvent 系统触发气流表现 |
| `HazeSphere.as` | `AHazeSphere` + `UHazeSphereComponent` + `UHazeSphereEditorSubsystem`：调试/占位球 |

### 6. 关卡专属美术道具（3 个）
`HologramWall_01.as`（全息墙）、`Pipes_Modular_02_Group_A.as`（模块化管道组）、`SurveillanceTower_01.as`（监控塔）——特定关卡的成品美术 actor。

---

## 三、贯穿的实现模式

| 模式 | 说明 | 体现 |
|---|---|---|
| **Actor + Component + DataAsset 三件套** | actor 放置、组件渲染、数据资产存可复用参数 | Godray/LensFlare/Breakable |
| **注册表子系统/单例** | 同类物件注册进一个中枢，批量驱动 | ClothSimSubsystem（GPU 批量）、ForceSingleton（力传播） |
| **复用 Core** | `UDisableComponent` 远距禁用、`FSoundDefReference` 音效、`UHazeEffectEventHandler` 表现事件、时序日志 | 几乎所有 actor |
| **`#if EDITOR` 编辑器辅助** | 摆放时的可视化/编辑组件，打包剔除 | AmbientMovement、HazeSphere |

---

## 四、协同关系

| 方向 | 对象 | 说明 |
|---|---|---|
| → | Core `UDisableComponent` | 环境 actor 普遍挂它做远距自动禁用（省性能） |
| → | Core/Audio `FSoundDefReference` / `UHazeAudioEvent` | Breakable 等的音效 |
| → | Core `UHazeEffectEventHandler` | AirVent 用 EffectEvent 触发气流表现 |
| → | 引擎 `Rendering::` / RenderTarget | ClothSim 的 GPU pin mask 打包、OceanWavePaint |
| → | 引擎 `UScriptWorldSubsystem` / `UHazeSingleton` | ClothSim 子系统、Force 单例 |
| ← | 关卡场景 | 由设计师从内容浏览器拖入摆放，无代码调用方 |

---

## 五、小结

Environment 是**"世界的皮肤"**：一组自包含的美术 actor，把光照、软体装饰、水体、遮挡、可破坏物做成即拖即用的积木。技术亮点是**布料的 GPU 批量模拟**（子系统注册表 + 大 RT 打包）与**环境力的单例传播**。它不发明玩法，而是复用 Core 的禁用/音频/事件/时序日志设施，专注于表现层。

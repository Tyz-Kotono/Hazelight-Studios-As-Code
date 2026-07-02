# Split Fiction — AS 脚本架构文档

> 本文档描述 **Split Fiction** 的 AngelScript 脚本架构。源码路径：`C:\Users\tangjianpeng\Desktop\AS\SplitFiction`
>
> 共 **15,814** 个 `.as` 文件。这是 Hazelight 基于 Unreal Engine + AngelScript 的 **Haze 引擎** 框架。

---

## 核心设计模式

整套代码贯穿统一的模式：**Capability（能力）+ Component（组件）+ Data（数据资产）+ Widget（UI）+ Statics（静态工具）**。

- **Capability（能力）** —— 继承自 `UHazeCapability` / `UHazePlayerCapability`，是行为驱动的基本单元，拥有生命周期：
  - `Setup()` — 初始化
  - `ShouldActivate()` / `ShouldDeactivate()` — 激活/停用条件判断
  - `OnActivated()` / `OnDeactivated()` — 激活/停用回调
  - `TickActive(float DeltaTime)` — 激活时每帧更新
  - 通过 `CapabilityTags` 与 `TickGroup` 控制分组与执行时机
- **Component（组件）** —— 继承自 `UActorComponent`，挂在 actor 上提供状态与服务。
- **Actor** —— 继承自 `AHazeActor`，可放置的游戏对象。
- **Data** —— 数据资产（如动画 Feature、音效定义、设置），实现数据与逻辑分离。
- **Statics** —— 命名空间下的静态工具函数库（如 `MathStatics`、`DamageStatics`）。

> `.vscode/templates/` 下提供了 7 个代码模板（Actor / Component / Capability / Player_Capability / Movement_Capability / Anim_Instance / Effect_Handler），体现了这套约定。

---

## 顶层文件树

```
SplitFiction/
├── .vscode/                    编辑器配置与代码模板（7 个 .as.template）
├── Binds.Cache / *.Headers     AngelScript ↔ C++ 绑定缓存（自动生成）
├── PrecompiledScript.Cache     预编译脚本缓存（127MB，自动生成）
│
├── Core/                       【引擎核心框架】通用系统，不绑定具体关卡（513 文件）
├── Gameplay/                   【通用玩法系统】跨关卡复用的玩法机制（831 文件）
├── Animation/                  【动画系统】AnimInstance + Feature 数据驱动动画（919 文件）
├── Audio/                      【音频定义】SoundDef 声音定义体系（1191 文件）
├── Effects/                    【视觉特效】运行时 VFX actor/组件（58 文件）
├── Environment/                【环境美术】可放置环境 actor（光/布料/海浪/链条）（28 文件）
├── GUI/                        【用户界面】玩家 UI + 开发者调试菜单（123 文件）
├── Editor/                     【编辑器工具】关卡制作工具/可视化器/校验器（88 文件）
├── Examples/                   【示例代码】各系统教学/参考脚本（69 文件）
└── LevelSpecific/              【关卡专属】每个游戏世界一个文件夹（11,992 文件，占 76%）
```

---

## 模块文档索引

| 模块 | 文件数 | 说明 | 文档 |
|---|---|---|---|
| **Core** | 513 | 引擎核心框架（相机/移动/玩家/UI 底层等） | [Core/](./Core/README.md) |
| **Gameplay** | 831 | 通用玩法系统（AI/战斗/拾取/物理等） | [Gameplay.md](./Gameplay.md) |
| **Animation** | 919 | 数据驱动动画（AnimInstance / Feature / Capability） | [Animation.md](./Animation.md) |
| **Audio** | 1191 | SoundDef 三层音频定义体系 | [Audio.md](./Audio.md) |
| **GUI** | 123 | 玩家界面 + 开发者调试菜单（DevMenu 工具链） | [GUI/](./GUI/README.md) |
| **Editor** | 88 | 关卡制作工具、可视化器、校验器 | [Editor/](./Editor/README.md) |
| **Effects / Environment / Examples** | 58 / 28 / 69 | 视觉特效 / 环境美术 / 示例代码 | [Effects-Environment-Examples.md](./Effects-Environment-Examples.md) |
| **LevelSpecific** | 11,992 | 24 个关卡各自的招牌机制实现 | [LevelSpecific.md](./LevelSpecific.md) |

---

## 架构总结

这是一个高度模块化的协作（双人）游戏引擎脚本库：

- **`Core`** 提供引擎级框架（相机、移动、玩家、UI、音频/VO、样条、序列等）。
- **`Gameplay`** 提供跨关卡复用的玩法机制（AI、战斗、拾取、伪物理等）。
- **`Animation` / `Audio`** 是数据驱动的动画与音频管线。
- **`Effects` / `Environment`** 提供视觉特效与环境美术 actor。
- **`GUI` / `Editor` / `Examples`** 分别是界面、制作工具与教学示例。
- **`LevelSpecific`**（占 3/4 体量）是 24 个关卡各自的招牌机制实现，横跨 **Sci-Fi（Mio 科幻线）** 与 **Fantasy（Zoe 奇幻线）** 双叙事。

# Effects / Environment / Examples

> 三个较小的辅助模块。

---

## Effects/ — 视觉特效（58 文件）

> 运行时视觉效果：actor、组件、效果处理器(Effect Handler)。

| 子模块 | 功能 |
|---|---|
| **Attacks** | `UTelegraphDecalComponent` —— 预告敌人攻击范围的贴花（奇幻/科幻变体） |
| **BP** | 由蓝图转换的杂项特效 actor（间歇泉、瀑布、故障、光束、水面） |
| **Critters** | 环境生物 actor（青蛙、老鼠、飞行/逃跑生物），继承 `ABaseCritterActor`，带网络同步 |
| **Dissolve** | 材质溶解/变形效果，由编辑器可 tick 的 actor 与组件驱动 |
| **Environment** | 环境 VFX（动态水、雨、瀑布、聚光、洪水填充、天气）与静态工具 |
| **FX_Chaos** | Chaos 场系统 actor（冲击、锚点、禁用），继承 `AFieldSystemActor`，用于破坏物理。含 FieldSystems 子目录 |
| **Movement** | 玩家移动效果处理器（如外壳网格），继承 `UPlayerCoreMovementEffectHandler` |
| **Projectile** | `AVisualEffectProjectile` —— 纯视觉（非玩法）抛射物 actor |
| **WardrobeChange** | 序列驱动的换装 actor，带编辑器预览 tick |

---

## Environment/ — 环境美术（28 文件，扁平目录）

可放置的环境玩法/美术 actor 与组件，例如：

- **光照**：`DistantLight`, `LensFlare` / `LensFlareComponent`, `Godray`, `Light_Scifi_Generic_01_Large_A`
- **物理/动态**：`EnvironmentClothSim` / `EnvironmentClothSimSubsystem`（布料模拟）, `EnvironmentSwing`（秋千）, `EnvironmentCable` / `EnvironmentChain`（缆绳/链条）, `AmbientMovement`（环境运动）, `EnvironmentForceSingleton`
- **可破坏**：`Breakable`
- **水/海洋**：`OceanWaveExample`, `OceanWavePaint`
- **其它**：`AirVent`（通风口）, `HazeSphere`, `HologramWall_01`（全息墙）, `Occluder`（遮挡体）, `LandscapeVirtualTextureController`

---

## Examples/ — 示例代码（69 文件）

> 演示引擎各系统用法的教学/参考脚本。

| 子模块 | 功能 |
|---|---|
| **Angelscript** | 语言入门：actor、数组、映射、结构体、枚举、委托、specifier、pickups |
| **Capabilities** | 核心能力系统示例（player、compound、network、request、sheets、模板） |
| **AI** | 复合能力 AI 行为（待机、惊吓、眩晕、逃跑），基于 `AHazeCharacter` |
| **Aiming** | 瞄准模式能力 + 自动目标演示 |
| **Animation** | 动画 Feature/数据资产与 AnimInstance 示例（移动、骨骼过滤） |
| **Camera** | 应用相机设置与兴趣点示例 |
| **Movement** | 自定义移动数据/解算器与移动能力示例 |
| **Network** | 同步值（crumb）网络示例 actor |
| **Physics** | 使用 `Trace::` 库的物理射线示例 |
| **Rendering** | 渲染示例（渲染目标、材质参数、纹理） |
| **Interaction** | 带能力/sheet 的交互示例 actor |
| **Math** | 运行时样条与无状态移动数学示例 |
| **Disables** | actor 禁用/功能屏蔽系统示例 |
| **Editor** | 编辑器扩展示例（菜单扩展、子系统输入、可选可视化器） |
| **Development** | 开发工具示例（dev 菜单、dev 输入、dev 开关） |
| **Features** | 广泛功能演示（计时器、时间轴、效果事件、队列、时序日志、软指针） |
| **Spawner** | 简单的 `AHazeActorSpawnerBase` 生成器示例 |

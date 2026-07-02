# GUI/Debug — 开发者工具（DevMenu 体系）总览

> 本目录是 Split Fiction 的**开发者工具代码**：一整套运行时可调出的调试菜单（DevMenu）、全局开关系统（Dev Toggles）、时序日志录制回放系统（Temporal Log），以及支撑它们的即时模式 UI 框架。绝大多数代码用 `#if !RELEASE` / `#if EDITOR` / `#if TEST` 包裹，正式发布版不含。

---

## 一、DevMenu 体系机制（工具代码的核心价值）

一个 DevMenu 是一个可在游戏里调出的调试面板。所有面板都继承自两种基类之一，配合 `DevMenu::` 命名空间服务和「即时模式 UI」框架工作。

### 1. 两种基类

| 基类 | 绘制方式 | 典型用法 | 代表 |
| --- | --- | --- | --- |
| `UHazeDevMenuEntryImmediateWidget` | **自带 `Drawer` 成员**，直接在 `Tick()` 里用 `Drawer.BeginVerticalBox()...` 即时绘制整棵 UI | 逻辑简单、纯代码画、无需美术摆盘 | AnimationDevMenu、PerformanceDevMenu、NetworkDevMenu、StencilEffectDevMenu、SpeedEffectDevMenu、ComposableSettingsDevMenu |
| `UHazeDevMenuEntryWidget` | **需 BindWidget 常规 UMG 控件**（`UListView`/`USplitter`/`UButton`…）；若要即时绘制则 BindWidget 一个 `UHazeImmediateWidget Content`，用 `Content.Drawer` 画 | 复杂布局、需列表/分栏/搜索框、混合 UMG + 即时 | CapabilityDevMenu、TemporalLogDevMenu、FunctionsDevMenu、TogglesDevMenu、Audio、EffectEventDevMenu、LevelsDevMenu、Vox 系 |

> 记忆点：**Immediate 基类 = 自带 Drawer，纯代码画**；**普通基类 = 摆好 UMG 再 BindWidget，需要即时绘制时借道成员 `UHazeImmediateWidget Content` 的 Drawer**。

### 2. `DevMenu::` 命名空间服务

菜单通过 `DevMenu::` 全局函数与运行时交互（引擎侧实现，本目录是消费方）：

| 服务 | 作用 | 使用方 |
| --- | --- | --- |
| `TriggerActorPicker()` | 弹出「选调试 Actor」拾取器 | Animation / ComposableSettings / EffectEvent |
| `GetDebugActor()` / `GetDebugActorName()` | 取当前被调试的 Actor | Capability / Animation / TemporalLog / … |
| `GatherAllDevFunctions(List)` | 收集全世界的 Dev Function | FunctionsDevMenu |
| `CallDevFunction(Object, FuncName)` | 运行时调用某 Dev Function | FunctionsDevMenu |
| `ClosePopupDevMenu()` / `CloseDevMenuOverlay()` | 关闭弹出菜单/覆盖层 | LevelsDevMenu |
| `AddTogglesCategoryAssociation` / `AddDevToggleTooltip` / `GetTogglesInCategory` / `GetTogglesGroup` / `AddTogglePreset` … | Dev Toggles 注册与查询 | TogglesDevMenu |

配合 `GetDebugActor()`，很多菜单区分**双人 Mio / Zoe**，用 `PlayerColor::Mio`（蓝）/ `Zoe`（红）或 `GetPlayerDebugColor()` 上色。

### 3. 即时模式 UI 框架（类似 ImGui）

即时模式：**每帧在 Tick 里链式重建 UI，用完即弃，代码即状态**。链式节点（详见 `Immediate/README.md`）：

```
Drawer.BeginVerticalBox()
   .BorderBox().BackgroundColor(...).MinDesiredHeight(...)
   .HorizontalBox()
      .Text("Metric: ").Scale(1.0)
      .ComboBox().ChooseEnum(PageType)   // 下拉直接回写枚举变量
   .SlotFill().ListView(N)               // for(int i : List){ List.Item()... }
      ...
   .PaintCanvas().Line()/.RectFill()/.Text()  // 低级自绘(图表)
```

`if (Box.Button("X")) { ... }` —— 按钮返回本帧是否被点击，直接内联处理。`Immediate/ImmediateGraph.as` 在此之上提供 `FImmediateGraph` 折线/散点图控件。

---

## 二、开关系统（Dev Toggles）

一套「声明即注册、运行时勾选、可存预设」的全局行为开关。三层结构（详见 `TogglesDevMenu/README.md`）：

- **声明层**：业务代码里定义 `FHazeDevToggleBool` / `...PerPlayer`（每玩家）/ `FHazeDevToggleGroup`（多选一）常量，构造时**从命名空间和变量名自动推导类别/路径**。`IsEnabled()` 读状态。
- **存储层**：`UHazeDevToggleSubsystem`（引擎子系统）持有所有开关的真实状态与可见性、脏标记；内置 Dev Print 打印开关。
- **展示层**：`UI/TogglesDevMenu`（左类别 / 右开关 / 搜索 / 预设），模糊搜索排序，预设可一键批量开启一整套调试配置。

---

## 三、时序日志系统（Temporal Log）

给运行中的游戏装「录像机 + 示波器」：每帧把对象数值/事件/调试形状/相机记录进日志，再用带时间轴、滚动条（scrub）、回放的菜单回看历史帧，并能把整个编辑器/PIE 世界回滚到历史帧（详见 `TemporalLog/README.md`）。

- 主控 `UTemporalLogDevMenu` 协调路径浏览器、时间轴、数值/事件列表、Watch 曲线、多日志切换、播放/暂停/步进/倍速。
- `UTemporalLogEditorScrubSubsystem`（`#if EDITOR`）负责世界整体回滚。
- 其数值/事件列表控件被 CapabilityDevMenu 直接复用来展示能力的时序数据。

---

## 四、全部 DevMenu 索引

| 目录 / 文件 | 作用 | 基类类型 |
| --- | --- | --- |
| **CapabilityDevMenu**/ | Haze 能力系统调试器：能力列表 + 详情 + 时序 + 复合树 + Block（★核心） | `UHazeDevMenuEntryWidget` |
| **TemporalLog**/ | 时序日志录制/回放/滚动/时间轴/Watch（★核心基建） | `UHazeDevMenuEntryWidget` |
| **TogglesDevMenu**/ | 全局开关系统（类别/组/预设/Dev Print）（★核心基建） | `UHazeDevMenuEntryWidget` |
| **PerformanceDevMenu**/ | 性能：Tick 的 Actor/Component 统计、Auto Disable 检查（分页） | `UHazeDevMenuEntryImmediateWidget` |
| **FunctionsDevMenu**/ | 收集并一键调用全世界的 Dev Function | `UHazeDevMenuEntryWidget` |
| **Immediate**/ | 即时模式图表控件 `FImmediateGraph`（通用基建，非菜单） | —（辅助结构） |
| AnimationDevMenu/ | 所选 Actor 的运动（locomotion）特征数据 | `UHazeDevMenuEntryImmediateWidget` |
| Audio/ | 音频调试：静音/网络阻断/测试音/Wwise CVar 可视化 | `UHazeDevMenuEntryWidget`（内嵌 Content.Drawer） |
| ComposableSettingsDevMenu/ | 所选 Actor 的可组合设置分层（类→层→细节） | `UHazeDevMenuEntryImmediateWidget` |
| DebugCamera/ | 调试相机的操作提示行（动态构建列表） | `UHazeUserWidget`（非 DevMenu 基类） |
| DevInput/ | 屏上覆盖层，列出可用 dev-input 快捷键（分页签） | `UHazeDevInputOverlayWidget`（专用基类） |
| EffectEventDevMenu/ | 所选 Actor 的 effect-event 处理器/事件列表 + 手动触发 | `UHazeDevMenuEntryWidget`（内嵌 Content.Drawer） |
| LevelFlow/ | 关卡流程配置数据（`UDeveloperSettings`，非菜单，供 Levels 菜单排序用） | —（配置类） |
| LevelsDevMenu/ | 关卡/进度点浏览与跳转 | `UHazeDevMenuEntryWidget` |
| NetworkDevMenu/ | 网络监控 + 带宽图 + 节流配置（ping/profile） | `UHazeDevMenuEntryImmediateWidget` |
| OutlineDevMenu/ | 每玩家的描边/模板（stencil）效果槽 | `UHazeDevMenuEntryImmediateWidget` |
| SpeedEffectDevMenu/ | 每玩家速度后处理效果（当前基本为桩） | `UHazeDevMenuEntryImmediateWidget` |
| VoxDevMenu/ | VOX 对白系统：配置/时间轴/触发校验/距离读数（+视口覆盖层、输入处理器） | `UHazeDevMenuEntryWidget` + 辅助类 |
| VoxPreviewDevMenu/ | 编辑器 VOX 资产浏览与试听（+临时说话人 Actor） | `UHazeDevMenuEntryWidget`（`#if EDITOR`） |

> 标 ★ 者与本文档另建的子目录 README 对应，含更细的内部架构说明。其余小型/单文件菜单在本表一行带过。

### 已单独建 README 的子目录
- `Immediate/README.md` —— 即时模式 UI 框架与图表控件
- `TogglesDevMenu/README.md` —— 开关系统三层结构
- `TemporalLog/README.md` —— 时序日志录制回放
- `PerformanceDevMenu/README.md` —— Tick 性能分析（分页）
- `FunctionsDevMenu/README.md` —— Dev 函数调用器
- `CapabilityDevMenu/README.md` —— 能力系统调试器

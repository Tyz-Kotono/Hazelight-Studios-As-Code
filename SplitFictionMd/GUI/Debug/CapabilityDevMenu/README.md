# CapabilityDevMenu（能力系统调试器）

> 职责一句话：Haze 能力（Capability）系统的核心调试器——左侧列出所选 Actor 上的所有能力（按 Tick 分组、可筛选），右侧显示选中能力的标签、激活时长、时序数据、复合能力树，并可手动阻断（Block）能力。

## 内部架构

BindWidget 型主菜单（`UHazeDevMenuEntryWidget`），左右分栏；深度复用了 TemporalLog 的数值/事件列表控件。

```
GetDebugActor() → UHazeCapabilityComponent::Get(Actor)
        │
        ▼
UCapabilityDevMenuWidget (UHazeDevMenuEntryWidget)  ← 主菜单
  ├─ CapabilityList (UDevCapabilityListWidget)   左: 能力列表
  │     CapabilityDebug::GetFilteredCapabilityDebug(Comp, Filters, Handles)
  │     - 按类别下拉 + 显隐"未激活/未启动" + 搜索框过滤
  │     - 按 Tick 组分隔, 显示 Tick 顺序; 子能力标 👶
  └─ CapabilityInfo (UDevCapabilityInfoWidget)   右: 选中能力详情
        ├─ GetInfoString(): 标签(被阻断标红)/激活时长/Tick 组提示
        ├─ TemporalValues/TemporalEvents (复用 TemporalLog 控件)
        │     UHazeTemporalLog.ReportOnFrame(该能力的时序路径)
        ├─ CompoundDrawer: 即时模式画复合能力树(可点节点跳转)
        └─ Block 按钮: SetBlockedByDevMenu() 强制阻断/解除
```

- **`CapabilityDevMenu.as`**（主文件）— 多个类：
  - `UCapabilityDevMenuConfig`（`EditorPerProjectUserSettings`）记忆分栏比例、过滤条件、上次选中的能力。
  - `UCapabilityDevMenuWidget` 主菜单：`Tick` 里跟踪 `GetDebugActor()` 的能力组件变化，切 Actor 时把选中能力映射到新组件（`GetCapabilityInOtherComponent`）；管理类别下拉（`UpdateCategoryList`）、状态过滤（手柄 F/G 键循环切换 category/status）、分栏比例存档。选中能力时会 `SetIsDebugging(true)` 通知运行时。
  - `UDevCapabilityInfoWidget` 右侧详情：`UpdateFromHandle` 拉能力调试信息 + 时序报告 + 画调试形状；复合/子能力时用即时 `CanvasPanel` 画整棵复合树（`GetCompoundTree`），点节点回调主菜单 `SelectCapability`。
  - `UDevCapabilityListWidget` / `UDevCapabilityEntryData` / `UDevCapabilityEntryWidget` 左侧列表与单行：按过滤器拉取句柄、标记选中/Tick 组首项，行内焦点/点击广播 `OnCapabilitySelected`。
- **`CompoundCapabilityDebug.as`** — `FCompoundCapabilityDebug` 辅助：在即时画布上绘制复合能力树（RunAll/Sequence/Selector/StatePicker 等类型），返回被点击/选中节点索引供跳转。
- **`CompoundCapabilityTemporalExtender.as`** — TemporalLog UI 扩展器，把复合能力信息接入时序日志侧栏面板。

## 关键文件

| 文件 | 说明 |
| --- | --- |
| `CapabilityDevMenu.as` | 主菜单（列表 + 详情 + 过滤 + Block）与全部子 Widget |
| `CompoundCapabilityDebug.as` | 复合能力树的即时模式绘制辅助 |
| `CompoundCapabilityTemporalExtender.as` | 复合能力接入 TemporalLog 的 UI 扩展器 |

## 它调试的是什么

**Haze 能力系统**（`UHazeCapabilityComponent` + Capability/CompoundCapability）——查看某个 Actor 上有哪些能力、Tick 顺序、激活/阻断状态、标签、随时间的数值/事件，以及复合能力（行为树式组合）的运行结构。是本项目玩法调试的主力工具，横跨 `CapabilityDebug::`、`DevMenu::GetDebugActor` 与 `UHazeTemporalLog` 三套系统。

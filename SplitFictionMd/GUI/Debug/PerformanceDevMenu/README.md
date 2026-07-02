# PerformanceDevMenu（性能 / Tick 分析）

> 职责一句话：即时模式的性能分析页，统计场景里正在 Tick 的 Actor/Component（按类聚合、按数量或次数排序），并检查它们的自动禁用（Auto Disable）配置。

## 内部架构

采用「主机 + 分页」结构：主 Widget 是即时模式基类，具体内容由可切换的 Page 提供。

```
UPerformanceDevMenu (UHazeDevMenuEntryImmediateWidget)
  - 顶部按钮栏: ComboBox 选 PageType (Ticking / MovingOverlaps)
  - Pages: TMap<PageType, TSubclassOf<UPerformanceDevMenuPage>>
        │  每帧: Page.UpdateButtonBar → UpdateState → UpdateList
        ▼
UPerformanceDevMenuPage (抽象基类, 3 个虚方法)
  ├─ UPerformanceDevMenuTickingPage       (Tick 统计)
  └─ UPerformanceDevMenuMovingOverlapsPage (移动重叠统计)
```

- **`PerformanceDevMenu.as`** — `UPerformanceDevMenu`。`#if EDITOR` 下非 PIE 时不刷新列表。每帧在 `Drawer` 上画按钮栏（Metric 下拉用 `ComboBox().ChooseEnum(PageType)` 直接绑枚举），再把当前页的三段逻辑依次调用。
- **`PerformanceDevMenuPage.as`** — `UPerformanceDevMenuPage` 抽象基类，仅定义 `UpdateButtonBar(ButtonBar)` / `UpdateState()` / `UpdateList(RootBox)` 三个虚方法，Page 各自 `override`。
- **`PerformanceDevMenuTickingPage.as`** — Tick 分析页：
  - 数据 `FPerformanceDevMenuTickingEntry`（按 `UClass` 聚合，含总 Tick 数、对象数、子 Tick 分解、Auto Disable 信息），`opCmp` 支持按 TickCount 或 ObjectCount 排序。
  - 两种模式（`Actors` / `Components`）：用 `Debug::GetActorsWithActiveTicks()` / `GetComponentsWithActiveTicks()` 收集；Actor 模式会把组件 Tick、能力 Tick（`GetNumTickingCapabilitiesOnCapabilityComponent`）归并到其 Owner，并读 `UDisableComponent` 的 `bAutoDisable`/`AutoDisableRange`/链接 actor，用绿/黄/红文本标出自动禁用状态。
  - `UpdateList` 用即时 `ListView`，每行还提供 🍝「跳到蓝图」/ 📜「跳到代码」按钮（`Editor::OpenEditorForClass`）。
- **`PerformanceDevMenuMovingOverlapsPage.as`** — 移动重叠分析页（同套 Page 接口，统计移动触发的重叠开销）。

## 关键文件

| 文件 | 说明 |
| --- | --- |
| `PerformanceDevMenu.as` | 主 Widget + 分页切换（即时模式基类） |
| `PerformanceDevMenuPage.as` | Page 抽象基类（3 个虚方法） |
| `PerformanceDevMenuTickingPage.as` | Tick 统计页（Actor/Component 聚合、Auto Disable 检查） |
| `PerformanceDevMenuMovingOverlapsPage.as` | 移动重叠统计页 |

## 它调试的是什么

引擎 **Tick 调度** 与 **自动禁用（DisableComponent）** 系统，以及移动重叠开销——用于定位「谁在无谓地 Tick / 谁没配好自动禁用」这类性能问题。

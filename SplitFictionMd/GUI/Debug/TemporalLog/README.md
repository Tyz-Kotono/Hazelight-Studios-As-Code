# TemporalLog（时序日志 / 录制回放系统）

> 职责一句话：把每帧的对象数值、事件、调试形状、相机视角记录进环形日志，再用一个带时间轴/滚动条（scrub）/回放的菜单回看历史帧——像给游戏运行装了个「录像机 + 示波器」。

## 内部架构

运行时侧的记录器是 `UHazeTemporalLog`（引擎侧，非本目录）。本目录是**查看与回放的 UI + 编辑器回滚支持**。

```
UHazeTemporalLog (引擎, 每帧写入)
  - 值/事件/形状按「路径」组织: /Mio/Movement/Velocity ...
  - ReportOnFrame(帧, 路径) → FHazeTemporalLogReport
  - ScrubbableComponents: 支持被回滚到历史帧的组件(动画/相机/可见性)
        ▲ 读取                              │ 触发回滚
        │                                   ▼
UTemporalLogDevMenu (UHazeDevMenuEntryWidget)  ← 主控
  ├─ Explorer  : 路径树浏览器(搜索/过滤/书签/面包屑)
  ├─ Timeline  : 时间轴(选帧/缩放/平移/框选区间/曲线)
  ├─ ValueList : 当前帧数值列表(可点击→加入 Watch)
  ├─ EventList : 事件历史列表(可点击→跳到该帧)
  ├─ WatchBox  : 已监视值(画曲线/世界里画包围盒)
  └─ 回放引擎  : 播放/暂停/步进/倍速/区间循环, 滚动时自动暂停 PIE
        │  scrub 到某帧
        ▼
UTemporalLogEditorScrubSubsystem (#if EDITOR)
  - 把整个编辑器/PIE 世界回滚到历史帧(相机/动画随之回退)
```

### 主控
- **`TemporalLogDevMenu.as`**（最大文件）— `UTemporalLogDevMenu`。协调所有子控件与 `UHazeTemporalLog`。要点：
  - **滚动/回放**：`bLockedToLatestFrame`（跟最新帧）vs 手动选帧；`TogglePlaying/UpdatePlaying` 按 `PlayRate` 与每帧真实 DeltaTime 回放，支持区间循环；`ConditionalPauseFromScrubbing` 在滚动时按配置自动暂停 PIE 世界。
  - **Scrub 分发**：`NotifyScrub(帧)` 遍历 `TemporalLog.ScrubbableComponents` 触发回滚（动画/相机/可见性可用按钮单独开关，存进 `IgnoreScrubbables`）；同时覆盖玩家/编辑器相机视角、覆盖材质时间 CVar。
  - **Watch**：点数值 → `AddWatch` 建监视条，时间轴上画曲线（`ReportGraph`），世界里画调试形状与被监视 actor 的包围盒。
  - **多日志**：`SelectableLogs` 下拉可切换多份录制（含导入 `Import` / 导出 `Export`）；`NetworkSideButton` 在 Mio/Zoe 两侧日志间切换（Mio 蓝、Zoe 红上色）。
  - **配置**：`UTemporalLogDevMenuConfig`（书签、各类记忆搜索/过滤、是否 scrub 动画/相机、暂停策略等，`EditorPerProjectUserSettings` 持久化）。选项菜单还能开关 Movement/Camera Rerun、复制事件/数值到剪贴板、看当前日志内存占用、跳 wiki 帮助。
  - **UI 扩展器**：`UpdateUIExtenders` 按报告里的 `ExtenderClasses` 动态挂载 `UTemporalLogUIExtender` 面板（见下）。

### 子控件
- **`TemporalLogExplorerWidget.as`** — 路径树浏览器：列出当前路径下的子节点，带搜索框、过滤下拉、状态下拉、书签。
- **`TemporalLogTimelineWidget.as`** — 时间轴控件：选帧、缩放、平移、框选区间、画状态条与 Watch 曲线（`OnFrameSelected` / `OnTimelineShifted` 回调主控）。
- **`TemporalLogValuesWidget.as`** — 当前帧数值列表（`UpdateFromReport`），点击广播 `OnValueClicked` 加/删 Watch。
- **`TemporalLogEventsWidget.as`** — 事件历史列表，可按帧号或游戏时间显示，点击 `OnBrowseToEvent` 跳到该帧。
- **`TemporalLogUIExtender.as`** — 扩展基类 `UTemporalLogUIExtender`：特定路径命中时用即时模式在侧栏画自定义面板（`DrawUI(Drawer, Report)`）。
- **`TemporalLogInputExtender.as`** — 手柄/键盘输入扩展。
- **`TemporalLogPath.as`** — 路径工具函数：`GetTemporalLogBaseName` / `GetTemporalLogParentPath` / `GetTemporalLogDisplayName`（去掉 `_0` 后缀）。

### 编辑器回滚
- **`TemporalLogEditorScrubSubsystem.as`**（`#if EDITOR`）— `UTemporalLogEditorScrubSubsystem`。把整个编辑器/PIE 世界回滚到选中帧：`ScrubToFrame` / `UpdateScrubbing` / `StopScrubbing`，让编辑器相机、动画姿态等随时间轴一起回到历史状态。

## 关键文件

| 文件 | 说明 |
| --- | --- |
| `TemporalLogDevMenu.as` | 主控 Widget：滚动/回放/Watch/多日志/配置 |
| `TemporalLogExplorerWidget.as` | 路径树浏览器（搜索/过滤/书签/面包屑） |
| `TemporalLogTimelineWidget.as` | 时间轴（选帧/缩放/框选/曲线） |
| `TemporalLogValuesWidget.as` | 当前帧数值列表 |
| `TemporalLogEventsWidget.as` | 事件历史列表 |
| `TemporalLogUIExtender.as` | 特定路径的自定义即时面板扩展基类 |
| `TemporalLogInputExtender.as` | 输入扩展 |
| `TemporalLogEditorScrubSubsystem.as` | 编辑器/PIE 世界整体回滚（`#if EDITOR`） |
| `TemporalLogPath.as` | 路径字符串工具函数 |

## 它调试的是什么

**几乎所有需要「看历史」的系统**：能力（Capability）状态随时间变化、移动、相机、动画姿态、任意用 `UHazeTemporalLog` 记录的对象数值与事件。CapabilityDevMenu 直接复用了本目录的 `UTemporalLogValueListWidget` / `UTemporalLogEventListWidget` 来显示所选能力的时序数据。是本项目最重量级的调试基建。

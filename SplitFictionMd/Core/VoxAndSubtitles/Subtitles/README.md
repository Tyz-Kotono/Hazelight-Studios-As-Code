# Core / VoxAndSubtitles / Subtitles

> 职责：每玩家字幕显示管理——双槽优先级仲裁、UMG 字幕控件渲染、全屏/分屏适配、本地化（CC、斜体、大字号、背景）处理。共 1 个文件。

## 内部架构（关键类/基类/数据结构/数据流）

数据流：任意系统（主要是 Vox 的 `UVoxSubtitles`）经 C++ 桥 `Subtitle::ShowSubtitle / ShowSubtitlesFromAsset / ClearSubtitlesBy*` 调到玩家身上的 `USubtitleManagerComponent`（重写 `UHazeSubtitleComponentBase` 的 BlueprintOverride 方法）。组件维护两个字幕槽，按优先级仲裁后交给 UMG 控件渲染。

```
来源系统（Vox VoiceLine / 过场 / 教程…）
   │ Subtitle::ShowSubtitle(Player, Line, Duration, Instigator, Priority)
   ▼
USubtitleManagerComponent（每玩家一个，双槽 SubtitleSlots[2]）
   │ AddSubtitle() 按 EHazeSubtitlePriority 仲裁、挤占低优先级
   ▼
USubtitleWidget（全屏控件，跟随玩家视口矩形 / 全屏切换）
   ▼
USubtitleLineWidget（单行：FilterSubtitleText 处理 CC、斜体、字号、背景）
```

### `USubtitleManagerComponent`（基类 `UHazeSubtitleComponentBase`）
- 双槽 `SubtitleSlots`（`BeginPlay` 初始化 2 个 `FActiveSubtitle`）。
- `FActiveSubtitle`：字幕行 `FHazeSubtitleLine`、剩余时长、`FInstigator`（来源标识，用于按来源清除）、来源资产、`EHazeSubtitlePriority`。
- `AddSubtitle()`：找空槽或按优先级挤占最旧/最低优先级槽，实现「双槽优先级」。
- 入口（均为 `BlueprintOverride`，由 C++ `Subtitle::` 命名空间转发）：`ShowSubtitle`、`ShowSubtitlesFromAsset`（按 `UHazeSubtitleAsset` 的时间轴 `StartTime/EndTime` 取匹配行）、`ClearSubtitlesByInstigator`、`ClearSubtitlesByAsset`。
- 控件生命周期：首条字幕 `ActivateSubtitles()` 创建/挂载全屏控件并开 Tick；超 2s 无字幕 `DeactivateSubtitles()` 移除。
- 强制状态（以 `FInstigator` 计数）：`ForceFullscreen` / `ForceHide` / `ForceTutorialSubtitleOffset`。

### `USubtitleWidget`（`UHazeUserWidget`，Abstract）
- `UpdateActiveLines()` 从组件取已显示字幕，倒序填入行控件。
- Tick 中跟随玩家未加黑边的视口矩形定位；双人过场（`bIsControlledByCutscene` 且同一 SequenceActor）或 `IsForcedFullscreen` 时切全屏；视口过小则隐藏。
- 字幕背景开关 `Haze.SubtitleBackground`。

### `USubtitleLineWidget`（`UHazeUserWidget`，Abstract）
- `FilterSubtitleText()`：按 `Haze.ClosedCaptionsEnabled` 过滤 `CC:` / `NO_CC:` 行。
- `ShouldBeItalic()`：他人台词斜体（亚洲语言除外，全屏除外）。
- `ShouldUseLargeFont()`：ko/zh 用大字号。
- 临时台词（`bDisplayAsTemp`）显示黄色。

## 协同关系（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
|---|---|---|---|
| 调入 | Vox `UVoxSubtitles` | 1 直接调用（C++ 桥 `Subtitle::`） | Vox 的 VoiceLine 经 HazeVoxSubtitles.as 推送 / 清除字幕，是主要消费者 |
| 调入 | `Subtitle::ShowSubtitlesFromAsset` | 1 直接调用 | 过场字幕资产按时间轴驱动 `ShowSubtitlesFromAsset` |
| 调入 | `FInstigator` 来源标识 | 5 可组合设置 | 各来源用 instigator 增删自己的字幕与强制状态（全屏/隐藏/偏移） |
| 调出 | `UTutorialComponent` | 1 直接调用 | `USubtitleWidget` 读取教程/取消提示以上移字幕位置 |
| 调出 | `SceneView::` / `Widget::` | 1 直接调用 | 全屏判定、视口矩形、全屏控件挂载 |
| 调出 | CVar `Haze.SubtitlesEnabled` 等 | 5 可组合设置 | 总开关、CC、Mio/Zoe 字幕、背景由控制台变量驱动 |

## 关键文件（逐文件一句话）

| 文件 | 说明 |
|---|---|
| `SubtitleManagerComponent.as` | 每玩家字幕管理组件（双槽优先级）+ 字幕全屏控件 `USubtitleWidget` + 单行控件 `USubtitleLineWidget`（CC/斜体/字号/背景）。 |

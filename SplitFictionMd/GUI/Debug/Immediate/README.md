# Immediate（即时模式 UI / 图表绘制）

> 职责一句话：为所有 DevMenu 提供类似 ImGui 的「即时模式」绘制辅助，本目录具体实现了折线/散点**图表**（`FImmediateGraph`）。

## 背景：什么是即时模式 UI

Split Fiction 的开发者菜单大量使用一套「即时模式」（immediate mode）绘制框架，核心是 `FHazeImmediateDrawer`（引擎侧，非本目录）。它与传统的 UMG（先在编辑器摆好 Widget、再 BindWidget）相反：**每帧在 `Tick()` 里用链式调用重新声明整棵 UI 树**，用完即弃、无需持久 Widget 对象。

链式 API 的常见节点（在各 DevMenu 中反复出现）：

| 调用 | 作用 |
| --- | --- |
| `Drawer.BeginVerticalBox()` / `.BeginCanvasPanel()` / `.Begin()` | 开一个根容器，返回 Handle |
| `.VerticalBox()` / `.HorizontalBox()` | 在当前槽内嵌套排布容器 |
| `.Text("...")` / `.Bold()` / `.AutoWrapText()` | 文本 |
| `.Button("...")` | 按钮，返回 `bool`（本帧是否被点击），直接 `if(...)` 处理 |
| `.ComboBox().ChooseEnum(EnumVar)` | 下拉框，直接绑定并回写枚举变量 |
| `.BorderBox()` / `.BackgroundColor()` / `.MinDesiredWidth/Height()` | 边框/背景/尺寸约束 |
| `.SlotFill(权重)` / `.SlotPadding()` / `.SlotVAlign()` | 槽的填充/内边距/对齐 |
| `.Scale()` / `.Color()` / `.Tooltip()` | 缩放/上色/悬浮提示 |
| `.ListView(N)` | 即时列表，可 `for (int i : List) { List.Item()... }` 迭代 |
| `.PaintCanvas()` → `.Line()/.RectFill()/.Text()` | 低级自绘画布（图表就画在这上面） |

因为每帧重建，代码即状态：菜单逻辑与绘制写在一起，改一行即见效，非常适合调试工具。

## 内部架构

本目录只有一个文件，提供**可复用的图表控件**，供任何 DevMenu 往即时画布上画数据曲线（例如 TemporalLog 的数值随帧变化曲线思路类似）。

### FImmediateGraphPoint
单个数据点 `(Key, Value)`。`GetPlanePosition()` 把数据坐标映射到画布像素坐标，注意 **Y 轴翻转**（屏幕 Y 向下为大）。

### FImmediateGraph
- **数据**：`TArray<FImmediateGraphPoint> Points`，`AddPoint()` 会**按 Key 有序插入**并自动扩展边界（MinX/MaxX/MinY/MaxY）。
- **边界**：支持 `SetCustomMinX/MaxX/MinY/MaxY` 手动锁定坐标范围（否则随数据自适应）。
- **绘制模式**：`EImmediateGraphDrawMode`（`Lines` / `Points` / `LinesAndPoints`，按位组合）。
- **样式**：平面背景色、点色、X/Y 轴色可配；`bLabelAxes` 控制是否画刻度与数值标签。
- **Draw()**：两个重载，一个直接吃 `FHazeImmediateSectionHandle`（自动开画布），一个吃 `FHazeImmediatePaintCanvasHandle` + 左上角 + 尺寸。内部依次画：X/Y 轴线与端点刻度、边界数值文本、平面背景矩形，最后遍历 Points 画点和相邻连线（超出 MinX/MaxX 的点跳过）。

## 关键文件

| 文件 | 说明 |
| --- | --- |
| `ImmediateGraph.as` | `FImmediateGraph` / `FImmediateGraphPoint` / `EImmediateGraphDrawMode`，即时模式下的折线/散点图控件 |

## 它调试的是什么

不针对单一运行时系统——这是**通用可视化基建**。任何需要把「一串数值 vs 时间/帧/自变量」画成曲线的 DevMenu 都可以用它（性能采样、数值监视等）。

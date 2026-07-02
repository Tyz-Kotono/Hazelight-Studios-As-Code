# Editor — 编辑器工具模块

> 路径：`SplitFiction/Editor`（88 文件，13 子模块）
>
> 全部是**仅编辑器（Editor-only）**代码，在 PIE / 打包运行时不参与。作用是给关卡设计师与美术提供制作工具、视口可视化与资产质检——本质上是项目的 **"内容制作 SDK"**。

---

## 核心理念：六大编辑器扩展基类

整个 Editor 模块并非零散脚本，而是围绕 Haze 引擎提供的 6 个 AngelScript 编辑器基类构建。理解这 6 个基类，即可理解全部 88 个文件的运作方式。

| 基类 | 作用 | 触发方式 |
|---|---|---|
| `UHazeScriptComponentVisualizer` | **组件可视化器**——在视口里为某组件画线/框/网格 | 选中带 `VisualizedClass` 组件的 actor 时自动触发 |
| `UHazeScriptDetailCustomization` | **细节面板定制**——改写 Details 面板：隐藏分类、加按钮、画即时 UI | 选中 `DetailClass` 指定的 actor 时 |
| `UEditorUtilityWidget` | **独立工具窗口**——可停靠的编辑器面板，带自定义逻辑与 UI | 从菜单手动打开 |
| `UScriptActorMenuExtension` / `UScriptAssetMenuExtension` | **右键菜单扩展**——给 actor / 资产右键菜单加操作项 | 右键选中对象时 |
| `UEditorValidatorBase` | **资产校验器**——保存/提交时检查资产合规性 | Data Validation 流程自动运行 |
| `UEditorScriptableClickDragTool` | **交互式视口工具**——捕获鼠标点击拖拽执行编辑操作 | 从工具栏激活 |

### 贯穿全模块的关键技术点

- **`UHazeImmediateDrawer` / `UHazeImmediateWidget`** —— 即时模式 UI（类似 Dear ImGui）。用 `.Text()` / `.Button()` / `.VerticalBox()` 等链式调用在面板中动态绘制 UI，大量工具依赖它。
- **`FScopedTransaction`** —— 把编辑操作包进可撤销事务（支持 Ctrl+Z）。
- **`bIsEditorOnly = true` / `SetIsVisualizationComponent(true)`** —— 标记仅编辑器对象，不进入游戏世界。
- **`UFUNCTION(CallInEditor)`** —— 把脚本函数暴露为右键菜单 / 按钮可点击的操作。

---

## 三大职责分类

Editor 模块的 13 个子模块按职责可归为三类：

### 1. 可视化（看得见）
把不可见的玩法数据（触发体、重生点、声区、纹理密度）画到视口，做到所见即所得。
- → [Visualization.md](./Visualization.md)：EditorVisualizers、VisualizationModes

### 2. 操作（改得动）
提供拉伸、画样条、合并网格、批处理、定制面板等高效编辑工具。
- → [LevelEditing.md](./LevelEditing.md)：LevelEditor、SplineEditor、Utilities、Props、HazeActorSpawner
- → [UtilityWidgets.md](./UtilityWidgets.md)：LevelFlow、Vox、BakeLightingStarter、TraceTestUtilty 等独立工具窗口

### 3. 质检（守得住）
自动拦截资产违规（开发资产泄漏、引用弃用资产等）。
- → [Validation.md](./Validation.md)：Validators、ContentBrowser

### 专项：音频工具链
- → [Audio.md](./Audio.md)：Editor/Audio 整个音频创作工具链（22 文件，第二大子模块）

---

## 子文档索引

| 文档 | 视角 | 覆盖子模块 | 文件数 |
|---|---|---|---|
| [Patterns.md](./Patterns.md) | **怎么写**（横切模式 / 骨架模板） | 全部 6 大基类 + 横切机制 | — |
| [Visualization.md](./Visualization.md) | 有什么 | EditorVisualizers (19) · VisualizationModes (12) | 31 |
| [Audio.md](./Audio.md) | 有什么 | Audio (22) | 22 |
| [LevelEditing.md](./LevelEditing.md) | 有什么 | LevelEditor (10) · SplineEditor (8) · Utilities (7) · Props (1) · HazeActorSpawner (1) | 27 |
| [Validation.md](./Validation.md) | 有什么 | Validators (1) · ContentBrowser (4) | 5 |
| [UtilityWidgets.md](./UtilityWidgets.md) | 有什么 | LevelFlow (1) · Vox (1) · BakeLightingStarter (1) · TraceTestUtilty (1) | 4 |

---

## 总结

Editor 模块是项目的**关卡 / 美术制作 SDK**，三类职责清晰：

1. **可视化** —— 把玩法数据画到视口，所见即所得。
2. **操作** —— 高效的拖拽、绘制、批处理、面板定制工具。
3. **质检** —— 自动化资产卫生守门员。

设计上高度统一：所有工具都从 6 个 Haze 编辑器基类派生，UI 普遍走即时模式绘制，编辑操作普遍包事务以支持撤销。

# Editor / 音频工具链 — Audio

> 职责：为 `Core/Audio` 与 `Audio/SoundDefinitions` 运行时音频系统提供编辑器侧的创作、预览、可视化与面板定制工具。
>
> 覆盖子模块：**Editor/Audio**（22 文件，Editor 下第二大子模块）。

---

## 子目录划分（6 组）

```
Editor/Audio/
├── Overview/              音频资产总览面板
├── SoundDefPreview/       SoundDef 声音定义预览器
├── SpotSoundMode/         点声源(Spot Sound)编辑模式
├── Visualizers/           音频对象视口可视化
├── DetailsCustomization/  音频组件细节面板定制
└── DataAssets/            Wwise 导入辅助
```

---

## 1. Overview/ — 音频资产总览（4 文件）

一个由主控件聚合多个子面板的总览工具，基于 `UHazeUserWidget` + 即时模式 UI（`UHazeImmediateWidget`）。

| 文件 | 角色 |
|---|---|
| `AudioOverviewWidget` | **主控件**。持有 Config / Tools / Assets 三个子控件（`BindWidget`），`Tick` 时统一驱动各子控件 `TickWidget`，并用 `Content.Drawer` 绘制总览内容 |
| `AudioOverviewConfigWidget` | 配置面板 |
| `AudioOverviewToolsWidget` | 工具面板 |
| `AudioOverviewAssetsWidget` | 资产列表面板 |

> 模式：一个主 Widget 组合多个职责单一的子 Widget，统一在主控件的 Tick 中分发——典型的即时模式面板组合。

---

## 2. SoundDefPreview/ — SoundDef 预览器（6 文件）

预览 `Audio/SoundDefinitions` 里的声音定义（SoundDef）。按 SoundDef 内部数据的不同类型分别渲染预览控件。

| 文件 | 预览内容 |
|---|---|
| `SoundDefPreviewWidget` | 预览器主控件 |
| `SoundDefPreviewDataWidget` | 数据节点预览 |
| `SoundDefPreviewStructWidget` | 结构体类型预览 |
| `SoundDefPreviewEnumWidget` | 枚举类型预览 |
| `SoundDefPreviewAssetObjectWidget` | 资产对象引用预览 |
| `SoundDefPreviewTriggerNodeWidget` | 触发节点（Trigger Node）预览 |

> 这套预览器让美术在编辑器内可视化检查一个 SoundDef 的内部结构（触发逻辑、参数、引用的资产），无需进游戏。

---

## 3. SpotSoundMode/ — 点声源编辑模式（4 文件）

点声源（Spot Sound，定点环境音）的专用编辑模式，以表格形式管理。

| 文件 | 角色 |
|---|---|
| `SpotAudioTableWidget` | 点声源表格主体 |
| `SpotAudioTableRowWidget` | 表格行 |
| `SpotAudioContextWidget` | 上下文/详情面板 |
| `SpotAudioToolkit` | 编辑模式 Toolkit 入口（**注意：当前文件内容整体被注释掉**，疑似停用或重构中；表格/上下文控件仍在） |

---

## 4. Visualizers/ — 音频视口可视化（4 文件）

`UHazeScriptComponentVisualizer`，把音频的空间范围画到视口（与 [Visualization.md](./Visualization.md) 同一机制）。

| 文件 | 可视化内容 |
|---|---|
| `AmbientZoneVisualizer` | 环境音区（Ambient Zone）范围 |
| `SpotSoundsVisualizer` | 点声源位置 |
| `SpotSoundsMultiVisualizer` | 多点声源 |
| `SpotSoundSplineVisualizer` | 沿样条分布的点声源 |

---

## 5. DetailsCustomization/ — 音频细节面板定制（4 文件）

`UHazeScriptDetailCustomization`，定制各类音频对象的 Details 面板。

| 文件 | 定制目标 |
|---|---|
| `AudioZoneDetailCustomization` | 音频区域 |
| `BusMixerDetailCustomization` | 总线混音器（Bus Mixer） |
| `SpotComponentDetailsCustomization` | 点声源组件 |
| `SpotMultiComponentDetailsCustomization` | 多点声源组件 |

---

## 6. DataAssets/ — 数据辅助（1 文件）

| 文件 | 功能 |
|---|---|
| `WwiseImportDataHelper` | Wwise 音频资产导入的数据辅助 |

---

## 与运行时音频模块的关系

```
Audio/SoundDefinitions   （运行时声音定义：SoundDef / Adapter / BaseImpl）
        ↑ 预览/可视化/面板定制
Editor/Audio             （本模块：让美术在编辑器内创作与检查上述定义）
        ↑ 可视化范围
Core/Audio               （运行时监听器/发射器/音区，本模块的 Visualizers 画其范围）
```

本模块不产生运行时行为，而是**让音频内容可被看见、可被预览、可被高效编辑**。

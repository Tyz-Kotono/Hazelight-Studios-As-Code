# Core / Rendering / Lights

> 职责：引擎点光 / 聚光的薄封装，仅为编辑器中的图标精灵提供可缩放尺寸，运行时行为等同原生灯光。共 2 个文件。

## 内部架构

两个类各自只在原生灯光基础上加一个编辑器图标缩放属性：

- **`AHazePointLight : APointLight`**：加 `EditorBillboardScale`，`ConstructionScript` 中（仅 `#if EDITOR`）调 `Editor::UpdateEditorSpriteSize`。
- **`AHazeSpotLight : ASpotLight`**：结构同上。

二者通过 `Meta`（DisplayName/DefaultActorLabel/HideCategories/HighlightPlacement）定制编辑器放置体验，隐藏大量无关分类。无运行时逻辑、无 Tick、无跨模块数据流。

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 调用 | 编辑器 | 直接调用 | `Editor::UpdateEditorSpriteSize(this, EditorBillboardScale)`（仅 EDITOR） |
| 被引用 | Sky 光照配置 | 数据引用 | `FHazeLightingConfig.Lights` 为 `TArray<ALight>`，可纳入这些封装灯光做配置切换 |

## 关键文件

- **HazePointLight.as**：`AHazePointLight`，点光薄封装 + 编辑器图标缩放。
- **HazeSpotLight.as**：`AHazeSpotLight`，聚光薄封装 + 编辑器图标缩放。

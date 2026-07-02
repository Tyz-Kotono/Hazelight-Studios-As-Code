# FunctionsDevMenu（Dev 函数调用器）

> 职责一句话：收集全世界所有标记为「Dev Function」的函数（关卡蓝图、Actor、组件上的），列表化、可搜索，点一下即在运行时调用。

## 内部架构

BindWidget 型菜单（`UHazeDevMenuEntryWidget`），核心是把 `DevMenu::` 服务返回的函数集合喂给一个 `UListView`。

```
DevMenu::GatherAllDevFunctions(List)   ← 引擎服务, 扫描世界里所有 Dev Function
        │
        ▼
UFunctionsDevMenu (UHazeDevMenuEntryWidget)
  - FilterTextBox: 多词搜索(匹配函数名/对象名/编辑器标签)
  - FunctionsList (UListView): 每个函数一行, 同一对象的函数带对象表头
        │  点击某行
        ▼
DevMenu::CallDevFunction(Object, FunctionName)  ← 运行时直接调用
```

- **`FunctionsDevMenu.as`** — 三个类：
  - `UFunctionsDevMenuConfig`（`EditorPerProjectUserSettings`）持久化搜索串。
  - `UFunctionsDevMenu` 主菜单：`UpdateFunctions()` 按脏标记 / 定时 / 关卡流送计数变化重建列表。为每个函数建 `UFunctionDevMenuEntryData`；当函数所属对象变化时插入「对象表头」，区分关卡脚本（绿色、大字，去 `_C` 后缀）、普通 Actor（用编辑器标签或去 `_0` 的名字）、其它对象（灰色，附上外层 Actor 名）。搜索被过滤掉的数量显示在 `FilteredCountText`。含焦点管理（列表首项获焦）。
  - `UFunctionsDevMenuEntryWidget` 单行 Widget：点击调用 `DevMenu::CallDevFunction`；表头样式在 `SetEntryData` 里按「是否关卡」上色。

## 关键文件

| 文件 | 说明 |
| --- | --- |
| `FunctionsDevMenu.as` | 配置类 + 主菜单（收集/搜索/分组）+ 单行 Widget（调用） |

## 它调试的是什么

**Dev Function 机制**本身——任何标了 Dev Function 元数据的蓝图/脚本函数都会自动出现在这里，供策划/程序在游戏里一键触发（跳关、给道具、切状态等），无需专门做按钮。是 `DevMenu::GatherAllDevFunctions` / `CallDevFunction` 两个服务的主要消费者。

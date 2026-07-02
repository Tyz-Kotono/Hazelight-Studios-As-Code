# TogglesDevMenu（开关系统 / Dev Toggles）

> 职责一句话：一套「按类别组织、可搜索、可存预设」的开发者开关系统——代码里声明 `FHazeDevToggleBool`，运行时在菜单里勾选即可影响游戏行为。

## 内部架构

Dev Toggles 分三层：**声明层（数据结构）→ 存储层（子系统）→ 展示层（UI 菜单）**。

```
业务代码里声明常量:
  DevTogglesMovement::SomeToggle = FHazeDevToggleBool(Category, n"Some Toggle")
        │  构造时经 DevMenu:: 服务把 (Category, Path, Tooltip) 注册进映射
        ▼
UHazeDevToggleSubsystem (引擎子系统, 全局单例)
  ├─ Toggles       : TMap<Path, FHazeInternalDevToggleBool>  (布尔开关的真实状态)
  ├─ ToggleGroups  : TMap<Path, FHazeInternalDevToggleGroup> (多选一的选项组)
  ├─ TogglesAddedThisSession / ...CalledButNotAdded          (可见性追踪)
  └─ bDirty                                                   (脏标记, 驱动 UI 刷新)
        │  IsEnabled() 读状态; OnChanged 广播变更
        ▼
UI/ 子目录: UTogglesDevMenu (UHazeDevMenuEntryWidget)
  左侧类别列表 + 右侧开关列表 + 顶部搜索框 + 预设行
```

### 声明层（本目录根文件）
- **`TogglesDevBool.as`** — `FHazeDevToggleBool`（单个布尔开关）与 `FHazeDevToggleBoolPerPlayer`（Mio/Zoe 各一份）。核心技巧：默认构造函数调用 `InitFromGlobalVariable()`，通过 `Script::GetNamespaceOfGlobalVariableBeingInitialized()` / `GetNameOfGlobalVariableBeingInitialized()` **从全局变量的命名空间和变量名自动推导出类别/路径**（例如 `DevTogglesMovement::DevToggle_Foo` → 类别 "Movement"、路径 "Movement/Foo"）。`IsEnabled()` 查子系统状态；`MakeVisible()` 让开关在本会话可见；`BindOnChanged()` 订阅变更。
- **`TogglesDevCategory.as`** — `FHazeDevToggleCategory`，仅一个 `CategoryName`；`MakeVisible()` 把该类别下所有开关一次性设为可见。
- **`TogglesDevGroup.as`** — `FHazeDevToggleGroup`（多选一分组）+ `FHazeDevToggleOption`（组内选项）。`GetCurrentChosenOption()` 取当前选中项；`Option.IsEnabled()` 判断某选项是否为当前选中（无显式默认则退化为第一个构造的选项）。
- **`TogglesDevPreset.as`** — `FHazeDevTogglePreset`，把一组布尔/每玩家布尔/选项打包成一个「预设」，一键批量开启。
- **`TogglesDevStatics.as`** — 内置「Dev Print」类别与 `DevPrintString` / `...Category` / `...Event` / `...Audio` 全局函数：只有对应打印开关打开时才 `PrintToScreenScaled` 到屏幕，是最常用的调试打印开关。

### 存储层
- **`TogglesDevSystem.as`** — `UHazeDevToggleSubsystem`（`UEngineSubsystem`）。持有所有开关/组的真实状态、可见性集合、脏标记。`GetInternalBool/Group()` 按需惰性创建条目；`ShowGenericGameCategories()` 显示通用类别（Movement / PlayerHealth / Input / TimeDilation / ForceFeedback / Dev Print）；`PrintCategoryString()` 支撑 `DevPrintStringCategory`。用 `access:ToggleSystem` 把内部字段只开放给开关系统相关类。整套逻辑 `#if !RELEASE` 编译期剔除。

### 展示层（UI/ 子目录）
- **`UI/TogglesDevMenu.as`** — `UTogglesDevMenu : UHazeDevMenuEntryWidget`（BindWidget 型）。左侧 `CategoryList`、右侧 `TogglesList`、搜索框、"Defaults" 重置按钮、预设 `WrapBox`、隐藏开关警告。`UpdateToggles()` 每帧按脏标记重建列表：收集本/上次会话添加的开关 → 模糊搜索评分排序 → 填充类别与开关条目。区分「通用类别」与「关卡专属类别」。`Destruct()` 会清空本会话状态并重置预设。
- **`UI/TogglesDevEntry.as`** — `FHazeDevToggleEntry`（一行的临时数据 + `AssignFuzzyScore()` 模糊搜索打分 + `opCmp` 排序：先按搜索分，再按「通用优先/类别/子类别/组」的准字母序）。
- **`UI/TogglesDevEntryWidget.as`** — `UTogglesDevMenuEntryWidget` 单行 Widget（类别头 / 子类别头 / 布尔按钮 / 选项组磁贴），`UToggleDevMenuEntryWidgetData` 携带渲染数据。
- **`UI/TogglesDevPresetEntryWidget.as`** — 单个预设按钮 Widget。

## 关键文件

| 文件 | 说明 |
| --- | --- |
| `TogglesDevBool.as` | 布尔开关 + 每玩家布尔开关；命名空间自动推导路径 |
| `TogglesDevCategory.as` | 类别定义 |
| `TogglesDevGroup.as` | 多选一开关组与选项 |
| `TogglesDevPreset.as` | 预设（批量开关打包） |
| `TogglesDevStatics.as` | Dev Print 类别与 `DevPrintString*` 函数 |
| `TogglesDevSystem.as` | `UHazeDevToggleSubsystem` 状态存储子系统 |
| `UI/TogglesDevMenu.as` | 主菜单 Widget（列表/搜索/预设） |
| `UI/TogglesDevEntry.as` | 条目数据 + 模糊搜索排序 |
| `UI/TogglesDevEntryWidget.as` | 单行条目 Widget |
| `UI/TogglesDevPresetEntryWidget.as` | 预设按钮 Widget |

## 它调试的是什么

不针对单一系统——它是**全局行为开关基建**。任何业务代码都能声明开关、在运行时无需改代码即切换（移动、玩家血量、输入、时间膨胀、力反馈、调试打印等），并可存成预设一键还原一整套调试配置。

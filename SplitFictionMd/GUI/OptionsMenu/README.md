# OptionsMenu — 选项设置体系

> 一句话职责：以「宿主 `UOptionsMenu` + 标签页 `UOptionsMenuPage` 子类 + 选项行 `UOptionWidget` 派生控件」三层结构，统一对接 `GameSettings::` API，提供游戏/图形/音频/输入/无障碍全部设置界面，含法律文本浏览、键位冲突检测与旁白支持。

主菜单（`UMainMenuOptions`）与暂停菜单（`UPauseMenu`）都内嵌同一套 `UOptionsMenu`。

---

## 内部架构

三层：

1. **宿主 `UOptionsMenu`**：顶层容器。负责标签栏创建、页切换、返回/重置/法律三个底部按钮、描述提示浮层定位、键位错误汇总、旁白。
2. **标签页 `UOptionsMenuPage`**：每个 tab 一个子类。`Construct` 时自动 `GetAllChildWidgetsOfClass(UOptionWidget)` 收集本页所有选项行并订阅焦点事件；提供 `ResetOptionsToDefault()` / `RefreshSettings()` / `ShouldShowPageOnCurrentPlatform()`。
3. **选项行 `UOptionWidget`**：单行设置的基类，定义虚接口 `Apply()` / `Reset()` / `Refresh()` / `GetDescription()` / `GetFullNarrationText()` 与焦点/悬停状态。派生出按钮/枚举/滑条/文本/键位五类。

### 宿主 `UOptionsMenu`（`OptionsMenu.as`）要点

- **标签栏**：`CreateTabButtons()` 依 `Pages[]` 建 `UOptionsMenuTabButton`（先剔除 `ShouldShowPageOnCurrentPlatform()==false` 的页，如图形页仅 PC）。
- **页切换**：`SwitchToPage(Index, FocusCause)` 清容器→`CreateWidget` 新页→更新 tab 高亮→设焦点→触发 `OnOptionsSwitchToPage` 音效→（非直接设定时）旁白。LB/RB 或 PageUp/Down 循环切页。
- **底部按钮**：Back（`OptionsMenuBack()`，退出前用 `GetOptionsErrors()` 检查键位冲突/未绑定，有错弹确认框）、Reset（弹「重置本页/全部」对话框）、Legal（`ShowLegal()` 显示分平台法律文本，摇杆/PageUp/Down 滚动）。
- **描述浮层**：`UpdateTooltipPosition()` 依当前聚焦选项几何位置与左右玩家（`bIsRightPlayer`，Zoe 镜像）+ 屏幕宽高比动态摆放描述框。
- **旁白**：`InternalNarrateFullMenu()` 拼「页名 + 聚焦项 + 控制键提示」整段朗读。
- 退出：`OnClosed` 事件（宿主态屏订阅它返回上级）+ `GameSettings::PostApplyNewSettings()`。

### `UOptionsMenuTabButton`（同文件）

`UMenuTabButtonWidget` 子类，`TabIndex` + 激活/悬停/按下高亮贴图，按 Mio/Zoe/Neutral 上色。

---

## 关键文件 / 控件

### 框架

| 文件 | 类 : 基类 | 职责 |
| --- | --- | --- |
| `OptionsMenu.as` | `UOptionsMenu : UHazeUserWidget` + `UOptionsMenuTabButton : UMenuTabButtonWidget` | 选项宿主（标签栏/页切换/底部按钮/描述浮层/法律浏览/键位错误/旁白）+ 标签按钮 |
| `OptionsMenuPage.as` | `UOptionsMenuPage : UHazeUserWidget` | 标签页基类：自动收集子选项行、焦点转发、`Reset/RefreshSettings`、`ShouldShowPageOnCurrentPlatform` |
| `OptionWidget.as` | `UOptionWidget : UHazeUserWidget` | 选项行基类：虚接口 `Apply/Reset/Refresh/GetDescription/GetFullNarrationText` + 焦点/悬停状态、`IsHoveredOrActive()` |

### 选项行控件（均 `UOptionWidget` 派生，内嵌 `UMenuSelectionHighlight` + `UHazeTextWidget`）

| 文件 | 类 : 基类 | 职责 |
| --- | --- | --- |
| `OptionButtonWidget.as` | `UOptionButtonWidget : UOptionWidget` (Abstract) | 触发动作的按钮行（非改值）；`OnClicked`、内嵌 `UMenuPromptOrButton`，键鼠/手柄布局切换 |
| `OptionEnumWidget.as` | `UOptionEnumWidget : UOptionWidget` (Abstract) | 左右箭头循环枚举值 + 圆点(pip)指示；`OptionId`/`bAutoApply`/`bShowPips`，`SelectNext/PreviousValue`。含 `UOptionEnumDotIndicator`（可点跳转的圆点） |
| `OptionSliderWidget.as` | `UOptionSliderWidget : UOptionWidget` (Abstract) | 数值滑条（`UHazeNumberSetting`）；`StepSize`/`DisplayValueMultiplier`/`ValueSuffix`，步进吸附，延迟应用，`bApplyOnFocusLost` |
| `OptionTextWidget.as` | `UOptionTextWidget : UOptionWidget` (Abstract) | 只读「标签+值」显示行（值由外部设）；如 3D 音频状态读出 |

### 标签页（均 `UOptionsMenuPage` 派生）

| 文件 | 类 : 基类 | 职责 |
| --- | --- | --- |
| `GameOptionsMenuPage.as` | `UGameOptionsMenuPage : UOptionsMenuPage` | 通用/隐私 tab：遥测枚举 + 遥测 ID 文本 + 权益覆盖；未成年隐藏遥测；隐藏开发者开关（Ctrl+F10/扳机+摇杆组合）显示 UID 与调试计时器 |
| `GraphicsOptionsMenuPage.as` | `UGraphicsOptionsMenuPage : UOptionsMenuPage` | **仅 PC**（`ShouldShowPageOnCurrentPlatform`=非主机）：窗口模式/分辨率/HDR/应用分辨率按钮/升采样(DLSS·TSR·FSR)+质量/垂直同步；分辨率变更确认框（15 秒自动回退）+ Steam Deck VSync 变通 |
| `AudioOptionsMenuPage.as` | `UAudioOptionsMenuPage : UOptionsMenuPage` | 音频 tab：扬声器类型/夜间模式/主播模式枚举 + 对象音频文本；查询 Wwise 3D 音频能力，3D 对象音频开启时禁夜间/主播模式 |
| `InputOptionsMenuPage.as` | `UInputOptionsMenuPage : UOptionsMenuPage` + `UKeybindOptionWidget` + `UKeybindInputBox` | 键位/手柄重绑 tab：由 `GameSettings::GetAllKeyBindingSettingsDescriptions` 建滚动列表，捕获新绑定、冲突检测（`UpdateInvalidBinds`）、重置。`UKeybindOptionWidget`=一标签+三绑定框（键盘/Mio/Zoe），`UKeybindInputBox`=单个可绑槽（内嵌 `UInputButtonWidget`）。含命名空间 `InputOptions::IsAllowedOverlappingBind` |
| `AccessibilityOptionsMenuPage.as` | `UAccessibilityOptionsMenuPage : UOptionsMenuPage` | 无障碍 tab：当前近乎空壳（实质内容注释掉，仅转发 `Construct`） |

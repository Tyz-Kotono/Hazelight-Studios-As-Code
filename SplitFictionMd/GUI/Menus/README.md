# Menus — 可复用菜单构件库

> 一句话职责：为 Split Fiction 所有菜单（主菜单、选项、暂停、弹窗、大厅…）提供统一的按钮/标签/面板/高亮/转圈基础控件，以及一条把 UI 交互解耦到音频/表现层的「菜单音效事件」总线。

`Menus/` 是整个 GUI 的地基。它本身不构成任何完整界面，而是被上层界面以继承或 `BindWidget` 组合方式复用。所有控件继承 `UHazeUserWidget`。

---

## 内部架构

分三类：

1. **交互控件**：`UMenuButtonWidget` / `UMenuTabButtonWidget` / `UMenuArrowButtonWidget` / `UMenuPromptOrButton` / `UMenuIconOrPrompt`——各自实现一套鼠标+键盘+手柄输入 → 事件广播（`OnClicked`/`OnPressed`/`OnFocused`…）。
2. **装饰/容器控件**：`UMenuSelectionHighlight`（选中高亮，含 Mio/Zoe/Neutral 三态贴图）、`UMenuPanelContainer`（面板头线，同样三态）、`USpinnerWidget`（转圈）。
3. **事件总线**：`UMenuEffectEventHandler`（继承 `UHazeEffectEventHandler`）+ 一整套 `FMenu*Data` / `FMainMenu*` / `FPauseMenu*` / `FMessageDialogData` 等负载结构体。控件在被点击/悬停/换态时通过 `UMenuEffectEventHandler::Trigger_On*(Menu::GetAudioActor(), Data)` 广播，音频/特效层订阅这些事件——UI 逻辑与音效解耦。

### 输入处理共性模式

所有交互控件遵循同一套 Slate 事件模式（可作为编写新菜单控件的模板）：

- `OnMouseEnter`：忽略近零位移的抖动（`CursorDelta.IsNearlyZero()`），置 `bHovered`，广播 `OnFocused`。
- `OnMouseButtonDown` 记 `bPressed`，`OnMouseButtonUp` 若仍 `bPressed` 才广播 `OnClicked`（避免拖出取消）。
- 键盘/手柄以 `OnKeyDown/Up` 处理 `EKeys::Enter` 与 `EKeys::Virtual_Accept`，区分 `bPressedByGamepad`。
- 可选 `bTriggerEffectEvents`：为纯 UMG 蓝图控件补发默认点击/悬停音效事件。

---

## 关键文件 / 控件

| 文件 | 类 : 基类 | 职责与要点 |
| --- | --- | --- |
| `MenuButtonWidget.as` | `UMenuButtonWidget : UHazeUserWidget` | **最核心的通用按钮**。可聚焦；`OnClicked`/`OnFocused` 事件；跟踪 `bFocused`/`bFocusedByMouse`/`bHovered`/`bPressed`；`IsHoveredOrActive()` 供子类换视觉；鼠标+键盘/手柄双通道触发。`UMainMenuButton`/`UPauseMenuButton`/`UMessageDialogButton`/`UOptionButtonWidget` 等均继承它 |
| `MenuTabButtonWidget.as` | `UMenuTabButtonWidget : UHazeUserWidget` | 标签页按钮（不可聚焦，靠鼠标/外部导航）。`bIsActiveTab` 标记当前页；`OnClicked`/`OnFocused`。`UOptionsMenuTabButton` 继承它 |
| `MenuArrowButtonWidget.as` | `UMenuArrowButtonWidget : UHazeUserWidget` | 左右箭头/步进按钮。四态贴图（Normal/Hovered/Pressed/Disabled），支持**长按连发**（`bRepeats`，`MouseRepeatTime` 随次数加速）、按下缩放、文字变色。用于滑条/枚举选项行 |
| `MenuPromptOrButton.as` | `UMenuPromptOrButton : UHazeUserWidget` (Abstract) | **提示/按钮二态控件**：手柄下显示按键图标（`UInputButtonWidget`），键鼠下显示可点击文字按钮。依 `GetControllerType()` 自动切换布局/对齐/最小宽度；`bDisabled`/`OnPressed`/`OnRepeat`/`OnEnabled`。选项菜单的返回/重置/法律按钮、邀请接受/拒绝均用它 |
| `MenuIconOrPrompt.as` | `UMenuIconOrPrompt : UHazeUserWidget` | `MenuPromptOrButton` 的轻量图标版：手柄下显按键图标，键鼠下显可点击图标边框（`UBorder`），按状态换色。`OnPressed`/`OnRepeat`/`OnFocused` |
| `MenuSelectionHighlight.as` | `UMenuSelectionHighlight : UHazeUserWidget` (Abstract) | 选中高亮底图。`bIsHighlighted` 控制显隐；`Style`（Short/Long）× 玩家（Mio/Zoe/Neutral）选贴图；底图为 `UScalableSlicedImage`。每 Tick 依 `PausingPlayer` 更新 |
| `MenuPanelContainer.as` | `UMenuPanelContainer : UHazeUserWidget` (Abstract) | 菜单面板容器，绘制顶部头线 `HeaderLine`（Mio/Zoe/Neutral 三态贴图，`bShowHeaderLine` 控制显隐） |
| `SpinnerWidget.as` | `USpinnerWidget : UHazeUserWidget` (Abstract) | 极简转圈：仅一个 `BindWidget UImage Image`，旋转/材质动画在 UMG 资产里。被存档转圈、忙碌任务等复用 |
| `MenuEffectEventHandler.as` | `UMenuEffectEventHandler : UHazeEffectEventHandler` (Abstract) + 负载结构体 | **UI→音效/表现事件总线**。定义一批 `BlueprintEvent`：`OnCameraTransition`/`OnMenuStateChanged`/`OnOptionsSwitchToPage`/`OnChapterSelect*`/`OnMessageDialog`/`OnPauseMenuStateChanged`/`OnCharacterSelected`/`OnDefaultClick`/`OnDefaultHover`/`OnBootMenuChanged`/`OnTrailUpsell`。同文件定义各事件负载结构体（如 `FMainMenuStateChangeData`、`FMenuActionData`、`FMessageDialogData` 等）——是前端各模块与音频层之间的契约集散地 |

---

## 复用关系

- `MenuButtonWidget` → 被 `MainMenuButton`、`PauseMenuButton`、`MessageDialogButton`、`OptionButtonWidget`、`LobbyStartTypeButton` 继承。
- `MenuTabButtonWidget` → 被 `OptionsMenuTabButton` 继承。
- `MenuPromptOrButton` / `MenuArrowButtonWidget` / `MenuSelectionHighlight` → 作为 `BindWidget` 广泛嵌入选项行、弹窗按钮、大厅面板。
- `MenuEffectEventHandler` 的负载结构体被 MainMenu / OptionsMenu / PauseMenu / MessageDialog / ChapterSelect 各处 `Trigger_On*` 调用。

# GUI — 玩家面向 UI 与可复用控件框架

> 一句话职责：Split Fiction 的全部游戏内/前端界面层——从启动引导、主菜单前端、选项设置、模态弹窗到常驻 HUD 叠加，全部构建在 `UHazeUserWidget` 基类与 `Menus/` 可复用构件库之上。

本目录（除 `Debug/` 外）覆盖玩家在游戏中可见的所有界面。所有开发者调试菜单/工具链单独归档在 [Debug/README.md](./Debug/README.md)。

---

## 1. 框架总览

### 1.1 控件基类 `UHazeUserWidget`

引擎（Haze）层提供的 UMG 用户控件基类，本目录几乎所有 AngelScript 控件都直接或间接继承它。约定：

- 生命周期回调以 `BlueprintOverride` 覆写：`PreConstruct` / `Construct` / `Tick` / `OnAdded` / `RemoveFromScreen` / `Destruct`。
- 输入事件同样以 `BlueprintOverride` 覆写：`OnFocusReceived` / `OnFocusLost` / `OnKeyDown/Up` / `OnMouseButtonDown/Up` / `OnMouseEnter/Leave` / `OnMouseMove` / `OnAnalogValueChanged`，返回 `FEventReply::Handled()/Unhandled()` 决定是否吞掉输入。
- UMG 蓝图资产中的子控件通过 `UPROPERTY(BindWidget)` 注入（命名必须匹配），脚本只写逻辑，视觉布局留在 `.uasset`。
- 焦点管理走 `Widget::SetAllPlayerUIFocus` / `SetAllPlayerUIFocusBeneathParent` / `DisableAllPlayerUIFocus` 等静态函数——双人（Mio/Zoe）共享同一套 UI，故多为「所有玩家焦点」语义。
- 全屏挂载走 `Widget::AddFullscreenWidget(Class, EHazeWidgetLayer::...)` / `RemoveFullscreenWidget`，层级用 `EHazeWidgetLayer`（Menu / Dev …）+ `SetWidgetZOrderInLayer`。

### 1.2 双人配色约定（Mio / Zoe / Neutral）

前端反复出现三态配色分支：读取 `Game::HazeGameInstance.PausingPlayer`（`EHazeSelectPlayer::Mio/Zoe`）或 `GetPausingPlayer()`，据此切换纹理/颜色：

- Mio ≈ 粉色系，Zoe ≈ 黄绿系，Neutral ≈ 青色系。

`MenuPanelContainer`、`MenuSelectionHighlight`、`OptionsMenuTabButton`、`MessageDialogButton` 等均实现了这套分支。

### 1.3 可复用构件库 `Menus/`

其他一切菜单的地基。提供按钮（`UMenuButtonWidget`）、标签页按钮（`UMenuTabButtonWidget`）、箭头按钮（`UMenuArrowButtonWidget`）、提示/按钮二态控件（`UMenuPromptOrButton` / `UMenuIconOrPrompt`）、选中高亮（`UMenuSelectionHighlight`）、面板容器（`UMenuPanelContainer`）、转圈（`USpinnerWidget`）以及菜单音效事件总线（`UMenuEffectEventHandler` + 一大批 `FMenu*Data` 结构体）。详见 [Menus/README.md](./Menus/README.md)。

### 1.4 前端 `MainMenu/`

由常驻世界 Actor `AMainMenu` 驱动的**状态机式前端**：`EMainMenuState`（Splash / MainMenu / Lobby* / Options / BusyTask / Credits …）每态对应一个 `UMainMenuStateWidget` 子类，`SwitchToState()` 负责换控件 + 换相机 + 触发音效。包含初始引导 `InitialBoot/`（EULA/隐私/遥测/无障碍/手柄页）、大厅 `Lobby/`（玩家/开始类型/章节/角色/跨平台）、本地无线联机 `LocalWireless/`。详见 [MainMenu/README.md](./MainMenu/README.md)。

### 1.5 选项体系 `OptionsMenu/`

`UOptionsMenu` 宿主 + 标签页 `UOptionsMenuPage` 子类（游戏/图形/音频/输入/无障碍）+ 选项行控件（`UOptionWidget` 派生的 Enum/Slider/Text/Button/Keybind）。统一对接 `GameSettings::` API，含法律文本浏览、冲突键位检测、旁白（narration）支持。详见 [OptionsMenu/README.md](./OptionsMenu/README.md)。

### 1.6 模态弹窗 `MessageDialog/`

「data / singleton / statics / widget」四件套：`FMessageDialog`（数据）→ `ShowPopupMessage()`（静态入口）→ `UMessageDialogSingleton`（队列/生命周期）→ `UMessageDialogWidget`（呈现）。全局最高优先级弹窗，也用于消费在线子系统的错误与问询。详见 [MessageDialog/README.md](./MessageDialog/README.md)。

### 1.7 常驻叠加 `GlobalHUD/`

游戏全程常驻的顶层叠加：存档转圈、身份接入提示、游戏邀请、会话计时器、跳过过场进度、调试信息条。详见 [GlobalHUD/README.md](./GlobalHUD/README.md)。

---

## 2. 子目录索引

| 子目录 | 文件数 | 职责 | 文档 |
| --- | --- | --- | --- |
| `Menus/` | 9 | 可复用菜单构件库（按钮/标签/面板/高亮/转圈/音效总线），一切菜单的基础 | [Menus/README.md](./Menus/README.md) |
| `MainMenu/` | 17 + InitialBoot 7 + Lobby 6 + LocalWireless 1 | 完整前端状态机：Splash/引导/大厅/角色·章节选择/选项/致谢/安装进度 | [MainMenu/README.md](./MainMenu/README.md) |
| `OptionsMenu/` | 12 | 选项设置：标签页体系 + 选项行控件，对接 GameSettings | [OptionsMenu/README.md](./OptionsMenu/README.md) |
| `MessageDialog/` | 4 | 模态弹窗四件套（data/singleton/statics/widget） | [MessageDialog/README.md](./MessageDialog/README.md) |
| `GlobalHUD/` | 4 | 常驻顶层叠加（存档转圈/邀请/计时器/跳过过场/调试信息） | [GlobalHUD/README.md](./GlobalHUD/README.md) |
| `ChapterSelect/` | 3 | 章节选择面板 + 章节预览图 + 章节角色 3D 网格（大厅复用） | 见下方「小控件」 |
| `PauseMenu/` | 2 | 游戏内暂停菜单（内嵌 OptionsMenu/ChapterSelect）+ 远端暂停遮罩 | 见下方「小控件」 |
| `LoadingScreen/` | 4 | 加载屏、加载后处理 Actor、过场转场特效（详见别处「加载屏轮询」文档） | 见下方「小控件」 |
| `InputButton/` | 2 | 按键/摇杆输入图标控件（键位设置与提示复用） | 见下方「小控件」 |
| `Credits/` | 1 | 滚动演职员表引擎（前端与关卡结束复用） | 见下方「小控件」 |
| `Image/` | 1 | `UScalableSlicedImage` 九宫格缩放图（菜单底图复用） | 见下方「小控件」 |
| `Text/` | 1 | `UHazeTextWidget` 全局样式化文本基元（几乎所有控件复用） | 见下方「小控件」 |
| `RadialProgress/` | 1 | 环形进度控件（材质驱动） | 见下方「小控件」 |
| `NetworkOverlay/` | 1 | 远端/网络标签叠加 | 见下方「小控件」 |
| `Debug/` | 20+ 子目录 | 开发者调试菜单与工具链（**单独归档**） | [Debug/README.md](./Debug/README.md) |

---

## 3. 小控件速览（未单独建篇的目录）

以下目录规模小或已在别处分析，在此一行带过。核心复用基元（`Text`/`Image`/`InputButton`）被大量控件以 `BindWidget` 组合。

| 文件 | 类 : 基类 | 职责 |
| --- | --- | --- |
| `Text/HazeTextWidget.as` | `UHazeTextWidget : UHazeUserWidget` | **全局样式化文本基元**。封装 `FText` + 尺寸/颜色/修饰/对齐/换行/暗色/描边/阴影；`ChangeText`/`Update()` 触发 UMG 重绘；`EHazeTextColor` 含 Mio/Zoe 玩家色。被所有选项行、键位框、进度文本等复用 |
| `Image/ScalableSlicedImage.as` | `UScalableSlicedImage : UHazeUserWidget` | 自绘九宫格图（`OnPaint`/`DrawBox`）按比例缩放且保持切片行为；`SetBrushFromTexture`/`SetBrushColor`。用作菜单磁贴底图（如章节磁贴、提示按钮） |
| `InputButton/InputButtonWidget.as` | `UInputButtonWidget : UHazeInputButton` | 显示某动作当前的按键/手柄图标；跟踪 `DisplayedKey`/`ControllerType`，按玩家上色。嵌入键位框、菜单提示中 |
| `InputButton/InputStickWidget.as` | `UInputStickWidget : UHazeUserWidget` | 摇杆方向提示图标；按玩家（Mio/Zoe）着色摇杆材质，跟踪手柄类型变化 |
| `RadialProgress/RadialProgressWidget.as` | `URadialProgressWidget : UHazeUserWidget` (Abstract) | 材质驱动的环形进度；暴露半径/衰减/阴影/角度/颜色/进度参数与 `SetProgress`/`SetBarAngles`/`SetRadius` |
| `NetworkOverlay/NetworkOverlay.as` | `UNetworkOverlay : UHazeUserWidget` | 依 `ShouldShowNetworkLabel()`（编辑器 vs 发行、分屏尺寸、是否有控制权）决定是否显示远端/网络标签 |
| `Credits/CreditsWidget.as` | `UCreditsWidget : UHazeUserWidget` | 滚动演职员表引擎；消费 `UCreditsData` 资产列表，生成分节控件滚动。含 `UCreditsSection_*`（自定义/文本块/多列/定时）子类 |
| `ChapterSelect/ChapterSelectWidget.as` | `UChapterSelectWidget : UHazeUserWidget` (Abstract) | 章节选择面板；从 `UHazeChapterDatabase` 构建分组/条目，箭头导航。含磁贴 `UChapterSelectItemWidget`（复用 `UScalableSlicedImage`，按状态×玩家换纹理） |
| `ChapterSelect/ChapterImageWidget.as` | `UChapterImageWidget : UHazeUserWidget` (Abstract) | 章节预览图；异步加载软引用纹理喂入材质参 `ChapterImage` |
| `ChapterSelect/ChapterSelectPlayerMesh.as` | `AChapterSelectPlayerMesh : AHazeActor` | 章节选择旁的 3D 角色；双缓冲 `MeshA`/`MeshB` 做故障(glitch)混合切换，驱动材质 `Glitch_*` 参 |
| `PauseMenu/PauseMenu.as` | `UPauseMenu : UHazeUserWidget` | 游戏内暂停菜单，状态机（`EPauseMenuState`: PauseMenu/OptionsMenu/ChapterSelect），内嵌 `UOptionsMenu` 与 `UChapterSelectWidget`，处理各类确认弹窗、左右玩家面板滑动、开发者隐藏菜单+自由相机。含 `UPauseMenuButton` |
| `PauseMenu/RemotePausedOverlay.as` | `URemotePausedOverlay : UHazeUserWidget` | 空壳（逻辑在 UMG 蓝图）；对方暂停时向本方显示 |
| `LoadingScreen/LoadingScreen.as` | `ULoadingScreen : UHazeLoadingScreen` (Abstract) | 加载 UI，显示流式安装下载进度、远端安装框、着色器编译进度（详见别处「加载屏轮询 Progress::」文档） |
| `LoadingScreen/LoadingScreenPostProcessActor.as` | `ALoadingScreenPostProcessActor : AHazeActor` (Abstract) | 生成常驻自副本驱动加载屏后处理材质、强制「无视图」渲染直到新关卡流入后自毁 |
| `LoadingScreen/LoadingTransistionEffect.as` | `ULoadingTransistionEffect : UHazeUserWidget` | 双 RT 乒乓模拟（Swap0/Swap1）喂入流体过场材质 |
| `LoadingScreen/LoadingTransistionScreen.as` | `ULoadingTransistionScreen : UHazeUserWidget` (Abstract) | 常驻过场遮罩，注册 `UFadeSingleton`，活动关卡切换且加载完成后触发 `BP_TransistionOver` |

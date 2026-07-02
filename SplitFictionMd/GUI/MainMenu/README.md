# MainMenu — 前端状态机

> 一句话职责：由常驻世界 Actor `AMainMenu` 驱动的完整游戏前端——从启动闪屏、初始引导、大厅组队、角色/章节选择，到选项、致谢、安装进度，全部以「状态 ↔ 控件 ↔ 相机」三元组的状态机形式组织。

---

## 内部架构

### 核心：`AMainMenu`（`MainMenu.as`）

一个 `AHazeActor`（`bTickEvenWhenPaused = true`），是前端的中枢：

- **状态枚举 `EMainMenuState`**：`None / Splash / MainMenu / LobbyPlayers / LobbyChooseStartType / LobbyChapterSelect / LobbyCharacterSelect / LobbyCrossplay / LocalWireless / Options / BusyTask / Credits`。
- **两张并行数组**（下标即状态枚举）：`StateWidgets[]`（每态的控件类）与 `StateCameras[]`（每态的相机 + 混入时间）。编辑器里逐态配置。
- **`SwitchToState(NewState, bSnap)`（私有核心）**：移除旧控件 → 挂新控件（`Widget::AddFullscreenWidget(..., EHazeWidgetLayer::Menu)`）→ 设焦点 → 调 `OnTransitionExit/Enter` → 触发 `OnMenuStateChanged` 音效事件 → 依相机 Tag 差异决定淡入淡出/混合/直切相机（`AMenuCameraUser`）。
- **`Goto*` / `ShowOptionsMenu` / `ShowCredits` / `ReturnToMainMenu` / `ReturnToSplashScreen` / `ConfirmMenuOwner` / `CloseMainMenu`**：对外的状态迁移入口，均先过 `CanSwitchToState()` 校验（主身份不匹配时只允许回 Splash）。
- **`Tick` 里的隐式状态收敛**：忙碌任务优先（`HasBusyTask` → BusyTask 态）；主身份变化回 Splash；大厅存在则强制进/出 Lobby 态；大厅已开局则 `CloseMainMenu`；播放菜单过场时挂 `MainMenuSkipCutsceneOverlay` 捕获跳过输入。
- **身份/所有权**：`OwnerIdentity`（当前操作菜单的玩家身份）、`Online::PrimaryIdentity` 联动；`IsOwnerInput()` 用于区分「菜单主人」与其它输入源。
- **角色网格预览**：`SetCharacterMeshVariants()` 异步加载 Mio/Zoe 的 `USkeletalMesh`+`UAnimSequence` 软引用，喂给 `AChapterSelectPlayerMesh`；`UpdateChapterSelectMeshVisibility()` 按态显隐。
- **关卡预载**：`UpdateLevelPreload()` 在大厅角色选择且新游戏时 `Progress::PreloadProgressPointFromDisk`。
- 单例访问：`AMainMenu::GetMainMenu()`（`TListedActors`）。

### 状态控件基类：`UMainMenuStateWidget`（`MainMenuStateWidget.as`）

所有「态屏」的基类：

- `OnTransitionEnter/Exit(PreviousState, bSnap)` + 蓝图钩子 `BP_OnTransitionEnter/Exit`；淡入淡出由 `FadeInDelay/Duration`、`FadeOutDuration` + Tick 里的 `AddTimer/RemoveTimer` 驱动（`bSnap` 则瞬切）。
- `bShowMenuBackground` / `MenuBackgroundTitle` / `bShowButtonBarBackground` 供 `AMainMenu.UpdateMenuBackgroundWidget()` 决定共享背景 `MainMenuBackgroundWidget` 的显隐与标题。
- `OnKeyDown` 默认吞掉非「菜单主人」的输入（大厅等会覆写以放行第二玩家）。

### 三条继承链

- `UMainMenuStateWidget` → `UMainMenuWidget`（主菜单根）、`UMainMenuOptions`、`UMainMenuCreditsWidget`、`UMainMenuBusyTaskWidget`、`USplashScreenWidget`、`ULocalWirelessFindLanGamesWidget`、`ULobbyCrossplayWidget`。
- `ULobbyWidgetBase : UMainMenuStateWidget` → 全部大厅态屏。
- `UInitialBootSequencePage : UHazeUserWidget` → 全部初始引导页（`UInitialBootOptionsPage` 为选项页中间基类）。

---

## 关键文件 / 控件

### 根级（`MainMenu/`）

| 文件 | 类 : 基类 | 职责 |
| --- | --- | --- |
| `MainMenu.as` | `AMainMenu : AHazeActor` | 前端状态机中枢（见上） |
| `MainMenuStateWidget.as` | `UMainMenuStateWidget : UHazeUserWidget` | 态屏基类，淡入淡出 + 转场钩子 + 输入门控（见上） |
| `MainMenuWidget.as` | `UMainMenuWidget : UMainMenuStateWidget` | **主菜单根屏**：本地/本地无线/在线开始、开发者加入/菜单、选项、致谢、退出、购买/好友通行证按钮；换身份、EA 登录；平台差异（Steam 主机弹窗选 Steam/EA 服务器、Switch 本地无线）；进入时按序动画显现按钮 |
| `MainMenuButton.as` | `UMainMenuButton : UMenuButtonWidget` | 单个主菜单导航按钮（常/悬贴图、连接线、文字）；`bIsFirstOption/bIsLastOption`、`AnimateVisible(Delay)`、悬停动画、聚焦旁白 |
| `MainMenuBackgroundWidget.as` | `UMainMenuBackgroundWidget : UHazeUserWidget` | 共享背景 chrome（标题+箭头+按钮栏底图），`SetMenuTitle`/`SetShowButtonBar` |
| `SplashScreenWidget.as` | `USplashScreenWidget : UMainMenuStateWidget` | **入口态**（Press any button）：身份/账号选择、登录、档案加载、EULA 门控、驱动初始引导序列；播放本地化闪屏 Bink；完成后 `MainMenu.ConfirmMenuOwner()`。是 splash→引导页→主菜单的编排者 |
| `MainMenuOptions.as` | `UMainMenuOptions : UMainMenuStateWidget` | 选项态，宿主 `UOptionsMenu`；`OnClosed` → `ReturnToMainMenu()` |
| `MainMenuCreditsWidget.as` | `UMainMenuCreditsWidget : UMainMenuStateWidget` | 致谢态，包 `UCreditsWidget`；LB+RB 长按 10× 快进，播完/Back 返回 |
| `MainMenuBusyTaskWidget.as` | `UMainMenuBusyTaskWidget : UMainMenuStateWidget` | 忙碌/连接态，显示 `GetBusyTaskText()`，Back 可取消（若 `CanCancelBusyTask`） |
| `MainMenuSkipCutsceneOverlay.as` | `UMainMenuSkipCutsceneOverlay : UHazeUserWidget` | 菜单过场跳过遮罩：各本地玩家长按 Cancel，映射输入设备→玩家，调 `CameraUser.ActiveLevelSequenceActor.NetSetPlayerWantsToSkipSequence()` |
| `MenuInstallProgressWidget.as` | `UMenuInstallProgressWidget : UHazeUserWidget` | 后台安装/下载进度小叠加（`Online::GetDisplayInstallProgress()`） |
| `TrialUpsellWidget.as` | `UTrialUpsellWidget : UHazeUserWidget` | 试玩/演示升级购买提示（购买/返回按钮+转圈），暂停游戏、发遥测、开商店页；含 `DemoUpsell` 命名空间助手 |
| `MenuCameraUser.as` | `AMenuCameraUser : AHazeAdditionalCameraUser` | 菜单相机 + 淡入淡出 + 字幕管理：`FadeIn/OutView`/`AddTemporaryFade`/`Snap/BlendToCamera`；`ASecondaryMenuCameraUser` 供分屏第二玩家 |
| `MenuCameraListeners.as` | `AMenuCameraListeners : AHazeActor` | 两个音频监听器平滑跟随 `CameraUser.ActiveCamera`（远则瞬移近则插值），设混响标记 |
| `MainMenuMusic.as` | `AMainMenuMusic : AHazeActor` | Begin/EndPlay 时挂/卸 `UHazeMusicSoundDef` 到音乐管理器 |
| `MainMenuHideLevelsVolume.as` | `AMainMenuHideLevelsVolume : AHazeActor` | 盒体积，据首玩家相机是否在内切换目标关卡渲染（菜单背景优化） |
| `EULALicenses.as` | `FLicenseContent` / `ULicenseAsset : UDataAsset` | 许可文本数据资产：分文化存储 + `GetLicenseForCulture()`（文化→语言→中文变体→英文兜底）。EULA/隐私引导页的后备数据 |

### 初始引导（`InitialBoot/`）——由 SplashScreen 经 `InitialBootSequence_Forward/Back` 驱动

| 文件 | 类 : 基类 | 职责 |
| --- | --- | --- |
| `InitialBootSequencePage.as` | `UInitialBootSequencePage : UHazeUserWidget` (Abstract) + `UInitialBootScaffold` | 全部引导页基类（持 `SplashScreen`/`Identity`/`Scaffold`，`Show()`/`CanBackToPage()`）；`UInitialBootScaffold` 为共享页框（标题/提示/`ShowAnim`） |
| `InitialBootControllerPage.as` | `UInitialBootControllerPage : UInitialBootSequencePage` | 「建议使用手柄」页，主机端自动跳过 |
| `InitialBootOptionsPage.as` | `UInitialBootOptionsPage : UInitialBootSequencePage` | 通用选项页：自动收集子 `UOptionWidget`，逐项提示，Continue/Back 导航。无障碍页基类 |
| `InitialBootAccessibilityPage.as` | `UInitialBootAccessibilityPage : UInitialBootOptionsPage` | 无障碍选项（扬声器类型/夜间模式/3D 对象音频），3D 对象音频开启时禁夜间模式 |
| `InitialBootPrivacyLicense.as` | `UInitialBoot_PrivacyLicense : UInitialBootSequencePage` | 分平台隐私许可（`ULicenseAsset`），摇杆滚动，Accept 前进，可自动接受 |
| `InitialBootTelemetryPage.as` | `UInitialBootTelemetryPage : UInitialBootSequencePage` | 遥测选择加入/退出，写 `TelemetryOptIn` 设置；未成年自动禁用 |
| `InitialBootEULA.as` | `UInitialBoot_EULA : UInitialBootSequencePage` | EULA 接受（分区/平台措辞）；Accept 写档案 `EULA_Accepted=true` 前进，Decline 弹退出/继续对话框并取消引导。SplashScreen 确认菜单主人前的门 |

### 大厅（`Lobby/`）——均继承 `ULobbyWidgetBase`（除 Crossplay），镜像 `UHazeLobby.LobbyState`

| 文件 | 类 : 基类 | 职责 |
| --- | --- | --- |
| `LobbyWidgetBase.as` | `ULobbyWidgetBase : UMainMenuStateWidget` | 大厅屏基类：进入时缓存 `Lobby`；Tick 持续把 `Lobby.LobbyState` 同步为 `MainMenu.Goto*`（保持两玩家视图一致）；`LeaveLobby()`、试玩模式/全本指示、网络/开始类型文案助手 |
| `LobbyPlayersWidget.as` | `ULobbyPlayersWidget : ULobbyWidgetBase` | 首个大厅屏：两玩家槽（`ULobbyPlayerInfo`）、邀请/加入流程（本地登录+档案、在线邀请、好友通行证信息、跨平台警告）；两人齐则进 ChooseStartType |
| `LobbyChooseStartTypeWidget.as` | `ULobbyChooseStartTypeWidget : ULobbyWidgetBase` | 新游戏/继续/章节选择三选（无存档则隐藏继续）；含 Konami 码解锁全章节（`FKCodeHandler`）与 `ULobbyStartTypeButton` |
| `LobbyChapterSelectWidget.as` | `ULobbyChapterSelectWidget : ULobbyWidgetBase` (Abstract) | 章节/进度点选择网格（`UChapterSelectWidget`），访客镜像主机；按章节设角色网格变体，校验可开始性；定义共享 `FKCodeHandler` 与 `ULobbyChapterSelectItemWidget` |
| `LobbyCharacterSelectWidget.as` | `ULobbyCharacterSelectWidget : ULobbyWidgetBase` (Abstract) | Mio/Zoe 选角+准备；播新游戏开场过场；双方就绪后倒计时+淡出并 `Lobby::Menu_StartLobbyGame()`+`CloseMainMenu()`。含 `ULobbyCharacterSelectPlayer` 与 3D 平板 Actor `ALobbyCharacterSelectTablet`。**终态，启动游戏** |
| `LobbyCrossplayWidget.as` | `ULobbyCrossplayWidget : UMainMenuStateWidget` | EA 跨平台好友屏：好友列表/搜索/黑名单/举报（PS5）；含 `FFriendSorter` 与 `ULobbyFriendWidget`（加入/邀请/加好友/拉黑/档案）。**注意直接继承态屏而非 `ULobbyWidgetBase`** |

### 本地无线（`LocalWireless/`）

| 文件 | 类 : 基类 | 职责 |
| --- | --- | --- |
| `LocalWirelessFindLanGamesWidget.as` | `ULocalWirelessFindLanGamesWidget : UMainMenuStateWidget` | LAN/本地无线游戏浏览器：`Online::GetLanGames` 列表（可加入者优先排序）、搜索/返回；含 `FLanGameSorter` 与条目 `ULocalWirelessLanGameWidget`（`Online::JoinLanGameLobby`） |

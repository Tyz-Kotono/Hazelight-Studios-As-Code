# GlobalHUD — 常驻顶层叠加

> 一句话职责：游戏全程常驻、位于最高层级的一组叠加控件——存档转圈、身份接入(engagement)提示、跨平台游戏邀请、会话/章节计时器、跳过过场进度、以及开发/测试构建下的调试信息条。

区别于随状态切换而增删的菜单控件，GlobalHUD 的控件通过高 ZOrder（801 / 9001）常驻，不随关卡或菜单态销毁。

---

## 内部架构

四个独立控件，各挂一处，互不依赖：

- `UGlobalHUDOverlayWidget`（ZOrder 801）：存档转圈 + 身份接入提示 + 内存警告，按可见性动态挂/卸音效。
- `UGlobalMenuOverlayWidget`（ZOrder 9001）：游戏邀请列表 + 会话/章节计时器；测试构建下额外挂调试信息控件；数据存于 `UGlobalMenuSingleton`。
- `USkipCutsceneWidget`：双人跳过过场的左右进度条（材质参数驱动）。
- `UDebugInfoWidget`：Ping/丢包/进度点/构建号 + 测试构建下的最新存档/过场信息（仅 `Debug::AreOnScreenMessagesEnabled()` 时显示）。

---

## 关键文件 / 控件

| 文件 | 类 : 基类 | 职责与要点 |
| --- | --- | --- |
| `GlobalHUDOverlay.as` | `UGlobalHUDOverlayWidget : UHazeUserWidget` + `UIdentityEngagementWidget` | **存档转圈 + 身份接入**。Tick 里：游戏开局稳定判定（`GameStartedTimer>4s`）后，据 `Save::HasRecentlySaved()`+档案脏标(`IsAnyProfileDirty`)驱动存档转圈（`SaveSpinner`，动态材质）；据 `Online::PrimaryIdentity`/各玩家身份的 `Engagement` 状态显隐接入提示（菜单态 / Mio / Zoe 三个 `UIdentityEngagementWidget`）；按可见性挂/卸 SoundDef；`#if TEST` 下显内存预算警告。`CVar Haze.HideSavingWidget` 可隐藏转圈 |
| `GlobalMenuOverlay.as` | `UGlobalMenuOverlayWidget : UHazeUserWidget` + `UGlobalMenuSingleton : UHazeSingleton` + `UGameInviteWidget` | **游戏邀请 + 计时器**（ZOrder 9001）。`UpdateInviteWidgets` 依 `Online::GetReceivedGameInvites` 增删邀请条目控件，被触发的邀请弹 `MessageDialog` 询问接受/拒绝；`UpdateTimer` 累计会话/当前章节/上一章节用时，`bDisplayTimer` 时格式化显示。单例 `UGlobalMenuSingleton` 存计时/章节状态，含 `NetResetTimers`。`UGameInviteWidget` 呈现单条邀请（好友名/平台图标/接受·拒绝 `UMenuPromptOrButton`/剩余时间进度条） |
| `SkipCutsceneWidget.as` | `USkipCutsceneWidget : UHazeSkipCutsceneTwoPlayersWidget` | **双人跳过过场进度条**。Tick 里把 `LeftProgressValue`/`RightProgressValue` 写入左右 `UImage` 动态材质的 `StartPercentage`/`EndPercentage`，从中心向两侧填充 |
| `DebugInfoWidget.as` | `UDebugInfoWidget : UHazeUserWidget` | **屏上调试信息条**。每 3 秒刷新：网络 Ping/丢包（`Network::IsGameNetworked`）、当前进度点+关卡组、构建号（`FHazeBuildInfo`）；非编辑器显帧号（`UHazeTemporalLog`）；`#if TEST` + CVar `Haze.ShowLatestSave`/`Haze.ShowPlayingCutscene` 显最新存档/跳过目标/当前过场。仅 `Debug::AreOnScreenMessagesEnabled()` 时可见 |

---

## 复用关系

- 邀请接受/拒绝确认走 [MessageDialog](../MessageDialog/README.md)。
- 存档转圈复用 `Menus/` 的 `USpinnerWidget`；邀请按钮复用 `UMenuPromptOrButton`。
- 计时器/章节数据依赖 `Progress::` / `Save::` / `UHazeChapterDatabase`。

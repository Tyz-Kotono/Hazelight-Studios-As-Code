# Core / Flow / Fades（屏幕淡化）

> 职责：管理每名玩家的屏幕淡入淡出叠加色，并统一处理加载屏的全屏淡黑与过渡时长，使关卡切换、过场、加载之间的画面衔接平滑。共 3 个文件。

## 内部架构

### 每玩家淡化管理
- **`UFadeManagerComponent`（UHazeFadeManagerComponent）**：挂在玩家（或菜单相机用户）上，逐帧合成屏幕叠加色。`NotPlaceable`、暂停时仍 Tick。
  - 活动淡化列表 `TArray<FCurrentFade>`：每项含 `Duration`、`FadeOutTime`、`FadeInTime`、`FadeColor`、`Priority`、`Instigator`、计时 `Time`。
  - **合成规则**：每个淡化各自向目标 alpha 混合，取所有淡化中**最高** alpha 与对应颜色；再叠加关卡序列淡化（`AHazeLevelSequenceActor::GetFadeSettingsForFadeManagerComponent`，取更高者）。
  - **加载屏**：`Game::IsInLoadingScreen()` 时强制全黑（受 CVar `Haze.FadeOutOnLoadingScreen` 控制），并保留若干额外帧（`EXTRA_LOADING_SCREEN_FADE_FRAMES`）以待场景就位，再以 `LOADING_SCREEN_FADE_IN_LENGTH` 淡入。
  - 结果写回 `SceneView::SetPlayerOverlayColor(OwningPlayer, ...)`（或菜单相机的 `SetFadeOverlayColor`）。
  - 负 `Duration` 表示无限期淡化，直到 `ClearFade(Instigator)` 清除（按 instigator 匹配）。

### 加载屏单例
- **`UFadeSingleton`（UHazeSingleton）**：跨玩家协调加载屏淡化。
  - 记录上次淡化色（供下个加载屏复用）、是否在加载屏前已淡出（`bWasFadedOutBeforeLoadingScreen`）。
  - 进入加载屏时按情况调 `Progress::SetMinimumLoadingScreenDuration`：若未提前淡出则强制至少 CVar `Haze.MinimumLoadingScreenTime` 时长，避免黑屏闪烁；已淡出或有加载过渡则可为 0。
  - 维护 `NextLoadingScreenMinimumDuration`（外部可指定下个加载屏最短时长）与活动加载过渡列表。

### 入口
- **FadeStatics**：便捷函数，多数经 `UFadeManagerComponent::GetOrCreate(Player)` 落地。
  - 单玩家：`FadeOut` / `FadeToColor` / `ClearFade`（mixin，玩家上）。
  - 全屏：`FadeOutFullscreen` / `FadeFullscreenToColor` / `ClearFullscreenFade`（遍历所有玩家，用 `Fullscreen` 优先级覆盖个体淡化）。
  - 加载屏：`SetMinimumDurationForNextLoadingScreen` 写入单例。

### 数据流
关卡/系统调 FadeStatics → `UFadeManagerComponent` 加入/清除淡化 → 每帧 `UpdateFades` 合成最高 alpha + 加载屏强制 → 写回玩家叠加色；`UFadeSingleton` 旁路决定加载屏时长与复用淡化色。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 取用 | `UFadeManagerComponent` | 直接调用 `GetOrCreate` | FadeStatics 全部 `UFadeManagerComponent::GetOrCreate(Player)` 后调 `AddFade`/`ClearFade` |
| 调用 | SceneView | 直接调用 | 结果经 `SceneView::SetPlayerOverlayColor` / `GetPlayerOverlayColor` 应用 |
| 叠加 | 关卡序列 | 直接调用 | 合成时取 `AHazeLevelSequenceActor::GetFadeSettingsForFadeManagerComponent(this)` 更高者 |
| 协调 | Progress 加载屏 | 直接调用 | `UFadeSingleton` 调 `Progress::SetMinimumLoadingScreenDuration(...)` |
| 被使用 | PlayerHealth/Sequencer 等 | 直接调用 | `PlayerRespawnCapability`、`DevCutscene` 等多处调用 Fade 静态函数（grep 命中 21 文件） |

## 关键文件

- `FadeManagerComponent.as`：`UFadeManagerComponent`，每玩家屏幕淡化合成与加载屏强制淡黑。
- `FadeSingleton.as`：`UFadeSingleton`，加载屏淡化色复用与最短时长协调。
- `FadeStatics.as`：淡化入口函数（单玩家 mixin、全屏、下个加载屏最短时长）。

# Core / Flow / Tutorial（教学提示）

> 职责：为玩家显示按键/摇杆等操作教学提示，支持屏幕固定提示、世界空间悬浮提示、提示链与取消提示，由教学体积或自定义教学能力驱动。共 8 个文件。

## 内部架构

### 数据中枢
- **`UTutorialComponent`（UActorComponent）**：挂在玩家上的提示状态中枢。维护四类活动提示：
  - `ActiveTutorials`（`FActiveTutorial`）：屏幕提示，可设 `MaximumDuration` 自动消失。
  - `ActiveChains`（`FActiveTutorialChain`）：提示链，含 `ChainPosition` 当前激活位置。
  - `WorldPrompts`（`FActiveTutorial`）：世界空间提示，挂到 `USceneComponent` 上跟随。
  - `CancelPrompts`（`FCancelPrompt`）：取消提示，显示在 HUD 取消槽，支持自定义文本与分屏/全屏切换。
  - 以 `NextTutorialId` 自增分配 ID；以 `FInstigator` 为键添加/移除/改状态；无任何提示时关闭 Tick。

### 展示层
- **`UTutorialWidgetsCapability`（UHazePlayerCapability）**：`Local` 网络模式、`Tutorial` 标签。当 `TutorialComponent.bHasTutorials` 时激活，每帧把组件中的提示同步到 `UTutorialContainerWidget`（增删提示/链、设状态、`RemoveWhenPressed` 模式按键消除）。
- **`UTutorialContainerWidget` / `UTutorialWidget`**：容器与单条提示控件。
- **`UCancelPromptWidget`（CancelPromptWidget.as）**：取消提示控件。

### 数据结构
- **`FTutorialPrompt`**：单条提示定义——`Action`、`Text`、`DisplayType`（动作/按住/释放、左右摇杆各方向/旋转/按压、纯文本等）、`AlternativeDisplayType`（键鼠）、`Mode`（Default / RemoveWhenPressed）、`MaximumDuration`、`OverrideControlsPlayer`。
- **`FTutorialPromptChain`**：`Prompts` 数组 + 连接类型（Plus/Arrow）。

### 入口
- **TutorialStatics（mixin）**：玩家上的便捷函数，全部经 `UTutorialComponent::Get(Player)` 落地：`ShowTutorialPrompt`、`ShowTutorialPromptChain`、`ShowTutorialPromptWorldSpace`、`SetTutorialPromptState`、`SetTutorialPromptChainPosition`、`RemoveTutorialPromptByInstigator`、`ShowCancelPrompt(WithText)`、`RemoveCancelPromptByInstigator`。

### 触发载体
- **`ATutorialVolume`（APlayerTrigger）**：进入时按 `VolumeType` 行为——`PromptOnce`（每玩家仅一次，`RemoveWhenPressed`）、`PromptWhileInVolume`（在内持续显示）、`UseTutorialCapability`（请求自定义教学能力）。可将提示挂到触发玩家或指定演员上世界空间显示。
- **`UTutorialCapability`（UHazePlayerCapability, Abstract）**：自定义教学逻辑基类，`Crumb` 网络模式。通过 `IsOverlappingTutorialVolume()` 判断玩家是否在引用本能力类的 `ATutorialVolume` 内决定激活，`SetTutorialCompleted()` 完成后不再激活。

### 数据流
关卡/能力调用静态函数 → `UTutorialComponent` 存入活动提示 → `UTutorialWidgetsCapability` 每帧同步到 Widget → 玩家按键或超时移除。世界提示由组件自行创建 `WorldPromptWidget` 并挂到目标组件。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
|---|---|---|---|
| 取用 | `UTutorialComponent` | 直接调用 `Type::Get` | TutorialStatics 全部 `UTutorialComponent::Get(Player)` 后调 `AddTutorial`/`AddWorldPrompt` 等 |
| 继承 | Triggers/`APlayerTrigger` | 继承 | `ATutorialVolume : APlayerTrigger`，复用进入/离开 |
| 调用 | 教学静态函数 | 直接调用 | `ATutorialVolume.ShowTutorial` 调 `Player.ShowTutorialPrompt(World)` / `RemoveTutorialPromptByInstigator` |
| 持有 | `UHazeRequestCapabilityOnPlayerComponent` | 请求组件 | `UseTutorialCapability` 模式下请求/停止教学能力 |
| 读取 | `UTutorialComponent.bHasTutorials` / `ActiveTutorials` | 直接读 | `UTutorialWidgetsCapability` 据此激活并构建 Widget |
| 适配 | SceneView 分屏 | 直接调用 | 取消提示按 `SceneView::IsFullScreen()` / `FullScreenPlayer` 切换显示屏幕 |

## 关键文件

- `TutorialComponent.as`：`UTutorialComponent`，教学提示状态中枢（屏幕/世界/链/取消四类）。
- `TutorialWidgetsCapability.as`：`UTutorialWidgetsCapability`，每帧把组件提示同步到容器 Widget。
- `TutorialPrompt.as`：`FTutorialPrompt` / `FTutorialPromptChain` 与各显示类型枚举。
- `TutorialStatics.as`：玩家 mixin 入口函数，提示的显示/移除/状态设置。
- `TutorialVolume.as`：`ATutorialVolume`，进入体积触发三种教学模式。
- `TutorialCapability.as`：抽象教学能力基类，按是否重叠教学体积激活。
- `TutorialWidget.as`：容器/单条提示控件 `UTutorialContainerWidget` 等。
- `CancelPromptWidget.as`：`UCancelPromptWidget` 取消提示控件。

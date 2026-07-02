# Core / Systems / WidgetPool

> 职责：UI 控件池化——按类型复用 `UPooledWidget`，支持取用/归还与「单帧控件」自动回收。共 2 文件。

## 内部架构

### 关键类型

| 类型 | 基类 / 形态 | 角色 |
| --- | --- | --- |
| `UWidgetPoolComponent` | `UActorComponent`（`TG_PostUpdateWork` tick） | 按 `UClass` 维护多个控件池，提供取用/归还/单帧取用 |
| `UPooledWidget` | `UHazeUserWidget`（Abstract） | 可池化控件基类，带池内状态与取用/归还回调 |
| `FWidgetPool` | struct | 单一类型的池：`AvailableWidgets`、`FrameActiveWidgets`、`LastPoolActivity` |

### 数据流

- **取用 `TakeWidgetFromPool(Class, Instigator)`**：从 `AvailableWidgets` 找一个未在使用、未在「延迟移除动画中」（或允许同 instigator 复用）的控件，标记 `bIsInPool=false`、记录 `PoolFrameUsage`、触发 `OnTakenFromPool`；池空则 `Widget::CreateUserWidget` 新建。
- **归还 `ReturnWidgetToPool(Widget)`**：放回 `AvailableWidgets`，`OnReturnedToPool`，若仍在渲染则从玩家或全屏层移除。
- **单帧控件 `TakeSingleFrameWidget(Class, Instigator)`**：按 `PooledInstigator` 复用同一控件并刷新 `PoolFrameUsage`；不存在则取用并加入 `FrameActiveWidgets`，可选 `AddExistingWidget` 到玩家。**不可手动归还**——`Tick` 中凡 `PoolFrameUsage < GFrameNumber`（上一帧未再请求）即自动归还。
- **闲置回收**：`Tick` 检测 `LastPoolActivity` 超过约 60×60 帧未活动时逐个删控件，池空则删除整池。

## 协同关系

| 方向 | 对象 | 机制 | 说明（grep 实证） |
| --- | --- | --- | --- |
| 被调用 | `Targetable/PlayerTargetablesComponent.as` | `GetOrCreate` + 单帧控件 | `UWidgetPoolComponent::GetOrCreate(TargetingPlayer)` 后 `TakeSingleFrameWidget(WidgetClass, WidgetInstigator, bAddToPlayer=false)` 取目标标记控件 |
| 被继承 | `Targetable/TargetableWidget.as`（`UTargetableWidget`） | 派生 `UPooledWidget` | 目标标记 UI 作为可池化控件 |
| 依赖 | `FInstigator` | 标识键 | 单帧控件以 instigator 区分复用，与全项目 instigator 体系一致 |
| 依赖 | `AHazePlayerCharacter`（`AddExistingWidget` / `RemoveWidget`）、`Widget::CreateUserWidget` / `RemoveFullscreenWidget` | 直接调用 | 控件挂载/卸载 |

## 关键文件

- `WidgetPoolComponent.as` — `UWidgetPoolComponent` 与 `FWidgetPool`：按类型池化控件，取用/归还、单帧控件自动回收、闲置池清理。
- `PooledWidget.as` — `UPooledWidget` 抽象基类：池内状态（`bIsInPool` / `PooledInstigator` / `PoolFrameUsage`）与 `OnTakenFromPool` / `OnReturnedToPool` 钩子。

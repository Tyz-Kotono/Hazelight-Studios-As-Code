# Core / SequencerAndTime / TimeDilation

> 职责：聚合式慢动作系统——收集所有时间缩放请求，取最低倍率施加到世界或单个 Actor，支持 blend in/out 与自动到期。共 2 文件。

## 内部架构

- **`FTimeDilationEffect`**（数据结构，定义于 Statics）：单个效果的配置。
  - `TimeDilation`：目标倍率（<1 即慢动作）。
  - `BlendInDurationInRealTime` / `BlendOutDurationInRealTime`：用真实时间做缓入缓出。
  - `MaxDurationInRealTime`：到期自动结束（负值表示需显式停止）。

- **`UTimeDilationEffectSingleton`**（`UHazeSingleton`）：运行期聚合器。
  - 内部维护 `TArray<FActiveTimeDilation>`，每个活动效果记录 Effect / Actor / Instigator / Timer / 是否正在 blend out / 是否世界级。
  - `FActiveTimeDilation::GetWantedTimeDilation()`：按计时器在 1.0 与目标倍率之间 `Math::Lerp`，得到当前应用倍率。
  - `Tick`：用 `Time::UndilatedWorldDeltaSeconds` 推进所有计时器（不受自身影响），处理自动到期/blend out 移除，再调 `GetWantedWorldTimeDilation()` **取所有世界级效果的最低倍率**，仅在变化时 `Time::SetWorldTimeDilation`；Actor 级效果逐个 `SetActorTimeDilation`。
  - `GetWantedWorldTimeDilation()`：遍历取 `LowestDilation`（最低倍率优先 = 最慢者胜）。
  - `ResetStateBetweenLevels`：清空并复位世界倍率为 1.0。
  - 暂停时（`Game::IsPausedForAnyReason()`）整体跳过。

### 数据流
请求方 → `TimeDilation::Start*`（Statics）→ 单例 `StartTimeDilationEffect` 入列 → 每帧 Tick 聚合最低倍率 → `Time::SetWorldTimeDilation` / `Actor.SetActorTimeDilation`。停止时按 blend out 时长平滑退出。

## 协同关系（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
|------|------|------|------|
| 入口 | `TimeDilation::StartWorldTimeDilationEffect` / `StopWorldTimeDilationEffect` | 直接调用 `Get` | 命名空间函数内部 `UTimeDilationEffectSingleton::Get()` 后转发（grep: `TimeDilationStatics.as`） |
| 入口 | `mixin StartActorTimeDilationEffect` / `StopActorTimeDilationEffect` | mixin + 直接调用 `Get` | 任意 `AHazeActor` 可点语法申请 Actor 级慢动作 |
| 调用 | 引擎 `Time::` 命名空间 | 直接调用 | `SetWorldTimeDilation`、`UndilatedWorldDeltaSeconds` |
| 调用 | 引擎 `AHazeActor` | 直接调用 | `SetActorTimeDilation` / `ClearActorTimeDilation(Instigator)` |
| 去重键 | `FInstigator` + `Actor` | 可组合设置 | 以 (Instigator, Actor) 为唯一键，重复 Start 先停旧的；多请求按最低倍率叠加 |
| 关联 | `ForceFeedback`（同 Core） | 参数解耦 | FF 的 `bIgnoreTimeDilation` 用于让震动强度绕过时间缩放 |

## 关键文件

| 文件 | 一句话 |
|------|--------|
| `TimeDilationStatics.as` | 定义 `FTimeDilationEffect` 与 `TimeDilation::` 世界级入口 + Actor 级 mixin 入口 |
| `TimeDilationEffectSingleton.as` | 聚合单例：入列、计时、取最低倍率、blend in/out、自动到期、应用到世界/Actor |

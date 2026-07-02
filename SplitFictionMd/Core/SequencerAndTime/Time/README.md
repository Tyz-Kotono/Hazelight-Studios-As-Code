# Core / SequencerAndTime / Time

> 职责：开发期专用——通过键盘/手柄按键与 Dev Toggle 切换“调试用”世界时间倍率，方便测试时加速/减速游戏。共 2 文件。

## 内部架构

注意：本模块走的是引擎的 **`Time::SetWorldDebugTimeDilation`** 调试通道，与 TimeDilation 子模块的正式慢动作系统相互独立。所有逻辑包裹在 `#if !RELEASE` 中。

- **`UTimeDilationComponent`**（`UActorComponent`，附着于 `Game::Mio`）：
  - 持有一组离散倍率档位 `DilationValues`（0.0001 … 1.0 … 16.0），默认索引 6（=1.0）。
  - `ChangeTimeDilation(Delta)`：在档位间移动并 `Time::SetWorldDebugTimeDilation`，非 1.0 时启用 Tick 屏显当前倍率。
  - `ChangeToMin/MaxTimeDilation`：跳到最慢/最快档。
  - `BeginPlay` 绑定 `DevTogglesTimeDilation::ToggledTimeDilation` 的变更回调，`SnapToToggle` 把 Toggle 选项映射到档位。
- **`UTimeDilationIncreaseDevInput` / `UTimeDilationDecreaseDevInput`**（`UHazeDevInputHandler`）：注册“Faster / Slower Time Dilation”开发输入（手柄上/下面键、+ / -），`Trigger` 时对 Mio 的组件 `ChangeTimeDilation(±1)`，`GetStatus` 在 HUD 显示当前倍率与颜色。
- **`DevTogglesTimeDilation`**（命名空间，`TimeDilationDevToggles.as`）：定义 Dev Toggle 分类/分组与 5 个选项（0.001 / 0.5 / 1.0 / 2.0 / 16.0），`1.0` 为默认。

### 数据流
开发输入按键 / Dev Toggle 切换 → `UTimeDilationComponent` 改档 → `Time::SetWorldDebugTimeDilation`。

## 协同关系（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
|------|------|------|------|
| 调用 | 引擎 `Time::SetWorldDebugTimeDilation` / `GetWorldTimeDilation` | 直接调用 | 走调试时间通道，不经正式 TimeDilation 单例 |
| 委托 | `DevTogglesTimeDilation::ToggledTimeDilation` | 委托（BindOnChanged） | 组件订阅 Toggle 变更并同步档位 |
| 关联 | `Game::Mio` | 直接调用 `GetOrCreate` | 组件与开发输入均作用于 Mio 上的 `UTimeDilationComponent` |
| 隔离 | TimeDilation 子模块 | 通道隔离 | 本模块用 Debug 倍率，正式慢动作用 `SetWorldTimeDilation`，互不干扰 |

## 关键文件

| 文件 | 一句话 |
|------|--------|
| `TimeDilationDevInput.as` | 调试时间倍率组件（档位表）+ 加速/减速开发输入处理器 |
| `TimeDilationDevToggles.as` | 定义时间倍率 Dev Toggle 分类与 5 个倍率选项 |

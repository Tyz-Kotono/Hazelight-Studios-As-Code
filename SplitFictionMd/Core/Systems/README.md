# Core / Systems

> 职责：支撑全项目的基础设施系统——联机所有权与确定性同步、编辑器预制体、可组合设置应用、UI 控件池化、跨世界 actor 链接、环境道具批量摆放。共 6 子模块（28 文件）。

## 概览

本功能域是「机制底座」：其中 **Network 是协作 Crumb 机制的实现基础**，其余子模块或建立在确定性同步之上（WidgetPool 借 `FInstigator`、Network 借 Crumb），或服务于编辑器内容生产（Prefab / Props / WorldLink），或承载游戏配置（Settings）。

| 子模块 | 文件数 | 形态 | 一句话 |
| --- | --- | --- | --- |
| `Network/` | 4（含 `Debug/`） | Component / CrumbTrail | 每玩家互斥锁与所有权转移、自适应 Crumb 缓冲与预测 |
| `Prefab/` | 7 | Actor / Asset / Subsystem | 编辑器预制体：数据采集、应用、编辑、合并网格 |
| `Settings/` | 2 | Component / Applicator | `BeginPlay` 应用可组合设置、音频设备设置应用 |
| `WidgetPool/` | 2 | Component / Widget | UI 控件池化（取用/归还/单帧控件） |
| `WorldLink/` | 3 | Component / Actor | 编辑器中跨双世界 actor 互链与同步定位 |
| `Props/` | 10 | Actor / Component / Subsystem | 沿样条/网格批量摆放环境网格 |

## 协作五机制中的角色

项目协作约定五种机制，本域是其中两种的**实现底座**：

| 机制 | 在本域的体现 |
| --- | --- |
| **2. Crumb（`HasControl` + `CrumbFunction`）** | `Network/` 全面承载：`UNetworkLockComponent` 用 `NetFunction` + `CrumbFunction`（`CrumbAllowLocking`）保证「Acquire→Crumb→Release」操作两端确定性；`UNetworkCrumbTrail` 实现自适应 crumb 缓冲与时间预测，是 crumb 流回放的引擎层支撑 |
| **5. 可组合设置（`ApplySettings`）** | `Settings/UDefaultSettingsComponent` 在 `BeginPlay` 遍历 `UHazeComposableSettings` 数组调用 `ApplyDefaultSettings`；`UGameSettingsApplicator` 把音频设置应用为全局 RTPC |
| **3. 标签阻塞** | `Props/UTagContainerComponent` 等以标签容器组织生成物（偏内容侧） |
| **4. 委托** | `UNetworkLockComponent` 的 `FNetworkLockDelegate`（仅在控制侧执行）、`APropLine` 的 `FPropLineUpdatedEvent` |
| **1. 直接调用 `Type::Get`** | `UNetworkLockComponent::GetOrCreate`、`UWidgetPoolComponent::GetOrCreate` 等被消费者直接获取 |

> **双人语境（Mio / Zoe）**：互斥锁、所有权转移、crumb 预测都以「两位玩家、一方有 `WorldControl`」为前提；`Network/Debug` 的视图切换甚至直接按 `Game::Mio` / `Game::Zoe` 命名。

## 子模块

- [`Network/`](Network/README.md) — 互斥锁 + Crumb 自适应同步（**Crumb 机制底座**）
- [`Prefab/`](Prefab/README.md) — 编辑器预制体系统
- [`Settings/`](Settings/README.md) — 可组合设置应用
- [`WidgetPool/`](WidgetPool/README.md) — UI 控件池化
- [`WorldLink/`](WorldLink/README.md) — 跨世界 actor 链接
- [`Props/`](Props/README.md) — 环境道具批量摆放

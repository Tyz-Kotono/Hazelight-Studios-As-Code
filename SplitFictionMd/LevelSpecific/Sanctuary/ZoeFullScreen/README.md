# LevelSpecific / Sanctuary / ZoeFullScreen（全屏演出段）

> 路径：`LevelSpecific/Sanctuary/ZoeFullScreen/`（86 文件）

## 职责

圣域的招牌演出段。常态双人分屏在此合并为**单人全屏**：一名玩家（多为 Mio）操纵整个世界（倾斜/旋转/转向/推进），另一名玩家（Zoe）在被操纵的世界里移动求生。共 5 套子玩法：倾斜世界、旋转世界、转段推进、冰湖风走、光群飞鸟——每套都是"Mio 控世界 / Zoe 应对"的非对称协同。

## 内部架构

- **全屏视点接管**：`UPlayerFullScreenCapability`（`OnActivated` → `Player.ApplyViewSizeOverride(this, EHazeViewPointSize::Fullscreen, Instant)`，退出 `ClearCameraSettingsByInstigator`）。`MioFullScreenComponent` 提供输入工具（`MioFullScreen::GetStickInput`），给双方菜单注册。`DisablePlayerCapability` 演出期屏蔽常规玩家控制。
- **五套子玩法**（均为 Mio 控制端 + Zoe 应对端 + 场景 Actor 三段式）：

| 子玩法 | 目录 | Mio 端 | Zoe 端 / 场景 |
|---|---|---|---|
| **倾斜世界** | `TiltingWorld/` | `MioTiltCapability`（摇杆倾斜世界）+ `MioTopDown` | `ZoePlayerAlignWithTilt`（对齐倾斜）+ `ZoeCamera`；场景 `TiltingWorldActor` + 吊灯/画/火把/伪物理 Actor |
| **旋转世界** | `SpinningWorld/` | `MioSpinCapability`（摇杆旋转） | `ZoePlayerAlignWithSpin` + `ZoeCamera` |
| **转段推进** | `TurnSegments/` | `MioTurnCapability`（转向选段） | `ZoeCamera`；场景 `TurnSegmentsActor` + Box/Hexagon + 样条生成器 + `TurnSegmentConstraint` |
| **样条弯道** | `SplineCorridorBend/` | `MioCapability`（弯折走廊） | `ZoeCapability`；`SplineCorridorBendActor` + 碰撞组件 + 样条生成 |
| **光群飞鸟** | `LightCrowd/` | `LightCrowdBirdCapability` + `FullScreenCapability` | 代理 `Agent/`（Spawn/Active/Movement/Deactive/Despawn）；`LightCrowdAgent` + `BirdComponent` |

- **冰系子段** `IceFractal/`（冰晶：移动平面 + 相机 + 生成器 + 玩家组件）与 `IceLake/`（冰湖：风走 WindWalk 移动/动画/力 + 风向 WindDirection 系统 + 木筏 Raft / 桅杆 / 沉船 / 冰面 Actor）——玩家在结冰湖面靠风力行走、乘筏。
- **组件模式**：每套子玩法配 `MioComponent` / `ZoeComponent` / `DataComponent` / `ResponseComponent` + `Settings`，Mio 能力从自身 `Get`、从 Zoe `GetOrCreate` 对端组件读写共享状态（如 `TiltingWorldMioCapability` 同时取 `MioComponent` 与 `ZoeComponent`）。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| 依赖 | Core 相机 | 全屏视点 | `PlayerFullScreenCapability` → `ApplyViewSizeOverride(EHazeViewPointSize::Fullscreen)` |
| 协同 | Mio ↔ Zoe | 双端组件共享 | `TiltingWorldMioTiltCapability` 取 `MioComponent::Get` + `ZoeComponent::GetOrCreate(Game::GetZoe())` |
| 依赖 | Core 输入 | 摇杆输入 | `MioFullScreen::GetStickInput(this)` 驱动倾斜/旋转/转向 |
| → | 玩家控制 | 演出屏蔽 | `DisablePlayerCapability` 全屏段屏蔽常规移动 |
| 依赖 | Core 能力四件套 | Capability/Component/Settings | 五套子玩法结构一致 |

## 关键文件

- `LevelSpecific/Sanctuary/ZoeFullScreen/PlayerFullScreenCapability.as` — 全屏视点接管
- `LevelSpecific/Sanctuary/ZoeFullScreen/MioFullScreenComponent.as` — Mio 控制端输入工具
- `LevelSpecific/Sanctuary/ZoeFullScreen/TiltingWorld/Capabilities/Mio/TiltingWorldMioTiltCapability.as` — 倾斜世界（双端组件协同范例）
- `LevelSpecific/Sanctuary/ZoeFullScreen/SpinningWorld/Capabilities/Mio/SpinningWorldMioSpinCapability.as` — 旋转世界
- `LevelSpecific/Sanctuary/ZoeFullScreen/TurnSegments/` — 转段推进
- `LevelSpecific/Sanctuary/ZoeFullScreen/IceLake/` — 冰湖风走
- `LevelSpecific/Sanctuary/ZoeFullScreen/LightCrowd/LightCrowdAgent.as` — 光群飞鸟代理

# LevelSpecific / Battlefield / Hoverboard（悬浮板载具）

> 路径：`LevelSpecific/Battlefield/Hoverboard/`（133 文件）

## 职责

Battlefield 的核心载具与移动系统。两名玩家骑悬浮板高速穿越战区，能在地面漂移、腾空、沿轨道磨轨（Grind）、贴墙跑（Wallrun）、用抓钩（Grapple）拉近或荡向锚点、荡绳（Swing）、跳跃、做特技（Trick）刷分，并有专门的相机、竞速橡皮筋、终点线减速、自由落体等一整套子系统。这是整关文件量最大、最复杂的机制，等价于一辆滑板车的完整"驾驶物理 + 花式动作 + 竞速平衡"栈。

## 内部架构

### 载具本体与中枢

- **实体** `ABattlefieldHoverboard : AHazeActor`：网格 + 碰撞 + `ListedActorComp` 的表现载体，持有 `Player` 与 `HoverboardComp` 回指。逻辑不在实体上。命名空间 `Hoverboard::` 提供 `GetHoverboard(Player)` / 竞速胜负 `GetBattlefieldWinnerState`（`EBattlefieldWinState`）等蓝图接口。
- **玩家组件** `UBattlefieldHoverboardComponent`：状态中枢，集中持有全部子系统的 Settings（Ground/Air/Grind/Jump/Swing/Grapple/WallRun/Camera…）与运行态（`WantedRotation` / `AccRotation` / 完成时间 / 各类开关）。子系统能力都从它取数据。
- **骑乘入口** `UBattlefieldHoverboardRidingCapability`：`Setup` 时 `SpawnActor` 出悬浮板、挂到玩家 `LeftHand_IK` 骨、注册传送响应；`OnRemoved` 销毁。常驻不退出。
- **自定义解算** `Resolver/BattlefieldHoverboardMovementResolver : USteppingMovementResolver`：重写 `ProjectMovementUponImpact`，让高速载具在斜坡/边缘重定向速度而不掉速、不被吸下地。

### 移动子系统（每套 = Component + Capabilities + Settings）

- **GroundMovement / AirMovement**：地面与空中基础移动（`Grounded / GroundMovement / AirMovement / GroundImpactFloatiness`）。
- **Grinding** `Grinding/`（最大子系统）：沿 `UBattlefieldHoverboardGrindSplineComponent` 磨轨；含进入/吸附（`SuctionToGrind`）、跳到轨道（`JumpToGrind`）、轨间跳（`JumpBetweenGrinds`）、离轨跳、轨上橡皮筋、以及 `Balancing/` 平衡小游戏（平衡条 Widget + 相机倾斜）。
- **Wallrun** `Wallrun/`：贴墙跑，含评估（`Evaluate`）、跳（`WallRunJump`）、墙间转移（`Transfer`），一组 `Settings/`。
- **Grapple** `Grapple/`：抓钩，含进入/发射/勾向磨轨点/勾接跑墙/快速抓钩（`QuickGrapple`）；`GrappleToGrindPoint` 是可抓的锚点。
- **Swinging** `Swinging/`：荡绳，含绳物理（`SwingRope`）、移动、跳、取消、冲击、相机。
- **Jump** `Jump/`：跳跃四件套。
- **Strafing** `Strafing/`：横移。
- **Loop** `Loop/`：环形轨道（做环）。
- **FreeFalling** `FreeFalling/`：自由落体段（配激活体积）。
- **SplineBasedTurning**：沿样条强制转向。

### 特技与 UI `Tricks/`

`TrickComponent` + `Trick / TrickCombo / TrickBoost / TrickGroundValidation` 能力实现腾空刷分连击；`TrickList` / `TrickAnimationData` 数据；`RampJumpVolume` / `TrickVolume` 触发体积。`Tricks/UI/` 一整套 HUD：连击分（Combo）、历史（History）、总分（Total）、死亡扣分（Death）。

### 竞速平衡与收尾

- **LevelRubberbanding** `LevelRubberbanding/`：沿关卡样条把落后玩家拉近（`LevelRubberbandingSplineComponent` + 设置体积 `RubberbandSettingsVolume` / `PreferredAheadPlayerSettingVolume`）。
- **FinishLineStopping** `FinishLineStopping/`：终点线减速停止（`FinishLineManager`）。
- **GoingBackwardsSnipe**：玩家倒退时的狙击惩罚段。
- **Camera** `Camera/`：悬浮板专属相机（转向偏移/倾斜/速度特效）。
- **Old** `Old/`：旧版悬浮板实现（已弃用，保留参考）。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| → | Core Move 系统 | 复用 + 覆盖 | `BattlefieldHoverboardMovementResolver : USteppingMovementResolver` 重写 `ProjectMovementUponImpact`；能力全依赖 `UPlayerMovementComponent` |
| → | 移动状态 | 标签互斥 | 能力带 `CapabilityTags::Movement` + `n"BattlefieldGrinding"` / `n"BattlefieldHoverboard"`（见 `BattleFieldHoverboardTags`） |
| → | 联机 | Crumb | `GrindingCapability` 等 `NetworkMode = EHazeCapabilityNetworkMode::Crumb` |
| → | 样条系统 | 磨轨/竞速 | `UBattlefieldHoverboardGrindSplineComponent`（`FSplinePosition`）、`LevelRubberbandingSplineComponent` |
| → | 传送响应 | 复位 | `RidingCapability` 注册 `UTeleportResponseComponent.OnTeleported` → 复位朝向 |
| → | 相机系统 | 直接调用 | `Camera/` 子系统套用 `UHazeCameraSettingsDataAsset` |
| ← | 场景剧本 | 被驱动 | `Scenarios/Loop/BattlefieldLoopSequenceManager` 读 `GrindSplineComponent.OnPlayerStoppedGrinding` 触发剧本；SlowMoGrapple 复用抓钩 |
| ← | 武器系统 | 受威胁 | 玩家在悬浮板上躲避 `WeaponSystems` 的火力（碰撞/死亡走 Trick UI 的 Death） |

## 关键文件

- `Hoverboard/BattlefieldHoverboard.as` — 载具实体 `ABattlefieldHoverboard` + `Hoverboard::` 命名空间（胜负/取用）
- `Hoverboard/BattlefieldHoverboardComponent.as` — 玩家状态中枢，集中持有全部子系统 Settings 与运行态
- `Hoverboard/BattlefieldHoverboardRidingCapability.as` — 骑乘入口：生成实体、挂手骨、常驻
- `Hoverboard/Resolver/BattlefieldHoverboardMovementResolver.as` — 高速载具自定义步进解算
- `Hoverboard/Grinding/Capabilities/BattlefieldHoverboardGrindingCapability.as` — 磨轨核心能力（Crumb）
- `Hoverboard/Grapple/` — 抓钩子系统（8 文件）
- `Hoverboard/Swinging/` — 荡绳子系统（8 文件）
- `Hoverboard/Tricks/` — 特技刷分 + HUD（18 文件）
- `Hoverboard/LevelRubberbanding/` — 双人竞速橡皮筋

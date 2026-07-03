# LevelSpecific / Coast / Train — 火车追逐

> 路径：`LevelSpecific/Coast/Train/`（65 文件）

## 职责

追逐序列的第一段：在一列飞驰的集装箱货运列车顶上进行攀爬与战斗。玩家在移动的车厢间用抓钩穿梭、躲避障碍、摧毁车载炮塔与飞行敌人，同时列车本身作为一组连续移动的平台驱动全程。

## 内部架构

- **列车本体** — `ACoastTrainCart : AHazeActor` 是移动平台车厢基类；`ACoastTrainDriver : ACoastTrainCart` 是带动整列的车头，移动由 `CoastTrainDriverMovementCapability` 驱动。`UCoastTrainRiderComponent : UActorComponent` 把玩家挂接到移动车厢上（继承车厢速度，保证站在车顶不被甩下）。
- **抓钩穿梭** — 玩家在车厢间借助抓钩系统移动：`CoastTrainGrappleToCarCapability` 配合场景中的 `CoastTrainRoofGrapplePoint`、`CoastTrainTwistableGrapplePoint` 等锚点；含坠落/弹射离车/重生处理。
- **车载敌人**：
  - `ACoastContainerTurret : AHazeActor` — 集装箱炮塔，其武器 `ACoastContainerTurretWeapon : ABasicAICharacter` 用复合能力＋攻击/滑动/侦测能力。
  - `ATrainFlyingEnemy : AHazeActor` — 悬停移动、护盾力场、投射能力的飞行敌人。
  - `ATrainGiggaTurret : AHazeActor` — 大型炮塔＋投射攻击。
- **Player/** — 火车段落专属的玩家移动/攀爬/抓钩能力集。

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
|---|---|---|---|
| 依赖 | Gameplay | `UPlayerMovementComponent` | 玩家在移动平台上的底层移动 |
| 依赖 | Gameplay/AI | `ABasicAICharacter` | 炮塔武器、飞行敌人的 AI 基类 |
| 依赖 | Core | `UHazeCrumbSynced*Component` | 列车位置与骑乘状态的联机同步 |
| 复用 | Coast/AI | TrainDrone 等原型 | 车顶飞行敌人来自通用 AI 库 |
| 衔接 | Coast/WingSuit | 飞出列车 | 火车段结束后进入翼装滑翔 |

## 关键文件

- `LevelSpecific/Coast/Train/CoastTrainCart.as` — 车厢移动平台基类
- `LevelSpecific/Coast/Train/CoastTrainDriver.as` — 车头驱动
- `LevelSpecific/Coast/Train/CoastTrainRiderComponent.as` — 玩家挂接组件
- `LevelSpecific/Coast/Train/TrainContainerTurret/` — 集装箱炮塔＋武器＋能力
- `LevelSpecific/Coast/Train/TrainFlyingEnemy/`、`TrainGiggaTurret/` — 飞行敌人与大型炮塔
- `LevelSpecific/Coast/Train/Player/` — 火车段玩家能力

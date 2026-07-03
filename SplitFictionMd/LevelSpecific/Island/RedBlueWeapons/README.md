# LevelSpecific / Island / RedBlueWeapons（红蓝双色武器）

> 路径：`LevelSpecific/Island/RedBlueWeapons/`（95 文件）

## 职责

Island 的核心射击玩法。Mio 持**红枪**、Zoe 持**蓝枪**，双手可各持一把武器，共有 6 档升级形态（单发 / 三连发 / 突击 / 过热突击 / 自动霰弹 / 横版突击）与一套**红蓝粘性手雷**。武器带靶向锁定、过热、护盾颜色互斥、动画/滑铲/摆荡/跑墙状态屏蔽、升级站与横版特化。目录分两大子系统：`RedBlueShooting/`（枪械 + 靶向 + 横版）与 `RedBlueStickyGrenade/`（粘弹）。

## 内部架构

### 枪械 RedBlueShooting

- **武器实体** `AIslandRedBlueWeapon : AHazeActor`（Abstract）：只是网格 + 枪口 `Muzzle` 的表现载体，记录 `WeaponType`(Red/Blue)、`HandType`(Left/Right)、`PlayerOwner`。真正逻辑不在武器上。
- **玩家组件** `UIslandRedBlueWeaponUserComponent`：挂在玩家身上的状态中枢，持有左右手武器数据 `FIslandRedBlueWeaponComponentData`、瞄准数据 `FIslandRedBlueWeaponAnimData`（`AimDirection` 覆盖、每帧开火/掷雷/引爆标志、`bIsOverheated`）、`HealthSettings`。
- **能力分层**（`Capabilities/`）：
  - `_Core/`：跨形态通用——`IslandRedBlueActiveCapability`（激活时 `ApplySettings(HealthSettings)`）、`Targeting` / `ForceTarget` / `FoundTarget`（靶向锁定）、`HoldWeaponInHand` / `ForceHoldWeaponInHand`（持枪）、`WeaponCoolDown`、`BlockCameraAssist`。
  - 每档升级一个子目录 `Single/ Burst/ Assault/ Overheat/ Shotgun/ SideAssault/`，各自成套：`XxxCapability`(状态) + `XxxFireCapability`(开火节奏) + `XxxShootBulletCapability`(生成子弹) + `XxxSettings`。过热档额外有 `Cooldown` / `Camera` / `UserComponent` / `Sidescroller` 特化。
  - `_Block/`：状态互斥屏蔽——动画中、滑铲、摆荡、跑墙、情境时屏蔽武器。
- **子弹与命中**：`IslandRedBlueWeaponBullet` + `IslandRedBlueWeaponBulletEffectHandler` + `IslandRedBlueContinuousBulletCollisionCheckerComponent`（连续碰撞检测）；`Targeting/` 下一族 `IslandRedBlueImpact*ResponseComponent`（普通 / 计数 / 过充 / 护盾 4 种命中响应）+ `IslandRedBlueTargetableComponent`（可锁定目标）+ 反射 `IslandRedBlueReflectComponent`。
- **UI**：`AimCrosshairWidget` / `Overheat2DWidget` / `TargetableWidget`。**升级站** `IslandRedBlueUpgradeStation` 切换武器档位。
- **横版特化** `Sidescroller/`：`SpotlightActor`、`LaserSightCapability`、`SidescrollerTargetingCapability` + 两套教学（跳下 / 瞄准）。

### 粘性手雷 RedBlueStickyGrenade

- **实体** `AIslandRedBlueStickyGrenade` + `EffectHandler`，一族碰撞忽略组件（忽略发射者 / 附着体 / 组件碰撞）、`BounceOffComponent`（弹开）、`KillTrigger`。
- **能力**（`Capabilities/`）：`Active` / `Aiming`（瞄准）/ `Throw` + `ThrowTrigger`（投掷）/ `Detonate`（引爆）/ `Despawn`。响应侧 `ResponseComponent` + `ResponseContainerComponent`，可锁定 `StickyGrenadeTargetable`。带 3 个教学能力。

## 协同关系

| 方向 | 对象 | 机制 | 说明（实证） |
|---|---|---|---|
| → | 玩家身份系统 | 颜色门控 | `IslandRedBlueWeaponTypes.as` `GetPlayerForColor`: Red→`Game::Mio`、Blue→`Game::Zoe` |
| → | Core 瞄准/锁定 | 直接调用 | 全 `ShootBullet` / `Targeting` / `ForceTarget` 能力依赖 `UPlayerAimingComponent`（grep 命中 18 文件） |
| → | Core/Flow 设置系统 | ApplySettings | `IslandRedBlueActiveCapability.OnActivated` → `Player.ApplySettings(HealthSettings, this)`；退出 `ClearSettingsByInstigator` |
| → | 移动状态 | 标签互斥 | `_Block/` 下 Slide/Swing/WallRun/Animation Block 能力屏蔽持枪与开火 |
| → | 力场/护盾机关 | 颜色互斥命中 | `IslandRedBlueImpactShieldResponseComponent` 按 `EIslandRedBlueShieldType`(Red/Blue/Both) 判定可否击破，联动 `RedBlueForceField/` |
| ← | 升级站 | 状态切换 | `IslandRedBlueUpgradeStation` 切换 6 档 `EIslandRedBlueWeaponUpgradeType` |
| → | Core 横版相机 | 复用 | `Sidescroller/` 子系统在横版段套用激光瞄准与聚光灯 |

## 关键文件

- `RedBlueShooting/IslandRedBlueWeapon.as` — 武器实体基类 `AIslandRedBlueWeapon`
- `RedBlueShooting/IslandRedBlueWeaponUserComponent.as` — 玩家状态中枢组件
- `RedBlueShooting/IslandRedBlueWeaponTypes.as` — 颜色/手/升级/护盾枚举 + `GetPlayerForColor` 颜色映射
- `RedBlueShooting/Capabilities/_Core/IslandRedBlueActiveCapability.as` — 激活/设置入口
- `RedBlueShooting/Capabilities/Overheat/` — 最复杂形态（过热突击，8 文件）
- `RedBlueShooting/Targeting/IslandRedBlueImpactShieldResponseComponent.as` — 颜色护盾命中判定
- `RedBlueStickyGrenade/IslandRedBlueStickyGrenade.as` — 粘性手雷实体

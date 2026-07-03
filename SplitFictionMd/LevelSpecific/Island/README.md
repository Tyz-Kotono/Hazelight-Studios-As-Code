# LevelSpecific / Island（科幻堡垒岛）

> 路径：`LevelSpecific/Island/`（1361 文件，科幻 / Mio 线）
>
> Island 是一座敌方海上军事堡垒岛。玩法主线是**载具突进 + 双色射击 + 近战/道具战斗**：玩家骑水上摩托穿越淹水走廊与巨型海怪，用喷气背包在高塔间垂直穿梭，装备红蓝双色武器与破盾/近战/夺枪等多种武器清剿科幻警卫。本层不发明新范式，是 Core/Gameplay 的**能力四件套**（Capability + Component + Settings + Statics）、**标签互斥**、**组件容器**在大量载具与武器玩法上的应用。上层设计见 [Core 设计哲学](../../Core/DesignPhilosophy.md)、[Gameplay 设计哲学](../../Gameplay/DesignPhilosophy.md)。

---

## 目录构成

| 类别 | 目录 | 说明 |
|---|---|---|
| **通用基础设施** | `AI/`(697)、`LevelActors/`(212)、`Audio/`、`Supervisor/` | 敌人 AI、可放置演员、音频、关卡流程编排。沿用 Gameplay 的 AI 框架与 Combat，**本文档一笔带过** |
| **招牌载具** ⭐ | `Jetski/`(83)、`Jetpack/`(31)、`HoverPerch/`(32) | 水上摩托、喷气背包、悬浮踏板/磨轨 |
| **招牌武器** ⭐ | `RedBlueWeapons/`(95)、`ScifiCopsGun/`(32)、`ScifiShieldBusterWeapon/`(18)、`IslandNunchuck/`(23) | 红蓝双色武器、夺取式警枪、破盾武器、双节棍近战 |
| **场景机关** | `ForceFieldWalls/`、`RedBlueForceField/`、`PlayerForceField/`、`ScifiGas/`、`ScifiGravityGrenade/`、`Portal/`、`Rift/`、`PipeSlide/`、`Sidescroller/`、`Stormdrain*` 等 | 力场墙、传送门、下水道旋转平台、横版段等一次性关卡机关 |

---

## 招牌机制分篇

| 机制 | 文件数 | 一句话 | 文档 |
|---|---:|---|---|
| [红蓝武器 RedBlueWeapons](./RedBlueWeapons/) | 95 | Mio=红 / Zoe=蓝 的双色枪械 + 粘弹，多档升级、过热、靶向、横版特化 | [RedBlueWeapons/](./RedBlueWeapons/) |
| [水上摩托 Jetski](./Jetski/) | 83 | 四态（空中/地面/水面/水下）样条载具，波浪、潜水、追赶橡皮筋、海怪剧本 | [Jetski/](./Jetski/) |
| [喷气背包 Jetpack](./Jetpack/) | 31(+32) | 燃料悬浮 + 冲刺 + 相位穿墙；合并 HoverPerch 悬浮踏板磨轨 | [Jetpack/](./Jetpack/) |
| [其它武器 Weapons](./Weapons/) | 73 | 合并夺枪 ScifiCopsGun、破盾 ShieldBuster、双节棍 Nunchuck 三套武器 | [Weapons/](./Weapons/) |

---

## Island 通用范式（三大机制共享，读一处懂全部）

1. **红=Mio / 蓝=Zoe 的颜色身份**
   全层贯穿双色对应关系，武器、力场、护盾都据此门控。`IslandRedBlueWeapon::GetPlayerForColor(Color)` 返回 `Red→Game::Mio / Blue→Game::Zoe`，`GetPlayerColor(Player)` 反向查询。护盾与弹药按颜色互斥：只有匹配颜色的玩家能击破对应护盾。

2. **Player 组件挂载 + 载具/武器按需生成**
   载具与武器不常驻场景，而是把状态存在玩家身上的 `UActorComponent`（`UJetskiDriverComponent`、`UIslandJetpackComponent`、`UIslandRedBlueWeaponUserComponent`、`UScifiPlayerCopsGunManagerComponent` 等），在能力激活时 `SpawnActor` 出实体、`ApplySettings` 施加设置，退出时 `ClearSettingsByInstigator` 清除（instigator 引用计数语义，与 Core/Flow 一致）。

3. **能力四件套 + 标签互斥**
   每套玩法一组 `Capability`（分 Input / Movement / Camera / VisualEvents / Tutorial 子类），配 `Settings`(DataAsset) 与 `Statics`(namespace)。能力用 `CapabilityTags` + `BlockedWhileIn::*` 与移动系统互斥（如 Jetpack 屏蔽 WallRun/Swing/Grapple/Ladder 等所有攀爬移动）。联机统一走 `NetworkMode = Crumb`。

4. **横版段（Sidescroller）复用**
   多套机制带 `Sidescroller/` 子目录（RedBlueWeapons、Jetpack），把同一玩法适配到强制 2.5D 横版镜头段，复用 Core 的 Sidescroller 相机与瞄准约束。

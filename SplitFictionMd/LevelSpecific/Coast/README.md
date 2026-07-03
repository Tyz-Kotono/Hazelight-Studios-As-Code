# LevelSpecific / Coast — 海岸追逐（科幻线）

> 路径：`LevelSpecific/Coast/`（305 文件）
>
> Mio 科幻线的高速追逐关卡：沿海岸线一路从火车顶、翼装滑翔、滑水，直到与空中 Boss（Helios）决战。核心体验是**连续切换的载具/移动形态**——火车攀爬、翼装飞行、水上滑行三段无缝衔接，最后进入弹幕空战 Boss。

本文承接 [LevelSpecific 总览](../README.md) 与 [Gameplay 设计哲学](../../Gameplay/DesignPhilosophy.md)（能力四件套 Capability+Component+Settings+Statics、标签互斥、组件容器）。Coast 不发明新范式，是这套范式在"追逐载具序列"上的应用，哲学不再重复。

---

## 关卡流程骨架

1. **火车（Train）** — 在飞驰的集装箱列车顶上攀爬、用抓钩在车厢间穿梭、摧毁车载炮塔。
2. **翼装（WingSuit）** — 从列车飞出后进入自由滑翔，穿越峡谷与障碍。
3. **滑水（Waterski）** — 翼装落水衔接为拖曳滑水，贴水面高速前进。
4. **Boss** — 空中弹幕战：玩家化身射击无人机，在 2D 约束平面内对抗多阶段 Boss，中途穿插翼装 Boss（Helios）与肩炮火力段落。

三段移动形态通过成对的 `*TransitionCapability` 无缝衔接（WingSuit↔Waterski 互转）。

---

## 招牌机制索引

| 机制 | 文件数 | 一句话职责 | 文档 |
|---|---:|---|---|
| **火车（Train）** | 65 | 飞驰列车顶攀爬、车厢间抓钩穿梭、摧毁车载炮塔 | [Train/](./Train/) |
| **翼装（WingSuit）** | 24+2 | 自由滑翔飞行：加速/俯冲/横滚、抓钩、下坡起飞 | [WingSuit/](./WingSuit/) |
| **滑水（Waterski）** | 22 | 拖曳滑水：贴水面移动、跳跃、浪花碰撞 | [Waterski/](./Waterski/) |
| **Boss + 肩炮** | 96+21 | 空中弹幕 Boss（BigGun/多无人机双形态）、翼装 Boss、肩炮武器 | [Boss/](./Boss/) |

---

## 通用目录（一笔带过，不建分篇）

- **`AI/`（75）** — 通用敌人库。各原型（Bombling、CircuitLocke、Jetski、Poltroon、TrainDrone、WaterJet）继承引擎 `ABasicAI*` 框架（`ABasicAICharacter` / `ABasicAIFlyingCharacter` / `ABasicAIGroundMovementCharacter`），每个原型一个 `*CompoundCapability` 编排 `Behaviours/` 子目录（追击、接敌、攻击、瞄准、受击反应）＋移动/死亡/特效能力＋设置资产。`PlayerCombat/`（`CoastPlayerAIHealthCapability`）与 `Traversal/`（`ATraversalAreaActor`）提供玩家对 AI 的血量与导航衔接。AI 框架本身见 [Gameplay/AI](../../Gameplay/README.md)。

---

## 复用的公共系统

| 方向 | 对象 | 说明 |
|---|---|---|
| 依赖 | `UPlayerMovementComponent` + `USweepingMovementData` | 全部玩家移动能力的底座 |
| 依赖 | `UHazeCrumbSynced*Component` | 载具/移动状态的联机同步 |
| 依赖 | `ABasicAI*` 引擎 AI 框架 | 敌人角色、飞行/地面移动、投射发射组件 |
| 依赖 | `UHazePlayerCapability` / `UHazeCapability` | 能力四件套基类 |

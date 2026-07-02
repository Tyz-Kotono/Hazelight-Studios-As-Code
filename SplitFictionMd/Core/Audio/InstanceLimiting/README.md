# Core / Audio / InstanceLimiting

> 职责：通过可配置的条件资产限制同一 SoundDef 的并发播放实例数（按监听距离、最大实例数、点积夹角等条件决定激活/停用/销毁）。共 5 个文件。

## 内部架构（关键类、基类、数据流）

本目录是引擎实例限制框架在 AngelScript 层的**默认数据驱动实现**，分三层：

1. **聚合/管理器层** —— `USoundDefInstanceLimitingDataAsset`（继承自引擎基类 `UHazeSoundDefInstanceLimitingDataAsset`）。持有 `TArray<USoundDefInstanceLimitingCondition> Conditions`，重写两个引擎抽象方法：
   - `ShouldActivateOrDeactivate(EvaluationData, IndexesToRemove)`：返回需要状态切换（激活/停用）的实例索引集合。
   - `ShouldDestroy(EvaluationData, IndexesToRemove)`：返回需要被杀死（Kill）的实例索引集合。
   两者根据 `Behaviour`（Disable / Kill）与 `LimitingType`（Conditional / Newest / Oldest）选择遍历策略（`EvaluateConditional` / `EvaluateNewest` / `EvaluateOldest`），核心判定在 `AnyConditionFailed`：对每个已存在实例逐一询问条件——Active 实例任一条件 `ShouldDeactivate` 为真即标记停用；非 Active 实例需所有条件 `ShouldActivate` 为真才请求重新启用；Kill 行为下任一 `ShouldKill` 为真即标记销毁。

2. **条件基类层** —— `USoundDefInstanceLimitingCondition`（`UCLASS(Abstract)`，继承 `UDataAsset`）。定义三个可扩展虚方法 `ShouldActivate` / `ShouldDeactivate` / `ShouldKill`，默认均返回 `false`，由子类按需重写。参数约定：`EvaluationData` 描述“即将创建的实例”，`SoundDef` 为某个“已存在实例”（创建场景下传 `nullptr`，`Index` 传 `-1`）。

3. **具体条件层（Conditions/）** —— 三个子类资产：
   - `USoundDefInstanceLimiting_ListenersInRange`：以 `MaxDistance` 为半径，检查是否有监听器在范围内。对已存在实例走 `SoundDef.ListenersInRange()`；对待创建实例通过 `Audio::GetListeners` 拉取监听器并与音频组件位置做平方距离比较。
   - `USoundDefInstanceLimiting_MaxInstanceCount`：以 `MaxInstanceCount` 为上限，`ToManyInstances` 用 `当前实例数 + Offset - 待移除数 > 上限` 判定是否超额。
   - `USoundDefInstanceLimiting_WithInDot`：以 `Dot` 阈值判定玩家方向与组件 `ForwardVector` 的点积是否落在夹角内（`OutsideOfDot`），并用 `ClearedSoundDefs` 缓存已放行实例；不支持 Kill（调用即 `devCheck(false)`）。

**数据流**：引擎实例限制系统在 SoundDef 创建/更新时构造 `FHazeSoundDefInstanceEvaluationData` → 调用本资产的 `ShouldActivateOrDeactivate` / `ShouldDestroy` → 资产遍历 `Conditions` 逐条求值 → 返回 `IndexesToRemove`，由引擎据此执行停用/销毁。

## 协同关系

| 方向 | 对象 | 机制 | 说明（基于实证） |
| --- | --- | --- | --- |
| 被调用 | 引擎实例限制系统（C++） | 委托/重写 | 引擎以 `FHazeSoundDefInstanceEvaluationData` 调用本资产重写的 `ShouldActivateOrDeactivate` / `ShouldDestroy`（`UFUNCTION(BlueprintOverride)`），本目录无 AngelScript 主动调用方。 |
| 父→子 | `USoundDefInstanceLimitingDataAsset` → `USoundDefInstanceLimitingCondition` | 直接调用 | 管理器遍历 `Conditions` 数组并调用各条件的 `ShouldActivate` / `ShouldDeactivate` / `ShouldKill`（`SoundDefInstanceLimiting.as` 第 17-60、141-165 行）。 |
| 子→静态库 | `USoundDefInstanceLimiting_ListenersInRange` → `Audio::` | 直接调用 | 通过 `Audio::GetListeners(EvaluationData.Outer, Listeners)` 获取监听器列表（`ListenersInRange.as` 第 20 行）。 |
| 配置 | 条件资产 / 数据资产 | 可组合设置 | `Conditions`、`MaxDistance`、`MaxInstanceCount`、`Dot` 均为 `UPROPERTY(EditAnywhere)`，由设计师在资产中组合配置，无需改代码。 |
| 读取 | `UHazeSoundDefBase` | 直接调用 | 条件读取实例状态：`GetActivationState()`、`ListenersInRange()`、`AudioComponents`、`TimeActive`、`HazeOwner`（多文件）。 |

## 关键文件（逐文件一句话）

- `SoundDefInstanceLimiting.as`：默认数据资产实现，聚合条件并按 Behaviour/LimitingType 重写引擎的激活/停用/销毁评估入口。
- `SoundDefInstanceLimitCondition.as`：抽象条件基类，定义 `ShouldActivate`/`ShouldDeactivate`/`ShouldKill` 三个可扩展虚方法（默认返回 false）。
- `Conditions/SoundDefInstanceLimiting_ListenersInRange.as`：按 `MaxDistance` 判定是否有监听器在范围内的条件。
- `Conditions/SoundDefInstanceLimiting_MaxInstanceCount.as`：按 `MaxInstanceCount` 判定并发实例是否超额的条件。
- `Conditions/SoundDefInstanceLimiting_WithInDot.as`：按 `Dot` 阈值判定玩家是否处于组件朝向夹角内的条件（不支持 Kill）。

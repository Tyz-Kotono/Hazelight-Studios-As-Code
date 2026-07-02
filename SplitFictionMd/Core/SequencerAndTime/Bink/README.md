# Core / SequencerAndTime / Bink

> 职责：Bink 视频播放——对引擎 `UBinkMediaPlayer` 的薄封装 Actor，提供播放/暂停/停止与时长查询。共 1 文件。

## 内部架构

- **`ABinkMediaActor`**（`AHazeActor`，Abstract）：
  - 持有一个 `UBinkMediaPlayer BinkMediaPlayer`（EditAnywhere，由蓝图/关卡指定）。
  - 全部方法为对 `BinkMediaPlayer` 的空安全转发：
    - `ForceInitialize()` → `InitializePlayer()`
    - `Play()` / `Stop()` / `Pause()`
    - `GetDuration()` / `GetPlayPosition()`：以秒返回总时长 / 当前播放位置。
  - 每个方法均先判 `BinkMediaPlayer == nullptr` 再调用，避免崩溃。

### 数据流
关卡/蓝图触发 `Play()` 等 → 转发到引擎 `UBinkMediaPlayer`。属于纯封装，无内部状态机。

## 协同关系（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
|------|------|------|------|
| 调用 | 引擎 `UBinkMediaPlayer` | 直接调用 | 所有方法转发到该播放器实例 |
| 被调用 | 关卡蓝图 / 序列 | 直接调用（UFUNCTION） | `Play`/`Stop`/`Pause` 等暴露给蓝图与关卡逻辑触发 |

> 注：grep 未在 Core 内发现其它脚本直接调用 `ABinkMediaActor`，主要由关卡资产 / 蓝图驱动，故协同面较窄。

## 关键文件

| 文件 | 一句话 |
|------|--------|
| `BinkMediaActor.as` | 封装 `UBinkMediaPlayer` 的视频播放 Actor（播放/暂停/停止/查询，全部空安全） |

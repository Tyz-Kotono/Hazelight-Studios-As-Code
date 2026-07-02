# Core / SequencerAndTime

> 职责：在 Haze 引擎的过场序列（Sequencer）之上叠加 AngelScript 能力层，并提供慢动作（TimeDilation）、开发期调试（Time）与 Bink 视频播放的胶水代码。共 4 子模块、20 个文件。

## 模块定位

本功能域是“游戏时间与过场表现”的脚本侧总成。引擎已提供 `AHazeLevelSequenceActor`、世界时间缩放等底层能力，本域只负责把这些能力接入 Split Fiction 的双人（Mio / Zoe）玩法规则与渲染需求：

- **Sequencer（15 文件）**：当玩家有活动序列时驱动一组带 `n"Sequencer"` 标签的 `UHazePlayerCapability`（描边抑制、移动阻塞、跳过投票），以及一组编辑器/运行期相机和后处理胶水 Actor（镜头门户、眼神光、Glitch 融合、全局后处理、分屏→全屏过渡）。还含纯脚本的 `DevCutscene` 文本占位过场。
- **TimeDilation（2 文件）**：`UTimeDilationEffectSingleton` 聚合所有慢动作请求，取最低倍率施加到世界/Actor，支持 blend in/out 与自动到期。`TimeDilation::` 命名空间为唯一入口。
- **Time（2 文件）**：仅开发期。键盘/手柄按键与 Dev Toggle 切换调试用世界时间倍率。
- **Bink（1 文件）**：`ABinkMediaActor` 对 `UBinkMediaPlayer` 的薄封装，播放/暂停/查询视频。

## 子模块导航

| 子模块 | 文件数 | 一句话职责 |
|--------|-------|-----------|
| [Sequencer](Sequencer/README.md) | 15 | 过场能力层 + 跳过投票 + 相机/渲染胶水 + 开发占位过场 |
| [TimeDilation](TimeDilation/README.md) | 2 | 聚合式慢动作单例，取最低倍率施加到世界/Actor |
| [Time](Time/README.md) | 2 | 开发期世界时间倍率调试输入与 Dev Toggle |
| [Bink](Bink/README.md) | 1 | Bink 视频播放 Actor 封装 |

## 跨域协同（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
|------|------|------|------|
| 被调用 | `Audio/Listeners/PlayerCutsceneListenerCapability.as` | 标签阻塞 + 直接调用 | 音频监听能力通过 `Player.GetActiveLevelSequenceActor()` 判断过场类型并阻塞能力 |
| 调用 | 引擎 `AHazeLevelSequenceActor` | 直接调用 | Sequencer 能力查询 `GetActiveLevelSequenceActor` / `SkippableSetting` / `SetPlayerWantsToSkipSequence` |
| 被调用 | TimeDilation 命名空间 | 直接调用 `Get` | 任意系统经 `TimeDilation::StartWorldTimeDilationEffect` / `mixin StartActorTimeDilationEffect` 申请慢动作 |
| 关联 | `ForceFeedback`（同 Core） | 参数解耦 | FF 提供 `bIgnoreTimeDilation` 让震动不受世界时间倍率影响 |

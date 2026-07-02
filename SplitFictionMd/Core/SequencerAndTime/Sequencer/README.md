# Core / SequencerAndTime / Sequencer

> 职责：在引擎 `AHazeLevelSequenceActor` 之上叠加 AngelScript 能力层（描边抑制 / 移动阻塞 / 协作跳过投票），并提供过场相机与渲染胶水、纯脚本占位过场。共 15 文件。

## 内部架构

围绕“玩家是否有活动序列”这一引擎查询 `Player.GetActiveLevelSequenceActor()` 展开，分三组：

### 1. 序列能力层（打 `n"Sequencer"` 标签的 `UHazePlayerCapability`）
有活动序列时激活，离开时反激活。基类均为 `UHazePlayerCapability`。

- **`UPlayerSequenceCapability`**：序列激活时，若 `SequenceActor.ShouldHideOutline(Player)` 则 `BlockCapabilities(CapabilityTags::Outline)` 抑制角色描边，反激活时解除。
- **`UPlayerSequenceBlockMovementCapability`**：当序列存在且 `Movement` 标签已被阻塞时激活，清空子动画实例并在结束时 `ResetMovement()`。
- **`USkipSequenceCapability`**（NetworkMode = Crumb）：玩家按住 `SkipCutscene` 且序列 `SkippableSetting != None` 时，调 `SetPlayerWantsToSkipSequence(Player, true)`，向序列 Actor 报告本玩家想跳过。
- **`USkipSequenceShowUICapability`**（NetworkMode = Crumb）：与上类似，但调 `SetPlayerWantsToShowSkipUI`，并带 0.5s 输入余量逻辑控制跳过提示 UI 显隐。

> 跳过是“双人感知”：能力只上报单人意愿（crumb 同步），最终是否触发跳过由序列 Actor 在双方都按下时裁决。

### 2. 开发期占位过场（DevCutscene 一族，纯脚本）
- **`ADevCutscene`**：核心。播放一串 `FDevCutsceneTextAndDuration` 文本，进度方式 `Duration`（按时长）或 `Input`（双人都按 `SkipCutscene` 才推进）。通过 `UHazeRequestCapabilityOnPlayerComponent` 启停玩家能力表，淡入淡出，支持 `Fullscreen`/`Mio`/`Zoe` 目标。`NetProgressDevCutscene`（NetFunction）保证双人推进同步。`CVar_SkipDevCutscenes` 可整体跳过。
- **`UDevCutsceneInputCapability`**（NetworkMode = Crumb，标签 `n"SkipCutscene"`）：读取玩家组件上的活动 DevCutscene，按键时 `SetPlayerWantsToProgress`。
- **`UDevCutscenePlayerComponent`**：单字段组件，持有 `ActiveDevCutscene`，作为能力与 Actor 的桥。
- **`UDevCutsceneWidget`**：仅一个 `DisplayText` 字段的占位 Widget。

### 3. 相机与渲染胶水（编辑器 + 运行期）
- **`ASequencerCameraPortal`** + 内嵌 `USequencerCameraPortalComponent`：`SceneCapture2D` 渲染到 RenderTarget，把“白空间相机”视角投到门户网格材质。
- **`ASequencerEyeLight`** + 组件：每帧把假眼神光参数（颜色/亮度/粗糙度/位置）写入目标角色 `Char_Eye` 材质。
- **`ASequencerGlitchBlend`**（333 行，最复杂）：双人角色“故障溶解”效果，把材质换成双面版、驱动 `Glitch_*` 材质参数与缝合 Niagara VFX，按 `BlendMio`/`BlendZoe` 控制。
- **`ASequencerGlobalPostProcess`**：全局后处理体，运行期接管 Mio/Zoe 的 `UPostProcessingComponent`（开始禁用、结束恢复），驱动白空间/加载屏材质参数。
- **`UHazeSequenceRenderSingleton`**（继承 `UHazeSequenceRenderBaseSingleton`）：缓存 Mio/Zoe 的少年龙/成年龙弱引用，供序列查询。
- **`AFullscreenTransitionInner`**：用 `SceneView::SetManualViews` 把垂直分屏按 Alpha 渐变扩张为单屏（第三屏相机），过渡结束恢复 `Vertical`。
- **`ACutsceneViewSizeOverrideControlActor`**：当任一玩家 `bIsControlledByCutscene` 时 `SceneView::SetViewSizeOverride`。

## 协同关系（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
|------|------|------|------|
| 调用 | 引擎 `AHazeLevelSequenceActor` | 直接调用 | 全部序列能力查询 `GetActiveLevelSequenceActor` 决定激活；`SetPlayerWantsToSkipSequence` / `SetPlayerWantsToShowSkipUI` 上报意愿 |
| 标签阻塞 | `CapabilityTags::Outline` / `Movement` | 标签阻塞 | `UPlayerSequenceCapability` 抑制描边；`UPlayerSequenceBlockMovementCapability` 配合 Movement 阻塞 |
| Crumb | Skip / DevCutscene 输入能力 | Crumb 网络模式 | `USkipSequenceCapability`、`UDevCutsceneInputCapability` 等 `NetworkMode = Crumb` 同步单人意愿 |
| 被调用 | `Audio/Listeners/PlayerCutsceneListenerCapability.as` | 直接调用 | 经 `Player.GetActiveLevelSequenceActor()` 读取 `bIsCutscene` 决定阻塞哪些能力（grep: `GetActiveLevelSequenceActor`） |
| 委托 | `ADevCutscene.FinishedDelegate` | 委托 | 占位过场结束回调外部逻辑 |
| 直接调用 | `SceneView::` 命名空间 | 直接调用 | `FullscreenTransitionInner` / `CutsceneViewSizeOverride` 设置手动视口与视口尺寸 |

## 关键文件

| 文件 | 一句话 |
|------|--------|
| `PlayerSequenceCapability.as` | 序列激活时按需抑制角色描边 |
| `PlayerSequenceBlockMovementCapability.as` | 序列+Movement阻塞时清子动画并冻结移动 |
| `SkipCutsceneCapability.as` | 上报“本玩家想跳过序列”（Crumb） |
| `SkipCutsceneShowUICapability.as` | 上报“本玩家想显示跳过提示UI”（Crumb，带输入余量） |
| `DevCutscene.as` | 纯脚本文本占位过场，双人输入推进 + 淡入淡出 + 能力表启停 |
| `DevCutsceneInputCapability.as` | 读取活动DevCutscene并上报推进意愿（Crumb） |
| `DevCutscenePlayerComponent.as` | 单字段桥组件，持有 ActiveDevCutscene |
| `DevCutsceneWidget.as` | 占位文本 Widget |
| `SequencerCameraPortal.as` | SceneCapture 渲染白空间相机到门户材质 |
| `SequencerEyeLight.as` | 把假眼神光参数写入角色眼睛材质 |
| `SequencerGlitchBlend.as` | 双人角色故障溶解材质+Niagara 效果驱动 |
| `SequencerGlobalPostProcess.as` | 全局后处理体，接管玩家后处理与白空间/加载屏参数 |
| `SequencerRenderSingleton.as` | 缓存 Mio/Zoe 少年/成年龙弱引用供序列查询 |
| `FullscreenTransitionInner.as` | 用手动视口把分屏渐变扩张为单屏 |
| `CutsceneViewSizeOverrideControlActor.as` | 过场控制时覆盖视口尺寸百分比 |

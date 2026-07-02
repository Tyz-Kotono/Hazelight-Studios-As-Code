# Core / Audio / AnimNotifies

> 职责：把动画序列上的通知关键帧（AnimNotify）转换为一次性（fire-and-forget）音频事件投放，让脚步、动作、技能等音效与动画帧精确对齐。共 2 个文件。

## 内部架构（关键类、基类、数据流）

两个类均继承自引擎基类 `UAnimNotify`，通过 `Notify()`（`BlueprintOverride`）回调被动画系统在到达通知帧时触发。回调参数携带触发动画的 `USkeletalMeshComponent`，由此反查 Owner 与挂点。

- **`UAnimNotify_PostAudioEvent`**：通用单事件通知。持有一个 `UHazeAudioEvent` 与一份 `FHazeAudioFireForgetEventParams`。触发时按以下优先级解析挂点（`EventParams.AttachComponent`）：`bUseSkelMeshAsAttach` 为真则用触发的骨骼网格 → 否则按 `AttachComponentName` 用 `USceneComponent::Get` 取指定组件 → 都没有则退回 `Actor.RootComponent`。最终调用 `AudioComponent::PostFireForget(Event, EventParams)`。

- **`UAnimNotify_PostPlayerAudioEvent`**：玩家专用双事件通知。持有 `MioEvent` 与 `ZoeEvent` 两个事件。触发时把 Owner `Cast` 成 `AHazePlayerCharacter`，再用 `Player.IsMio()` 选出对应角色的事件，挂点按 `AttachComponentName`（取自 Player）或 `Player.RootComponent` 解析，再 `PostFireForget`。Cast 失败或事件为空则直接返回 false。

- **数据流**：动画播放 → 命中 Notify 帧 → `Notify()` 解析事件 + 挂点参数 → `AudioComponent::PostFireForget`（底层桥接 Wwise，在挂点位置生成一次性发射器播放声音，分屏声像由底层按 Mio/Zoe 屏幕位置处理）。两类都有 `#if EDITOR` 分支：非运行态（`!Editor::IsPlaying()`）时改走 `AudioComponent::PostGlobalEvent` 并包裹 `FScopeDebugEditorWorld`，用于编辑器内预览，不依赖运行时挂点。

与三角色的关系：这两个通知不直接持有/归还池化发射器，而是把发射器的获取与生命周期托管给 `AudioComponent::PostFireForget`（fire-and-forget 语义，播完自动回收），因此它们是发射器的"投放者"而非"管理者"。

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 调用→ | `AudioComponent::PostFireForget` | 直接调用 Type::Get | 两类的运行时主路径，提交一次性音频事件并由底层托管发射器生命周期 |
| 调用→ | `AudioComponent::PostGlobalEvent` | 直接调用 Type::Get | 编辑器预览分支（非运行态）调用，播放非空间化全局事件 |
| 调用→ | `USceneComponent::Get` | 直接调用 Type::Get | 按 `AttachComponentName` 在 Owner/Player 上解析音频挂点组件 |
| 调用→ | `AHazePlayerCharacter.IsMio()` | 直接调用 Type::Get | PlayerAudioEvent 据此在 MioEvent / ZoeEvent 间选择角色专属事件 |
| ←被调用 | 动画系统（AnimSequence Notify 轨道） | 委托 event/Broadcast | `Notify()` 由动画通知轨道在关键帧回调触发，是本目录唯一入口 |
| 配合 | `FHazeAudioFireForgetEventParams` | 可组合设置 ApplySettings | 以 `EditInstanceOnly` 暴露的参数结构，复制后写入 `AttachComponent` 再随事件提交 |

> 注：`AudioComponent::` 命名空间与 `FHazeAudioFireForgetEventParams` 在 AngelScript 层无 `.as` 定义，为 C++ 绑定（桥接 Wwise/Ak）。同类动画通知模式在多个关卡可见（如 `LevelSpecific/Summit/Giants/Audio/AnimNotify_GiantsAudio.as`），本目录提供的是通用与玩家两个基础变体。

## 关键文件（逐文件一句话）

- `AnimNotify_PostAudioEvent.as`：通用动画通知，在动画帧上向指定挂点（骨骼网格 / 命名组件 / 根组件）投放单个一次性音频事件。
- `AnimNotify_PostPlayerAudioEvent.as`：玩家专用动画通知，按 `IsMio()` 在 Mio/Zoe 两个事件中择一，在玩家挂点上投放一次性音频事件。

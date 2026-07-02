# Core / SplineAndSpace / Attachment

> 职责：两个编辑器友好的"把组件/Actor 动态附着到目标组件"的辅助组件。共 2 个文件。

## 内部架构（关键类/基类/数据结构/数据流）

两个组件都在 `BeginPlay` 时执行一次 `AttachToComponent`，并都带 `#if EDITOR` 的细节面板（`UHazeScriptDetailCustomization`）用下拉框选择目标组件名、对缺失组件给出告警。

- **`AttachOwnerToParentComponent.as`**（`UAttachOwnerToParentComponent : UActorComponent`）：把**本 Actor 的 RootComponent**附着到其"附着父 Actor"上的某个具名组件（`ComponentNameOnParent`），而非默认的父 Root。`BeginPlay` 中取 `Owner.RootComponent.GetAttachParent()` 所属 Actor 上的 `USceneComponent::Get(..., ComponentNameOnParent)` 并 `AttachToComponent`（`AttachmentRule` 默认 `KeepWorld`）。
- **`DynamicAttachmentComponent.as`**（`UDynamicAttachmentComponent : UActorComponent`）：更通用——把**本 Actor 上的某个具名组件**（`SubjectComponentToAttach`）附着到**任意目标 Actor**（`ActorToAttachTo`，留空则用附着父 Actor）上的某具名组件（`TargetComponentToAttachTo`）的某 socket（`TargetAttachSocket`）。
- 数据流：编辑器细节面板写入组件名/目标 → `BeginPlay` 解析名字到实际 `USceneComponent` → 执行附着。纯附着工具，无 tick 运行时逻辑（编辑器面板的 Tick 仅用于绘制 UI）。

## 协同关系（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
| --- | --- | --- | --- |
| 调用 | 引擎 `USceneComponent::Get` / `AttachToComponent` | 直接调用 `Type::Get` | 按名字解析组件并执行附着 |
| 调用 | `Owner.RootComponent.GetAttachParent` / `GetAttachParentActor` | 直接调用 | 默认目标取自附着父 Actor |
| 提供 | 关卡美术/搭建（编辑器细节面板） | 编辑器 UI（机制 5 类设置） | 下拉框选组件名、缺失时高亮告警 |

## 关键文件（逐文件一句话）

- `AttachOwnerToParentComponent.as`：把本 Actor 的 Root 附着到父 Actor 上的某具名组件（而非父 Root），带编辑器选择面板。
- `DynamicAttachmentComponent.as`：把本 Actor 的某组件附着到任意目标 Actor 的某组件/socket，带编辑器选择面板。

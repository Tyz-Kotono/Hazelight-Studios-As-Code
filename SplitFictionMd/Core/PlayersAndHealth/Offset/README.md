# Core / PlayersAndHealth / Offset

> 职责：以 mixin 形式暴露玩家内置「相机 / 根 / 网格」三个偏移组件（`UHazeOffsetComponent`）的便捷访问器，共 1 文件。

## 内部架构

`PlayerOffsets.as` 仅定义 3 个 mixin 函数，挂在 `AHazePlayerCharacter` 上，把引擎内置的偏移组件以语义化名字转发出来，无自有数据结构与逻辑：

- `GetCameraOffsetComponent(Player)` → `Player.CameraOffsetComponent`
- `GetRootOffsetComponent(Player)` → `Player.RootOffsetComponent`
- `GetMeshOffsetComponent(Player)` → `Player.MeshOffsetComponent`

`UHazeOffsetComponent` 与底层 `CameraOffsetComponent`/`RootOffsetComponent`/`MeshOffsetComponent` 均由 Haze 引擎（C++/父类 `AHazePlayerCharacter`）提供，本模块只是 AngelScript 侧的访问糖。

## 协同关系

| 方向 | 对象 | 机制 | 说明 |
|------|------|------|------|
| 访问 | `AHazePlayerCharacter` 内置偏移组件 | mixin 直接访问 | 转发引擎内置成员，供其它系统（相机、动画、移动）取偏移组件 |
| 返回 | `UHazeOffsetComponent`（引擎类型） | 直接返回 | 调用方可在其上做 Apply/Clear 等偏移操作 |

## 关键文件

- `PlayerOffsets.as`：相机/根/网格偏移组件的 3 个访问 mixin。

# Editor / 独立工具窗口 — Utility Widgets

> 职责：从编辑器菜单手动打开的独立功能性工具窗口，各自解决一个专项制作 / 排查需求。
>
> 覆盖子模块：**LevelFlow**（1）、**Vox**（1）、**BakeLightingStarter**（1）、**TraceTestUtilty**（1）。
>
> 这些都是 `UEditorUtilityWidget`（或编辑器 actor），不挂在任何运行时对象上，属于"打开即用"的独立面板。

---

## 1. LevelFlow/ — 关卡流程工具（1 文件）

#### `LevelFlowUtilityWidget`
`UEditorUtilityWidget`，用于检查游戏的**关卡流程（Level Flow）**——关卡之间的加载/跳转关系。可输出 **cook 清单**（哪些关卡/资产需要被打包），辅助构建流程。

---

## 2. Vox/ — 孤立语音事件查找（1 文件）

#### `VoxOrphanedEventsFinderUtilityWidget`
`UEditorUtilityWidget`，查找**孤立的语音(VOX)事件**——即在语音表(sheet)里存在、但没有被任何地方引用，或缺少音频资产的事件。

- 异步从语音系统拉取记录（`bWaitingForRecords` / `PendingSheetsToGet`）。
- 丰富的过滤开关：按引用状态（已引用/未引用/有 VOX 资产）、按类型（TTS 文字转语音 / Rendered 已录制 / None）、仅显示无 ID 的。
- 用颜色编码区分类型（TTS 绿 / Temp 蓝 / Rendered 蓝 / 已引用橙 / Wav 红）。

> 配合 `Core/Vox` 语音系统，帮音频团队清理无用/未挂接的语音事件。

---

## 3. BakeLightingStarter/ — 光照烘焙启动器（1 文件）

#### `BakeLightingStarterUtilityWidget`
`UEditorUtilityWidget`，提供一个面板：选择若干关卡，一键启动**光照烘焙（Bake Lighting）**。把繁琐的逐关卡烘焙操作集中成一个工具。

---

## 4. TraceTestUtilty/ — 射线检测测试（1 文件）

#### `TraceTestUtilty`
编辑器内的 actor / 组件，用于**可视化并测试物理射线检测（trace）**。在场景里摆放后可实时看到射线起讫、命中点、命中物，帮程序/设计验证碰撞查询行为（配合 `Examples/Physics` 的 `Trace::` 库）。

---

## 小结

| 工具 | 一句话 | 服务对象 |
|---|---|---|
| `LevelFlowUtilityWidget` | 关卡流程检查 + cook 清单输出 | 关卡/构建 |
| `VoxOrphanedEventsFinderUtilityWidget` | 找孤立/缺资产的语音事件 | 音频 |
| `BakeLightingStarterUtilityWidget` | 批量启动光照烘焙 | 美术/灯光 |
| `TraceTestUtilty` | 可视化测试物理射线 | 程序/设计 |

这些工具彼此独立、无共享框架，唯一共性是"从菜单打开的一次性辅助面板"。

# Core / VoxAndSubtitles / Accessibility

> 职责：无障碍（屏幕朗读 / TTS）测试入口——通过控制台命令把文本交给引擎文字转语音朗读。共 1 个文件（`TestNarration.As`）。

## 内部架构（关键类/基类/数据结构/数据流）

仅一个开发期（`#if !RELEASE`）控制台命令，无类、无状态：

```
开发者输入控制台命令
   Haze.TestNarration <文本…>
   ▼
Narration_TestNarration(Arguments)
   │ 检查 Game::IsNarrationEnabled()
   │ 拼接参数为整句
   ▼
Game::NarrateString(NarrationText)   →  引擎 TTS 朗读
```

- `FConsoleCommand Command_TestNarration("Haze.TestNarration", n"Narration_TestNarration")` 注册命令。
- `local void Narration_TestNarration(TArray<FString> Arguments)`：
  - 若朗读未启用（`Game::IsNarrationEnabled()` 为假），`devErrorAlways` 报错返回。
  - 拼接所有参数为整句并 `TrimStartAndEnd`，非空则 `Game::NarrateString` 朗读；否则朗读默认占位句 "Testing narration text to speech"。
- 整体被 `#if !RELEASE` 包裹，发行版不编入。

## 协同关系（grep 实证）

| 方向 | 对象 | 机制 | 说明 |
|---|---|---|---|
| 调出 | `Game::NarrateString` | 1 直接调用 | 把文本交给引擎 TTS 朗读 |
| 调出 | `Game::IsNarrationEnabled` | 1 直接调用 | 朗读前确认无障碍朗读已启用 |
| 调入 | 控制台命令 `Haze.TestNarration` | 5 可组合设置（CVar/命令） | 开发者手动触发，不与 Vox/Subtitles 直接耦合 |

> 说明：本模块仅是独立的 TTS 测试工具，与 Vox / Subtitles 无代码级调用关系；三者同属「语音与文字可达性」功能域而并列归档。

## 关键文件（逐文件一句话）

| 文件 | 说明 |
|---|---|
| `TestNarration.As` | 开发期控制台命令 `Haze.TestNarration`，将文本经 `Game::NarrateString` 做 TTS 朗读测试。 |

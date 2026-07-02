# MessageDialog — 模态弹窗四件套

> 一句话职责：提供全局最高优先级的模态确认/选择弹窗，以「data / singleton / statics / widget」四件套解耦——调用方只需构造一个 `FMessageDialog` 数据并调 `ShowPopupMessage()`，队列管理、控件生命周期与呈现全部由单例与控件自动完成。

---

## 内部架构

四层，各司其职：

1. **data（`MessageDialogData.as`）**：纯数据。`FMessageDialog`（消息文本 + 选项数组 + `bInstantCloseOnCancel`）与 `FMessageDialogOption`（标签/描述/回调 `FOnMessageDialogOptionChosen`/类型）。提供便捷构造方法 `AddOKOption`/`AddConfirmCancelOptions`/`AddOption`/`AddCancelOption`。
2. **statics（`MessageDialogStatics.as`）**：全局函数入口。`ShowPopupMessage(Message, Instigator)`、`ClosePopupMessageByInstigator(Instigator)`、`IsMessageDialogShown()`——上层代码只跟这三个函数打交道。
3. **singleton（`MessageDialogSingleton.as`）**：`UMessageDialogSingleton : UHazeSingleton`。持有消息队列 `Messages[]`（`FActiveMessageDialog`：Id+Dialog+Instigator）与当前控件。`AddMessage` 入队（空选项自动补 OK）→ `Update()` 保证队首消息对应的控件存在、设焦点/鼠标、发音效事件、旁白；`CloseMessage`/`CloseMessageWithInstigator` 出队。**还兼职轮询在线子系统**：Tick 里 `Online::ConsumeLastError` → 弹错误框、`Online::ConsumeQuestion` → 弹是/否问询框；控件在时**永远强制持有焦点**。
4. **widget（`MessageDialogWidget.as`）**：`UMessageDialogWidget : UHazeUserWidget`（Abstract）+ 按钮 `UMessageDialogButton : UMenuButtonWidget`。呈现层：`UpdateMessage()` 依选项数组生成按钮列表，Show/Hide 动画，描述提示浮层定位，`OnKeyDown/Up` 处理 Back/Escape → 取消选项，按钮点击 → 执行对应回调 + 出队。

### 关键调用流

```
调用方: FMessageDialog Dialog; Dialog.AddOption(...); ShowPopupMessage(Dialog, this);
        │
        ▼
statics.ShowPopupMessage → singleton.AddMessage(入队) → Update()
        │
        ▼
若队首无对应控件: AddFullscreenWidget(DialogWidgetClass, Menu 层, ZOrder=1000)
                 → widget.UpdateMessage() 建按钮 → Show() 动画 → 设焦点/鼠标 → 音效/旁白
        │
        ▼
用户按下按钮/Back → widget.OnButtonClicked → singleton.CloseMessage(出队)
                    → 执行 Option.OnChosen 回调 → Update() 显示下一条或移除控件
```

- **Instigator 机制**：每条消息记录发起者（`FInstigator`）。`ClosePopupMessageByInstigator` 让某个界面（如主菜单退出时）批量关闭自己弹出的对话框，避免残留。
- **优先级**：控件挂在 `EHazeWidgetLayer::Menu`、ZOrder=1000、`SetWidgetPersistent(true)`，Tick 里强制 `SetAllPlayerUIFocusBeneathParent`——凌驾于主菜单/暂停菜单之上。

---

## 关键文件 / 控件

| 文件 | 类型 | 职责与要点 |
| --- | --- | --- |
| `MessageDialogData.as` | `FMessageDialog` / `FMessageDialogOption` / `EMessageDialogOptionType` / delegate `FOnMessageDialogOptionChosen` | 弹窗数据模型 + 便捷构造（OK/确认取消/取消项）。选项类型 `Option`/`Cancel`（Cancel 项可由 B/Esc 触发） |
| `MessageDialogStatics.as` | 全局 `UFUNCTION` | `ShowPopupMessage` / `ClosePopupMessageByInstigator` / `IsMessageDialogShown` 三个入口 |
| `MessageDialogSingleton.as` | `UMessageDialogSingleton : UHazeSingleton` + `FActiveMessageDialog` | 队列与生命周期核心；轮询在线错误/问询；控件强制持焦；含 `Haze.TestMessageDialog` 控制台命令 |
| `MessageDialogWidget.as` | `UMessageDialogWidget : UHazeUserWidget` (Abstract) + `UMessageDialogButton : UMenuButtonWidget` | 呈现层：动态生成按钮、Show/Hide 动画、描述浮层、Back/Escape→取消、点击→回调+出队。按钮含线段/圆点装饰、Mio/Zoe/Neutral 三态、激活动画、聚焦旁白 |

---

## 复用关系

- 被主菜单（主机选择 Steam/EA 服务器、Switch 本地无线、账号切换）、选项菜单（重置确认、退出错误确认）、大厅（跨平台断连确认）、`GlobalMenuOverlay`（游戏邀请接受/拒绝）、在线子系统（错误/问询）广泛调用。
- 按钮继承 `Menus/` 的 `UMenuButtonWidget`，装饰复用 `UMenuSelectionHighlight`。

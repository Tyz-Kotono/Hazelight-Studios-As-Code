# Gameplay / Movement / Tutorials —— 单招式教程

> 职责：为**单个运动招式**弹出操作提示的教学能力。共 **7 文件**，每个对应一招。玩家进入合适情境时显示"按哪个键做什么"的提示链，做出动作后自动收起。

---

## 机制

每个文件是一个 `UTutorialCapability` 子类（Core 教程能力基类），套路一致：

1. **`ShouldActivate`**：判断玩家是否处在该招式的教学时机（如空冲教学要求先在地面）。
2. **`OnActivated`**：构造 `FTutorialPromptChain`——一串 `FTutorialPrompt`（按键 `Action` + 本地化文本），调 `Player.ShowTutorialPromptChain`。
3. **`TickActive`**：根据玩家进度切换提示链当前项（`SetTutorialPromptChainPosition`）——例如空冲教学：地面显示"跳"→ 空中显示"二段跳"→ 二段跳后显示"空冲"。
4. **`OnDeactivated`**：`RemoveTutorialPromptByInstigator` 收起提示。

提示文本走 `NSLOCTEXT` 本地化；按键走 `ActionNames::*`。教程能力只读运动状态（`UPlayerMovementComponent`、`IsAnyCapabilityActive`），不改变运动本身。

---

## 七个教程

| 文件 | 教的招式 |
| --- | --- |
| `PlayerDashTutorialCapability.as` | 地面冲刺 |
| `PlayerAirDashTutorialCapability.as` | 空中冲刺（提示链：跳 → 二段跳 → 空冲） |
| `PlayerAirJumpTutorialCapability.as` | 二段跳 |
| `PlayerSwingTutorialCapability.as` | 荡绳 |
| `PlayerSwimmingUpDownTutorialCapability.as` | 游泳上浮/下潜 |
| `PlayerPoleDashTutorialCapability.as` | 爬杆冲刺 |
| `PlayerFastLadderClimbTutorialCapability.as` | 快速爬梯 |

---

## 范例：空中冲刺教程

`Tutorials/PlayerAirDashTutorialCapability.as`

```cpp
void OnActivated()  // 构造三步提示链
{
    FTutorialPromptChain TutorialChain;
    // 步 0：跳    步 1：二段跳    步 2：空冲
    TutorialChain.Prompts.Add(JumpPrompt);      // Action=MovementJump
    TutorialChain.Prompts.Add(AirJumpPrompt);   // Action=MovementJump "Double Jump"
    TutorialChain.Prompts.Add(AirDashPrompt);   // Action=MovementDash "Air Dash"
    Player.ShowTutorialPromptChain(TutorialChain, this, 0);
}

void TickActive(float DeltaTime)  // 按玩家状态推进提示链
{
    if (MoveComp.IsOnAnyGround())              SetTutorialPromptChainPosition(this, 0);  // 地面：提示跳
    else if (!bShowAirDashPrompt)              SetTutorialPromptChainPosition(this, 1);  // 空中：提示二段跳
    else                                       SetTutorialPromptChainPosition(this, 2);  // 二段跳后：提示空冲
    if (Player.IsAnyCapabilityActive(n"AirJump"))  bShowAirDashPrompt = true;
}
```

---

## 关键文件

| 文件 | 作用 |
| --- | --- |
| `PlayerAirDashTutorialCapability.as` | 提示链推进范例（跳→二段跳→空冲） |
| 其余 6 个 `Player*TutorialCapability.as` | 各对应一招，结构同上 |
</content>

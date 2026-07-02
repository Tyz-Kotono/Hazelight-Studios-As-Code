# Audio — 音频定义体系

> 路径：`SplitFiction/Audio`（1191 文件）
>
> 全部位于 `Audio/SoundDefinitions`，围绕 `USoundDefBase` 三层组织。采用 Wwise 风格的事件/RTPC 音频触发。

---

## 三层架构

### SoundDefinitions/（根）
主体文件。具体的 `USoundDefBase` 子类（如 `Character_AI_Shared_Basic_Movement_SoundDef`、各 Boss 专属定义），它们：
- 缓存游戏组件与发射器的引用
- 包含自动生成的事件钩子段落
- 调用 `Trigger_*` 函数 / 设置 RTPC，响应游戏状态播放音频

这一层拥有实际的音频行为。

### BaseImplementations/
共享的抽象基类，供其它 SoundDef 继承以复用通用装配，将可复用的音频管线抽离出来（而非关卡专属）。例如：
- `SoftSplit_SoundDef_Base` —— 把 spot-sound 组件接到音频组件
- `UI_SoundDef`
- `Foliage_SoundDef`
- `Spot_Tracking_SoundDef`

### Adapters/
桥接类（~85 文件）。**不**继承 `USoundDefBase`，而是各自继承一个玩法事件处理器（如 `UScifiCopsGunEventHandler`, `UMagneticFieldEventHandler`）。它们：
- 实现玩法事件回调（`OnBulletImpact`, `OnShoot` 等）
- 把玩法事件参数转换成音频参数
- 转发给所属的 SoundDef（通过 `Outer` 与 `GetSoundDef` 属性访问）

多数回调体是自动生成的桩，只填充相关事件。

---

## 工作模式

```
玩法系统            发出类型化事件
    ↓
Adapter            监听并转换事件参数 → 音频参数
    ↓
SoundDef           执行实际音频触发（Trigger_* / RTPC）
（常基于 BaseImplementation）
```

1. 一个玩法系统发出**类型化事件**。
2. 一个 **Adapter** 监听并把事件转换成对其关联 SoundDef 的调用。
3. 该 **SoundDef**（常基于某个 BaseImplementation）执行实际的音频触发。

> 该设计彻底**解耦**了玩法事件源与音频播放逻辑：玩法代码无需知道音频细节，音频代码无需嵌入玩法代码。

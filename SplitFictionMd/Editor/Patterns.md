# Editor / 编写模式 — Patterns

> 视角：前 5 篇讲 Editor 模块**有什么**，本篇讲**怎么写**。
>
> 目标：读完能照着骨架写出一个新的编辑器工具。内容为横切总结——6 大基类的代码模板、即时模式 UI、可撤销事务、`CallInEditor` 暴露操作、编辑器专用标记。

---

## 总览：先选对基类

所有 Editor 工具都从 6 个 Haze 编辑器基类之一派生。先按"我要做什么"选基类：

| 我想…… | 选基类 | 触发方式 |
|---|---|---|
| 在视口里给某组件画辅助图形 | `UHazeScriptComponentVisualizer` | 选中带 `VisualizedClass` 组件的 actor 时自动调用 |
| 改写某类 actor 的 Details 面板 | `UHazeScriptDetailCustomization` | 选中 `DetailClass` 指定的对象时 |
| 做一个可停靠的独立工具窗口 | `UEditorUtilityWidget`（或 `UHazeUserWidget`） | 从菜单手动打开 |
| 给 actor / 资产右键菜单加操作 | `UScriptActorMenuExtension` / `UScriptAssetMenuExtension` | 右键选中对象时 |
| 保存 / 提交时校验资产 | `UEditorValidatorBase` | Data Validation 流程自动运行 |
| 用鼠标点击拖拽做交互式编辑 | `UEditorScriptableClickDragTool` | 从工具栏 / 控制台激活 |

配套还有 `UHazeEditorSubsystem`（常驻编辑器的 `Tick` 子系统，用于帧回收、状态监视）和 `UHazeScriptContentBrowserFilter`（内容浏览器过滤器）。

---

## 一、组件可视化器 `UHazeScriptComponentVisualizer`

**骨架**：声明 `VisualizedClass`，重写 `VisualizeComponent`，用基类自带的 `DrawLine` / `DrawWireSphere` / `DrawWireBox` / `DrawDashedLine` 等画线 API 画图。每帧自动调用，无需管理生命周期。

```cpp
class UMyThingVisualizer : UHazeScriptComponentVisualizer
{
    default VisualizedClass = UMyThingComponent;   // 选中带此组件的 actor 时触发

    UFUNCTION(BlueprintOverride)
    void VisualizeComponent(const UActorComponent Component)
    {
        auto Comp = Cast<UMyThingComponent>(Component);
        if (Comp == nullptr)
            return;

        // 直接用基类 API 画线/框/球（内置管线，不需要 RenderTarget）
        DrawLine(Comp.WorldLocation, Comp.TargetLocation, FLinearColor::Blue, 5.0);
        DrawWireSphere(Comp.WorldLocation, 100.0, FLinearColor::Red, 2.0, 16);
    }
}
```

**要点**
- `VisualizedClass` 指向的是**组件类**（如 `URespawnPointVolumeVisualizerComponent`），不是 actor 类。
- 画线用轻量 API；若要画**实心网格 / 实心盒 / 骨骼动画预览**，走 `EditorVisualizationSubsystem`（见第七节）。
- 一个 `.as` 文件里常把 Visualizer 与它对应的 `UHazeScriptDetailCustomization` 写在一起（如 `RespawnPointVolumeVisualizer.as`）。

---

## 二、细节面板定制 `UHazeScriptDetailCustomization`

**骨架**：声明 `DetailClass`，在 `CustomizeDetails()` 里重排面板（隐藏分类、加属性、加即时 UI 行），在 `Tick()` 里绘制动态 UI 与响应逻辑。

```cpp
class UMyActorDetails : UHazeScriptDetailCustomization
{
    default DetailClass = AMyActor;          // 选中此类对象时定制其 Details 面板
    UHazeImmediateDrawer Drawer;             // 持有一个即时绘制器

    UFUNCTION(BlueprintOverride)
    void CustomizeDetails()
    {
        auto MyActor = Cast<AMyActor>(GetCustomizedObject());
        if (MyActor == nullptr) return;

        HideCategory(n"LOD");                 // 隐藏无关分类
        AddDefaultProperty(n"MyCategory", n"SomeProperty");   // 显式加回某属性
        Drawer = AddImmediateRow(n"MyCategory");              // 插入一行即时 UI
    }

    UFUNCTION(BlueprintOverride)
    void Tick(float DeltaTime)
    {
        auto MyActor = Cast<AMyActor>(GetCustomizedObject());
        if (MyActor == nullptr) return;

        // 状态变化后强制刷新面板布局
        // if (布局依赖的东西变了) { ForceRefresh(); return; }

        if (Drawer != nullptr && Drawer.IsVisible())
        {
            auto Root = Drawer.Begin();       // 每帧重建 UI（即时模式）
            if (Root.Button("Do Something"))
                DoSomething();                // 按钮返回 true 即"本帧被点击"
        }
    }
}
```

**常用面板 API**：`HideCategory(n"...")`、`AddDefaultProperty(分类, 属性)`、`AddImmediateRow(分类[, 名字, 展开])`、`GetCustomizedObject()`、`ForceRefresh()`（重建布局）、`NotifyPropertyModified(Obj, n"Prop")`（改属性后通知编辑器）。

**范本**：`Props/PropLineDetailCustomization.as` 是最完整的示例（隐藏分类 + 即时按钮 + 网格合并 + 错误提示 + 自动重命名 + 事务）。

---

## 三、独立工具窗口 `UEditorUtilityWidget`

**骨架**：`BindWidget` 一个 `UHazeImmediateWidget`，在 `Tick` 里拿 `Drawer` 每帧重绘。数据缓存在成员里，UI 只是数据的即时投影。

```cpp
class UMyUtilityWidget : UEditorUtilityWidget
{
    UPROPERTY(BindWidget)
    UHazeImmediateWidget ImmediateWidget;      // 蓝图里放置的即时控件

    UFUNCTION(BlueprintOverride)
    void Tick(FGeometry MyGeometry, float InDeltaTime)
    {
        auto Drawer = ImmediateWidget.GetDrawer();
        if (!Drawer.IsVisible())
            return;

        auto Root = Drawer.BeginVerticalBox();
        Root.Text("My Tool").Bold().Scale(1.4);
        if (Root.Button("Refresh"))
            RefreshData();
    }
}
```

**打开窗口**：从右键菜单或控制台调用 `Blutility::OpenEditorUtilityWindow("/Game/Editor/.../WBP_MyUtilityWidget.WBP_...")`（路径指向蓝图资产，脚本类是它的父类）。

**跨对象通信**：想让"打开窗口的那个 Action"给"窗口实例"传参，常用 `DefaultObject`（CDO）当共享黑板——Action 写标志位，窗口 `Tick` 里读并清除。见 `ActorReferencesUtilityWidget.as`（`bRequestUpdateSelection`）。

**组合模式**：主控件 `BindWidget` 多个子控件，在主 `Tick` 里统一 `TickWidget(...)` 分发驱动，见 `Audio/Overview/AudioOverviewWidget.as`。

---

## 四、右键菜单扩展 `UScriptActorMenuExtension` / `UScriptAssetMenuExtension`

**骨架**：用 `SupportedClasses.Add(...)` 限定适用类型，把操作写成 `UFUNCTION(CallInEditor)` —— 每个 `CallInEditor` 函数就是右键菜单里的一个可点条目。

```cpp
// Actor 版：作用于视口 / 世界大纲里选中的 actor
class UMyActorAction : UScriptActorMenuExtension
{
    default ExtensionPoint = n"ActorGeneral";                 // 挂在哪个菜单区
    default ExtensionOrder = EScriptEditorMenuExtensionOrder::After;
    default SupportedClasses.Add(AActor);                     // 仅这些类显示

    UFUNCTION(CallInEditor, meta = (EditorIcon = "GenericCommands.Copy"))
    void CopySelectedActorsAsArray()
    {
        TArray<AActor> Selected = Editor::SelectedActors;
        // ……操作……
    }
}

// Asset 版：作用于内容浏览器里选中的资产
class UMyAssetAction : UScriptAssetMenuExtension
{
    default SupportedClasses.Add(UStaticMesh);

    UFUNCTION(CallInEditor, Category = "Breakable")
    void MakeBreakable()
    {
        for (UObject Asset : EditorUtility::GetSelectedAssets())
        {
            auto Mesh = Cast<UStaticMesh>(Asset);
            if (Mesh == nullptr) continue;
            // ……用 GeometryScript 等处理 Mesh……
        }
    }
}
```

**要点**
- 选中项：actor 用 `Editor::SelectedActors`，资产用 `EditorUtility::GetSelectedAssets()`。
- `meta = (EditorIcon = "...")` 给菜单项加图标；`Category = "..."` 分组。
- 范本：`CopyActorsAsArrayActorMenuExtension.as`（actor）、`MakeBreakable.as` / `HazeTextureActionUtilities.as`（asset）。

---

## 五、资产校验器 `UEditorValidatorBase`

**骨架**：`CanValidateAsset` 圈定校验范围（返回 `true` 才校验），`ValidateLoadedAsset` 做检查，用 `AssetPasses` / `AssetFails` 报告结果。保存 / 提交时自动跑。

```cpp
class UMyValidator : UEditorValidatorBase
{
    UFUNCTION(BlueprintOverride)
    bool CanValidateAsset(UObject InAsset) const
    {
        return InAsset.IsA(UStaticMesh) || InAsset.IsA(UWorld);   // 只校验这些
    }

    UFUNCTION(BlueprintOverride)
    EDataValidationResult ValidateLoadedAsset(UObject InAsset)
    {
        if (/* 某违规条件 */ false)
            AssetFails(InAsset, FText::FromString("说明为什么失败"));

        if (GetValidationResult() == EDataValidationResult::Invalid)
            return EDataValidationResult::Invalid;

        AssetPasses(InAsset);
        return EDataValidationResult::Valid;
    }
}
```

**常用判定函数**：`Editor::IsDeveloperOnlyPath(path)`、`Editor::IsDeprecatedAssetPath(path)`、`Editor::GetAssetDependenciesNonEditorOnly(pkg, out)`、`EditorAsset::DoesAssetExist(path)`。范本：`DeveloperContentValidator.as`。

> 相关：内容浏览器过滤器 `UHazeScriptContentBrowserFilter` 用同一套判定，只是重写 `IsAllowedByFilter(...)` 返回是否显示，见 `ReferencedDevAssetContentBrowserFilter.as`。

---

## 六、交互式视口工具 `UEditorScriptableClickDragTool`

**骨架**：声明工具栏元数据 → 在 `OnScriptSetup` / `OnScriptShutdown` 恢复/保存设置 → 重写 `TestIfCanBeginClickDrag`（判定能否拖）+ `OnDragBegin/Update/End`（拖拽逻辑）+ `OnScriptRender`（画手柄）。

```cpp
// 工具的持久化设置（跟工具一起存读）
class UMyToolPropertySet : UScriptableInteractiveToolPropertySet
{
    UPROPERTY(EditAnywhere, Category = "Settings")
    bool bSomeOption = false;
}

class UMyTool : UEditorScriptableClickDragTool
{
    default ToolCategory = FText::FromString("Placement");
    default ToolName = FText::FromString("MyTool");
    default CustomIconPath = "Editor/Slate/MenuIcons/MyTool.png";

    UMyToolPropertySet Settings;

    UFUNCTION(BlueprintOverride)
    void OnScriptSetup()
    {
        EToolsFrameworkOutcomePins Outcome;
        Settings = Cast<UMyToolPropertySet>(AddPropertySetOfType(UMyToolPropertySet, "Settings", Outcome));
        RestorePropertySetSettings(Settings, "MyTool");     // 读回上次设置
    }

    UFUNCTION(BlueprintOverride)
    void OnScriptShutdown(EToolShutdownType ShutdownType)
    {
        SavePropertySetSettings(Settings, "MyTool");        // 保存设置
    }

    // 命中测试：返回 bHit=true 才会进入拖拽
    UFUNCTION(BlueprintOverride)
    FInputRayHit TestIfCanBeginClickDrag(FInputDeviceRay ClickPos, FScriptableToolModifierStates Modifiers)
    {
        FInputRayHit Hit;
        Hit.bHit = /* 光标射线是否命中我的手柄 */ true;
        return Hit;
    }

    UFUNCTION(BlueprintOverride)
    void OnDragBegin(FInputDeviceRay Start, FScriptableToolModifierStates Modifiers)
    {
        Editor::BeginTransaction("My Tool Edit");           // 开始可撤销事务
        for (UActorComponent Comp : Editor::GetSelectedComponents())
            Comp.Modify();                                  // 登记将被修改的对象
    }

    UFUNCTION(BlueprintOverride)
    void OnDragUpdatePosition(FInputDeviceRay New, FScriptableToolModifierStates Modifiers)
    {
        // 用 New.WorldRay.Origin / .Direction 做射线-平面求交，更新组件 transform
    }

    UFUNCTION(BlueprintOverride)
    void OnDragEnd(FInputDeviceRay End, FScriptableToolModifierStates Modifiers)
    {
        Editor::EndTransaction();                           // 结束事务
    }

    UFUNCTION(BlueprintOverride)
    void OnScriptRender(UScriptableTool_RenderAPI RenderAPI)
    {
        RenderAPI.DrawLine(A, B, ColorDebug::Yellow, 10, 0, false);   // 画交互手柄
    }
}
```

**要点**
- 激活/关闭：`Blutility::ActivateEditorTool(UMyTool)` / `Blutility::DeactivateEditorTool(...)`；`Blutility::ActivateSelectionMode()` 回到普通选择。
- 射线：拖拽点位来自 `FInputDeviceRay.WorldRay`（`Origin` + `Direction`），配合 `Math::RayPlaneIntersection` / `FindNearestPointsOnLineSegments` 求交。
- 网格吸附：`Editor::IsGridEnabled()` + `Vector.GridSnap(Editor::GetGridSize())`。
- 常配一个 `UHazeEditorSubsystem`，在其 `Tick` 里监视"编辑器模式被切走"等情况自动关闭工具（见 `ScaleStretchTool.as` 底部的 `UScaleStretchEditorSubsystem`）。
- 范本：`LevelEditor/ScaleStretchTool.as`（拖包围盒边/面/顶点缩放）、`SplineEditor/HazeSplineDrawTool.as`（在地表画样条）。

---

## 七、横切机制

### 7.1 即时模式 UI：`UHazeImmediateDrawer` / `UHazeImmediateWidget`

类似 Dear ImGui：**每帧重建**整棵 UI，用链式调用拼装。三种入口拿到根节点：

- 细节面板：`AddImmediateRow(...)` 得到 `UHazeImmediateDrawer`，`Drawer.Begin()` / `Drawer.BeginVerticalBox()` 拿根。
- 工具窗口：`ImmediateWidget.GetDrawer()`，同样 `Begin*`。
- 绘制前先判 `Drawer.IsVisible()`，不可见直接 return（省开销）。

**容器 / 元素**：`.VerticalBox()` `.HorizontalBox()` `.WrapBox()` `.BorderBox()` `.ScrollBox()` `.Section()`；`.Text(...)` `.Button(...)` `.CheckBox()` `.Spacer(n)`。
**修饰（链在元素后）**：`.Color(...)` `.Bold()` `.Scale(f)` `.Tooltip(...)` `.Padding(x,y)` `.AutoWrapText()` `.MinDesiredWidth(n)`；槽位对齐前缀 `.SlotFill()` `.SlotPadding(...)` `.SlotHAlign(...)` `.SlotVAlign(...)`。

**交互靠返回值**：`.Button(...)` 当帧被点返回 `true`；`.CheckBox().Checked(state)` 隐式转 `bool` 得到新勾选值——所以写法是"读回并回写成员"：

```cpp
auto Root = Drawer.BeginVerticalBox();
if (Root.Button("🛠️ Merge Mesh").Tooltip("合并全部网格"))
    MergeMesh();

auto Check = Root.CheckBox().Label("Visualize").Checked(bVisualize);
bVisualize = Check;      // CheckBox 的返回值即新状态，回写成员
```

> 注意：即时 UI 无回调、无持久 widget 树。所有状态存在脚本成员里，UI 每帧照着成员重画。

### 7.2 可撤销事务：`FScopedTransaction` / `Editor::BeginTransaction`

任何修改场景/资产的操作都应包进事务，才能 Ctrl+Z。两种写法等价：

```cpp
// A) 作用域对象：离开作用域自动结束（适合面板按钮里的一次性操作）
{
    FScopedTransaction Transaction("Generate Merged Mesh");
    PropLine.Modify();                 // 关键：先 Modify() 登记要改的对象
    PropLine.MergedMeshes.Reset();
    PropLine.UpdatePropLine();
}

// B) 显式配对：适合跨越多个回调的拖拽（Begin 在 DragBegin，End 在 DragEnd）
Editor::BeginTransaction("Stretch Scale");
Comp.Modify();
// ……
Editor::EndTransaction();
```

**铁律**：改任何 `UObject` 属性/数组前先 `Obj.Modify()`，否则撤销记录不到旧值。改完属性可 `NotifyPropertyModified(Obj, n"Prop")` 让面板/构造脚本刷新。

### 7.3 `UFUNCTION(CallInEditor)`：把函数暴露为可点操作

`CallInEditor` 是把脚本函数变成"用户可触发操作"的通用开关：
- 在**菜单扩展**类里 → 变成右键菜单条目（见第四节）。
- 在普通 actor / 组件上 → 变成 Details 面板里的一个按钮。
- 可加 `meta = (EditorIcon = "...")` 图标、`Category = "..."` 分组。

### 7.4 编辑器专用标记：只活在编辑器里

| 标记 | 用途 |
|---|---|
| `default bIsEditorOnly = true;` | 组件/actor 仅编辑器存在，打包与游戏世界中被剥离 |
| `default SetIsVisualizationComponent(true);` | 标记为纯可视化组件（不参与游戏逻辑/序列化到游戏态） |
| `Editor::SetComponentSelectable(Comp, false)` | 让临时可视化组件不可被框选，避免误选 |
| `if (GetWorld().IsGameWorld()) return;` | 构造脚本里跳过 PIE / 运行时，只在编辑世界执行 |

```cpp
class UEditorBillboardComponent : UBillboardComponent
{
    default SetIsVisualizationComponent(true);
    default bIsEditorOnly = true;

    UFUNCTION(BlueprintOverride)
    void ConstructionScript()
    {
        if (GetWorld() != nullptr && GetWorld().IsGameWorld())
            return;                       // 游戏世界里什么都不做
        // ……只在编辑器加载图标贴图……
    }
}
```

### 7.5 一帧式可视化子系统：`EditorVisualizationSubsystem`

当 Visualizer 的画线 API 不够（要画**实心网格/盒/骨骼动画预览**）时用它。设计是**"每帧请求一次，自动回收"**：

```cpp
auto Sub = UEditorVisualizationSubsystem::Get();
Sub.DrawMesh(Instigator, Transform, SomeStaticMesh);              // 画静态网格
Sub.DrawSolidBox(Instigator, Loc, Rot, Extents, Color, 0.5);     // 画半透明实心盒
Sub.DrawAnimation(Transform, AnimSeq, Time, bApplyRootMotion);   // 画骨骼动画预览
```

- 每个请求以 `FInstigator`（或动画序列）为 key 复用同一临时 actor/组件，并记 `LastUsedFrame = GFrameNumber`。
- 子系统 `Tick` 里凡 `LastUsedFrame < GFrameNumber - 1`（上一帧没再被请求）就销毁——**调用方无需手动清理**，停止请求即自动消失。
- `mixin void DrawSolidBox(...)` 让任意 `UHazeScriptComponentVisualizer` 直接 `this.DrawSolidBox(...)` 画实心盒 + 线框边。

---

## 八、速查：新工具从零到一

1. 明确目标 → 用第 0 节表选基类。
2. 复制对应章节的骨架，替换 `VisualizedClass` / `DetailClass` / `SupportedClasses` / 工具元数据。
3. UI 用即时模式（7.1），状态存脚本成员。
4. 任何改场景/资产的操作：先 `Modify()`，包 `FScopedTransaction`（7.2）。
5. 用户可触发的操作用 `CallInEditor`（7.3）；纯编辑器对象加 `bIsEditorOnly`（7.4）。
6. 要画实心几何/动画预览走 `EditorVisualizationSubsystem`（7.5）。
7. 找最接近的现成文件当范本（各节末尾已列）。
</content>
</invoke>

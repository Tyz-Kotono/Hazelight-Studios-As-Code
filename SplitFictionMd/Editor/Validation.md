# Editor / 资产质检 — Validation

> 职责：自动拦截资产违规问题，维护项目内容卫生。是正式内容与"仅开发"内容之间的守门员。
>
> 覆盖子模块：**Validators**（1 文件）、**ContentBrowser**（4 文件）。

---

## 核心概念：Developer-only / Deprecated 资产

项目把资产分为几类路径：
- **正式资产** —— 会进入打包、最终交付给玩家的内容。
- **"仅开发"资产（developer-only）** —— 位于开发目录，仅供测试/调试，**不应**被正式资产引用，否则会被意外打包。
- **弃用资产（deprecated）** —— 已废弃，正式资产不应再依赖。

判定靠引擎静态函数：`Editor::IsDeveloperOnlyPath(path)`、`Editor::IsDeprecatedAssetPath(path)`、`Editor::GetAssetDependenciesNonEditorOnly(...)`。

**唯一例外**：`/Game/Editor/EditorBillboards/`（编辑器图标贴图）——它们只用于视口可视化，允许被引用。

---

## 一、Validators/ — 保存时自动校验（1 文件）

#### `DeveloperContentValidator`
继承 `UEditorValidatorBase`，接入引擎 Data Validation 流程，在保存/提交资产时自动运行。

- **`CanValidateAsset`** —— 限定校验范围：Niagara 系统、材质 / 材质实例、静态 / 骨骼网格、蓝图、数据资产、World。
- **`ValidateLoadedAsset`** —— 核心逻辑：
  1. 若资产本身是 developer-only，直接通过（`AssetPasses`）。
  2. 否则遍历其所有依赖：
     - 依赖了 developer-only 资产 → `AssetFails`（编辑器广告牌除外）。
     - 依赖了 deprecated 资产（而自己不是 deprecated）→ `AssetFails`。

> 作用：从根上阻止"正式内容引用了开发/废弃资产"被提交，避免误打包与脏依赖。

---

## 二、ContentBrowser/ — 内容浏览器过滤与操作（4 文件）

### 过滤器（`UHazeScriptContentBrowserFilter`）

实现 `IsAllowedByFilter(...)` 返回是否在内容浏览器中显示该资产，配 `FilterName` / `Tooltip` / `Category` / `Color`。

| 文件 | 过滤器 | 作用 |
|---|---|---|
| `ReferencedDevAssetContentBrowserFilter` | `UAssetsWithInvalidDevAssetReferencesContentBrowserFilter` | 红色高亮：**正式资产中错误引用了 developer-only 资产**的那些（与 Validator 同逻辑，但用于浏览筛查） |
| 同上文件 | `UReferencedDevAssetContentBrowserFilter` | 红色高亮：**被正式资产错误引用的 developer-only 资产**（反向视角） |
| `MoveRatioAnimationsContentBrowserFilter` | — | 筛选 ratio 动画（资产迁移/整理用） |

> Validator 是"提交时拦截"，过滤器是"主动浏览排查"——两者用同一套 developer-only 判定逻辑，互为补充。

### 操作（资产右键 Action）

| 文件 | 功能 |
|---|---|
| `DeprecateAsset` | 把资产标记为弃用（移入 deprecated 路径 / 打标） |
| `RenameAssetsActions` | 批量重命名资产 |

---

## 与可视化体系的联动

- [Visualization.md](./Visualization.md) 里的 `VisualizeDeveloperAssets` 视口模式，把场景中引用了开发资产的对象染色显示——和本模块共享同一套 developer-only 判定，形成 **"视口染色 + 浏览器过滤 + 保存校验"** 三道防线。

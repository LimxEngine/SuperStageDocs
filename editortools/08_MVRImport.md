# SuperStage MVR 导入 用户手册

## 1. 概述

MVR 导入工具（MVR Import）用于从 MVR（My Virtual Rig）文件中导入灯具布局数据。MVR 是由 GDTF 组织定义的开放标准文件格式（.mvr），广泛用于灯光设计软件之间的数据交换。通过此工具，你可以将 Vectorworks、Capture、WYSIWYG 等软件导出的灯位图直接导入到 SuperStage 场景中。

工具会按**灯具类型分组**显示解析结果，用户为每种类型指定本地 Actor Class 后一键导入。

---

## 2. 打开方式

**主菜单路径**：工具栏 **SuperStage** 下拉菜单 → **SuperDMXTool** → **MVRImport**

打开后面板最小尺寸为 900 × 500 像素。

---

## 3. 界面说明

```
┌──────────────────────────────────────────────────────┐
│  MVR File: [<Choose .mvr>                  ] [Browse]│
│                                                      │
│  [Select All] [Select None]              [Import]    │
│                                                      │
│  ┌──────────────────────────────────────────────┐    │
│  │ Import │ Fixture Type   │ Count │ Actor Class │    │
│  │   ☑   │ Spot380        │  24   │ BP_Spot  ▼  │    │
│  │   ☑   │ WashLED600     │  16   │ BP_Wash  ▼  │    │
│  │   ☑   │ BeamMoving     │   8   │ (None)   ▼  │    │
│  └──────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

---

## 4. 操作流程

### 步骤 1：选择 MVR 文件

点击 **"Browse"** 按钮，在弹出的文件对话框中选择一个 `.mvr` 文件。

支持的文件格式：
- `.mvr` — 标准 MVR 文件（ZIP 压缩包，内含 `GeneralSceneDescription.xml`）
- `.xml` — 直接选择 MVR XML 描述文件

> **自动解析**：选择文件后，工具会**自动**读取并解析文件内容，无需点击额外按钮。解析结果按灯具类型分组显示在列表中。

### 步骤 2：为每种类型指定 Actor Class

解析完成后，列表中每一行代表一种灯具类型。你需要在 **Actor Class** 列的下拉菜单中为每种类型选择对应的 SuperStage 灯具蓝图类（`ASuperDmxActorBase` 的子类）。

### 步骤 3：选择要导入的类型

使用每行的 **Import** 勾选框选择需要导入的灯具类型。也可以使用工具栏的 **Select All** / **Select None** 按钮进行批量操作。

### 步骤 4：导入

点击 **"Import"** 按钮。工具会：
1. 检查所有勾选类型是否都指定了 Actor Class（未指定会弹出提示）
2. 在 UE Transaction 中创建所有灯具 Actor
3. 设置每个灯具的位置、旋转、DMX 属性
4. 导入完成后通过通知弹窗显示结果

---

## 5. 灯具列表字段

| 列名 | 宽度 | 说明 |
|------|------|------|
| **Import** | 70px | 勾选框，是否导入该类型的灯具 |
| **Fixture Type** | 自适应 | MVR 文件中定义的灯具类型名称 |
| **Count** | 80px | 该类型灯具的数量 |
| **Actor Class** | 240px | 下拉选择本地对应的 SuperStage 灯具蓝图类 |

列表按灯具类型名称字母升序排列。

---

## 6. MVR 文件解析

### 6.1 解析流程

1. 如果是 `.mvr` 文件（ZIP 格式），工具会从压缩包中提取 `GeneralSceneDescription.xml`
2. 如果是 `.xml` 文件，直接读取 XML 内容
3. 解析 XML 中的 `<Fixture>` 节点，提取类型名称、位置、旋转和 DMX 地址
4. 按灯具类型分组统计

### 6.2 ZIP 解压策略

工具支持多种解压方式以保证兼容性：
- **优先**：内存中直接解压（支持 Stored 和 Deflate 压缩方式）
- **回退**：使用系统工具（Windows 上调用 PowerShell `Expand-Archive` 或 `tar.exe`）

### 6.3 编码支持

XML 文件支持以下编码：UTF-8、UTF-16 LE、UTF-16 BE。

---

## 7. DMX 地址解析

MVR 文件中的 DMX 地址信息支持多种格式，工具会自动识别：

| 格式 | 示例 | 说明 |
|------|------|------|
| **Break 属性** | `DMXBreakOverride` | 从 XML 节点属性中读取 |
| **绝对地址** | `1025` | 自动计算为 Universe 3, Address 1 |
| **U.A 格式** | `1.001` | Universe 1, Address 1 |

---

## 8. 导入后的操作

灯具导入到场景后：
- 灯具会放置在 MVR 文件记录的 3D 位置（自动坐标转换）
- 灯具会使用 MVR 文件中的旋转信息
- DMX 属性（Universe、StartAddress、FixtureID）会自动设置
- 灯具使用 MVR 中的标签作为 Actor Label

你可以在导入后使用 SuperStage 的其他工具（批量 Patch、Patch 预览等）进一步调整灯具配置。

---

## 9. 支持的 MVR 版本

| 版本 | 支持状态 |
|------|----------|
| MVR 1.0 | ✅ 完全支持 |
| MVR 1.4 | ✅ 完全支持 |
| MVR 1.5 | ✅ 完全支持 |
| MVR 1.6 | ✅ 完全支持 |

---

## 10. 注意事项

- 选择文件后会自动解析，无需手动触发
- 每种灯具类型**必须**指定 Actor Class 才能导入，否则会弹出错误提示
- 同时支持 `.mvr`（ZIP）和 `.xml`（直接 XML）两种文件格式
- MVR 中的 GDTF 灯具描述文件不会自动导入到 SuperStage 灯库
- 大型 MVR 文件（包含数百个灯具）解析可能需要几秒钟
- 导入操作支持 **Ctrl+Z 撤销**（在 UE Transaction 中执行）

---

## 11. 常见问题

| 问题 | 解决方法 |
|------|----------|
| 解析后列表为空 | 确认 MVR 文件中确实包含灯具数据 |
| Actor Class 下拉为空 | 确认项目中有继承自 `ASuperDmxActorBase` 的蓝图类 |
| 点击 Import 弹出提示 | 所有勾选的类型都必须指定 Actor Class |
| 灯具位置全部在原点 | MVR 文件可能没有包含位置信息 |
| 灯具方向不对 | 不同软件的坐标系可能不同，导入后手动调整旋转 |
| 无法打开 .mvr 文件 | 确认文件未损坏，或尝试解压后选择内部的 .xml 文件 |

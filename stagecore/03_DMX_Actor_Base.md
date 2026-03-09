# 03 - DMX 灯具基础 (SuperDmxActorBase)

> **所属模块**: SuperStage 运行时 (ASuperDmxActorBase)  
> **适用对象**: 灯光设计师、蓝图开发者  
> **前置阅读**: [02 - 灯具库](/docs/stage-core/fixture-library)  
> **最后更新**: 2026-03-06

---

## 一、概述

**SuperDmxActorBase** 是 SuperStage 中所有 DMX 灯具的基础类。场景中每一台需要响应 DMX 信号的灯具——无论是电脑灯、LED 面板、烟机还是自定义设备——都继承自这个基类。

它提供以下核心能力：
- **DMX 地址分配**（Universe、起始地址、灯具 ID）
- **DMX 通道读取**（从网络接收的数据中提取通道值，支持 8/16/24 位精度）
- **灯具库关联**（与 Fixture Library 资产绑定，定义通道表）
- **矩阵批量读取**（一次性读取所有模块实例的同名属性值）
- **地址标签显示**（在场景中直观显示灯具的 DMX 信息）
- **每帧自动更新**（Tick 驱动，实时响应 DMX 变化，编辑器中也生效）

---

## 二、类继承链

```
AActor (UE5 引擎基类)
  │
  └── ASuperBaseActor ·············· SuperStage 根基类 (L1)
        │   ├ USceneComponent* SceneBase          ← 根组件
        │   ├ UArrowComponent* ForwardArrow       ← +X 方向预览箭头
        │   ├ UArrowComponent* UpArrow            ← +Z 方向预览箭头
        │   ├ FAssetMetaData AssetData             ← 资产元数据（UUID/制造商/分组）
        │   └ bool bPreviewAssetOrientation        ← 方向预览开关
        │
        └── ASuperDmxActorBase ····· DMX 灯具基类 (L2)  ← 本文档描述的类
              │   ├ FSuperDMXFixture SuperDMXFixture     ← DMX 地址（Universe/Address/FixtureID）
              │   ├ USuperFixtureLibrary* FixtureLibrary ← 灯具库资产引用
              │   ├ UTextRenderComponent* AddressLabel   ← 地址码文本组件
              │   ├ GetChannelValue / GetAttributeRaw8/16/24ByIndex   ← 通道值读取
              │   ├ GetMatrixAttributeRaw / Raw16        ← 矩阵批量读取
              │   ├ GetSuperDmxAttributeValue            ← 归一化属性值读取
              │   ├ GetSuperDMXColorValue                ← RGB 颜色快捷读取
              │   ├ SuperDMXTick(DeltaTime)              ← 蓝图每帧事件
              │   ├ ForceRefreshDMX()                    ← 编辑器强制刷新
              │   └ OnSuperDMXInitialized                ← 初始化完成委托
              │
              ├── ASuperLightBase ···── 电脑灯基类 (L3)       → 文档 04
              ├── ASuperDMXCamera ····── DMX 摄像机            → 文档 05
              ├── ASuperLiftingMachinery ── 升降机械（6轴）    → 文档 10
              ├── ASuperRailMachinery ···── 轨道机械（7轴）    → 文档 10
              └── ASuperLightStripEffect ── LED灯带特效       → 文档 13
```

### 组件层级图

```
ASuperDmxActorBase
  └── SceneBase (USceneComponent)                    ← 根组件（继承自 ASuperBaseActor）
        ├── ForwardArrow (UArrowComponent)            ← +X 方向预览（红色箭头）
        ├── UpArrow (UArrowComponent)                 ← +Z 方向预览（蓝色箭头）
        └── AddressLabel (UTextRenderComponent)       ← DMX 地址码文本
              │  文字颜色: 浅蓝色 (135, 206, 250)
              │  文字大小: 16 世界单位
              │  对齐: 水平居中，垂直顶部
              └  仅编辑器可见（HiddenInGame = true）
```

> **设计要点**：`ASuperDmxActorBase` 本身不创建任何可视网格组件。3D 灯具模型由子类（如 `ASuperStageLight`）自行添加。这使得基类保持轻量，适用于任何需要 DMX 数据读取的场景。

---

## 三、DMX Patch 配置

选中场景中的灯具 Actor，在细节面板中找到 **DMX** 分组，可以看到以下配置项：

### 3.1 核心 Patch 参数

| 参数 | 位置 | 说明 | 范围 | 默认值 |
|------|------|------|------|--------|
| **Universe** | SuperDMXFixture > Universe | 灯具所在的 DMX 域 | 1 - 32,767 | 1 |
| **Start Address** | SuperDMXFixture > StartAddress | 灯具在该域中的起始通道地址 | 1 - 512 | 1 |
| **Fixture ID** | SuperDMXFixture > FixtureID | 灯具的唯一编号 | 0 - 任意正整数 | 0 |

#### Universe（域）

- 决定灯具从哪个 DMX 域读取数据
- 必须与控台中该灯具分配的 Universe 一致
- 同一场景中可以使用多个 Universe

#### Start Address（起始地址）

- 灯具从该地址开始连续占用通道
- 占用的通道数量由灯具库（Fixture Library）决定
- 同一 Universe 中不同灯具的地址范围**不应重叠**

> **示例**：一台占用 20 通道的灯具，Start Address 设为 101，则它占用通道 101-120。下一台灯具的 Start Address 至少应从 121 开始。

#### Fixture ID（灯具 ID）

- 灯具的唯一识别编号
- 与 MA 控台中的 Fixture ID 对应
- 在导出 MA 宏文件时使用
- 在 Patch 工具中自动分配（避免冲突）

### 3.2 Fixture Library（灯具库）

| 参数 | 说明 |
|------|------|
| **Fixture Library** | 关联的灯具库资产（USuperFixtureLibrary） |

从下拉列表中选择灯具对应的灯具库。灯具库定义了通道表，告诉系统每个通道控制什么功能。

> **重要**：如果没有关联灯具库，灯具仍然可以通过通道号直接读取 DMX 数据，但无法使用属性名称方式读取。

---

## 四、地址标签 (Address Label)

SuperStage 可以在场景视口中每台灯具旁边显示 DMX 地址信息标签，帮助你快速识别灯具。

### 4.1 标签显示控制

| 参数 | 说明 | 默认值 |
|------|------|--------|
| **Show Address Label** | 是否在场景中显示地址标签 | 开启 |
| **Include Fixture ID In Label** | 标签中是否包含 Fixture ID | 开启 |
| **Address Label Offset** | 标签相对灯具的偏移位置（局部坐标） | (0, 0, 0) |

### 4.2 标签格式

标签文字格式如下：

- **包含 Fixture ID 时**：`UA.{Universe}.{StartAddress}  ID.{FixtureID}`
  - 示例：`UA.1.101  ID.201`
- **不包含 Fixture ID 时**：`UA.{Universe}.{StartAddress}`
  - 示例：`UA.1.101`

### 4.3 标签外观

| 属性 | 值 |
|------|-----|
| **文字颜色** | 浅蓝色 (135, 206, 250) |
| **文字大小** | 16 世界单位 |
| **对齐方式** | 水平居中，垂直顶部对齐 |
| **游戏中可见** | 否（仅编辑器中可见） |

### 4.4 标签位置

标签默认放置在灯具的前方方向（由 ForwardArrow 组件决定），并可通过 **Address Label Offset** 进行微调。

- 标签始终保持水平（Pitch/Roll = 0），仅 Yaw 跟随灯具前方方向
- 这确保标签从各个角度都容易阅读

---

## 五、资产元数据 (Asset Meta Data)

每台灯具 Actor 都继承自 SuperBaseActor，包含以下元数据信息（在细节面板的 **Asset** 分组中）：

| 参数 | 说明 | 用途 |
|------|------|------|
| **UUID** | 资产唯一标识符 | 内部使用 |
| **Group** | 资产分组 | 内容浏览器分类 |
| **Manufacturer** | 制造商名称 | CAD 施工图、报表 |
| **DMX Mode** | DMX 模式名称 | 标识灯具当前使用的通道模式 |
| **Display Name** | 显示名称 | UI 显示 |
| **Thumbnail** | 缩略图 | 内容浏览器预览 |

---

## 六、DMX 数据读取方式

灯具 Actor 提供多种方式读取 DMX 数据。以下是面向蓝图设计师的说明：

### 6.1 按通道号读取（简单模式）

**适用场景**：不需要灯具库，直接按通道偏移读取。

在蓝图中调用 **Get Channel Value** 节点：
- **输入**：通道号（相对于 Start Address 的偏移，从 1 开始）
- **输出**：浮点值 0.0 - 255.0

**示例**：Start Address = 101，调用 GetChannelValue(5) → 读取通道 105 的值

### 6.2 按属性名读取（推荐模式）

**适用场景**：已关联灯具库，通过属性名称读取归一化值。

在蓝图中调用 **Get Super DMX Attribute Value** 节点：
- **输入**：DMX Attribute（包含属性名称和实例索引）
- **输出**：浮点值 0.0 - 1.0（归一化后的值）

**归一化规则**：

| 通道类型 | 原始值范围 | 归一化公式 | 输出范围 |
|---------|-----------|-----------|---------|
| Coarse (8位) | 0 - 255 | 值 ÷ 255 | 0.0 - 1.0 |
| Fine (16位) | 0 - 65,535 | 值 ÷ 65,535 | 0.0 - 1.0 |
| Ultra (24位) | 0 - 16,777,215 | 值 ÷ 16,777,215 | 0.0 - 1.0 |

> **好处**：无论属性是 8 位还是 16 位精度，输出始终是 0.0 ~ 1.0，蓝图逻辑不需要关心位深差异。

### 6.3 按属性名读取原始值（不归一化）

在蓝图中调用 **Get Super DMX Attribute Value No Conversion** 节点：
- **输出**：浮点值，为原始 DMX 值（8位: 0-255, 16位: 0-65535, 24位: 0-16777215）

### 6.4 读取颜色

在蓝图中调用 **Get Super DMX Color Value** 节点：
- **输入**：三个 DMX Attribute（Red、Green、Blue）
- **输出**：FLinearColor（RGBA，A 始终为 1.0）

这是读取 RGB 颜色的快捷方式，等同于分别读取 R、G、B 三个属性的归一化值。

### 6.5 矩阵读取

对于矩阵灯（多个模块实例），可以一次性读取所有实例的同名属性：

**Get Matrix Attribute Raw** 节点：
- **输入**：属性名称（如 `Red`）
- **输出**：浮点数组（每个元素对应一个模块实例的值，0.0 - 1.0）
- **排序选项**：可选择按 Patch 顺序排序

**Get Matrix Attribute Raw 16** 节点：
- 与上述相同，但用于 16 位精度属性

---

## 七、DMX Attribute 参数结构

在蓝图中使用属性读取节点时，需要配置 **FSuperDMXAttribute** 结构：

| 参数 | 说明 | 示例 |
|------|------|------|
| **Attrib Name** | 属性名称（与灯具库中的定义一致） | `Dimmer`、`Pan`、`Tilt` |
| **Instance Index** | 模块实例索引（从 0 开始） | 主模块 = 0，矩阵像素从 1 开始 |
| **DMX Channel Type** | 读取精度 | Coarse / Fine / Ultra |

> **提示**：Instance Index 和 DMX Channel Type 通常在灯具蓝图的默认值中设好，运行时不需要修改。

---

## 八、SuperDMXTick 蓝图事件

每台灯具 Actor 每帧都会触发 **SuperDMXTick** 蓝图事件。这是灯具响应 DMX 数据的核心入口点。

### 8.1 工作原理

```
每帧 Tick
  ↓
触发 SuperDMXTick(DeltaTime) 蓝图事件
  ↓
蓝图中读取 DMX 属性值
  ↓
将值应用到灯具参数（光照强度、颜色、旋转等）
  ↓
UE 渲染更新
```

### 8.2 关键特性

- **编辑器中也会触发**：即使不点"Play"，只要编辑器打开，灯具就会实时响应 DMX
- **自动 Tick**：不需要手动启用，放入场景即自动工作
- **编辑器脚本安全**：内部使用 FEditorScriptExecutionGuard 保护

### 8.3 Force Refresh DMX

当需要立即刷新灯具状态时（例如从控台切换了一个 Cue），可以调用 **Force Refresh DMX**：

- 立即执行一次 SuperDMXTick
- 标记渲染状态需要更新
- 触发属性变化通知（更新材质参数等）

> **注意**：通常不需要手动调用。系统会自动在需要时批量刷新。

---

## 九、通道跨度 (Channel Span)

灯具的**通道跨度**是指该灯具从第一个通道到最后一个通道的长度。

**计算方式**：最大通道地址 - 最小通道地址 + 1

**示例**：
- 一台灯具的属性占用通道 1、2、3、4、5（Dimmer）和 6、7（Pan 16位）
- 通道跨度 = 7 - 1 + 1 = **7**

通道跨度在以下场景中使用：
- **Patch 工具**：自动计算下一台灯具的起始地址
- **CAD 施工图**：显示灯具信息
- **MA 导出**：生成正确的 Patch 信息

---

## 十、灯具蓝图开发指南

### 10.1 创建自定义灯具蓝图

1. 在 Content Browser 中右键 → **Blueprint Class**
2. 选择 **SuperDmxActorBase**（或 **SuperLightBase** 如果是电脑灯）作为父类
3. 命名蓝图（例如 `BP_MyCustomFixture`）
4. 在蓝图默认值中设置 Fixture Library
5. 在事件图中实现 **SuperDMXTick** 事件

### 10.2 SuperDMXTick 典型实现

```
Event SuperDMXTick(DeltaTime)
  │
  ├─ GetSuperDmxAttributeValue("Dimmer") → 设置灯光强度
  ├─ GetSuperDmxAttributeValue("Red")   ─┐
  ├─ GetSuperDmxAttributeValue("Green") ─┤→ 设置灯光颜色
  ├─ GetSuperDmxAttributeValue("Blue")  ─┘
  ├─ GetSuperDmxAttributeValue("Pan")   → 设置水平旋转
  └─ GetSuperDmxAttributeValue("Tilt")  → 设置垂直旋转
```

### 10.3 注意事项

- 属性名必须与灯具库中定义的名称**完全一致**（区分大小写）
- 归一化后的值 (0.0-1.0) 需要根据灯具物理参数映射到实际范围
  - 例如：Pan 值 0.0-1.0 → -270° ~ +270°
  - 例如：Dimmer 值 0.0-1.0 → 亮度 0% ~ 100%

---

## 十一、常见问题

### Q: 灯具在编辑器中不更新？
确认灯具蓝图中实现了 SuperDMXTick 事件，并且 DMX 配置面板中输入已启用。

### Q: 地址标签显示不正确？
修改灯具的 Universe / Start Address / Fixture ID 后，标签会在下一帧自动更新。如果没有更新，尝试移动一下灯具触发 OnConstruction。

### Q: 多台灯具使用同一个 Fixture Library 可以吗？
可以。灯具库是共享的配置文件。同型号的灯具应该使用同一个灯具库，只需设置不同的 Universe 和 Start Address。

### Q: 如何知道我的灯具占用了哪些通道？
- 查看灯具库中各属性的 Coarse/Fine/Ultra 偏移值
- 使用 Patch 预览工具查看所有灯具的地址占用情况
- 通道跨度 (Channel Span) 可以在属性面板中查看

---

> **下一步**：请阅读 [04 - 电脑灯](/docs/stage-core/computer-light) 了解 Pan/Tilt 控制和矩阵灯的使用。

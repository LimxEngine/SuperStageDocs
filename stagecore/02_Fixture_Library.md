# 02 - 灯具库 (Fixture Library)

> **所属模块**: SuperFixtureLibrary (USuperFixtureLibrary)  
> **适用对象**: 灯光设计师、灯具蓝图制作者  
> **前置阅读**: [00 - DMX 系统总览](/docs/stage-core/overview)  
> **最后更新**: 2026-03-06

---

## 一、什么是灯具库

**灯具库 (Fixture Library)** 是 SuperStage 中定义灯具 DMX 通道表的配置文件。它告诉系统：

- 这台灯具有哪些可控功能（亮度、Pan、Tilt、颜色、Gobo 等）
- 每个功能占用哪个通道（相对偏移）
- 每个功能的精度（8位/16位/24位）
- 每个功能中不同值范围对应的含义（例如：通道值 0-127 = 白光开，128-255 = 频闪）

简单来说，灯具库就是灯具的"说明书"——告诉 SuperStage 如何理解控台发来的 DMX 数据。

> **类比**：灯具库就像是 MA 控台中的 Fixture Profile，或 GDTF 文件中的通道定义。

---

## 二、创建灯具库资产

### 步骤

1. 在 Content Browser 中**右键** → **Miscellaneous** → **Data Asset**
2. 在弹出的类选择窗口中，选择 **SuperFixtureLibrary**
3. 为资产命名（建议使用灯具型号名，例如 `FL_ClayPaky_Sharpy`）
4. 双击打开资产进行编辑

### 命名建议

| 前缀 | 示例 | 说明 |
|------|------|------|
| `FL_` | `FL_Robe_MegaPointe` | Fixture Library 的缩写 |
| `FL_制造商_型号` | `FL_Chauvet_MaverickMK3` | 便于识别 |

---

## 三、灯具库属性详解

打开灯具库资产后，在细节面板中可以看到以下属性：

### 3.1 基本信息

| 属性 | 说明 | 示例 |
|------|------|------|
| **Fixture Name** | 灯具型号名称 | `MegaPointe` |
| **Manufacturer** | 制造商 | `Robe` |
| **Source** | 数据来源（可选） | `GDTF` 或 `Manual` |
| **Power** | 额定功率（瓦特） | `1700` |
| **Weight** | 重量（千克） | `36.8` |

这些信息主要用于 CAD 施工图和报表统计，不影响 DMX 控制逻辑。

### 3.2 模块实例列表 (Modules)

灯具库最核心的部分是 **Modules（模块实例列表）**。

每个模块实例代表灯具的一个功能模组。对于大多数灯具，只需要 **一个模块实例**。对于矩阵灯（如 LED 面板灯），每个灯珠/像素点就是一个独立的模块实例。

点击 **Modules** 旁边的 **+** 号添加模块实例。

#### 模块实例属性

| 属性 | 说明 | 示例 |
|------|------|------|
| **Module Name** | 模块名称（可选，便于识别） | `Main` 或 `Pixel_1` |
| **Patch** | 模块在灯具中的通道偏移（1 基）。对于单模块灯具，设为 **1**。对于矩阵灯，每个模块的 Patch 值不同 | `1` |
| **Attribute Defs** | 该模块的属性定义列表（详见下文） | — |

---

## 四、属性定义 (Attribute Defs)

每个模块实例中可以添加多个**属性定义 (Attribute Def)**，每个属性对应灯具的一个可控功能。

点击 **Attribute Defs** 旁边的 **+** 号添加属性。

### 4.1 属性基本参数

| 参数 | 说明 | 详细解释 |
|------|------|---------|
| **Attrib Name** | 属性名称（FName） | 必须唯一。这是蓝图中引用此属性的标识符。建议使用标准命名：`Dimmer`、`Pan`、`Tilt`、`Red`、`Green`、`Blue`、`White`、`Gobo`、`Prism`、`Focus`、`Zoom`、`Shutter`、`ColorWheel` 等 |
| **Coarse** | 粗调通道偏移（1 基） | **必填**。该属性的主通道在模块内的偏移。例如 Coarse=5 表示该属性从模块的第 5 个通道开始 |
| **Fine** | 精调通道偏移（1 基） | **可选**。设为 0 表示无精调通道。用于 16 位精度控制（如 Pan/Tilt） |
| **Ultra** | 超精调通道偏移（1 基） | **可选**。设为 0 表示无超精调通道。用于 24 位精度控制（极少使用） |
| **Category** | 属性分类 | 用于 UI 分组显示，不影响功能逻辑 |
| **Channel Type** | 通道精度类型 | 决定读取方式：Coarse（8位）/ Fine（16位）/ Ultra（24位） |

### 4.2 通道偏移详解

通道偏移是**相对于模块 Patch 的偏移**，从 1 开始计数。

**地址计算公式**：
```
绝对通道地址 = 灯具起始地址 (StartAddress) + 模块偏移 (Patch - 1) + 属性偏移 (Coarse - 1)
```

**示例**：一台灯具的 StartAddress = 101，模块 Patch = 1

| 属性 | Coarse | Fine | 实际占用通道 |
|------|--------|------|-------------|
| Dimmer | 1 | 0 | 101 |
| Pan | 2 | 3 | 102-103 |
| Tilt | 4 | 5 | 104-105 |
| Color Wheel | 6 | 0 | 106 |
| Gobo 1 | 7 | 8 | 107-108 |
| Prism | 9 | 0 | 109 |

### 4.3 通道精度类型 (Channel Type)

| 类型 | 位深 | 使用的通道 | 值范围 | 适用场景 |
|------|------|-----------|--------|---------|
| **Coarse** | 8 位 | 仅 Coarse 通道 | 0 - 255 | 开关型功能（Gobo、棱镜、颜色轮等） |
| **Fine** | 16 位 | Coarse + Fine 通道 | 0 - 65,535 | 需要平滑运动的功能（Pan、Tilt、Zoom 等） |
| **Ultra** | 24 位 | Coarse + Fine + Ultra 通道 | 0 - 16,777,215 | 极高精度需求（极少使用） |

> **重要**：Channel Type 决定了蓝图中读取该属性时的归一化方式：
> - Coarse → 原始值 ÷ 255 = 0.0 ~ 1.0
> - Fine → 原始值 ÷ 65,535 = 0.0 ~ 1.0
> - Ultra → 原始值 ÷ 16,777,215 = 0.0 ~ 1.0

#### 通道精度快速选择指南

| 属性类型 | 推荐精度 | 理由 |
|---------|---------|------|
| **Dimmer（亮度）** | Fine (16位) | 低亮度时 8 位精度会出现可见跳变 |
| **Pan / Tilt** | Fine (16位) | 旋转运动需要平滑过渡，8 位精度仅 256 步远不够 |
| **Zoom / Focus / Iris** | Fine (16位) | 光学参数调整需要精细控制 |
| **颜色轮 (Color Wheel)** | Coarse (8位) | 离散选择（6-12 个颜色），不需要连续过渡 |
| **Gobo 选择** | Coarse (8位) | 离散选择（6-12 个图案） |
| **Gobo 旋转** | Fine (16位) | 连续旋转速度需要平滑 |
| **棱镜 (Prism)** | Coarse (8位) | 离散选择（开/关/旋转方向） |
| **频闪 (Strobe)** | Coarse (8位) | 频闪速度通常 8 位足够 |
| **RGB / RGBW** | Coarse (8位) | 颜色混合通常 8 位（256 级）足够 |
| **CMY** | Coarse (8位) | 同 RGB |
| **CTO / CTB** | Coarse (8位) | 色温调整通常 8 位足够 |
| **控制通道 (Reset/Lamp)** | Coarse (8位) | 指令型通道，无需精调 |

> **经验法则**：如果灯具说明书中为某个属性分配了 2 个通道（Coarse + Fine），就使用 Fine。如果只分配了 1 个通道，就使用 Coarse。严格按照灯具说明书配置。

### 4.4 属性分类 (Category)

| 分类 | 说明 | 典型属性 |
|------|------|---------|
| **Intensity** | 亮度相关 | Dimmer, Shutter, Strobe |
| **Position** | 位置相关 | Pan, Tilt, PanRot, TiltRot |
| **Color** | 颜色相关 | Red, Green, Blue, White, Amber, ColorWheel, CTO, CTB |
| **Beam** | 光束相关 | Zoom, Focus, Iris, Frost |
| **Gobo** | Gobo 相关 | Gobo1, Gobo2, GoboRot |
| **Effect** | 效果相关 | Prism, PrismRot, Animation |
| **Control** | 控制相关 | Reset, LampOn, LampOff |
| **Custom** | 自定义 | 任何其他功能 |

---

## 五、子属性 (Sub-Attributes)

每个属性定义中可以添加**子属性 (Sub-Attributes)**，用于定义通道值范围内的功能细分。这类似于 MA 控台中的"通道集 (Channel Set)"。

### 5.1 子属性参数

| 参数 | 说明 | 示例 |
|------|------|------|
| **Name** | 子属性名称 | `Open White`、`Gobo 1`、`Strobe Slow-Fast` |
| **DMX Range Min** | DMX 值范围的最小值（0-255） | `0` |
| **DMX Range Max** | DMX 值范围的最大值（0-255） | `127` |
| **Physical Min** | 物理值最小值（真实单位） | `0.0`（度/米/百分比等） |
| **Physical Max** | 物理值最大值（真实单位） | `540.0` |

### 5.2 子属性示例

**Gobo 通道的子属性定义**：

| 子属性名称 | DMX Min | DMX Max | 说明 |
|-----------|---------|---------|------|
| Open | 0 | 7 | 无 Gobo（白光） |
| Gobo 1 | 8 | 15 | 第一个 Gobo 图案 |
| Gobo 2 | 16 | 23 | 第二个 Gobo 图案 |
| Gobo 3 | 24 | 31 | 第三个 Gobo 图案 |
| Gobo 1 Spin CW | 32 | 95 | Gobo 1 顺时针旋转（慢→快） |
| Gobo 1 Spin CCW | 96 | 159 | Gobo 1 逆时针旋转（慢→快） |

**Pan 通道的子属性定义**：

| 子属性名称 | DMX Min | DMX Max | Physical Min | Physical Max | 说明 |
|-----------|---------|---------|-------------|-------------|------|
| Pan Range | 0 | 255 | -270.0 | 270.0 | Pan 角度范围 ±270° |

---

## 六、通道集 (Channel Sets)

**通道集 (Channel Set)** 是一种快捷方式，类似于 MA2 的 Channel Set 功能。它为属性定义预设的命名值。

### 6.1 通道集参数

| 参数 | 说明 | 示例 |
|------|------|------|
| **Name** | 通道集名称 | `Open`、`Slow`、`Fast` |
| **DMX Value** | 对应的 DMX 值（0-255） | `128` |

### 6.2 使用场景

通道集通常用于快速访问特定功能值，例如：

- Gobo 通道：Open=0, Gobo1=10, Gobo2=20 ...
- Shutter 通道：Open=255, Closed=0, Strobe=128

---

## 七、矩阵灯配置

对于 LED 矩阵灯（如 Robe Spiider、Ayrton MagicPanel 等），每个灯珠/像素点需要定义为独立的模块实例。

### 7.1 矩阵配置步骤

1. 确定灯具的像素数量（例如 7 个灯珠）
2. 确定每个像素的通道布局（例如每个像素 4 通道：RGBW）
3. 确定主控通道的数量（例如 Dimmer、Pan、Tilt 等占用前 16 通道）

**示例**：一台 7 珠 LED 矩阵灯，主控 16 通道，每珠 RGBW 4 通道

| 模块实例 | Module Name | Patch | 属性 |
|---------|-------------|-------|------|
| 0 | Main | 1 | Dimmer(1), Pan(2/3), Tilt(4/5), ... |
| 1 | Pixel 1 | 17 | Red(1), Green(2), Blue(3), White(4) |
| 2 | Pixel 2 | 21 | Red(1), Green(2), Blue(3), White(4) |
| 3 | Pixel 3 | 25 | Red(1), Green(2), Blue(3), White(4) |
| 4 | Pixel 4 | 29 | Red(1), Green(2), Blue(3), White(4) |
| 5 | Pixel 5 | 33 | Red(1), Green(2), Blue(3), White(4) |
| 6 | Pixel 6 | 37 | Red(1), Green(2), Blue(3), White(4) |
| 7 | Pixel 7 | 41 | Red(1), Green(2), Blue(3), White(4) |

> **注意**：每个像素模块的 Patch 值 = 主控通道数 + (像素索引 × 像素通道数) + 1

### 7.2 矩阵属性读取

配置完矩阵后，蓝图中可以使用矩阵读取功能一次性获取所有像素的同名属性值（例如获取所有像素的 Red 值），返回一个数组。详见 [04 - 电脑灯](/docs/stage-core/computer-light) 中的矩阵控制部分。

---

## 八、灯具库与灯具 Actor 的关联

创建好灯具库后，需要将其关联到场景中的灯具 Actor：

1. 选中场景中的灯具 Actor
2. 在细节面板中找到 **Fixture Library** 属性
3. 从下拉列表中选择对应的灯具库资产

> **提示**：如果是自制灯具蓝图，建议在蓝图的默认值中预设好 Fixture Library，这样放置到场景中时就自动关联了。

---

## 九、内置灯具库

SuperStage 在 `Content/LightingLibrary/` 目录下提供了多个品牌的预制灯具库：

| 品牌 | 目录 | 说明 |
|------|------|------|
| **Acme** | `LightingLibrary/Acme/` | Acme 灯具系列 |
| **Chauvet** | `LightingLibrary/Chauvet/` | Chauvet 灯具系列 |
| **ClayPaky** | `LightingLibrary/ClayPaky/` | Clay Paky 灯具系列 |
| 其他 | `LightingLibrary/...` | 更多品牌 |

你可以直接使用这些预制灯具库，也可以复制一份作为自定义灯具库的起点。

---

## 十、最佳实践

### 灯具库命名
- 使用 `FL_品牌_型号_模式` 格式，例如 `FL_Robe_MegaPointe_Standard`
- 同一灯具不同 DMX 模式应创建不同的灯具库

### 通道偏移
- 严格按照灯具说明书中的通道表填写
- 注意偏移从 **1** 开始（不是 0）
- Fine 通道通常紧跟在 Coarse 通道后面

### 属性命名
- 使用标准化名称（Dimmer、Pan、Tilt、Red、Green、Blue...）
- 同一项目中保持命名一致，便于蓝图复用
- 属性名称区分大小写

### 验证
- 配置完成后，在场景中放置灯具，连接控台
- 逐个通道推值，确认每个属性响应正确
- 使用 DMX 活动监视器辅助调试

---

## 十一、常见问题

### Q: 灯具不响应某个通道？
检查该通道的 Coarse 偏移值是否正确。注意偏移从 1 开始。

### Q: Pan/Tilt 运动不够平滑？
确保 Fine 通道已正确设置，且 Channel Type 设为 Fine（16位）。

### Q: 矩阵灯只有第一个像素响应？
检查每个像素模块实例的 Patch 值是否正确计算。

### Q: 如何知道灯具占用了多少个通道？
灯具的通道跨度（Channel Span）会在 Patch 工具和 CAD 视图中自动计算显示。它等于所有模块中最大通道地址 - 最小通道地址 + 1。

---

> **下一步**：请阅读 [03 - DMX 灯具基础](/docs/stage-core/dmx-actor-base) 了解如何在场景中配置灯具的 DMX 地址。

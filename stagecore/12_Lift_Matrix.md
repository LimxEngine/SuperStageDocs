# 12 - 升降矩阵 (Lift Matrix)

> **所属模块**: SuperStage 运行时 (ASuperLiftMatrix)  
> **适用对象**: 灯光设计师、舞美技术人员  
> **前置阅读**: [04 - 电脑灯](/docs/stage-core/computer-light)  
> **最后更新**: 2026-03-06

---

## 一、概述

**SuperLiftMatrix** 是一种特殊的舞台灯具 Actor，将多个发光组件垂直排列在钢丝绳悬挂结构上，通过 DMX 控制实现**升降展开/收拢**和**独立颜色/效果控制**。它模拟了演出中常见的升降矩阵灯阵（Kinetic Light）。

### 类继承链

```
AActor (UE5 引擎基类)
  │
  └── ASuperBaseActor ·············· SuperStage 根基类 (L1)
        │
        └── ASuperDmxActorBase ····· DMX 灯具基类 (L2)
              │                      ├ DMX 地址 / 灯具库 / 通道读取
              │                      └ SuperDMXTick / 矩阵读取
              │
              └── ASuperLightBase ·· 电脑灯基类 (L3)
                    │                 ├ Pan/Tilt 控制 / 无极旋转
                    │                 ├ LiftRange 升降控制
                    │                 └ PTSpeed 插值速度
                    │
                    └── ASuperLiftMatrix ·· 升降矩阵 (L4) ← 本文档
                          ├ 5 层 LiftComponent（各含 2 个 EffectComponent）
                          ├ 4 根 CableComponent 钢丝绳
                          ├ 升降展开/收拢控制（InPosZ）
                          ├ 矩阵效果 + 矩阵颜色独立控制
                          └ MaxLightIntensity 亮度上限
```

> **设计要点**：`ASuperLiftMatrix` 继承自 `ASuperLightBase`，因此它天然拥有 Pan/Tilt 旋转能力——整个升降矩阵灯阵可以在旋转的同时做升降展开动作。升降功能复用了父类的 `InPosZ` 通道和 `LiftRange` 参数。

### 物理结构

```
    ┌─ CableOrigin（钢丝绳起点，固定在顶部）
    │
    ║  ║                    ←── 4根钢丝绳（四角悬挂）
    ║  ║
    ├──┤ LiftComponent_0   ←── 第1层（各含2个EffectComponent）
    ║  ║
    ├──┤ LiftComponent_1   ←── 第2层
    ║  ║
    ├──┤ LiftComponent_2   ←── 第3层
    ║  ║
    ├──┤ LiftComponent_3   ←── 第4层
    ║  ║
    └──┘ LiftComponent_4   ←── 第5层
```

### 核心特点

- **5 层发光组件** — 垂直排列，每层包含 2 个 SuperEffectComponent
- **4 根钢丝绳** — 从顶部四角延伸到最底层，长度随升降自动调整
- **DMX 升降控制** — 通过一个 DMX 通道控制整体展开/收拢
- **矩阵效果控制** — 每层可独立控制效果和颜色（通过矩阵 DMX 读取）
- **继承 Pan/Tilt** — 继承自 SuperLightBase，支持整体旋转

### 应用场景

- 升降矩阵灯阵（Kinetic Light）
- 动态吊挂灯具装置
- 互动装置艺术
- 舞台背景动态灯光装饰

---

## 二、组件层级

```
Actor Root
  ├── CableOrigin（钢丝绳起点，高度 15cm）
  │     ├── Cable_0（左前钢丝绳）
  │     ├── Cable_1（右前钢丝绳）
  │     ├── Cable_2（左后钢丝绳）
  │     └── Cable_3（右后钢丝绳）
  │
  ├── LiftComponent_0（第1层）
  │     ├── EffectComponentA_0
  │     └── EffectComponentB_0
  ├── LiftComponent_1（第2层）
  │     ├── EffectComponentA_1
  │     └── EffectComponentB_1
  ├── LiftComponent_2（第3层）
  │     ├── EffectComponentA_2
  │     └── EffectComponentB_2
  ├── LiftComponent_3（第4层）
  │     ├── EffectComponentA_3
  │     └── EffectComponentB_3
  └── LiftComponent_4（第5层）
        ├── EffectComponentA_4
        └── EffectComponentB_4
```

**共计**：5 个升降层 × 2 个效果组件 = **10 个独立可控的发光单元**

---

## 三、属性详解

### 3.1 默认参数

| 参数 | 位置 | 说明 | 范围 | 默认值 |
|------|------|------|------|--------|
| **MaxLightIntensity** | B.DefaultParameter | 最大灯光强度 | 0 - 无上限 | 1.0 |
| **LiftRange** | B.DefaultParameter（继承） | 升降范围（总行程） | 0 - 无上限 | 500.0 cm |

**LiftRange** 来自父类 SuperLightBase，定义了从完全收拢到完全展开的总高度（单位：厘米）。

#### 调参建议

| 场景 | MaxLightIntensity | LiftRange | 说明 |
|------|-------------------|-----------|------|
| 小型装置 (3m 以内) | 0.5 - 2.0 | 100 - 200 cm | 近距离观赏，亮度适中 |
| 中型舞台 (3-8m) | 2.0 - 5.0 | 300 - 600 cm | 标准演出场景 |
| 大型演唱会 (8m+) | 5.0 - 15.0 | 600 - 1200 cm | 远距离需要更高亮度和更大行程 |
| 互动装置艺术 | 1.0 - 3.0 | 50 - 150 cm | 小幅运动，注重精度 |

> **提示**：`LiftRange` 决定了完全展开后最底层到最顶层的距离。实际安装时请确保物理空间足够容纳设定的行程。`ComponentSpacing`（5cm）为收拢状态下的层间距，无法修改。

### 3.2 内部常量

| 常量 | 值 | 说明 |
|------|-----|------|
| **ComponentCount** | 5 | 升降层数量 |
| **ComponentSpacing** | 5.0 cm | 收拢状态下各层间距 |
| **DivisionCount** | 6 | 展开等分数（5层占6等分的1/6~5/6位置） |
| **CableCornerOffset** | 22.0 cm | 钢丝绳距中心的偏移距离 |
| **CableOriginHeight** | 15.0 cm | 钢丝绳起点高度 |
| **CableRadius** | 0.5 cm | 钢丝绳半径 |

---

## 四、升降控制

### 4.1 升降映射

通过一个 DMX 属性（InPosZ，来自父类）控制所有层的展开程度：

| DMX 值 | 状态 | 说明 |
|--------|------|------|
| 0.0 | 完全收拢 | 所有层紧密排列，间距 5cm |
| 0.5 | 半展开 | 各层均匀分布在 LiftRange 的一半范围内 |
| 1.0 | 完全展开 | 各层均匀分布在整个 LiftRange 范围内 |

### 4.2 展开分布算法

每层的 Z 位置按**等分分布**计算：

```
第 i 层（i=0~4）:
  收拢位置 = -ComponentSpacing × i        （紧密排列）
  展开位置 = -LiftRange × (i+1) / 6       （6等分，第1~5份）
  实际位置 = Lerp(收拢位置, 展开位置, DMX值)
```

**示例**（LiftRange = 600cm，DMX = 1.0 完全展开）：

| 层 | 收拢 Z | 展开 Z | 说明 |
|----|--------|--------|------|
| 第0层 | 0 cm | -100 cm | 1/6 处 |
| 第1层 | -5 cm | -200 cm | 2/6 处 |
| 第2层 | -10 cm | -300 cm | 3/6 处 |
| 第3层 | -15 cm | -400 cm | 4/6 处 |
| 第4层 | -20 cm | -500 cm | 5/6 处 |

> **注意**：最顶部的 1/6（0~-100cm）空置，用于容纳钢丝绳起点和悬挂机构。

### 4.3 钢丝绳自动更新

升降时，4 根钢丝绳自动调整长度：

- 钢丝绳从 CableOrigin（高度 15cm）延伸到最底层组件
- 长度 = CableOriginHeight - 最底层Z位置
- 位于四角（偏移 ±22cm）
- 通过缩放圆柱体网格实现

---

## 五、效果控制

### 5.1 初始化

| 函数 | 说明 |
|------|------|
| **SetEffectMatrixDefaultValue** | 初始化所有 10 个效果组件的材质，设置 MaxLightIntensity |

**必须在使用前调用**——通常在蓝图的 BeginPlay 或 SuperDMXTick 第一次执行前调用。

### 5.2 效果选择

| 函数 | 说明 |
|------|------|
| **SetEffectMatrix(DmxAttribute)** | 通过矩阵 DMX 通道控制每层的效果选择 |

使用**矩阵 DMX 读取**（GET_SUPER_DMX_MATRIX_VALUE 宏），每个效果组件通过独立的 DMX 通道控制：

- DMX 值（0~255）通过**效果查找表（EffectLUT）**映射为具体的效果参数
- 效果参数包括：Effect（效果类型）、Speed（速度）、Width（宽度）
- 10 个效果组件可以独立控制不同的效果

### 5.3 颜色控制

| 函数 | 说明 |
|------|------|
| **SetEffectColorMatrix(DmxR, DmxG, DmxB)** | 通过矩阵 DMX 通道控制每层的颜色 |

同样使用矩阵 DMX 读取，每个效果组件通过 3 个独立通道（R/G/B）控制颜色。

---

## 六、蓝图控制

### 6.1 典型蓝图实现

```
Event BeginPlay
  │
  └─ SetEffectMatrixDefaultValue()    ← 初始化材质

Event SuperDMXTick(DeltaTime)
  │
  ├─ LiftMatrix(DmxLift)              ← 升降控制
  ├─ SetEffectMatrix(DmxEffect)        ← 效果选择（矩阵）
  └─ SetEffectColorMatrix(R, G, B)     ← 颜色控制（矩阵）
```

### 6.2 DMX 通道规划

| 通道 | 属性 | 类型 | 说明 |
|------|------|------|------|
| 1-2 | Lift | 16位 | 升降控制（InPosZ） |
| 3+ | Effect × N | 矩阵 | 效果选择（每个灯头独立） |
| N+ | R/G/B × N | 矩阵 | 颜色控制（每个灯头独立，各3通道） |

> **通道数取决于矩阵灯头数量**。10 个效果组件 × (1效果 + 3颜色) = 40 通道 + 2升降通道 = 42 通道。

---

## 七、常见问题

### Q: 升降范围不够大/太大？
调整 **LiftRange** 参数。例如设为 1000 表示 10 米行程。

### Q: 效果组件不亮？
1. 确认已调用 **SetEffectMatrixDefaultValue** 初始化材质
2. 检查 **MaxLightIntensity** 不为 0
3. 确认 DMX 信号正确到达

### Q: 钢丝绳不可见？
钢丝绳使用默认引擎圆柱体网格，如果场景中没有该资源则不会显示。这不影响功能。

### Q: 如何只用部分层？
目前组件数量固定为 5 层（10个效果组件）。如果只需较少层数，可以将不需要的层的 DMX 效果通道设为 0（关闭）。

---

> **下一步**：请阅读 [13 - LED 灯带特效](/docs/stage-core/light-strip) 了解灯带效果控制。

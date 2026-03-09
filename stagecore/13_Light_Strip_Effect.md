# 13 - LED 灯带特效 (Light Strip Effect)

> **所属模块**: SuperStage 运行时 (ASuperLightStripEffect)  
> **适用对象**: 灯光设计师、LED 编程师  
> **前置阅读**: [03 - DMX 灯具基础](/docs/stage-core/dmx-actor-base)  
> **最后更新**: 2026-03-06

---

## 一、概述

**SuperLightStripEffect** 是一个 DMX 控制的 LED 灯带特效 Actor。它通过 GPU 材质（MI_Effect_Inst）实现多种可编程的灯带效果（流水、追逐、呼吸等），并能将同一材质效果**批量应用到场景中的多个静态网格体**上。

### 类继承链

```
AActor (UE5 引擎基类)
  │
  └── ASuperBaseActor ·············· SuperStage 根基类 (L1)
        │                           ├ SceneBase / ForwardArrow / UpArrow
        │                           └ AssetMetaData
        │
        └── ASuperDmxActorBase ····· DMX 灯具基类 (L2)
              │                      ├ DMX 地址 / 灯具库 / 通道读取
              │                      └ SuperDMXTick / 地址标签
              │
              └── ASuperLightStripEffect · LED 灯带特效 (L3) ← 本文档
                    ├ TargetMeshActors[] → 批量材质应用
                    ├ MI_Effect_Inst 动态材质实例
                    ├ 9 通道 DMX 控制（Dimmer/Strobe/Effect/Speed/Width/Direction/RGB）
                    ├ 10 种内置 GPU 效果
                    └ MaxLightIntensity / MaterialIndex
```

> **设计要点**：`ASuperLightStripEffect` 直接继承 `ASuperDmxActorBase`（而非 `ASuperLightBase`），因为灯带**不需要 Pan/Tilt 旋转轴**。它的核心功能是通过 DMX 驱动 GPU 材质参数来产生视觉效果，而不是控制物理灯头运动。

### 核心特点

- **10 种内置效果** — 流水、追逐、呼吸、渐变等 GPU 材质特效
- **9 通道 DMX 控制** — Dimmer、频闪、效果选择、速度、宽度、方向、RGB 颜色
- **多目标应用** — 一个灯带 Actor 可以同时驱动场景中多个静态网格体的材质
- **材质共享** — 所有目标网格共享同一动态材质实例（统一控制、节省内存）
- **8 种频闪模式** — 关闭、常亮、线性、脉冲、上升、下降、正弦、随机

### 应用场景

- LED 灯条/灯带效果控制
- 建筑轮廓灯/洗墙灯
- 舞台背景 LED 装饰
- Truss 灯带 / 地面灯带
- 任何需要统一材质效果控制的线性灯具

---

## 二、基本原理

SuperLightStripEffect 的工作原理：

```
DMX 信号 → 读取 9 个通道值 → 更新动态材质参数 → GPU 渲染效果
                                    │
                                    ├─→ 自身 YStaticMeshEffect 网格
                                    └─→ 场景中 TargetMeshActors[] 的网格
```

**关键**：所有目标网格共享**同一个动态材质实例**。这意味着：
- 修改材质参数时，所有目标同时更新（统一效果）
- 只需一组 DMX 通道即可控制所有目标
- 如果需要独立控制不同灯带，使用多个 SuperLightStripEffect Actor

---

## 三、属性详解

### 3.1 目标设置 (A.ModelTargets)

| 参数 | 说明 | 默认值 |
|------|------|--------|
| **TargetMeshActors** | 场景中要应用效果的 StaticMeshActor 数组 | 空 |
| **StaticMeshEffect** | 自身网格组件使用的静态网格资源 | 无 |

#### TargetMeshActors（目标网格 Actor 数组）

这是灯带效果的**核心配置**——你需要在这里选择场景中要被这个灯带控制的静态网格体：

1. 在细节面板中展开 **A.ModelTargets** → **TargetMeshActors**
2. 点击 **+** 按钮添加元素
3. 使用吸管工具或下拉菜单选择场景中的 StaticMeshActor
4. 可以添加**多个**目标，它们会共享同一效果

> **提示**：目标 Actor 必须是 **StaticMeshActor** 类型。普通的 Actor 或蓝图 Actor 不支持。

#### StaticMeshEffect（自身网格）

如果不使用外部目标 Actor，可以给灯带 Actor 本身设置一个静态网格，效果材质会直接应用到上面。

### 3.2 默认参数 (B.DefaultParameter)

| 参数 | 说明 | 范围 | 默认值 |
|------|------|------|--------|
| **MaxLightIntensity** | 最大灯光强度（材质亮度上限） | 0 - 1000 | 1.0 |
| **MaterialIndex** | 材质槽索引（应用到目标网格的第几个材质槽） | 0 - 255 | 0 |

#### MaxLightIntensity

控制材质的最大亮度值。这个值在初始化时设置到材质的 `MaxBrightness` 参数：
- 值越大 → 灯带最大亮度越高
- 通常设为 1.0 ~ 10.0（根据场景需要）
- 实际亮度 = MaxLightIntensity × Dimmer

#### MaterialIndex

指定效果材质应用到目标网格的**第几个材质槽**：
- **0**（默认）→ 应用到第一个材质槽
- 如果目标网格有多个材质槽，可以指定特定槽位
- 系统会自动检查索引是否在有效范围内（防止越界）

#### 调参建议

| 场景 | MaxLightIntensity | 说明 |
|------|-------------------|------|
| 近距离装饰灯带 (< 3m) | 0.5 - 2.0 | 避免过曝刺眼 |
| 舞台灯带 (3-10m) | 2.0 - 5.0 | 标准亮度 |
| 大型场馆 (10m+) | 5.0 - 15.0 | 远距离需要更高亮度 |
| 建筑外墙洗墙灯 | 3.0 - 10.0 | 需穿透环境光 |
| 氛围背景 | 0.3 - 1.0 | 低调柔和效果 |

> **提示**：`MaterialIndex` 通常保持 **0** 即可。仅当目标网格有多个材质槽且你希望只替换特定槽位的材质时才需要修改。

### 3.3 控制参数 (C.ControlParameter)

| 参数 | 说明 | 范围 | 默认值 |
|------|------|------|--------|
| **Dimmer** | 总亮度（调光器） | 0.0 - 1.0 | 0.0 |
| **Strobe** | 频闪频率 | 0.0 - 1.0 | 0.0 |
| **Effect** | 效果选择 | 0.0 - 1.0 | 0.0 |
| **Speed** | 效果运动速度 | 0.0 - 1.0 | 0.5 |
| **Width** | 效果宽度/大小 | 0.0 - 1.0 | 0.0 |
| **Direction** | 效果运动方向 | 0.0 - 1.0 | 0.0 |
| **Color** | RGB 颜色 | 每通道 0.0 - 1.0 | 白色 (1,1,1) |

---

## 四、控制参数详解

### 4.1 Dimmer（调光器）

总亮度控制，乘以 MaxLightIntensity 得到最终亮度：

| DMX 值 | 材质参数 | 效果 |
|--------|---------|------|
| 0.0 | Brightness = 0 | 完全熄灭 |
| 0.5 | Brightness = 0.5 | 半亮 |
| 1.0 | Brightness = 1.0 | 最大亮度 |

### 4.2 Strobe（频闪）

DMX 值映射为频闪频率（0~255）：

```
材质 Strobe 参数 = DMX值 × 255
```

频闪模式由材质内部的 StrobeMode 参数控制：

| StrobeMode | 名称 | 说明 |
|-----------|------|------|
| 0 | Closed | 关闭（常暗） |
| 1 | Open | 常亮（无频闪） |
| 2 | Linear | 线性闪烁 |
| 3 | Pulse | 脉冲闪烁 |
| 4 | RampUp | 渐亮闪烁 |
| 5 | RampDown | 渐暗闪烁 |
| 6 | Sine | 正弦波闪烁 |
| 7 | Random | 随机闪烁 |

### 4.3 Effect（效果选择）

DMX 值映射到 10 种效果（0~10）：

```
材质 Effect 参数 = Lerp(0, 10, DMX值)
```

| DMX 值范围 | 效果索引 | 典型效果 |
|-----------|---------|---------|
| 0.0 ~ 0.1 | 0 ~ 1 | 静态/纯色 |
| 0.1 ~ 0.2 | 1 ~ 2 | 流水 |
| 0.2 ~ 0.3 | 2 ~ 3 | 追逐 |
| 0.3 ~ 0.4 | 3 ~ 4 | 渐变 |
| ... | ... | ... |
| 0.9 ~ 1.0 | 9 ~ 10 | 复杂组合效果 |

> **注意**：具体效果外观取决于 MI_Effect_Inst 材质的实现。不同效果的视觉表现由材质内部的着色器逻辑决定。

### 4.4 Speed（效果速度）

DMX 值映射到速度范围（-10 ~ +10）：

```
材质 Speed 参数 = Lerp(-10, 10, DMX值)
```

| DMX 值 | 速度 | 效果 |
|--------|------|------|
| 0.0 | -10 | 最快**反向**运动 |
| 0.25 | -5 | 中速反向 |
| 0.5 | 0 | **停止**运动 |
| 0.75 | +5 | 中速正向 |
| 1.0 | +10 | 最快正向运动 |

> **提示**：Speed 支持负值，可以反转效果的运动方向。DMX 值 = 0.5 时效果静止。

### 4.5 Width（效果宽度）

DMX 值映射到宽度范围（0 ~ 10）：

```
材质 Width 参数 = Lerp(0, 10, DMX值)
```

- 值越小 → 效果图案越窄/密集
- 值越大 → 效果图案越宽/稀疏

### 4.6 Direction（方向）

直接映射到材质的 EffectDirection 参数（0 ~ 1）：

- 控制效果的运动方向或扫描方向
- 具体行为取决于所选效果

### 4.7 Color（RGB 颜色）

通过 3 个 DMX 通道分别控制红、绿、蓝分量：

| 通道 | DMX 值 0.0 | DMX 值 1.0 |
|------|-----------|-----------|
| Red | 无红色 | 最大红色 |
| Green | 无绿色 | 最大绿色 |
| Blue | 无蓝色 | 最大蓝色 |

颜色值直接设置到材质的 `LightColor` 向量参数。

---

## 五、蓝图控制

### 5.1 控制函数

#### SetLightStripDefault（初始化）

初始化灯带材质并应用到所有目标网格。**必须在开始控制前调用一次**。

执行内容：
1. 为自身网格设置 StaticMeshEffect 资源
2. 创建动态材质实例（如果尚未创建）
3. 设置 MaxBrightness 参数
4. 将材质应用到自身网格的 MaterialIndex 槽
5. 遍历 TargetMeshActors，将材质应用到每个目标的 MaterialIndex 槽

#### SetLightStripEffect（9 通道控制）

主控制函数，在 SuperDMXTick 中每帧调用：

| 输入参数 | 对应控制 |
|---------|---------|
| **DmxDimmer** | 亮度 |
| **DmxStrobe** | 频闪 |
| **DmxEffect** | 效果选择 |
| **DmxSpeed** | 速度 |
| **DmxWidth** | 宽度 |
| **DmxDirection** | 方向 |
| **DmxRed** | 红色 |
| **DmxGreen** | 绿色 |
| **DmxBlue** | 蓝色 |

### 5.2 典型蓝图实现

```
Event BeginPlay
  │
  └─ SetLightStripDefault()    ← 初始化材质

Event SuperDMXTick(DeltaTime)
  │
  └─ SetLightStripEffect(Dimmer, Strobe, Effect, Speed, Width, Direction, R, G, B)
```

---

## 六、使用步骤

### 第 1 步：准备场景中的灯带网格

在场景中放置你想要作为灯带显示的 StaticMeshActor（例如长条形网格）。

### 第 2 步：放置灯带 Actor

将 SuperLightStripEffect 蓝图拖入场景。

### 第 3 步：选择目标

在灯带 Actor 的细节面板中：
1. 展开 **A.ModelTargets** → **TargetMeshActors**
2. 点击 **+** 添加元素
3. 选择场景中的 StaticMeshActor

### 第 4 步：设置参数

- **MaterialIndex** — 设为目标网格上要替换的材质槽索引（通常为 0）
- **MaxLightIntensity** — 设置最大亮度

### 第 5 步：配置 DMX

- 分配 Universe 和 Start Address
- 创建灯具库，定义 9 个属性（Dimmer/Strobe/Effect/Speed/Width/Direction/R/G/B）

### 第 6 步：从控台控制

推动对应通道推杆，灯带效果实时响应。

---

## 七、DMX 通道规划

| 通道 | 属性 | 说明 |
|------|------|------|
| 1 | Dimmer | 亮度 |
| 2 | Strobe | 频闪 |
| 3 | Effect | 效果选择 |
| 4 | Speed | 速度 |
| 5 | Width | 宽度 |
| 6 | Direction | 方向 |
| 7 | Red | 红色 |
| 8 | Green | 绿色 |
| 9 | Blue | 蓝色 |

> **共 9 个通道**。如果使用 16 位精度，按需为关键属性分配 Fine 通道。

---

## 八、多灯带独立控制

如果需要**独立控制**不同区域的灯带效果，需要创建多个 SuperLightStripEffect Actor：

```
灯带Actor_1 (Universe 1, Addr 1)
  └─ TargetMeshActors: [舞台左侧灯带A, 舞台左侧灯带B]

灯带Actor_2 (Universe 1, Addr 10)
  └─ TargetMeshActors: [舞台右侧灯带A, 舞台右侧灯带B]

灯带Actor_3 (Universe 1, Addr 19)
  └─ TargetMeshActors: [地面灯带]
```

每个 Actor 使用不同的 DMX 地址，可以独立控制效果、颜色和速度。

---

## 九、常见问题

### Q: 目标网格没有显示效果？
1. 确认已调用 **SetLightStripDefault** 初始化材质
2. 检查 **MaterialIndex** 是否正确（从 0 开始）
3. 确认目标 Actor 是 **StaticMeshActor** 类型
4. 确认 **Dimmer** DMX 值 > 0

### Q: 效果在编辑器中就能看到吗？
是的。SetLightStripDefault 在 OnConstruction（放置/修改时）和 BeginPlay（运行时）都会调用。编辑器中配置好 TargetMeshActors 后，材质就会应用。

### Q: 如何更换效果材质？
默认使用 MI_Effect_Inst 材质。如果需要自定义材质，创建新的材质实例，确保包含相同的参数名称（MaxBrightness、Brightness、Strobe、LightColor、Effect、Speed、Width、EffectDirection）。

### Q: 添加新的目标 Actor 后效果没有应用？
修改 TargetMeshActors 后，需要**重新触发初始化**。在编辑器中移动灯带 Actor 可以触发 OnConstruction，或运行时重新调用 SetLightStripDefault。

### Q: 效果速度不对？
Speed 的 DMX 值 = 0.5 时速度为 0（停止），不是半速。< 0.5 为反向，> 0.5 为正向。

---

> **下一步**：请阅读 [14 - 舞台特效 VFX](/docs/stage-core/stage-vfx) 了解 Niagara 粒子特效控制。

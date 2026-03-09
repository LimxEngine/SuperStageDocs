# 15 - 全功能电脑灯 (SuperStageLight)

> **所属模块**: SuperStage 运行时 (ASuperStageLight)  
> **适用对象**: 灯光设计师、蓝图开发者  
> **前置阅读**: [04 - 电脑灯基类](04_Computer_Light.md)

---

## 一、概述

**SuperStageLight** 是 SuperStage 系统中**最核心的灯具 Actor**，继承自 `ASuperLightBase`，是所有全功能电脑灯（Spot、Beam、Wash、Profile、切割灯等）的统一实现类。它在基类 Pan/Tilt 运动控制的基础上，提供了专业舞台灯具所需的全部功能：

- **光束控制** — 亮度(Dimmer)、频闪(Strobe)、缩放(Zoom)、对焦(Focus)、雾化(Frost)、光圈(Iris)
- **颜色系统** — RGB/RGBW/HSV 直控、颜色盘(ColorWheel)、CMY 减色混色、色温(CTO)、冷暖混色(CoolWarm)、多通道自由混色(ColorMix)
- **图案系统** — 最多 3 个 Gobo 轮 + 图案旋转(静态/无极/抖动/流水)
- **棱镜系统** — 最多 2 个棱镜 + 棱镜旋转
- **切割系统** — 4 组切割叶片(A/B) + 切割旋转
- **效果系统** — 内置 256 级效果 LUT 查表 + 效果亮度/频闪/颜色
- **矩阵系统** — 像素级独立控制(亮度/颜色/频闪/效果)
- **辅光系统** — Spot 辅光(独立亮度/频闪/颜色/Zoom/Frost/色温控制)
- **升降控制** — Z 轴位移(继承自 SuperLightBase)

---

## 二、类继承关系

```
AActor
  └── ASuperBaseActor          ← 资产元数据 + 方向预览
        └── ASuperDmxActorBase  ← DMX 通道读取 + 灯具库 + 地址标签
              └── ASuperLightBase    ← Pan/Tilt + 无极旋转 + 速度 + 升降
                    └── ASuperStageLight  ← 本文档（全功能电脑灯）
```

---

## 三、组件结构

```
SceneBase (根组件)
  ├── ForwardArrow     — 方向箭头（继承自 SuperBaseActor，编辑器可视）
  ├── UpArrow          — 向上箭头（继承自 SuperBaseActor，编辑器可视）
  ├── AddressLabel     — DMX 地址标签（继承自 SuperDmxActorBase）
  ├── SM_Hook          — 吊钩/安装件模型（UStaticMeshComponent）
  ├── SM_Base          — 灯具底座模型（UStaticMeshComponent）
  ├── SceneRocker      — 摇臂旋转轴（继承自 SuperLightBase，Pan 旋转）
  │     └── SM_Rocker  — 摇臂模型（UStaticMeshComponent）
  └── SceneHard        — 灯头旋转轴（继承自 SuperLightBase，Tilt 旋转）
        ├── SM_Hard           — 灯头模型（UStaticMeshComponent）
        ├── SuperLighting     — 主光源组件（USuperLightingComponent / USuperBeamComponent / USuperCuttingComponent）
        ├── SuperLightingArray[] — 矩阵主光源组件数组
        ├── SpotLighting      — Spot 辅光组件（USuperSpotComponent）
        ├── SuperSpotArray[]  — 矩阵 Spot 辅光组件数组
        ├── SuperEffect       — 效果组件（USuperEffectComponent）
        ├── SuperEffectArray[] — 矩阵效果组件数组
        └── SuperMatrix       — 矩阵像素组件（USuperMatrixComponent）
```

### 3.1 模型可见性控制

| 参数 | 说明 | 默认值 |
|------|------|--------|
| **bHookVisibility** | 控制吊钩模型 SM_Hook 的可见性 | true |
| **bModelVisibility** | 控制底座/摇臂/灯头模型的可见性 | true |

> **提示**：在某些场景中（如纯灯光效果预览），可以关闭模型可见性以提高渲染性能。

---

## 四、默认参数 (B.DefaultParameter)

默认参数定义了灯具的物理特性，在蓝图中设置一次后不随 DMX 变化。

### 4.1 FLightDefaultValue — 灯光默认值

| 参数 | 说明 | 默认值 | 单位 |
|------|------|--------|------|
| **MaxLightIntensity** | 光源最大亮度百分比 | 100.0 | % |
| **LensIntensity** | 镜头强度乘数 | 1.0 | — |
| **MaxLightDistance** | 光照最大距离 | 2345.0 | cm |
| **LightSpotDefaultValue** | 聚光灯参数（见下表） | — | — |
| **BeamDefaultValue** | 光束参数（见下表） | — | — |

### 4.2 FLightSpotDefaultValue — 聚光灯参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| **LightSpotIntensity** | 聚光灯亮度乘数 | 1.0 |
| **VolumetricScattering** | 体积散射强度 | 1.0 |
| **bLightShadow** | 是否启用阴影 | false |
| **bAffectTransmission** | 是否影响透射 | false |
| **SpecularScale** | 高光强度缩放 | 1.0 |
| **SourceRadius** | 光源半径（影响软阴影） | 0.0 |
| **bLightingChannel0** | 光照通道 0 | true |
| **bLightingChannel1** | 光照通道 1 | false |
| **bLightingChannel2** | 光照通道 2 | false |

### 4.3 FBeamDefaultValue — 光束参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| **BeamIntensity** | 光束亮度乘数 | 1.0 |
| **AtmosphericDensity** | 大气散射密度 | 1.0 |
| **BeamFogIntensity** | 光束雾气强度 | 1.0 |
| **AtmosBeamFogSpeed** | 大气/雾气流动速度 | 0.0 |
| **LensRadius** | 镜头光晕半径 | 1.0 |
| **BeamQuality** | 光束渲染质量 | 1.0 |
| **bBeamBlock** | 是否启用光束遮挡检测 | false |

### 4.4 默认参数调参建议

以下是针对不同灯具类型和场景的推荐参数起始值：

#### 灯光亮度与距离

| 场景 | MaxLightIntensity | MaxLightDistance | LensIntensity | 说明 |
|------|-------------------|-----------------|---------------|------|
| 小型演播室 (10m) | 50 - 100 | 1500 | 0.5 - 1.0 | 近距离照明，避免过曝 |
| 中型剧场 (20m) | 100 - 200 | 2500 | 1.0 | 标准舞台灯光 |
| 大型体育馆 (50m+) | 200 - 500 | 5000 - 8000 | 1.5 - 3.0 | 远距离投射需要更高强度 |
| 户外音乐节 | 300 - 800 | 8000 - 10000 | 2.0 - 5.0 | 开阔空间、环境光竞争 |

#### 聚光灯参数

| 场景 | VolumetricScattering | bLightShadow | SourceRadius | 说明 |
|------|---------------------|-------------|-------------|------|
| 写实渲染 | 0.5 - 1.0 | true | 5.0 - 20.0 | 柔和阴影，真实体积光 |
| 实时预演（性能优先） | 0.0 | false | 0.0 | 关闭阴影和体积光，最佳帧率 |
| 氛围效果 | 1.5 - 3.0 | false | 0.0 | 强烈体积散射，无阴影开销 |

#### 光束参数

| 场景 | BeamIntensity | AtmosphericDensity | BeamFogIntensity | AtmosFogSpeed | 说明 |
|------|--------------|-------------------|-----------------|--------------|------|
| 无烟雾环境 | 0.5 - 1.0 | 0.3 - 0.5 | 0.0 | 0.0 | 淡光束，无雾效 |
| 轻度烟雾 | 1.0 - 2.0 | 0.8 - 1.0 | 0.5 - 1.0 | 5.0 | 标准舞台烟雾效果 |
| 浓雾效果 | 2.0 - 5.0 | 1.5 - 3.0 | 2.0 - 5.0 | 10.0 - 20.0 | 浓烟中的强光束可见性 |
| 激光般锐利光束 | 3.0 - 8.0 | 0.1 - 0.3 | 0.0 | 0.0 | 高亮度低散射 |

> **性能提示**：`bBeamBlock`（光束遮挡检测）会消耗额外 CPU 资源进行射线检测。仅在需要光束被物体遮挡截断的效果时启用。`BeamQuality` 值越高渲染越精细但 GPU 开销越大，建议在性能允许范围内调整。

### 4.5 继承的运动参数（来自 SuperLightBase）

| 参数 | 说明 | 默认值 | 范围 |
|------|------|--------|------|
| **PanRange** | Pan 旋转范围 | (-270, 270) | 度 |
| **TiltRange** | Tilt 旋转范围 | (-135, 135) | 度 |
| **PanAngle** | 手动 Pan 角度 | 0 | -180 ~ 180° |
| **TiltAngle** | 手动 Tilt 角度 | 0 | -180 ~ 180° |
| **InfiniteRotationalSpeed** | 无极旋转速度乘数 | 1.0 | 0.01 ~ 10 |
| **LiftSpeed** | 升降速度（FInterpTo 参数） | 1.0 | 0 ~ 10 |
| **LiftRange** | 升降范围 | 500.0 | cm |

---

## 五、控制参数 (C.ControlParameter)

控制参数在运行时由 DMX 信号驱动，归一化值范围 **0.0 ~ 1.0**（除特殊说明外）。

### 5.1 光束控制参数

| 参数 | DisplayName | 说明 | 默认值 | 值域 | 开关 |
|------|-------------|------|--------|------|------|
| **Dimmer** | Dimmer | 灯光亮度 | 1.0 | 0~1 | bDimmer |
| **Strobe** | Strobe | 频闪速度 | 1.0 | 0~1 | bStrobe |
| **Zoom** | Zoom | 光束/光斑缩放 | 0.0 | 0~1 | bZoom |
| **Focus** | Focus | 光束对焦 | 0.0 | 0~1 | bFocus |
| **Frost** | Frost | 雾化效果强度 | 0.0 | 0~1 | bFrost |
| **Iris** | Iris | 光圈大小（1=全开, 0=最小） | 1.0 | 0~1 | bIris |

#### Dimmer 工作原理

- DMX 值 0 → Dimmer=0.0 → 灯具完全关闭（组件隐藏）
- DMX 值 255 → Dimmer=1.0 → 最大亮度
- 内部调用 `SetLightingIntensity(Dimmer)` 设置亮度百分比
- **自动可见性**：Dimmer > 0 时显示组件，= 0 时隐藏组件

#### Strobe 工作原理

频闪通过子属性（SubAttribute）系统支持多种模式：

| 频闪模式 | 值 | 说明 |
|----------|-----|------|
| **Open** | 1 | 常亮（无频闪） |
| **Linear** | 2 | 线性频闪 |
| **Pulse** | 3 | 脉冲频闪 |
| **PulseOpen** | 4 | 脉冲开启 |
| **PulseClose** | 5 | 脉冲关闭 |
| **RampUp** | 6 | 渐亮 |
| **Random** | 7 | 随机频闪（每台灯具独立随机种子） |

> **随机频闪种子**：每台灯具在 `OnConstruction` 时生成固定随机种子 `StrobeRandomSeed`，矩阵模式下每个灯头有独立种子 `StrobeRandomSeedArray`，确保每次加载后行为一致。

#### Iris 工作原理

```
DMX 值 0   → Iris 归一化 0.0 → 实际 IrisValue = 1.0 (全开, 100%)
DMX 值 255 → Iris 归一化 1.0 → 实际 IrisValue = 0.3 (最细, 30%)
```

映射公式：`IrisValue = Lerp(1.0, 0.3, Iris)`

### 5.2 颜色控制参数

| 参数 | DisplayName | 说明 | 默认值 | 开关 |
|------|-------------|------|--------|------|
| **Color** | Color | RGB 颜色值 | 白色(1,1,1) | bColor |
| **ColorWheel** | ColorWheel | 颜色盘位置 | 0.0 | bColorWheel |
| **ColorTemperature** | ColorTemperature | 色温控制 | 0.0 | bColorTemperature |
| **Cool** | Cool | 冷光通道 | 0.0 | bCool |
| **Warm** | Warm | 暖光通道 | 0.0 | bWarm |

### 5.3 图案/棱镜控制参数

| 参数 | DisplayName | 说明 | 默认值 | 值域 | 开关 |
|------|-------------|------|--------|------|------|
| **Gobo1** | Gobo1 | 图案轮 1 位置 | 0.0 | 0~255 | bGobo1 |
| **Gobo2** | Gobo2 | 图案轮 2 位置 | 0.0 | 0~255 | bGobo2 |
| **Gobo_Rot** | GoboRot | 图案旋转 | 0.0 | 0~255 | bGoboRot |
| **Prism1** | Prism1 | 棱镜 1 | 0.0 | 0~255 | bPrism1 |
| **Prism2** | Prism2 | 棱镜 2 | 0.0 | 0~255 | bPrism2 |
| **Prism_Rot** | PrismRot | 棱镜旋转 | 0.0 | 0~255 | bPrismRot |

### 5.4 切割控制参数

| 参数 | DisplayName | 说明 | 默认值 | 开关 |
|------|-------------|------|--------|------|
| **A1** | A1 | 切割叶片 1 位置 A | 0.0 | bCuttingChannel |
| **B1** | B1 | 切割叶片 1 位置 B | 0.0 | bCuttingChannel |
| **A2** | A2 | 切割叶片 2 位置 A | 0.0 | bCuttingChannel |
| **B2** | B2 | 切割叶片 2 位置 B | 0.0 | bCuttingChannel |
| **A3** | A3 | 切割叶片 3 位置 A | 0.0 | bCuttingChannel |
| **B3** | B3 | 切割叶片 3 位置 B | 0.0 | bCuttingChannel |
| **A4** | A4 | 切割叶片 4 位置 A | 0.0 | bCuttingChannel |
| **B4** | B4 | 切割叶片 4 位置 B | 0.0 | bCuttingChannel |
| **ShaperRot** | ShaperRot | 切割旋转 | 0.0 | bCuttingChannel |

### 5.5 效果控制参数

| 参数 | DisplayName | 说明 | 默认值 | 开关 |
|------|-------------|------|--------|------|
| **EffectDimmer** | EffectDimmer | 效果亮度 | 0.0 | bEffect |
| **EffectStrobe** | EffectStrobe | 效果频闪 | 0.0 | bEffect |
| **EffectValue** | Effect | 效果 LUT 索引 | 0.0 | bEffect |
| **EffectColor** | EffectColor | 效果 RGB 颜色 | 白色 | bEffect |

### 5.6 继承的运动控制参数（来自 SuperLightBase）

| 参数 | 说明 | 默认值 | 开关 |
|------|------|--------|------|
| **Pan** | 水平旋转位置 | 0.5 | bPan |
| **Tilt** | 垂直旋转位置 | 0.5 | bTilt |
| **PTSpeed** | Pan/Tilt 运动速度 | 5.0 | bPTSpeed |
| **PanRot** | Pan 无极旋转 | 0.5 | bPanRot |
| **TiltRot** | Tilt 无极旋转 | 0.5 | bTiltRot |
| **InPosZ** | Z 轴升降位置 | 0.0 | bInPosZ |

---

## 六、通道开关 (D.ChannelSwitch)

通道开关决定灯具响应哪些 DMX 属性。在灯具蓝图的默认值中设置，运行时不可修改。

### 6.1 完整开关列表

| 开关 | 功能 | 默认值 |
|------|------|--------|
| **bDimmer** | 亮度控制 | false |
| **bStrobe** | 频闪控制 | false |
| **bZoom** | 缩放控制 | false |
| **bFocus** | 对焦控制 | false |
| **bFrost** | 雾化控制 | false |
| **bIris** | 光圈控制 | false |
| **bColor** | RGB 颜色控制 | false |
| **bColorWheel** | 颜色盘控制 | false |
| **bColorTemperature** | 色温控制 | false |
| **bCool** | 冷光通道 | false |
| **bWarm** | 暖光通道 | false |
| **bGobo1** | 图案轮 1 | false |
| **bGobo2** | 图案轮 2 | false |
| **bGoboRot** | 图案旋转 | false |
| **bPrism1** | 棱镜 1 | false |
| **bPrism2** | 棱镜 2 | false |
| **bPrismRot** | 棱镜旋转 | false |
| **bCutting** | 切割叶片 | false |
| **bCuttingRot** | 切割旋转 | false |
| **bEffect** | 效果控制 | false |
| **bEffectDimmer** | 效果亮度 | false |
| **bEffectStrobe** | 效果频闪 | false |
| **bEffectColor** | 效果颜色 | false |
| **bPan** | Pan 水平旋转 | false |
| **bTilt** | Tilt 垂直旋转 | false |
| **bPTSpeed** | PT 速度 | false |
| **bPanRot** | Pan 无极旋转 | false |
| **bTiltRot** | Tilt 无极旋转 | false |
| **bInPosZ** | Z 轴升降 | false |
| **bLampAngle** | 灯具角度 | false |

> **设计原则**：只启用灯具实际拥有的通道功能，关闭不需要的开关可以避免不必要的 DMX 读取和计算开销。

---

## 七、颜色系统详解

SuperStageLight 提供了业界最完整的颜色控制体系，支持从最简单的 RGB 直控到复杂的多通道自由混色 + CTO 色温调节。

### 7.1 RGB / RGBW 直控

最基础的颜色控制方式，直接映射 DMX 通道到颜色分量。

**函数列表**：

| 函数 | 说明 | 矩阵版本 |
|------|------|----------|
| **SetLightingColorRGB** | RGB 三通道颜色 | SetLightingColorRGBMatrix |
| **SetLightingColorRGBW** | RGBW 四通道颜色 | SetLightingColorRGBWMatrix |

**RGBW 混色公式**：
```
R_out = Min(R + W, 1.0)
G_out = Min(G + W, 1.0)
B_out = Min(B + W, 1.0)
```
白色通道(W)叠加到 RGB 上，Clamp 到 1.0 防止过曝。

### 7.2 HSV 颜色空间

支持色相(H)、饱和度(S)、明度(V) 颜色空间控制。

| 函数 | 说明 | 矩阵版本 |
|------|------|----------|
| **SetLightingColorHSV** | H+S+V 三通道 | SetLightingColorHSVMatrix |
| **SetLightingColorHS** | H+S 两通道（V 固定 1.0，由 Dimmer 控制亮度） | SetLightingColorHSMatrix |
| **SetLightingColorHSAndCTO** | H+S + CTO 色温叠乘 | — |

> **提示**：使用 HS 模式时，明度(V)固定为 1.0，实际亮度由 Dimmer 通道统一控制，避免双重亮度调节。

### 7.3 颜色盘 (ColorWheel)

支持最多 **3 个颜色盘**同时使用，通过颜色图集贴图（ColorAtlas）实现：

| 函数 | 颜色盘数量 | 附加功能 |
|------|-----------|----------|
| **SetLightingColorWheel** | 1 个 | — |
| **SetLightingColorWheel2** | 2 个 | 优先级选择 |
| **SetLightingColorWheel3** | 3 个 | 优先级选择 |

**优先级规则**：当多个颜色盘有非零值时，按编号优先（ColorWheel1 > ColorWheel2 > ColorWheel3）；全部为零时使用最高优先级的颜色盘。

**颜色盘模式**：

| 模式 | 说明 |
|------|------|
| **静态选色** | DMX 值映射到图集中的固定颜色位置 |
| **流水正向** | 颜色连续正向滚动，速度由子属性物理范围决定 |
| **流水反向** | 颜色连续反向滚动 |

### 7.4 CMY 减色混色

通过 Cyan/Magenta/Yellow 三个滤色片的透过率模型混色：

```
透过率计算：
  cmyPass.R = 1.0 - Cyan
  cmyPass.G = 1.0 - Magenta
  cmyPass.B = 1.0 - Yellow
```

| 函数 | 说明 |
|------|------|
| **SetLightingColorWheelAndCMY** | 1 颜色盘 + CMY |
| **SetLightingColorWheel2AndCMY** | 2 颜色盘 + CMY |
| **SetLightingColorWheel3AndCMY** | 3 颜色盘 + CMY |
| **SetLightingColorWheel2AndCMYAndCTO** | 2 颜色盘 + CMY + CTO |

CMY 颜色通过 `SetLightingColor(cmyPass)` 叠加到颜色盘上，材质中两者相乘实现物理透过率模型。

### 7.5 色温控制 (CTO)

色温控制提供两种模式：

#### 单通道色温 (ColorTemperature)

```
Color = Lerp(WarmColor, CoolColor, DMX值)
```
- DMX = 0 → 暖色温（默认 3200K）
- DMX = 1 → 冷色温（默认 6500K）

| 函数 | 说明 | 矩阵版本 |
|------|------|----------|
| **SetLightingColorTemperature** | 单通道色温 | — |
| **SetSpotColorTemperatureMatrix** | Spot 辅光矩阵色温 | ✓ |

#### CTO 色温叠乘

CTO（Color Temperature Orange）作为附加滤片叠加到其他颜色上：

```
CTOTint = Lerp(CoolColor, WarmColor, CTO值)
FinalColor = BaseColor * CTOTint
```

| 函数 | 说明 |
|------|------|
| **SetLightingColorRGBWWithCTO** | RGBW + CTO |
| **SetLightingColorRGBWMatrixWithCTO** | RGBW 矩阵 + CTO |

#### 色温转 RGB 算法

系统使用 **Tanner Helland 算法**将开尔文色温转换为 RGB：

- 输入范围：1000K ~ 40000K
- 分段计算 R/G/B 分量
- 6600K 为分界点（低于→偏暖，高于→偏冷）

### 7.6 冷暖双通道混色 (CoolWarm)

两个独立通道分别控制冷光和暖光强度，加法混色：

```
Color.R = CoolColor.R × Cool + WarmColor.R × Warm
Color.G = CoolColor.G × Cool + WarmColor.G × Warm
Color.B = CoolColor.B × Cool + WarmColor.B × Warm
```

| 函数 | 说明 | 矩阵版本 |
|------|------|----------|
| **SetLightingCoolWarmMix** | 主光源冷暖混色 | — |
| **SetSpotCoolWarmMix** | Spot 辅光冷暖混色 | SetSpotCoolWarmMixMatrix |

**组合函数**（RGBW + CoolWarm）：

| 函数 | 说明 |
|------|------|
| **SetSpotColorRGBWWithCoolWarmMix** | Spot: RGBW × CoolWarm |
| **SetSpotColorRGBWWithCoolWarmMixMatrix** | Spot 矩阵: RGBW × CoolWarm |

### 7.7 多通道自由混色 (ColorMix)

最灵活的颜色控制方式，支持任意数量的颜色通道。通过 `FSuperColorChannel` 结构定义：

```cpp
struct FSuperColorChannel
{
    FSuperDMXAttribute DmxAttribute;   // DMX 属性（通道值 0-1）
    FLinearColor ChannelColor;          // 该通道对应的颜色
};
```

**混色原理**（加法混色）：
```
对每个 ColorChannel:
  R += ChannelColor.R × DMX值
  G += ChannelColor.G × DMX值
  B += ChannelColor.B × DMX值
最终 Clamp 到 [0, 1]
```

| 函数 | 说明 | 矩阵版本 |
|------|------|----------|
| **SetLightingColorMix** | 自由混色 | SetLightingColorMixMatrix |
| **SetLightingColorMixWithCTO** | 自由混色 + CTO | SetLightingColorMixMatrixWithCTO |

**脏值检测**：`SetLightingColorMix` 内置颜色变化阈值检测（0.001），颜色无变化时跳过下发，减少组件调用。

**应用场景**：适用于 RGBW、RGBA、RGBWAL（红/绿/蓝/白/琥珀/暖白）等任意颜色通道组合的灯具。

---

## 八、图案系统 (Gobo)

图案系统通过图案图集贴图（GoboAtlas）实现图案投射。

### 8.1 图案选择

每个图案轮通过子属性系统支持三种模式：

| 模式 | EGoboMode | 说明 |
|------|-----------|------|
| **静态 (Static)** | Static | DMX 值映射到图集中的固定图案位置 |
| **抖动 (Shake)** | Shake | 图案在当前位置抖动，速度 1-10 Hz |
| **流水 (Scrolling)** | Scrolling | 图案连续滚动，速度由物理范围决定 |

### 8.2 图案旋转

图案旋转通过子属性的物理范围自动判断模式：

| 条件 | 模式 | 说明 |
|------|------|------|
| 有 PhysicalRange | 无极旋转 | 速度由物理范围插值 |
| 无 PhysicalRange | 静态旋转 | 归一化值 × 360° |

### 8.3 多图案轮优先级

支持最多 **3 个图案轮**同时配置：

| 函数 | Gobo 轮数 | 说明 |
|------|-----------|------|
| **SetBeamGobo1** | 1 | 单图案轮 + 旋转 |
| **SetBeamGobo2** | 2 | 双图案轮 + 各自旋转 |
| **SetBeamGobo3** | 3 | 三图案轮 + 各自旋转 |

**优先级规则**：有非零 DMX 值的图案轮优先；多个非零时 Gobo1 > Gobo2 > Gobo3；全零时使用 Gobo1。

---

## 九、棱镜系统 (Prism)

棱镜系统通过子属性的 `ChannelSet` 定义棱镜参数。

### 9.1 棱镜参数

每个棱镜由以下参数定义（在灯具库的 ChannelSet 中配置）：

| 参数 | 说明 |
|------|------|
| **PrismFacets** | 棱镜面数（3/5/8 等） |
| **PrismRadius** | 棱镜半径（默认 0.3） |
| **PrismScale** | 棱镜缩放（默认 0.15） |

### 9.2 棱镜旋转

棱镜旋转复用图案旋转逻辑（`CalcGoboRotation`），支持静态和无极两种模式。

### 9.3 函数

| 函数 | 说明 |
|------|------|
| **SetBeamPrism** | 设置棱镜（支持 2 个棱镜 + 旋转） |

**优先级**：Prism1 > Prism2。无匹配棱镜时关闭棱镜效果（Facets=0）。

---

## 十、切割系统 (Cutting)

切割系统用于 Profile 灯的四叶片光束裁剪。

### 10.1 切割叶片

每组叶片由 A（起始位置）和 B（结束位置）两个参数控制：

```
叶片 1: A1 → Lerp(0, 0.4)    B1 → Lerp(0, 0.4)
叶片 2: A2 → Lerp(1, 0.6)    B2 → Lerp(1, 0.6)
叶片 3: A3 → Lerp(0, 0.4)    A3 → Lerp(0, 0.4)
叶片 4: A4 → Lerp(1, 0.6)    B4 → Lerp(1, 0.6)
```

- A 参数：从 0（无切割）到 0.4（最大切割 40%）
- B 参数：从 1.0（无切割）到 0.6（最大切割 40%）

### 10.2 切割旋转

切割旋转由两个来源叠加：

```
FinalRotate = ShaperRot + GoboRot
```

- **ShaperRot**：切割系统自身旋转，通过子属性物理范围映射
- **GoboRot**：图案旋转的叠加（静态 + 无极）

| 函数 | 说明 |
|------|------|
| **SetBeamCutting** | 设置 4 组切割叶片位置（8 个 DMX 属性） |
| **SetBeamCuttingRotate** | 设置切割旋转（ShaperRot + GoboRot 叠加） |

---

## 十一、效果系统 (Effect)

效果系统通过内置 LUT（查找表）将 DMX 值映射到效果参数。

### 11.1 效果 LUT

- DMX 值 0~255 → 查表 → `FEffectParams { Effect, Speed, Width }`
- 256 级精度，覆盖所有内置效果组合

### 11.2 效果控制函数

| 函数 | 说明 |
|------|------|
| **SetEffectDefaultValue** | 初始化效果组件（设置材质和最大亮度） |
| **SetEffectMatrixDefaultValue** | 批量初始化矩阵效果组件 |
| **SetEffectIntensity** | 效果亮度控制 |
| **SetEffectStrobe** | 效果频闪控制（支持子属性频闪模式） |
| **SetEffectColor** | 效果 RGB 颜色 |
| **SetEffectColorMatrix** | 矩阵效果 RGB 颜色 |
| **SetEffect** | 效果 LUT 查表（单灯） |
| **SetEffectMatrix** | 效果 LUT 查表（矩阵） |
| **SetEffectWithSpeed** | 效果 + 独立速度通道 |
| **SetEffectMatrixWithSpeed** | 矩阵效果 + 独立速度通道 |

### 11.3 脏值优化

`SetEffect` 函数使用 `LastSingleParams` 缓存上次参数，通过 `NearlySame` 浮点比较（精度 `KINDA_SMALL_NUMBER`），参数未变化时跳过 `SetEffectControl` 调用。

### 11.4 独立速度模式

`SetEffectWithSpeed` 和 `SetEffectMatrixWithSpeed` 使用不含速度的 LUT（`GetEffectLUTNoSpeed`），速度由独立 DMX 通道控制：

```
Speed = Lerp(-4.0, 4.0, DMX速度值)
```
- 负值 → 反向播放
- 0 → 停止
- 正值 → 正向播放

---

## 十二、Spot 辅光系统

Spot 辅光是独立于主光源的第二光源层，使用 `USuperSpotComponent` 组件。适用于需要双层光源的灯具（如带辅助光束的 Wash 灯）。

### 12.1 初始化

| 函数 | 说明 |
|------|------|
| **SetSpotDefaultValue** | 初始化单个 Spot 辅光 |
| **SetSpotMatrixDefaultValue** | 批量初始化矩阵 Spot 辅光 |

### 12.2 控制函数

| 函数 | 说明 | 矩阵版本 |
|------|------|----------|
| **SetSpotIntensity** | Spot 亮度 | SetSpotIntensityMatrix |
| **SetSpotStrobe** | Spot 频闪 | SetSpotStrobeMatrix |
| **SetSpotColorRGB** | Spot RGB 颜色 | SetSpotColorRGBMatrix |
| **SetSpotColorRGBW** | Spot RGBW 颜色 | SetSpotColorRGBWMatrix |
| **SetSpotZoom** | Spot 缩放 | SetSpotZoomMatrix |
| **SetSpotFrost** | Spot 雾化 | SetSpotFrostMatrix |
| **SetSpotCoolWarmMix** | Spot 冷暖混色 | SetSpotCoolWarmMixMatrix |
| **SetSpotColorRGBWWithCoolWarmMix** | Spot RGBW+冷暖 | SetSpotColorRGBWWithCoolWarmMixMatrix |

> Spot 辅光的控制逻辑与主光源完全一致，区别仅在于作用的组件不同。

---

## 十三、矩阵像素组件 (Matrix Component)

矩阵像素组件 `USuperMatrixComponent` 用于控制 LED 面板灯、像素灯条等设备的**独立像素颜色**。

### 13.1 矩阵初始化

```
SetMatrixComponent(R, G, B, StrobeAttr, MatrixComponent)
```
自动根据 `SegCount` 判断模式：
- `SegCount > 1` → 矩阵模式（使用 `GET_SUPER_DMX_MATRIX_RGB` 宏）
- `SegCount == 1` → 单像素模式

### 13.2 白光控制

| 函数 | 说明 |
|------|------|
| **SetMatrixWhiteSingle** | 设置单个像素的白光亮度 |
| **SetMatrixWhiteMultiple** | 设置所有像素的独立白光亮度 |

### 13.3 RGB 颜色控制

| 函数 | 说明 |
|------|------|
| **SetMatrixColorSingle** | 单像素 RGB |
| **SetMatrixColorMultiple** | 多像素独立 RGB |
| **SetMatrixColorRGBWSingle** | 单像素 RGBW |
| **SetMatrixColorRGBWMultiple** | 多像素独立 RGBW |
| **SetMatrixColorRGBWithCTOSingle** | 单像素 RGB + CTO |
| **SetMatrixColorRGBWithCTOMultiple** | 多像素 RGB + CTO |
| **SetMatrixColorRGBWWithCTOSingle** | 单像素 RGBW + CTO |
| **SetMatrixColorRGBWWithCTOMultiple** | 多像素 RGBW + CTO |

### 13.4 冷暖混色控制

| 函数 | 说明 |
|------|------|
| **SetMatrixCoolWarmMixSingle** | 单像素冷暖混色 |
| **SetMatrixCoolWarmMixMultiple** | 多像素独立冷暖混色 |
| **SetMatrixColorRGBWithCoolWarmSingle** | 单像素 RGB + 冷暖 |
| **SetMatrixColorRGBWithCoolWarmMultiple** | 多像素 RGB + 冷暖 |
| **SetMatrixColorRGBWWithCoolWarmSingle** | 单像素 RGBW + 冷暖 |
| **SetMatrixColorRGBWWithCoolWarmMultiple** | 多像素 RGBW + 冷暖 |

### 13.5 自由混色

| 函数 | 说明 |
|------|------|
| **SetMatrixColorMix** | 多通道自由混色（FSuperColorChannel 数组） |

### 13.6 频闪与亮度

| 函数 | 说明 |
|------|------|
| **SetMatrixStrobe** | 矩阵整体频闪（支持子属性频闪模式） |
| **SetMatrixIntensity** | 矩阵整体亮度 |

---

## 十四、FSuperColorChannel 结构

`FSuperColorChannel` 是自由混色系统的核心数据结构：

```cpp
USTRUCT(BlueprintType)
struct FSuperColorChannel
{
    // DMX 属性（通道值 0-1）
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FSuperDMXAttribute DmxAttribute;

    // 该通道对应的颜色（如红色、白色、琥珀色等）
    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FLinearColor ChannelColor = FLinearColor::White;
};
```

**使用示例**（RGBWAL 灯具配置）：

| 通道 | ChannelColor | 说明 |
|------|-------------|------|
| Red | (1, 0, 0) | 红色 LED |
| Green | (0, 1, 0) | 绿色 LED |
| Blue | (0, 0, 1) | 蓝色 LED |
| White | (1, 1, 1) | 白色 LED |
| Amber | (1, 0.75, 0) | 琥珀色 LED |
| Lime | (0.75, 1, 0) | 暖白/石灰色 LED |

---

## 十五、Blueprint 集成

### 15.1 典型 Spot 灯蓝图结构

```
Event BeginPlay
  ├─ SetLightingDefaultValue(SuperLighting)        ← 初始化主光源
  └─ SetEffectDefaultValue(SuperEffect)            ← 初始化效果组件

Event SuperDMXTick(DeltaTime)
  ├─ 运动控制
  │   ├─ YPan(Pan)
  │   ├─ YTilt(Tilt)
  │   └─ YPTSpeed(PTSpeed)
  │
  ├─ 光束控制
  │   ├─ SetLightingIntensity(Dimmer)
  │   ├─ SetLightingStrobe(Strobe)
  │   ├─ SetLightingZoom(Zoom)
  │   ├─ SetLightingFrost(Frost)
  │   └─ SetLightingIris(Iris)
  │
  ├─ 颜色控制
  │   └─ SetLightingColorRGB(R, G, B)
  │
  ├─ 图案控制
  │   └─ SetBeamGobo1(Gobo1, Gobo1Rot, GoboAtlas)
  │
  └─ 效果控制
      ├─ SetEffectIntensity(EffectDimmer)
      └─ SetEffect(Effect)
```

### 15.2 典型 Profile 切割灯蓝图结构

```
Event SuperDMXTick(DeltaTime)
  ├─ ... (运动/光束/颜色 同上)
  │
  ├─ 切割控制
  │   ├─ SetBeamCutting(A1, B1, A2, B2, A3, B3, A4, B4)
  │   └─ SetBeamCuttingRotate(ShaperRot, GoboRot)
  │
  └─ 图案+棱镜控制
      └─ SetBeamGobo2(Gobo1, Gobo1Rot, Gobo2, Gobo2Rot, Atlas1, Atlas2)
```

### 15.3 典型矩阵灯蓝图结构

```
Event BeginPlay
  ├─ SetLightingMatrixDefaultValue(SuperLightingArray)  ← 批量初始化
  └─ SetEffectMatrixDefaultValue(SuperEffectArray)

Event SuperDMXTick(DeltaTime)
  ├─ 运动控制（矩阵版）
  │   ├─ YMatrixPan(Pan, MatrixRockerArray)
  │   └─ YMatrixTilt(Tilt, MatrixHardArray)
  │
  ├─ 光束控制（矩阵版）
  │   ├─ SetLightingIntensityMatrix(Dimmer)
  │   ├─ SetLightingStrobeMatrix(Strobe)
  │   └─ SetLightingZoomMatrix(Zoom)
  │
  ├─ 颜色控制（矩阵版）
  │   └─ SetLightingColorRGBMatrix(R, G, B)
  │
  └─ 效果控制（矩阵版）
      └─ SetEffectMatrix(Effect)
```

---

## 十六、DMX 通道规划示例

### 16.1 基础 Spot 灯（18 通道）

| 通道 | 属性 | 位深 | 说明 |
|------|------|------|------|
| 1-2 | Pan | 16bit | 水平旋转 |
| 3-4 | Tilt | 16bit | 垂直旋转 |
| 5 | PTSpeed | 8bit | 运动速度 |
| 6 | Dimmer | 8bit | 亮度 |
| 7 | Strobe | 8bit | 频闪 |
| 8 | Red | 8bit | 红色 |
| 9 | Green | 8bit | 绿色 |
| 10 | Blue | 8bit | 蓝色 |
| 11 | White | 8bit | 白色 |
| 12 | Zoom | 8bit | 缩放 |
| 13 | Gobo1 | 8bit | 图案轮 |
| 14 | Gobo1Rot | 8bit | 图案旋转 |
| 15 | Prism1 | 8bit | 棱镜 |
| 16 | PrismRot | 8bit | 棱镜旋转 |
| 17 | Focus | 8bit | 对焦 |
| 18 | Frost | 8bit | 雾化 |

### 16.2 Profile 切割灯（28 通道）

| 通道 | 属性 | 位深 | 说明 |
|------|------|------|------|
| 1-2 | Pan | 16bit | 水平旋转 |
| 3-4 | Tilt | 16bit | 垂直旋转 |
| 5 | PTSpeed | 8bit | 运动速度 |
| 6 | Dimmer | 8bit | 亮度 |
| 7 | Strobe | 8bit | 频闪 |
| 8 | Cyan | 8bit | CMY-Cyan |
| 9 | Magenta | 8bit | CMY-Magenta |
| 10 | Yellow | 8bit | CMY-Yellow |
| 11 | CTO | 8bit | 色温 |
| 12 | ColorWheel | 8bit | 颜色盘 |
| 13 | Zoom | 8bit | 缩放 |
| 14 | Focus | 8bit | 对焦 |
| 15 | Iris | 8bit | 光圈 |
| 16 | Gobo1 | 8bit | 图案轮 1 |
| 17 | Gobo1Rot | 8bit | 图案旋转 1 |
| 18 | Gobo2 | 8bit | 图案轮 2 |
| 19 | Gobo2Rot | 8bit | 图案旋转 2 |
| 20 | Prism1 | 8bit | 棱镜 |
| 21 | PrismRot | 8bit | 棱镜旋转 |
| 22 | Frost | 8bit | 雾化 |
| 23 | A1 | 8bit | 切割 1A |
| 24 | B1 | 8bit | 切割 1B |
| 25 | A2 | 8bit | 切割 2A |
| 26 | B2 | 8bit | 切割 2B |
| 27 | ShaperRot | 8bit | 切割旋转 |
| 28 | Effect | 8bit | 效果 |

---

## 十七、常见问题

### Q: 灯具亮度开了但看不到光？
1. 检查 `MaxLightIntensity` 和 `MaxLightDistance` 是否合理
2. 确认 `bDimmer` 开关已启用
3. 检查 `bModelVisibility` 是否为 true
4. 确认光照通道设置正确（LightingChannel0/1/2）

### Q: 颜色盘不显示颜色？
1. 确认传入了有效的 `ColorAtlas` 贴图
2. 确认灯具库中该属性定义了子属性（SubAttributes）和 ChannelSets
3. 检查 `bColorWheel` 开关是否已启用

### Q: 切割不生效？
1. 确认主光源组件是 `USuperCuttingComponent` 类型
2. 确认 `bCutting` 开关已启用
3. A 参数从 0 增加到 0.4 表示切割量增大，B 参数从 1.0 减少到 0.6 表示切割量增大

### Q: 棱镜没有效果？
1. 确认主光源组件是 `USuperBeamComponent` 类型
2. 确认灯具库中棱镜属性的 ChannelSet 定义了 `PrismFacets > 0`
3. 检查 `PrismRadius` 和 `PrismScale` 是否合理

### Q: 效果 LUT 不响应？
1. 确认已调用 `SetEffectDefaultValue` 初始化效果组件
2. 确认 `bEffect` 开关已启用
3. DMX 值 0 通常对应"无效果"

### Q: 矩阵控制只有部分像素响应？
1. 检查灯具库中模块实例数量是否与组件数组长度匹配
2. 确认每个模块实例的 Patch 值正确
3. 使用带 Matrix 后缀的函数（如 `SetLightingIntensityMatrix`）而非单灯版本

### Q: Spot 辅光与主光源不同步？
Spot 辅光有独立的控制函数（`SetSpot*` 系列），需要在蓝图中分别调用。它们共享 `LightDefaultValue` 的默认参数配置。

### Q: 频闪模式如何配置？
频闪模式通过灯具库的子属性（SubAttribute）配置。每个子属性定义 DMX 范围和对应的 `StrobeMode` 枚举。无子属性定义时，默认使用线性频闪（Mode=2）。

### Q: 如何配置 RGBWAL 灯具的颜色？
使用 `SetLightingColorMix` 函数，传入 `FSuperColorChannel` 数组。每个通道定义一个 DMX 属性和对应颜色。支持任意数量的颜色通道组合。

---

> **导航**：[← 14 - 舞台特效 VFX](14_Stage_VFX.md) | [返回总览](00_DMX_System_Overview.md)

# 04 - 电脑灯基类 (SuperLightBase)

> **所属模块**: SuperStage 运行时 (ASuperLightBase)  
> **适用对象**: 灯光设计师、蓝图开发者  
> **前置阅读**: [03 - DMX 灯具基础](/docs/stage-core/dmx-actor-base)  
> **最后更新**: 2026-03-06

---

## 一、概述

**SuperLightBase** 是所有电脑灯（Moving Light / Moving Head）的基类，继承自 SuperDmxActorBase。它在基础 DMX 能力之上，增加了舞台电脑灯特有的功能：

- **Pan / Tilt 运动控制** — 水平/垂直旋转，支持自定义角度范围
- **无极旋转 (Infinite Rotation)** — 连续旋转模式，模拟无限旋转电机
- **Pan/Tilt 速度控制** — 运动过渡速度调节
- **矩阵控制** — 多灯头独立 Pan/Tilt（矩阵灯、多颗灯珠灯具）
- **升降控制** — Z 轴位移（模拟灯具升降机），可配置升降范围和速度
- **灯具角度控制** — 光束发散角手动固定

---

## 二、类继承链

```
AActor (UE5 引擎基类)
  │
  └── ASuperBaseActor ·············· SuperStage 根基类 (L1)
        │
        └── ASuperDmxActorBase ····· DMX 灯具基类 (L2)
              │                      ├ DMX 地址 / 灯具库 / 通道读取
              │                      └ SuperDMXTick / 地址标签
              │
              └── ASuperLightBase ·· 电脑灯基类 (L3) ← 本文档描述的类
                    │                 ├ SceneRocker → SceneHard（Pan/Tilt 旋转轴）
                    │                 ├ Pan/Tilt 范围映射 + 无极旋转
                    │                 ├ PTSpeed 速度控制
                    │                 ├ 矩阵 Pan/Tilt
                    │                 ├ Z 轴升降
                    │                 └ 灯具角度
                    │
                    ├── ASuperStageLight ···· 全功能电脑灯   → 文档 15
                    ├── ASuperStageVFXActor · 舞台特效 VFX   → 文档 14
                    └── ASuperLiftMatrix ···· 升降矩阵灯阵   → 文档 12
```

> **设计要点**：`ASuperLightBase` 只管 Pan/Tilt 运动控制，完全不涉及颜色、图案、棱镜等灯光效果。这些功能由子类 `ASuperStageLight` 实现。如果你只需要一个可旋转的 DMX 设备（例如旋转的烟雾机喷头），直接继承 `ASuperLightBase` 即可。

---

## 三、组件结构

每台电脑灯 Actor 包含以下关键组件：

```
ASuperLightBase
  └── SceneBase (USceneComponent)                    ← 根组件（继承自 ASuperBaseActor）
        ├── ForwardArrow (UArrowComponent)            ← +X 方向预览（继承）
        ├── UpArrow (UArrowComponent)                 ← +Z 方向预览（继承）
        ├── AddressLabel (UTextRenderComponent)       ← DMX 地址标签（继承）
        │
        └── SceneRocker (USceneComponent)             ← 灯臂 / Pan 旋转轴（绕 Z 轴）
              │   灯具3D模型的摇臂部分附加到此组件
              │
              └── SceneHard (USceneComponent)          ← 灯头 / Tilt 旋转轴（绕 Y 轴）
                    │   灯具3D模型的灯头部分附加到此组件
                    │   子类的光束/颜色/效果组件也挂载在这里
                    │
                    └── [子类组件挂载点]
                          ├ ASuperStageLight: 光束组件、SpotLight、材质...
                          ├ ASuperStageVFXActor: UNiagaraComponent
                          └ ASuperLiftMatrix: LiftComponents[]、EffectComponents[]
```

- **SceneRocker**（灯臂）：整个灯具的水平旋转部分，Pan 旋转作用于此组件
- **SceneHard**（灯头）：灯具的垂直旋转部分，Tilt 旋转作用于此组件。所有光效组件都挂载在 SceneHard 下，确保随灯头一起旋转

---

## 四、Pan / Tilt 控制

### 4.1 范围设置

| 参数 | 属性名 | 说明 | 默认值 | 单位 |
|------|--------|------|--------|------|
| **Pan Range** | `PanRange` | Pan（水平旋转）的角度范围 | (-270, +270) | 度 |
| **Tilt Range** | `TiltRange` | Tilt（垂直旋转）的角度范围 | (-135, +135) | 度 |
| **Pan Angle** | `PanAngle` | Pan 初始偏移角度 | 0 | 度 |
| **Tilt Angle** | `TiltAngle` | Tilt 初始偏移角度 | 0 | 度 |

**范围含义**：
- PanRange = (-270, +270) 表示灯具可从 -270° 旋转到 +270°（总跨度 540°）
- TiltRange = (-135, +135) 表示灯具可从 -135° 旋转到 +135°（总跨度 270°）

> **提示**：请参考实际灯具的规格说明书设置正确的范围。常见电脑灯的 Pan 范围通常为 540°（±270°）或 630°（±315°），Tilt 范围通常为 270°（±135°）或 306°（±153°）。

**调参建议**：

| 灯具类型 | 推荐 PanRange | 推荐 TiltRange | 说明 |
|---------|--------------|---------------|------|
| 标准电脑灯 | (-270, +270) | (-135, +135) | 大多数 Moving Head 的标准规格 |
| 高速扫描灯 | (-315, +315) | (-153, +153) | Robe MegaPointe 等高端灯具 |
| 小型 Wash 灯 | (-180, +180) | (-90, +90) | 简易 Wash 灯，旋转范围较小 |
| 旋转 LED 面板 | (-180, +180) | (0, 0) | 仅 Pan 旋转，Tilt 固定 |

### 4.2 角度计算

DMX 值（归一化为 0.0-1.0）到旋转角度的映射：

```
Pan 角度 = (DMX值 - 0.5) × PanRange
Tilt 角度 = (DMX值 - 0.5) × TiltRange
```

**示例**（Pan Range = 540°）：

| DMX 值（归一化） | Pan 角度 |
|-----------------|---------|
| 0.0 | -270° |
| 0.25 | -135° |
| 0.5 | 0°（正中） |
| 0.75 | +135° |
| 1.0 | +270° |

---

## 五、功能开关

SuperLightBase 提供一系列布尔开关，用于启用/禁用特定 DMX 控制功能。这些开关在灯具蓝图的默认值中设置：

### 5.1 开关列表

| 开关 | 说明 | 默认值 |
|------|------|--------|
| **bPan** | 启用 Pan（水平旋转）控制 | 开 |
| **bTilt** | 启用 Tilt（垂直旋转）控制 | 开 |
| **bPTSpeed** | 启用 Pan/Tilt 速度控制通道 | 关 |
| **bPanRot** | 启用 Pan 无极旋转模式 | 关 |
| **bTiltRot** | 启用 Tilt 无极旋转模式 | 关 |
| **bInPosZ** | 启用 Z 轴升降位移控制 | 关 |
| **bLampAngle** | 启用灯具光束角度控制 | 关 |

### 5.2 开关使用说明

- **关闭不需要的开关**可以避免不必要的 DMX 读取和计算
- 每个开关对应一个或多个 DMX 属性
- 如果灯具不支持某个功能（例如不支持无极旋转），应关闭对应开关

---

## 六、无极旋转 (Infinite Rotation)

无极旋转允许灯具的 Pan 或 Tilt 进行**连续不停止的旋转**，而不是像普通模式那样在固定范围内定位。

### 6.1 启用无极旋转

1. 在灯具蓝图中，将 **bPanRot**（Pan 无极旋转）或 **bTiltRot**（Tilt 无极旋转）设为 **True**
2. 确保灯具库中定义了 `PanRot` 或 `TiltRot` 属性

### 6.2 无极旋转速度

| 参数 | 属性名 | 说明 | 默认值 | 范围 | 单位 |
|------|--------|------|--------|------|------|
| **Infinite Rotational Speed** | `InfiniteRotationalSpeed` | 无极旋转的最大转速 | 1.0 | 0.1 - 10.0 | 速度倍率 |

**调参建议**：
- 值为 1.0 时，转速适中（适合大多数灯具）
- 值越大转速越快，但可能导致视觉抖动
- 建议根据真实灯具的无极旋转速度（通常 0.3-2.0 转/秒）调整

### 6.3 旋转模式

| 模式 | 说明 |
|------|------|
| **Standard** | 标准模式 — 无极旋转使用默认逻辑 |
| **Custom** | 自定义模式 — 可在蓝图中自定义旋转逻辑 |

### 6.4 无极旋转工作原理

无极旋转通道的 DMX 值映射：

| DMX 值范围（归一化） | 行为 |
|---------------------|------|
| 0.0 - 0.49 | 逆时针旋转，速度从最大递减到停止 |
| 0.49 - 0.51 | 停止（死区） |
| 0.51 - 1.0 | 顺时针旋转，速度从停止递增到最大 |

---

## 七、Pan/Tilt 速度控制

当 **bPTSpeed** 开启时，灯具会响应速度控制通道，用于调整 Pan/Tilt 运动的过渡速度。

### 7.1 速度参数

| 参数 | 属性名 | 说明 | 默认值 | 范围 |
|------|--------|------|--------|------|
| **PT Speed** | `PTSpeed` | Pan/Tilt 移动插值速度 | 5.0 | 0.1 - 20.0 |

**调参建议**：
- 值越大，灯具响应越快（更接近"跳转"）
- 值越小，灯具响应越慢（更平滑的过渡）
- 推荐范围 3.0-8.0，模拟真实灯具的运动惯性
- 快速追光场景建议 8.0-15.0
- 慢速氛围场景建议 1.0-3.0

### 7.2 工作方式

- 速度通道 DMX 值 = 0 → 最快速度（直接跳到目标位置）
- 速度通道 DMX 值 = 255 → 最慢速度（缓慢移动到目标位置）

> **注意**：速度控制的具体行为取决于灯具蓝图中的实现。基类提供了 YPTSpeed 函数作为框架。

---

## 八、矩阵控制 (Matrix)

矩阵功能用于控制**多灯头灯具**，例如 LED 面板灯、多颗灯珠的 Wash 灯等。

### 8.1 矩阵概念

一台矩阵灯具包含多个"灯头"（模块实例），每个灯头可以独立控制：

```
一台 7×1 矩阵灯
┌───┬───┬───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │  ← 7 个模块实例
└───┴───┴───┴───┴───┴───┴───┘
 │    │    │    │    │    │    │
 各自独立的 RGBW / Pan / Tilt 通道
```

### 8.2 矩阵 Pan/Tilt

当灯具有多个灯头且每个灯头可独立旋转时，SuperLightBase 提供矩阵 Pan/Tilt 功能：

- **YMatrixPan** — 所有灯头的独立 Pan 控制
- **YMatrixTilt** — 所有灯头的独立 Tilt 控制
- **YMatrixPanRot** — 所有灯头的独立 Pan 无极旋转
- **YMatrixTiltRot** — 所有灯头的独立 Tilt 无极旋转

### 8.3 矩阵读取方式

在蓝图中，使用 **Get Matrix Attribute Raw** 节点可以一次性获取所有模块实例的某个属性值：

- 输入：属性名（如 `Red`）
- 输出：浮点数组，每个元素是一个模块实例的值（0.0 - 1.0）

**带索引的矩阵读取**（Get Matrix Attribute Raw With Index）：
- 额外输出模块索引数组，便于知道每个值对应哪个模块实例
- 结果按 Patch 值排序

### 8.4 矩阵配置要求

1. 灯具库中必须定义多个模块实例（每个灯头一个）
2. 每个模块实例的 Patch 值正确设置
3. 每个模块实例中定义相同名称的属性（如每个像素都有 Red、Green、Blue）

---

## 九、升降控制 (Lift Z)

当 **bInPosZ** 开启时，灯具可以通过 DMX 控制其 Z 轴位置（上下移动），模拟灯具升降机的效果。

### 9.1 升降参数

| 参数 | 属性名 | 说明 | 默认值 | 范围 | 单位 |
|------|--------|------|--------|------|------|
| **Lift Range** | `LiftRange` | 最大升降距离 | 500.0 | 0 - 任意正数 | 厘米 |
| **Lift Speed** | `LiftSpeed` | 升降插值速度 | 1.0 | 0.1 - 10.0 | 速度倍率 |
| **InPosZ** | `InPosZ` | 当前 Z 轴升降控制值 | 0.0 | 0.0 - 1.0 | 归一化 |

### 9.2 升降映射

```
Z 位移 = DMX值(0~1) × LiftRange
```

- DMX 值 = 0.0 → Z 偏移 = 0cm（原始位置）
- DMX 值 = 0.5 → Z 偏移 = 250cm（LiftRange 默认 500 时）
- DMX 值 = 1.0 → Z 偏移 = 500cm（最大升降距离）

**调参建议**：

| 场景 | 推荐 LiftRange | 推荐 LiftSpeed | 说明 |
|------|---------------|----------------|------|
| 小型舞台升降 | 200-500 cm | 1.0-2.0 | 标准升降台 |
| 大型演出升降 | 500-1500 cm | 0.5-1.5 | 大行程升降机 |
| 快速弹出效果 | 50-200 cm | 5.0-10.0 | 突然弹出的视觉效果 |

**SetLiftZ** 函数接收归一化 DMX 值，映射到灯具的 Z 轴位移范围。

---

## 十、灯具角度控制 (Lamp Angle)

当 **bLampAngle** 开启时，灯具可以通过 DMX 控制其光束发散角（而非使用 Zoom 变焦通道）。

**YLampAngle** 函数接收归一化 DMX 值，映射到灯具光束角度范围。

> **注意**：此功能用于手动固定灯具角度，与 `ASuperStageLight` 中的 Zoom 功能不同。Zoom 是通过光学系统实现的连续变焦，LampAngle 是灯体物理角度的手动调节。

---

## 十一、完整默认参数一览

以下是 `ASuperLightBase` 层级的所有可配置默认参数：

| 参数 | 属性名 | 类别 | 默认值 | 范围 | 说明 |
|------|--------|------|--------|------|------|
| Pan 初始角度 | `PanAngle` | B.DefaultParameter | 0 | -360~360 | Pan 轴初始偏移 |
| Tilt 初始角度 | `TiltAngle` | B.DefaultParameter | 0 | -360~360 | Tilt 轴初始偏移 |
| Pan 范围 | `PanRange` | B.DefaultParameter | (-270, +270) | 自定义 | 水平旋转角度范围 |
| Tilt 范围 | `TiltRange` | B.DefaultParameter | (-135, +135) | 自定义 | 垂直旋转角度范围 |
| PT 速度 | `PTSpeed` | B.DefaultParameter | 5.0 | 0~20 | Pan/Tilt 插值速度 |
| 无极旋转速度 | `InfiniteRotationalSpeed` | B.DefaultParameter | 1.0 | 0~10 | 无极旋转速度倍率 |
| 升降范围 | `LiftRange` | B.DefaultParameter | 500.0 | 0~任意 | Z 轴升降最大距离(cm) |
| 升降速度 | `LiftSpeed` | B.DefaultParameter | 1.0 | 0~10 | 升降插值速度 |

---

## 十二、内置控制函数一览

以下是 SuperLightBase 提供的核心控制函数（在蓝图中以 BlueprintCallable 形式调用）：

| 函数 | 说明 | 输入 |
|------|------|------|
| **YPan** | 设置 Pan 旋转角度 | DMX Attribute |
| **YTilt** | 设置 Tilt 旋转角度 | DMX Attribute |
| **YPTSpeed** | 设置 Pan/Tilt 运动速度 | DMX Attribute |
| **YPanRot** | 设置 Pan 无极旋转 | DMX Attribute |
| **YTiltRot** | 设置 Tilt 无极旋转 | DMX Attribute |
| **YMatrixPan** | 矩阵灯各灯头独立 Pan | 属性名 |
| **YMatrixTilt** | 矩阵灯各灯头独立 Tilt | 属性名 |
| **YMatrixPanRot** | 矩阵灯各灯头独立 Pan 无极旋转 | 属性名 |
| **YMatrixTiltRot** | 矩阵灯各灯头独立 Tilt 无极旋转 | 属性名 |
| **YLampAngle** | 设置灯具光束角度 | DMX Attribute |
| **SetLiftZ** | 设置灯具 Z 轴升降位置 | DMX Attribute |

---

## 十三、典型电脑灯蓝图结构

一个典型的电脑灯蓝图中 SuperDMXTick 事件的实现流程：

```
Event SuperDMXTick(DeltaTime)
  │
  ├─ YPan(PanAttribute)           ← 设置水平旋转
  ├─ YTilt(TiltAttribute)         ← 设置垂直旋转
  ├─ YPTSpeed(SpeedAttribute)     ← 设置运动速度（可选）
  │
  ├─ GetSuperDmxAttributeValue("Dimmer") → 设置光照强度
  ├─ GetSuperDMXColorValue(R, G, B)      → 设置灯光颜色
  │
  ├─ GetSuperDmxAttributeValue("Zoom")   → 设置光束角度
  ├─ GetSuperDmxAttributeValue("Focus")  → 设置对焦
  │
  └─ GetSuperDmxAttributeValue("Gobo")   → 切换 Gobo 图案
```

---

## 十四、常见问题

### Q: Pan/Tilt 方向反了？
检查灯具蓝图中 SceneRocker 和 SceneHard 组件的旋转轴设置。Pan 应绕 Z 轴旋转，Tilt 应绕 Y 轴旋转。

### Q: Pan/Tilt 运动不够平滑？
确保灯具库中的 Pan/Tilt 属性设置了 Fine 通道（16 位精度），并且 Channel Type 设为 Fine。

### Q: 矩阵灯只有一个灯头响应？
1. 检查灯具库中是否为每个灯头创建了独立的模块实例
2. 确认每个模块实例的 Patch 值正确
3. 确认蓝图中使用的是矩阵读取函数（GetMatrixAttributeRaw）而非单实例读取

### Q: 无极旋转不工作？
1. 确认 bPanRot 或 bTiltRot 开关已开启
2. 确认灯具库中定义了 PanRot 或 TiltRot 属性
3. 确认 DMX 通道有变化的值（不在死区范围内）

### Q: 灯具角度范围不对？
调整细节面板中的 Pan Range 和 Tilt Range 值，使其与实际灯具规格一致。

---

> **下一步**：请阅读 [05 - DMX 摄像机](/docs/stage-core/dmx-camera) 了解如何通过 DMX 控制虚拟摄像机。

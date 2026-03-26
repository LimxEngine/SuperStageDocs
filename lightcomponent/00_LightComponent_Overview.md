# 光源组件总览 (Light Component Overview)

> **所属模块**: SuperStage 运行时  
> **适用对象**: 灯光设计师、舞美技术人员、蓝图开发者  
> **前置阅读**: [15 - 全功能电脑灯 (SuperStageLight)](../15_SuperStageLight.md)

---

## 一、概述

SuperStage 的光源组件系统是灯具 Actor（如 SuperStageLight）内部实际承载「灯光渲染」的核心模块。每个灯具 Actor 可以组合使用多种光源组件，从而模拟真实舞台灯具的各类出光方式——聚光灯投射、体积光束、切割成型、矩形面光、效果图案、矩阵像素、激光投射等。

灯光设计师在日常工作中并不需要直接操作这些组件——它们由灯具蓝图预先配置好，并通过 DMX 通道自动驱动。但理解这些组件的参数含义和工作方式，有助于：

- **制作自定义灯具蓝图**时正确选择和配置组件
- **调试灯光效果**时快速定位问题所在
- **优化场景性能**时了解各组件的开销特征

---

## 二、组件继承关系

SuperStage 的光源组件形成了一个清晰的继承体系：

```
USceneComponent（引擎基类）
  │
  ├── USuperLightingComponent ← 基础灯光组件（镜片 + 光斑材质）
  │     │
  │     ├── USuperSpotComponent ← 聚光灯组件（SpotLight + 变焦/雾化/光圈）
  │     │     │
  │     │     └── USuperBeamComponent ← 体积光束组件（光束网格 + 大气散射）
  │     │           │
  │     │           └── USuperCuttingComponent ← 切割光束组件（四叶光闸裁剪）
  │     │
  │     └── USuperRectComponent ← 矩形面光组件（RectLight + 遮光板）
  │
  ├── USuperEffectComponent ← 效果平面组件（LED屏/效果面板材质驱动）
  │
  ├── USuperMatrixComponent ← 矩阵灯组件（分段像素控制 + 分段光源）
  │
  ├── USuperLaserProComponent ← 激光 Pro 组件（点数据驱动激光线）
  │
  └── USuperLiftComponent ← 升降控制组件（Z轴位移）
```

---

## 三、组件速查表

| 组件 | 用途 | 典型灯具 | 文档链接 |
|------|------|----------|----------|
| **SuperLightingComponent** | 基础灯光渲染（镜片+光斑） | 所有灯具的光源基础 | [01_SuperLightingComponent.md](01_SuperLightingComponent.md) |
| **SuperSpotComponent** | 聚光灯投射（真实 SpotLight） | Spot 灯、追光灯 | [02_SuperSpotComponent.md](02_SuperSpotComponent.md) |
| **SuperBeamComponent** | 体积光束可视化 | Beam 灯、光束灯 | [03_SuperBeamComponent.md](03_SuperBeamComponent.md) |
| **SuperCuttingComponent** | 光闸切割成型 | Profile 灯、成像灯 | [04_SuperCuttingComponent.md](04_SuperCuttingComponent.md) |
| **SuperRectComponent** | 矩形面光照明 | LED 面板灯、柔光灯 | [05_SuperRectComponent.md](05_SuperRectComponent.md) |
| **SuperEffectComponent** | 效果面板/LED 屏幕 | LED 效果灯、频闪灯 | [06_SuperEffectComponent.md](06_SuperEffectComponent.md) |
| **SuperMatrixComponent** | 矩阵像素分段控制 | 矩阵灯、LED 条灯 | [07_SuperMatrixComponent.md](07_SuperMatrixComponent.md) |
| **SuperLaserProComponent** | 激光线投射 | 激光灯（Beyond 驱动） | [08_SuperLaserProComponent.md](08_SuperLaserProComponent.md) |
| **SuperLiftComponent** | Z 轴升降运动 | 带升降功能的灯具 | [09_SuperLiftComponent.md](09_SuperLiftComponent.md) |

---

## 四、典型灯具的组件配置

不同类型的灯具使用不同的组件组合。以下是常见灯具的配置参考：

### 4.1 Spot 灯（仅投射光斑，无可见光束）

```
SuperStageLight
  └── 灯头
        └── SuperSpotComponent (主光源)
```

- 使用 SpotLight 产生真实光照
- 通过光照函数材质投射 Gobo 图案
- 适合不需要可见光束效果的场景

### 4.2 Beam 灯（体积光束 + 投射光斑）

```
SuperStageLight
  └── 灯头
        └── SuperBeamComponent (主光源)
              ├── SpotLight — 投射光照
              ├── 镜片网格 — 镜片发光效果
              └── 光束网格 — 可见体积光束
```

- 在 Spot 基础上增加了可见的体积光束网格
- 光束受 Zoom、Frost、颜色、Gobo 等参数影响
- 支持大气散射、烟雾流动效果

### 4.3 Profile/切割灯（光闸裁剪 + 体积光束）

```
SuperStageLight
  └── 灯头
        └── SuperCuttingComponent (主光源)
              ├── SpotLight — 投射光照
              ├── 镜片网格 — 镜片发光 + 切割图案
              └── 光束网格 — 可见体积光束 + 切割形状
```

- 在 Beam 基础上增加了四组切割叶片控制
- 可以将光斑裁剪为任意四边形
- 支持切割角度旋转

### 4.4 Wash 灯/面板灯（矩形面光）

```
SuperStageLight
  └── 灯头
        └── SuperRectComponent (主光源)
              ├── RectLight — 矩形面光照明
              └── 镜片网格 — 镜片发光效果
```

- 使用 RectLight 产生柔和的面光照明
- 支持遮光板（BarnDoor）控制光束形状
- 适合 Wash 类灯具和 LED 面板灯

### 4.5 效果灯（LED 效果面板）

```
SuperStageLight
  └── 灯头
        ├── SuperBeamComponent (主光源，可选)
        └── SuperEffectComponent (效果面板)
```

- 使用平面网格 + 材质实现 LED 效果显示
- 支持独立的亮度、频闪、颜色控制
- 支持效果图案动画（速度、宽度、方向）

### 4.6 矩阵灯（像素级控制）

```
SuperStageLight
  └── 灯头
        ├── SuperBeamComponent (主光源，可选)
        └── SuperMatrixComponent (矩阵像素)
              └── 分段光源[] (PointLight 或 SpotLight)
```

- 将灯具面板分为多个独立像素段
- 每段可独立控制颜色
- 可选配真实光源（PointLight/SpotLight）映射像素

### 4.7 激光灯（Beyond 驱动）

```
SuperStageLight / LaserActor
  └── SuperLaserProComponent
        └── ProceduralMesh — 激光线几何体
```

- 由 Beyond 激光控制器提供点数据
- 实时生成激光线程序化网格
- 支持碰撞遮挡检测

---

## 五、通用功能说明

以下功能在多个光源组件中通用，这里统一说明。

### 5.1 亮度控制 (Dimmer)

所有光源组件的最终亮度由以下因素共同决定：

```
最终亮度 = Actor级Dimmer × 组件级亮度分控(MaxLightIntensity) × 频闪倍率(Strobe)
```

| 参数 | 说明 | 范围 | 默认值 |
|------|------|------|--------|
| **Actor 级 Dimmer** | 灯具整体亮度，由 DMX Dimmer 通道控制 | 0.0 ~ 1.0 | 1.0 |
| **MaxLightIntensity** | 组件级亮度分控，用于多模组灯具中独立控制某个组件的亮度 | 0.0 ~ 1.0 | 1.0 |

> **使用场景**：当一个灯具 Actor 同时拥有主光源和辅光源时，可以通过 MaxLightIntensity 独立降低辅光源的亮度，而不影响主光源。

> **初始化说明**：组件默认参数由 `OnRegister()` 自动初始化，组件注册时会自动调用 `SetLightingMaterial()` 创建材质实例，再调用 `SetLightingDefaultValue()` 设置默认值。无需在 Actor 中手动调用初始化函数。

### 5.2 亮度响应曲线 (Dimmer Curve)

灯光组件支持「亮度响应曲线」功能，用于将线性 DMX 值转换为感知上更均匀的亮度输出。

| 指数值 | 曲线名称 | 说明 |
|--------|----------|------|
| **1.0** | 线性 | DMX 值直接映射为亮度（不推荐，低段变化剧烈，高段变化平坦） |
| **2.0** | 平方曲线 | **推荐**，模拟真实调光台的感知曲线，调光过渡更自然 |
| **2.2** | Gamma 校正 | 显示器标准校正曲线 |
| **3.0** | 立方曲线 | 更强的非线性响应，低段更平缓 |

> **提示**：默认值为 2.0（平方曲线），适合大多数舞台灯具。如果觉得低亮度段变化太快，可以适当增大指数值。

### 5.3 频闪系统 (Strobe)

所有光源组件共享相同的频闪计算逻辑，支持 8 种频闪模式：

| 模式编号 | 名称 | 效果描述 |
|----------|------|----------|
| **0** | Closed（关闭） | 灯光完全关闭（亮度 = 0） |
| **1** | Open（常亮） | 灯光常亮，无频闪效果（默认） |
| **2** | Linear（线性） | 三角波渐亮渐灭，带平滑过渡 |
| **3** | Pulse（脉冲） | 方波闪烁，50% 亮 / 50% 灭，硬切换 |
| **4** | Ramp Up（渐亮） | 锯齿波，从暗到亮线性上升后瞬间变暗 |
| **5** | Ramp Down（渐灭） | 锯齿波，从亮到暗线性下降后瞬间变亮 |
| **6** | Sine（正弦） | 正弦波平滑脉动，最柔和的频闪效果 |
| **7** | Random（随机） | 随机乱闪，每个时间槽随机决定亮或灭 |

频闪速度由 DMX Strobe 通道控制，值越大闪烁越快。

> **性能提示**：频闪模式 0（Closed）和 1（Open）不需要实时计算，组件会自动关闭 Tick 以节省性能。只有模式 2~7 才会启用每帧更新。

### 5.4 碰撞检测 (Ray Detection)

带光照的组件（Spot、Beam、Cutting、Rect）支持光线碰撞检测，用于自动截断光照距离：

- 从灯具镜片位置向出光方向发射射线
- 检测射线与场景中物体的交点
- 将光照距离限制在碰撞点处（灯光不会穿透墙壁/地板）
- 使用两级检测策略：先粗射线（球体扫掠），再细射线（兜底）

### 5.5 镜片模型 (Lens)

基础灯光组件及其子类都支持镜片模型显示：

| 参数 | 说明 |
|------|------|
| **StaticMeshLens** | 镜片使用的静态网格模型 |
| **LensTransform** | 镜片相对于组件的位置/旋转/缩放偏移 |

镜片模型会随灯光亮度、颜色、图案等参数变化而实时更新材质表现，模拟真实灯具镜片的发光效果。

---

## 六、与灯具 Actor 的关系

光源组件**不能独立存在**，必须作为灯具 Actor（如 SuperStageLight）的子组件使用。灯具 Actor 负责：

1. **接收 DMX 数据** — 从 DMX 子系统获取通道值
2. **解析灯具库定义** — 将通道值映射到具体参数
3. **驱动光源组件** — 调用组件的参数设置接口更新视觉效果
4. **管理运动控制** — Pan/Tilt 旋转影响组件的出光方向

灯光设计师通常只需要：
- 在灯具蓝图中**添加和配置**所需的光源组件
- 在灯具库中**定义 DMX 通道映射**
- 在控台或 Sequencer 中**发送 DMX 信号**控制灯光效果

---

## 七、性能优化建议

| 建议 | 说明 |
|------|------|
| **合理选择组件类型** | 不需要体积光束时使用 SuperSpotComponent 而非 SuperBeamComponent |
| **控制矩阵分段数** | SuperMatrixComponent 的分段数不宜超过必要数量（最大 200） |
| **关闭不需要的碰撞检测** | 激光组件的碰撞检测有一定开销，不需要时可关闭 |
| **利用 MaxLightIntensity** | 将不使用的组件亮度分控设为 0，系统会自动优化 |
| **注意频闪模式** | 频闪模式 0/1 不消耗 Tick 资源；模式 2~7 会启用每帧计算 |
| **透明材质按需启用** | Effect 和 Matrix 组件的透明材质比不透明材质开销略高 |

---

## 八、各组件详细文档

请点击以下链接查看各组件的详细参数说明和使用指南：

1. [SuperLightingComponent — 基础灯光组件](01_SuperLightingComponent.md)
2. [SuperSpotComponent — 聚光灯组件](02_SuperSpotComponent.md)
3. [SuperBeamComponent — 体积光束组件](03_SuperBeamComponent.md)
4. [SuperCuttingComponent — 切割光束组件](04_SuperCuttingComponent.md)
5. [SuperRectComponent — 矩形面光组件](05_SuperRectComponent.md)
6. [SuperEffectComponent — 效果平面组件](06_SuperEffectComponent.md)
7. [SuperMatrixComponent — 矩阵灯组件](07_SuperMatrixComponent.md)
8. [SuperLaserProComponent — 激光 Pro 组件](08_SuperLaserProComponent.md)
9. [SuperLiftComponent — 升降控制组件](09_SuperLiftComponent.md)

# 11 - 激光系统 (Laser System)

> **所属模块**: SuperStage 运行时 (ASuperLaserActor / ASuperLaserProActor)  
> **适用对象**: 灯光设计师、激光编程师、虚拟制作技术人员  
> **前置阅读**: [00 - DMX 系统总览](/docs/stage-core/overview)  
> **最后更新**: 2026-03-06

---

## 一、概述

SuperStage 提供与 **Pangolin Beyond** 激光控制软件的实时集成，可以在 UE 场景中**实时可视化**激光投射效果。系统提供两种激光 Actor，适用于不同的渲染模式：

| 类型 | 类名 | 渲染方式 | 说明 |
|------|------|---------|------|
| **纹理投射激光** | SuperLaserActor | 动态材质 + 纹理映射 | 通过 RenderTarget 纹理驱动光束材质，模拟激光投影效果 |
| **点数据激光 (Pro)** | SuperLaserProActor | 程序化网格绘制 | 通过激光点坐标数据绘制精确的激光线条，支持碰撞遮挡 |

两者都从 **SuperLaserSubsystem**（激光引擎子系统）获取 Beyond 设备的实时数据。

### 类继承链

```
AActor (UE5 引擎基类)
  │
  └── ASuperBaseActor ·············· SuperStage 根基类 (L1)
        │                           ├ SceneBase / ForwardArrow / UpArrow
        │                           └ AssetMetaData
        │
        ├── ASuperLaserActor ······· 纹理投射激光 (L2)
        │     ├ DeviceID → SuperLaserSubsystem 获取 RenderTarget
        │     ├ 动态材质参数（LaserIntensity/MaxDistance/雾效）
        │     └ 光束静态网格渲染
        │
        └── ASuperLaserProActor ···· 点数据激光 Pro (L2)
              ├ DeviceID → SuperLaserSubsystem 获取点数据
              ├ SuperLaserProComponent → 程序化网格
              ├ EnableCollision → 碰撞遮挡检测
              └ Sprite + Arrow 编辑器辅助
```

> **设计要点**：两种激光 Actor 直接继承 `ASuperBaseActor`（而非 `ASuperDmxActorBase`），因为它们**不使用 DMX 协议控制**——数据直接来自 Pangolin Beyond 软件的实时流，不经过 DMX 通道。因此它们没有 Universe/Address 配置，也没有灯具库。

### 应用场景

- **演唱会 / 音乐节** — 在虚拟场景中预览激光秀效果
- **虚拟制作** — 激光与舞台灯光的集成调试
- **方案演示** — 向客户展示激光效果的 3D 预览
- **编程调试** — 在 UE 编辑器中实时查看 Beyond 的激光输出

### 数据流

```
Pangolin Beyond 软件
       │
       │ 实时数据流（纹理 / 点数据）
       ▼
SuperLaserSubsystem（引擎子系统）
       │
       ├──→ SuperLaserActor（纹理模式）
       │      └─ RenderTarget → 动态材质 → 光束网格
       │
       └──→ SuperLaserProActor（点数据模式）
              └─ 点坐标数组 → SuperLaserProComponent → 程序化网格
```

---

# 第一部分：纹理投射激光 (SuperLaserActor)

## 二、基本原理

SuperLaserActor 使用 Beyond 设备输出的 **RenderTarget 纹理** 作为激光图案，将其投射到一个光束形状的静态网格上，模拟激光在空中的投射效果。

**渲染流程**：
```
每帧 Tick
  ↓
从 SuperLaserSubsystem 获取设备纹理 (RenderTarget)
  ↓
设置到动态材质的 LightTexture 参数
  ↓
根据强度控制光束可见性
  ↓
渲染光束静态网格
```

---

## 三、属性详解

### 3.1 设备配置

| 参数 | 位置 | 说明 | 范围 | 默认值 |
|------|------|------|------|--------|
| **DeviceID** | SuperLaser | Beyond 设备编号。对应 Beyond 软件中的设备序号 | 1 - 12 | 1 |

**DeviceID 说明**：
- 每个 DeviceID 对应 Beyond 中的一台激光设备
- 支持最多 **12 台** Beyond 设备同时连接
- DeviceID 从 **1** 开始编号（与 Beyond 界面一致）
- 不同的 SuperLaserActor 可以设置不同的 DeviceID，显示不同设备的激光效果

### 3.2 光束外观参数

| 参数 | 说明 | 范围 | 默认值 | 单位 |
|------|------|------|--------|------|
| **LaserIntensity** | 激光亮度强度 | 1 - 5,000 | 100 | 百分比 |
| **MaxLaserDistance** | 光束最大投射距离 | 100 - 10,000 | 2,345 | 厘米 |
| **AtmosphericDensity** | 大气衰减系数（光束随距离淡化的程度） | 0.0 - 1.0 | 0.03 | — |
| **FogIntensity** | 光束中雾效强度 | 0 - 100 | 20 | 百分比 |
| **AtmosFogSpeed** | 雾效流动速度 | 0 - 100 | 10 | 百分比 |

#### LaserIntensity（激光强度）

控制光束的整体亮度：
- 值越高，光束越亮
- 设为 **0** 时光束自动隐藏（节省渲染性能）
- 建议范围：50 ~ 500（过高可能导致过曝）

#### MaxLaserDistance（最大投射距离）

控制光束在场景中能延伸多远：
- 单位为厘米（100cm = 1米）
- 默认 2,345cm ≈ 23.45 米
- 根据场景大小调整——大型场馆需要更大的值

#### AtmosphericDensity（大气衰减）

模拟光束在空气中的散射衰减：
- **0.0** → 无衰减（光束从头到尾同样亮）
- **0.03**（默认）→ 轻微衰减（真实效果）
- **1.0** → 极强衰减（光束很快消失）

#### FogIntensity（雾效强度）

控制光束中的体积雾效果（模拟烟雾中的激光散射）：
- **0** → 无雾效（干净的光束线）
- **20**（默认）→ 轻微雾效
- **100** → 浓厚雾效（类似舞台烟雾中的效果）

#### AtmosFogSpeed（雾效流速）

控制雾效纹理的流动动画速度：
- **0** → 静止
- **10**（默认）→ 缓慢流动
- **100** → 快速流动

### 3.3 调参建议

| 场景 | LaserIntensity | MaxLaserDistance | AtmosphericDensity | FogIntensity | FogSpeed | 说明 |
|------|---------------|-----------------|-------------------|-------------|---------|------|
| **小型演播室** | 50 - 150 | 1000 - 1500 | 0.05 - 0.1 | 10 - 30 | 5 - 10 | 近距离投射，避免过曝 |
| **中型剧场** | 100 - 300 | 2000 - 3000 | 0.03 | 20 - 40 | 10 | 标准舞台效果 |
| **大型体育馆** | 200 - 500 | 5000 - 8000 | 0.01 - 0.02 | 30 - 60 | 10 - 20 | 远距离需要高亮低衰减 |
| **户外音乐节** | 300 - 800 | 8000 - 10000 | 0.005 - 0.01 | 5 - 15 | 5 | 开放空间雾效稀薄 |
| **沉浸式体验** | 50 - 100 | 500 - 1000 | 0.1 - 0.3 | 50 - 100 | 15 - 30 | 近距离浓雾营造氛围 |

> **性能提示**：Pro 模式 (SuperLaserProActor) 的 `EnableCollision` 会为每条激光线段执行射线检测，在线段密集的激光秀中会增加 CPU 负载。建议仅在需要遮挡效果时启用。

---

## 四、使用步骤（纹理模式）

1. **确保 Beyond 已启动** — Pangolin Beyond 软件正在运行并已加载激光内容
2. **确认连接** — SuperLaserSubsystem 已与 Beyond 建立连接
3. **放置 Actor** — 将 SuperLaserActor 蓝图拖入场景
4. **设置 DeviceID** — 选择对应的 Beyond 设备编号
5. **调整位置** — 将 Actor 放置到激光投射器的位置，调整旋转方向
6. **调整外观参数** — 根据场景调整强度、距离、雾效等
7. **实时预览** — Beyond 中播放激光内容，UE 场景中同步显示

---

# 第二部分：点数据激光 Pro (SuperLaserProActor)

## 五、基本原理

SuperLaserProActor 使用 Beyond 设备输出的**激光点坐标数据**，通过 SuperLaserProComponent 以程序化网格的方式**精确绘制激光线条**。

与纹理模式相比，点数据模式具有更高的精度和更真实的效果。

**数据流**：
```
每帧 Tick
  ↓
从 SuperLaserSubsystem 获取点数据数组 (TArray<FLaserPoint>)
  ↓
传递给 SuperLaserProComponent
  ↓
程序化网格绘制激光线
  ↓
（可选）碰撞遮挡检测
```

### 点数据格式

每个激光点包含以下信息：

| 字段 | 说明 | 范围 |
|------|------|------|
| **X** | 水平位置 | -1.0 ~ 1.0（归一化到投射范围） |
| **Y** | 垂直位置 | -1.0 ~ 1.0（归一化到投射范围） |
| **Z** | 光束标记 | > 0 表示光束点（激光开启），≤ 0 表示暗移点 |
| **R** | 红色分量 | 0.0 ~ 1.0 |
| **G** | 绿色分量 | 0.0 ~ 1.0 |
| **B** | 蓝色分量 | 0.0 ~ 1.0 |

---

## 六、属性详解

### 6.1 设备配置

| 参数 | 位置 | 说明 | 范围 | 默认值 |
|------|------|------|------|--------|
| **DeviceID** | A.DefaultParameter | Beyond 设备编号 | 1 - 12 | 1 |
| **EnableCollision** | A.DefaultParameter | 启用碰撞遮挡检测 | 开/关 | 关 |

#### EnableCollision（碰撞遮挡）

- **关（默认）** — 激光线穿透所有物体（纯视觉效果）
- **开** — 激光线遇到场景中的物体会**被遮挡截断**，模拟真实激光照射到物体表面的效果

> **性能提示**：碰撞遮挡会消耗额外的 CPU 资源（射线检测），仅在需要真实遮挡效果时启用。

### 6.2 激光组件

| 参数 | 位置 | 说明 |
|------|------|------|
| **LaserComponent** | B.Component | SuperLaserProComponent 实例（自动创建，通常无需手动修改） |

LaserComponent 是实际绘制激光线条的程序化网格组件。它的渲染参数（线宽、颜色强度等）可以在组件的属性中单独调整。

### 6.3 编辑器辅助

在编辑器中，SuperLaserProActor 提供以下辅助可视化：

| 组件 | 说明 |
|------|------|
| **Sprite（图标）** | 在场景中显示一个灯光图标，便于定位激光投射器 |
| **Arrow（方向箭头）** | 绿色箭头指向 -Z 方向（向下），表示激光的默认投射方向 |

---

## 七、使用步骤（Pro 模式）

1. **确保 Beyond 已启动** — Pangolin Beyond 软件运行中
2. **放置 Actor** — 将 SuperLaserProActor 蓝图拖入场景
3. **设置 DeviceID** — 选择对应的 Beyond 设备编号
4. **调整位置和方向** — 放置到激光投射器位置，箭头方向为投射方向
5. **（可选）启用碰撞** — EnableCollision = True（如需遮挡效果）
6. **实时预览** — Beyond 播放激光内容时，UE 场景中同步显示精确的激光线条

---

## 八、蓝图控制接口

### SuperLaserProActor 提供的蓝图函数

| 函数 | 说明 |
|------|------|
| **SetLaserEnabled(bEnabled)** | 手动开启/关闭激光显示 |

通常不需要手动调用此函数——系统会根据 Beyond 的数据自动控制可见性（有点数据时显示，无数据时隐藏）。

---

## 九、纹理模式 vs Pro 模式对比

| 特性 | 纹理模式 (SuperLaserActor) | Pro 模式 (SuperLaserProActor) |
|------|--------------------------|------------------------------|
| **渲染方式** | 纹理映射到光束网格 | 程序化网格绘制激光线 |
| **精度** | 中等（受纹理分辨率限制） | **高**（基于精确坐标数据） |
| **外观** | 光束体积效果（类似投影） | 精确激光线条 |
| **碰撞遮挡** | 不支持 | **支持** |
| **雾效控制** | **支持**（5 个参数） | 取决于组件设置 |
| **性能开销** | 较低（静态网格 + 材质） | 中等（程序化网格每帧更新） |
| **适用场景** | 远景/氛围激光效果 | 近景/精确激光线条 |
| **数据来源** | RenderTarget 纹理 | 点坐标数组 |

### 选择建议

- **追求视觉氛围** → 使用纹理模式（有雾效、光束体积感）
- **追求精确还原** → 使用 Pro 模式（精确线条、碰撞遮挡）
- **两者混合** → 同一个 DeviceID 可以同时放置两种 Actor，叠加效果

---

## 十、多设备配置

SuperStage 支持同时连接**最多 12 台** Beyond 设备。每台设备可以输出独立的激光内容。

### 多设备场景示例

```
Beyond 设备 1 (DeviceID=1) → SuperLaserProActor_1 (舞台左侧激光)
Beyond 设备 2 (DeviceID=2) → SuperLaserProActor_2 (舞台右侧激光)
Beyond 设备 3 (DeviceID=3) → SuperLaserActor_1   (背景投影激光)
Beyond 设备 4 (DeviceID=4) → SuperLaserProActor_3 (观众区扫射激光)
```

每台 Actor 独立配置 DeviceID，系统自动从 SubSystem 获取对应设备的数据。

---

## 十一、编辑器实时预览

两种激光 Actor 都支持**编辑器实时预览**——无需点击 Play 按钮：

- 编辑器模式下 Tick 始终启用（ShouldTickIfViewportsOnly = true）
- 只要 Beyond 软件正在运行并发送数据，编辑器中就能看到激光效果
- 修改参数（如 LaserIntensity、DeviceID）后立即生效

---

## 十二、常见问题

### Q: 场景中看不到激光？
1. 确认 **Beyond 软件已启动**并正在播放激光内容
2. 确认 **SuperLaserSubsystem** 已与 Beyond 建立连接
3. 检查 **DeviceID** 是否与 Beyond 中的设备编号一致
4. （纹理模式）检查 **LaserIntensity** 是否 > 0
5. 确认 Actor 的**旋转方向**正确（激光应指向预期方向）

### Q: 激光在编辑器中可见但运行时消失？
检查是否有权限相关的限制。系统使用 `SUPER_REQUIRE_PERMISSION` 进行授权检查。

### Q: 激光穿透了墙壁？
- **纹理模式**：不支持碰撞遮挡，这是正常行为
- **Pro 模式**：将 **EnableCollision** 设为 **True** 可启用碰撞遮挡

### Q: 激光效果卡顿/不流畅？
1. 确认 Beyond 的数据输出频率正常
2. （Pro 模式）大量激光点可能导致性能问题，尝试在 Beyond 中降低点密度
3. 确认 UE 编辑器帧率正常（大场景可能导致帧率下降）

### Q: 如何让激光指向特定方向？
调整 Actor 的**旋转**：
- **纹理模式**：光束网格默认沿 Actor 的前方方向投射
- **Pro 模式**：默认投射方向为 -Z（向下），绿色箭头指示方向。旋转 Actor 可改变投射方向

### Q: 可以同时使用两种模式吗？
可以。同一个 DeviceID 可以同时用 SuperLaserActor（纹理模式）和 SuperLaserProActor（Pro 模式），两者会显示同一台设备的激光内容，效果叠加。

---

> **返回总览**：[00 - DMX 系统总览](/docs/stage-core/overview)

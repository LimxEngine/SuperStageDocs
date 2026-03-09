# SuperLaser 激光系统 — 用户手册

> **版本**：SuperStage 26Q1  
> **最后更新**：2026-03-05  
> **引擎版本**：Unreal Engine 5.6 – 5.7  
> **目标平台**：Windows 64-bit（依赖 Win32 原生 Socket API 与第三方 DLL）  
> **适用对象**：灯光设计师、虚拟制作操作员、技术导演

---

## 目录

1. [系统概述](#1-系统概述)
2. [前置准备与环境要求](#2-前置准备与环境要求)
3. [快速上手（5 分钟入门）](#3-快速上手5-分钟入门)
4. [激光 Actor 详解](#4-激光-actor-详解)
   - 4.1 [SuperLaserActor（纹理投影模式）](#41-superlaseractor纹理投影模式)
   - 4.2 [SuperLaserProActor（程序化网格模式）](#42-superlaserproactor程序化网格模式)
5. [SuperLaserProComponent 组件参数详解](#5-superlaserprocomponent-组件参数详解)
6. [网络连接与 Beyond 软件对接](#6-网络连接与-beyond-软件对接)
7. [扫描仪模拟与画质设置](#7-扫描仪模拟与画质设置)
8. [光束检测与渲染参数](#8-光束检测与渲染参数)
9. [Sequencer 集成与激光回放](#9-sequencer-集成与激光回放)
10. [蓝图 API 与 C++ 接口](#10-蓝图-api-与-c-接口)
11. [多设备管理](#11-多设备管理)
12. [性能优化指南](#12-性能优化指南)
13. [常见问题排查（FAQ）](#13-常见问题排查faq)
14. [附录：参数速查表](#14-附录参数速查表)

---

## 1. 系统概述

SuperLaser 是 SuperStage 插件中的 **激光可视化模块**，用于在 Unreal Engine 5 中实时接收并渲染来自 **Pangolin Beyond** 激光控制软件的激光点数据。

### 核心能力

| 能力 | 说明 |
|------|------|
| **实时接收** | 通过 UDP 多播协议从 Beyond 软件实时接收激光点数据 |
| **双模渲染** | 提供"纹理投影"和"程序化网格"两种渲染模式 |
| **扫描仪模拟** | 物理级激光振镜模拟，还原真实激光扫描效果 |
| **多设备支持** | 默认支持 4 台 Beyond 激光设备（Device 0–3），Actor 属性 DeviceID 可配置 1–12，搭配 FLaserSettings::MaxLaserDevices 扩展上限 |
| **Sequencer 回放** | 支持将激光关键帧数据存储到 Level Sequence 中，通过 Sequencer 回放和编辑 |
| **编辑器实时预览** | 无需运行游戏，编辑器视口中即可实时预览激光效果 |
| **碰撞遮挡** | Pro 模式支持激光线与场景物体的碰撞检测，被遮挡时自动截断 |

### 系统架构总览

```
Beyond 激光软件
      │
      │ UDP 多播 (239.255.x.x:5568)
      ▼
┌─────────────────────────────────────────────┐
│           SuperLaser 子系统                    │
│  ┌──────────────┐  ┌──────────────────────┐  │
│  │ 网络接收器    │→ │ 数据处理器（每台设备） │  │
│  │ (UDP Thread)  │  │ 插值→平滑→降采样→光束 │  │
│  └──────────────┘  └──────────┬───────────┘  │
│                               │              │
│          ┌────────────────────┼──────────┐   │
│          ▼                    ▼          │   │
│  ┌──────────────┐   ┌──────────────┐    │   │
│  │ 纹理渲染器    │   │ 点数据引用    │    │   │
│  │ (RenderTarget)│   │ (零拷贝)     │    │   │
│  └──────┬───────┘   └──────┬───────┘    │   │
└─────────┼──────────────────┼────────────┘   │
          ▼                  ▼                │
  SuperLaserActor    SuperLaserProActor       │
  (纹理投影模式)      (程序化网格模式)         │
                                              │
          ┌───────────────────────────────────┘
          ▼
   Sequencer（关键帧录制与回放）
```

---

## 2. 前置准备与环境要求

### 2.1 软件要求

| 软件 | 版本要求 | 用途 |
|------|----------|------|
| Unreal Engine | **5.6 – 5.7** | 渲染引擎（仅支持这两个版本） |
| SuperStage 插件 | 26Q1 及以上 | 激光可视化 |
| Pangolin Beyond | 任意版本 | 激光控制软件 |
| 操作系统 | **Windows 10/11 64-bit** | 依赖 Winsock2、IP_PKTINFO、WSARecvMsg 等 Win32 API |

> **平台限制**：SuperLaser 的网络接收层基于 Win32 原生 Socket API（Winsock2）实现，且依赖 Windows DLL，因此 **仅支持 Windows 64-bit 平台**。macOS/Linux 不可用。

### 2.2 DLL 文件

SuperLaser 需要以下两个 DLL 文件来解析 Beyond 协议数据：

| 文件名 | 用途 |
|--------|------|
| `linetD2_x64.dll` | Beyond 协议数据解析库 |
| `matrix64.dll` | 矩阵运算依赖库（必须先于 linetD2 加载） |

**自动部署**：这两个 DLL 已内置于 `Source/SuperLaser/ThirdParty/BeyondLink/bin/` 目录，并通过 `SuperLaser.Build.cs` 中的 `RuntimeDependencies` 规则 **自动复制** 到插件的二进制输出目录。正常开发和打包流程中 **无需手动拷贝**。

如果自动部署失败，运行时会按以下顺序搜索 DLL：

1. `Plugins/SuperStage/Binaries/Win64/`（插件二进制目录，通过 IPluginManager 定位）
2. `项目目录/Binaries/Win64/`（打包后备选）
3. `FPlatformProcess::GetModulesDirectory()`（模块目录，最后备选）

> **重要提示**：如果 DLL 文件缺失，网络接收功能可以正常启动，但无法解析激光数据。输出日志（Output Log）中会显示：
> - `❌ matrix64.dll not found in any search path` — 矩阵库未找到
> - `❌ Failed to load linetD2_x64.dll` — 解析库加载失败
> - `❌ Failed to get function pointers from linetD2_x64.dll` — DLL 函数导出异常

### 2.3 网络要求

| 项目 | 值 |
|------|-----|
| 协议 | UDP 多播 |
| 端口 | **5568**（Beyond 默认端口） |
| 多播地址范围 | `239.255.0.0` ~ `239.255.{MaxDevices-1}.30`（默认 MaxDevices=4，即 `239.255.0.0` ~ `239.255.3.30`） |
| 多播组数 | MaxDevices × 31（默认 4 × 31 = **124 个**） |
| 防火墙 | 需要放行 UDP 5568 入站 |

**Beyond 和 UE 必须在同一局域网内**，且网络交换机/路由器需要支持多播转发（IGMP）。

---

## 3. 快速上手（5 分钟入门）

以下步骤帮你在最短时间内看到激光效果：

### 第一步：确认 DLL 文件就位

DLL 通常由 Build.cs 自动部署。如果你是第一次使用，先编译一次插件，然后检查 `Plugins/SuperStage/Binaries/Win64/` 目录下是否有 `linetD2_x64.dll` 和 `matrix64.dll`。

### 第二步：启动 Beyond 并开始输出

1. 打开 Pangolin Beyond 软件
2. 选择一个激光效果/图案
3. 确认 Beyond 正在通过网络输出数据（默认端口 5568）

### 第三步：在 UE 中放置激光 Actor

1. 打开 UE 编辑器
2. 在关卡视口中点击 **Place Actors** 面板，或通过菜单 **Edit → Place Actor**，搜索 `SuperLaserProActor`（推荐）或 `SuperLaserActor`
3. 将 Actor 放置到场景中
4. 在 **Details** 面板中设置 **DeviceID** 为 `1`（对应 Beyond 的第一台设备）

### 第四步：观看实时激光

如果网络连接正常且 DLL 就位，你应该立即在编辑器视口中看到实时激光效果。**无需点击 Play**，编辑器模式下即可实时预览。

> **没有看到效果？** 请查看 [第 13 节 常见问题排查](#13-常见问题排查faq)。

---

## 4. 激光 Actor 详解

SuperStage 提供两种激光 Actor，适用于不同场景：

### 4.1 SuperLaserActor（纹理投影模式）

**类名**：`ASuperLaserActor`（继承自 `ASuperBaseActor`）

**工作原理**：将激光点数据通过 `FLaserTextureRenderer` 渲染到一张 HDR 纹理（`UTextureRenderTarget2D`，格式 `PF_FloatRGBA`），然后通过 **静态网格 + 动态材质** 的方式投射到场景中，类似于投影灯效果。渲染采用 **加法混合**（`SE_BLEND_Additive`），支持 HDR 超过 1.0 的颜色叠加。

**适用场景**：
- 需要体积光效果（雾气中的激光光束）
- 需要大气衰减效果
- 对渲染质量要求较高

#### 参数列表

| 参数名 | 显示名 | 范围 | 默认值 | 说明 |
|--------|--------|------|--------|------|
| **DeviceID** | DeviceID | 1–12 | 1 | Beyond 设备编号。Beyond 中的设备 1 对应此处的 1。系统内部会自动转换为 0-based 索引 |
| **LaserIntensity** | LaserIntensity | 1–5000 | 100 | 控制激光的最大亮度强度。值越大光束越亮。推荐范围 50–500 |
| **MaxLaserDistance** | MaxLaserDistance | 100–10000 cm | 2345 | 激光能照射到的最大距离（厘米）。根据场景大小调整 |
| **AtmosphericDensity** | AtmosphericDensity | 0–1 | 0.03 | 大气衰减系数。控制激光随距离变暗的程度。0 = 不衰减，1 = 快速衰减 |
| **FogIntensity** | BeamFogIntensity | 0–100 | 20 | 激光光束中的烟雾效果强度。值越大雾气越浓 |
| **AtmosFogSpeed** | AtmosBeamFogSpeed | 0–100 | 10 | 烟雾流动速度。值越大烟雾飘动越快 |

#### 使用技巧

- **亮度调节**：先将 `LaserIntensity` 调到能看到的程度（通常 100–300），再微调 `AtmosphericDensity`
- **雾效**：配合场景中的 Exponential Height Fog 使用效果更佳
- **可见性**：当 `LaserIntensity` 为 0 时，光束网格会自动隐藏以节省渲染开销

---

### 4.2 SuperLaserProActor（程序化网格模式）

**类名**：`ASuperLaserProActor`（继承自 `ASuperBaseActor`）

**工作原理**：直接读取激光点数据，通过内置的 `USuperLaserProComponent` 为每个可见激光点生成一条 **从 Actor 位置到目标点的程序化网格线**（`UProceduralMeshComponent`），实现逐线渲染。

**适用场景**（推荐）：
- 需要精确的逐线激光渲染
- 需要碰撞遮挡检测（激光线被物体挡住时自动截断）
- 需要更灵活的材质控制（Fresnel、深度衰减、烟雾等）

#### Actor 级参数

| 参数名 | 显示名 | 范围 | 默认值 | 说明 |
|--------|--------|------|--------|------|
| **DeviceID** | DeviceID | 1–12 | 1 | Beyond 设备编号，与 SuperLaserActor 规则相同 |
| **bEnableCollision** | EnableCollision | 开/关 | 关 | 启用碰撞遮挡检测。开启后，激光线会在碰到场景物体时截断 |

#### 编辑器特性

- **箭头指示器**：场景中显示绿色箭头指向 -Z 方向（激光向下投射方向）
- **图标**：编辑器中显示聚光灯图标便于选择
- **实时预览**：编辑器模式下自动更新，无需 Play

> **Pro 模式的所有激光渲染参数在 LaserComponent 组件上设置**，详见下一节。

---

## 5. SuperLaserProComponent 组件参数详解

`SuperLaserProComponent` 是 SuperLaserProActor 的核心渲染组件。选中 Actor 后，在 Details 面板中展开 **LaserComponent** 节点即可看到所有参数。

### 5.1 基础几何参数

| 参数名 | 显示名 | 范围 | 默认值 | 说明 |
|--------|--------|------|--------|------|
| **BeamLength** | BeamLength | 100–50000 cm | — | 激光投射距离。从 Actor 位置到激光线末端的最大长度（厘米）。根据场景大小设置，例如体育馆可设为 30000 |
| **ProjectionAngle** | ProjectionAngle | 1–90° | — | 激光投射角度范围。控制激光点数据中 X/Y 坐标 [-1, 1] 映射到多大的锥角。角度越大，激光扫描范围越广 |
| **LaserWidth** | LaserWidth | 0.1–20 cm | — | 激光线的宽度（粗细）。值越大线条越粗。真实激光通常较细（0.5–2.0） |

### 5.2 材质与视觉效果参数

| 参数名 | 显示名 | 范围 | 默认值 | 说明 |
|--------|--------|------|--------|------|
| **CoreSharpness** | CoreSharpness | 1–10 | — | 核心锐度。控制激光线中心光芯的集中程度。值越大，中心越亮、边缘越暗，看起来越"锐利"。推荐 2–5 |
| **DepthFade** | DepthFade | 0–1 | — | 深度衰减。控制激光线远端变暗的程度。0 = 全程等亮度，1 = 远端完全消失。推荐 0.2–0.5 |
| **Dim** | Dim | 1–10 | — | 自发光强度。控制激光的整体亮度。使用指数曲线，1 = 正常亮度，3 = 较亮，5 = 很亮，10 = 最亮 |
| **OpacityScale** | OpacityScale | 0–1 | — | 整体透明度缩放。0 = 完全透明，1 = 完全不透明。可用于做渐显/渐隐效果 |
| **ComponentDimmer** | ComponentDimmer | 0–1 | 1.0 | 组件级亮度分控。与 Dim 乘法叠加：`最终亮度 = Dim × ComponentDimmer`。用于多模组灯具中独立调节激光组件亮度。0 = 全灭，1 = 全亮 |

### 5.3 烟雾效果参数

| 参数名 | 显示名 | 范围 | 默认值 | 说明 |
|--------|--------|------|--------|------|
| **FogSpeed** | FogSpeed | 0–2 | — | 烟雾流动速度。控制激光光束内部噪声纹理的滚动速度。0 = 静止，2 = 快速流动 |
| **FogInfluence** | FogInfluence | 0–1 | — | 烟雾影响强度。控制烟雾纹理对激光线外观的影响程度。0 = 无烟雾（纯激光线），1 = 完全受烟雾控制（类似真实烟雾中的激光效果） |

### 5.4 光斑参数

| 参数名 | 显示名 | 范围 | 默认值 | 说明 |
|--------|--------|------|--------|------|
| **SpotDimmer** | SpotDimmer | 0–5 | — | 光斑亮度。激光线打在场景物体表面时产生的光斑亮度。0 = 无光斑，5 = 非常亮的光斑 |

### 5.5 碰撞参数

| 参数名 | 显示名 | 范围 | 默认值 | 说明 |
|--------|--------|------|--------|------|
| **bEnableCollision** | EnableCollision | 开/关 | 关 | 碰撞遮挡检测开关。开启后，每条激光线会进行射线检测，碰到物体时自动截断。**注意**：开启碰撞会增加性能开销，场景中有大量激光线时请谨慎使用 |

> **补充说明**：`bEnableCollision` 同时存在于 `ASuperLaserProActor`（Actor 级）和 `USuperLaserProComponent`（组件级）上。Actor 的属性会同步到其内部的 LaserComponent，两者保持一致。在 Details 面板中设置任一处即可。

### 5.6 参数调节建议

**追求真实感**的设置参考：
```
BeamLength      = 15000 (150米)
ProjectionAngle = 30
LaserWidth      = 1.0
CoreSharpness   = 3.0
DepthFade       = 0.3
Dim             = 4.0
OpacityScale    = 0.8
FogSpeed        = 0.3
FogInfluence    = 0.4
SpotDimmer      = 1.5
bEnableCollision = 开
```

**追求性能**的设置参考：
```
BeamLength      = 10000 (100米)
ProjectionAngle = 45
LaserWidth      = 0.5
CoreSharpness   = 2.0
DepthFade       = 0.0
Dim             = 3.0
OpacityScale    = 1.0
FogSpeed        = 0.0
FogInfluence    = 0.0
SpotDimmer      = 0.0
bEnableCollision = 关
```

---

## 6. 网络连接与 Beyond 软件对接

### 6.1 Beyond 端配置

1. 打开 Pangolin Beyond 软件
2. 确认你的激光效果正在播放
3. Beyond 会自动通过 UDP 多播在端口 **5568** 上广播激光数据
4. 默认情况下无需额外配置，Beyond 的网络输出是默认开启的

### 6.2 网络协议说明

| 项目 | 值 |
|------|-----|
| 传输协议 | UDP 多播 |
| 默认端口 | 5568 |
| 多播地址格式 | `239.255.{DeviceID}.{SubnetID}` |
| DeviceID 范围 | 0 ~ MaxLaserDevices-1（默认 0–3，即 4 台设备） |
| SubnetID 范围 | 0–30（每台设备 31 个子网） |
| 总多播组数 | MaxLaserDevices × 31（默认 4 × 31 = **124 个**多播地址） |

系统启动时会自动加入所有多播组以接收数据。多播组数量取决于 `FLaserSettings::MaxLaserDevices` 配置（默认 4）。

### 6.3 DeviceID 映射关系

Beyond 和 UE 中的 DeviceID 存在 **偏移关系**：

| Beyond 设备编号 | UE Actor 中填写的 DeviceID | 系统内部索引 |
|:---:|:---:|:---:|
| 设备 1 | **1** | 0 |
| 设备 2 | **2** | 1 |
| 设备 3 | **3** | 2 |
| 设备 4 | **4** | 3 |

> 简单来说：**Actor 的 DeviceID 属性直接填 Beyond 中显示的设备编号即可**，系统内部会自动做 -1 转换。

### 6.4 防火墙设置

如果激光数据无法接收，请检查 Windows 防火墙：

1. 打开 **Windows Defender 防火墙** → **高级设置**
2. 点击 **入站规则** → **新建规则**
3. 选择 **端口** → **UDP** → 填入 `5568`
4. 选择 **允许连接**
5. 命名为 `SuperLaser Beyond`

### 6.5 多网卡环境

如果你的电脑有多块网卡（如 WiFi + 有线），系统默认绑定到 `0.0.0.0`（所有网卡）。通常无需额外配置。

---

## 7. 扫描仪模拟与画质设置

SuperLaser 内置了物理级的 **激光振镜模拟器**，可以还原真实激光扫描仪的物理特性。这些设置通过 `FLaserSettings` 结构体全局配置，可通过 `USuperLaserSubsystem::SetSettings()` 在 C++ 或蓝图中修改。所有活动设备共享同一套设置。

### 7.1 扫描仪模拟参数

| 参数名 | 范围 | 默认值 | 说明 |
|--------|------|--------|------|
| **bScannerSimulation** | 开/关 | 开 | 总开关。关闭后激光点直接显示，无物理模拟 |
| **SampleCount** | 1–16 | 8 | 点插值采样数量。两个相邻激光点之间插入多少中间点。值越大线条越平滑，但计算量越大 |
| **EdgeFade** | 0–1 | 0.1 | 边缘淡化因子。模拟真实激光在高速拐角处变暗的物理现象。0 = 不淡化，1 = 强淡化 |
| **VelocitySmoothing** | 0–1 | 0.83 | 速度平滑因子。模拟振镜惯性。0 = 无惯性（瞬间到位），1 = 最大惯性（缓慢追踪目标）。推荐 0.7–0.9 |

**扫描仪模拟的工作原理**：

1. **点插值**：在相邻两个激光点之间生成 `SampleCount` 个中间点，使线条更平滑
2. **速度平滑**：模拟振镜加速度限制，振镜不能瞬间从一个位置跳到另一个位置
3. **边缘淡化**：振镜高速移动时（大角度拐弯），激光强度会降低，产生边角变暗效果

### 7.2 画质级别

| 质量级别 | 降采样倍数 | 说明 |
|----------|:---:|------|
| **Low** | 8x | 性能优先。激光点数减少到 1/8。适合大量设备同时预览 |
| **Medium** | 4x | 平衡模式。点数减少到 1/4。大多数场景推荐 |
| **High** | 2x | 高质量。点数减少到 1/2。默认级别 |
| **Ultra** | 1x（无降采样） | 最高质量。不做任何降采样。仅在需要逐点精度时使用 |

### 7.3 纹理分辨率（仅影响纹理投影模式）

| 参数名 | 可选值 | 默认值 | 说明 |
|--------|--------|--------|------|
| **TextureSize** | 256 / 512 / 1024 / 2048 | 512 | 激光纹理的渲染分辨率。分辨率越高激光图案越清晰，但占用显存越多 |
| **bEnableMipmaps** | 开/关 | 关 | 是否为纹理生成 Mipmap。关闭可以提升性能 |

---

## 8. 光束检测与渲染参数

### 8.1 光束检测

光束检测用于识别激光数据中的 **静态光束点**（如激光笔直射向一个方向时，多个连续点位于同一位置）。

| 参数名 | 范围 | 默认值 | 说明 |
|--------|------|--------|------|
| **BeamRepeatThreshold** | ≥1 | 8 | 光束重复点阈值。连续相同位置的点数超过此值时，识别为光束。值越小越敏感，值越大需要更多重复点才能触发 |
| **BeamIntensityCount** | ≥1 | 160 | 高强度光束额外生成的点数。检测到光束后，在该位置堆叠这么多个点来增强亮度效果 |

### 8.2 渲染参数（纹理投影模式）

| 参数名 | 范围 | 默认值 | 说明 |
|--------|------|--------|------|
| **LineWidth** | 0.01–10 | 0.1 | 激光点在纹理上的绘制大小（像素）。值越大点越粗 |
| **MaxBeamBrush** | 0.1–10 | 2.0 | 最大光束画刷大小。控制光束区域的绘制范围 |
| **bEnableBeamBrush** | 开/关 | 关 | 光束画刷开关。开启后，光束位置的重复点会被移除并用更大的画刷绘制。可以产生更集中的光斑效果 |

---

## 9. Sequencer 集成与激光回放

SuperLaser 提供了完整的 **Sequencer 轨道基础设施**，可将激光关键帧数据存储到 Level Sequence 中并回放。

### 9.1 轨道架构

```
Level Sequence
 └─ UMovieSceneSuperLaserTrack ("Super Laser")
     └─ UMovieSceneSuperLaserSection
         ├─ DeviceID: 0（内部索引，0-based）
         ├─ Keyframes: TArray<FLaserKeyframe>
         │    └─ 每帧: Time + TArray<FCompressedLaserPoint>
         └─ bIsRecording: 录制/播放互斥标志
```

- **Track**（`UMovieSceneSuperLaserTrack`）：继承 `UMovieSceneNameableTrack`，Sequencer 中显示名为 **"Super Laser"**
- **Section**（`UMovieSceneSuperLaserSection`）：存储 DeviceID 和关键帧数组，每帧记录一组压缩后的激光点数据
- **Template**（`FMovieSceneSuperLaserTemplate`）：评估模板，播放时生成执行 Token 调用 `EvaluateAndSetLaserData`

### 9.2 数据压缩

关键帧数据采用 `FCompressedLaserPoint` 自动量化压缩：

| 数据 | 原始精度 | 压缩精度 | 占用空间 |
|------|----------|----------|----------|
| X/Y 坐标 | float32（4 字节） | int16（2 字节） | 精度损失 < 0.003% |
| R/G/B 颜色 | float32（4 字节） | uint8（1 字节） | 精度损失 < 0.4% |
| Focus + Z 标志 | float32 × 2（8 字节） | uint8（1 字节） | Focus 精度 7bit，Z 为布尔值 |
| **每点总计** | **28 字节** | **8 字节** | **压缩比 71.4%** |

录制文件仅为原始数据的约 **29%** 大小，但视觉效果几乎无差异。

### 9.3 播放机制

1. 打开包含激光轨道的 **Level Sequence** 资产
2. 确保场景中有对应 DeviceID 的 SuperLaserActor 或 SuperLaserProActor
3. 在 Sequencer 中按 **播放** 按钮
4. 激光数据会通过 `USuperLaserSubsystem::SetDevicePoints()` 自动分发到场景中的激光 Actor

**插值策略**：播放时使用 **常量插值**（阶梯式），而非线性插值：
- 在两个关键帧之间，显示的始终是 **前一个关键帧** 的数据（二分查找定位）
- 这符合激光的物理特性——激光图案是瞬间切换的，不存在"渐变"过程
- 不会出现两帧数据"混合"导致的闪烁问题

### 9.4 录制与播放互斥

- **录制中**（`bIsRecording = true`）：Template 评估阶段会跳过该 Section，防止新旧数据冲突
- **播放中**：实时网络数据仍然在接收和处理，但 Sequencer 的数据会通过 `SetDevicePoints` 覆盖显示

### 9.5 C++ 录制接口

通过 `UMovieSceneSuperLaserSection` 的 API 手动录制激光数据：

```cpp
// 获取或创建 Section
UMovieSceneSuperLaserSection* Section = ...;
Section->SetIsRecording(true);

// 每帧添加关键帧（自动压缩）
Section->AddKeyframe(CurrentTime, LaserPoints);
Section->ExpandToFrame(CurrentFrame);

// 录制完成
Section->SetIsRecording(false);
```

> **注意**：Section 中的 DeviceID 使用 **0-based 内部索引**（0 = Beyond 设备 1），与 Actor 属性的 1-based DeviceID 不同。

---

## 10. 蓝图 API 与 C++ 接口

### 10.1 USuperLaserSubsystem（引擎子系统）

`USuperLaserSubsystem` 是激光系统的核心管理器，作为 `UEngineSubsystem` 全局唯一。

**获取方式**（C++）：
```cpp
USuperLaserSubsystem* LaserSys = GEngine->GetEngineSubsystem<USuperLaserSubsystem>();
```

**主要接口**：

| 方法 | 说明 |
|------|------|
| `StartReceiving()` | BlueprintCallable。启动网络接收（自动加入多播组、加载 DLL） |
| `StopReceiving()` | BlueprintCallable。停止网络接收并清理资源 |
| `IsReceiving()` | BlueprintPure。查询网络接收是否运行中 |
| `GetDeviceTexture(int32 DeviceID)` | BlueprintCallable。获取指定设备的 RenderTarget（纹理投影模式） |
| `GetDevicePoints(int32 DeviceID)` | BlueprintCallable。获取设备点数据副本 |
| `GetDevicePointsRef(int32 DeviceID)` | C++ only。获取设备点数据 **const 引用**（零拷贝，高性能） |
| `SetDevicePoints(int32 DeviceID, Points)` | 设置设备点数据（用于 Sequencer 回放） |
| `GetSettings()` / `SetSettings()` | BlueprintCallable。读取/修改全局 `FLaserSettings` 配置 |
| `SetRecording(bool)` | 设置录制模式标志（录制时阻止 Sequencer 播放） |
| `IsRecording()` | 查询录制模式状态 |
| `GetNetworkStats()` | BlueprintPure。获取网络接收统计（`FLaserNetworkStats`） |
| `ClearDeviceResources(int32 DeviceID)` | BlueprintCallable。释放指定设备的处理器和渲染器 |
| `ClearAllDeviceResources()` | BlueprintCallable。释放所有设备资源 |

### 10.2 ASuperLaserProActor 蓝图接口

| 方法 | 类别 | 说明 |
|------|------|------|
| `SetLaserEnabled(bool)` | BlueprintCallable | 设置激光可见性开关 |

### 10.3 USuperLaserProComponent 蓝图接口

| 方法 | 类别 | 说明 |
|------|------|------|
| `SetLaserPoints(TArray<FLaserPoint>)` | BlueprintCallable | 设置激光点数据 |
| `RebuildMesh()` | BlueprintCallable | 强制重建程序化网格 |
| `SetLaserVisibility(bool)` | BlueprintCallable | 设置激光可见性 |
| `SetComponentDimmer(float)` | BlueprintCallable | 设置组件级亮度分控（与 Dim 乘法叠加） |

---

## 11. 多设备管理

### 11.1 多台 Beyond 设备

SuperLaser 默认支持 **4 台 Beyond 设备**（内部索引 0–3），通过修改 `FLaserSettings::MaxLaserDevices` 可扩展至最多 12 台。每台设备独立拥有：
- 一个数据处理器（Processor）
- 一个纹理渲染器（Renderer）
- 独立的点数据缓冲

### 11.2 场景中放置多个激光 Actor

你可以在场景中放置任意数量的激光 Actor，它们可以指向 **同一个或不同的** DeviceID：

| 场景 | 配置方式 |
|------|----------|
| 4 台不同的激光设备 | 放 4 个 Actor，分别设 DeviceID = 1, 2, 3, 4（Actor 属性支持 1–12） |
| 同一台设备的多个视角 | 放多个 Actor，都设置相同的 DeviceID |
| 混合使用两种模式 | 同一设备可同时用 SuperLaserActor 和 SuperLaserProActor 显示 |

### 11.3 资源释放

系统会自动按需创建每个设备的处理器和渲染器。如果不再使用某个设备，可以通过蓝图调用以下函数释放内存：

- **`ClearDeviceResources(DeviceID)`**：释放指定设备的处理器和渲染器（BlueprintCallable）
- **`ClearAllDeviceResources()`**：释放所有设备资源（BlueprintCallable）

---

## 12. 性能优化指南

### 12.1 性能影响因素

| 因素 | 影响程度 | 优化建议 |
|------|:---:|------|
| 画质级别 | ★★★★★ | 从 Ultra 降到 Medium 可减少 75% 的点处理量 |
| 扫描仪模拟 | ★★★★ | 关闭 bScannerSimulation 可大幅降低 CPU 开销 |
| 碰撞检测 | ★★★★ | 关闭 bEnableCollision 可避免逐线射线检测 |
| SampleCount | ★★★ | 从 16 降到 4 可减少 75% 的插值计算 |
| 纹理分辨率 | ★★★ | 从 2048 降到 512 可减少 93% 的显存占用 |
| 烟雾效果 | ★★ | FogInfluence 设为 0 可跳过噪声采样计算 |
| 光斑效果 | ★ | SpotDimmer 设为 0 可简化着色计算 |

### 12.2 推荐配置方案

**高性能场景**（4+ 台设备，大量激光线）：
- 画质级别：**Low** 或 **Medium**
- 扫描仪模拟：**关闭**
- 碰撞检测：**关闭**
- 纹理分辨率：**256** 或 **512**
- 烟雾/光斑：**关闭**

**高质量场景**（1–2 台设备，最终渲染）：
- 画质级别：**High** 或 **Ultra**
- 扫描仪模拟：**开启**，SampleCount = 8–16
- 碰撞检测：**开启**
- 纹理分辨率：**1024** 或 **2048**
- 烟雾/光斑：按需开启

### 12.3 性能监控

在 UE 编辑器的 **Stat** 系统中，SuperLaser 注册了以下性能计数器：

- **Stat SuperLaser** → `STAT_SuperLaserTick`：显示每帧子系统处理耗时

在 **Output Log** 中可通过以下日志类别过滤信息：

| 日志类别 | 模块 | 说明 |
|----------|------|------|
| `LogSuperLaser` | 子系统/模块入口 | 核心系统状态、初始化、DLL 验证 |
| `LogLaserNetwork` | 网络接收器 | 多播组加入/离开、DLL 加载、数据包解析、网络统计 |
| `LogLaserProcessor` | 数据处理器 | 扫描仪模拟、插值、降采样、光束检测 |
| `LogLaserRenderer` | 纹理渲染器 | RenderTarget 创建、Canvas 绘制、纹理清空 |

---

## 13. 常见问题排查（FAQ）

### Q1：放置了 Actor 但看不到任何激光效果

**排查步骤**：

1. **检查 DLL 文件**：查看 Output Log 是否有 `❌ matrix64.dll not found` 或 `❌ Failed to load linetD2_x64.dll` 错误
2. **检查 Beyond 是否正在输出**：在 Beyond 中确认激光效果正在播放
3. **检查网络连接**：Beyond 和 UE 必须在同一局域网内
4. **检查防火墙**：确认 UDP 5568 端口已放行
5. **检查 DeviceID**：确认 Actor 上的 DeviceID 与 Beyond 中的设备编号匹配（Actor 填 1 对应 Beyond 设备 1）
6. **检查亮度参数**：SuperLaserActor 的 `LaserIntensity` 不能为 0；SuperLaserProActor 的 `Dim` 和 `ComponentDimmer` 不能为 0

### Q2：激光效果有明显延迟

**可能原因及解决方案**：

- **网络延迟**：确保 Beyond 和 UE 在同一台机器或千兆局域网内
- **画质过高**：将质量级别从 Ultra 降到 High 或 Medium
- **扫描仪模拟 SampleCount 过大**：降低 SampleCount（推荐 4–8）
- **CPU 过载**：检查 Stat SuperLaser 中的 Tick 耗时

### Q3：编辑器模式下无法预览

SuperLaser 支持编辑器模式实时预览。如果无法预览：

1. 确认 Actor 的 `ShouldTickIfViewportsOnly` 已启用（默认启用）
2. 确认编辑器视口的 **Realtime** 按钮已开启（视口左上角的时钟图标）

### Q4：Sequencer 中没有激光轨道数据

**排查步骤**：

1. 确认已通过 C++ 代码正确创建了 `UMovieSceneSuperLaserTrack` 和 `UMovieSceneSuperLaserSection`
2. 确认 Section 的 `DeviceID` 与场景中 Actor 的设备对应（Section 使用 0-based 索引）
3. 确认 `AddKeyframe()` 被正确调用且关键帧数据非空
4. 查看 Output Log 中 `LogSuperLaser` 日志确认子系统状态正常

### Q5：Sequencer 播放时激光不显示

1. 确认场景中有 SuperLaserActor 或 SuperLaserProActor，且 DeviceID 与录制时使用的设备匹配
2. 确认 Level Sequence 中的激光轨道包含关键帧数据
3. 确认没有处于录制模式（录制中会禁止播放）

### Q6：SuperLaserActor 和 SuperLaserProActor 该选哪个？

| 对比维度 | SuperLaserActor | SuperLaserProActor |
|----------|:---:|:---:|
| 渲染方式 | 纹理投影 | 程序化网格逐线绘制 |
| 体积光/雾气效果 | ★★★★★ | ★★★★ |
| 单线精度 | ★★★ | ★★★★★ |
| 碰撞遮挡 | 不支持 | ★★★★★ |
| 材质可控性 | ★★★ | ★★★★★ |
| 性能开销 | 较低 | 中等 |

**推荐**：大多数场景使用 **SuperLaserProActor**，它提供更精确的激光渲染和碰撞遮挡能力。

### Q7：多台电脑运行 UE 能同时接收同一台 Beyond 的数据吗？

可以。由于使用 UDP 多播协议，同一局域网内的多台电脑可以同时接收同一台 Beyond 设备的激光数据，实现多机同步预览。

---

## 14. 附录：参数速查表

### A. SuperLaserActor 参数

| 参数 | 类型 | 范围 | 默认 | 分类 |
|------|------|------|------|------|
| DeviceID | int32 | 1–12 | 1 | SuperLaser |
| LaserIntensity | float | 1–5000 | 100 | SuperLaser |
| MaxLaserDistance | float | 100–10000 cm | 2345 | SuperLaser |
| AtmosphericDensity | float | 0–1 | 0.03 | SuperLaser |
| FogIntensity | float | 0–100 | 20 | SuperLaser |
| AtmosFogSpeed | float | 0–100 | 10 | SuperLaser |

### B. SuperLaserProActor 参数

| 参数 | 类型 | 范围 | 默认 | 分类 |
|------|------|------|------|------|
| DeviceID | int32 | 1–12 | 1 | A.DefaultParameter |
| bEnableCollision | bool | 开/关 | 关 | A.DefaultParameter |

### C. SuperLaserProComponent 参数

| 参数 | 类型 | 范围 | 默认 | 分类 |
|------|------|------|------|------|
| BeamLength | float | 100–50000 cm | — | A.DefaultParameter |
| ProjectionAngle | float | 1–90° | — | A.DefaultParameter |
| LaserWidth | float | 0.1–20 cm | — | A.DefaultParameter |
| CoreSharpness | float | 1–10 | — | A.DefaultParameter |
| DepthFade | float | 0–1 | — | A.DefaultParameter |
| Dim | float | 1–10 | — | A.DefaultParameter |
| OpacityScale | float | 0–1 | — | A.DefaultParameter |
| ComponentDimmer | float | 0–1 | 1.0 | A.DefaultParameter |
| FogSpeed | float | 0–2 | — | A.DefaultParameter |
| FogInfluence | float | 0–1 | — | A.DefaultParameter |
| SpotDimmer | float | 0–5 | — | A.DefaultParameter |
| bEnableCollision | bool | 开/关 | 关 | A.DefaultParameter |

### D. 系统全局设置 (FLaserSettings)

| 参数 | 类型 | 范围 | 默认 | 分类 |
|------|------|------|------|------|
| NetworkPort | int32 | — | 5568 | Network |
| MaxLaserDevices | int32 | — | 4 | Network |
| TextureSize | int32 | 256/512/1024/2048 | 512 | Rendering |
| bEnableMipmaps | bool | 开/关 | 关 | Rendering |
| LineWidth | float | 0.01–10 | 0.1 | Rendering |
| MaxBeamBrush | float | 0.1–10 | 2.0 | Rendering |
| bEnableBeamBrush | bool | 开/关 | 关 | Rendering |
| bScannerSimulation | bool | 开/关 | 开 | Scanner Simulation |
| SampleCount | int32 | 1–16 | 8 | Scanner Simulation |
| EdgeFade | float | 0–1 | 0.1 | Scanner Simulation |
| VelocitySmoothing | float | 0–1 | 0.83 | Scanner Simulation |
| BeamRepeatThreshold | int32 | ≥1 | 8 | Beam Detection |
| BeamIntensityCount | int32 | ≥1 | 160 | Beam Detection |
| LaserQuality | enum | Low/Medium/High/Ultra | High | Quality |
| RecordingDownsample | int32 | 1–10 | 1 | Recording |
| bRemoveBlankPointsInRecording | bool | 开/关 | 开 | Recording |
| bEnableParallelProcessing | bool | 开/关 | 关 | Performance |

### E. Sequencer 轨道类型

| 类 | 说明 |
|------|------|
| `UMovieSceneSuperLaserTrack` | Sequencer 轨道，显示名 "Super Laser" |
| `UMovieSceneSuperLaserSection` | 关键帧数据存储，DeviceID (0-based) + TArray\<FLaserKeyframe\> |
| `FMovieSceneSuperLaserTemplate` | 评估模板，播放时生成执行 Token |

---

> **技术支持**：如遇到本手册未覆盖的问题，请查看 UE 编辑器的 Output Log，使用以下日志类别过滤：`LogSuperLaser`、`LogLaserNetwork`、`LogLaserProcessor`、`LogLaserRenderer`。它们通常包含详细的错误原因和解决线索。

# SuperStage & SuperDMX 插件架构总览

> **文档版本**: 1.0  
> **适用版本**: SuperStage Plugin 2025/2026  
> **目标读者**: 蓝图开发者 / C++ 插件集成开发者（无源码修改权限）  
> **最后更新**: 2026-03

---

## 1. 插件概述

SuperStage 是一套面向 **Unreal Engine 5** 的专业舞台灯光与演出可视化插件体系，由两个紧密协作的运行时模块组成：

| 模块 | 导出宏 | 职责 |
|------|--------|------|
| **SuperDMX** | `SUPERDMX_API` | DMX 协议通信层：Art-Net/sACN 收发、数据缓存、Sequencer 集成 |
| **SuperStage** | `SUPERSTAGE_API` | 舞台设备层：灯具 Actor、灯光组件、灯库、机械控制、激光、NDI、VFX |

两者的依赖关系为 **SuperStage → SuperDMX**（SuperStage 依赖 SuperDMX，反向无依赖）。

### 1.1 核心能力矩阵

| 能力域 | 说明 |
|--------|------|
| **DMX 通信** | Art-Net 收/发，sACN 多播接收，多 Universe 并行，线程安全缓存 |
| **灯具模拟** | 电脑灯（Beam/Spot/Wash/Profile）、LED 矩阵灯、灯带、投影仪 |
| **运动控制** | Pan/Tilt 旋转、无极旋转、升降机械、轨道机械（Spline 样条） |
| **颜色系统** | RGB/RGBW/HSV/CMY/色温/冷暖混色/颜色轮/多通道动态混色 |
| **光学系统** | Zoom/Focus/Frost/Iris、图案轮（Gobo×3）、棱镜（Prism×3）、切割（Shaper） |
| **效果系统** | LED 效果 LUT（10 种内置效果）、Niagara 粒子 VFX |
| **视频集成** | NDI 视频流接收与屏幕显示 |
| **激光集成** | Beyond 激光设备纹理与点数据可视化 |
| **Sequencer** | DMX Track/Section 录制与回放 |
| **灯库系统** | GDTF/MA2/MA3 兼容的 DataAsset 灯库配置 |

---

## 2. 系统架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        外部控台 / 软件                               │
│                   (grandMA, Hog, ETC Eos, etc.)                     │
└───────────────┬─────────────────────────────────┬───────────────────┘
                │ Art-Net / sACN                  │ NDI / Beyond
                ▼                                 ▼
┌───────────────────────────┐   ┌─────────────────────────────────────┐
│      SuperDMX 模块         │   │          第三方子系统                 │
│  ┌─────────────────────┐  │   │  SuperNDISubsystem                  │
│  │ USuperDMXSubsystem   │  │   │  SuperLaserSubsystem                │
│  │  ├─ ApplyConfig()   │  │   └─────────────────────────────────────┘
│  │  ├─ GetDMXBuffer()  │  │                 │
│  │  ├─ GetDMXValue()   │  │                 │
│  │  ├─ SendDMXBuffer() │  │                 │
│  │  └─ 线程安全缓存     │  │                 │
│  └─────────────────────┘  │                 │
│  ┌─────────────────────┐  │                 │
│  │ Sequencer 集成       │  │                 │
│  │  UMovieSceneSuperDMX │  │                 │
│  │  Track / Section     │  │                 │
│  └─────────────────────┘  │                 │
└───────────┬───────────────┘                 │
            │ GetDMXBuffer / SendDMXBuffer    │
            ▼                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        SuperStage 模块                               │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    灯库系统 (Data Layer)                      │   │
│  │  USuperFixtureLibrary → FSuperDMXModuleInstance              │   │
│  │                       → FSuperDMXAttributeDef               │   │
│  │                       → FSubAttribute / FChannelSet         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                     │
│                              ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   Actor 继承体系                              │   │
│  │                                                             │   │
│  │  AActor                                                     │   │
│  │    └─ ASuperBaseActor          (元数据 / 方向预览)            │   │
│  │        ├─ ASuperDmxActorBase   (DMX 读取核心)                │   │
│  │        │   ├─ ASuperLightBase  (Pan/Tilt 运动控制)           │   │
│  │        │   │   ├─ ASuperStageLight     (全功能电脑灯)        │   │
│  │        │   │   ├─ ASuperStageVFXActor  (Niagara VFX)        │   │
│  │        │   │   └─ ASuperLiftMatrix     (升降矩阵)            │   │
│  │        │   ├─ ASuperLiftingMachinery   (升降机械 6轴)        │   │
│  │        │   ├─ ASuperRailMachinery      (轨道机械 7轴)        │   │
│  │        │   ├─ ASuperLightStripEffect   (LED灯带效果)         │   │
│  │        │   └─ ASuperDMXCamera          (DMX摄像机)           │   │
│  │        ├─ ASuperLaserActor       (激光纹理显示)              │   │
│  │        ├─ ASuperLaserProActor    (激光点数据显示)             │   │
│  │        ├─ ASuperNDIScreen        (NDI视频屏幕)               │   │
│  │        ├─ ASuperProjector        (投影Mapping)              │   │
│  │        ├─ ASuperTruss            (桁架)                     │   │
│  │        ├─ ASuperScaffold         (脚手架)                    │   │
│  │        ├─ ASuperCurvedScaffold   (弧形脚手架)                │   │
│  │        └─ ASuperDrape            (幕布)                     │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                     │
│                              ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   组件体系 (Components)                       │   │
│  │                                                             │   │
│  │  USceneComponent                                            │   │
│  │    └─ USuperLightingComponent   (光束基础组件)                │   │
│  │        ├─ USuperSpotComponent   (SpotLight 辅光)             │   │
│  │        │   └─ USuperBeamComponent   (Beam 光束)              │   │
│  │        │       └─ USuperCuttingComponent (切割/Shaper)       │   │
│  │        └─ USuperRectComponent   (RectLight 面光)             │   │
│  │    └─ USuperEffectComponent     (LED效果组件)                 │   │
│  │    └─ USuperMatrixComponent     (LED矩阵像素组件)            │   │
│  │    └─ USuperLiftComponent       (升降组件)                   │   │
│  │    └─ USuperLaserProComponent   (激光Pro渲染组件)            │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. 模块详解

### 3.1 SuperDMX 模块

SuperDMX 是底层通信引擎，以 `UEngineSubsystem` 形式提供**全局单例**访问，在引擎初始化时自动创建，所有模块/关卡均可共享。

#### 3.1.1 USuperDMXSubsystem — DMX 通信子系统

**获取方式**（C++）：
```cpp
USuperDMXSubsystem* DMX = GEngine->GetEngineSubsystem<USuperDMXSubsystem>();
```

**核心职责**：
- **网络 I/O**：通过 `ApplyConfig()` 配置 Art-Net/sACN 端点，自动监听并解析 DMX 数据包
- **数据缓存**：内部维护 `TMap<int32, TArray<uint8>>` 按 Universe 缓存最新 512 字节 DMX 帧
- **线程安全**：接收线程与游戏线程通过 `FCriticalSection` 隔离，读写操作线程安全
- **数据输出**：通过 `SendDMXBuffer()` 向网络发送 DMX 数据（Sequencer 回放、程序化控制）
- **诊断**：`GetSecondsSinceLastInput()` 跟踪信号活跃度

**关键数据类型**：

| 类型 | 说明 |
|------|------|
| `ESuperDMXProtocol` | 协议枚举：`ArtNet` |
| `FSuperDMXEndpoint` | 网络端点配置（LocalIp, RemoteIp, Port, StartUniverse） |
| `FSuperDMXIOConfig` | 完整 I/O 配置（Protocol + Input/Output 端点） |

#### 3.1.2 Sequencer 集成

| 类 | 说明 |
|----|------|
| `UMovieSceneSuperDMXTrack` | Sequencer 轨道，继承自 `UMovieSceneNameableTrack`，包含多个 Section |
| `UMovieSceneSuperDMXSection` | Sequencer 段落，每个 Section 对应一个 Universe，内含 512 通道的动画曲线 |

Sequencer 回放时通过 `USuperDMXSubsystem::SendDMXBuffer()` 输出 DMX 数据，实现 **时间轴驱动的 DMX 编程**。

---

### 3.2 SuperStage 模块

SuperStage 构建在 SuperDMX 之上，提供完整的舞台设备建模与控制能力。

#### 3.2.1 灯库系统（Fixture Library）

灯库是连接 DMX 通道号与设备功能的桥梁。每个灯具 Actor 引用一个 `USuperFixtureLibrary` 数据资产，该资产定义了灯具的所有 DMX 通道布局。

**数据层级结构**：

```
USuperFixtureLibrary (DataAsset)
 ├── FixtureName          "Acme XP-380 Beam 17CH"
 ├── Manufacturer         "Acme"
 ├── Source               EFixtureLibrarySource (Custom/GDTF/MA2/MA3)
 ├── PowerConsumption     540.0W
 ├── Weight               21.2kg
 └── Modules[]            TArray<FSuperDMXModuleInstance>
      ├── [0] ModuleName = "Main"
      │    ├── Patch = 0     (相对 StartAddress 的偏移)
      │    └── AttributeDefs[]
      │         ├── [0] AttribName="Dimmer", Coarse=1, Fine=2
      │         │        └── SubAttributes[] → FSubAttribute
      │         │             └── ChannelSets[] → FChannelSet
      │         ├── [1] AttribName="Pan", Coarse=3, Fine=4
      │         ├── [2] AttribName="Tilt", Coarse=5, Fine=6
      │         └── ...
      ├── [1] ModuleName = "Pixel1"  (矩阵灯的子模块)
      │    ├── Patch = 17
      │    └── AttributeDefs[] ...
      └── ...
```

**核心结构体一览**：

| 结构体 | 说明 |
|--------|------|
| `FSuperDMXModuleInstance` | 模块实例（矩阵灯的每个像素/灯头为一个模块） |
| `FSuperDMXAttributeDef` | 属性定义：名称、分类、Coarse/Fine/Ultra 通道偏移、默认值 |
| `FSubAttribute` | 子属性/功能（对应 MA2 ChannelFunction），含频闪模式、旋转模式 |
| `FChannelSet` | 槽位集合（对应 MA2 ChannelSet），含图案纹理、颜色、棱镜参数 |
| `FSuperDMXAttribute` | 属性查询键（InstanceIndex + AttribName + ChannelType） |
| `FSuperDMXFixture` | 灯具 DMX 地址配置（Universe + StartAddress） |

**属性分类枚举 `EDMXAttributeCategory`**：

| 值 | 含义 | 典型属性 |
|----|------|----------|
| `Dimmer` | 亮度 | Dimmer, MasterDimmer |
| `Position` | 位置 | Pan, Tilt, PanRot |
| `Gobo` | 图案 | Gobo1, Gobo2, GoboRot |
| `Color` | 颜色 | ColorWheel, Red, Green, Blue |
| `Beam` | 光束 | Zoom, Iris |
| `Focus` | 聚焦 | Focus |
| `Control` | 控制 | Reset, Lamp |
| `Shapers` | 切割 | ShapA1, ShapB1 |
| `Strobe` | 频闪 | Strobe |
| `Prism` | 棱镜 | Prism1, PrismRot |
| `Frost` | 雾化 | Frost |
| `Effects` | 效果 | Effect |
| `Other` | 其他 | — |

#### 3.2.2 Actor 继承体系

##### 第一层：ASuperBaseActor — 基础资产层

所有 SuperStage Actor 的根基类，提供：
- **元数据管理** (`FAssetMetaData`)：UUID、分组、制造商、DMX 模式名、缩略图
- **方向预览**：编辑器内通过箭头组件可视化 +X 前向和 +Z 上方向
- **根组件** `SceneBase`：所有子组件的挂载点

##### 第二层：ASuperDmxActorBase — DMX 数据读取层

**核心职责**：从 `USuperDMXSubsystem` 读取 DMX 数据并转换为可用值。

**关键属性**：
- `SuperDMXFixture`（`FSuperDMXFixture`）：Universe + StartAddress
- `FixtureLibrary`（`USuperFixtureLibrary*`）：通道配置引用

**DMX 值读取 API**（按使用频率排序）：

| 函数 | 返回 | 说明 |
|------|------|------|
| `GetSuperDmxAttributeValue()` | `float [0,1]` | 读取属性值并**归一化**到 0-1 范围 |
| `GetSuperDmxAttributeValueNoConversion()` | `float` | 读取属性值，**不做**归一化 |
| `GetSuperDMXColorValue()` | `FLinearColor` | 读取 RGB 三通道并合成颜色 |
| `GetSuperDmxAttributeRawValue()` | `int32 [0,255]` | 读取原始 DMX 值 |
| `GetAttributeRaw8ByIndex()` | `int32` | 按模块索引读取 8-bit 原始值 |
| `GetAttributeRaw16ByIndex()` | `int32` | 按模块索引读取 16-bit 原始值（Coarse×256+Fine） |
| `GetAttributeRaw24ByIndex()` | `int32` | 按模块索引读取 24-bit 原始值 |
| `GetChannelValue()` | `float [0,1]` | 按相对地址读取单通道值 |
| `GetMatrixAttributeRaw()` | `TArray<float>` | 矩阵模式：读取所有模块实例的同名属性 |
| `GetMatrixAttributeRaw16()` | `TArray<float>` | 矩阵模式：16-bit 版本 |

**蓝图事件**：
- `SuperDMXTick(float DeltaSeconds)`：**每帧触发**的蓝图可实现事件，用户在此事件中编写 DMX 控制逻辑
- `LightInitialization`：灯具初始化委托

##### 第三层：ASuperLightBase — Pan/Tilt 运动控制层

在 DMX 读取层之上增加 **摇头灯运动控制**：

**组件结构**：
```
SceneBase
  └─ SceneRocker  (Pan 旋转轴 — 水平旋转)
      └─ SceneHard (Tilt 旋转轴 — 垂直旋转)
          └─ 灯头组件 (光束/模型等)
```

**运动控制 API**：

| 函数 | 说明 |
|------|------|
| `YPan()` | 水平旋转（Pan），DMX→角度范围映射 |
| `YTilt()` | 垂直旋转（Tilt） |
| `YPTSpeed()` | Pan/Tilt 速度控制 |
| `YPanRot()` / `YTiltRot()` | 无极旋转（连续旋转模式） |
| `YMatrixPan()` / `YMatrixTilt()` | 矩阵多灯头独立 Pan/Tilt |
| `YMatrixPanRot()` / `YMatrixTiltRot()` | 矩阵多灯头无极旋转 |
| `YLampAngle()` | 手动设置灯具角度 |
| `SetLiftZ()` | 升降控制（配合 `USuperLiftComponent`） |

**运动参数**：
- `PanRange` / `TiltRange`：角度范围（默认 ±270° / ±135°）
- `InfiniteRotationalSpeed`：无极旋转速度倍率
- `PTSpeed`：Pan/Tilt 插值速度（`FInterpTo` 平滑）

##### 第四层：ASuperStageLight — 全功能电脑灯

**最丰富的灯具类**，提供完整的专业灯具控制能力，包含 **90+ 个 BlueprintCallable 函数**，覆盖以下功能域：

**1. 主光源控制 (Lighting)**

| 功能 | 单灯函数 | 矩阵函数 |
|------|---------|---------|
| 亮度 | `SetLightingIntensity` | `SetLightingIntensityMatrix` |
| 频闪 | `SetLightingStrobe` | `SetLightingStrobeMatrix` |
| RGB | `SetLightingColorRGB` | `SetLightingColorRGBMatrix` |
| RGBW | `SetLightingColorRGBW` | `SetLightingColorRGBWMatrix` |
| HSV | `SetLightingColorHSV` | `SetLightingColorHSVMatrix` |
| HS | `SetLightingColorHS` | `SetLightingColorHSMatrix` |
| 颜色轮 | `SetLightingColorWheel` | — |
| 颜色轮+CMY | `SetLightingColorWheelAndCMY` | — |
| 多颜色轮 | `SetLightingColorWheel2/3` | — |
| 色温 | `SetLightingColorTemperature` | — |
| 冷暖混色 | `SetLightingCoolWarmMix` | — |
| 多通道混色 | `SetLightingColorMix` | `SetLightingColorMixMatrix` |
| Zoom | `SetLightingZoom` | `SetLightingZoomMatrix` |
| Frost | `SetLightingFrost` | — |
| Iris | `SetLightingIris` | — |

**2. Spot 辅光控制**

| 功能 | 单灯函数 | 矩阵函数 |
|------|---------|---------|
| 亮度 | `SetSpotIntensity` | `SetSpotIntensityMatrix` |
| 频闪 | `SetSpotStrobe` | `SetSpotStrobeMatrix` |
| RGB | `SetSpotColorRGB` | `SetSpotColorRGBMatrix` |
| RGBW | `SetSpotColorRGBW` | `SetSpotColorRGBWMatrix` |
| 冷暖混色 | `SetSpotCoolWarmMix` | `SetSpotCoolWarmMixMatrix` |
| Zoom | `SetSpotZoom` | `SetSpotZoomMatrix` |
| Frost | `SetSpotFrost` | `SetSpotFrostMatrix` |
| 色温矩阵 | — | `SetSpotColorTemperatureMatrix` |

**3. 图案/棱镜/切割**

| 函数 | 说明 |
|------|------|
| `SetBeamGobo1/2/3()` | 1~3 个图案轮 + 旋转控制（含图集纹理） |
| `SetBeamPrism()` | 棱镜选择 + 旋转 |
| `SetBeamFocus()` | 调焦控制 |
| `SetBeamCutting()` | 四叶片切割（A1/B1 ~ A4/B4，8 通道） |
| `SetBeamCuttingRotate()` | 切割系统旋转 |

**4. 效果层**

| 函数 | 说明 |
|------|------|
| `SetEffectDefaultValue()` | 初始化效果组件 |
| `SetEffectIntensity()` | 效果亮度 |
| `SetEffectStrobe()` | 效果频闪 |
| `SetEffectColor()` / `SetEffectColorMatrix()` | 效果颜色 |
| `SetEffect()` / `SetEffectMatrix()` | 效果选择（LUT 查表，10 种内置效果） |
| `SetEffectWithSpeed()` | 效果 + 独立速度通道 |

**5. 矩阵像素控制**

| 函数 | 说明 |
|------|------|
| `SetMatrixComponent()` | 矩阵 RGB + 频闪 |
| `SetMatrixWhiteSingle/Multiple()` | 矩阵白光（单点/多点） |
| `SetMatrixColorSingle/Multiple()` | 矩阵 RGB（单像素/多像素） |
| `SetMatrixIntensity()` | 矩阵亮度 |
| `SetMatrixStrobe()` | 矩阵频闪 |
| `SetMatrixCoolWarmMixSingle/Multiple()` | 矩阵冷暖混色 |
| `SetMatrixColorMix()` | 矩阵多通道混色 |
| `SetMatrixColorRGBW*()` | 矩阵 RGBW 系列 |
| `SetMatrixColorRGBWithCTO*()` | 矩阵 RGB+CTO 系列 |

---

### 3.3 其他 Actor 类型

#### 3.3.1 ASuperLiftingMachinery — 升降机械

通过 DMX 控制舞台设备的 **6 轴运动**（XYZ 位移 + XYZ 旋转），支持绝对位置模式和无极旋转模式。

| 函数 | 说明 |
|------|------|
| `LiftingMachinery()` | 主控制函数（6 个 DMX 属性参数） |
| `BootRefresh()` | 初始化运动范围基准 |

**控制参数**：PosX/Y/Z（位移 [0,1]）、RotX/Y/Z（旋转 [0,1]）、PosSpeed/RotSpeed（插值速度）

#### 3.3.2 ASuperRailMachinery — 轨道机械

在 6 轴运动基础上新增 **样条曲线轨道轴**，共 **7 轴 DMX 控制**。

| 函数 | 说明 |
|------|------|
| `RailMachinery()` | 7 轴主控制（RailPos + XYZ 偏移 + XYZ 旋转） |
| `RailPositionOnly()` | 仅轨道位置控制 |
| `BootRefresh()` | 初始化范围基准 |

**特色功能**：
- `USplineComponent` 定义轨道路径（编辑器可视化编辑）
- `bLockOrientationToRail`：锁定朝向沿轨道切线
- `bClosedLoop`：闭合轨道循环运动

#### 3.3.3 ASuperDMXCamera — DMX 摄像机

通过 DMX 控制 `UCineCameraComponent` 的位置、旋转和摄像机参数。

| 函数 | 说明 |
|------|------|
| `DMXCameraControl()` | 6 轴运动控制（XYZ 位移 + XYZ 旋转） |
| `DMXCameraParams()` | 摄像机参数（FOV + 光圈 + 对焦距离） |
| `GetCineCamera()` | 获取摄像机组件 |
| `GetRenderTarget()` | 获取渲染目标（用于 LED 屏幕输出） |

#### 3.3.4 ASuperStageVFXActor — Niagara VFX

通过 DMX 控制 Niagara 粒子系统（烟雾/火焰/雪花/CO2 等）。

| 函数 | 说明 |
|------|------|
| `SetNiagaraDefaultValue()` | 初始化 Niagara 参数 |
| `SetNiagaraSpawnCount()` | DMX 控制粒子生成数量 |
| `SetNiagaraColor()` | DMX 控制粒子颜色 |

**Niagara 变量映射**：Speed, LoopTime, Minimum, Maximum, SpawnCount, Color

#### 3.3.5 ASuperLightStripEffect — LED 灯带

通过材质实现可编程 LED 灯带效果。

| 函数 | 说明 |
|------|------|
| `SetLightStripDefault()` | 初始化材质 |
| `SetLightStripEffect()` | 完整控制（亮度/频闪/效果/速度/宽度/方向/RGB） |

**特色**：支持将效果材质批量应用到多个 `AStaticMeshActor`。

#### 3.3.6 ASuperLiftMatrix — 升降矩阵

包含 5 个垂直排列的场景组件 + 10 个效果组件 + 4 根钢丝绳。

| 函数 | 说明 |
|------|------|
| `LiftMatrix()` | 升降控制 |
| `SetEffectMatrixDefaultValue()` | 初始化效果组件 |
| `SetEffectMatrix()` | 效果选择 |
| `SetEffectColorMatrix()` | 效果颜色 |

#### 3.3.7 ASuperNDIScreen — NDI 视频屏幕

从 NDI 网络接收实时视频流并显示到屏幕网格。

**关键属性**：
- `InputName`：NDI 源名称（带下拉选择）
- `TargetStaticMeshActors`：多屏输出目标
- `bTransparent`：透明模式开关
- `Deformation`：四点梯形校正（投影融合）

#### 3.3.8 ASuperLaserActor / ASuperLaserProActor — 激光

| 类 | 说明 |
|----|------|
| `ASuperLaserActor` | 基于 RenderTarget 纹理的激光显示 |
| `ASuperLaserProActor` | 基于点数据的激光线绘制（`USuperLaserProComponent`） |

#### 3.3.9 ASuperProjector — 投影 Mapping

使用 Light Function 实现 Projection Mapping，支持 RGB 三通道叠加投影 + 梯形校正。

#### 3.3.10 舞台搭建资产

| 类 | 说明 |
|----|------|
| `ASuperTruss` | 桁架（直线/弧形） |
| `ASuperScaffold` | 脚手架（可配置层数和尺寸） |
| `ASuperCurvedScaffold` | 弧形脚手架 |
| `ASuperDrape` | 幕布（可控制褶皱和颜色） |

---

## 4. 组件体系

SuperStage 的灯光渲染通过**组件化设计**实现高度复用：

### 4.1 USuperLightingComponent — 光束基础组件

所有光源组件的基类，提供：
- 材质管理（动态材质实例）
- 强度/颜色/频闪控制
- Zoom/Frost/Iris 控制
- 光束旋转
- 光线遮挡检测（Ray Detection）

**派生组件**：

| 组件 | 说明 |
|------|------|
| `USuperSpotComponent` | SpotLight 辅光组件，管理 `USpotLightComponent`，支持 Zoom/Iris/Frost/色温 |
| `USuperBeamComponent` | Beam 光束组件（聚光灯/光束灯效果） |
| `USuperRectComponent` | RectLight 面光组件（Wash 灯/面板灯效果） |

### 4.2 独立功能组件

| 组件 | 说明 |
|------|------|
| `USuperEffectComponent` | LED 效果组件：10 种内置效果 + 速度/宽度/方向控制 |
| `USuperCuttingComponent` | 切割系统组件：四叶片 Shaper + 旋转 |
| `USuperMatrixComponent` | LED 矩阵像素组件：逐像素颜色/亮度/频闪 |
| `USuperLiftComponent` | 升降组件：平滑升降运动 |
| `USuperLaserProComponent` | 激光 Pro 渲染组件：点数据→程序化网格 |

---

## 5. 数据流与执行模型

### 5.1 DMX 数据流

```
┌──────────────────┐
│   外部 DMX 控台   │
│   (Art-Net 输出)  │
└────────┬─────────┘
         │ UDP 包
         ▼
┌──────────────────────────────────┐
│   USuperDMXSubsystem             │
│   ┌──────────────────────────┐  │
│   │   接收线程               │  │
│   │   解析 Art-Net 包         │  │
│   │   写入 UniverseToSlots   │  │
│   └──────────┬───────────────┘  │
│              │ FCriticalSection  │
│   ┌──────────▼───────────────┐  │
│   │   UniverseToSlots        │  │
│   │   TMap<int32, uint8[512]>│  │
│   └──────────┬───────────────┘  │
└──────────────┼──────────────────┘
               │ GetDMXBuffer / GetDMXValue
               ▼
┌──────────────────────────────────┐
│   ASuperDmxActorBase::Tick()     │
│   ├─ 获取 Universe 快照          │
│   ├─ 调用 SuperDMXTick() 蓝图事件│
│   │   └─ 用户蓝图逻辑            │
│   │       ├─ SetLightingIntensity│
│   │       ├─ YPan / YTilt        │
│   │       ├─ SetLightingColorRGB │
│   │       └─ ...                 │
│   └─ 更新组件渲染状态             │
└──────────────────────────────────┘
```

### 5.2 蓝图开发工作流

典型的蓝图开发流程：

1. **放置灯具 Actor** — 将 `ASuperStageLight`（或其子类蓝图）拖入场景
2. **配置灯库** — 设置 `FixtureLibrary`（灯库数据资产）
3. **配置 DMX 地址** — 设置 `SuperDMXFixture`（Universe + StartAddress）
4. **设置组件默认参数**（可选）— 在组件面板调整 `MaxLightIntensity`、`ZoomRange` 等
5. **编写控制逻辑** — 在蓝图中实现 `SuperDMXTick` 事件：
   ```
   Event SuperDMXTick
     → SetLightingIntensity (Dimmer 属性)
     → SetLightingStrobe (Strobe 属性)
     → SetLightingColorRGB (Red/Green/Blue 属性)
     → YPan (Pan 属性)
     → YTilt (Tilt 属性)
     → SetLightingZoom (Zoom 属性)
     → ...
   ```
6. **启动 DMX 接收** — 通过编辑器工具或蓝图调用 `USuperDMXSubsystem::ApplyConfig()`

> **26Q2.2 更新**：组件默认参数由 `OnRegister()` 自动初始化。组件注册时会自动调用 `SetLightingMaterial()` 创建材质实例，再调用 `SetLightingDefaultValue()` 设置默认值。**无需在 Actor 中手动调用初始化函数**。

### 5.3 FSuperDMXAttribute 查询机制

所有灯具控制函数接受 `FSuperDMXAttribute` 结构体作为参数：

```
FSuperDMXAttribute
 ├── InstanceIndex : int32    → 模块实例索引（0 = 主模块，1+ = 矩阵子模块）
 ├── AttribName    : FName    → 属性名称（必须与灯库中定义一致）
 └── DMXChannelType: EDMXChannelType → 通道精度（8-Bit / 16-Bit / 24-Bit）
```

**查询流程**：
1. 根据 `InstanceIndex` 定位 `FSuperDMXModuleInstance`
2. 根据 `AttribName` 查找 `FSuperDMXAttributeDef`
3. 计算绝对 DMX 地址 = `StartAddress + Patch + Coarse - 1`
4. 从 `USuperDMXSubsystem` 缓存读取对应字节
5. 根据 `DMXChannelType` 组合 Coarse/Fine/Ultra 字节
6. 归一化到 [0,1] 范围（如使用带归一化的 API）

---

## 6. 权限系统

SuperStage 内建 **订阅权限检查**，控制 DMX 读取和功能使用：

```cpp
// 宏定义于 SuperStagePermission.h
SUPER_REQUIRE_PERMISSION_VOID()    // 无权限则 return;
SUPER_REQUIRE_PERMISSION_RET(val)  // 无权限则 return val;
SUPER_IF_PERMISSION { ... }        // 条件执行
```

**权限规则**：
- ✅ **打包运行**（Standalone）：所有功能无限制
- ✅ **编辑器预览**（Preview）：所有功能无限制
- 🔒 **PIE / GamePreview / Editor**：需要有效订阅

> **对集成开发者的影响**：DMX 数据读取函数内部已包含权限检查，无需手动处理。未授权时这些函数将返回默认值（0 或不执行）。

---

## 7. 颜色系统详解

SuperStage 支持多种颜色控制模式，覆盖行业所有主流方案：

### 7.1 直接控制模式

| 模式 | API | 说明 |
|------|-----|------|
| **RGB** | `SetLightingColorRGB` | 三通道独立控制（最常用） |
| **RGBW** | `SetLightingColorRGBW` | 四通道，白光通道叠加到 RGB |
| **HSV** | `SetLightingColorHSV` | 色相/饱和度/明度 |
| **HS** | `SetLightingColorHS` | 色相/饱和度（明度由 Dimmer 控制） |

### 7.2 颜色轮模式

| API | 说明 |
|-----|------|
| `SetLightingColorWheel` | 单颜色轮（传入颜色图集纹理） |
| `SetLightingColorWheel2/3` | 多颜色轮叠加 |
| `SetLightingColorWheelAndCMY` | 颜色轮 + CMY 减色混合 |
| `SetLightingColorWheel2AndCMYAndCTO` | 多轮 + CMY + CTO 色温调节 |

### 7.3 色温模式

| API | 说明 |
|-----|------|
| `SetLightingColorTemperature` | 单通道色温（暖→冷，1700K-12000K） |
| `SetLightingCoolWarmMix` | 双通道冷暖混色（独立 Cool/Warm 控制） |

### 7.4 动态多通道混色

| API | 说明 |
|-----|------|
| `SetLightingColorMix` | 任意多通道混色（RGBW/RGBA/RGBLAM 等） |
| `SetLightingColorMixWithCTO` | 多通道混色 + CTO 色温调节 |

多通道混色使用 `FSuperColorChannel` 结构体：
```
FSuperColorChannel
 ├── DmxAttribute : FSuperDMXAttribute  → DMX 通道
 └── ChannelColor : FLinearColor         → 该通道对应的颜色
```
**混色算法**：所有通道的 `(DMX值 × 通道颜色)` 加法叠加后 Clamp。

---

## 8. 矩阵控制模式

矩阵控制允许对灯具中的每个像素/灯头进行独立控制。

### 8.1 工作原理

1. 灯库中定义多个 `FSuperDMXModuleInstance`（每个像素一个模块）
2. 每个模块通过 `Patch` 偏移量区分通道范围
3. 矩阵 API（如 `SetLightingIntensityMatrix`）自动遍历所有模块实例
4. `bSortByPatch` 参数控制是否按 Patch 排序输出

### 8.2 单点 vs 多点 API 对比

| 类别 | 单点（Single） | 多点（Multiple） |
|------|---------------|-----------------|
| 参数 | 需要 `int32 Index` | 自动遍历所有模块 |
| 用途 | 精确控制某个像素 | 批量控制所有像素 |
| 示例 | `SetMatrixColorSingle(R,G,B,0,Matrix)` | `SetMatrixColorMultiple(R,G,B,Matrix)` |

---

## 9. C++ 插件集成指南

### 9.1 模块依赖配置

在你的插件 `.Build.cs` 中添加：

```csharp
PublicDependencyModuleNames.AddRange(new string[]
{
    "SuperDMX",     // DMX 通信
    "SuperStage",   // 灯具控制
});
```

### 9.2 获取 DMX 子系统

```cpp
#include "SuperDMXSubsystem.h"

USuperDMXSubsystem* DMX = GEngine->GetEngineSubsystem<USuperDMXSubsystem>();
if (DMX)
{
    // 配置 Art-Net 接收
    FSuperDMXIOConfig Config;
    Config.Protocol = ESuperDMXProtocol::ArtNet;
    Config.Input.bEnabled = true;
    Config.Input.Port = 6454;
    DMX->ApplyConfig(Config);

    // 读取 DMX 数据
    TArray<uint8> Buffer;
    if (DMX->GetDMXBuffer(1, Buffer))  // Universe 1
    {
        uint8 DimmerValue = Buffer[0]; // Channel 1
    }
}
```

### 9.3 访问灯具属性

```cpp
#include "SuperDmxActorBase.h"
#include "AssetTool/SuperFixtureLibrary.h"

// 获取场景中的灯具 Actor
ASuperDmxActorBase* Light = /* FindActor... */;

// 读取 DMX 属性值（归一化 0-1）
FSuperDMXAttribute DimmerAttr;
DimmerAttr.InstanceIndex = 0;
DimmerAttr.AttribName = FName("Dimmer");
DimmerAttr.DMXChannelType = EDMXChannelType::Conarse;

float DimmerValue = 1.0f;
Light->GetSuperDmxAttributeValue(DimmerAttr, DimmerValue);
// DimmerValue 现在包含归一化的亮度值
```

---

## 10. 公开 API 类索引

### SuperDMX 模块

| 类 | 头文件 | 说明 |
|----|--------|------|
| `USuperDMXSubsystem` | `SuperDMXSubsystem.h` | DMX 通信子系统（全局单例） |
| `UMovieSceneSuperDMXTrack` | `Sequencer/MovieSceneSuperDMXTrack.h` | Sequencer DMX 轨道 |
| `UMovieSceneSuperDMXSection` | `Sequencer/MovieSceneSuperDMXSection.h` | Sequencer DMX 段落 |

### SuperStage 模块 — Actor

| 类 | 头文件 | 说明 |
|----|--------|------|
| `ASuperBaseActor` | `SuperBaseActor.h` | 基础资产 Actor |
| `ASuperDmxActorBase` | `SuperDmxActorBase.h` | DMX 读取基类 |
| `ASuperLightBase` | `LightActor/SuperLightBase.h` | Pan/Tilt 运动基类 |
| `ASuperStageLight` | `LightActor/SuperStageLight.h` | 全功能电脑灯 |
| `ASuperStageVFXActor` | `LightActor/SuperStageVFXActor.h` | Niagara VFX Actor |
| `ASuperLiftMatrix` | `LightActor/SuperLiftMatrix.h` | 升降矩阵 |
| `ASuperLiftingMachinery` | `LightActor/SuperLiftingMachinery.h` | 升降机械（6轴） |
| `ASuperRailMachinery` | `LightActor/SuperRailMachinery.h` | 轨道机械（7轴） |
| `ASuperDMXCamera` | `LightActor/SuperDMXCamera.h` | DMX 摄像机 |
| `ASuperLightStripEffect` | `LightActor/SuperLightStripEffect.h` | LED 灯带效果 |
| `ASuperNDIScreen` | `LightActor/SuperNDIScreen.h` | NDI 视频屏幕 |
| `ASuperLaserActor` | `LightActor/SuperLaserActor.h` | 激光纹理显示 |
| `ASuperLaserProActor` | `LightActor/SuperLaserProActor.h` | 激光点数据显示 |
| `ASuperProjector` | `LightActor/SuperProjector.h` | 投影 Mapping |
| `ASuperTruss` | `StageAssets/SuperTruss.h` | 桁架 |
| `ASuperScaffold` | `StageAssets/SuperScaffold.h` | 脚手架 |
| `ASuperCurvedScaffold` | `StageAssets/SuperCurvedScaffold.h` | 弧形脚手架 |
| `ASuperDrape` | `StageAssets/SuperDrape.h` | 幕布 |

### SuperStage 模块 — 组件

| 类 | 头文件 | 说明 |
|----|--------|------|
| `USuperLightingComponent` | `LightComponent/SuperLightingComponent.h` | 光束基础组件 |
| `USuperSpotComponent` | `LightComponent/SuperSpotComponent.h` | SpotLight 辅光组件 |
| `USuperBeamComponent` | `LightComponent/SuperBeamComponent.h` | Beam 光束组件 |
| `USuperRectComponent` | `LightComponent/SuperRectComponent.h` | RectLight 面光组件 |
| `USuperEffectComponent` | `LightComponent/SuperEffectComponent.h` | LED 效果组件 |
| `USuperCuttingComponent` | `LightComponent/SuperCuttingComponent.h` | 切割系统组件 |
| `USuperMatrixComponent` | `LightComponent/SuperMatrixComponent.h` | LED 矩阵像素组件 |
| `USuperLiftComponent` | `LightComponent/SuperLiftComponent.h` | 升降组件 |
| `USuperLaserProComponent` | `LightComponent/SuperLaserProComponent.h` | 激光 Pro 渲染组件 |

### SuperStage 模块 — 数据类型

| 类型 | 头文件 | 说明 |
|------|--------|------|
| `USuperFixtureLibrary` | `AssetTool/SuperFixtureLibrary.h` | 灯库数据资产 |
| `FSuperDMXModuleInstance` | `AssetTool/SuperFixtureLibrary.h` | 模块实例定义 |
| `FSuperDMXAttributeDef` | `AssetTool/SuperFixtureLibrary.h` | 属性通道定义 |
| `FSuperDMXAttribute` | `AssetTool/SuperFixtureLibrary.h` | 属性查询键 |
| `FSubAttribute` | `AssetTool/SuperFixtureLibrary.h` | 子属性/功能定义 |
| `FChannelSet` | `AssetTool/SuperFixtureLibrary.h` | 槽位集合 |
| `FSuperDMXFixture` | `LightActor/SuperLightTypes.h` | 灯具 DMX 地址 |
| `FAssetMetaData` | `SuperBaseActor.h` | 资产元数据 |
| `FSuperColorChannel` | `LightActor/SuperStageLight.h` | 动态混色通道 |
| `ESuperDMXProtocol` | `SuperDMXTypes.h` | DMX 协议枚举 |
| `FSuperDMXIOConfig` | `SuperDMXTypes.h` | DMX I/O 配置 |
| `FSuperDMXEndpoint` | `SuperDMXTypes.h` | DMX 网络端点 |
| `EDMXChannelType` | `AssetTool/SuperFixtureLibrary.h` | 通道精度枚举 |
| `EDMXAttributeCategory` | `AssetTool/SuperFixtureLibrary.h` | 属性分类枚举 |
| `EStrobeMode` | `AssetTool/SuperFixtureLibrary.h` | 频闪模式枚举 |
| `EGoboMode` | `AssetTool/SuperFixtureLibrary.h` | 图案模式枚举 |
| `EInfiniteRotationMode` | `AssetTool/SuperFixtureLibrary.h` | 无极旋转模式枚举 |
| `EFixtureLibrarySource` | `AssetTool/SuperFixtureLibrary.h` | 灯库来源枚举 |

---

## 11. 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| 灯库 | Fixture Library | 定义灯具 DMX 通道配置的数据资产 |
| Universe | Universe | DMX 域，每个域包含 512 个通道 |
| 通道 | Channel | DMX 地址（1-512），每个通道携带 0-255 的值 |
| Coarse/Fine/Ultra | — | 8/16/24 位精度控制，Fine 为低字节 |
| Pan/Tilt | — | 水平/垂直旋转，电脑灯基本运动轴 |
| Gobo | — | 图案轮，通过旋转金属片或玻璃片投射图案 |
| Prism | — | 棱镜，将光束分裂为多束 |
| CMY | Cyan/Magenta/Yellow | 减色混色系统（青/品红/黄色滤片） |
| CTO | Color Temperature Orange | 色温调节滤片（橙色，降低色温） |
| Frost | — | 雾化滤片，使光束边缘柔和 |
| Iris | — | 光圈，控制光束直径 |
| Shaper | — | 切割系统（四叶片，也称 Framing System） |
| Art-Net | — | 基于 UDP 的 DMX over IP 协议，默认端口 6454 |
| sACN | E1.31 | ANSI E1.31 流式 ACN 协议，使用 IP 多播 |
| NDI | Network Device Interface | NewTek 网络视频传输协议 |
| Sequencer | — | UE5 内置时间轴编辑器 |
| LUT | Look-Up Table | 查找表，用于效果索引→参数映射 |
| Patch | — | 灯具模块的通道偏移量 |

---

> **下一步**：各类详细 API 文档将在后续章节中逐一展开，包含每个公开函数的参数说明、返回值、使用示例和注意事项。

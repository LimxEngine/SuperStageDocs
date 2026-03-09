# SuperStage 灯具开发手册

> **文档版本**: 1.0  
> **适用版本**: SuperStage Plugin 2025/2026  
> **目标读者**: C++ 灯具开发者 / 蓝图灯具制作者  
> **最后更新**: 2026-03

---

## 目录

1. [概述](#1-概述)
2. [核心概念](#2-核心概念)
3. [灯库系统](#3-灯库系统)
4. [Actor 继承体系](#4-actor-继承体系)
5. [组件体系](#5-组件体系)
6. [C++ 灯具开发流程](#6-c-灯具开发流程)
7. [蓝图灯具开发流程](#7-蓝图灯具开发流程)
8. [DMX 通道读取 API](#8-dmx-通道读取-api)
9. [颜色系统详解](#9-颜色系统详解)
10. [图案与棱镜系统](#10-图案与棱镜系统)
11. [切割系统](#11-切割系统)
12. [效果系统](#12-效果系统)
13. [矩阵系统](#13-矩阵系统)
14. [运动控制](#14-运动控制)
15. [频闪系统](#15-频闪系统)
16. [完整示例：C++ Beam 电脑灯](#16-完整示例c-beam-电脑灯)
17. [完整示例：蓝图 Wash 染色灯](#17-完整示例蓝图-wash-染色灯)
18. [完整示例：蓝图频闪灯](#18-完整示例蓝图频闪灯)
19. [完整示例：蓝图 Profile 摇头灯](#19-完整示例蓝图-profile-摇头灯)
20. [完整示例：蓝图矩阵灯（带辅光）](#20-完整示例蓝图矩阵灯带辅光)
21. [完整示例：蓝图摇头矩阵灯](#21-完整示例蓝图摇头矩阵灯)
22. [完整示例：蓝图多功能摇头灯](#22-完整示例蓝图多功能摇头灯)
23. [各类灯具制作详解](#23-各类灯具制作详解)
24. [灯库编辑器使用指南](#24-灯库编辑器使用指南)
25. [常见问题与排错](#25-常见问题与排错)
26. [附录：标准属性名对照表](#26-附录标准属性名对照表)

---

## 1. 概述

SuperStage 灯具开发系统提供从数据定义到视觉渲染的完整链路：

```
灯库资产 (USuperFixtureLibrary)      ← 定义 DMX 通道映射
    ↓ 引用
灯具 Actor (ASuperStageLight 等)      ← 读取 DMX 数据 + 驱动组件
    ↓ 拥有
灯光组件 (USuperBeamComponent 等)     ← 渲染光束/光斑/效果
    ↓ 写入
动态材质 (UMaterialInstanceDynamic)   ← GPU 上可视化
```

**两种开发路径**：

| 路径 | 适用场景 | 技术要求 |
|------|---------|---------|
| **C++ 子类** | 复杂灯具、需要极致性能、新灯具类型 | C++ + UE 反射系统 |
| **蓝图子类** | 快速原型、标准灯具变体、用户自定义 | 蓝图 Event Graph |

---

## 2. 核心概念

### 2.1 数据流

```
DMX 信号（Art-Net/sACN）
    ↓ USuperDMXSubsystem 接收
DMX Buffer（uint8[512] × 256 Universe）
    ↓ ASuperDmxActorBase::Tick() 读取
归一化值 [0,1] 或原始值 [0-255]
    ↓ 控制函数（SetLightingIntensity 等）
组件参数（材质参数/光源属性/网格变换）
    ↓ 渲染
可视化输出
```

### 2.2 归一化约定

**所有** DMX 属性值在 Actor 层面读取后归一化为 `[0, 1]`：

| DMX 精度 | 原始范围 | 归一化公式 |
|----------|---------|-----------|
| 8-bit | `0-255` | `value / 255.0` |
| 16-bit | `0-65535` | `value / 65535.0` |
| 24-bit | `0-16777215` | `value / 16777215.0` |

### 2.3 FSuperDMXAttribute — DMX 属性引用

```cpp
USTRUCT(BlueprintType)
struct FSuperDMXAttribute
{
    int32 InstanceIndex = 0;    // 模块实例索引（0基）
    FName AttribName;           // 属性名（必须与灯库定义完全一致）
    EDMXChannelType DMXChannelType; // 精度：Conarse(8bit) / Fine(16bit) / Ultra(24bit)
};
```

这是蓝图和 C++ 中引用 DMX 通道的**统一结构体**。

### 2.4 FSuperDMXFixture — DMX 地址

```cpp
USTRUCT(BlueprintType)
struct FSuperDMXFixture
{
    int32 FixtureID = 1;     // 灯具编号（用户标识）
    int32 Universe = 1;      // DMX Universe（1-256）
    int32 StartAddress = 1;  // 起始地址（1-512）
};
```

每个灯具 Actor 的 `SuperDMXFixture` 属性定义了它在 DMX 网络中的位置。

---

## 3. 灯库系统

### 3.1 数据模型

```
USuperFixtureLibrary (UPrimaryDataAsset)
├── FixtureName: "Acme XP-380 Beam 17CH"
├── Manufacturer: "Acme"
├── Source: Custom / GDTF / MA2 / MA3
├── PowerConsumption: 540.0 W
├── Weight: 21.2 kg
└── Modules[]: FSuperDMXModuleInstance
    ├── ModuleName: "Mode 1"
    ├── Patch: 1 (通道偏移，0基)
    └── AttributeDefs[]: FSuperDMXAttributeDef
        ├── AttribName: "Dimmer"
        ├── Category: EDMXAttributeCategory::Dimmer
        ├── Coarse: 1 (粗字节通道，1基)
        ├── Fine: 2 (细字节通道，0=未使用)
        ├── Ultra: 0 (超细字节，0=未使用)
        ├── DefaultValue: 0 (默认DMX值)
        ├── HighlightValue: 255 (高亮DMX值)
        └── SubAttributes[]: FSubAttribute
            ├── StrobeMode / RotationMode
            ├── DmxMin / DmxMax
            ├── PhysicalRange: FVector2D
            └── ChannelSets[]: FChannelSet
                ├── Name / GoboMode
                ├── DmxMin / DmxMax
                ├── PhysicalRange
                ├── Texture / Color / ColorIndex
                └── PrismFacets / PrismRadius / PrismScale
```

### 3.2 灯库层级关系

```
USuperFixtureLibrary
 └── Modules[] — 模块实例（对应灯具的不同控制单元）
      └── AttributeDefs[] — 属性定义（每个 DMX 通道）
           └── SubAttributes[] — 子属性（DMX 值范围分段）
                └── ChannelSets[] — 槽位集合（如图案轮的每个图案）
```

**关键概念**：
- **Module（模块）**：对应灯具的一个控制单元。单头灯具通常只有 1 个模块；矩阵灯/多头灯有多个模块，每个模块对应一个像素/灯头。
- **AttributeDef（属性定义）**：对应一个 DMX 通道组（Coarse + 可选 Fine + 可选 Ultra）。
- **SubAttribute（子属性）**：将一个通道的 DMX 值范围分段，每段可以有不同的行为（如 Dimmer 通道的 0-7=关闭，8-255=亮度）。
- **ChannelSet（槽位）**：子属性内的更细粒度分段，用于图案轮的每个图案、颜色轮的每个颜色等。

### 3.3 通道地址计算

```
绝对地址 = StartAddress + Module.Patch + AttributeDef.Coarse - 1
```

**示例**：StartAddress=101, Patch=0, Coarse=3 → 绝对地址=103

### 3.4 通道跨度

```cpp
int32 USuperFixtureLibrary::GetChannelSpan() const;
```

返回灯具占用的总通道数（最大地址 - 最小地址 + 1）。

### 3.5 创建灯库资产

**方法 1 — Content Browser**：
1. Content Browser → 右键 → SuperStage → `Super灯库`
2. 双击打开灯库编辑器
3. 填写灯具名称、制造商、来源
4. 添加模块和属性定义
5. Ctrl+S 保存

**方法 2 — C++ 运行时创建**：
```cpp
USuperFixtureLibrary* Lib = NewObject<USuperFixtureLibrary>(GetTransientPackage());
Lib->FixtureName = TEXT("My Custom Beam");
Lib->Manufacturer = TEXT("MyBrand");

FSuperDMXModuleInstance& Mod = Lib->Modules.AddDefaulted_GetRef();
Mod.ModuleName = FName(TEXT("Mode 1"));
Mod.Patch = 1;

// 添加 Dimmer 属性
FSuperDMXAttributeDef& Dimmer = Mod.AttributeDefs.AddDefaulted_GetRef();
Dimmer.AttribName = FName(TEXT("Dimmer"));
Dimmer.Category = EDMXAttributeCategory::Dimmer;
Dimmer.Coarse = 1;
Dimmer.Fine = 2; // 16bit
```

### 3.6 灯库枚举类型速查

#### EDMXChannelType

| 值 | 精度 | 范围 |
|----|------|------|
| `Conarse` | 8-bit | 0-255 |
| `Fine` | 16-bit | 0-65535 |
| `Ultra` | 24-bit | 0-16777215 |

#### EDMXAttributeCategory

| 值 | 含义 | 典型属性名 |
|----|------|-----------|
| `Dimmer` | 亮度 | Dimmer, Intensity |
| `Position` | 位置 | Pan, Tilt, PanRot |
| `Gobo` | 图案 | Gobo1, Gobo2 |
| `Color` | 颜色 | ColorR, ColorG, ColorWheel |
| `Beam` | 光束 | Zoom, Iris |
| `Focus` | 聚焦 | Focus |
| `Control` | 控制 | Control, Reset |
| `Shapers` | 切割 | A1, B1, ShaperRot |
| `Strobe` | 频闪 | Strobe, StrobeMode |
| `Prism` | 棱镜 | Prism1, Prism2 |
| `Frost` | 雾化 | Frost |
| `Effects` | 效果 | EffectValue, EffectSpeed |
| `Other` | 其他 | — |

#### EStrobeMode

| 值 | 模式 | 波形 |
|----|------|------|
| 0 | Closed | 常灭 |
| 1 | Open | 常亮 |
| 2 | Linear | 线性（三角波） |
| 3 | Pulse | 方波 |
| 4 | RampUp | 锯齿上升 |
| 5 | RampDown | 锯齿下降 |
| 6 | Sine | 正弦波 |
| 7 | Random | 随机 |

#### EInfiniteRotationMode

| 值 | 模式 | 行为 |
|----|------|------|
| `Off` | 关闭 | 正常位置控制 |
| `Stop` | 停止 | 无极旋转暂停 |
| `Position` | 位置 | 从当前位置旋转到目标 |
| `Infinite` | 无极旋转 | 持续旋转 |

---

## 4. Actor 继承体系

```
AActor
 └─ ASuperBaseActor
     │  ├─ SceneBase (根组件)
     │  ├─ ForwardArrow / UpArrow (方向预览)
     │  └─ AssetData (FAssetMetaData: UUID/制造商/缩略图)
     └─ ASuperDmxActorBase
         │  ├─ SuperDMXFixture (Universe/StartAddress/FixtureID)
         │  ├─ FixtureLibrary → USuperFixtureLibrary*
         │  ├─ AddressLabel (地址码文本)
         │  ├─ GetChannelValue() / GetSuperDmxAttributeValue()
         │  ├─ GetAttributeRaw8/16/24ByIndex()
         │  ├─ GetMatrixAttributeRaw/Raw16()
         │  └─ SuperDMXTick (蓝图可实现事件)
         └─ ASuperLightBase
             │  ├─ SceneRocker (Pan旋转轴)
             │  ├─ SceneHard (Tilt旋转轴)
             │  ├─ YPan() / YTilt() / YPTSpeed()
             │  ├─ YPanRot() / YTiltRot() (无极旋转)
             │  └─ YLampAngle() / SetLiftZ()
             └─ ASuperStageLight
                 ├─ SetLightingDefaultValue() / SetLightingIntensity()
                 ├─ SetLightingStrobe() / SetLightingColorRGB()
                 ├─ SetLightingColorWheel() / SetLightingColorTemperature()
                 ├─ SetLightingZoom() / SetLightingFrost() / SetLightingIris()
                 ├─ SetBeamGobo1/2/3() / SetBeamPrism() / SetBeamFocus()
                 ├─ SetBeamCutting() / SetBeamCuttingRotate()
                 ├─ SetEffect*() / SetMatrix*()
                 └─ 80+ BlueprintCallable 控制函数
```

### 选择基类

| 灯具类型 | 推荐基类 | 理由 |
|---------|---------|------|
| 标准电脑灯（Beam/Spot/Wash/Profile） | `ASuperStageLight` | 提供全套控制函数 |
| 摄像机 | `ASuperLightBase` | 需要 Pan/Tilt，不需要灯光组件 |
| 升降设备 | `ASuperDmxActorBase` | 仅需 DMX 读取 |
| LED 灯带/面板 | `ASuperDmxActorBase` | 自定义效果控制 |
| 激光 | `ASuperDmxActorBase` | 自定义激光组件 |

---

## 5. 组件体系

### 5.1 组件选择指南

| 组件 | 渲染方式 | 适用场景 |
|------|---------|---------|
| `USuperBeamComponent` | SpotLight + 光束网格 + 光斑材质 | Beam/Spot 灯（可见光柱） |
| `USuperCuttingComponent` | 同上 + 切割材质 | Profile 灯（需要切割） |
| `USuperSpotComponent` | SpotLight + 光斑材质 | 辅光/补光（无可见光柱） |
| `USuperRectComponent` | RectLight + 光斑材质 | LED 面板/Wash 灯 |
| `USuperEffectComponent` | 静态网格 + 效果材质 | LED 效果灯/灯带 |
| `USuperMatrixComponent` | 静态网格 + 分段材质 | 矩阵灯/像素灯 |
| `USuperLiftComponent` | 无渲染（仅运动） | 升降控制 |
| `USuperLaserProComponent` | 程序化网格 | Beyond 激光 |

### 5.2 组件创建模式

**C++ 构造函数中创建**：
```cpp
// 在灯具 Actor 构造函数中
SuperLighting = CreateDefaultSubobject<USuperBeamComponent>(TEXT("SuperLighting"));
SuperLighting->SetupAttachment(SceneHard);
SuperLighting->StaticMeshLens = LoadObject<UStaticMesh>(...);
```

**蓝图中创建**：
1. 在蓝图编辑器的 Components 面板添加组件
2. 将组件附加到正确的父级（通常是 SceneHard）
3. 在 Details 面板设置默认参数

### 5.3 USuperLightingComponent 详解（基础灯光组件）

所有灯光组件的**基类**，提供镜片网格、光斑材质、频闪计算。

**继承链**：`USceneComponent → USuperLightingComponent`

**关键属性**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `StaticMeshLens` | `UStaticMesh*` | 镜片网格模型 |
| `LensTransform` | `FTransform` | 镜片变换偏移 |
| `Angle` | `float` | 光斑角度 |
| `DimmerCurveExponent` | `float` | 亮度响应曲线（1.0=线性，2.0=平方推荐） |
| `ComponentDimmer` | `float` | 组件级亮度分控 [0,1] |

**初始化函数**（在 OnConstruction / BeginPlay 中调用）：

```cpp
// 初始化材质（内部自动调用，通常不需手动调用）
SetLightingMaterial();

// 设置默认灯光参数（必须调用！）
SetLightingDefaultValue(
    float NewMaxLightDistance,       // 光照最大距离(cm)
    float NewLightMaxIntensity,      // 最大亮度
    float VolumetricScattering,      // 体积散射强度
    bool  LightShadow,              // 是否启用阴影
    float NewLightSpotIntensity,     // 光斑强度（默认100）
    float LensIntensity              // 镜片发光强度（默认1）
);
```

**控制函数**（在 Tick / SuperDMXTick 中调用）：

```cpp
SetLightingIntensity(float NewIntensity);          // 亮度 [0,1]
SetLightingStrobe(float NewStrobe);                // 频闪速度 [0,1]
SetLightingStrobeMode(float NewStrobeMode);        // 频闪模式（0-7枚举）
SetRandomSeed(float NewSeed);                      // 随机种子（每灯不同）
SetLightingColor(FLinearColor NewColor);            // 颜色
SetLightingZoom(float NewZoom);                    // 变焦 [0,1]
SetLightingFrost(float NewFrost);                  // 雾化 [0,1]
SetLightingIris(float NewIris);                    // 光圈 [0,1]（1=全开）
SetLightingTexture(UTexture2D* Texture);           // 设置纹理
SetLightingRotate(float Rotate, float Infinite);   // 旋转
SetLightingVisibility(bool bVisible);              // 可见性
SetComponentDimmer(float NewDimmer);               // 组件亮度分控
```

**亮度计算公式**：
```
FinalIntensity = pow(DMXDimmer, DimmerCurveExponent) × ComponentDimmer × StrobeMultiplier
```

### 5.4 USuperSpotComponent 详解（聚光灯组件）

继承 `USuperLightingComponent`，增加 UE SpotLight 真实光照。

**继承链**：`USuperLightingComponent → USuperSpotComponent`

**新增属性**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `YSpotLight` | `USilentSpotLightComponent*` | 内部 SpotLight（自动创建） |
| `ZoomRange` | `FVector2D` | 光束角范围（默认 1°~10°） |
| `bDisableLightFunction` | `bool` | 禁用光照函数（频闪改用 Tick 控制） |

**Zoom 映射**：`DMX[0,1] → Lerp(ZoomRange.X, ZoomRange.Y)` → SpotLight OuterConeAngle

**适用灯具**：Wash 灯、辅光、无可见光柱的灯具

**蓝图初始化**：
```
OnConstruction / BeginPlay:
    SetLightingDefaultValue(MaxDistance, MaxIntensity, Scatter, Shadow, SpotIntensity, LensIntensity)
```

**蓝图 Tick 控制**：
```
SuperDMXTick:
    SetLightingIntensity(DmxDimmer)
    SetLightingColor(Color)
    SetLightingZoom(DmxZoom)
    SetLightingFrost(DmxFrost)
```

### 5.5 USuperBeamComponent 详解（光束组件）

继承 `USuperSpotComponent`，增加可见体积光束（光柱）。

**继承链**：`USuperSpotComponent → USuperBeamComponent`

**新增属性**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `YStaticMeshBeam` | `UStaticMeshComponent*` | 光束圆锥网格 |
| `BeamMaterial` / `DynamicMaterialBeam` | 材质 | 光束体积材质 |

**初始化**（OnConstruction / BeginPlay）：
```cpp
// 先调用基类初始化
SetLightingDefaultValue(MaxDist, MaxIntensity, Scatter, Shadow, SpotIntensity, LensIntensity);

// 然后设置光束参数
SetBeamDefaultValue(
    float MaxLightDistance,    // 光束长度(cm)
    float MaxIntensity,        // 最大亮度
    float AtmosphericDensity,  // 大气密度（光束衰减，0.03推荐）
    float LensRadius,          // 镜头半径（光斑大小，默认10）
    float BeamQuality,         // 光束质量（步进数，越高越平滑）
    float FogInfluence,        // 雾效强度
    float FogSpeed,            // 雾效流速
    float BeamIntensity        // 光束亮度倍率（默认1）
);
```

**光束专属控制**：
```cpp
SetBeamTexture(Texture, NumGobos, GoboIndex, GoboSpeed, ShakeSpeed);  // Gobo 纹理
SetBeamPrism(Facets, Radius, Scale, Rotation, RotSpeed);              // 棱镜
SetBeamFocus(Focus);                                                   // 调焦 [0=清晰, 1=模糊]
SetColorTexture(Texture, NumColors, Index, Speed);                     // 颜色轮 1
SetColorTexture2(...);                                                  // 颜色轮 2
SetColorTexture3(...);                                                  // 颜色轮 3
```

**适用灯具**：Beam 灯、Spot 灯（需要可见光柱的灯具）

### 5.6 USuperCuttingComponent 详解（切割组件）

继承 `USuperBeamComponent`，增加四叶切割（光闸）。

**继承链**：`USuperBeamComponent → USuperCuttingComponent`

**切割控制**（8 个参数，对应 4 条叶片的两端位置）：
```cpp
SetCuttingValue(
    float UpperLeftY,     // 左上 Y
    float LowerLeftY,     // 左下 Y
    float TopRightY,      // 右上 Y
    float BottomRightY,   // 右下 Y
    float TopLeftX,       // 上左 X
    float BottomLeftX,    // 下左 X
    float TopRightX,      // 上右 X
    float BottomRightX    // 下右 X
);

// 切割系统旋转（与光斑旋转同步）
SetLightingRotate(float Rotate, float InfiniteRotation);
```

**切割通道命名约定**：`A1, A2, A3, A4`（A 侧）+ `B1, B2, B3, B4`（B 侧）+ `ShaperRot`

**适用灯具**：Profile 灯（需要光闸切割的灯具）

### 5.7 USuperRectComponent 详解（矩形面光组件）

继承 `USuperLightingComponent`，使用 UE RectLight 实现面光源。

**继承链**：`USuperLightingComponent → USuperRectComponent`

**新增属性**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `YRectLight` | `USilentRectLightComponent*` | 内部矩形光源 |
| `SourceWidth` | `float` | 面光宽度 (cm) |
| `SourceHeight` | `float` | 面光高度 (cm) |
| `BarnDoorAngle` | `float` | 挡光板角度 (0-80°) |
| `BarnDoorLength` | `float` | 挡光板长度 (cm) |

**适用灯具**：LED 面板灯、Wash 灯（柔光效果）、面光源

**初始化与控制**（与 `USuperLightingComponent` 一致）：
```
OnConstruction / BeginPlay:
    SetLightingDefaultValue(MaxDist, MaxIntensity, Scatter, Shadow)

SuperDMXTick:
    SetLightingIntensity(DmxDimmer)
    SetLightingColor(Color)
```

### 5.8 USuperEffectComponent 详解（效果组件）

独立的效果平面组件，使用材质驱动的 LED 效果。

**继承链**：`USceneComponent → USuperEffectComponent`

**关键属性**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `YStaticMeshEffect` | `UStaticMeshComponent*` | 效果展示网格 |
| `StaticMeshEffect` | `UStaticMesh*` | 效果网格模型（蓝图中设置） |
| `EffectMaterialNo` | `int` | 材质编号 |
| `bTransparent` | `bool` | 是否使用透明材质 |
| `EffectTransform` | `FTransform` | 效果组件变换 |
| `ComponentDimmer` | `float` | 组件级亮度分控 |

**初始化**（OnConstruction / BeginPlay）：
```cpp
SetEffectMaterial(float NewIntensity = 1.0f);  // 创建动态材质并绑定
```

**控制函数**：
```cpp
// 分别控制（推荐精细控制）
SetEffectIntensity(float Intensity);
SetEffectStrobe(float Strobe);
SetEffectStrobeMode(float StrobeMode);
SetEffectColor(FLinearColor Color);
SetEffectControl(float Effect, float Speed, float Width);  // 效果选择+速度+宽度
SetComponentDimmer(float NewDimmer);

// 一次性控制（便捷方式）
SetEffectsControl(
    float Intensity,       // 亮度
    float Strobe,          // 频闪
    FLinearColor Color,    // 颜色
    float Direction,       // 方向
    float Effect,          // 效果索引（0-10）
    float Speed,           // 速度（-10 到 10）
    float Width            // 宽度（0-10）
);
```

**效果索引映射**（Effect 参数 0-10）：
| 值 | 效果 |
|----|------|
| 0 | 无效果（纯色） |
| 1-10 | 内置 GPU 效果（流水、追逐、呼吸等） |

**适用灯具**：LED 灯条、效果灯、装饰灯

### 5.9 USuperMatrixComponent 详解（矩阵组件）

分段控制的矩阵像素组件，每段独立颜色。

**继承链**：`USceneComponent → USuperMatrixComponent`

**关键属性**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `SegCount` | `int32` | 分段数量（≤200） |
| `bUseUAxis` | `bool` | true=按 U 方向切分, false=按 V 方向 |
| `StaticMeshMatrix` | `UStaticMesh*` | 矩阵网格模型 |
| `bTransparent` | `bool` | 透明材质 |
| `MatrixTransform` | `FTransform` | 矩阵变换 |
| `SegmentLightType` | `ESegmentLightType` | 分段光源类型（None/PointLight/SpotLight） |
| `SegmentLightIntensity` | `float` | 分段光源强度 |
| `SegmentLightRadius` | `float` | 分段光源衰减半径 |
| `ComponentDimmer` | `float` | 组件级亮度分控 |

**初始化**（OnConstruction / BeginPlay）：
```cpp
SetMatrixMaterial(float NewIntensity = 1.0f);  // 创建材质 + 重建分段光源
```

**控制函数**：
```cpp
SetSegmentColor(int32 Index, FLinearColor RGB);  // 设置单像素颜色
SetMatrixStrobe(float Strobe);                    // 矩阵频闪
SetMatrixStrobeMode(float StrobeMode);            // 矩阵频闪模式
SetMatrixIntensity(float Intensity);              // 矩阵亮度
SetComponentDimmer(float NewDimmer);              // 组件亮度分控

// 光源管理
RebuildSegmentLights();     // 类型或数量变化时重建
DestroyAllSegmentLights();  // 销毁所有分段光源
UpdateLightParameters();    // 更新光源参数（无需重建）
```

**矩阵像素颜色设置模式**（在 Actor 层调用）：
```cpp
// 单像素设置（通过 Index 指定）
SetMatrixColorRGBSingle(DmxR, DmxG, DmxB, Index, MatrixComp);

// 多像素批量设置（GET_SUPER_DMX_MATRIX_VALUE 宏遍历）
SetMatrixColorRGBMultiple(DmxR, DmxG, DmxB, MatrixComp);
```

### 5.10 USuperLiftComponent 详解（升降组件）

简洁的 Z 轴升降控制组件，由父 Actor 主动调用。

**继承链**：`USceneComponent → USuperLiftComponent`

**控制函数**：
```cpp
void SetLiftZ(
    float InPosZ,       // DMX 归一化值 [0,1]
    float LiftRange,    // 升降范围 (cm)
    float LiftSpeed     // 插值速度 (0-10)
);
```

**内部运算**：`DMX[0,1] → Lerp(0, LiftRange) → FInterpTo 平滑 → SetRelativeLocation.Z`

**用法**（蓝图/C++）：
```
SuperDMXTick:
    SetLiftZ(SuperLift, DmxLiftZ)  // Actor 层函数包装
```

### 5.11 USuperLaserProComponent 详解（激光 Pro 组件）

基于程序化网格实现点数据驱动的激光线可视化。

**继承链**：`USceneComponent → USuperLaserProComponent`

**关键属性**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `BeamLength` | `float` | 激光投射距离 (100-50000 cm) |
| `ProjectionAngle` | `float` | 投射角度范围 (1-90°) |
| `LaserWidth` | `float` | 激光线宽度 (0.1-20 cm) |
| `CoreSharpness` | `float` | 核心锐度 [1,10] |
| `DepthFade` | `float` | 深度衰减 [0,1] |
| `Dim` | `float` | 自发光强度 [1,10] |
| `OpacityScale` | `float` | 透明度缩放 [0,1] |
| `FogSpeed` / `FogInfluence` | `float` | 烟雾效果 |
| `SpotDimmer` | `float` | 光斑亮度 [0,5] |
| `bEnableCollision` | `bool` | 碰撞遮挡检测 |
| `ComponentDimmer` | `float` | 组件级亮度分控 |

**控制函数**：
```cpp
SetLaserPoints(const TArray<FLaserPoint>& Points);  // 设置激光点数据（每帧）
RebuildMesh();                                        // 强制重建网格
SetLaserVisibility(bool bVisible);                   // 可见性
SetComponentDimmer(float NewDimmer);                 // 亮度分控
```

**点数据格式** (`FLaserPoint`)：
```
X, Y: 位置 [-1,1]（归一化到 ProjectionAngle 范围）
R, G, B: 颜色 [0,1]
Z: 光束标记（>0 表示光束点）
```

---

## 6. C++ 灯具开发流程

### 6.1 步骤总览

1. **创建灯库资产** — 定义 DMX 通道映射
2. **创建 C++ Actor 类** — 继承合适的基类
3. **添加组件** — 在构造函数中创建灯光/效果组件
4. **实现初始化** — OnConstruction + BeginPlay 中调用 SetLightingDefaultValue（各一次）
5. **实现 DMX 控制** — Tick 中调用控制函数
6. **创建蓝图子类** — 设置模型和材质
7. **测试** — 在场景中放置并连接 DMX

### 6.2 C++ Actor 骨架

```cpp
// MyCustomLight.h
#pragma once
#include "CoreMinimal.h"
#include "LightActor/SuperStageLight.h"
#include "MyCustomLight.generated.h"

UCLASS(meta = (DisplayName = "MyCustomLight"))
class SUPERSTAGE_API AMyCustomLight : public ASuperStageLight
{
    GENERATED_BODY()
    
public:
    AMyCustomLight();

protected:
    virtual void BeginPlay() override;
    virtual void Tick(float DeltaTime) override;

    // DMX 属性引用（在蓝图子类中配置）
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "E.DMXMapping")
    FSuperDMXAttribute DmxDimmer;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "E.DMXMapping")
    FSuperDMXAttribute DmxStrobe;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "E.DMXMapping")
    FSuperDMXAttribute DmxColorR;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "E.DMXMapping")
    FSuperDMXAttribute DmxColorG;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "E.DMXMapping")
    FSuperDMXAttribute DmxColorB;
    
    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "E.DMXMapping")
    FSuperDMXAttribute DmxZoom;
};
```

```cpp
// MyCustomLight.cpp
#include "MyCustomLight.h"

AMyCustomLight::AMyCustomLight()
{
    // 组件在 ASuperStageLight 构造函数中已创建
    // 此处可添加自定义组件或修改默认值
    
    // 启用需要的通道开关
    bDimmer = true;
    bStrobe = true;
    bColor = true;
    bZoom = true;
}

void AMyCustomLight::OnConstruction(const FTransform& Transform)
{
    Super::OnConstruction(Transform);
    
    // 初始化光束组件（编辑器中放置时即执行）
    SetLightingDefaultValue(SuperLighting);
}

void AMyCustomLight::BeginPlay()
{
    Super::BeginPlay();
    
    // 运行时再初始化一次（确保材质正确）
    SetLightingDefaultValue(SuperLighting);
}

void AMyCustomLight::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);
    
    // DMX 控制（不要在这里调用 SetLightingDefaultValue）
    SetLightingIntensity(DmxDimmer);
    SetLightingStrobe(DmxStrobe);
    SetLightingColorRGB(DmxColorR, DmxColorG, DmxColorB);
    SetLightingZoom(DmxZoom);
}
```

### 6.3 通道开关系统

`ASuperStageLight` 使用 `bChannelEdit` + 各通道开关控制编辑器中属性的可见性：

```cpp
// 通道开关 — EditDefaultsOnly，在蓝图子类的 Class Defaults 中设置
bChannelEdit = true;   // 总开关（必须为 true 才能编辑其他开关）
bDimmer = true;        // 亮度通道
bStrobe = true;        // 频闪通道
bColor = true;         // RGB 颜色
bColorWheel = true;    // 颜色轮
bZoom = true;          // 变焦
bFrost = true;         // 雾化
bIris = true;          // 光圈
bGobo1 = true;         // 图案轮1
bGobo2 = true;         // 图案轮2
bGobo_Rot = true;      // 图案旋转
bPrism1 = true;        // 棱镜1
bPrism_Rot = true;     // 棱镜旋转
bEffect = true;        // 效果
bCuttingChannel = true;// 切割
bPan = true;           // Pan
bTilt = true;          // Tilt
bPTSpeed = true;       // PT速度
bPanRot = true;        // Pan无极旋转
bTiltRot = true;       // Tilt无极旋转
bInPosZ = true;        // 升降
```

---

## 7. 蓝图灯具开发流程

### 7.1 最简蓝图灯具（5 步）

**步骤 1：创建灯库**
1. Content Browser → 右键 → SuperStage → Super灯库
2. 双击打开，添加属性（Dimmer ch1, Pan ch3/4, Tilt ch5/6, R ch7, G ch8, B ch9, Zoom ch10）
3. 保存

**步骤 2：创建蓝图 Actor**
1. Content Browser → 右键 → Blueprint Class
2. 选择父类 `SuperStageLight`
3. 命名为 `BP_MyWashLight`

**步骤 3：设置组件与模型**
1. 打开蓝图编辑器
2. 在 Components 面板中选择已有组件
3. 设置 SM_Hook/SM_Base/SM_Rocker/SM_Hard 的 Static Mesh
4. 选择 SuperLighting 组件（USuperBeamComponent 或其子类），设置镜头模型等

**步骤 4：配置通道开关与 DMX 映射**
1. 打开 Class Defaults
2. `ChannelEdit = true`
3. 勾选需要的通道：Dimmer、Pan、Tilt、Color、Zoom 等
4. 设置 `灯库` 属性指向步骤 1 创建的灯库资产

**步骤 5：实现 SuperDMXTick**
1. 在 Event Graph 中添加 `Super DMX Tick` 事件
2. 连线调用控制函数：

```
Event OnConstruction
    └─ SetLightingDefaultValue (SuperLighting)  ← 编辑器放置时初始化

Event BeginPlay
    └─ SetLightingDefaultValue (SuperLighting)  ← 运行时初始化

Event SuperDMXTick
    ├─ Pan (DmxPan)
    ├─ Tilt (DmxTilt)
    │
    ├─ SetLightingIntensity (DmxDimmer)
    ├─ SetLightingStrobe (DmxStrobe)
    ├─ SetLightingColorRGB (DmxR, DmxG, DmxB)
    └─ SetLightingZoom (DmxZoom)
```

### 7.2 蓝图中的 FSuperDMXAttribute 配置

在蓝图中，每个 DMX 属性变量需要配置三个字段：

- **InstanceIndex**: 模块索引（通常为 0）
- **AttribName**: 属性名（必须与灯库中完全一致）
- **DMXChannelType**: 精度选择（8-Bit / 16-Bit / 24-Bit）

**示例配置**（在蓝图的变量默认值中设置）：

```
DmxDimmer:
  InstanceIndex = 0
  AttribName = "Dimmer"
  DMXChannelType = Fine (16-bit)

DmxPan:
  InstanceIndex = 0
  AttribName = "Pan"
  DMXChannelType = Fine (16-bit)

DmxColorR:
  InstanceIndex = 0
  AttribName = "ColorR"
  DMXChannelType = Conarse (8-bit)
```

### 7.3 蓝图初始化模式

推荐在 `OnConstruction`（构造脚本）和 `BeginPlay`（游戏开始）中各调用一次初始化函数：

```
Event OnConstruction
    ├─ SetLightingDefaultValue(SuperLighting)    ← 编辑器放置/属性修改时
    ├─ SetLightingDefaultTexture(GoboTexture)    ← 可选
    └─ SetEffectDefaultValue(SuperEffect)        ← 如果有效果组件

Event BeginPlay
    ├─ SetLightingDefaultValue(SuperLighting)    ← 运行时
    ├─ SetLightingDefaultTexture(GoboTexture)    ← 可选
    └─ SetEffectDefaultValue(SuperEffect)        ← 如果有效果组件
```

**关键**：
- `SetLightingDefaultValue` 必须在 `SuperDMXTick` 中的控制函数之前调用，否则材质未初始化
- 在 `OnConstruction` 中调用可确保编辑器中放置灯具时即完成初始化
- **不要**在 `Tick` 或 `SuperDMXTick` 中重复调用初始化函数

---

## 8. DMX 通道读取 API

### 8.1 高层接口（推荐）

这些函数自动处理归一化、多精度、权限检查：

```cpp
// 读取归一化值 [0,1]（自动根据 DMXChannelType 选择精度）
void GetSuperDmxAttributeValue(const FSuperDMXAttribute& DmxAttribute, float& InOutDefault) const;

// 读取归一化值（不做 [0,1] 转换）
void GetSuperDmxAttributeValueNoConversion(const FSuperDMXAttribute& DmxAttribute, float& InOutDefault) const;

// 读取 RGB 颜色
void GetSuperDMXColorValue(const FSuperDMXAttribute& DmxRed, const FSuperDMXAttribute& DmxGreen, 
    const FSuperDMXAttribute& DmxBlue, FLinearColor& OutColor) const;

// 读取 DMX 原始值 [0-255]
void GetSuperDmxAttributeRawValue(const FSuperDMXAttribute& DmxAttribute, int32& OutRawValue) const;
```

### 8.2 按实例索引读取（中层）

```cpp
// 8-bit [0..255]
int32 GetAttributeRaw8ByIndex(int32 InstanceIndex, FName AttribName, int32 DefaultValue = 0) const;

// 16-bit [0..65535]
int32 GetAttributeRaw16ByIndex(int32 InstanceIndex, FName AttribName, int32 DefaultValue = 0) const;

// 24-bit [0..16777215]
int32 GetAttributeRaw24ByIndex(int32 InstanceIndex, FName AttribName, int32 DefaultValue = 0) const;

// 查询位深
int32 GetAttributeBitDepthByIndex(int32 InstanceIndex, FName AttribName) const;
```

### 8.3 原始通道读取（底层）

```cpp
// 直接按通道偏移读取 8-bit 值（1基）
float GetChannelValue(int32 Address = 1, float DefaultValue = 1) const;
```

### 8.4 矩阵批量读取

```cpp
// 遍历所有模块的同名属性（8-bit）
void GetMatrixAttributeRaw(FName AttribName, TArray<float>& OutValues, 
    float DefaultValue = 0, bool bSortByPatch = true) const;

// 遍历所有模块的同名属性（16-bit）
void GetMatrixAttributeRaw16(FName AttribName, TArray<float>& OutValues, 
    float DefaultValue = 0, bool bSortByPatch = true) const;
```

### 8.5 矩阵宏

C++ 中推荐使用宏简化矩阵遍历：

```cpp
// 基础矩阵遍历（提供 LightIndex 和 DefaultValue）
GET_SUPER_DMX_MATRIX_VALUE(DmxAttribute, DefaultValue,
{
    // LightIndex: 当前像素索引
    // DefaultValue: 当前像素的归一化值
    SomeComponent->SetIntensity(LightIndex, DefaultValue);
});

// 带子属性的矩阵遍历（额外提供 DmxRawValue, AttrDef, SubAttr）
GET_SUPER_DMX_MATRIX_VALUE_WITH_SUBATTR(DmxAttribute, DefaultValue,
{
    // DmxRawValue: DMX 原始值 [0-255]
    // AttrDef: 属性定义指针
    // SubAttr: 匹配的子属性指针
    if (SubAttr && SubAttr->StrobeMode == EStrobeMode::Linear)
    {
        // 处理频闪
    }
});

// RGB 矩阵遍历
GET_SUPER_DMX_MATRIX_RGB(DmxR, DmxG, DmxB, OutColor,
{
    // LightIndex: 当前像素索引
    // OutColor: FLinearColor
    MatrixComp->SetSegmentColor(LightIndex, OutColor);
});
```

### 8.6 子属性查询

```cpp
// 查找属性定义
const FSuperDMXAttributeDef* AttrDef = FindAttributeDef(InstanceIndex, AttribName);

// 根据 DMX 原始值查找匹配的子属性
const FSubAttribute* SubAttr = AttrDef->FindSubAttribute(DmxRawValue);

// 根据 DMX 原始值查找匹配的槽位
const FChannelSet* ChannelSet = AttrDef->FindChannelSet(DmxRawValue);

// 子属性内归一化
float NormPos = SubAttr->GetNormalizedPosition(DmxRawValue);

// 子属性内物理值映射
float Physical = SubAttr->GetMappedPhysical(DmxRawValue);
```

---

## 9. 颜色系统详解

### 9.1 直控 RGB

```cpp
// 单灯
SetLightingColorRGB(FSuperDMXAttribute DmxR, FSuperDMXAttribute DmxG, FSuperDMXAttribute DmxB);

// 矩阵（每像素独立）
SetLightingColorRGBMatrix(FSuperDMXAttribute DmxR, FSuperDMXAttribute DmxG, FSuperDMXAttribute DmxB);
```

### 9.2 RGBW（白色通道叠加）

```cpp
SetLightingColorRGBW(DmxR, DmxG, DmxB, DmxW);
SetLightingColorRGBWMatrix(DmxR, DmxG, DmxB, DmxW);
```

白色通道叠加到 RGB：`FinalColor = RGB + W * White`

### 9.3 HSV / HS

```cpp
SetLightingColorHSV(DmxH, DmxS, DmxV);  // 色相/饱和度/明度
SetLightingColorHS(DmxH, DmxS);          // 色相/饱和度（明度由 Dimmer 控制）
```

### 9.4 颜色轮（Color Wheel）

```cpp
// 单颜色轮
SetLightingColorWheel(DmxAttribute, ColorAtlas);

// 颜色轮 + CMY 混色
SetLightingColorWheelAndCMY(DmxWheel, DmxC, DmxM, DmxY, ColorAtlas);

// 双/三颜色轮
SetLightingColorWheel2(Dmx1, Dmx2, Atlas1, Atlas2);
SetLightingColorWheel3(Dmx1, Dmx2, Dmx3, Atlas1, Atlas2, Atlas3);

// 颜色轮 + CMY + CTO
SetLightingColorWheel2AndCMYAndCTO(Dmx1, Dmx2, DmxC, DmxM, DmxY, DmxCTO, 
    WarmTemp, CoolTemp, Atlas1, Atlas2);
```

**颜色轮工作原理**：
- 灯库中颜色类属性的子属性定义各颜色槽位
- 每个 `FChannelSet` 包含 `Color` 和 `ColorIndex`
- Actor 根据 DMX 原始值查找匹配的 ChannelSet
- 使用 ColorAtlas 纹理 + ColorIndex 实现精确的颜色混合

### 9.5 色温（Color Temperature）

```cpp
// 单通道色温
SetLightingColorTemperature(DmxAttribute, WarmTemperature=1700, CoolTemperature=12000);

// 冷暖双通道
SetLightingCoolWarmMix(DmxCool, DmxWarm, CoolTemperature=6500, WarmTemperature=3200);
```

### 9.6 动态多通道混色

```cpp
// 支持任意颜色通道组合（RGBW/RGBA/RGBL/RGBLAM 等）
TArray<FSuperColorChannel> Channels;
Channels.Add(FSuperColorChannel(DmxR, FLinearColor::Red));
Channels.Add(FSuperColorChannel(DmxG, FLinearColor::Green));
Channels.Add(FSuperColorChannel(DmxB, FLinearColor::Blue));
Channels.Add(FSuperColorChannel(DmxW, FLinearColor::White));
Channels.Add(FSuperColorChannel(DmxAmber, FLinearColor(1, 0.75, 0))); // 琥珀

SetLightingColorMix(Channels);
SetLightingColorMixWithCTO(Channels, DmxCTO, CoolTemp, WarmTemp);
```

### 9.7 Spot 辅光颜色

所有主光源颜色函数都有对应的 Spot 辅光版本：

```cpp
SetSpotColorRGB(DmxR, DmxG, DmxB);
SetSpotColorRGBW(DmxR, DmxG, DmxB, DmxW);
SetSpotCoolWarmMix(DmxCool, DmxWarm, CoolTemp, WarmTemp);
// ... 以及 Matrix 版本
```

---

## 10. 图案盘、颜色盘与棱镜系统

### 10.1 核心数据结构：FChannelSet

`FChannelSet`（槽位集合）是图案盘、颜色盘、棱镜三大系统的**共用数据载体**，定义在灯库属性的子属性（`FSubAttribute`）内。

```
FSuperDMXAttributeDef (属性)
 └─ SubAttributes[] (子属性)
     └─ ChannelSets[] (槽位列表)
         ├─ Name          — 槽位名称（如 "Open", "Gobo1", "Red"）
         ├─ DmxMin/DmxMax — DMX 范围
         ├─ PhysicalRange — 物理值映射范围
         ├─ GoboMode      — 图案模式（Static/Scrolling/Shake）
         ├─ Texture       — 图案纹理（UTexture2D*）
         ├─ Color         — 颜色值（FLinearColor）
         ├─ ColorIndex    — 颜色索引（用于 Color Wheel 定位）
         ├─ PrismFacets   — 棱镜面数（0=关闭，3/5/6/8 等）
         ├─ PrismRadius   — 棱镜半径（0.0~1.0）
         └─ PrismScale    — 棱镜缩放（0.01~1.0）
```

**每种属性只使用其中一部分字段**：

| 属性类型 | 使用的 FChannelSet 字段 | 不使用的字段 |
|---------|----------------------|------------|
| **图案 (Gobo)** | GoboMode, Texture, DmxMin/Max | Color, PrismFacets/Radius/Scale |
| **颜色 (Color)** | Color, ColorIndex, DmxMin/Max | Texture, GoboMode, PrismFacets |
| **棱镜 (Prism)** | PrismFacets, PrismRadius, PrismScale, DmxMin/Max | Texture, Color |

---

### 10.2 图案盘（Gobo Wheel）

#### 10.2.1 灯库中的图案配置

每个图案占一个 `FChannelSet` 槽位，需要配置：

```
属性: Gobo1, 分类: Gobo, Coarse: 7
 └─ SubAttribute[0]: DmxMin=0, DmxMax=127 (静态图案选择)
     ├─ ChannelSet[0]: Name="Open",   DmxMin=0,  DmxMax=9,  GoboMode=Static, Texture=null
     ├─ ChannelSet[1]: Name="Gobo1",  DmxMin=10, DmxMax=19, GoboMode=Static, Texture=T_Gobo_Star
     ├─ ChannelSet[2]: Name="Gobo2",  DmxMin=20, DmxMax=29, GoboMode=Static, Texture=T_Gobo_Circle
     ├─ ChannelSet[3]: Name="Gobo3",  DmxMin=30, DmxMax=39, GoboMode=Static, Texture=T_Gobo_Dot
     ├─ ChannelSet[4]: Name="Gobo4",  DmxMin=40, DmxMax=49, GoboMode=Static, Texture=T_Gobo_Line
     ├─ ChannelSet[5]: Name="Gobo5",  DmxMin=50, DmxMax=59, GoboMode=Static, Texture=T_Gobo_Flower
     └─ ChannelSet[6]: Name="Gobo6",  DmxMin=60, DmxMax=69, GoboMode=Static, Texture=T_Gobo_Cross
 └─ SubAttribute[1]: DmxMin=128, DmxMax=189 (抖动图案)
     ├─ ChannelSet[0]: Name="Shake1", DmxMin=128, DmxMax=138, GoboMode=Shake, Texture=T_Gobo_Star
     ├─ ...（抖动图案，GoboMode=Shake）
 └─ SubAttribute[2]: DmxMin=190, DmxMax=255 (流水模式)
     └─ (无 ChannelSet，PhysicalRange=(-1,1) 表示速度范围)
```

#### 10.2.2 EGoboMode 枚举

| 模式 | 行为 | ChannelSet 要求 |
|------|------|----------------|
| `Static` | 固定显示指定图案 | 每个槽位含 Texture |
| `Scrolling` | 图案连续滚动 | 无 ChannelSet，用 PhysicalRange 映射速度 |
| `Shake` | 图案在当前位置抖动 | 每个槽位含 Texture，DMX 值映射抖动频率(1-10Hz) |

#### 10.2.3 图案图集（Gobo Atlas）

运行时**不直接使用**单张 Gobo 纹理，而是将多张图案合并为一张**横向图集纹理**，传给材质。

**图集规格**：
- **尺寸**：4096 × 256 像素
- **排列**：横向等分，每个图案占 `4096 / N` 像素宽
- **格式**：BGRA8，无 Mipmap，Linear（sRGB=false）

**命名规则**：`LTA_{灯库名去除SL_前缀}_{属性名}.uasset`
- 例：灯库 `SL_Acme_XP-380Beam` + 属性 `Gobo1` → `LTA_Acme_XP-380Beam_Gobo1`

**构建方式**（推荐使用编辑器工具）：
1. 打开 **GOBO 图集构建器**（`SGoboAtlasBuilder`）
2. 选择灯库资产 → 输入属性名（如 `Gobo1`）→ 搜索
3. 工具自动从灯库的 `SubAttribute[0].ChannelSets` 提取所有 `Texture`
4. 点击"生成图集"→ 输出到灯库同级目录

**构建流程（内部）**：
```
1. 遍历 Modules[].AttributeDefs[]，找到 AttribName 匹配的属性
2. 取第一个 SubAttribute 的 ChannelSets[]
3. 提取每个 ChannelSet.Texture（跳过 null）
4. 将所有纹理缩放绘制到 4096×256 图集的对应位置
5. 创建 UTexture2D 资产并保存
```

**运行时数据流**：
```
DMX 原始值 → FindSubAttribute(DmxValue) → FindChannelSet(DmxValue)
  → 计算 GoboIndex（ChannelSet 在数组中的下标）
  → SuperBeam->SetBeamTexture(GoboAtlas, NumGobos, GoboIndex, Speed, ShakeSpeed)
  → 材质中 floor(GoboIndex) 定位图集中的子图案
```

#### 10.2.4 图案旋转（GoboRot）

图案旋转通道使用 `CalcGoboRotation` 计算，支持两种模式：

| 模式 | 条件 | 映射 |
|------|------|------|
| **静态旋转** | SubAttribute.PhysicalRange == (0,0) | DMX 归一化 × 360° |
| **无极旋转** | SubAttribute.PhysicalRange != (0,0) | DMX → Lerp(PhysicalRange.X, PhysicalRange.Y) → 转速 |

灯库配置示例：
```
属性: Gobo1Rot, 分类: Gobo, Coarse: 8
 └─ SubAttribute[0]: DmxMin=0,   DmxMax=127, PhysicalRange=(0,0)     → 静态旋转 0-360°
 └─ SubAttribute[1]: DmxMin=128, DmxMax=189, PhysicalRange=(-360,0)  → 逆时针无极旋转
 └─ SubAttribute[2]: DmxMin=190, DmxMax=194, PhysicalRange=(0,0)     → 停止
 └─ SubAttribute[3]: DmxMin=195, DmxMax=255, PhysicalRange=(0,360)   → 顺时针无极旋转
```

#### 10.2.5 蓝图中使用图案

**SuperDMXTick 中连线**：
```
// 单图案轮
SetBeamGobo1(DmxGobo1, DmxGobo1Rot, Gobo1Atlas)

// 双图案轮（自动优先级选择：非零值优先，全零用 Gobo1）
SetBeamGobo2(DmxGobo1, DmxGobo1Rot, DmxGobo2, DmxGobo2Rot, Atlas1, Atlas2)

// 三图案轮
SetBeamGobo3(DmxGobo1, DmxGobo1Rot, DmxGobo2, DmxGobo2Rot,
    DmxGobo3, DmxGobo3Rot, Atlas1, Atlas2, Atlas3)
```

**参数说明**：
- `DmxGoboN` — 图案选择通道（`FSuperDMXAttribute`，指向灯库中的 GoboN 属性）
- `DmxGoboNRot` — 图案旋转通道（`FSuperDMXAttribute`，指向 GoboNRot 属性）
- `GoboNAtlas` — 图案图集纹理（`UTexture2D*`，由 GoboAtlasBuilder 生成）

**蓝图变量设置**：在灯具蓝图的 Class Defaults 中创建 `UTexture2D*` 类型变量，命名为 `Gobo1Atlas` / `Gobo2Atlas`，赋值为构建器生成的图集资产。

---

### 10.3 颜色盘（Color Wheel）

#### 10.3.1 灯库中的颜色配置

每种颜色占一个 `FChannelSet` 槽位，需要配置 `Color` 字段：

```
属性: ColorWheel1, 分类: Color, Coarse: 9
 └─ SubAttribute[0]: DmxMin=0, DmxMax=127 (静态颜色选择)
     ├─ ChannelSet[0]: Name="Open",    DmxMin=0,  DmxMax=9,  Color=(1,1,1,1)        → 白
     ├─ ChannelSet[1]: Name="Red",     DmxMin=10, DmxMax=19, Color=(1,0,0,1)        → 红
     ├─ ChannelSet[2]: Name="Orange",  DmxMin=20, DmxMax=29, Color=(1,0.5,0,1)      → 橙
     ├─ ChannelSet[3]: Name="Yellow",  DmxMin=30, DmxMax=39, Color=(1,1,0,1)        → 黄
     ├─ ChannelSet[4]: Name="Green",   DmxMin=40, DmxMax=49, Color=(0,1,0,1)        → 绿
     ├─ ChannelSet[5]: Name="Cyan",    DmxMin=50, DmxMax=59, Color=(0,1,1,1)        → 青
     ├─ ChannelSet[6]: Name="Blue",    DmxMin=60, DmxMax=69, Color=(0,0,1,1)        → 蓝
     ├─ ChannelSet[7]: Name="Magenta", DmxMin=70, DmxMax=79, Color=(1,0,1,1)        → 品红
     └─ ChannelSet[8]: Name="CTO",     DmxMin=80, DmxMax=89, Color=(1,0.8,0.6,1)    → 暖白
 └─ SubAttribute[1]: DmxMin=128, DmxMax=189, PhysicalRange=(-1,1) (正向流水)
 └─ SubAttribute[2]: DmxMin=190, DmxMax=255, PhysicalRange=(1,-1) (反向流水)
```

#### 10.3.2 颜色图集（Color Atlas）

与 Gobo Atlas 类似，颜色盘也使用图集纹理。

**图集规格**：
- **尺寸**：256 × 16 像素（比 Gobo 图集小得多）
- **排列**：横向等分，每种颜色占 `256 / N` 像素宽
- **格式**：BGRA8，无 Mipmap，sRGB=true，点采样（TF_Nearest）
- **压缩**：TC_VectorDisplacementmap（无损）

**命名规则**：`CLA_{灯库名去除SL_前缀}_{属性名}.uasset`
- 例：灯库 `SL_Robe_T1Profile` + 属性 `ColorWheel1` → `CLA_Robe_T1Profile_ColorWheel1`

**构建方式**（推荐使用编辑器工具）：
1. 打开 **颜色图集构建器**（`SColorAtlasBuilder`）
2. 选择灯库资产 → 输入属性名（如 `ColorWheel1`）→ 搜索
3. 工具自动从灯库的 `SubAttribute[0].ChannelSets` 提取所有 `Color`
4. 预览颜色排列 → 点击"生成图集"

**构建流程（内部）**：
```
1. 遍历 Modules[].AttributeDefs[]，找到 AttribName 匹配的属性
2. 取第一个 SubAttribute 的 ChannelSets[]
3. 提取每个 ChannelSet.Color（FLinearColor）
4. 将每种颜色填充到 256×16 图集的对应列
5. 线性颜色 → sRGB FColor 转换
6. 创建 UTexture2D 资产并保存
```

**运行时数据流**：
```
DMX 原始值 → FindSubAttribute(DmxValue) → FindChannelSet(DmxValue)
  → 计算 ColorIndex（ChannelSet 在数组中的下标）
  → NumColors = SubAttributes[0].ChannelSets.Num()
  → SuperLighting->SetColorTexture(ColorAtlas, NumColors, ColorIndex, ColorSpeed)
  → 材质中按 ColorIndex 采样图集纹理
```

#### 10.3.3 颜色盘 vs RGB 混色

| 特性 | 颜色盘 (ColorWheel) | RGB 混色 |
|------|---------------------|---------|
| 通道数 | 1 通道 | 3-4 通道 (R/G/B/W) |
| 颜色数 | 有限（8-14 种） | 连续（1670 万色） |
| 原理 | 机械色轮/图集采样 | 电子混色 |
| 适用 | Beam/Spot/Profile | Wash/Matrix |
| 支持流水 | ✅（子属性 PhysicalRange 控制速度） | ❌ |

#### 10.3.4 蓝图中使用颜色盘

**SuperDMXTick 中连线**：
```
// 单颜色盘
SetLightingColorWheel(DmxColorWheel, ColorAtlas)

// 双颜色盘
SetLightingColorWheel2(DmxCW1, DmxCW2, ColorAtlas1, ColorAtlas2)

// 三颜色盘
SetLightingColorWheel3(DmxCW1, DmxCW2, DmxCW3, Atlas1, Atlas2, Atlas3)

// 颜色盘 + CMY
SetLightingColorWheelAndCMY(DmxCW, DmxC, DmxM, DmxY, ColorAtlas)

// 双颜色盘 + CMY
SetLightingColorWheel2AndCMY(DmxCW1, DmxCW2, DmxC, DmxM, DmxY, Atlas1, Atlas2)

// 双颜色盘 + CMY + CTO
SetLightingColorWheel2AndCMYAndCTO(DmxCW1, DmxCW2, DmxC, DmxM, DmxY, DmxCTO,
    WarmTemperature, CoolTemperature, Atlas1, Atlas2)

// 三颜色盘 + CMY
SetLightingColorWheel3AndCMY(DmxCW1, DmxCW2, DmxCW3, DmxC, DmxM, DmxY, Atlas1, Atlas2, Atlas3)
```

**多颜色盘优先级规则**：
- 有非零 DMX 值的颜色盘优先
- 多个非零时按参数顺序（CW1 > CW2 > CW3）优先
- 全部为零时使用第一个颜色盘（通常为白色/Open）

---

### 10.4 棱镜（Prism）

#### 10.4.1 灯库中的棱镜配置

棱镜属性使用 `FChannelSet` 的 `PrismFacets`、`PrismRadius`、`PrismScale` 字段：

```
属性: Prism1, 分类: Prism, Coarse: 12
 └─ SubAttribute[0]: DmxMin=0, DmxMax=127
     ├─ ChannelSet[0]: Name="Open",    DmxMin=0,  DmxMax=7,  PrismFacets=0  → 关闭
     ├─ ChannelSet[1]: Name="3-Facet", DmxMin=8,  DmxMax=39, PrismFacets=3, PrismRadius=0.3, PrismScale=0.15
     ├─ ChannelSet[2]: Name="5-Facet", DmxMin=40, DmxMax=69, PrismFacets=5, PrismRadius=0.25, PrismScale=0.12
     └─ ChannelSet[3]: Name="8-Facet", DmxMin=70, DmxMax=127,PrismFacets=8, PrismRadius=0.2,  PrismScale=0.10
```

#### 10.4.2 棱镜参数说明

| 参数 | 类型 | 范围 | 说明 |
|------|------|------|------|
| `PrismFacets` | int32 | 0~48 | 棱镜面数。0=关闭，3=三棱镜，5=五棱镜等 |
| `PrismRadius` | float | 0.0~1.0 | 棱镜效果半径。越大棱镜分离越明显 |
| `PrismScale` | float | 0.01~1.0 | 棱镜缩放。控制每个棱面的子图像大小 |

**视觉效果**：
- `PrismFacets=3, PrismRadius=0.3, PrismScale=0.15` — 标准三棱镜（最常见）
- `PrismFacets=5, PrismRadius=0.25, PrismScale=0.12` — 五棱镜（更密集）
- `PrismFacets=8, PrismRadius=0.2, PrismScale=0.10` — 八棱镜（密集光圈）
- `PrismFacets=0` — 无棱镜（Open）

#### 10.4.3 棱镜旋转（PrismRot）

棱镜旋转与图案旋转共用 `CalcGoboRotation` 逻辑：

```
属性: PrismRot, 分类: Prism, Coarse: 13
 └─ SubAttribute[0]: DmxMin=0,   DmxMax=127, PhysicalRange=(-360,0) → 逆时针
 └─ SubAttribute[1]: DmxMin=128, DmxMax=132, PhysicalRange=(0,0)    → 停止
 └─ SubAttribute[2]: DmxMin=133, DmxMax=255, PhysicalRange=(0,360)  → 顺时针
```

#### 10.4.4 运行时查找流程

```
DMX 原始值 → FindAttributeDef(InstanceIndex, "Prism1")
  → FindSubAttribute(DmxValue)
    → 遍历 ChannelSets，找到 Contains(DmxValue) 的槽位
      → 读取 PrismFacets / PrismRadius / PrismScale
        → SuperBeam->SetBeamPrism(Facets, Radius, Scale, Rotation, RotSpeed)
若 PrismFacets > 0 → 启用棱镜效果
若 PrismFacets == 0 → 关闭棱镜（Open）
```

**双棱镜优先级**：`SetBeamPrism(DmxPrism1, DmxPrism2, DmxPrismRot)`
- 先查 Prism1，若找到有效棱镜（Facets > 0）则使用
- 否则查 Prism2
- 都无效则关闭棱镜（Facets=0, Radius=0.3, Scale=0.15）

#### 10.4.5 蓝图中使用棱镜

**SuperDMXTick 中连线**：
```
SetBeamPrism(DmxPrism1, DmxPrism2, DmxPrismRot)
```

- `DmxPrism1` — 第一棱镜选择通道
- `DmxPrism2` — 第二棱镜选择通道（无则传空 FSuperDMXAttribute）
- `DmxPrismRot` — 棱镜旋转通道

**注意**：棱镜**不需要图集纹理**，参数直接从灯库的 `FChannelSet` 中读取。

---

### 10.5 图集构建工具

#### 10.5.1 GOBO 图集构建器（SGoboAtlasBuilder）

**打开方式**：UE 编辑器主菜单 → **Tools** → **GOBO Atlas Builder**（位于 SuperStageTools 分区）

**界面**：
```
┌──────────────────────────────────────┐
│ 灯库: [选择灯库资产]                    │
│ 属性名称: [Gobo1        ] [搜索]        │
│ 找到 7 个图案纹理                       │
│ ┌──────────────────────────────────┐ │
│ │ [0] T_Gobo_Open (256x256)       │ │
│ │ [1] T_Gobo_Star (256x256)       │ │
│ │ [2] T_Gobo_Circle (256x256)     │ │
│ │ ...                              │ │
│ └──────────────────────────────────┘ │
│                          [生成图集]    │
└──────────────────────────────────────┘
```

**操作流程**：
1. 选择灯库资产（`USuperFixtureLibrary`）
2. 输入图案属性名（如 `Gobo1`、`Gobo2`）
3. 点击"搜索"— 自动从灯库的第一个子属性提取所有纹理
4. 确认图案列表无误
5. 点击"生成图集"— 输出 `LTA_xxx.uasset` 到灯库同级目录

**输出纹理设置**：

| 属性 | 值 | 说明 |
|------|---|------|
| SRGB | false | 线性空间 |
| Compression | TC_Default | 默认压缩 |
| MipGenSettings | NoMipmaps | 无 Mip |
| LODGroup | UI | UI 组 |
| NeverStream | true | 常驻内存 |

#### 10.5.2 颜色图集构建器（SColorAtlasBuilder）

**打开方式**：UE 编辑器主菜单 → **Tools** → **Color Atlas Builder**（位于 SuperStageTools 分区）

**界面**：
```
┌──────────────────────────────────────┐
│ 灯库: [选择灯库资产]                    │
│ 属性名称: [ColorWheel1   ] [搜索]       │
│ 找到 9 个颜色 (图集尺寸: 256x16)        │
│ ┌──────────────────────────────────┐ │
│ │ [■][■][■][■][■][■][■][■][■]     │ │ ← 颜色预览色块
│ │  0  1  2  3  4  5  6  7  8      │ │
│ └──────────────────────────────────┘ │
│                          [生成图集]    │
└──────────────────────────────────────┘
```

**操作流程**：
1. 选择灯库资产
2. 输入颜色属性名（如 `ColorWheel1`、`ColorWheel2`）
3. 点击"搜索"— 自动提取所有 `FLinearColor`
4. 预览颜色排列
5. 点击"生成图集"— 输出 `CLA_xxx.uasset` 到灯库同级目录

**输出纹理设置**：

| 属性 | 值 | 说明 |
|------|---|------|
| SRGB | true | sRGB 空间 |
| Compression | VectorDisplacementmap | 无损 |
| MipGenSettings | NoMipmaps | 无 Mip |
| Filter | TF_Nearest | 点采样，防止颜色插值 |
| NeverStream | true | 常驻内存 |

#### 10.5.3 图集构建对比

| 对比项 | GOBO 图集 | 颜色图集 |
|-------|-----------|---------|
| 数据来源 | ChannelSet.Texture | ChannelSet.Color |
| 输出尺寸 | 4096 × 256 | 256 × 16 |
| 输出前缀 | `LTA_` | `CLA_` |
| 色彩空间 | Linear (sRGB=false) | sRGB (sRGB=true) |
| 压缩 | TC_Default | TC_VectorDisplacementmap (无损) |
| 采样模式 | 默认（双线性） | TF_Nearest（点采样） |

---

### 10.6 完整配置示例：Beam 灯的图案 + 颜色 + 棱镜

以 17 通道 Beam 灯为例，展示完整的图案/颜色/棱镜灯库配置和蓝图连线。

**灯库通道表**（图案/颜色/棱镜相关部分）：

| 通道 | 属性名 | 分类 | 说明 |
|------|--------|------|------|
| 9 | ColorWheel1 | Color | 颜色轮 |
| 10 | Gobo1 | Gobo | 图案轮1选择 |
| 11 | Gobo1Rot | Gobo | 图案轮1旋转 |
| 12 | Prism1 | Prism | 棱镜选择 |
| 13 | PrismRot | Prism | 棱镜旋转 |

**灯库配置步骤**：

1. **颜色轮**：编辑 `ColorWheel1` 属性 → 进入 SubAttribute[0] → 添加 9 个 ChannelSet，每个设置 `Color`（FLinearColor）和 `DmxMin/Max`
2. **图案轮**：编辑 `Gobo1` 属性 → 进入 SubAttribute[0] → 添加 7 个 ChannelSet，每个设置 `Texture` 和 `GoboMode=Static`，可选添加 SubAttribute[1] 设置 `GoboMode=Shake` 的抖动段
3. **棱镜**：编辑 `Prism1` 属性 → 进入 SubAttribute[0] → 添加 4 个 ChannelSet，ChannelSet[0] 设置 `PrismFacets=0`（Open），后续设置不同面数/半径/缩放

**图集构建**：
1. 打开 GOBO 图集构建器 → 选灯库 → 输入 `Gobo1` → 搜索 → 生成 → 得到 `LTA_xxx_Gobo1`
2. 打开颜色图集构建器 → 选灯库 → 输入 `ColorWheel1` → 搜索 → 生成 → 得到 `CLA_xxx_ColorWheel1`

**蓝图变量**：
- `Gobo1Atlas` (UTexture2D*) = `LTA_xxx_Gobo1`
- `ColorAtlas` (UTexture2D*) = `CLA_xxx_ColorWheel1`

**SuperDMXTick 连线**：
```
SetLightingColorWheel(DmxColorWheel1, ColorAtlas)      // 颜色轮
SetBeamGobo1(DmxGobo1, DmxGobo1Rot, Gobo1Atlas)        // 图案轮
SetBeamPrism(DmxPrism1, None, DmxPrismRot)              // 棱镜（无第二棱镜传空）
```

---

## 11. 切割系统

```cpp
// 四叶片切割（8 个 DMX 通道）
SetBeamCutting(DmxA1, DmxB1, DmxA2, DmxB2, DmxA3, DmxB3, DmxA4, DmxB4);

// 切割旋转（Shaper 旋转 + Gobo 旋转叠加）
SetBeamCuttingRotate(DmxShaperRot, DmxGoboRot);
```

**切割映射**：
- A 侧：`[0,1] → [0, 0.4]`（从打开到半关）
- B 侧：`[0,1] → [1, 0.6]`（从全开到部分关闭）
- 旋转叠加：`ShaperRot[-45°, 45°] + GoboRot[0°, 360°]`

**需要 `USuperCuttingComponent`**（继承自 `USuperBeamComponent`）。

---

## 12. 效果系统

### 12.1 效果组件初始化

```cpp
// 单效果
SetEffectDefaultValue(USuperEffectComponent* NewSuperEffect);

// 效果矩阵
SetEffectMatrixDefaultValue(TArray<USuperEffectComponent*> NewSuperEffectArray);
```

### 12.2 效果控制

```cpp
SetEffectIntensity(DmxAttribute);   // 亮度
SetEffectStrobe(DmxAttribute);      // 频闪
SetEffectColor(DmxR, DmxG, DmxB);  // 颜色

// 效果选择（通过 LUT 查表）
SetEffect(DmxAttribute);            // 单效果组件
SetEffectMatrix(DmxAttribute);      // 效果矩阵

// 效果 + 速度分离
SetEffectWithSpeed(DmxEffect, DmxSpeed);
SetEffectMatrixWithSpeed(DmxEffect, DmxSpeed);
```

**效果 LUT**：DMX 值 [0-255] 映射到 `{Effect, Speed, Width}` 参数组合。

---

## 13. 矩阵系统

### 13.1 矩阵组件控制

```cpp
// 综合控制（RGB + 频闪）
SetMatrixComponent(DmxR, DmxG, DmxB, DmxStrobe, USuperMatrixComponent*);

// 白光（单像素 / 多像素）
SetMatrixWhiteSingle(DmxAttribute, Index, USuperMatrixComponent*);
SetMatrixWhiteMultiple(DmxAttribute, USuperMatrixComponent*);

// RGB 颜色（单像素 / 多像素）
SetMatrixColorSingle(DmxR, DmxG, DmxB, Index, USuperMatrixComponent*);
SetMatrixColorMultiple(DmxR, DmxG, DmxB, USuperMatrixComponent*);

// 频闪
SetMatrixStrobe(DmxAttribute, USuperMatrixComponent*);

// 亮度
SetMatrixIntensity(DmxAttribute, USuperMatrixComponent*);

// 冷暖混色
SetMatrixCoolWarmMixSingle(DmxCool, DmxWarm, CoolTemp, WarmTemp, Index, Matrix);
SetMatrixCoolWarmMixMultiple(DmxCool, DmxWarm, CoolTemp, WarmTemp, Matrix);

// RGBW
SetMatrixColorRGBWSingle(DmxR, DmxG, DmxB, DmxW, Index, Matrix);
SetMatrixColorRGBWMultiple(DmxR, DmxG, DmxB, DmxW, Matrix);

// 动态多色混合
SetMatrixColorMix(ColorChannels, Matrix);
```

### 13.2 单像素 vs 多像素

- **单像素**（Single）：所有模块共用一个通道值，按 Index 指定目标像素
- **多像素**（Multiple）：每个模块独立通道，使用 `GET_SUPER_DMX_MATRIX_VALUE` 遍历

---

## 14. 运动控制

### 14.1 Pan/Tilt

```cpp
// 蓝图调用
YPan(DmxPan);     // 水平旋转
YTilt(DmxTilt);   // 垂直旋转
YPTSpeed(DmxPT);  // PT 速度控制
```

**映射**：`DMX[0,1] → Lerp(PanRange.X, PanRange.Y)` + `FInterpTo` 平滑

**默认范围**：
- Pan: `[-270°, 270°]`
- Tilt: `[-135°, 135°]`

### 14.2 无极旋转

```cpp
YPanRot(DmxInfinity, DmxPan);   // Pan 无极旋转
YTiltRot(DmxInfinity, DmxTilt); // Tilt 无极旋转
```

- 灯库中位置属性的子属性定义 `RotationMode`
- `EInfiniteRotationMode::Infinite` → 持续旋转
- `EInfiniteRotationMode::Position` → 从当前位置旋转到目标
- 速度 = `DMX[0,1] → Lerp(-5, 5) × InfiniteRotationalSpeed`

### 14.3 矩阵运动

```cpp
YMatrixPan(DmxPan, MatrixHard);     // 矩阵 Pan
YMatrixTilt(DmxTilt, MatrixRocker); // 矩阵 Tilt
YMatrixPanRot(DmxInfinity, DmxPan, MatrixRocker);  // 矩阵 Pan 无极
YMatrixTiltRot(DmxInfinity, DmxTilt, MatrixHard);   // 矩阵 Tilt 无极
```

### 14.4 升降

```cpp
SetLiftZ(USuperLiftComponent* NewSuperLift, FSuperDMXAttribute DmxLiftZ);
```

- `DMX[0,1] → Lerp(0, LiftRange)` + `FInterpTo(CurrentZ, Target, DeltaTime, LiftSpeed)`

### 14.5 灯具角度

```cpp
YLampAngle();  // 应用 PanAngle / TiltAngle（手动角度模式）
```

---

## 15. 频闪系统

### 15.1 频闪控制函数

```cpp
// 主光源
SetLightingStrobe(DmxStrobe);         // 单灯
SetLightingStrobeMatrix(DmxStrobe);   // 矩阵

// Spot 辅光
SetSpotStrobe(DmxStrobe);
SetSpotStrobeMatrix(DmxStrobe);

// 效果
SetEffectStrobe(DmxStrobe);

// 矩阵组件
SetMatrixStrobe(DmxStrobe, MatrixComp);
```

### 15.2 频闪模式（子属性驱动）

灯库中 Dimmer/Strobe 属性的子属性可以定义不同 DMX 范围对应不同频闪模式：

```
SubAttribute[0]: DmxMin=0,   DmxMax=7,   StrobeMode=Closed   (关闭)
SubAttribute[1]: DmxMin=8,   DmxMax=255, StrobeMode=Open      (常亮)
SubAttribute[2]: DmxMin=64,  DmxMax=95,  StrobeMode=Linear    (线性频闪)
SubAttribute[3]: DmxMin=96,  DmxMax=127, StrobeMode=Pulse     (脉冲)
SubAttribute[4]: DmxMin=128, DmxMax=159, StrobeMode=RampUp    (渐亮)
SubAttribute[5]: DmxMin=160, DmxMax=191, StrobeMode=RampDown  (渐灭)
SubAttribute[6]: DmxMin=192, DmxMax=223, StrobeMode=Sine      (正弦)
SubAttribute[7]: DmxMin=224, DmxMax=255, StrobeMode=Random    (随机)
```

### 15.3 ComponentDimmer

每个灯光/效果/矩阵组件都有 `ComponentDimmer` 属性：

```
FinalIntensity = ActorDimmer × ComponentDimmer × StrobeMultiplier
```

用于多模组灯具中独立控制每个组件的亮度。

---

## 16. 完整示例：C++ 电脑灯

以下展示一个 17 通道 Beam 灯的完整 C++ 实现思路。

### 16.1 灯库定义

| 通道 | 属性名 | 分类 | 位深 |
|------|--------|------|------|
| 1-2 | Pan | Position | 16-bit |
| 3-4 | Tilt | Position | 16-bit |
| 5 | PTSpeed | Control | 8-bit |
| 6-7 | Dimmer | Dimmer | 16-bit |
| 8 | Strobe | Strobe | 8-bit |
| 9 | ColorWheel | Color | 8-bit |
| 10 | Gobo1 | Gobo | 8-bit |
| 11 | Gobo1Rot | Gobo | 8-bit |
| 12 | Prism1 | Prism | 8-bit |
| 13 | PrismRot | Prism | 8-bit |
| 14 | Focus | Focus | 8-bit |
| 15 | Zoom | Beam | 8-bit |
| 16 | Frost | Frost | 8-bit |
| 17 | Control | Control | 8-bit |

### 16.2 蓝图 SuperDMXTick 逻辑

```
Event OnConstruction / BeginPlay
    └─ SetLightingDefaultValue(SuperLighting)

Event SuperDMXTick(DeltaSeconds)
    │
    ├─ Pan(DmxPan)                              // ch1-2
    ├─ Tilt(DmxTilt)                            // ch3-4
    ├─ PTSpeed(DmxPTSpeed)                      // ch5
    │
    ├─ SetLightingIntensity(DmxDimmer)          // ch6-7
    ├─ SetLightingStrobe(DmxStrobe)             // ch8
    │
    ├─ SetLightingColorWheel(DmxColorWheel, ColorAtlas) // ch9
    │
    ├─ SetBeamGobo1(DmxGobo1, DmxGobo1Rot, GoboAtlas) // ch10-11
    ├─ SetBeamPrism(DmxPrism1, None, DmxPrismRot)     // ch12-13
    ├─ SetBeamFocus(DmxFocus)                          // ch14
    │
    ├─ SetLightingZoom(DmxZoom)                 // ch15
    └─ SetLightingFrost(DmxFrost)               // ch16
```

---

## 17. 完整示例：蓝图 Wash 灯

### 17.1 通道表

| 通道 | 属性名 | 分类 |
|------|--------|------|
| 1-2 | Pan | Position |
| 3-4 | Tilt | Position |
| 5 | Dimmer | Dimmer |
| 6 | Strobe | Strobe |
| 7 | ColorR | Color |
| 8 | ColorG | Color |
| 9 | ColorB | Color |
| 10 | ColorW | Color |
| 11 | Zoom | Beam |
| 12 | CTO | Color |

### 17.2 蓝图节点连线

```
Event LightInitialization
    └─ SetLightingDefaultValue(SuperLighting)

Event SuperDMXTick
    ├─ Pan(DmxPan)                                          // 16-bit
    ├─ Tilt(DmxTilt)                                        // 16-bit
    ├─ SetLightingIntensity(DmxDimmer)                      // 8-bit
    ├─ SetLightingStrobe(DmxStrobe)                         // 8-bit
    ├─ SetLightingColorRGBWWithCTO(DmxR, DmxG, DmxB, DmxW, DmxCTO, 6500, 3200) // 8-bit ×5
    └─ SetLightingZoom(DmxZoom)                             // 8-bit
```

### 17.3 组件配置

- **SuperLighting** 使用 `USuperSpotComponent`（无可见光柱的 Wash 灯）
- **ZoomRange**: `(10, 60)` 表示 10°-60° 光束角
- 无 Gobo/Prism/Cutting 通道

---

## 18. 完整示例：蓝图频闪灯

频闪灯（Strobe）是最简单的灯具类型之一，没有 Pan/Tilt，只有亮度和频闪控制。

### 18.1 灯库定义

| 通道 | 属性名 | 分类 | 位深 | 说明 |
|------|--------|------|------|------|
| 1-2 | Dimmer | Dimmer | 16-bit | 亮度（高精度） |
| 3 | Strobe | Strobe | 8-bit | 频闪速度 |
| 4 | StrobeMode | Strobe | 8-bit | 频闪波形 |
| 5 | Duration | Control | 8-bit | 闪烁持续时间 |
| 6 | ColorR | Color | 8-bit | 红色 |
| 7 | ColorG | Color | 8-bit | 绿色 |
| 8 | ColorB | Color | 8-bit | 蓝色 |
| 9 | ColorW | Color | 8-bit | 白色 |

**频闪子属性配置**（Strobe 通道的 SubAttribute）：
```
SubAttr[0]: DmxMin=0,   DmxMax=7   → StrobeMode=Closed    (关闭)
SubAttr[1]: DmxMin=8,   DmxMax=15  → StrobeMode=Open      (常亮)
SubAttr[2]: DmxMin=16,  DmxMax=95  → StrobeMode=Linear    (线性频闪 慢→快)
SubAttr[3]: DmxMin=96,  DmxMax=127 → StrobeMode=Pulse     (脉冲)
SubAttr[4]: DmxMin=128, DmxMax=159 → StrobeMode=RampUp    (渐亮)
SubAttr[5]: DmxMin=160, DmxMax=191 → StrobeMode=RampDown  (渐灭)
SubAttr[6]: DmxMin=192, DmxMax=223 → StrobeMode=Sine      (正弦)
SubAttr[7]: DmxMin=224, DmxMax=255 → StrobeMode=Random    (随机)
```

### 18.2 组件选择

- **基类**：`ASuperStageLight`（虽然没有 Pan/Tilt，但需要灯光控制函数）
- **核心组件**：`USuperSpotComponent`（SpotLight 照射效果）
- **辅助组件**：`USuperEffectComponent`（灯体发光效果，可选）

### 18.3 组件层级

```
SceneBase（根）
 └─ SM_Body（灯体模型）
     ├─ SuperLighting（USuperSpotComponent — 照射光源）
     └─ SuperEffect（USuperEffectComponent — 灯体自发光，可选）
```

### 18.4 蓝图配置

- `PanRange`: `(0, 0)` / `TiltRange`: `(0, 0)` — 无运动
- `bDimmer=true`, `bStrobe=true`, `bColor=true`
- `bPan=false`, `bTilt=false`, `bZoom=false`, `bGobo1=false`
- `ZoomRange`: `(40, 40)` — 固定光束角
- `DimmerCurveExponent`: `1.0`（频闪灯通常使用线性响应）

### 18.5 蓝图节点连线

```
Event OnConstruction / BeginPlay
    ├─ SetLightingDefaultValue(SuperLighting)
    └─ SetEffectDefaultValue(SuperEffect)     // 如果有灯体自发光

Event SuperDMXTick(DeltaSeconds)
    │
    ├─ SetLightingIntensity(DmxDimmer)            // ch1-2  16-bit 亮度
    ├─ SetLightingStrobe(DmxStrobe)               // ch3    频闪速度
    │
    ├─ SetLightingColorRGBW(DmxR, DmxG, DmxB, DmxW)  // ch6-9  RGBW 混色
    │
    │  // 如果有灯体自发光效果组件
    ├─ SetEffectIntensity(DmxDimmer)              // 同步亮度
    ├─ SetEffectStrobe(DmxStrobe)                 // 同步频闪
    └─ SetEffectColor(DmxR, DmxG, DmxB)          // 同步颜色
```

### 18.6 关键要点

- 频闪灯的 `DimmerCurveExponent` 通常设为 `1.0`（线性），因为需要快速响应
- 频闪模式由灯库 SubAttribute 的 `StrobeMode` 字段驱动，无需在蓝图中额外处理
- `SetRandomSeed` 可让多台频闪灯产生不同的随机序列（各灯传入不同种子值）
- 灯体自发光组件可与主光源共享 Dimmer/Strobe/Color 通道

---

## 19. 完整示例：蓝图 Profile 摇头灯

Profile（成像灯/切割灯）是最复杂的灯具类型，具备全套光学系统。

### 19.1 灯库定义

| 通道 | 属性名 | 分类 | 位深 | 说明 |
|------|--------|------|------|------|
| 1-2 | Pan | Position | 16-bit | 水平旋转 |
| 3-4 | Tilt | Position | 16-bit | 垂直旋转 |
| 5 | PTSpeed | Control | 8-bit | PT 速度 |
| 6-7 | Dimmer | Dimmer | 16-bit | 亮度 |
| 8 | Strobe | Strobe | 8-bit | 频闪 |
| 9 | Color1 | Color | 8-bit | 颜色轮 1 |
| 10 | Color2 | Color | 8-bit | 颜色轮 2 |
| 11 | CyanMix | Color | 8-bit | 青色混色 (CMY) |
| 12 | MagentaMix | Color | 8-bit | 品红混色 (CMY) |
| 13 | YellowMix | Color | 8-bit | 黄色混色 (CMY) |
| 14 | CTO | Color | 8-bit | 色温调节 |
| 15 | Gobo1 | Gobo | 8-bit | 图案轮 1 |
| 16 | Gobo1Rot | Gobo | 8-bit | 图案 1 旋转 |
| 17 | Gobo2 | Gobo | 8-bit | 图案轮 2 |
| 18 | Gobo2Rot | Gobo | 8-bit | 图案 2 旋转 |
| 19 | Prism1 | Prism | 8-bit | 棱镜 |
| 20 | PrismRot | Prism | 8-bit | 棱镜旋转 |
| 21 | Focus | Focus | 8-bit | 调焦 |
| 22 | Zoom | Beam | 8-bit | 变焦 |
| 23 | Frost | Frost | 8-bit | 雾化 |
| 24 | Iris | Beam | 8-bit | 光圈 |
| 25 | A1 | Cutting | 8-bit | 切割叶片 A1 |
| 26 | B1 | Cutting | 8-bit | 切割叶片 B1 |
| 27 | A2 | Cutting | 8-bit | 切割叶片 A2 |
| 28 | B2 | Cutting | 8-bit | 切割叶片 B2 |
| 29 | A3 | Cutting | 8-bit | 切割叶片 A3 |
| 30 | B3 | Cutting | 8-bit | 切割叶片 B3 |
| 31 | A4 | Cutting | 8-bit | 切割叶片 A4 |
| 32 | B4 | Cutting | 8-bit | 切割叶片 B4 |
| 33 | ShaperRot | Cutting | 8-bit | 切割系统旋转 |

### 19.2 组件选择

- **基类**：`ASuperStageLight`
- **核心组件**：`USuperCuttingComponent`（支持切割的光束组件，替代 USuperBeamComponent）
- **辅助组件**：`USuperSpotComponent`（辅光，可选）

### 19.3 组件层级

```
SceneBase（根）
 └─ SM_Hook
     └─ SM_Base
         └─ SceneRocker（Pan）
             └─ SM_Rocker
                 └─ SceneHard（Tilt）
                     └─ SM_Hard
                         ├─ SuperLighting（USuperCuttingComponent — 主光源+切割）
                         └─ SpotLighting（USuperSpotComponent — 辅光，可选）
```

### 19.4 通道开关配置

```
bChannelEdit=true（先勾选才能看到下面的开关）

bDimmer=true, bStrobe=true, bPan=true, bTilt=true, bPTSpeed=true,
bColorWheel=true, bColor=true,
bGobo1=true, bGobo2=true, bGobo_Rot=true,
bPrism1=true, bPrism_Rot=true,
bFocus=true, bZoom=true, bFrost=true, bIris=true,
bCuttingChannel=true
```

### 19.5 蓝图默认参数

- `ZoomRange`: `(7, 50)` — Profile 灯典型角度范围
- `LightDefaultValue.MaxLightDistance`: `8000` — 光照距离
- `LightDefaultValue.MaxLightIntensity`: `500`
- `LightDefaultValue.bLightShadow`: `true`
- `BeamDefaultValue.AtmosphericDensity`: `0.03`
- `BeamDefaultValue.BeamQuality`: `64`
- `DimmerCurveExponent`: `2.0`

### 19.6 蓝图节点连线

```
Event OnConstruction / BeginPlay
    ├─ SetLightingDefaultValue(SuperLighting)
    └─ SetSpotDefaultValue(SpotLighting)       // 如果有辅光

Event SuperDMXTick(DeltaSeconds)
    │
    │  ── 运动 ──
    ├─ Pan(DmxPan)                                                     // ch1-2
    ├─ Tilt(DmxTilt)                                                   // ch3-4
    ├─ PTSpeed(DmxPTSpeed)                                             // ch5
    │
    │  ── 亮度/频闪 ──
    ├─ SetLightingIntensity(DmxDimmer)                                 // ch6-7
    ├─ SetLightingStrobe(DmxStrobe)                                    // ch8
    │
    │  ── 颜色（颜色轮 + CMY + CTO）──
    ├─ SetLightingColorWheel2AndCMYAndCTO(                             // ch9-14
    │      DmxColor1, DmxColor2,
    │      DmxCyanMix, DmxMagentaMix, DmxYellowMix,
    │      DmxCTO, 2700, 6500,
    │      ColorAtlas1, ColorAtlas2)
    │
    │  ── 图案 ──
    ├─ SetBeamGobo2(DmxGobo1, DmxGobo1Rot,                            // ch15-18
    │              DmxGobo2, DmxGobo2Rot,
    │              Gobo1Atlas, Gobo2Atlas)
    │
    │  ── 棱镜 ──
    ├─ SetBeamPrism(DmxPrism1, None, DmxPrismRot)                     // ch19-20
    │
    │  ── 光学 ──
    ├─ SetBeamFocus(DmxFocus)                                          // ch21
    ├─ SetLightingZoom(DmxZoom)                                        // ch22
    ├─ SetLightingFrost(DmxFrost)                                      // ch23
    ├─ SetLightingIris(DmxIris)                                        // ch24
    │
    │  ── 切割 ──
    ├─ SetBeamCutting(DmxA1, DmxB1, DmxA2, DmxB2,                    // ch25-32
    │                DmxA3, DmxB3, DmxA4, DmxB4)
    └─ SetBeamCuttingRotate(DmxShaperRot, DmxGobo1Rot)               // ch33 + ch16
```

### 19.7 关键要点

- `USuperCuttingComponent` 继承自 `USuperBeamComponent`，拥有全部光束功能 + 切割
- `SetBeamCuttingRotate` 的第二个参数 `DmxGoboRot` 用于同步 Gobo 旋转与切割旋转
- CMY 混色是减色模式：C=1 表示完全去红，M=1 去绿，Y=1 去蓝
- CTO 的色温方向：`DmxCTO=0` 对应 `CoolTemperature`，`DmxCTO=1` 对应 `WarmTemperature`
- 两个颜色轮需要分别提供对应的 `ColorAtlas` 贴图

---

## 20. 完整示例：蓝图矩阵灯（带辅光）

LED 矩阵灯，7×7=49 个像素独立 RGB 控制，带一个 SpotLight 辅光用于照明。

### 20.1 灯库定义（多模块）

**模块 0（主模块，7 通道）**：

| 通道偏移 | 属性名 | 分类 | 位深 | 说明 |
|---------|--------|------|------|------|
| 0-1 | Dimmer | Dimmer | 16-bit | 总亮度 |
| 2 | Strobe | Strobe | 8-bit | 总频闪 |
| 3 | SpotDimmer | Dimmer | 8-bit | 辅光亮度 |
| 4 | SpotColorR | Color | 8-bit | 辅光红色 |
| 5 | SpotColorG | Color | 8-bit | 辅光绿色 |
| 6 | SpotColorB | Color | 8-bit | 辅光蓝色 |

**模块 1-49（像素模块，每模块 3 通道）**：

| 通道偏移 | 属性名 | 分类 | 说明 |
|---------|--------|------|------|
| 0 | MatrixR | Color | 像素红色 |
| 1 | MatrixG | Color | 像素绿色 |
| 2 | MatrixB | Color | 像素蓝色 |

**总通道数**：7 + 49×3 = **154 通道**

**模块 Patch 设置**：
```
Module 0:  Patch = 0    → 通道 1-7
Module 1:  Patch = 7    → 通道 8-10
Module 2:  Patch = 10   → 通道 11-13
...
Module 49: Patch = 151  → 通道 152-154
```

### 20.2 组件选择

- **基类**：`ASuperStageLight`（虽然无 Pan/Tilt，但需要 Matrix 控制函数）
- **核心组件**：`USuperMatrixComponent`（49 像素矩阵）
- **辅助组件**：`USuperSpotComponent`（辅光照明）

### 20.3 组件层级

```
SceneBase（根）
 └─ SM_Body（灯体模型）
     ├─ SuperMatrix（USuperMatrixComponent — 49 像素面板）
     └─ SpotLighting（USuperSpotComponent — 辅光照明）
```

### 20.4 通道开关配置

```
bDimmer=true, bStrobe=true, bColor=true,
bPan=false, bTilt=false
```

### 20.5 蓝图默认参数

- `PanRange`: `(0, 0)` / `TiltRange`: `(0, 0)` — 无运动
- **USuperMatrixComponent**：
  - `SegCount`: `49`
  - `bUseUAxis`: `true`（按 U 方向 7 列切分）
  - `SegmentLightType`: `PointLight`（每像素添加点光源）
  - `SegmentLightIntensity`: `100`
  - `SegmentLightRadius`: `50`
- **USuperSpotComponent（辅光）**：
  - `ZoomRange`: `(15, 60)` — 辅光角度

### 20.6 蓝图节点连线

```
Event OnConstruction / BeginPlay
    ├─ SetMatrixMaterial(SuperMatrix)           // 初始化矩阵材质 + 分段光源
    └─ SetSpotDefaultValue(SpotLighting)        // 初始化辅光

Event SuperDMXTick(DeltaSeconds)
    │
    │  ── 主模块控制 ──
    ├─ SetMatrixIntensity(DmxDimmer, SuperMatrix)            // 总亮度
    ├─ SetMatrixStrobe(DmxStrobe, SuperMatrix)               // 总频闪
    │
    │  ── 辅光控制 ──
    ├─ SetSpotIntensity(DmxSpotDimmer)                       // 辅光亮度
    ├─ SetSpotColorRGB(DmxSpotR, DmxSpotG, DmxSpotB)        // 辅光颜色
    │
    │  ── 矩阵像素（49 像素批量设置）──
    └─ SetMatrixColorMultiple(DmxMatrixR, DmxMatrixG, DmxMatrixB, SuperMatrix)
```

### 20.7 关键要点

- `SetMatrixColorMultiple` 内部使用 `GET_SUPER_DMX_MATRIX_VALUE` 宏自动遍历模块 1-49
- 矩阵灯库**必须**保证所有像素模块的属性名完全一致（都叫 `MatrixR/G/B`）
- 每个像素模块的 `Patch` 值必须正确偏移，否则像素地址错乱
- `SegmentLightType=PointLight` 让每个像素产生真实光照，但会增加性能开销
- 辅光使用 `SetSpotIntensity` / `SetSpotColorRGB` 独立控制，不受矩阵亮度影响
- `SetMatrixMaterial` 必须在 OnConstruction/BeginPlay 调用，不能在 Tick 中

---

## 21. 完整示例：蓝图摇头矩阵灯

摇头矩阵灯 = Pan/Tilt 运动 + 矩阵像素控制。以 3×3=9 像素为例。

### 21.1 灯库定义（多模块）

**模块 0（主模块，8 通道）**：

| 通道偏移 | 属性名 | 分类 | 位深 | 说明 |
|---------|--------|------|------|------|
| 0-1 | Pan | Position | 16-bit | 水平旋转 |
| 2-3 | Tilt | Position | 16-bit | 垂直旋转 |
| 4 | PTSpeed | Control | 8-bit | PT 速度 |
| 5-6 | Dimmer | Dimmer | 16-bit | 总亮度 |
| 7 | Strobe | Strobe | 8-bit | 总频闪 |

**模块 1-9（像素模块，每模块 4 通道）**：

| 通道偏移 | 属性名 | 分类 | 说明 |
|---------|--------|------|------|
| 0 | MatrixR | Color | 像素红色 |
| 1 | MatrixG | Color | 像素绿色 |
| 2 | MatrixB | Color | 像素蓝色 |
| 3 | MatrixW | Color | 像素白色 |

**总通道数**：8 + 9×4 = **44 通道**

### 21.2 组件层级

```
SceneBase（根）
 └─ SM_Hook
     └─ SM_Base
         └─ SceneRocker（Pan）
             └─ SM_Rocker
                 └─ SceneHard（Tilt）
                     └─ SM_Hard
                         └─ SuperMatrix（USuperMatrixComponent — 9 像素）
```

### 21.3 通道开关配置

```
bDimmer=true, bStrobe=true, bPan=true, bTilt=true, bPTSpeed=true
```

### 21.4 蓝图默认参数

- `PanRange`: `(-270, 270)`
- `TiltRange`: `(-135, 135)`
- **USuperMatrixComponent**：
  - `SegCount`: `9`
  - `bUseUAxis`: `true`（3 列切分）
  - `SegmentLightType`: `SpotLight`（矩阵像素用聚光灯朝外照射）
  - `SegmentLightIntensity`: `200`
  - `SegmentLightRadius`: `100`

### 21.5 蓝图节点连线

```
Event OnConstruction / BeginPlay
    └─ SetMatrixMaterial(SuperMatrix)

Event SuperDMXTick(DeltaSeconds)
    │
    │  ── 运动 ──
    ├─ Pan(DmxPan)                                       // ch1-2
    ├─ Tilt(DmxTilt)                                     // ch3-4
    ├─ PTSpeed(DmxPTSpeed)                               // ch5
    │
    │  ── 总控 ──
    ├─ SetMatrixIntensity(DmxDimmer, SuperMatrix)        // ch6-7
    ├─ SetMatrixStrobe(DmxStrobe, SuperMatrix)           // ch8
    │
    │  ── 矩阵像素（9 像素批量 RGBW）──
    └─ SetMatrixColorRGBWMultiple(DmxMatrixR, DmxMatrixG, DmxMatrixB, DmxMatrixW, SuperMatrix)
```

### 21.6 关键要点

- 摇头矩阵灯的矩阵组件**必须**挂在 `SceneHard` 下，这样才能跟随 Tilt 旋转
- `SetMatrixColorRGBWMultiple` 自动遍历模块 1-9，处理 RGBW 四通道
- 如果灯具同时需要 Pan/Tilt **无极旋转**，还需添加 `PanRot`/`TiltRot` 通道
- 矩阵组件的 `SegmentLightType` 选择 `SpotLight` 时，每个像素对外打出聚光

---

## 22. 完整示例：蓝图多功能摇头灯

高端摇头灯 = Beam/Spot 主光 + 效果环 + 矩阵像素 + 辅光 + 升降。展示多组件协同。

### 22.1 灯库定义

**模块 0（主模块，30 通道）**：

| 通道偏移 | 属性名 | 分类 | 位深 | 说明 |
|---------|--------|------|------|------|
| 0-1 | Pan | Position | 16-bit | 水平 |
| 2-3 | Tilt | Position | 16-bit | 垂直 |
| 4 | PTSpeed | Control | 8-bit | PT 速度 |
| 5-6 | Dimmer | Dimmer | 16-bit | 主光亮度 |
| 7 | Strobe | Strobe | 8-bit | 主光频闪 |
| 8 | ColorWheel | Color | 8-bit | 颜色轮 |
| 9 | Gobo1 | Gobo | 8-bit | 图案轮 |
| 10 | Gobo1Rot | Gobo | 8-bit | 图案旋转 |
| 11 | Prism1 | Prism | 8-bit | 棱镜 |
| 12 | PrismRot | Prism | 8-bit | 棱镜旋转 |
| 13 | Focus | Focus | 8-bit | 调焦 |
| 14 | Zoom | Beam | 8-bit | 变焦 |
| 15 | Frost | Frost | 8-bit | 雾化 |
| 16 | SpotDimmer | Dimmer | 8-bit | 辅光亮度 |
| 17 | SpotStrobe | Strobe | 8-bit | 辅光频闪 |
| 18 | SpotColorR | Color | 8-bit | 辅光红色 |
| 19 | SpotColorG | Color | 8-bit | 辅光绿色 |
| 20 | SpotColorB | Color | 8-bit | 辅光蓝色 |
| 21 | SpotZoom | Beam | 8-bit | 辅光变焦 |
| 22 | EffectDimmer | Effect | 8-bit | 效果环亮度 |
| 23 | EffectStrobe | Effect | 8-bit | 效果环频闪 |
| 24 | EffectValue | Effect | 8-bit | 效果选择 |
| 25 | EffectColorR | Color | 8-bit | 效果环红色 |
| 26 | EffectColorG | Color | 8-bit | 效果环绿色 |
| 27 | EffectColorB | Color | 8-bit | 效果环蓝色 |
| 28 | MatrixDimmer | Dimmer | 8-bit | 矩阵亮度 |
| 29 | MatrixStrobe | Strobe | 8-bit | 矩阵频闪 |

**模块 1-12（矩阵像素，每模块 3 通道）**：

| 通道偏移 | 属性名 | 分类 | 说明 |
|---------|--------|------|------|
| 0 | MatrixR | Color | 像素红色 |
| 1 | MatrixG | Color | 像素绿色 |
| 2 | MatrixB | Color | 像素蓝色 |

**总通道数**：30 + 12×3 = **66 通道**

### 22.2 组件层级

```
SceneBase（根）
 └─ SM_Hook
     └─ SM_Base
         └─ SceneRocker（Pan）
             └─ SM_Rocker
                 └─ SceneHard（Tilt）
                     └─ SM_Hard
                         ├─ SuperLighting（USuperBeamComponent — 主光源+光柱）
                         ├─ SpotLighting（USuperSpotComponent — 辅光 Wash）
                         ├─ SuperEffect（USuperEffectComponent — 效果环）
                         └─ SuperMatrix（USuperMatrixComponent — 12 像素环）
```

### 22.3 通道开关配置

```
bDimmer=true, bStrobe=true, bPan=true, bTilt=true, bPTSpeed=true,
bColorWheel=true,
bGobo1=true, bGobo_Rot=true,
bPrism1=true, bPrism_Rot=true,
bFocus=true, bZoom=true, bFrost=true,
bEffect=true
```

### 22.4 蓝图默认参数

- `ZoomRange`: `(3, 45)` — 主光束角
- **SpotLighting**：`ZoomRange`: `(10, 60)` — 辅光角度
- **SuperEffect**：`bTransparent`: `true` — 效果环透明材质
- **SuperMatrix**：
  - `SegCount`: `12`（环形排列）
  - `bUseUAxis`: `true`
  - `SegmentLightType`: `PointLight`
  - `SegmentLightIntensity`: `80`

### 22.5 蓝图节点连线

```
Event OnConstruction / BeginPlay
    ├─ SetLightingDefaultValue(SuperLighting)    // 主光初始化
    ├─ SetSpotDefaultValue(SpotLighting)         // 辅光初始化
    ├─ SetEffectDefaultValue(SuperEffect)        // 效果环初始化
    └─ SetMatrixMaterial(SuperMatrix)            // 矩阵初始化

Event SuperDMXTick(DeltaSeconds)
    │
    │  ══════════ 运动控制 ══════════
    ├─ Pan(DmxPan)                                                // ch1-2
    ├─ Tilt(DmxTilt)                                              // ch3-4
    ├─ PTSpeed(DmxPTSpeed)                                        // ch5
    │
    │  ══════════ 主光源 ══════════
    ├─ SetLightingIntensity(DmxDimmer)                            // ch6-7
    ├─ SetLightingStrobe(DmxStrobe)                               // ch8
    ├─ SetLightingColorWheel(DmxColorWheel, ColorAtlas)           // ch9
    ├─ SetBeamGobo1(DmxGobo1, DmxGobo1Rot, GoboAtlas)            // ch10-11
    ├─ SetBeamPrism(DmxPrism1, None, DmxPrismRot)                // ch12-13
    ├─ SetBeamFocus(DmxFocus)                                     // ch14
    ├─ SetLightingZoom(DmxZoom)                                   // ch15
    ├─ SetLightingFrost(DmxFrost)                                 // ch16
    │
    │  ══════════ 辅光（独立控制）══════════
    ├─ SetSpotIntensity(DmxSpotDimmer)                            // ch17
    ├─ SetSpotStrobe(DmxSpotStrobe)                               // ch18
    ├─ SetSpotColorRGB(DmxSpotR, DmxSpotG, DmxSpotB)             // ch19-21
    ├─ SetSpotZoom(DmxSpotZoom)                                   // ch22
    │
    │  ══════════ 效果环 ══════════
    ├─ SetEffectIntensity(DmxEffectDimmer)                        // ch23
    ├─ SetEffectStrobe(DmxEffectStrobe)                           // ch24
    ├─ SetEffect(DmxEffectValue)                                  // ch25
    ├─ SetEffectColor(DmxEffectR, DmxEffectG, DmxEffectB)        // ch26-28
    │
    │  ══════════ 矩阵像素 ══════════
    ├─ SetMatrixIntensity(DmxMatrixDimmer, SuperMatrix)           // ch29
    ├─ SetMatrixStrobe(DmxMatrixStrobe, SuperMatrix)              // ch30
    └─ SetMatrixColorMultiple(DmxMatrixR, DmxMatrixG, DmxMatrixB, SuperMatrix)
```

### 22.6 关键要点

- **四组件协同**：主光/辅光/效果环/矩阵各自独立的亮度、频闪和颜色控制
- 主光源使用 `USuperBeamComponent`（有光柱），辅光使用 `USuperSpotComponent`（无光柱）
- 效果环和矩阵像素都挂在 `SceneHard` 下跟随灯头旋转
- 辅光的 `SetSpot*` 系列函数与主光源的 `SetLighting*` 完全独立
- 矩阵的 `SetMatrixIntensity` / `SetMatrixStrobe` 控制所有像素的总亮度/频闪
- 效果环可以使用 `SetEffectWithSpeed(DmxEffect, DmxSpeed)` 分离效果选择和速度控制
- 初始化时所有组件的 `DefaultValue` / `Material` 函数**都必须调用**，缺一个该组件就不工作

---

## 23. 各类灯具制作详解

本章为每种灯具类型提供完整的制作流程，包括基类选择、组件配置、灯库定义、初始化和控制逻辑。

---

### 23.1 Beam 灯（光束灯）

**特征**：窄光束、可见光柱、Gobo/棱镜/颜色轮

**基类**：`ASuperStageLight`  
**核心组件**：`USuperBeamComponent`（SuperLighting）  
**辅助组件**（可选）：`USuperSpotComponent`（SpotLighting 辅光）

**组件层级**：
```
SceneBase（根）
 └─ SM_Hook（挂钩模型）
     └─ SM_Base（底座模型）
         └─ SceneRocker（Pan 旋转轴）
             └─ SM_Rocker（摇臂模型）
                 └─ SceneHard（Tilt 旋转轴）
                     └─ SM_Hard（灯头模型）
                         ├─ SuperLighting（USuperBeamComponent）
                         └─ SpotLighting（USuperSpotComponent，可选辅光）
```

**灯库定义**（典型 17 通道）：

| 通道 | 属性名 | 分类 | 位深 |
|------|--------|------|------|
| 1-2 | Pan | Position | 16-bit |
| 3-4 | Tilt | Position | 16-bit |
| 5 | PTSpeed | Control | 8-bit |
| 6-7 | Dimmer | Dimmer | 16-bit |
| 8 | Strobe | Strobe | 8-bit |
| 9 | ColorWheel | Color | 8-bit |
| 10 | Gobo1 | Gobo | 8-bit |
| 11 | Gobo1Rot | Gobo | 8-bit |
| 12 | Prism1 | Prism | 8-bit |
| 13 | PrismRot | Prism | 8-bit |
| 14 | Focus | Focus | 8-bit |
| 15 | Zoom | Beam | 8-bit |
| 16 | Frost | Frost | 8-bit |
| 17 | Control | Control | 8-bit |

**通道开关配置**（bChannelEdit=true 后勾选）：
```
bDimmer=true, bStrobe=true, bPan=true, bTilt=true, bPTSpeed=true,
bColor=true, bGobo=true, bPrism=true, bFocus=true, bZoom=true, bFrost=true
```

**初始化**（OnConstruction + BeginPlay）：
```
SetLightingDefaultValue(SuperLighting)
```

**SuperDMXTick 控制逻辑**：
```
Pan(DmxPan)                              // 水平旋转
Tilt(DmxTilt)                            // 垂直旋转
PTSpeed(DmxPTSpeed)                      // PT 速度

SetLightingIntensity(DmxDimmer)          // 亮度
SetLightingStrobe(DmxStrobe)             // 频闪

SetLightingColorWheel(DmxColorWheel, ColorAtlas)  // 颜色轮

SetBeamGobo1(DmxGobo1, DmxGobo1Rot, GoboAtlas)   // 图案轮1
SetBeamPrism(DmxPrism1, None, DmxPrismRot)        // 棱镜
SetBeamFocus(DmxFocus)                             // 调焦

SetLightingZoom(DmxZoom)                 // 变焦
SetLightingFrost(DmxFrost)               // 雾化
```

**蓝图默认参数配置**：
- `ZoomRange`: `(1, 8)` — 1°~8° 光束角
- `PanRange`: `(-270, 270)` — 默认即可
- `TiltRange`: `(-135, 135)` — 默认即可
- `LightDefaultValue.MaxLightDistance`: `5000` — 光照距离(cm)
- `LightDefaultValue.MaxLightIntensity`: `1000` — 最大亮度

---

### 23.2 Spot 灯（聚光灯）

**特征**：与 Beam 灯类似但光束角更大，通常有颜色轮和 Gobo

**基类**：`ASuperStageLight`  
**核心组件**：`USuperBeamComponent`（带光柱）或 `USuperSpotComponent`（无光柱）

**与 Beam 灯的区别**：
- `ZoomRange`: `(10, 45)` — 更大的光束角范围
- 通常有 Iris 通道
- 可选第二图案轮

**额外通道**（在 Beam 基础上）：

| 通道 | 属性名 | 分类 |
|------|--------|------|
| 额外 | Iris | Beam |
| 额外 | Gobo2 / Gobo2Rot | Gobo |

**额外 SuperDMXTick 节点**：
```
SetLightingIris(DmxIris)                          // 光圈
SetBeamGobo2(DmxGobo2, DmxGobo2Rot, Gobo2Atlas)  // 第二图案轮
```

---

### 23.3 Wash 灯（染色灯）

**特征**：柔和均匀光束、RGB/RGBW 混色、大角度 Zoom

**基类**：`ASuperStageLight`  
**核心组件**：`USuperSpotComponent`（无可见光柱，柔光效果）

**灯库定义**（典型 12 通道）：

| 通道 | 属性名 | 分类 | 位深 |
|------|--------|------|------|
| 1-2 | Pan | Position | 16-bit |
| 3-4 | Tilt | Position | 16-bit |
| 5 | Dimmer | Dimmer | 8-bit |
| 6 | Strobe | Strobe | 8-bit |
| 7 | ColorR | Color | 8-bit |
| 8 | ColorG | Color | 8-bit |
| 9 | ColorB | Color | 8-bit |
| 10 | ColorW | Color | 8-bit |
| 11 | Zoom | Beam | 8-bit |
| 12 | CTO | Color | 8-bit |

**通道开关**：
```
bDimmer=true, bStrobe=true, bPan=true, bTilt=true,
bColor=true, bZoom=true
```

**SuperDMXTick**：
```
Pan(DmxPan)
Tilt(DmxTilt)
SetLightingIntensity(DmxDimmer)
SetLightingStrobe(DmxStrobe)
SetLightingColorRGBWWithCTO(DmxR, DmxG, DmxB, DmxW, DmxCTO, 6500, 3200)
SetLightingZoom(DmxZoom)
```

**蓝图默认参数**：
- `ZoomRange`: `(10, 60)` — 10°~60° 柔光角
- 无 Gobo/Prism/Cutting 组件

---

### 23.4 Profile 灯（成像灯/切割灯）

**特征**：可见光柱 + 四叶切割系统 + Gobo + 棱镜

**基类**：`ASuperStageLight`  
**核心组件**：`USuperCuttingComponent`（替代 USuperBeamComponent）

**与 Beam 灯的关键区别**：
- 使用 `USuperCuttingComponent` 作为主光源（支持切割）
- 增加 8 个切割通道（A1-A4, B1-B4）+ ShaperRot

**额外灯库通道**：

| 通道 | 属性名 | 分类 | 说明 |
|------|--------|------|------|
| 额外 | A1 | Cutting | 叶片 A1 位置 |
| 额外 | A2 | Cutting | 叶片 A2 位置 |
| 额外 | A3 | Cutting | 叶片 A3 位置 |
| 额外 | A4 | Cutting | 叶片 A4 位置 |
| 额外 | B1 | Cutting | 叶片 B1 位置 |
| 额外 | B2 | Cutting | 叶片 B2 位置 |
| 额外 | B3 | Cutting | 叶片 B3 位置 |
| 额外 | B4 | Cutting | 叶片 B4 位置 |
| 额外 | ShaperRot | Cutting | 切割系统旋转 |

**通道开关**：除 Beam 灯的开关外，额外勾选 `bCutting=true`

**SuperDMXTick**（切割部分）：
```
SetBeamCutting(DmxA1, DmxA2, DmxA3, DmxA4, DmxB1, DmxB2, DmxB3, DmxB4)
SetBeamCuttingRotate(DmxShaperRot)
```

**C++ 构造函数中的组件替换**：
```cpp
// Profile 灯使用 CuttingComponent 替代 BeamComponent
SuperLighting = CreateDefaultSubobject<USuperCuttingComponent>(TEXT("SuperLighting"));
SuperLighting->SetupAttachment(SceneHard);
```

---

### 23.5 矩阵灯 / 像素灯

**特征**：多像素独立颜色控制、多模块灯库

**基类**：`ASuperStageLight`  
**核心组件**：`USuperMatrixComponent`（矩阵像素）+ `USuperBeamComponent`/`USuperSpotComponent`（可选主光源）

**灯库定义**（多模块结构）：

以 5×5 矩阵灯为例，灯库需要 26 个模块：
- **模块 0（主模块）**：Dimmer(16-bit) + Strobe + Pan + Tilt + Zoom = 6 通道
- **模块 1-25（像素模块）**：每模块 ColorR + ColorG + ColorB = 3 通道
- **总通道**：6 + 25×3 = 81 通道

**通道开关**：
```
bDimmer=true, bStrobe=true, bPan=true, bTilt=true,
bZoom=true, bMatrix=true
```

**初始化**（OnConstruction + BeginPlay）：
```
SetLightingDefaultValue(SuperLighting)    // 主光源初始化
SetMatrixMaterial()                        // 在 USuperMatrixComponent 上调用
```

**SuperDMXTick**：
```
// 主控制
Pan(DmxPan)
Tilt(DmxTilt)
SetLightingIntensity(DmxDimmer)

// 矩阵像素（使用 Multiple 批量函数 + GET_SUPER_DMX_MATRIX_VALUE 宏）
SetMatrixColorRGBMultiple(DmxR, DmxG, DmxB, MatrixComp)

// 或逐像素设置
for Index in 0..24:
    SetMatrixColorRGBSingle(DmxR, DmxG, DmxB, Index, MatrixComp)
```

**USuperMatrixComponent 配置**：
- `SegCount` = 25（与像素模块数量匹配）
- `bUseUAxis` = true/false（根据矩阵方向）
- `SegmentLightType` = PointLight（可选，为每个像素添加真实光源）

---

### 23.6 LED 灯条/灯带效果灯

**特征**：GPU 材质驱动效果、可批量应用到多个网格

**基类**：`ASuperLightStripEffect`（继承 `ASuperDmxActorBase`，**不是** ASuperStageLight）  
**内置组件**：效果材质系统（无需手动添加组件）

**灯库定义**（典型 9 通道）：

| 通道 | 属性名 | 分类 | 说明 |
|------|--------|------|------|
| 1 | Dimmer | Dimmer | 亮度 |
| 2 | Strobe | Strobe | 频闪 |
| 3 | Effect | Effect | 效果选择（0-10） |
| 4 | Speed | Effect | 效果速度 |
| 5 | Width | Effect | 效果宽度 |
| 6 | Direction | Effect | 效果方向 |
| 7 | ColorR | Color | 红色 |
| 8 | ColorG | Color | 绿色 |
| 9 | ColorB | Color | 蓝色 |

**蓝图配置**：
- `TargetMeshActors`：场景中选取需要应用效果的 StaticMeshActor
- `MaxLightIntensity`：最大亮度
- `MaterialIndex`：材质槽索引

**初始化**（OnConstruction + BeginPlay）：
```
SetLightStripDefault()
```

**SuperDMXTick**：
```
SetLightStripEffect(DmxDimmer, DmxStrobe, DmxEffect, DmxSpeed, DmxWidth, DmxDirection, DmxR, DmxG, DmxB)
```

**效果映射**：
| DMX 归一化 | 效果 |
|-----------|------|
| 0.0 | 无效果（纯色） |
| 0.1~1.0 | 10 种内置效果（流水/追逐/呼吸/闪烁等） |

---

### 23.7 VFX 特效灯（烟雾/火焰/雪花等）

**特征**：Niagara 粒子系统驱动、支持 Pan/Tilt

**基类**：`ASuperStageVFXActor`（继承 `ASuperLightBase`，有 Pan/Tilt）  
**内置组件**：`UNiagaraComponent*` Niagara

**灯库定义**（典型 8 通道）：

| 通道 | 属性名 | 分类 | 说明 |
|------|--------|------|------|
| 1-2 | Pan | Position | 水平旋转 |
| 3-4 | Tilt | Position | 垂直旋转 |
| 5 | SpawnCount | Control | 粒子生成量 |
| 6 | ColorR | Color | 粒子红色 |
| 7 | ColorG | Color | 粒子绿色 |
| 8 | ColorB | Color | 粒子蓝色 |

**蓝图默认参数**：
- `MaxSpawnCount`：最大生成数量
- `MaxSpeed`：粒子最大速度
- `LoopTime`：循环时间
- `Minimum` / `Maximum`：粒子范围

**初始化**（OnConstruction + BeginPlay）：
```
SetNiagaraDefaultValue()   // 激活 Niagara + 设置 Speed/LoopTime/Min/Max
```

**SuperDMXTick**：
```
Pan(DmxPan)
Tilt(DmxTilt)
SetNiagaraSpawnCount(DmxSpawnCount)              // DMX→粒子数量
SetNiagaraColor(DmxColorR, DmxColorG, DmxColorB) // DMX→粒子颜色
```

**Niagara 变量映射**：
| 函数 | Niagara 变量 | 映射 |
|------|-------------|------|
| `SetNiagaraDefaultValue` | Speed, LoopTime, Minimum, Maximum | 直接写入 float |
| `SetNiagaraSpawnCount` | SpawnCount | DMX[0,1] → Round(Lerp(0, MaxSpawnCount)) → int |
| `SetNiagaraColor` | Color | RGB → FLinearColor |

**蓝图中设置 Niagara 资产**：在 Components 面板选择 Niagara 组件 → Details → Niagara System Asset → 选择粒子系统

---

### 23.8 DMX 摄像机

**特征**：6 轴运动（XYZ 位移 + XYZ 旋转）+ 摄像机参数（FOV/光圈/对焦）

**基类**：`ASuperDMXCamera`（继承 `ASuperDmxActorBase`，**不是** ASuperLightBase）  
**内置组件**：`UCineCameraComponent` + `USceneCaptureComponent2D`

**灯库定义**（典型 9 通道）：

| 通道 | 属性名 | 分类 | 说明 |
|------|--------|------|------|
| 1 | PosX | Position | X 轴位移 |
| 2 | PosY | Position | Y 轴位移 |
| 3 | PosZ | Position | Z 轴位移 |
| 4 | RotX | Rotation | X 轴旋转 (Pitch) |
| 5 | RotY | Rotation | Y 轴旋转 (Yaw) |
| 6 | RotZ | Rotation | Z 轴旋转 (Roll) |
| 7 | FOV | Camera | 视场角 |
| 8 | Aperture | Camera | 光圈 |
| 9 | FocusDistance | Camera | 对焦距离 |

**蓝图配置**：
- `Start`：启动开关
- `BootRefresh`：点击初始化（记录当前位置/旋转为基准）
- `MovingRange`：XYZ 移动范围 (cm)
- `RotRange`：Pitch/Yaw/Roll 旋转范围 (°)
- `FOVRange`：FOV 范围（默认 15°~120°）
- `ApertureRange`：光圈范围（默认 f/1.2~f/22）
- `FocusDistanceRange`：对焦距离范围（默认 50~100000 cm）
- `RenderTarget`：渲染目标（用于 LED 屏幕输出）
- `bEnableSceneCapture`：启用场景捕获

**使用前必须**：
1. 放置 Actor 到场景中正确位置
2. 点击 `BootRefresh`（自动计算 InitialPosition/EndPosition、InitialRotation/EndRotation）
3. 设置 `Start = true`

**SuperDMXTick**：
```
DMXCameraControl(DmxPosX, DmxPosY, DmxPosZ, DmxRotX, DmxRotY, DmxRotZ)
DMXCameraParams(DmxFOV, DmxAperture, DmxFocusDistance)
```

**DMX 映射**：
```
位置: DMX[0,1] → Lerp(InitialPosition, EndPosition) + FInterpTo 平滑
旋转: DMX[0,1] → Lerp(InitialRotation, EndRotation) + FInterpTo 平滑
FOV:  DMX[0,1] → Lerp(FOVRange.X, FOVRange.Y)
光圈: DMX[0,1] → Lerp(ApertureRange.X, ApertureRange.Y)
对焦: DMX[0,1] → Lerp(FocusDistanceRange.X, FocusDistanceRange.Y)
```

---

### 23.9 升降机械

**特征**：6 轴运动（3 轴位移 + 3 轴旋转）、支持绝对/无极旋转

**基类**：`ASuperLiftingMachinery`（继承 `ASuperDmxActorBase`）

**灯库定义**（典型 6 通道）：

| 通道 | 属性名 | 分类 | 说明 |
|------|--------|------|------|
| 1 | PosX | Position | X 轴位移 |
| 2 | PosY | Position | Y 轴位移 |
| 3 | PosZ | Position | Z 轴位移 |
| 4 | RotX | Rotation | X 轴旋转 |
| 5 | RotY | Rotation | Y 轴旋转 |
| 6 | RotZ | Rotation | Z 轴旋转 |

**蓝图配置**：
- `Start`：启动开关
- `BootRefresh`：点击初始化
- `MovingRange`：XYZ 移动范围 (cm)
- `RotRange`：旋转范围 (°)（绝对模式）
- `PosSpeed`：位移插值速度 (0-10)
- `RotSpeed`：旋转插值速度 (0-10)
- `PolarRotation`：无极旋转模式开关
- `PolarRotationSpeed`：无极旋转速度 (0-10)

**使用前必须**：
1. 放置 Actor 到正确位置
2. 点击 `BootRefresh`
3. 设置 `Start = true`

**SuperDMXTick**：
```
LiftingMachinery(DmxPosX, DmxPosY, DmxPosZ, DmxRotX, DmxRotY, DmxRotZ)
```

**两种旋转模式**：
- **绝对模式**（PolarRotation=false）：DMX[0,1] → Lerp(InitialRotation, EndRotation)
- **无极模式**（PolarRotation=true）：DMX[0,1] → 速度[-PolarRotationSpeed, +PolarRotationSpeed]

---

### 23.10 轨道机械

**特征**：样条曲线路径运动 + 6 轴偏移/旋转，共 7 个 DMX 控制轴

**基类**：`ASuperRailMachinery`（继承 `ASuperDmxActorBase`）

**灯库定义**（典型 7 通道）：

| 通道 | 属性名 | 分类 | 说明 |
|------|--------|------|------|
| 1 | RailPos | Position | 轨道位置 [0=起点, 1=终点] |
| 2 | PosX | Position | X 轴偏移 |
| 3 | PosY | Position | Y 轴偏移 |
| 4 | PosZ | Position | Z 轴偏移 |
| 5 | RotX | Rotation | X 轴旋转 |
| 6 | RotY | Rotation | Y 轴旋转 |
| 7 | RotZ | Rotation | Z 轴旋转 |

**内置组件**：
- `RailSplineComponent`（USplineComponent）：定义轨道路径
- `RailMountPoint`：沿样条移动的挂载点
- `OffsetComponent`：局部 6 轴偏移/旋转

**蓝图配置**：
- `BootRefresh`：初始化偏移/旋转基准
- `bLockOrientationToRail`：锁定朝向沿轨道切线方向
- `bClosedLoop`：闭合轨道（首尾循环）
- `OffsetRange`：偏移范围 (cm)
- `RailSpeed` / `OffsetSpeed` / `RotSpeed`：各轴插值速度

**使用流程**：
1. 放置 Actor → 编辑 Spline 控制点定义轨道路径
2. 点击 `BootRefresh`
3. 设置 `Start = true`
4. 子 Actor 挂载到 `GetDefaultAttachComponent()`（即 OffsetComponent）

**SuperDMXTick**：
```
// 完整 7 轴控制
RailMachinery(DmxRailPos, DmxPosX, DmxPosY, DmxPosZ, DmxRotX, DmxRotY, DmxRotZ)

// 或仅轨道位置
RailPositionOnly(DmxRailPos)
```

**DMX 映射**：
```
轨道: DMX[0,1] → SplineLength × t → 沿样条位置
偏移: DMX[0,1] → Lerp(InitialOffset, EndOffset) + FInterpTo
闭合轨道: 最短弧路径插值（避免绕远路）
```

---

### 23.11 NDI 视频屏幕

**特征**：接收 NDI 视频流并显示到屏幕网格

**基类**：`ASuperNDIScreen`（继承 `ASuperBaseActor`，**无 DMX 控制**）  
**不需要灯库**：NDI 屏幕不通过 DMX 控制，直接接收网络视频流

**蓝图配置**：
- `InputName`：NDI 输入源名称（下拉选择）
- `TargetStaticMeshActors`：输出到的屏幕网格 Actor（支持多屏）
- `bTransparent`：透明模式
- `Transparency`：透明度 [0,1]
- `Brightness`：亮度
- `Color`：颜色滤镜
- `Contrast`：对比度
- `Deformation`：梯形校正（四角 UV 偏移）

**使用流程**：
1. 场景中放置 `ASuperNDIScreen`
2. 放置用作屏幕的 `AStaticMeshActor`（平面网格）
3. 将屏幕 Actor 添加到 `TargetStaticMeshActors` 数组
4. 选择 `InputName`（NDI 源名称）
5. 运行即自动接收并显示视频

**数据流**：
```
NDI 网络流 → SuperNDISubsystem → OnFrame 回调 → HandleNDIFrame →
  EnsureTexture（创建/重建纹理） → UpdateTextureGPU（RHI 上传） →
  DynamicMaterial.SetTextureParameterValue → 应用到 TargetStaticMeshActors
```

---

### 23.12 投影仪 / Mapping

**特征**：三通道 RGB SpotLight 实现 Projection Mapping

**基类**：`ASuperProjector`（继承 `ASuperBaseActor`，**无 DMX 控制**）  
**内置组件**：3 个 SpotLight（R/G/B 分通道投影）

**蓝图配置**：
- `MappingTexture`：投影纹理
- `MappingScale`：投影分辨率（默认 1920×1080）
- `Dimmer`：亮度 (%)
- `MaxLightDistance`：投影距离 (cm)
- `Zoom`：投影角度 (10-100°)
- `DilutionFactor`：边缘柔化 [0,1]
- `MappingDeformation`：梯形校正（四角偏移）

**投影原理**：
```
SpotLightR/G/B 分别设置 LightFunctionMaterial
  → 光照函数材质包含纹理
  → 梯形校正在 Shader 内实现
  → 三通道 RGB 叠加合成全彩投影
```

**使用流程**：
1. 放置 `ASuperProjector` Actor
2. 设置 `MappingTexture`（投影图片/视频）
3. 调整 `Zoom` 和 `MaxLightDistance` 使投影覆盖目标
4. 使用 `MappingDeformation` 进行梯形校正
5. 调整 `Dimmer` 控制亮度

---

### 23.13 激光（纹理模式）

**特征**：从 Beyond 软件获取激光纹理并在 UE 中可视化

**基类**：`ASuperLaserActor`（继承 `ASuperBaseActor`，**无 DMX**）

**蓝图配置**：
- `DeviceID`：Beyond 设备编号 (1-12)
- `LaserIntensity`：激光强度 (1-5000)
- `MaxLaserDistance`：投射距离 (100-10000 cm)
- `AtmosphericDensity`：大气衰减 [0,1]
- `FogIntensity`：雾效强度 (0-100)
- `AtmosFogSpeed`：雾效流速 (0-100)

**数据流**：
```
Beyond 设备 → SuperLaserSubsystem → GetLaserTexture(DeviceID) →
  SetLaserTexture → DynamicMaterialLaser → StaticMeshLaser
```

---

### 23.14 激光 Pro（点数据模式）

**特征**：从 Beyond 获取点数据，程序化网格绘制激光线

**基类**：`ASuperLaserProActor`（继承 `ASuperBaseActor`，**无 DMX**）  
**内置组件**：`USuperLaserProComponent`

**蓝图配置**：
- `DeviceID`：Beyond 设备编号 (1-12)
- `bEnableCollision`：碰撞遮挡检测
- LaserComponent 上的参数：BeamLength, ProjectionAngle, LaserWidth, CoreSharpness 等

**数据流**：
```
Beyond 设备 → SuperLaserSubsystem → GetLaserPoints(DeviceID) →
  LaserComponent.SetLaserPoints → BuildLaserMesh → ProceduralMeshComponent
```

**使用流程**：
1. 放置 `ASuperLaserProActor`
2. 设置 `DeviceID`
3. 调整 LaserComponent 参数（BeamLength, ProjectionAngle 等）
4. 确保 Beyond 软件正在运行并输出到对应设备

---

### 23.15 升降矩阵

**特征**：5 组升降效果组件 + 钢丝绳可视化

**基类**：`ASuperLiftMatrix`（继承 `ASuperLightBase`，有 Pan/Tilt）

**灯库定义**（典型通道）：

| 通道 | 属性名 | 分类 | 说明 |
|------|--------|------|------|
| 1-2 | Pan | Position | 水平旋转 |
| 3-4 | Tilt | Position | 垂直旋转 |
| 5 | Lift | Position | 升降位置 |
| 6 | EffectSelect | Effect | 效果选择 |
| 7 | EffectColorR | Color | 效果红色 |
| 8 | EffectColorG | Color | 效果绿色 |
| 9 | EffectColorB | Color | 效果蓝色 |

**内置组件结构**：
```
SceneHard
 ├─ LiftComponents[0..4]（5 个场景组件，间距 5cm）
 │   └─ EffectComponents[0,1]（每个下挂 2 个 SuperEffectComponent，共 10 个）
 ├─ CableOrigin（钢丝绳起点）
 └─ CableMeshes[0..3]（4 根钢丝绳圆柱体，四角）
```

**初始化**（OnConstruction + BeginPlay）：
```
SetEffectMatrixDefaultValue()   // 初始化所有效果组件材质
```

**SuperDMXTick**：
```
Pan(DmxPan)
Tilt(DmxTilt)
LiftMatrix(DmxLift)                                          // 升降控制
SetEffectMatrix(DmxEffectSelect)                              // 效果选择
SetEffectColorMatrix(DmxColorR, DmxColorG, DmxColorB)        // 效果颜色
```

---

### 23.16 灯具类型速查表

| 灯具类型 | 基类 | 核心组件 | 有 DMX | 有 Pan/Tilt | 典型通道数 |
|---------|------|---------|--------|------------|-----------|
| Beam 灯 | ASuperStageLight | USuperBeamComponent | ✅ | ✅ | 15-20 |
| Spot 灯 | ASuperStageLight | USuperBeamComponent | ✅ | ✅ | 18-25 |
| Wash 灯 | ASuperStageLight | USuperSpotComponent | ✅ | ✅ | 10-15 |
| Profile 灯 | ASuperStageLight | USuperCuttingComponent | ✅ | ✅ | 25-35 |
| 矩阵灯 | ASuperStageLight | USuperMatrixComponent | ✅ | ✅ | 30-200+ |
| LED 灯条 | ASuperLightStripEffect | 内置效果材质 | ✅ | ❌ | 8-10 |
| VFX 特效 | ASuperStageVFXActor | UNiagaraComponent | ✅ | ✅ | 5-10 |
| DMX 摄像机 | ASuperDMXCamera | UCineCameraComponent | ✅ | ❌ | 6-9 |
| 升降机械 | ASuperLiftingMachinery | 无（仅运动） | ✅ | ❌ | 6 |
| 轨道机械 | ASuperRailMachinery | USplineComponent | ✅ | ❌ | 7 |
| NDI 屏幕 | ASuperNDIScreen | 内置纹理系统 | ❌ | ❌ | 0 |
| 投影仪 | ASuperProjector | 3×SpotLight | ❌ | ❌ | 0 |
| 激光(纹理) | ASuperLaserActor | 内置材质 | ❌ | ❌ | 0 |
| 激光(Pro) | ASuperLaserProActor | USuperLaserProComponent | ❌ | ❌ | 0 |
| 升降矩阵 | ASuperLiftMatrix | USuperEffectComponent×10 | ✅ | ✅ | 8-12 |

---

## 24. 灯库编辑器使用指南

### 24.1 创建灯库资产

在 Content Browser 中 **右键** → 找到 **SuperStage** 分类 → 点击 **Super灯库** 创建新的灯库资产。

> 资产由 `UFixtureLibraryFactory` 工厂创建，默认包含一个名为 `Mode 1` 的空模块。

### 24.2 打开灯库编辑器

在 Content Browser 中 **双击** 已有的 `SuperFixtureLibrary` 资产即可打开灯库编辑器。

> 底层流程：`FAssetTypeActions_FixtureLibrary::OpenAssetEditor` → 创建 `FFixtureLibraryEditorToolkit` → 内嵌 `SFixtureLibraryEditor` 控件。

### 24.3 编辑器布局

```
┌──────────────────────────────────────────┐
│ 💡 灯具名称: [___] 制造商: [___] 来源: [▼] 功率: [___] 质量: [___] │
├──────────────────────────────────────────┤
│ 面包屑导航: 模块列表 > Mode 1 > Dimmer > SubAttr 1 > ChannelSet │
├────────┬─────────────────────────────────┤
│ 模块列表 │ 属性列表 / 子属性列表 / 槽位列表（根据导航层级切换）│
│ Mode 1  │ No | Attrib | Category | Coarse | Fine | Ultra | Default │
│ Mode 2  │  1 | Dimmer |   亮度   |   1    |  2   |  0    |   0     │
│ [+][-]  │  2 | Pan    |   位置   |   3    |  4   |  0    |  128    │
│         │  3 | Tilt   |   位置   |   5    |  6   |  0    |  128    │
│         │  4 | ColorR |   颜色   |   7    |  0   |  0    |   0     │
│         │ [+ Attribute] [- Attribute]                               │
├──────────────────────────────────────────┤
│ [通道总数: 12] [保存] [另存为]                                     │
└──────────────────────────────────────────┘
```

### 24.4 操作流程

1. **添加模块** — 左侧模块列表点击 `[+]`
2. **选择模块** — 点击模块名称进入属性列表
3. **添加属性** — 点击 `[+ Attribute]`
4. **编辑属性** — 直接在表格中修改名称、分类、通道号
5. **进入子属性** — 点击属性行进入子属性层级
6. **编辑子属性** — 设置 DmxMin/DmxMax/PhysicalRange
7. **进入槽位** — 点击子属性行进入槽位层级
8. **编辑槽位** — 根据属性分类显示不同编辑控件：
   - **图案类**：GoboMode + Texture 选择器
   - **颜色类**：Color 拾色器 + ColorIndex
   - **棱镜类**：PrismFacets + PrismRadius + PrismScale

### 24.5 属性名约定

属性名 `AttribName` 是灯库与 Actor 之间的**唯一链接**。必须：
- 与 `FSuperDMXAttribute.AttribName` 完全一致（大小写敏感）
- 同一模块内不允许重复
- 推荐使用英文命名（参见附录）

---

## 25. 常见问题与排错

### Q1: 灯具无反应

**排查步骤**：
1. 确认 `SuperDMXFixture` 的 Universe 和 StartAddress 正确
2. 确认 `FixtureLibrary` 已赋值
3. 确认属性名匹配（大小写敏感）
4. 确认 `SetLightingDefaultValue` 已在控制函数之前调用
5. 确认 DMX 数据源（Art-Net/sACN）正在发送

### Q2: 灯光闪烁

**可能原因**：
- 频闪通道未正确配置（StrobeMode 子属性缺失 → 默认 Open）
- DimmerCurveExponent 设置为 1.0（线性响应导致低亮度抖动）
- Tick/SuperDMXTick 中重复调用 SetLightingDefaultValue（应仅在 OnConstruction 和 BeginPlay 中各调用一次）

### Q3: 颜色不准

**排查步骤**：
1. 确认 RGB 通道的 DMXChannelType 正确（通常为 8-bit）
2. 颜色轮模式下确认 ColorAtlas 纹理已设置
3. CMY 混色确认公式方向（C=1 表示全青，减去红色）
4. CTO 通道确认温度范围方向

### Q4: 矩阵灯只有部分像素工作

**排查步骤**：
1. 确认灯库中每个模块的 `Patch` 正确（区分不同像素的地址偏移）
2. 确认所有模块的属性名完全一致
3. 确认 `USuperMatrixComponent.SegCount` 与模块数量匹配
4. 使用 `GetChannelSpan()` 验证总通道数

### Q5: 编辑器中看不到通道参数

确认 `bChannelEdit = true`（在蓝图子类的 Class Defaults 中勾选）。

### Q6: 材质未初始化（组件全黑/透明）

确保在蓝图的 `OnConstruction`（构造脚本）和 `BeginPlay` 中各调用一次：
- 光束组件：`SetLightingDefaultValue(SuperLighting)`
- 效果组件：`SetEffectDefaultValue(SuperEffect)`
- 矩阵组件：`SetMatrixMaterial()`（在 `USuperMatrixComponent` 上调用）

**不要**将这些初始化函数放在 `Tick` 或 `SuperDMXTick` 中。

---

## 26. 附录：标准属性名对照表

### 位置类

| 属性名 | 说明 | 典型精度 | 备注 |
|--------|------|---------|------|
| `Pan` | 水平旋转 | 16-bit | 默认 ±270° |
| `Tilt` | 垂直旋转 | 16-bit | 默认 ±135° |
| `PTSpeed` | PT 速度 | 8-bit | 0=最快 |
| `PanRot` | Pan 无极旋转 | 8-bit | 需子属性 RotationMode |
| `TiltRot` | Tilt 无极旋转 | 8-bit | 需子属性 RotationMode |
| `InPosZ` | 升降位置 | 8-bit | 0=最低, 1=最高 |

### 亮度/频闪类

| 属性名 | 说明 | 典型精度 |
|--------|------|---------|
| `Dimmer` | 亮度 | 16-bit |
| `Strobe` | 频闪 | 8-bit |
| `StrobeMode` | 频闪模式 | 8-bit |

### 颜色类

| 属性名 | 说明 | 典型精度 |
|--------|------|---------|
| `ColorR` | 红色 | 8-bit |
| `ColorG` | 绿色 | 8-bit |
| `ColorB` | 蓝色 | 8-bit |
| `ColorW` | 白色 | 8-bit |
| `ColorWheel` / `Color1` | 颜色轮 1 | 8-bit |
| `Color2` / `Color3` | 颜色轮 2/3 | 8-bit |
| `CyanMix` / `MagentaMix` / `YellowMix` | CMY 混色 | 8-bit |
| `CTO` | 色温调节 | 8-bit |
| `Cool` / `Warm` | 冷暖混色 | 8-bit |
| `Hue` / `Saturation` | 色相/饱和度 | 8-bit |

### 光束类

| 属性名 | 说明 | 典型精度 |
|--------|------|---------|
| `Zoom` | 变焦 | 8-bit 或 16-bit |
| `Focus` | 调焦 | 8-bit |
| `Frost` | 雾化 | 8-bit |
| `Iris` | 光圈 | 8-bit |

### 图案类

| 属性名 | 说明 | 典型精度 |
|--------|------|---------|
| `Gobo1` | 图案轮 1 | 8-bit |
| `Gobo1Rot` | 图案轮 1 旋转 | 8-bit |
| `Gobo2` / `Gobo2Rot` | 图案轮 2 | 8-bit |
| `Gobo3` / `Gobo3Rot` | 图案轮 3 | 8-bit |

### 棱镜类

| 属性名 | 说明 | 典型精度 |
|--------|------|---------|
| `Prism1` | 棱镜 1 | 8-bit |
| `Prism2` | 棱镜 2 | 8-bit |
| `PrismRot` | 棱镜旋转 | 8-bit |

### 切割类

| 属性名 | 说明 | 典型精度 |
|--------|------|---------|
| `A1`~`A4` | 叶片 A 侧 (1-4) | 8-bit |
| `B1`~`B4` | 叶片 B 侧 (1-4) | 8-bit |
| `ShaperRot` | 切割系统旋转 | 8-bit |

### 效果类

| 属性名 | 说明 | 典型精度 |
|--------|------|---------|
| `EffectDimmer` | 效果亮度 | 8-bit |
| `EffectStrobe` | 效果频闪 | 8-bit |
| `EffectValue` | 效果选择 | 8-bit |
| `EffectColorR/G/B` | 效果颜色 | 8-bit |

### 矩阵类

| 属性名 | 说明 | 典型精度 |
|--------|------|---------|
| `MatrixDimmer` | 矩阵亮度 | 8-bit |
| `MatrixStrobe` | 矩阵频闪 | 8-bit |
| `MatrixR/G/B` | 矩阵颜色 | 8-bit |
| `MatrixW` | 矩阵白色 | 8-bit |

> **注意**：以上属性名仅为推荐约定，实际可自由命名。唯一要求是灯库中的 `AttribName` 与 Actor 中的 `FSuperDMXAttribute.AttribName` 完全一致。

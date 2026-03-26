# ASuperLightBase & ASuperStageLight API 文档

> **文档版本**: 1.0  
> **适用版本**: SuperStage Plugin 2025/2026  
> **目标读者**: 蓝图开发者 / C++ 插件集成开发者（无源码修改权限）  
> **最后更新**: 2026-03

---

## 目录

1. [ASuperLightBase — Pan/Tilt 运动控制层](#1-asuperlightbase--pantilt-运动控制层)
   - [组件结构](#11-组件结构)
   - [默认参数](#12-默认参数)
   - [控制参数](#13-控制参数)
   - [通道开关](#14-通道开关)
   - [位置控制函数](#15-位置控制函数)
   - [矩阵位置控制](#16-矩阵位置控制)
   - [无极旋转控制](#17-无极旋转控制)
   - [矩阵无极旋转控制](#18-矩阵无极旋转控制)
   - [辅助控制](#19-辅助控制)
2. [ASuperStageLight — 全功能电脑灯](#2-asuperstagelight--全功能电脑灯)
   - [组件结构](#21-组件结构)
   - [默认参数](#22-默认参数)
   - [控制参数](#23-控制参数)
   - [通道开关](#24-通道开关)
   - [初始化函数](#25-初始化函数)
   - [亮度控制](#26-亮度控制)
   - [频闪控制](#27-频闪控制)
   - [颜色控制](#28-颜色控制)
   - [光学控制](#29-光学控制)
   - [图案轮控制](#210-图案轮控制)
   - [棱镜控制](#211-棱镜控制)
   - [切割控制](#212-切割控制)
   - [效果控制](#213-效果控制)
   - [矩阵组件控制](#214-矩阵组件控制)
   - [Spot 辅光控制](#215-spot-辅光控制)
3. [FSuperColorChannel — 动态混色通道](#3-fsupercolorchannel--动态混色通道)
4. [FLightDefaultValue — 灯光默认参数](#4-flightdefaultvalue--灯光默认参数)
5. [Pan/Tilt 运动系统详解](#5-pantilt-运动系统详解)
6. [颜色系统详解](#6-颜色系统详解)
7. [使用示例](#7-使用示例)
8. [API 速查表](#8-api-速查表)

---

## 1. ASuperLightBase — Pan/Tilt 运动控制层

**头文件**: `LightActor/SuperLightBase.h`  
**基类**: `ASuperDmxActorBase`  
**导出宏**: `SUPERSTAGE_API`

提供水平（Pan）和垂直（Tilt）旋转控制基础功能。所有具有运动能力的电脑灯 Actor 都继承自此类。

### 继承链

```
AActor → ASuperBaseActor → ASuperDmxActorBase → ASuperLightBase ← 本类
```

---

### 1.1 组件结构

```
SceneBase (继承自 ASuperDmxActorBase)
 └─ SceneRocker (Pan 旋转轴, Yaw)
     └─ SceneHard (Tilt 旋转轴, Roll)
         └─ 灯头组件（光束/模型，由子类创建）
```

| 组件 | 类型 | 类别 | 说明 |
|------|------|------|------|
| `SceneRocker` | `USceneComponent*` | `Z.Components` | Pan 旋转轴，绕 Yaw 旋转 |
| `SceneHard` | `USceneComponent*` | `Z.Components` | Tilt 旋转轴，绕 Roll 旋转 |

> **旋转轴映射**: Pan → `SceneRocker.Yaw`，Tilt → `SceneHard.Roll`。其他轴保持不变。

---

### 1.2 默认参数

| 属性 | 类型 | 默认值 | 范围 | 说明 |
|------|------|--------|------|------|
| `PanRange` | `FVector2D` | `(-270, 270)` | — | Pan 角度映射范围（度） |
| `TiltRange` | `FVector2D` | `(-135, 135)` | — | Tilt 角度映射范围（度） |
| `InfiniteRotationalSpeed` | `float` | `1.0` | 0.01-10 | 无极旋转速度倍率 |
| `PanAngle` | `float` | `0` | -180~180 | 手动 Pan 角度（度） |
| `TiltAngle` | `float` | `0` | -180~180 | 手动 Tilt 角度（度） |
| `LiftSpeed` | `float` | `1.0` | 0-10 | 升降插值速度 |
| `LiftRange` | `float` | `500.0` | — | 升降范围（cm） |

**Pan/Tilt 映射公式**:

```
目标角度 = Lerp(Range.X, Range.Y, DMX归一化值)

示例（默认 PanRange）:
  DMX = 0.0 → -270°
  DMX = 0.5 → 0° (中心)
  DMX = 1.0 → +270°
```

---

### 1.3 控制参数

这些属性由 DMX 函数自动更新，也可在 Details 面板手动调整。所有值为 `[0, 1]` 归一化。

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `Pan` | `float` | `0.5` | 当前 Pan 归一化值 |
| `Tilt` | `float` | `0.5` | 当前 Tilt 归一化值 |
| `PTSpeed` | `float` | `5.0` | Pan/Tilt 插值速度（`FInterpTo` 参数） |
| `PanRot` | `float` | `0.5` | 无极 Pan 旋转控制值 |
| `TiltRot` | `float` | `0.5` | 无极 Tilt 旋转控制值 |
| `InPosZ` | `float` | `0.0` | 升降位置值 |

---

### 1.4 通道开关

通道开关控制 Details 面板中对应控制参数的可见性。需先启用 `bChannelEdit = true` 才可见。

| 开关 | 默认值 | 控制的参数 |
|------|--------|-----------|
| `bChannelEdit` | `false` | 显示/隐藏所有通道开关 |
| `bPan` | `false` | Pan 控制参数 |
| `bTilt` | `false` | Tilt 控制参数 |
| `bPTSpeed` | `false` | PTSpeed 控制参数 |
| `bPanRot` | `false` | PanRot 控制参数 |
| `bTiltRot` | `false` | TiltRot 控制参数 |
| `bLampAngle` | `false` | PanAngle/TiltAngle 手动角度 |
| `bYPolarRotation` | `false` | InfiniteRotationalSpeed 无极旋转速度 |
| `bInPosZ` | `false` | InPosZ 升降位置 |

---

### 1.5 位置控制函数

#### YPan

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|Base|Position",
    meta = (DisplayName = "Pan"))
void YPan(FSuperDMXAttribute DmxPan);
```

控制灯具水平旋转（Pan）。

**行为**:
1. 调用 `GetSuperDmxAttributeValue()` 读取归一化 DMX 值到 `Pan`
2. 计算目标角度: `Lerp(PanRange.X, PanRange.Y, Pan)`
3. 使用 `FInterpTo` 平滑插值 `CurrentPan`（速度 = `PTSpeed`）
4. 设置 `SceneRocker` 的 Yaw 旋转

**参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| `DmxPan` | `FSuperDMXAttribute` | Pan 通道的 DMX 属性 |

---

#### YTilt

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|Base|Position",
    meta = (DisplayName = "Tilt"))
void YTilt(FSuperDMXAttribute DmxTilt);
```

控制灯具垂直旋转（Tilt）。

**行为**: 与 `YPan` 对称，使用 `TiltRange` 映射，设置 `SceneHard` 的 Roll 旋转。

---

#### YPTSpeed

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|Base|Position",
    meta = (DisplayName = "PTSpeed"))
void YPTSpeed(FSuperDMXAttribute DmxAttribute);
```

动态调整 Pan/Tilt 的插值速度。

**行为**:
1. 读取 DMX 原始值（`GetSuperDmxAttributeValueNoConversion`）
2. 如果灯库定义了子属性（`SubAttribute`），使用物理范围映射
3. 否则归一化: `PTSpeed /= 255.0`

**影响范围**: 所有使用 `FInterpTo` 的旋转函数（`YPan`、`YTilt`、矩阵 Pan/Tilt）

---

### 1.6 矩阵位置控制

#### YMatrixPan

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|Base|Matrix",
    meta = (DisplayName = "MatrixPan"))
void YMatrixPan(FSuperDMXAttribute DmxPan, TArray<USceneComponent*> MatrixHard);
```

矩阵模式下控制多个灯头的 Pan 旋转。每个灯头独立插值。

**参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| `DmxPan` | `FSuperDMXAttribute` | Pan 通道属性（矩阵模式读取） |
| `MatrixHard` | `TArray<USceneComponent*>` | 灯头组件数组（每个元素对应一个像素） |

**行为**:
- 使用 `GET_SUPER_DMX_MATRIX_VALUE` 宏遍历矩阵数据
- 每个灯头使用 `FInterpTo` 平滑插值到目标角度
- 自动管理 `CurrentMatrixPan` 内部数组大小

---

#### YMatrixTilt

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|Base|Matrix",
    meta = (DisplayName = "MatrixTilt"))
void YMatrixTilt(FSuperDMXAttribute DmxTilt, TArray<USceneComponent*> MatrixRocker);
```

矩阵模式下控制多个灯头的 Tilt 旋转。与 `YMatrixPan` 对称。

---

### 1.7 无极旋转控制

无极旋转支持四种模式（由灯库 `SubAttribute.RotationMode` 定义）：

| RotationMode | 行为 |
|-------------|------|
| `Off` | 回退到普通位置控制（`YPan`/`YTilt`） |
| `Stop` | 保持当前角度不动 |
| `Position` | 从当前角度开始，按物理范围偏移旋转 |
| `Infinite` | 连续旋转，速度由物理范围映射值决定 |

#### YPanRot

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|Base|Infinite",
    meta = (DisplayName = "PanRot"))
void YPanRot(FSuperDMXAttribute DmxInfinity, FSuperDMXAttribute DmxPan);
```

Pan 无极旋转控制。

**参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| `DmxInfinity` | `FSuperDMXAttribute` | 无极旋转控制通道 |
| `DmxPan` | `FSuperDMXAttribute` | Pan 位置通道（`Off` 模式时使用） |

**行为**:
1. 读取 `DmxInfinity` 原始值
2. 查找对应子属性的 `RotationMode`
3. 无子属性或 `Off` → 调用 `YPan(DmxPan)` 位置控制
4. `Stop` → 不做任何操作
5. `Position` → 从进入时的角度开始偏移旋转
6. `Infinite` → `SceneRocker->AddRelativeRotation(Speed * InfiniteRotationalSpeed)`

---

#### YTiltRot

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|Base|Infinite",
    meta = (DisplayName = "TiltRot"))
void YTiltRot(FSuperDMXAttribute DmxInfinity, FSuperDMXAttribute DmxTilt);
```

Tilt 无极旋转控制。与 `YPanRot` 对称。

---

### 1.8 矩阵无极旋转控制

#### YMatrixPanRot / YMatrixTiltRot

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|Base|Infinite",
    meta = (DisplayName = "MatrixPanRot"))
void YMatrixPanRot(FSuperDMXAttribute DmxInfinity, FSuperDMXAttribute DmxPan,
    TArray<USceneComponent*> MatrixRocker);

UFUNCTION(BlueprintCallable, Category = "A.Yunsio|Base|Infinite",
    meta = (DisplayName = "MatrixTiltRot"))
void YMatrixTiltRot(FSuperDMXAttribute DmxInfinity, FSuperDMXAttribute DmxTilt,
    TArray<USceneComponent*> MatrixHard);
```

矩阵模式下的无极旋转控制。每个灯头独立判断 `RotationMode` 并执行。

---

### 1.9 辅助控制

#### YLampAngle

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|Base|Angle",
    meta = (DisplayName = "LampAngle"))
void YLampAngle();
```

将 `SceneRocker`/`SceneHard` 设置为 `PanAngle`/`TiltAngle` 固定角度。用于手动对灯（非 DMX 控制）。

---

#### SetLiftZ

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|Base|Lift",
    meta = (DisplayName = "SetLiftZ"))
void SetLiftZ(USuperLiftComponent* NewSuperLift, FSuperDMXAttribute DmxLiftZ);
```

控制灯具升降。

**参数**:

| 参数 | 类型 | 说明 |
|------|------|------|
| `NewSuperLift` | `USuperLiftComponent*` | 升降组件引用 |
| `DmxLiftZ` | `FSuperDMXAttribute` | 升降通道 DMX 属性 |

---

## 2. ASuperStageLight — 全功能电脑灯

**头文件**: `LightActor/SuperStageLight.h`  
**基类**: `ASuperLightBase`  
**导出宏**: `SUPERSTAGE_API`

完整的电脑灯实现类，涵盖光束、颜色、图案、棱镜、切割、效果和矩阵控制。适用于 Martin、Robe、ClayPaky 等各品牌专业电脑灯。

### 继承链

```
AActor → ASuperBaseActor → ASuperDmxActorBase → ASuperLightBase → ASuperStageLight ← 本类
```

---

### 2.1 组件结构

```
SceneBase
 ├─ SM_Hook (UStaticMeshComponent, 灯钩)
 ├─ SM_Base (UStaticMeshComponent, 底座)
 └─ SceneRocker (Pan 轴)
     ├─ SM_Rocker (UStaticMeshComponent, 灯臂)
     └─ SceneHard (Tilt 轴)
         ├─ SM_Hard (UStaticMeshComponent, 灯头)
         ├─ SuperLighting (USuperLightingComponent, 主光源)
         ├─ SuperLightingArray[] (USuperLightingComponent[], 矩阵光源)
         ├─ SuperEffect (USuperEffectComponent, 效果光源)
         ├─ SuperEffectArray[] (USuperEffectComponent[], 效果矩阵)
         ├─ SpotLighting (USuperSpotComponent, Spot 辅光)
         └─ SuperSpotArray[] (USuperSpotComponent[], Spot 矩阵)
```

> **注意**: `SuperLighting`、`SuperLightingArray`、`SuperEffect`、`SuperEffectArray`、`SpotLighting`、`SuperSpotArray` 均为 `Transient`，由蓝图 `SuperDMXTick` 中调用初始化函数时设置。

---

### 2.2 默认参数

| 属性 | 类型 | 类别 | 说明 |
|------|------|------|------|
| `LightDefaultValue` | `FLightDefaultValue` | `B.DefaultParameter` | 灯光默认参数集合 |
| `bHookVisibility` | `bool` | `F.Preview` | 灯钩模型显示开关 |
| `bModelVisibility` | `bool` | `F.Preview` | 灯具模型显示开关 |

---

### 2.3 控制参数

所有控制参数支持 `Interp`（Sequencer 动画）。带 `EditCondition` 的参数仅在对应通道开关启用时显示。

#### 基础参数

| 属性 | 类型 | 默认值 | 范围 | 说明 |
|------|------|--------|------|------|
| `Dimmer` | `float` | `1.0` | 0-1 | 亮度（0=全灭，1=全亮） |
| `Strobe` | `float` | `1.0` | 0-1 | 频闪速度 |
| `Zoom` | `float` | `0.0` | 0-1 | 光束角/光斑大小 |
| `Focus` | `float` | `0.0` | 0-1 | 聚焦 |
| `Frost` | `float` | `0.0` | 0-1 | 雾化效果强度 |
| `Iris` | `float` | `1.0` | 0-1 | 光圈大小（1=全开，0=全关） |

#### 颜色参数

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `Color` | `FLinearColor` | `(1,1,1)` | RGB 颜色 |
| `ColorWheel` | `float` | `0` | 颜色轮位置（0-255） |
| `ColorTemperature` | `float` | `0` | 色温控制值 |
| `Cool` | `float` | `0` | 冷光通道 |
| `Warm` | `float` | `0` | 暖光通道 |

#### 图案/棱镜参数

| 属性 | 类型 | 默认值 | 范围 | 说明 |
|------|------|--------|------|------|
| `Gobo1` | `float` | `0` | 0-255 | 图案轮 1 选择 |
| `Gobo2` | `float` | `0` | 0-255 | 图案轮 2 选择 |
| `Gobo_Rot` | `float` | `0` | 0-255 | 图案旋转 |
| `Prism1` | `float` | `0` | 0-255 | 棱镜 1 选择 |
| `Prism2` | `float` | `0` | 0-255 | 棱镜 2 选择 |
| `Prism_Rot` | `float` | `0` | 0-255 | 棱镜旋转 |

#### 切割参数

| 属性 | 说明 |
|------|------|
| `A1`, `B1` | 叶片对 1（0-1） |
| `A2`, `B2` | 叶片对 2 |
| `A3`, `B3` | 叶片对 3 |
| `A4`, `B4` | 叶片对 4 |
| `ShaperRot` | 切割系统旋转（0-1） |

#### 效果参数

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `EffectDimmer` | `float` | `0` | 效果层亮度 |
| `EffectStrobe` | `float` | `0` | 效果层频闪 |
| `EffectValue` | `float` | `0` | 效果选择（0-255，LUT 查表） |
| `EffectColor` | `FLinearColor` | `(1,1,1)` | 效果层颜色 |

---

### 2.4 通道开关

| 开关 | 功能 |
|------|------|
| `bDimmer` | 亮度通道 |
| `bStrobe` | 频闪通道 |
| `bZoom` | 变焦通道 |
| `bFocus` | 聚焦通道 |
| `bFrost` | 雾化通道 |
| `bIris` | 光圈通道 |
| `bColor` | RGB 颜色通道 |
| `bColorWheel` | 颜色轮通道 |
| `bColorTemperature` | 色温通道 |
| `bCoolWarmMix` | 冷暖混色通道 |
| `bGobo1` / `bGobo2` | 图案轮 1/2 通道 |
| `bGobo_Rot` | 图案旋转通道 |
| `bPrism1` / `bPrism2` / `bPrism3` | 棱镜 1/2/3 通道 |
| `bPrism_Rot` | 棱镜旋转通道 |
| `bEffect` | 效果通道 |
| `bCuttingChannel` | 切割通道 |
| `bPositionZ` | 升降通道 |
| `bYRotation` | 旋转通道 |

---

### 2.5 初始化函数

在 `SuperDMXTick` 事件的**首帧**调用，设置组件默认值。

#### SetLightingDefaultValue

```cpp
UFUNCTION(BlueprintCallable, Category="A.Yunsio|1.Lighting|1.Init",
    meta = (DisplayName = "SetLightingDefaultValue"))
void SetLightingDefaultValue(USuperLightingComponent* NewSuperLighting);
```

初始化单个主光源组件的默认参数（亮度上限、距离、体积雾、阴影等）。

**参数**: `NewSuperLighting` — 主光源组件引用（蓝图中拖入组件引用）

---

#### SetLightingMatrixDefaultValue

```cpp
UFUNCTION(BlueprintCallable, Category="A.Yunsio|1.Lighting|1.Init",
    meta = (DisplayName = "SetLightingMatrixDefaultValue"))
void SetLightingMatrixDefaultValue(TArray<USuperLightingComponent*> NewSuperLightingArray);
```

初始化矩阵模式的光源组件数组。

---

#### SetLightingDefaultTexture / SetLightingDefaultTextureMatrix

```cpp
void SetLightingDefaultTexture(UTexture2D* NewTexture);
void SetLightingDefaultTextureMatrix(UTexture2D* NewTexture);
```

设置默认 Gobo 纹理（无图案轮选择时的默认图案）。

---

### 2.6 亮度控制

#### SetLightingIntensity

```cpp
UFUNCTION(BlueprintCallable, Category="A.Yunsio|1.Lighting|2.Dimmer",
    meta = (DisplayName = "SetLightingIntensity"))
void SetLightingIntensity(FSuperDMXAttribute DmxAttribute);
```

通过 DMX 设置主光源亮度。

**行为**:
1. 读取 DMX 归一化值 → `Dimmer`
2. 应用亮度曲线: `pow(Dimmer, DimmerCurveExponent)`
3. 调用 `SuperLighting->SetLightingIntensity()`

---

#### SetLightingIntensityMatrix

```cpp
UFUNCTION(BlueprintCallable, Category="A.Yunsio|1.Lighting|2.Dimmer",
    meta = (DisplayName = "SetLightingIntensityMatrix"))
void SetLightingIntensityMatrix(FSuperDMXAttribute DmxAttribute);
```

矩阵模式亮度控制，每个像素独立。

---

### 2.7 频闪控制

#### SetLightingStrobe / SetLightingStrobeMatrix

```cpp
void SetLightingStrobe(FSuperDMXAttribute DmxAttribute);
void SetLightingStrobeMatrix(FSuperDMXAttribute DmxAttribute);
```

频闪控制。支持子属性系统定义的频闪模式：

| 频闪模式 | 说明 |
|---------|------|
| Open | 常亮（无频闪） |
| Strobe | 标准频闪 |
| Pulse | 脉冲频闪 |
| Random | 随机频闪 |
| Close | 常灭 |

---

### 2.8 颜色控制

ASuperStageLight 提供多种颜色控制方案，覆盖所有主流灯具的颜色系统。

#### RGB 直控

```cpp
// 单灯
void SetLightingColorRGB(FSuperDMXAttribute DMXAttR, FSuperDMXAttribute DMXAttG,
    FSuperDMXAttribute DMXAttB);
// 矩阵
void SetLightingColorRGBMatrix(FSuperDMXAttribute DMXAttR, FSuperDMXAttribute DMXAttG,
    FSuperDMXAttribute DMXAttB);
```

三通道独立 RGB 颜色控制。每个通道 `[0, 1]` 归一化。

---

#### RGBW 混合

```cpp
// 单灯
void SetLightingColorRGBW(FSuperDMXAttribute DMXAttR, FSuperDMXAttribute DMXAttG,
    FSuperDMXAttribute DMXAttB, FSuperDMXAttribute DMXAttW);
// 矩阵
void SetLightingColorRGBWMatrix(...);
// RGBW + CTO
void SetLightingColorRGBWWithCTO(..., FSuperDMXAttribute DmxCTO,
    float CoolTemperature = 6500.f, float WarmTemperature = 3200.f);
void SetLightingColorRGBWMatrixWithCTO(...);
```

RGBW 四通道混色。白色通道叠加到 RGB 上。可选 CTO（色温橙）通道调节。

---

#### HSV 颜色

```cpp
// 色相/饱和度/明度
void SetLightingColorHSV(FSuperDMXAttribute DMXAttH, FSuperDMXAttribute DMXAttS,
    FSuperDMXAttribute DMXAttV);
void SetLightingColorHSVMatrix(...);

// 色相/饱和度（明度由 Dimmer 通道控制）
void SetLightingColorHS(FSuperDMXAttribute DMXAttH, FSuperDMXAttribute DMXAttS);
void SetLightingColorHSMatrix(...);

// HS + CTO
void SetLightingColorHSAndCTO(FSuperDMXAttribute DMXAttH, FSuperDMXAttribute DMXAttS,
    FSuperDMXAttribute DMXAttCTO, float WarmTemperature = 2700.f, float CoolTemperature = 6500.f);
```

HSV 色彩空间控制。`SetLightingColorHS` 版本中 V（明度）由 `Dimmer` 通道独立控制。

---

#### 颜色轮

```cpp
// 单颜色轮
void SetLightingColorWheel(FSuperDMXAttribute DmxAttribute, UTexture2D* ColorAtlas = nullptr);

// 颜色轮 + CMY
void SetLightingColorWheelAndCMY(FSuperDMXAttribute DmxAttribute,
    FSuperDMXAttribute DMXAttC, FSuperDMXAttribute DMXAttM, FSuperDMXAttribute DMXAttY,
    UTexture2D* ColorAtlas = nullptr);

// 双颜色轮
void SetLightingColorWheel2(FSuperDMXAttribute DmxAttribute1, FSuperDMXAttribute DmxAttribute2,
    UTexture2D* ColorAtlas1 = nullptr, UTexture2D* ColorAtlas2 = nullptr);

// 三颜色轮
void SetLightingColorWheel3(...);

// 双颜色轮 + CMY
void SetLightingColorWheel2AndCMY(...);

// 双颜色轮 + CMY + CTO
void SetLightingColorWheel2AndCMYAndCTO(..., FSuperDMXAttribute DMXAttCTO,
    float WarmTemperature = 2700.f, float CoolTemperature = 6500.f, ...);

// 三颜色轮 + CMY
void SetLightingColorWheel3AndCMY(...);
```

颜色轮使用灯库中定义的 `ColorWheelModel` 查表，支持：
- **固定色**: DMX 值在 `StaticColorRange` 区间 → 选择预设颜色
- **流水**: DMX 值在 `FlowRange` 区间 → 颜色自动循环
- **白光**: DMX 值在 `WhiteRange` 区间 → 白色

`ColorAtlas` 为可选的颜色图集贴图，传入后会同步更新光束材质中的颜色贴图参数。

---

#### 色温控制

```cpp
// 单通道色温
void SetLightingColorTemperature(FSuperDMXAttribute DmxAttribute,
    float WarmTemperature = 1700.0f, float CoolTemperature = 12000.0f);

// 冷暖双通道混色
void SetLightingCoolWarmMix(FSuperDMXAttribute DmxCool, FSuperDMXAttribute DmxWarm,
    float CoolTemperature = 6500.0f, float WarmTemperature = 3200.0f);
```

**单通道色温**: DMX `[0, 1]` → `Lerp(WarmTemperature, CoolTemperature)` → 开尔文色温 → 颜色

**冷暖混色**: 两个独立通道分别控制冷光和暖光强度，混合输出。

---

#### 动态多通道混色

```cpp
// 任意通道组合混色
void SetLightingColorMix(const TArray<FSuperColorChannel>& ColorChannels);
void SetLightingColorMixMatrix(const TArray<FSuperColorChannel>& ColorChannels);

// 多通道混色 + CTO
void SetLightingColorMixWithCTO(const TArray<FSuperColorChannel>& ColorChannels,
    FSuperDMXAttribute DmxCTO, float CoolTemperature = 6500.f, float WarmTemperature = 3200.f);
void SetLightingColorMixMatrixWithCTO(...);
```

支持 **RGBW/RGBA/RGBL/RGBLAM** 等任意颜色通道组合。

**混色算法**: `FinalColor = Σ(DMX_Value[i] × ChannelColor[i])`，最终 Clamp 到 `[0, 1]`。

**参数**: `ColorChannels` — `FSuperColorChannel` 数组，每个元素包含 DMX 属性和对应颜色。

---

### 2.9 光学控制

#### SetLightingZoom / SetLightingZoomMatrix

```cpp
void SetLightingZoom(FSuperDMXAttribute DmxAttribute);
void SetLightingZoomMatrix(FSuperDMXAttribute DmxAttribute);
```

控制光束角度/光斑大小。同时触发光束遮挡距离更新。

---

#### SetLightingFrost

```cpp
void SetLightingFrost(FSuperDMXAttribute DmxAttribute);
```

控制雾化效果。`[0, 1]`，0 = 无雾化，1 = 完全雾化。

---

#### SetLightingIris

```cpp
void SetLightingIris(FSuperDMXAttribute DmxAttribute);
```

控制光圈大小。`[0, 1]`，0 = 全关（最小光斑），1 = 全开。

---

#### UpdateBeamOcclusion

```cpp
void UpdateBeamOcclusion();
```

独立更新光束遮挡距离（不依赖 Zoom 通道）。内部以 **30fps** 节流调用，避免每帧射线检测的性能开销。

---

### 2.10 图案轮控制

#### SetBeamGobo1 / SetBeamGobo2 / SetBeamGobo3

```cpp
// 单图案轮
void SetBeamGobo1(FSuperDMXAttribute DmxGobo1, FSuperDMXAttribute DmxGobo1Rot,
    UTexture2D* Gobo1Atlas = nullptr);

// 双图案轮
void SetBeamGobo2(FSuperDMXAttribute DmxGobo1, FSuperDMXAttribute DmxGobo1Rot,
    FSuperDMXAttribute DmxGobo2, FSuperDMXAttribute DmxGobo2Rot,
    UTexture2D* Gobo1Atlas = nullptr, UTexture2D* Gobo2Atlas = nullptr);

// 三图案轮
void SetBeamGobo3(..., FSuperDMXAttribute DmxGobo3, FSuperDMXAttribute DmxGobo3Rot,
    ..., UTexture2D* Gobo3Atlas = nullptr);
```

**参数**:

| 参数 | 说明 |
|------|------|
| `DmxGoboN` | 图案轮 N 的选择通道 |
| `DmxGoboNRot` | 图案轮 N 的旋转通道 |
| `GoboNAtlas` | 图案图集纹理（可选） |

**旋转模式**:
- DMX `[0, 127]` → 静态角度
- DMX `[128, 255]` → 无极旋转（速度映射）

---

#### SetBeamFocus

```cpp
void SetBeamFocus(FSuperDMXAttribute DmxFocus);
```

调焦控制。影响光束材质的 Focus 参数。

---

### 2.11 棱镜控制

#### SetBeamPrism

```cpp
void SetBeamPrism(FSuperDMXAttribute DmxPrism1, FSuperDMXAttribute DmxPrism2,
    FSuperDMXAttribute DmxPrismRot);
```

双棱镜控制（含旋转）。

**优先级**: 棱镜启用时**覆盖**图案轮效果（Prism > Gobo）。

---

### 2.12 切割控制

#### SetBeamCutting

```cpp
void SetBeamCutting(
    FSuperDMXAttribute DmxA1, FSuperDMXAttribute DmxB1,
    FSuperDMXAttribute DmxA2, FSuperDMXAttribute DmxB2,
    FSuperDMXAttribute DmxA3, FSuperDMXAttribute DmxB3,
    FSuperDMXAttribute DmxA4, FSuperDMXAttribute DmxB4);
```

四叶片切割控制（8 通道）。

**叶片映射**:
- `A` 侧: DMX `[0, 1]` → 叶片位置 `[0, 0.4]`
- `B` 侧: DMX `[0, 1]` → 叶片位置 `[1, 0.6]`

---

#### SetBeamCuttingRotate

```cpp
void SetBeamCuttingRotate(FSuperDMXAttribute DmxShaperRot, FSuperDMXAttribute DmxGoboRot);
```

切割系统整体旋转。

**旋转叠加**: `ShaperRot[-45°, 45°] + GoboRot[0°, 360°]`

---

### 2.13 效果控制

#### 初始化

```cpp
void SetEffectDefaultValue(USuperEffectComponent* NewSuperEffect);
void SetEffectMatrixDefaultValue(TArray<USuperEffectComponent*> NewSuperEffectArray);
```

初始化效果光源组件。

---

#### 效果亮度/频闪/颜色

```cpp
void SetEffectIntensity(FSuperDMXAttribute DmxAttribute);
void SetEffectStrobe(FSuperDMXAttribute DmxAttribute);
void SetEffectColor(FSuperDMXAttribute DMXAttR, FSuperDMXAttribute DMXAttG,
    FSuperDMXAttribute DMXAttB);
void SetEffectColorMatrix(FSuperDMXAttribute DMXAttR, FSuperDMXAttribute DMXAttG,
    FSuperDMXAttribute DMXAttB);
```

效果层的独立亮度、频闪和颜色控制。

---

#### 效果选择（LUT）

```cpp
// 效果选择（单通道，效果+速度+宽度由 LUT 查表决定）
void SetEffect(FSuperDMXAttribute DmxAttribute);
void SetEffectMatrix(FSuperDMXAttribute DmxAttribute);

// 效果+速度分离控制
void SetEffectWithSpeed(FSuperDMXAttribute DmxEffect, FSuperDMXAttribute DmxSpeed);
void SetEffectMatrixWithSpeed(FSuperDMXAttribute DmxEffect, FSuperDMXAttribute DmxSpeed);
```

**LUT 系统**: DMX `[0, 255]` → 查表 `GetEffectLUT()` → 返回 `{Effect, Speed, Width}` 三元组。内置脏值检测，值未变化时不重复设置。

---

### 2.14 矩阵组件控制

矩阵组件（`USuperMatrixComponent`）用于 LED 矩阵灯的像素级控制。

#### 初始化与颜色

```cpp
// 初始化（RGB + Dimmer）
void SetMatrixComponent(FSuperDMXAttribute DMXAttR, FSuperDMXAttribute DMXAttG,
    FSuperDMXAttribute DMXAttB, FSuperDMXAttribute DmxAttribute,
    USuperMatrixComponent* NewSuperMatrix);

// 白光控制
void SetMatrixWhiteSingle(FSuperDMXAttribute DmxAttribute, int32 Index,
    USuperMatrixComponent* NewSuperMatrix);
void SetMatrixWhiteMultiple(FSuperDMXAttribute DmxAttribute,
    USuperMatrixComponent* NewSuperMatrix);

// 频闪
void SetMatrixStrobe(FSuperDMXAttribute DmxAttribute, USuperMatrixComponent* NewSuperMatrix);

// 亮度
void SetMatrixIntensity(FSuperDMXAttribute DmxAttribute, USuperMatrixComponent* NewSuperMatrix);

// RGB 颜色
void SetMatrixColorSingle(FSuperDMXAttribute DMXAttR, FSuperDMXAttribute DMXAttG,
    FSuperDMXAttribute DMXAttB, int32 Index, USuperMatrixComponent* NewSuperMatrix);
void SetMatrixColorMultiple(FSuperDMXAttribute DMXAttR, FSuperDMXAttribute DMXAttG,
    FSuperDMXAttribute DMXAttB, USuperMatrixComponent* NewSuperMatrix);
```

**Single vs Multiple**:
- `Single`: 使用指定的 `Index` 控制单个像素
- `Multiple`: 使用矩阵读取，自动遍历所有像素

---

#### 矩阵扩展颜色控制

```cpp
// 冷暖混色（单点/多点）
void SetMatrixCoolWarmMixSingle(..., int32 Index, USuperMatrixComponent*);
void SetMatrixCoolWarmMixMultiple(..., USuperMatrixComponent*);

// 多通道混色
void SetMatrixColorMix(const TArray<FSuperColorChannel>& ColorChannels, USuperMatrixComponent*);

// RGB + CTO（单点/多点）
void SetMatrixColorRGBWithCTOSingle(..., int32 Index, USuperMatrixComponent*);
void SetMatrixColorRGBWithCTOMultiple(..., USuperMatrixComponent*);

// RGBW（单点/多点）
void SetMatrixColorRGBWSingle(..., int32 Index, USuperMatrixComponent*);
void SetMatrixColorRGBWMultiple(..., USuperMatrixComponent*);

// RGBW + CTO（单点/多点）
void SetMatrixColorRGBWWithCTOSingle(...);
void SetMatrixColorRGBWWithCTOMultiple(...);

// RGB + CoolWarm（单点/多点）
void SetMatrixColorRGBWithCoolWarmSingle(...);
void SetMatrixColorRGBWithCoolWarmMultiple(...);

// RGBW + CoolWarm（单点/多点）
void SetMatrixColorRGBWWithCoolWarmSingle(...);
void SetMatrixColorRGBWWithCoolWarmMultiple(...);
```

---

### 2.15 Spot 辅光控制

Spot 辅光（`USuperSpotComponent`）是与主光源分离的 SpotLight 组件，用于补充照明效果。

#### 初始化

```cpp
void SetSpotDefaultValue(USuperSpotComponent* NewSuperSpot);
void SetSpotMatrixDefaultValue(TArray<USuperSpotComponent*> NewSuperSpotArray);
```

#### 亮度/频闪

```cpp
void SetSpotIntensity(FSuperDMXAttribute DmxAttribute);
void SetSpotIntensityMatrix(FSuperDMXAttribute DmxAttribute);
void SetSpotStrobe(FSuperDMXAttribute DmxAttribute);
void SetSpotStrobeMatrix(FSuperDMXAttribute DmxAttribute);
```

#### 颜色

```cpp
// RGB
void SetSpotColorRGB(FSuperDMXAttribute DMXAttR, FSuperDMXAttribute DMXAttG,
    FSuperDMXAttribute DMXAttB);
void SetSpotColorRGBMatrix(...);

// RGBW
void SetSpotColorRGBW(..., FSuperDMXAttribute DMXAttW);
void SetSpotColorRGBWMatrix(...);

// 冷暖混色
void SetSpotCoolWarmMix(FSuperDMXAttribute DmxCool, FSuperDMXAttribute DmxWarm,
    float CoolTemperature = 6500.f, float WarmTemperature = 3200.f);
void SetSpotCoolWarmMixMatrix(...);

// RGBW + 冷暖混色
void SetSpotColorRGBWWithCoolWarmMix(...);
void SetSpotColorRGBWWithCoolWarmMixMatrix(...);

// 色温矩阵
void SetSpotColorTemperatureMatrix(FSuperDMXAttribute DmxAttribute,
    float WarmTemperature = 1700.0f, float CoolTemperature = 12000.0f);
```

#### 光学

```cpp
void SetSpotZoom(FSuperDMXAttribute DmxAttribute);
void SetSpotZoomMatrix(FSuperDMXAttribute DmxAttribute);
void SetSpotFrost(FSuperDMXAttribute DmxAttribute);
void SetSpotFrostMatrix(FSuperDMXAttribute DmxAttribute);
```

---

## 3. FSuperColorChannel — 动态混色通道

**头文件**: `LightActor/SuperStageLight.h`

```cpp
USTRUCT(BlueprintType)
struct FSuperColorChannel
```

用于 `SetLightingColorMix` 系列函数的颜色通道定义。

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `DmxAttribute` | `FSuperDMXAttribute` | — | DMX 属性（通道值 0-1） |
| `ChannelColor` | `FLinearColor` | `White` | 该通道对应的颜色 |

**使用示例**（蓝图伪代码）:

```
ColorChannels = [
    { DmxAttribute = Red通道,    ChannelColor = (1,0,0) },  // 红
    { DmxAttribute = Green通道,  ChannelColor = (0,1,0) },  // 绿
    { DmxAttribute = Blue通道,   ChannelColor = (0,0,1) },  // 蓝
    { DmxAttribute = White通道,  ChannelColor = (1,1,1) },  // 白
    { DmxAttribute = Amber通道,  ChannelColor = (1,0.75,0) }, // 琥珀
    { DmxAttribute = Lime通道,   ChannelColor = (0.5,1,0) }, // 酸橙
]
SetLightingColorMix(ColorChannels)
```

---

## 4. FLightDefaultValue — 灯光默认参数

**头文件**: `LightActor/SuperLightTypes.h`

```cpp
USTRUCT(BlueprintType)
struct FLightDefaultValue
```

### 顶层属性

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `MaxLightIntensity` | `float` | `100.0` | 最大亮度（%），≥1 |
| `LensIntensity` | `float` | `1.0` | 镜头强度乘数 |
| `MaxLightDistance` | `float` | `2345.0` | 最大光照距离（cm），≥100 |
| `LightSpotDefaultValue` | `FLightSpotDefaultValue` | — | 光斑参数 |
| `BeamDefaultValue` | `FBeamDefaultValue` | — | 光束参数 |

### FLightSpotDefaultValue

| 属性 | 类型 | 默认值 | 范围 | 说明 |
|------|------|--------|------|------|
| `LightSpotIntensity` | `float` | `100.0` | ≥0 | 光斑强度（%） |
| `VolumetricScattering` | `float` | `0.0` | 0-100 | 体积雾强度（%） |
| `bLightShadow` | `bool` | `false` | — | 阴影开关 |
| `bLightingChannel0/1/2` | `bool` | `true/false/false` | — | 光照通道 |
| `bAffectTransmission` | `bool` | `true` | — | 透射影响 |
| `SpecularScale` | `float` | `1.0` | 0-1 | 高光缩放 |
| `SourceRadius` | `float` | `0.0` | ≥0 | 软阴影半径 |

### FBeamDefaultValue

| 属性 | 类型 | 默认值 | 范围 | 说明 |
|------|------|--------|------|------|
| `BeamIntensity` | `float` | `1.0` | ≥0 | 光束强度 |
| `AtmosphericDensity` | `float` | `0.03` | 0-1 | 大气衰减密度 |
| `BeamFogIntensity` | `float` | `20.0` | 0-100 | 光束雾强度（%） |
| `AtmosBeamFogSpeed` | `float` | `10.0` | 0-100 | 光束雾流速（%） |
| `LensRadius` | `float` | `10.0` | 1-100 | 镜头尺寸（%） |
| `BeamQuality` | `float` | `75.0` | 0-100 | 光束渲染质量（%） |
| `bBeamBlock` | `bool` | `false` | — | 光束遮挡开关 |

---

## 5. Pan/Tilt 运动系统详解

### 运动流程

```
DMX 数据 → GetSuperDmxAttributeValue → 归一化 [0,1]
    ↓
Lerp(Range.X, Range.Y, DMX值) → 目标角度（度）
    ↓
FInterpTo(Current, Target, DeltaTime, PTSpeed) → 平滑角度
    ↓
SetRelativeRotation → 更新组件旋转
```

### 插值行为

- **PTSpeed = 0**: 极慢响应（几乎不动）
- **PTSpeed = 5**: 默认速度，约 0.5-1 秒达到目标
- **PTSpeed = 10+**: 快速响应，接近即时

### 无极旋转流程

```
DMX 数据 → GetSuperDmxAttributeValueNoConversion → 原始值 0-255
    ↓
FindSubAttribute(RawValue) → RotationMode 判断
    ↓
├─ Off      → 回退到 YPan/YTilt 位置控制
├─ Stop     → 保持当前角度
├─ Position → 从起始角度偏移旋转（用 FInterpTo）
└─ Infinite → GetMappedPhysical → 速度 × InfiniteRotationalSpeed
                ↓
            AddRelativeRotation → 每帧累加旋转
```

### 矩阵 vs 单灯模式

| 特性 | 单灯模式 | 矩阵模式 |
|------|---------|----------|
| 函数 | `YPan`/`YTilt` | `YMatrixPan`/`YMatrixTilt` |
| 目标组件 | `SceneRocker`/`SceneHard` | 传入的 `MatrixHard[]`/`MatrixRocker[]` |
| 插值状态 | `CurrentPan`/`CurrentTilt` | `CurrentMatrixPan[]`/`CurrentMatrixTilt[]` |
| DMX 读取 | 单值 | `GET_SUPER_DMX_MATRIX_VALUE` 批量读取 |

---

## 6. 颜色系统详解

### 颜色控制选择指南

| 灯具类型 | 推荐函数 |
|----------|---------|
| 纯 RGB LED | `SetLightingColorRGB` |
| RGBW LED | `SetLightingColorRGBW` |
| RGBW + CTO | `SetLightingColorRGBWWithCTO` |
| RGBALWM 等多色 | `SetLightingColorMix` |
| 带颜色轮的传统灯 | `SetLightingColorWheel` |
| 颜色轮 + CMY | `SetLightingColorWheelAndCMY` |
| 双颜色轮 + CMY | `SetLightingColorWheel2AndCMY` |
| 双颜色轮 + CMY + CTO | `SetLightingColorWheel2AndCMYAndCTO` |
| 三颜色轮 + CMY | `SetLightingColorWheel3AndCMY` |
| 纯色温灯 | `SetLightingColorTemperature` |
| 冷暖双色温 | `SetLightingCoolWarmMix` |
| HSV 控制 | `SetLightingColorHSV` |
| HS（明度跟随 Dimmer） | `SetLightingColorHS` |

### CMY 混色原理

CMY（青/品红/黄）是**减色混合**系统，物理上通过滤色片透光实现：

```
FinalColor = BaseColor × (1 - Cyan) × (1 - Magenta) × (1 - Yellow)
```

- `C = 1, M = 0, Y = 0` → 红色被吸收 → 青色
- `C = 0, M = 1, Y = 0` → 绿色被吸收 → 品红
- `C = 1, M = 1, Y = 0` → 红绿被吸收 → 蓝色

### CTO（色温橙）

CTO 通道调节整体色温：

```
CTO = 0 → 无滤片（CoolTemperature，默认 6500K 或更高）
CTO = 1 → 全橙滤片（WarmTemperature，默认 2700K-3200K）
```

---

## 7. 使用示例

### 7.1 蓝图典型用法 — 完整电脑灯控制

在 `SuperDMXTick` 事件中连接以下节点：

```
Event SuperDMXTick
 │
 ├─ SetLightingDefaultValue(SuperLighting)          // 首帧初始化
 │
 ├─ YPan(DmxPan)                                     // Pan 控制
 ├─ YTilt(DmxTilt)                                   // Tilt 控制
 ├─ YPTSpeed(DmxPTSpeed)                             // PT 速度
 │
 ├─ SetLightingIntensity(DmxDimmer)                  // 亮度
 ├─ SetLightingStrobe(DmxStrobe)                     // 频闪
 │
 ├─ SetLightingColorRGB(DmxR, DmxG, DmxB)           // RGB 颜色
 ├─ SetLightingZoom(DmxZoom)                         // 变焦
 ├─ SetLightingFrost(DmxFrost)                       // 雾化
 │
 ├─ SetBeamGobo1(DmxGobo1, DmxGobo1Rot)             // 图案轮
 ├─ SetBeamPrism(DmxPrism1, DmxPrism2, DmxPrismRot) // 棱镜
 │
 └─ SetBeamCutting(A1,B1, A2,B2, A3,B3, A4,B4)      // 切割
```

### 7.2 蓝图 — 矩阵 LED 灯

```
Event SuperDMXTick
 │
 ├─ SetLightingMatrixDefaultValue(LightingArray)
 ├─ SetLightingIntensityMatrix(DmxDimmer)
 ├─ SetLightingColorRGBMatrix(DmxR, DmxG, DmxB)
 ├─ SetLightingZoomMatrix(DmxZoom)
 └─ YMatrixTilt(DmxTilt, MatrixHardArray)
```

### 7.3 蓝图 — RGBW + CTO 灯具

```
Event SuperDMXTick
 │
 ├─ SetLightingDefaultValue(SuperLighting)
 ├─ SetLightingIntensity(DmxDimmer)
 └─ SetLightingColorRGBWWithCTO(DmxR, DmxG, DmxB, DmxW, DmxCTO,
                                 CoolTemp=6500, WarmTemp=3200)
```

### 7.4 蓝图 — 多色 LED（RGBALWM）

```
// 构建 ColorChannels 数组
ColorChannels = MakeArray(
    FSuperColorChannel(DmxR,     Red),
    FSuperColorChannel(DmxG,     Green),
    FSuperColorChannel(DmxB,     Blue),
    FSuperColorChannel(DmxAmber, (1, 0.75, 0)),
    FSuperColorChannel(DmxLime,  (0.5, 1, 0)),
    FSuperColorChannel(DmxWhite, White),
    FSuperColorChannel(DmxMint,  (0.6, 1, 0.8))
)

Event SuperDMXTick
 │
 ├─ SetLightingDefaultValue(SuperLighting)
 ├─ SetLightingIntensity(DmxDimmer)
 └─ SetLightingColorMix(ColorChannels)
```

### 7.5 C++ 插件 — 创建自定义灯具

```cpp
#include "LightActor/SuperStageLight.h"

// 在你的自定义 Actor 或 Controller 中：
void AMyLightController::UpdateLight(ASuperStageLight* Light, float DeltaTime)
{
    if (!Light) return;

    // 可以直接设置控制参数（不通过 DMX）
    Light->Dimmer = 0.8f;
    Light->Color = FLinearColor(1.0f, 0.5f, 0.2f);
    Light->Zoom = 0.3f;

    // 或通过 DMX 属性读取
    FSuperDMXAttribute DimmerAttr;
    DimmerAttr.InstanceIndex = 0;
    DimmerAttr.AttribName = FName("Dimmer");
    DimmerAttr.DMXChannelType = EDMXChannelType::Conarse;
    Light->SetLightingIntensity(DimmerAttr);
}
```

---

## 8. API 速查表

### ASuperLightBase — 位置控制

| 函数 | 类别 | 说明 |
|------|------|------|
| `YPan(DmxPan)` | Position | Pan 水平旋转 |
| `YTilt(DmxTilt)` | Position | Tilt 垂直旋转 |
| `YPTSpeed(DmxAttr)` | Position | PT 速度 |
| `YMatrixPan(DmxPan, Array)` | Matrix | 矩阵 Pan |
| `YMatrixTilt(DmxTilt, Array)` | Matrix | 矩阵 Tilt |
| `YPanRot(DmxInf, DmxPan)` | Infinite | Pan 无极旋转 |
| `YTiltRot(DmxInf, DmxTilt)` | Infinite | Tilt 无极旋转 |
| `YMatrixPanRot(DmxInf, DmxPan, Array)` | Infinite | 矩阵 Pan 无极 |
| `YMatrixTiltRot(DmxInf, DmxTilt, Array)` | Infinite | 矩阵 Tilt 无极 |
| `YLampAngle()` | Angle | 手动角度 |
| `SetLiftZ(Lift, DmxZ)` | Lift | 升降控制 |

### ASuperStageLight — 初始化

| 函数 | 说明 |
|------|------|
| `SetLightingDefaultValue(Comp)` | 主光源初始化 |
| `SetLightingMatrixDefaultValue(Array)` | 矩阵光源初始化 |
| `SetLightingDefaultTexture(Tex)` | 默认 Gobo |
| `SetEffectDefaultValue(Comp)` | 效果初始化 |
| `SetEffectMatrixDefaultValue(Array)` | 效果矩阵初始化 |
| `SetSpotDefaultValue(Comp)` | Spot 辅光初始化 |
| `SetSpotMatrixDefaultValue(Array)` | Spot 矩阵初始化 |

### ASuperStageLight — 亮度/频闪

| 函数 | 单灯 | 矩阵 |
|------|------|------|
| 亮度 | `SetLightingIntensity` | `SetLightingIntensityMatrix` |
| 频闪 | `SetLightingStrobe` | `SetLightingStrobeMatrix` |
| 效果亮度 | `SetEffectIntensity` | — |
| 效果频闪 | `SetEffectStrobe` | — |
| Spot 亮度 | `SetSpotIntensity` | `SetSpotIntensityMatrix` |
| Spot 频闪 | `SetSpotStrobe` | `SetSpotStrobeMatrix` |

### ASuperStageLight — 颜色

| 色彩模式 | 单灯函数 | 矩阵函数 |
|---------|---------|---------|
| RGB | `SetLightingColorRGB` | `SetLightingColorRGBMatrix` |
| RGBW | `SetLightingColorRGBW` | `SetLightingColorRGBWMatrix` |
| RGBW+CTO | `SetLightingColorRGBWWithCTO` | `SetLightingColorRGBWMatrixWithCTO` |
| HSV | `SetLightingColorHSV` | `SetLightingColorHSVMatrix` |
| HS | `SetLightingColorHS` | `SetLightingColorHSMatrix` |
| HS+CTO | `SetLightingColorHSAndCTO` | — |
| 颜色轮 | `SetLightingColorWheel` | — |
| 颜色轮+CMY | `SetLightingColorWheelAndCMY` | — |
| 色温 | `SetLightingColorTemperature` | — |
| 冷暖混色 | `SetLightingCoolWarmMix` | — |
| 多通道混色 | `SetLightingColorMix` | `SetLightingColorMixMatrix` |
| 多通道+CTO | `SetLightingColorMixWithCTO` | `SetLightingColorMixMatrixWithCTO` |

### ASuperStageLight — 光学/图案/棱镜/切割

| 函数 | 说明 |
|------|------|
| `SetLightingZoom` / `Matrix` | 变焦 |
| `SetLightingFrost` | 雾化 |
| `SetLightingIris` | 光圈 |
| `UpdateBeamOcclusion` | 光束遮挡（30fps 节流） |
| `SetBeamGobo1/2/3` | 图案轮（1-3 个） |
| `SetBeamFocus` | 调焦 |
| `SetBeamPrism` | 棱镜 |
| `SetBeamCutting` | 四叶片切割 |
| `SetBeamCuttingRotate` | 切割旋转 |

### ASuperStageLight — 效果/矩阵

| 函数 | 说明 |
|------|------|
| `SetEffect` / `SetEffectMatrix` | 效果选择（LUT） |
| `SetEffectWithSpeed` / `Matrix` | 效果+独立速度 |
| `SetEffectColor` / `Matrix` | 效果颜色 |
| `SetMatrixComponent` | 矩阵初始化+颜色+亮度 |
| `SetMatrixWhiteSingle/Multiple` | 矩阵白光 |
| `SetMatrixColorSingle/Multiple` | 矩阵 RGB |
| `SetMatrixStrobe` | 矩阵频闪 |
| `SetMatrixIntensity` | 矩阵亮度 |
| `SetMatrixColorMix` | 矩阵多通道混色 |

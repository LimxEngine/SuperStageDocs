# 组件类 API 文档

> **文档版本**: 1.0  
> **适用版本**: SuperStage Plugin 2025/2026  
> **目标读者**: 蓝图开发者 / C++ 插件集成开发者（无源码修改权限）  
> **最后更新**: 2026-03

---

## 目录

1. [组件继承体系](#1-组件继承体系)
2. [USuperLightingComponent — 灯光组件基类](#2-usuperlightingcomponent--灯光组件基类)
3. [USuperSpotComponent — 聚光灯组件](#3-usuperspotcomponent--聚光灯组件)
4. [USuperBeamComponent — 体积光束组件](#4-usuperbeamcomponent--体积光束组件)
5. [USuperCuttingComponent — 切割光束组件](#5-usupercuttingcomponent--切割光束组件)
6. [USuperRectComponent — 矩形面光组件](#6-usuperrectcomponent--矩形面光组件)
7. [USuperEffectComponent — 效果平面组件](#7-usupereffectcomponent--效果平面组件)
8. [USuperMatrixComponent — 矩阵灯组件](#8-usupermatrixcomponent--矩阵灯组件)
9. [USuperLiftComponent — 升降组件](#9-usuperliftcomponent--升降组件)
10. [USuperLaserProComponent — 激光 Pro 组件](#10-usuperlaserprocomponent--激光-pro-组件)
11. [辅助光源类](#11-辅助光源类)
12. [API 速查表](#12-api-速查表)

---

## 1. 组件继承体系

```
USceneComponent
 ├─ USuperLightingComponent (灯光基类)
 │   ├─ USuperSpotComponent (聚光灯 = SpotLight + 镜头 + 光斑材质)
 │   │   └─ USuperBeamComponent (体积光束 = SpotLight + 光束网格 + 光束材质)
 │   │       └─ USuperCuttingComponent (切割光束 = 体积光束 + 切割材质)
 │   └─ USuperRectComponent (矩形面光 = RectLight + 镜头)
 ├─ USuperEffectComponent (效果平面 = 静态网格 + 效果材质)
 ├─ USuperMatrixComponent (矩阵灯 = 静态网格 + 分段材质 + 分段光源)
 ├─ USuperLiftComponent (升降控制)
 └─ USuperLaserProComponent (激光线 = 程序化网格)
```

**设计原则**:
- 每个组件封装一种物理光源或视觉效果
- Actor（如 `ASuperStageLight`）组合多个组件实现完整灯具
- 组件通过虚函数接口统一控制，Actor 层无需关心具体实现

---

## 2. USuperLightingComponent — 灯光组件基类

**头文件**: `LightComponent/SuperLightingComponent.h`  
**基类**: `USceneComponent`  
**导出宏**: `SUPERSTAGE_API`  
**蓝图标识**: `BlueprintSpawnableComponent`，`DisplayName = "SuperLightComponent"`

所有灯光组件的基类，提供镜头网格、光斑材质、亮度/颜色/频闪等通用接口。

### 子组件

| 组件 | 类型 | 说明 |
|------|------|------|
| `YStaticMeshLens` | `UStaticMeshComponent*` | 镜头网格（运行时创建） |
| `YArrowComponent` | `UArrowComponent*` | 方向箭头（编辑器辅助） |

### 默认参数

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `StaticMeshLens` | `UStaticMesh*` | — | 镜头模型 |
| `LensTransform` | `FTransform` | `Identity` | 镜头变换 |
| `Angle` | `float` | `1.0` | Spot 角度 |
| `DimmerCurveExponent` | `float` | `2.0` | 亮度响应曲线指数（1-3） |
| `ComponentDimmer` | `float` | `1.0` | 组件级亮度分控（0-1） |

#### DimmerCurveExponent 说明

| 值 | 曲线 | 推荐场景 |
|----|------|---------|
| `1.0` | 线性 | 不推荐（低亮度区域变化过大） |
| `2.0` | 平方 | **推荐**（模拟真实调光台） |
| `2.2` | Gamma | 显示器标准校正 |
| `3.0` | 立方 | 强非线性响应 |

**公式**: `Output = pow(Input, DimmerCurveExponent)`

#### ComponentDimmer 说明

组件级亮度分控，与 Actor 级 Dimmer 乘法叠加：

```
FinalIntensity = ActorDimmer × ComponentDimmer × StrobeMultiplier
```

### 材质

| 属性 | 类型 | 说明 |
|------|------|------|
| `LensMaterial` | `UMaterialInstance*` | 镜头源材质 |
| `LightSpotMaterial` | `UMaterialInstance*` | 光斑源材质 |
| `DynamicMaterialLens` | `UMaterialInstanceDynamic*` | 镜头动态材质 |
| `DynamicMaterialLightSpot` | `UMaterialInstanceDynamic*` | 光斑动态材质 |

### 射线检测参数（内部）

| 属性 | 默认值 | 说明 |
|------|--------|------|
| `MinReturnDistance` | `100.0` cm | 射线返回最小距离 |
| `bSweepFallback` | `true` | Miss 时 Sweep 兜底 |
| `SweepRadius` | `2.0` cm | Sweep 半径 |
| `TraceChannel` | `ECC_Visibility` | 碰撞通道 |

### 函数

#### 初始化

```cpp
virtual void SetLightingMaterial();
```
创建动态材质实例并绑定到子组件。

```cpp
virtual void SetLightingDefaultValue(
    float NewMaxLightDistance,
    float NewLightMaxIntensity,
    float VolumetricScattering,
    bool LightShadow,
    float NewLightSpotIntensity = 100.0f,
    float LensIntensity = 1.0f);
```
设置灯光默认参数。

---

#### 亮度与频闪

```cpp
virtual void SetLightingIntensity(float NewLightIntensity = 1.0f);
virtual void SetLightingStrobe(float NewStrobe = 0.0f);
virtual void SetLightingStrobeMode(float NewStrobeMode = 1.0f);
virtual void SetRandomSeed(float NewSeed);
```

**频闪模式**（`StrobeMode` 值）:

| 值 | 模式 | 说明 |
|----|------|------|
| 0 | Close | 常灭 |
| 1 | Open | 常亮（默认） |
| 2 | Strobe | 标准频闪 |
| 3 | Pulse | 脉冲 |
| 4 | RampUp | 渐亮 |
| 5 | RampDown | 渐灭 |
| 6 | Sine | 正弦 |
| 7 | Random | 随机 |

**频闪实现**: `TickComponent` 中调用 `CalculateStrobeMultiplier()` 计算当前帧的频闪倍率，然后通过 `UpdateLightIntensity()` 应用到光源。

```cpp
UFUNCTION(BlueprintCallable, meta = (DisplayName = "SetComponentDimmer"))
void SetComponentDimmer(float NewDimmer);
```
设置组件级亮度分控。

---

#### 颜色与光学

```cpp
virtual void SetLightingColor(FLinearColor NewColor = FLinearColor(1,1,1));
virtual void SetLightingZoom(float NewZoom = 0.0f);
virtual void SetLightingFrost(float NewFrost = 0.0f);
virtual void SetLightingIris(float NewIris = 1.0f);
```

---

#### 纹理

```cpp
virtual void SetLightingTexture(UTexture2D* NewBeamTexture);

// 颜色轮贴图（基类空实现，子类重写）
virtual void SetColorTexture(UTexture2D*, int32 NumColors, float ColorIndex, float ColorSpeed);
virtual void SetColorTexture2(...);
virtual void SetColorTexture3(...);
```

---

#### 旋转与可见性

```cpp
UFUNCTION(BlueprintCallable, meta = (DisplayName = "SetLightingRotate"))
virtual void SetLightingRotate(float NewRotate = 0, float NewInfiniteRotation = 0);

UFUNCTION(BlueprintCallable, meta = (DisplayName = "SetLightingVisibility"))
virtual void SetLightingVisibility(bool bNewVisibility = false);

virtual void SetLightingLensVisibility(bool bLensVisibility = true);
```

---

#### 高级光照设置

```cpp
UFUNCTION(BlueprintCallable, meta = (DisplayName = "SetLightingChannels"))
virtual void SetLightingChannels(bool bChannel0 = true, bool bChannel1 = false, bool bChannel2 = false);

UFUNCTION(BlueprintCallable, meta = (DisplayName = "SetLightingTransmission"))
virtual void SetLightingTransmission(bool bAffectTransmission = true);

virtual void SetSpecularScale(float NewSpecularScale = 1.0f);
virtual void SetSourceRadius(float NewSourceRadius = 0.0f);
```

---

#### 射线检测

```cpp
float GetRayDetectionDistance(float NewMaxLightDistance) const;
virtual void UpdateBeamBlockDistance(float NewMaxLightDistance);
```

`GetRayDetectionDistance`: 从组件位置沿前向射线检测碰撞，返回命中距离。支持 Sweep 兜底和最小返回距离限制。

`UpdateBeamBlockDistance`: 更新光束遮挡距离（供 `ASuperStageLight::UpdateBeamOcclusion` 调用）。

---

## 3. USuperSpotComponent — 聚光灯组件

**头文件**: `LightComponent/SuperSpotComponent.h`  
**基类**: `USuperLightingComponent`  
**导出宏**: `SUPERSTAGE_API`  
**蓝图标识**: `DisplayName = "SuperSpotComponent"`

继承灯光基类，增加实际 SpotLight 光源。适用于需要真实光照投射的灯具（辅光、补光）。

### 子组件

| 组件 | 类型 | 说明 |
|------|------|------|
| `YSpotLight` | `USilentSpotLightComponent*` | 聚光灯光源（静默版本，绕过 GPU Scene 同步问题） |

### 默认参数

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `ZoomRange` | `FVector2D` | `(1, 10)` | Zoom 角度范围（度） |
| `bDisableLightFunction` | `bool` | `false` | 禁用光照函数（改用 Tick 直控强度） |

### 重写函数

所有 `USuperLightingComponent` 的虚函数均被重写，以同步更新 `YSpotLight` 的参数：

- `SetLightingMaterial()` — 创建 SpotLight 并设置材质
- `SetLightingDefaultValue(...)` — 配置 SpotLight 参数（衰减、阴影、体积雾等）
- `SetLightingIntensity(float)` — 更新 SpotLight 亮度
- `SetLightingColor(FLinearColor)` — 更新 SpotLight 颜色
- `SetLightingZoom(float)` — 更新 OuterConeAngle（通过 ZoomRange 映射）
- `SetLightingFrost(float)` — 调整 InnerConeAngle 模拟雾化
- `SetLightingIris(float)` — 调整光斑大小
- `SetLightingRotate(float, float)` — 旋转光照函数
- `SetLightingVisibility(bool)` — SpotLight 可见性
- `SetLightingChannels(...)` — 光照通道设置
- `SetLightingTransmission(bool)` — 透射开关
- `SetSpecularScale(float)` — 高光缩放
- `SetSourceRadius(float)` — 软阴影半径
- `SetColorTexture/2/3(...)` — 颜色轮贴图

### 内部缓存

| 属性 | 说明 |
|------|------|
| `LastSegIdx` | 上一帧 Zoom 分段索引（加速查表） |
| `LastConeDeg` | 上次下发的锥角（去抖） |
| `LastAttenuationRadius` | 上次下发的衰减半径（去抖） |
| `BaseOuterConeAngle` | 基础外锥角（未受 Frost/Iris 影响） |
| `CurrentZoomValue` | 当前 Zoom 值 |
| `CurrentIrisValue` | 当前 Iris 值 |
| `CurrentFrostValue` | 当前 Frost 值 |

### USilentSpotLightComponent

```cpp
UCLASS()
class SUPERSTAGE_API USilentSpotLightComponent : public USpotLightComponent
```

安全版 SpotLight，提供 `SetAttenuationRadiusSafe(float)` 函数：
- 绕过 `PushRadiusToRenderThread()` 避免 GPU Scene stale 断言
- 仅设置属性 + `MarkRenderStateDirty()`，让引擎在帧末统一更新

---

## 4. USuperBeamComponent — 体积光束组件

**头文件**: `LightComponent/SuperBeamComponent.h`  
**基类**: `USuperSpotComponent`  
**导出宏**: `SUPERSTAGE_API`  
**蓝图标识**: `DisplayName = "SuperBeamComponent"`

在 SpotLight 基础上增加体积光束网格，通过光束材质实现可见光柱效果。

### 继承链

```
USceneComponent → USuperLightingComponent → USuperSpotComponent → USuperBeamComponent
```

### 子组件

| 组件 | 类型 | 说明 |
|------|------|------|
| `YStaticMeshBeam` | `UStaticMeshComponent*` | 光束网格 |
| （继承）`YSpotLight` | `USilentSpotLightComponent*` | SpotLight 光源 |

### 材质

| 属性 | 类型 | 说明 |
|------|------|------|
| `BeamMaterial` | `UMaterialInstance*` | 光束源材质 |
| `DynamicMaterialBeam` | `UMaterialInstanceDynamic*` | 光束动态材质 |

### 函数

#### 光束初始化

```cpp
void SetBeamDefaultValue(
    float NewMaxLightDistance,
    float NewLightMaxIntensity,
    float AtmosphericDensity,
    float LensRadius,
    float NewBeamQuality,
    float NewFogInfluence,
    float NewFogSpeed,
    float NewBeamIntensity = 1.0f) const;
```

设置光束材质的默认参数。

**参数说明**:

| 参数 | 说明 |
|------|------|
| `AtmosphericDensity` | 大气衰减密度（0-1） |
| `LensRadius` | 镜头尺寸（%），影响光束宽度 |
| `NewBeamQuality` | 光束渲染质量（%） |
| `NewFogInfluence` | 光束雾强度（%） |
| `NewFogSpeed` | 光束雾流速（%） |
| `NewBeamIntensity` | 光束基础强度 |

---

#### 光束纹理（Gobo）

```cpp
void SetBeamTexture(
    UTexture2D* NewBeamTexture = nullptr,
    int32 NewNumGobos = 1,
    float GoboIndex = 0.f,
    float GoboSpeed = 0.f,
    float ShakeSpeed = 0.f) const;
```

设置光束纹理/图案轮参数。

| 参数 | 说明 |
|------|------|
| `NewBeamTexture` | 图案图集纹理 |
| `NewNumGobos` | 图案数量 |
| `GoboIndex` | 当前图案索引 |
| `GoboSpeed` | 图案旋转速度 |
| `ShakeSpeed` | 图案抖动速度 |

---

#### 棱镜

```cpp
void SetBeamPrism(
    int32 PrismFacets = 0,
    float PrismRadius = 0.3f,
    float PrismScale = 0.15f,
    float PrismRotation = 0.f,
    float PrismRotationSpeed = 0.f) const;
```

| 参数 | 说明 |
|------|------|
| `PrismFacets` | 棱镜面数（0=关闭，8/16/24 等） |
| `PrismRadius` | 棱镜半径 |
| `PrismScale` | 棱镜缩放 |
| `PrismRotation` | 棱镜旋转角度 |
| `PrismRotationSpeed` | 棱镜旋转速度 |

---

#### 调焦

```cpp
void SetBeamFocus(float Focus = 0.f) const;
```

`0` = 清晰，`1` = 模糊。

---

### 重写函数

所有 `USuperSpotComponent` 的虚函数均被重写，以同步更新光束材质参数。如 `SetLightingIntensity` 会同时更新 SpotLight 强度和光束材质的 `Brightness` 参数。

---

## 5. USuperCuttingComponent — 切割光束组件

**头文件**: `LightComponent/SuperCuttingComponent.h`  
**基类**: `USuperBeamComponent`  
**导出宏**: `SUPERSTAGE_API`  
**蓝图标识**: `DisplayName = "SuperCuttingComponent"`

在体积光束基础上增加四叶片切割控制。构造函数中加载切割专用材质。

### 继承链

```
USuperLightingComponent → USuperSpotComponent → USuperBeamComponent → USuperCuttingComponent
```

### 函数

#### SetCuttingValue

```cpp
void SetCuttingValue(
    float NewUpperLeftY,  float NewLowerLeftY,
    float NewTopRightY,   float NewBottomRightY,
    float NewTopLeftX,    float NewBottomLeftX,
    float NewTopRightX,   float NewBottomRightX) const;
```

设置四角八点切割参数。

**材质参数映射**:

| 参数 | 材质参数名 | 说明 |
|------|-----------|------|
| `NewUpperLeftY` | `UpperLeftY` | 左上角 Y 位置 |
| `NewLowerLeftY` | `LowerLeftY` | 左下角 Y 位置 |
| `NewTopRightY` | `TopRightY` | 右上角 Y 位置 |
| `NewBottomRightY` | `BottomRightY` | 右下角 Y 位置 |
| `NewTopLeftX` | `TopLeftX` | 左上角 X 位置 |
| `NewBottomLeftX` | `BottomLeftX` | 左下角 X 位置 |
| `NewTopRightX` | `TopRightX` | 右上角 X 位置 |
| `NewBottomRightX` | `BottomRightX` | 右下角 X 位置 |

---

#### SetLightingRotate (重写)

```cpp
virtual void SetLightingRotate(float NewRotate, float NewInfiniteRotation = 0.f) override;
```

重写父类旋转，同时设置 LightSpot 材质的旋转参数（切割与光斑同步旋转）。

---

## 6. USuperRectComponent — 矩形面光组件

**头文件**: `LightComponent/SuperRectComponent.h`  
**基类**: `USuperLightingComponent`  
**导出宏**: `SUPERSTAGE_API`  
**蓝图标识**: `DisplayName = "SuperRectComponent"`

矩形面光组件，桥接 `URectLightComponent` 与灯光基类接口。适用于 LED 面板灯、Wash 灯等面光源。

### 子组件

| 组件 | 类型 | 说明 |
|------|------|------|
| `YRectLight` | `USilentRectLightComponent*` | 矩形面光源 |

### 默认参数

| 属性 | 类型 | 默认值 | 范围 | 说明 |
|------|------|--------|------|------|
| `SourceWidth` | `float` | `20.0` | 0-200 cm | 光源宽度 |
| `SourceHeight` | `float` | `20.0` | 0-200 cm | 光源高度 |
| `BarnDoorAngle` | `float` | `20.0` | 0-80° | 挡光板角度 |
| `BarnDoorLength` | `float` | `20.0` | 0-1000 cm | 挡光板长度 |

### 重写函数

- `SetLightingMaterial()` — 创建 RectLight 并设置材质
- `SetLightingDefaultValue(...)` — 配置 RectLight 参数
- `SetLightingIntensity(float)` — 更新 RectLight 强度
- `SetLightingColor(FLinearColor)` — 更新 RectLight 颜色
- `SetLightingVisibility(bool)` — RectLight 可见性
- `SetLightingChannels(...)` — 光照通道
- `SetLightingTransmission(bool)` — 透射开关

### USilentRectLightComponent

```cpp
UCLASS()
class SUPERSTAGE_API USilentRectLightComponent : public URectLightComponent
```

空壳子类，用于隐藏编辑器图标。

---

## 7. USuperEffectComponent — 效果平面组件

**头文件**: `LightComponent/SuperEffectComponent.h`  
**基类**: `USceneComponent`  
**导出宏**: `SUPERSTAGE_API`  
**蓝图标识**: `DisplayName = "SuperEffectComponent"`

效果平面组件，使用静态网格 + 动态材质实现 LED 效果（流水、追逐、脉冲等）。支持不透明/半透明材质切换。

### 子组件

| 组件 | 类型 | 说明 |
|------|------|------|
| `YStaticMeshEffect` | `UStaticMeshComponent*` | 效果网格 |

### 默认参数

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `StaticMeshEffect` | `UStaticMesh*` | — | 效果网格模型 |
| `EffectMaterialNo` | `int` | `0` | 材质编号 |
| `bTransparent` | `bool` | `false` | 透明材质开关 |
| `EffectTransform` | `FTransform` | `Identity` | 效果网格变换 |
| `ComponentDimmer` | `float` | `1.0` | 组件级亮度分控 |

### 材质

| 属性 | 类型 | 说明 |
|------|------|------|
| `EffectMaterial` | `UMaterialInstance*` | 不透明效果材质 |
| `EffectMaterialTransparent` | `UMaterialInstance*` | 半透明效果材质 |
| `DynamicMaterialEffect` | `UMaterialInstanceDynamic*` | 当前动态材质 |

### 函数

#### 初始化

```cpp
void SetEffectMaterial(float NewIntensity = 1.0f);
```

创建动态材质实例并应用到网格。根据 `bTransparent` 选择源材质。

---

#### 亮度与频闪

```cpp
void SetEffectIntensity(float NewIntensity);
void SetEffectStrobe(float NewStrobe = 0.f);
void SetEffectStrobeMode(float NewStrobeMode = 1.f);
void SetRandomSeed(float NewSeed);
```

```cpp
UFUNCTION(BlueprintCallable, meta = (DisplayName = "SetComponentDimmer"))
void SetComponentDimmer(float NewDimmer);
```

---

#### 颜色与效果

```cpp
void SetEffectColor(FLinearColor NewColor = FLinearColor(1,1,1)) const;
void SetEffectControl(float NewEffect = 0.f, float NewSpeed = 1.f, float NewWidth = 0.5f) const;
```

---

#### 综合控制

```cpp
void SetEffectsControl(
    float NewIntensity = 1.f,
    float NewStrobe = 0.f,
    FLinearColor NewColor = FLinearColor(1,1,1),
    float NewDirection = 0.f,
    float NewEffect = 0.f,
    float NewSpeed = 1.f,
    float NewWidth = 0.5f) const;
```

一次性设置所有效果参数（亮度/频闪/颜色/方向/效果/速度/宽度）。

---

## 8. USuperMatrixComponent — 矩阵灯组件

**头文件**: `LightComponent/SuperMatrixComponent.h`  
**基类**: `USceneComponent`  
**导出宏**: `SUPERSTAGE_API`  
**蓝图标识**: `DisplayName = "SuperMatrixComponent"`

矩阵灯组件，按分段（Segment）控制颜色、频闪和亮度。每个分段对应一个像素。支持分段光源（PointLight/SpotLight）模拟真实光照。

### 子组件

| 组件 | 类型 | 说明 |
|------|------|------|
| `YStaticMeshMatrix` | `UStaticMeshComponent*` | 矩阵网格 |
| `SegmentPointLights` | `TArray<USilentPointLightComponent*>` | 分段点光源数组 |
| `SegmentSpotLights` | `TArray<USilentMatrixSpotLightComponent*>` | 分段聚光灯数组 |

### 默认参数

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `SegCount` | `int32` | `5` | 分段数量（≤200） |
| `bUseUAxis` | `bool` | `true` | 分段方向（true=U 轴，false=V 轴） |
| `StaticMeshMatrix` | `UStaticMesh*` | — | 矩阵网格模型 |
| `bTransparent` | `bool` | `false` | 透明材质开关 |
| `MatrixTransform` | `FTransform` | `Identity` | 网格变换 |
| `ComponentDimmer` | `float` | `1.0` | 组件级亮度分控 |

### 分段光源配置

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `SegmentLightType` | `ESegmentLightType` | `None` | 光源类型 |
| `SegmentLightIntensity` | `float` | `1000.0` | 光源强度（流明） |
| `SegmentLightRadius` | `float` | `2400.0` | 衰减半径 |
| `SpotOuterConeAngle` | `float` | `90.0°` | SpotLight 外锥角 |

#### ESegmentLightType

| 值 | 说明 |
|----|------|
| `None` | 无光源（仅材质发光） |
| `PointLight` | 点光源 |
| `SpotLight` | 聚光灯 |

### 材质

| 属性 | 类型 | 说明 |
|------|------|------|
| `MatrixMaterial` | `UMaterialInstance*` | 不透明矩阵材质 |
| `MatrixMaterialTransparent` | `UMaterialInstance*` | 半透明矩阵材质 |
| `DynamicMaterialMatrix` | `UMaterialInstanceDynamic*` | 当前动态材质 |

**材质参数命名**: 分段颜色参数名为 `C00`、`C01`、...、`C99`（与 SuperLightMatrix 材质一致）。

### 函数

#### 初始化

```cpp
UFUNCTION(BlueprintCallable, meta = (DisplayName = "SetMatrixMaterial"))
void SetMatrixMaterial(float NewIntensity = 1.0f);
```

创建动态材质实例，推送 `SegCount`/`UseUAxis` 核心参数。

---

#### 颜色

```cpp
UFUNCTION(BlueprintCallable, meta = (DisplayName = "SetSegmentColor"))
void SetSegmentColor(int32 Index = 0, FLinearColor RGB = FLinearColor::White);
```

设置单个分段的颜色。`Index` 从 0 开始。

---

#### 频闪

```cpp
UFUNCTION(BlueprintCallable, meta = (DisplayName = "SetMatrixStrobe"))
void SetMatrixStrobe(float NewStrobe = 255.f);

UFUNCTION(BlueprintCallable)
void SetMatrixStrobeMode(float NewStrobeMode = 1.f);
```

---

#### 亮度

```cpp
UFUNCTION(BlueprintCallable)
void SetMatrixIntensity(float NewIntensity);

UFUNCTION(BlueprintCallable, meta = (DisplayName = "SetComponentDimmer"))
void SetComponentDimmer(float NewDimmer);
```

---

#### 分段光源管理

```cpp
UFUNCTION(BlueprintCallable)
void RebuildSegmentLights();    // 重建分段光源

UFUNCTION(BlueprintCallable)
void DestroyAllSegmentLights(); // 销毁所有分段光源

UFUNCTION(BlueprintCallable)
void UpdateLightParameters();   // 更新光源参数（无需重建）
```

---

## 9. USuperLiftComponent — 升降组件

**头文件**: `LightComponent/SuperLiftComponent.h`  
**基类**: `USceneComponent`  
**导出宏**: `SUPERSTAGE_API`  
**蓝图标识**: `DisplayName = "SuperLiftComponent"`

Z 轴升降控制组件。不使用 Tick 自动更新，由父 Actor 主动调用。

### 函数

```cpp
void SetLiftZ(float InPosZ = 0.f, float LiftRange = 100.f, float LiftSpeed = 1.f);
```

| 参数 | 说明 |
|------|------|
| `InPosZ` | 归一化升降位置 `[0, 1]` |
| `LiftRange` | 升降范围（cm） |
| `LiftSpeed` | 插值速度 |

**行为**: `Clamp(InPosZ, 0, 1) → Lerp(0, LiftRange) → FInterpTo(CurrentZ, Target, DeltaTime, LiftSpeed) → SetRelativeLocation(0, 0, NewZ)`

### 缓存

| 属性 | 类型 | 说明 |
|------|------|------|
| `CurrentZ` | `float` | 当前插值后的 Z 位置 |

---

## 10. USuperLaserProComponent — 激光 Pro 组件

**头文件**: `LightComponent/SuperLaserProComponent.h`  
**基类**: `USceneComponent`  
**导出宏**: `SUPERSTAGE_API`  
**蓝图标识**: `DisplayName = "SuperLaserProComponent"`

基于程序化网格实现点数据驱动的激光线可视化。每个点对应一条从光源到目标点的激光线。

### 子组件（内部）

| 组件 | 类型 | 说明 |
|------|------|------|
| `YBeamMeshComponent` | `UProceduralMeshComponent*` | 激光线网格 |
| `DynamicMaterialInstance` | `UMaterialInstanceDynamic*` | 激光材质 |

### 默认参数

| 属性 | 类型 | 默认值 | 范围 | 说明 |
|------|------|--------|------|------|
| `BeamLength` | `float` | — | 100-50000 cm | 激光投射距离 |
| `ProjectionAngle` | `float` | — | 1-90° | 投射角度范围 |
| `LaserWidth` | `float` | — | 0.1-20 cm | 激光线宽度 |
| `CoreSharpness` | `float` | — | 1-10 | 核心锐度（集中程度） |
| `DepthFade` | `float` | — | 0-1 | 深度衰减（0=不衰减） |
| `Dim` | `float` | — | 1-10 | 自发光强度（指数曲线） |
| `OpacityScale` | `float` | — | 0-1 | 整体透明度 |
| `FogSpeed` | `float` | — | 0-2 | 烟雾流动速度 |
| `FogInfluence` | `float` | — | 0-1 | 烟雾影响强度 |
| `SpotDimmer` | `float` | — | 0-5 | 光斑亮度（打在物体上） |
| `ComponentDimmer` | `float` | `1.0` | 0-1 | 组件级亮度分控 |
| `bEnableCollision` | `bool` | — | — | 碰撞遮挡检测 |
| `BeamMaterial` | `UMaterialInterface*` | — | — | 激光线材质 |

### 函数

#### 核心接口

```cpp
UFUNCTION(BlueprintCallable, meta = (DisplayName = "SetLaserPoints"))
void SetLaserPoints(const TArray<FLaserPoint>& InPoints);
```

设置激光点数据（每帧从 Subsystem 获取）。自动触发网格重建。

```cpp
UFUNCTION(BlueprintCallable, meta = (DisplayName = "RebuildMesh"))
void RebuildMesh();
```

强制重建网格。

```cpp
UFUNCTION(BlueprintCallable, meta = (DisplayName = "SetLaserVisibility"))
void SetLaserVisibility(bool bNewVisibility);
```

设置可见性。

```cpp
UFUNCTION(BlueprintCallable, meta = (DisplayName = "SetComponentDimmer"))
void SetComponentDimmer(float NewDimmer);
```

设置组件级亮度分控。

### 性能优化

- **网格缓存**: `CachedVertexCount` + `bMeshNeedsRebuild` 避免每帧重建
- **材质参数缓存**: 变更检测，值未变化时跳过
- **碰撞距离缓存**: `HitDistances` 数组，仅 `bEnableCollision` 时射线检测
- **角度缓存**: `CachedTanAngle` 避免每帧计算 `tan(ProjectionAngle)`

### 渲染流程

```
SetLaserPoints(InPoints)
 → CachedLaserPoints = InPoints
 → bMeshNeedsRebuild = true
 → TraceAllCollisions()  // 碰撞检测（可选）
 → BuildLaserMesh()      // 程序化网格生成
 → UpdateMaterialParameters() // 材质参数同步
```

---

## 11. 辅助光源类

### USilentSpotLightComponent

```cpp
class SUPERSTAGE_API USilentSpotLightComponent : public USpotLightComponent
```

**定义位置**: `SuperSpotComponent.h`

安全版 SpotLight。提供 `SetAttenuationRadiusSafe(float)` 绕过 `PushRadiusToRenderThread()` 避免 GPU Scene 同步断言。

### USilentRectLightComponent

```cpp
class SUPERSTAGE_API USilentRectLightComponent : public URectLightComponent
```

**定义位置**: `SuperRectComponent.h`

空壳子类，用于隐藏编辑器图标。

### USilentPointLightComponent

```cpp
class SUPERSTAGE_API USilentPointLightComponent : public UPointLightComponent
```

**定义位置**: `SuperMatrixComponent.h`

静默版点光源，用于矩阵分段光源。

### USilentMatrixSpotLightComponent

```cpp
class SUPERSTAGE_API USilentMatrixSpotLightComponent : public USpotLightComponent
```

**定义位置**: `SuperMatrixComponent.h`

静默版聚光灯，用于矩阵分段光源。

---

## 12. API 速查表

### 组件选择指南

| 灯具类型 | 推荐组件 |
|---------|---------|
| Profile / 追光 | `USuperCuttingComponent` |
| 标准电脑灯（Spot/Beam） | `USuperBeamComponent` |
| Wash 灯 / LED 面光 | `USuperRectComponent` 或 `USuperSpotComponent` |
| LED 矩阵灯 | `USuperMatrixComponent` |
| 效果灯（LED 灯带/面板） | `USuperEffectComponent` |
| 激光（Beyond） | `USuperLaserProComponent` |
| 灯具升降 | `USuperLiftComponent` |
| 辅光/补光 | `USuperSpotComponent` |

### 通用接口（USuperLightingComponent 及子类）

| 函数 | 说明 |
|------|------|
| `SetLightingMaterial()` | 初始化材质 |
| `SetLightingDefaultValue(...)` | 设置默认参数 |
| `SetLightingIntensity(float)` | 亮度 |
| `SetLightingStrobe(float)` | 频闪速度 |
| `SetLightingStrobeMode(float)` | 频闪模式 |
| `SetLightingColor(FLinearColor)` | 颜色 |
| `SetLightingZoom(float)` | 变焦 |
| `SetLightingFrost(float)` | 雾化 |
| `SetLightingIris(float)` | 光圈 |
| `SetLightingTexture(UTexture2D*)` | 纹理 |
| `SetColorTexture/2/3(...)` | 颜色轮贴图 |
| `SetLightingRotate(float, float)` | 旋转 |
| `SetLightingVisibility(bool)` | 可见性 |
| `SetLightingLensVisibility(bool)` | 镜头可见性 |
| `SetLightingChannels(...)` | 光照通道 |
| `SetLightingTransmission(bool)` | 透射 |
| `SetSpecularScale(float)` | 高光缩放 |
| `SetSourceRadius(float)` | 软阴影半径 |
| `SetComponentDimmer(float)` | 组件亮度分控 |
| `GetRayDetectionDistance(float)` | 射线检测距离 |
| `UpdateBeamBlockDistance(float)` | 光束遮挡距离 |

### 光束专用接口（USuperBeamComponent）

| 函数 | 说明 |
|------|------|
| `SetBeamDefaultValue(...)` | 光束默认参数 |
| `SetBeamTexture(...)` | Gobo 纹理 |
| `SetBeamPrism(...)` | 棱镜 |
| `SetBeamFocus(float)` | 调焦 |

### 切割专用接口（USuperCuttingComponent）

| 函数 | 说明 |
|------|------|
| `SetCuttingValue(8个float)` | 四角八点切割 |

### 效果专用接口（USuperEffectComponent）

| 函数 | 说明 |
|------|------|
| `SetEffectMaterial(float)` | 初始化 |
| `SetEffectIntensity(float)` | 亮度 |
| `SetEffectStrobe(float)` | 频闪 |
| `SetEffectStrobeMode(float)` | 频闪模式 |
| `SetEffectColor(FLinearColor)` | 颜色 |
| `SetEffectControl(float,float,float)` | 效果/速度/宽度 |
| `SetEffectsControl(...)` | 综合控制 |

### 矩阵专用接口（USuperMatrixComponent）

| 函数 | 说明 |
|------|------|
| `SetMatrixMaterial(float)` | 初始化 |
| `SetSegmentColor(int32, FLinearColor)` | 分段颜色 |
| `SetMatrixStrobe(float)` | 频闪 |
| `SetMatrixStrobeMode(float)` | 频闪模式 |
| `SetMatrixIntensity(float)` | 亮度 |
| `RebuildSegmentLights()` | 重建分段光源 |
| `DestroyAllSegmentLights()` | 销毁分段光源 |
| `UpdateLightParameters()` | 更新光源参数 |

### 激光专用接口（USuperLaserProComponent）

| 函数 | 说明 |
|------|------|
| `SetLaserPoints(TArray<FLaserPoint>)` | 设置点数据 |
| `RebuildMesh()` | 重建网格 |
| `SetLaserVisibility(bool)` | 可见性 |

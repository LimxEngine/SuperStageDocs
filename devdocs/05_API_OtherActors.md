# 其他 Actor 类 API 文档

> **文档版本**: 1.0  
> **适用版本**: SuperStage Plugin 2025/2026  
> **目标读者**: 蓝图开发者 / C++ 插件集成开发者（无源码修改权限）  
> **最后更新**: 2026-03

---

## 目录

1. [ASuperDMXCamera — DMX 摄像机](#1-asuperdmxcamera--dmx-摄像机)
2. [ASuperLiftingMachinery — 升降机械](#2-asuperliftingmachinery--升降机械)
3. [ASuperRailMachinery — 轨道机械](#3-asuperrailmachinery--轨道机械)
4. [ASuperStageVFXActor — 舞台特效](#4-asuperstagevfxactor--舞台特效)
5. [ASuperLightStripEffect — LED 灯带特效](#5-asuperlightstripeffect--led-灯带特效)
6. [ASuperLiftMatrix — 升降矩阵](#6-asuperliftmatrix--升降矩阵)
7. [ASuperLaserActor — 激光显示（纹理模式）](#7-asuperlaseractor--激光显示纹理模式)
8. [ASuperLaserProActor — 激光显示（点数据模式）](#8-asuperlaserproactor--激光显示点数据模式)
9. [ASuperNDIScreen — NDI 视频屏幕](#9-asuperndiscreen--ndi-视频屏幕)
10. [ASuperProjector — 投影仪 / Mapping](#10-asuperprojector--投影仪--mapping)
11. [API 速查表](#11-api-速查表)

---

## 1. ASuperDMXCamera — DMX 摄像机

**头文件**: `LightActor/SuperDMXCamera.h`  
**基类**: `ASuperDmxActorBase`  
**导出宏**: `SUPERSTAGE_API`

通过 DMX 控制电影摄像机的 6 轴运动（3 轴位移 + 3 轴旋转）以及摄像机参数（FOV/光圈/对焦）。支持渲染到 RenderTarget，可用于 LED 屏幕实时内容输出。

### 继承链

```
AActor → ASuperBaseActor → ASuperDmxActorBase → ASuperDMXCamera
```

### 组件

| 组件 | 类型 | 说明 |
|------|------|------|
| `CineCamera` | `UCineCameraComponent*` | 电影摄像机组件 |
| `SceneCapture` | `USceneCaptureComponent2D*` | 场景捕获组件（渲染到 RenderTarget） |

### 默认参数

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `Start` | `bool` | `false` | 启动开关 |
| `MovingRange` | `FVector` | `(0,0,0)` | 移动范围（cm） |
| `InitialPosition` | `FVector` | — | 初始位置（`BootRefresh` 计算，只读） |
| `EndPosition` | `FVector` | — | 结束位置（`BootRefresh` 计算，只读） |
| `RotRange` | `FRotator` | `(180,360,90)` | 旋转范围（度） |
| `InitialRotation` | `FRotator` | — | 初始旋转（只读） |
| `EndRotation` | `FRotator` | — | 结束旋转（只读） |
| `FOVRange` | `FVector2D` | `(15, 120)` | FOV 范围（度） |
| `ApertureRange` | `FVector2D` | `(1.2, 22)` | 光圈范围（f-stop） |
| `FocusDistanceRange` | `FVector2D` | `(50, 100000)` | 对焦距离范围（cm） |
| `CameraParamSpeed` | `float` | `3.0` | 摄像机参数插值速度（0-10） |
| `RenderTarget` | `UTextureRenderTarget2D*` | `nullptr` | 渲染输出目标 |
| `RenderResolution` | `FIntPoint` | `(1920,1080)` | 渲染分辨率 |
| `bEnableSceneCapture` | `bool` | `false` | 启用场景捕获 |

### 控制参数

所有控制参数为 `[0, 1]` 归一化值，支持 `Interp`（Sequencer 动画）。

| 属性 | 默认值 | 说明 |
|------|--------|------|
| `PosX` / `PosY` / `PosZ` | `0.5` | XYZ 位置控制 |
| `RotX` / `RotY` / `RotZ` | `0.5` | XYZ 旋转控制 |
| `FOV` | `0.5` | 视场角 |
| `Aperture` | `0.5` | 光圈 |
| `FocusDistance` | `0.5` | 对焦距离 |

### 缓存信息（只读）

| 属性 | 说明 |
|------|------|
| `CurrentPosX/Y/Z` | 当前插值后的位置 |
| `CurrentRotX/Y/Z` | 当前插值后的旋转 |
| `CurrentFOV` | 当前 FOV（度） |
| `CurrentAperture` | 当前光圈（f-stop） |
| `CurrentFocusDistance` | 当前对焦距离（cm） |

### 函数

#### BootRefresh

```cpp
UFUNCTION(CallInEditor, Category = "B.DefaultParameter",
    meta = (DisplayName = "BootRefresh"))
void BootRefresh();
```

**编辑器按钮**。记录当前位置/旋转为基准，计算 `InitialPosition`/`EndPosition` 和 `InitialRotation`/`EndRotation`。

**计算公式**:
```
InitialPosition = CurrentWorldPosition - MovingRange * 0.5
EndPosition     = CurrentWorldPosition + MovingRange * 0.5
InitialRotation = CurrentRotation - RotRange * 0.5
EndRotation     = CurrentRotation + RotRange * 0.5
```

> **重要**: 必须在 Actor 放置到正确位置后调用。

---

#### DMXCameraControl

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|Camera|Transform",
    meta = (DisplayName = "DMXCameraControl"))
void DMXCameraControl(
    FSuperDMXAttribute DmxXPos, FSuperDMXAttribute DmxYPos, FSuperDMXAttribute DmxZPos,
    FSuperDMXAttribute DmxXRot, FSuperDMXAttribute DmxYRot, FSuperDMXAttribute DmxZRot);
```

6 轴运动控制（3 轴位移 + 3 轴旋转）。在 `SuperDMXTick` 中调用。

**DMX 映射**:
- 位置: `DMX[0,1] → Lerp(InitialPosition, EndPosition)`
- 旋转: `DMX[0,1] → Lerp(InitialRotation, EndRotation)`

所有轴使用 `FInterpTo` 平滑插值。

---

#### DMXCameraParams

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|Camera|Params",
    meta = (DisplayName = "DMXCameraParams"))
void DMXCameraParams(
    FSuperDMXAttribute DmxFOV,
    FSuperDMXAttribute DmxAperture,
    FSuperDMXAttribute DmxFocusDistance);
```

摄像机参数控制。

**DMX 映射**:
- FOV: `DMX[0,1] → Lerp(FOVRange.X, FOVRange.Y)`
- 光圈: `DMX[0,1] → Lerp(ApertureRange.X, ApertureRange.Y)`
- 对焦: `DMX[0,1] → Lerp(FocusDistanceRange.X, FocusDistanceRange.Y)`

---

#### GetCineCamera / GetRenderTarget

```cpp
UFUNCTION(BlueprintPure, meta = (DisplayName = "GetCineCamera"))
UCineCameraComponent* GetCineCamera() const;

UFUNCTION(BlueprintPure, meta = (DisplayName = "GetRenderTarget"))
UTextureRenderTarget2D* GetRenderTarget() const;
```

纯函数，获取摄像机组件和渲染目标引用。

---

### 使用示例

```
1. 在场景中放置 ASuperDMXCamera
2. 设置 MovingRange（如 (500, 500, 300)）和 RotRange
3. 点击 BootRefresh 初始化范围
4. 设置 Start = true
5. 在 SuperDMXTick 中连接：
   DMXCameraControl(DmxX, DmxY, DmxZ, DmxRotX, DmxRotY, DmxRotZ)
   DMXCameraParams(DmxFOV, DmxAperture, DmxFocus)
6. （可选）启用 bEnableSceneCapture 并设置 RenderTarget 输出到 LED 屏幕
```

---

## 2. ASuperLiftingMachinery — 升降机械

**头文件**: `LightActor/SuperLiftingMachinery.h`  
**基类**: `ASuperDmxActorBase`  
**导出宏**: `SUPERSTAGE_API`

通过 DMX 控制舞台设备的 6 轴运动（3 轴位移 + 3 轴旋转），支持绝对旋转和无极旋转两种模式。适用于升降台、旋转舞台、机械臂等设备。

### 继承链

```
AActor → ASuperBaseActor → ASuperDmxActorBase → ASuperLiftingMachinery
```

### 默认参数

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `Start` | `bool` | `false` | 启动开关 |
| `MovingRange` | `FVector` | `(0,0,0)` | 移动范围（cm） |
| `InitialPosition` | `FVector` | — | 初始位置（只读） |
| `EndPosition` | `FVector` | — | 结束位置（只读） |
| `RotRange` | `FRotator` | `(360,360,360)` | 旋转范围（度，绝对模式） |
| `InitialRotation` | `FRotator` | — | 初始旋转（只读，绝对模式） |
| `EndRotation` | `FRotator` | — | 结束旋转（只读，绝对模式） |
| `PosSpeed` | `float` | `1.0` | 移动插值速度（0-10） |
| `RotSpeed` | `float` | `1.0` | 旋转插值速度（0-10，绝对模式） |
| `PolarRotationSpeed` | `float` | `1.0` | 无极旋转速度（0-10，无极模式） |
| `PolarRotation` | `bool` | `false` | 无极旋转模式开关 |

### 控制参数

| 属性 | 默认值 | 说明 |
|------|--------|------|
| `PosX` / `PosY` / `PosZ` | `0.5` | XYZ 位置控制 `[0, 1]` |
| `RotX` / `RotY` / `RotZ` | `0.5` | XYZ 旋转控制 `[0, 1]` |

### 缓存信息（只读）

`CurrentPosX/Y/Z`、`CurrentRotX/Y/Z` — 当前插值后的实际值。

### 函数

#### BootRefresh

```cpp
UFUNCTION(CallInEditor, meta = (DisplayName = "BootRefresh"))
void BootRefresh();
```

编辑器按钮。记录当前位姿为基准，计算运动范围。

---

#### LiftingMachinery

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|Machinery|Transform",
    meta = (DisplayName = "LiftingMachinery"))
void LiftingMachinery(
    FSuperDMXAttribute DmxXPos, FSuperDMXAttribute DmxYPos, FSuperDMXAttribute DmxZPos,
    FSuperDMXAttribute DmxXRot, FSuperDMXAttribute DmxYRot, FSuperDMXAttribute DmxZRot);
```

6 轴 DMX 控制函数。在 `SuperDMXTick` 中调用。

**行为**:
- **位移**: `DMX[0,1] → Lerp(InitialPosition, EndPosition)` + `FInterpTo(PosSpeed)`
- **旋转（绝对）**: `DMX[0,1] → Lerp(InitialRotation, EndRotation)` + `FInterpTo(RotSpeed)`
- **旋转（无极）**: `DMX[0,1] → 速度[-PolarRotationSpeed, +PolarRotationSpeed]` → 每帧累加

---

### 使用示例

```
1. 放置 ASuperLiftingMachinery 到场景
2. 设置 MovingRange（如 (0, 0, 500) 仅做升降）
3. 点击 BootRefresh
4. Start = true
5. SuperDMXTick 中：
   LiftingMachinery(DmxX, DmxY, DmxZ, DmxRotX, DmxRotY, DmxRotZ)
```

**纯升降场景**: `MovingRange = (0, 0, 500)`, `RotRange = (0, 0, 0)`, 只传 `DmxZPos` 即可。

---

## 3. ASuperRailMachinery — 轨道机械

**头文件**: `LightActor/SuperRailMachinery.h`  
**基类**: `ASuperDmxActorBase`  
**导出宏**: `SUPERSTAGE_API`

沿 `USplineComponent` 样条曲线路径运动 + 6 轴局部偏移/旋转的复合控制。共 **7 个 DMX 控制轴**：轨道位置(RailPos) + XYZ 偏移 + XYZ 旋转。

### 继承链

```
AActor → ASuperBaseActor → ASuperDmxActorBase → ASuperRailMachinery
```

### 组件结构

```
SceneBase
 ├─ RailSplineComponent (USplineComponent, 轨道路径)
 └─ RailMountPoint (USceneComponent, 沿样条移动)
     └─ OffsetComponent (USceneComponent, 6轴偏移/旋转)
         └─ 子Actor/组件挂载点
```

| 组件 | 类型 | 说明 |
|------|------|------|
| `RailSplineComponent` | `USplineComponent*` | 样条曲线（编辑器可拖拽控制点） |
| `RailMountPoint` | `USceneComponent*` | 轨道挂载点（沿 Spline 移动） |
| `OffsetComponent` | `USceneComponent*` | 偏移组件（6 轴局部变换） |

> **挂载方式**: 子 Actor 应附加到 `GetDefaultAttachComponent()` 返回的 `OffsetComponent`。

### 默认参数

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `Start` | `bool` | `false` | 启动开关 |
| `bLockOrientationToRail` | `bool` | `false` | 锁定朝向沿轨道切线方向 |
| `bClosedLoop` | `bool` | `false` | 闭合轨道（首尾相连循环） |
| `OffsetRange` | `FVector` | `(0,0,0)` | 偏移范围（cm） |
| `InitialOffset` / `EndOffset` | `FVector` | — | 偏移范围（`BootRefresh` 计算，只读） |
| `RotRange` | `FRotator` | `(360,360,360)` | 旋转范围（绝对模式） |
| `InitialRotation` / `EndRotation` | `FRotator` | — | 旋转范围（只读） |
| `RailSpeed` | `float` | `1.0` | 轨道移动插值速度（0-10） |
| `OffsetSpeed` | `float` | `1.0` | 偏移移动插值速度（0-10） |
| `RotSpeed` | `float` | `1.0` | 旋转插值速度（绝对模式，0-10） |
| `PolarRotationSpeed` | `float` | `1.0` | 无极旋转速度（0-10） |
| `PolarRotation` | `bool` | `false` | 无极旋转模式开关 |

### 控制参数

| 属性 | 默认值 | 说明 |
|------|--------|------|
| `RailPos` | `0.0` | 轨道位置 `[0,1]`（0=起点，1=终点） |
| `PosX` / `PosY` / `PosZ` | `0.5` | XYZ 偏移控制 `[0, 1]` |
| `RotX` / `RotY` / `RotZ` | `0.5` | XYZ 旋转控制 `[0, 1]` |

### 函数

#### BootRefresh

```cpp
UFUNCTION(CallInEditor, meta = (DisplayName = "BootRefresh"))
void BootRefresh();
```

编辑器按钮。记录当前偏移/旋转为基准。

---

#### RailMachinery

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|Machinery|Rail",
    meta = (DisplayName = "RailMachinery"))
void RailMachinery(
    FSuperDMXAttribute DmxRailPos,
    FSuperDMXAttribute DmxXPos, FSuperDMXAttribute DmxYPos, FSuperDMXAttribute DmxZPos,
    FSuperDMXAttribute DmxXRot, FSuperDMXAttribute DmxYRot, FSuperDMXAttribute DmxZRot);
```

7 轴 DMX 控制函数。在 `SuperDMXTick` 中调用。

**DMX 映射**:
- **轨道位置**: `DMX[0,1] → SplineLength × t → 沿样条位置`
- **偏移**: `DMX[0,1] → Lerp(InitialOffset, EndOffset)`
- **旋转**: 同 `ASuperLiftingMachinery` 的绝对/无极模式

**闭合轨道插值**: 开启 `bClosedLoop` 时使用最短弧路径插值（`[-0.5, 0.5]` 区间），避免绕远路。

---

#### RailPositionOnly

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|Machinery|Rail",
    meta = (DisplayName = "RailPositionOnly"))
void RailPositionOnly(FSuperDMXAttribute DmxRailPos);
```

仅控制轨道位置，不含偏移/旋转。适用于只需沿轨道移动的简单场景。

---

### 使用示例

```
1. 放置 ASuperRailMachinery 到场景
2. 编辑 RailSplineComponent 控制点定义轨道路径
3. 设置 OffsetRange, RotRange
4. 点击 BootRefresh
5. Start = true
6. SuperDMXTick 中：
   RailMachinery(DmxRail, DmxX, DmxY, DmxZ, DmxRotX, DmxRotY, DmxRotZ)
7. 将灯具/道具等子 Actor 附加到 OffsetComponent
```

---

## 4. ASuperStageVFXActor — 舞台特效

**头文件**: `LightActor/SuperStageVFXActor.h`  
**基类**: `ASuperLightBase`  
**导出宏**: `SUPERSTAGE_API`

通过 DMX 控制 Niagara 粒子系统（烟雾/火焰/雪花/CO2/彩带/烟花等）。继承自 `ASuperLightBase`，因此支持 Pan/Tilt 旋转控制。

### 继承链

```
AActor → ASuperBaseActor → ASuperDmxActorBase → ASuperLightBase → ASuperStageVFXActor
```

### 组件

| 组件 | 类型 | 说明 |
|------|------|------|
| `Niagara` | `UNiagaraComponent*` | Niagara 粒子发射器 |

### 默认参数

| 属性 | 类型 | 默认值 | 范围 | 说明 |
|------|------|--------|------|------|
| `MaxSpawnCount` | `float` | `1.0` | 0-100 | 最大粒子生成数量 |
| `MaxSpeed` | `float` | `100` | 0-100 | 最大速度 |
| `LoopTime` | `float` | `100` | 0-100 | 循环时间（秒） |
| `Minimum` | `float` | `100` | 0-100000 | 最小范围（cm） |
| `Maximum` | `float` | `100` | 0-100000 | 最大范围（cm） |

### 控制参数

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `SpawnCount` | `float` | `0.0` | 粒子生成量控制 `[0, 1]` |
| `Color` | `FLinearColor` | `White` | 粒子颜色 |

### 函数

#### SetNiagaraDefaultValue

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|VFX|Init",
    meta = (DisplayName = "SetNiagaraDefaultValue"))
void SetNiagaraDefaultValue();
```

初始化 Niagara 系统参数。设置 `Speed`、`LoopTime`、`Minimum`、`Maximum` 变量并激活系统。

---

#### SetNiagaraSpawnCount

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|VFX|Control",
    meta = (DisplayName = "SetNiagaraSpawnCount"))
void SetNiagaraSpawnCount(const FSuperDMXAttribute DmxAttribute);
```

DMX 控制粒子生成数量。

**映射**: `DMX[0,1] → Lerp(0, MaxSpawnCount) → RoundToInt → SetVariableInt("SpawnCount")`

---

#### SetNiagaraColor

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|VFX|Color",
    meta = (DisplayName = "SetNiagaraColor"))
void SetNiagaraColor(const FSuperDMXAttribute DMXAttR,
    const FSuperDMXAttribute DMXAttG, const FSuperDMXAttribute DMXAttB);
```

DMX 控制粒子颜色（RGB 三通道）。

**映射**: `GetSuperDMXColorValue(R,G,B) → SetVariableLinearColor("Color")`

---

### Niagara 变量映射表

| Niagara 变量名 | 类型 | 来源 |
|---------------|------|------|
| `Speed` | `float` | `MaxSpeed` |
| `LoopTime` | `float` | `LoopTime` |
| `Minimum` | `float` | `Minimum` |
| `Maximum` | `float` | `Maximum` |
| `SpawnCount` | `int32` | DMX 动态控制 |
| `Color` | `FLinearColor` | DMX 动态控制 |

---

### 使用示例

```
1. 创建 Niagara System 资产（烟雾/火焰等）
2. 创建 ASuperStageVFXActor 子类蓝图
3. 将 Niagara System 分配给 Niagara 组件
4. 设置 MaxSpawnCount, MaxSpeed 等默认参数
5. SuperDMXTick 中：
   SetNiagaraDefaultValue()       // 首帧初始化
   SetNiagaraSpawnCount(DmxSpawn) // 生成量
   SetNiagaraColor(DmxR, DmxG, DmxB) // 颜色
   YPan(DmxPan)                   // Pan（继承）
   YTilt(DmxTilt)                 // Tilt（继承）
```

---

## 5. ASuperLightStripEffect — LED 灯带特效

**头文件**: `LightActor/SuperLightStripEffect.h`  
**基类**: `ASuperDmxActorBase`  
**导出宏**: `SUPERSTAGE_API`

通过材质实现可编程 LED 灯带效果（流水、追逐等）。支持将效果材质批量应用到场景中的多个 StaticMeshActor。

### 继承链

```
AActor → ASuperBaseActor → ASuperDmxActorBase → ASuperLightStripEffect
```

### 默认参数

| 属性 | 类型 | 默认值 | 范围 | 说明 |
|------|------|--------|------|------|
| `MaxLightIntensity` | `float` | `1.0` | 0-1000 | 最大亮度 |
| `MaterialIndex` | `int` | `0` | 0-255 | 目标网格的材质槽索引 |
| `TargetMeshActors` | `TArray<AStaticMeshActor*>` | — | — | 批量应用的目标网格体 |
| `StaticMeshEffect` | `UStaticMesh*` | — | — | 自身效果网格 |

### 控制参数

| 属性 | 类型 | 默认值 | 范围 | 说明 |
|------|------|--------|------|------|
| `Dimmer` | `float` | `0.0` | 0-1 | 亮度 |
| `Strobe` | `float` | `0.0` | 0-1 | 频闪 |
| `Effect` | `float` | `0.0` | 0-1 | 效果选择 |
| `Speed` | `float` | `0.5` | 0-1 | 效果速度 |
| `Width` | `float` | `0.0` | 0-1 | 效果宽度 |
| `Direction` | `float` | `0.0` | 0-1 | 效果方向 |
| `Color` | `FLinearColor` | `White` | — | RGB 颜色 |

### 材质参数映射

| 材质参数名 | DMX 映射 |
|-----------|---------|
| `MaxBrightness` | `MaxLightIntensity` |
| `Brightness` | `Dimmer` |
| `Strobe` | `Strobe × 255` |
| `StrobeMode` | 子属性定义 |
| `LightColor` | `Color` |
| `Effect` | `Effect → Lerp(0, 10)` |
| `Speed` | `Speed → Lerp(-10, 10)` |
| `Width` | `Width → Lerp(0, 10)` |
| `EffectDirection` | `Direction` |

### 函数

#### SetLightStripDefault

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|LightStrip|Init")
void SetLightStripDefault();
```

初始化灯带材质。创建动态材质实例并应用到自身网格和所有 `TargetMeshActors`。

---

#### SetLightStripEffect

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|LightStrip|Control")
void SetLightStripEffect(
    FSuperDMXAttribute DmxDimmer, FSuperDMXAttribute DmxStrobe,
    FSuperDMXAttribute DmxEffect, FSuperDMXAttribute DmxSpeed,
    FSuperDMXAttribute DmxWidth,  FSuperDMXAttribute DmxDirection,
    FSuperDMXAttribute DmxRed,    FSuperDMXAttribute DmxGreen,
    FSuperDMXAttribute DmxBlue);
```

全通道 DMX 控制函数（9 通道）。在 `SuperDMXTick` 中调用。

---

### 频闪模式（StrobeMode 值）

| 值 | 模式 | 说明 |
|----|------|------|
| 0 | Closed | 关闭（常灭） |
| 1 | Open | 常亮 |
| 2 | Linear | 线性频闪 |
| 3 | Pulse | 脉冲频闪 |
| 4 | RampUp | 渐亮 |
| 5 | RampDown | 渐灭 |
| 6 | Sine | 正弦频闪 |
| 7 | Random | 随机频闪 |

---

## 6. ASuperLiftMatrix — 升降矩阵

**头文件**: `LightActor/SuperLiftMatrix.h`  
**基类**: `ASuperLightBase`  
**导出宏**: `SUPERSTAGE_API`

升降矩阵 Actor，包含 5 个垂直排列的场景组件（间距 5cm），每个组件下有两个 `USuperEffectComponent`。四角有钢丝绳模拟。

### 继承链

```
AActor → ASuperBaseActor → ASuperDmxActorBase → ASuperLightBase → ASuperLiftMatrix
```

### 组件

| 组件 | 类型 | 数量 | 说明 |
|------|------|------|------|
| `LiftComponents` | `TArray<USceneComponent*>` | 5 | 升降场景组件 |
| `EffectComponents` | `TArray<USuperEffectComponent*>` | 10 | 效果组件（每节 2 个） |
| `CableMeshes` | `TArray<UStaticMeshComponent*>` | 4 | 钢丝绳（四角） |
| `CableOrigin` | `USceneComponent*` | 1 | 钢丝绳起点 |

### 常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `ComponentSpacing` | `5.0` cm | 组件间距 |
| `ComponentCount` | `5` | 升降组件数量 |
| `DivisionCount` | `6` | 分布等分数 |
| `CableCornerOffset` | `22.0` cm | 钢丝绳角偏移 |
| `CableOriginHeight` | `15.0` cm | 钢丝绳起点高度 |
| `CableRadius` | `0.5` cm | 钢丝绳半径 |

### 默认参数

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `MaxLightIntensity` | `float` | `1.0` | 效果最大亮度 |

### 函数

#### LiftMatrix

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|LiftMatrix|Lift",
    meta = (DisplayName = "LiftMatrix"))
void LiftMatrix(FSuperDMXAttribute DmxLift);
```

升降控制。5 个组件按 6 等分分布，DMX 控制整体升降高度。自动更新钢丝绳长度。

---

#### SetEffectMatrixDefaultValue

```cpp
UFUNCTION(BlueprintCallable, meta = (DisplayName = "SetEffectMatrixDefaultValue"))
void SetEffectMatrixDefaultValue();
```

初始化所有效果组件的材质。

---

#### SetEffectMatrix / SetEffectColorMatrix

```cpp
void SetEffectMatrix(FSuperDMXAttribute DmxAttribute);
void SetEffectColorMatrix(FSuperDMXAttribute DMXAttR, FSuperDMXAttribute DMXAttG,
    FSuperDMXAttribute DMXAttB);
```

效果选择和颜色控制。

---

## 7. ASuperLaserActor — 激光显示（纹理模式）

**头文件**: `LightActor/SuperLaserActor.h`  
**基类**: `ASuperBaseActor`  
**导出宏**: `SUPERSTAGE_API`

在 UE 场景中实时显示激光投影纹理。从 `SuperLaserSubsystem` 获取指定 Beyond 设备的 RenderTarget 并应用到光束材质。

### 继承链

```
AActor → ASuperBaseActor → ASuperLaserActor
```

> **注意**: 此类不继承 `ASuperDmxActorBase`，**不使用 DMX 控制**。激光内容由外部 Beyond 软件生成。

### 默认参数

| 属性 | 类型 | 默认值 | 范围 | 说明 |
|------|------|--------|------|------|
| `DeviceID` | `int32` | `1` | 1-12 | Beyond 设备编号 |
| `LaserIntensity` | `float` | `100.0` | 1-5000 | 激光强度 |
| `MaxLaserDistance` | `float` | `2345.0` | 100-10000 | 最大投射距离（cm） |
| `AtmosphericDensity` | `float` | `0.03` | 0-1 | 大气衰减 |
| `FogIntensity` | `float` | `20.0` | 0-100 | 雾效强度（%） |
| `AtmosFogSpeed` | `float` | `10.0` | 0-100 | 雾效流速（%） |

### 渲染流程

```
Tick → GetLaserTexture(查询Subsystem)
     → SetLaserTexture(更新材质 LightTexture 参数)
     → SetLaserVisibility(根据强度自动显隐)
     → MarkRenderStateDirty(通知渲染线程)
```

---

## 8. ASuperLaserProActor — 激光显示（点数据模式）

**头文件**: `LightActor/SuperLaserProActor.h`  
**基类**: `ASuperBaseActor`  
**导出宏**: `SUPERSTAGE_API`

基于点数据驱动的激光显示。从 `SuperLaserSubsystem` 获取激光点数据，使用 `USuperLaserProComponent` 程序化生成网格绘制激光线。

### 继承链

```
AActor → ASuperBaseActor → ASuperLaserProActor
```

> **注意**: 此类不继承 `ASuperDmxActorBase`，**不使用 DMX 控制**。

### 默认参数

| 属性 | 类型 | 默认值 | 范围 | 说明 |
|------|------|--------|------|------|
| `DeviceID` | `int32` | `1` | 1-12 | Beyond 设备编号 |
| `bEnableCollision` | `bool` | `false` | — | 启用碰撞遮挡检测 |

### 组件

| 组件 | 类型 | 说明 |
|------|------|------|
| `LaserComponent` | `USuperLaserProComponent*` | 激光 Pro 组件 |

### 函数

#### SetLaserEnabled

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|LaserPro|Control")
void SetLaserEnabled(bool bEnabled);
```

控制激光可见性。

### 点数据格式

| 字段 | 范围 | 说明 |
|------|------|------|
| `X`, `Y` | `[-1, 1]` | 归一化位置（映射到投射范围） |
| `Z` | `>0` = 光束点 | 光束标记 |
| `R`, `G`, `B` | `[0, 1]` | 颜色 |

### 渲染流程

```
Tick → UpdateLaserPoints()
     → GetLaserPoints(查询Subsystem, DeviceID)
     → LaserComponent->SetLaserPoints(点数据)
     → 程序化网格绘制激光线
```

---

## 9. ASuperNDIScreen — NDI 视频屏幕

**头文件**: `LightActor/SuperNDIScreen.h`  
**基类**: `ASuperBaseActor`  
**导出宏**: `SUPERSTAGE_API`

从 `SuperNDISubsystem` 接收实时 NDI 视频流并显示到屏幕网格。支持多屏输出、梯形校正、透明材质切换。

### 继承链

```
AActor → ASuperBaseActor → ASuperNDIScreen
```

> **注意**: 此类不继承 `ASuperDmxActorBase`，**不使用 DMX 控制**。视频内容来自 NDI 网络流。

### 属性

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `InputName` | `FName` | `NAME_None` | NDI 输入名称（带下拉选择） |
| `OutputTexture` | `UTexture2D*` | — | 输出纹理（运行时创建，只读） |
| `TargetStaticMeshActors` | `TArray<AStaticMeshActor*>` | — | 目标屏幕网格体 |
| `bTransparent` | `bool` | `false` | 透明材质开关 |
| `Transparency` | `float` | `0.95` | 透明度（0-1，仅透明模式） |
| `Brightness` | `float` | `1.0` | 亮度 |
| `Color` | `FLinearColor` | `White` | 颜色滤镜 |
| `Contrast` | `float` | `1.5` | 对比度 |
| `Deformation` | `FDeformation` | — | 梯形校正参数 |

### FDeformation — 梯形校正

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `UpperLeftCorner` | `FVector2D` | `(0,0)` | 左上角 UV 偏移 `[0,1]` |
| `LowerLeftCorner` | `FVector2D` | `(0,0)` | 左下角 UV 偏移 |
| `UpperRightCorner` | `FVector2D` | `(0,0)` | 右上角 UV 偏移 |
| `LowerRightCorner` | `FVector2D` | `(0,0)` | 右下角 UV 偏移 |

### 数据流

```
NDI 网络流 → SuperNDISubsystem → OnFrame 回调
           → HandleNDIFrame(BGRA, Width, Height)
           → EnsureTexture(创建/重建纹理)
           → UpdateTextureGPU(RHI 上传到 GPU)
           → DynamicMaterial.SetTextureParameterValue
           → 应用到所有 TargetStaticMeshActors
```

### 纹理管理

- **格式**: `PF_B8G8R8A8`（BGRA8）
- **创建**: `UTexture2D::CreateTransient`
- **尺寸变化**: 释放旧纹理 → GC 回收 → 创建新纹理
- **更新**: `RHIUpdateTexture2D`（渲染线程异步命令）

### 材质系统

- `MaterialOpaque` — 不透明材质（默认）
- `TransparentMaterial` — 半透明材质
- 通过 `bTransparent` 切换，`PostEditChangeProperty` 触发材质重建

---

## 10. ASuperProjector — 投影仪 / Mapping

**头文件**: `LightActor/SuperProjector.h`  
**基类**: `ASuperBaseActor`  
**导出宏**: `SUPERSTAGE_API`

使用三个 SpotLight 的光照函数（Light Function）实现 Projection Mapping。RGB 三通道分别投影，叠加合成全彩效果。

### 继承链

```
AActor → ASuperBaseActor → ASuperProjector
```

> **注意**: 此类不继承 `ASuperDmxActorBase`，**不使用 DMX 控制**。投影内容由 `MappingTexture` 定义。

### 组件

| 组件 | 类型 | 说明 |
|------|------|------|
| `MappingModel` | `UStaticMeshComponent*` | 投影模型 |
| `SpotLightR` | `USpotLightComponent*` | 红色通道投影光 |
| `SpotLightG` | `USpotLightComponent*` | 绿色通道投影光 |
| `SpotLightB` | `USpotLightComponent*` | 蓝色通道投影光 |

### 属性

| 属性 | 类型 | 默认值 | 范围 | 说明 |
|------|------|--------|------|------|
| `MappingScale` | `FVector2D` | `(1920,1080)` | 128-4096 | 投影分辨率 |
| `MappingTexture` | `UTexture*` | `nullptr` | — | 投影纹理内容 |
| `Dimmer` | `float` | `100.0` | ≥0 | 亮度（%） |
| `MaxLightDistance` | `float` | `3000.0` | ≥100 | 投影距离（cm） |
| `Zoom` | `float` | `30.0` | 10-100 | 投影角度（度） |
| `DilutionFactor` | `float` | `0.0` | 0-1 | 边缘柔化 |
| `MappingDeformation` | `FMappingDeformation` | — | — | 梯形校正 |

### FMappingDeformation — 梯形校正

与 `FDeformation` 结构相同，四角 UV 偏移 `[0, 1]`。

### 投影原理

```
SpotLightR/G/B 分别设置 LightFunctionMaterial
    ↓
光照函数材质包含 MappingTexture
    ↓
梯形校正在材质 Shader 内实现 UV 变换
    ↓
三通道 RGB 叠加 → 全彩投影
```

**SpotLight 参数映射**:
- `Intensity` = `Dimmer`（Candelas 单位）
- `AttenuationRadius` = `MaxLightDistance`
- `OuterConeAngle` = `Zoom`
- `InnerConeAngle` = `Zoom × (1 - DilutionFactor)`
- `LightFunctionScale` = `(MappingScale.X, MappingScale.Y, Diagonal)`

---

## 11. API 速查表

### 按类别分类

#### DMX 控制型 Actor

| Actor | 基类 | DMX 轴数 | 主要用途 |
|-------|------|---------|---------|
| `ASuperDMXCamera` | `ASuperDmxActorBase` | 9 (6轴+FOV+光圈+对焦) | DMX 摄像机 |
| `ASuperLiftingMachinery` | `ASuperDmxActorBase` | 6 (3轴位移+3轴旋转) | 升降/旋转机械 |
| `ASuperRailMachinery` | `ASuperDmxActorBase` | 7 (轨道+3轴偏移+3轴旋转) | 轨道机械 |
| `ASuperStageVFXActor` | `ASuperLightBase` | 3+2 (Pan/Tilt+SpawnCount+RGB) | Niagara 特效 |
| `ASuperLightStripEffect` | `ASuperDmxActorBase` | 9 (亮度+频闪+效果+速度+宽度+方向+RGB) | LED 灯带 |
| `ASuperLiftMatrix` | `ASuperLightBase` | 1+3 (升降+效果+RGB) | 升降矩阵 |

#### 非 DMX 型 Actor

| Actor | 基类 | 数据来源 | 主要用途 |
|-------|------|---------|---------|
| `ASuperLaserActor` | `ASuperBaseActor` | SuperLaserSubsystem (纹理) | Beyond 激光（纹理模式） |
| `ASuperLaserProActor` | `ASuperBaseActor` | SuperLaserSubsystem (点数据) | Beyond 激光（点数据模式） |
| `ASuperNDIScreen` | `ASuperBaseActor` | SuperNDISubsystem (视频流) | NDI 视频屏幕 |
| `ASuperProjector` | `ASuperBaseActor` | UTexture (静态纹理) | 投影仪 Mapping |

### 主控制函数速查

| 函数 | Actor | 说明 |
|------|-------|------|
| `DMXCameraControl` | `ASuperDMXCamera` | 6轴摄像机运动 |
| `DMXCameraParams` | `ASuperDMXCamera` | FOV/光圈/对焦 |
| `LiftingMachinery` | `ASuperLiftingMachinery` | 6轴升降机械 |
| `RailMachinery` | `ASuperRailMachinery` | 7轴轨道机械 |
| `RailPositionOnly` | `ASuperRailMachinery` | 仅轨道位置 |
| `SetNiagaraSpawnCount` | `ASuperStageVFXActor` | 粒子生成量 |
| `SetNiagaraColor` | `ASuperStageVFXActor` | 粒子颜色 |
| `SetLightStripEffect` | `ASuperLightStripEffect` | 灯带全通道控制 |
| `LiftMatrix` | `ASuperLiftMatrix` | 升降矩阵 |
| `SetLaserEnabled` | `ASuperLaserProActor` | 激光开关 |

### 初始化函数速查

| 函数 | Actor | 调用时机 |
|------|-------|---------|
| `BootRefresh` | Camera/Machinery/Rail | 编辑器按钮，放置后调用 |
| `SetNiagaraDefaultValue` | VFXActor | SuperDMXTick 首帧 |
| `SetLightStripDefault` | LightStrip | SuperDMXTick 首帧 |
| `SetEffectMatrixDefaultValue` | LiftMatrix | SuperDMXTick 首帧 |

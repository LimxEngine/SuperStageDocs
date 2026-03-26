# ASuperBaseActor & ASuperDmxActorBase API 文档

> **文档版本**: 1.0  
> **适用版本**: SuperStage Plugin 2025/2026  
> **目标读者**: 蓝图开发者 / C++ 插件集成开发者（无源码修改权限）  
> **最后更新**: 2026-03

---

## 目录

1. [ASuperBaseActor — 基础资产 Actor](#1-asuperbaseactor--基础资产-actor)
2. [FAssetMetaData — 资产元数据](#2-fassetmetadata--资产元数据)
3. [FSuperDMXFixture — DMX 地址配置](#3-fsuperdmxfixture--dmx-地址配置)
4. [ASuperDmxActorBase — DMX 数据读取基类](#4-asuperdmxactorbase--dmx-数据读取基类)
   - [属性](#41-属性)
   - [蓝图事件](#42-蓝图事件)
   - [控制函数](#43-控制函数)
   - [属性值读取（归一化）](#44-属性值读取归一化)
   - [原始值读取（按实例索引）](#45-原始值读取按实例索引)
   - [矩阵批量读取](#46-矩阵批量读取)
   - [地址解析与信息查询](#47-地址解析与信息查询)
5. [DMX 地址计算详解](#5-dmx-地址计算详解)
6. [蓝图宏工具](#6-蓝图宏工具)
7. [使用示例](#7-使用示例)
8. [API 速查表](#8-api-速查表)

---

## 1. ASuperBaseActor — 基础资产 Actor

**头文件**: `SuperBaseActor.h`  
**基类**: `AActor`  
**导出宏**: `SUPERSTAGE_API`

所有 SuperStage Actor 的**根基类**，提供资产元数据管理和编辑器方向预览功能。

### 属性

| 属性 | 类型 | 类别 | 说明 |
|------|------|------|------|
| `AssetData` | `FAssetMetaData` | `A.Asset` | 资产元数据（UUID、制造商、DMX 模式等） |
| `SceneBase` | `USceneComponent*` | `Z.Components` | 根场景组件，所有子组件的挂载点 |
| `bPreviewAssetOrientation` | `bool` | `F.Preview` | 方向预览开关（显示前向/上方箭头） |
| `ForwardArrow` | `UArrowComponent*` | — | 前向箭头（绿色，+X 方向），`Transient` |
| `UpArrow` | `UArrowComponent*` | — | 向上箭头（蓝色，+Z 方向），`Transient` |

### 组件结构

```
ASuperBaseActor
 └─ SceneBase (USceneComponent, 根组件)
     ├─ ForwardArrow (UArrowComponent, +X, 绿色)
     └─ UpArrow (UArrowComponent, +Z, 蓝色)
```

> **注意**: `ForwardArrow` 和 `UpArrow` 标记为 `Transient`，不序列化到磁盘。仅在 `bPreviewAssetOrientation = true` 时可见。

---

## 2. FAssetMetaData — 资产元数据

**头文件**: `SuperBaseActor.h`

存储 SuperStage 资产的描述性信息，用于资产管理和识别。

```cpp
USTRUCT(BlueprintType)
struct FAssetMetaData
```

### 属性

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `UUID` | `FGuid` | `FGuid()` (零 GUID) | 资产唯一标识符 |
| `Group` | `FString` | `""` | 分组/类别（如 "Beam", "Wash", "Spot"） |
| `Manufacturer` | `FString` | `""` | 制造商名称（如 "Acme", "Robe"） |
| `DMXMode` | `FString` | `""` | DMX 模式名称（如 "17CH", "Extended"） |
| `DisplayName` | `FString` | `""` | 显示名称 |
| `Thumbnail` | `UTexture2D*` | `nullptr` | 缩略图纹理 |

> **设计说明**: `UUID` 默认为零 GUID（而非 `NewGuid()`），避免每次序列化/反序列化时产生不必要的变更（序列化抖动）。

---

## 3. FSuperDMXFixture — DMX 地址配置

**头文件**: `LightActor/SuperLightTypes.h`

定义灯具在 DMX 网络中的地址信息。每个 DMX 控制的 Actor 都持有一个此结构体实例。

```cpp
USTRUCT(BlueprintType)
struct FSuperDMXFixture
```

### 属性

| 属性 | 类型 | 默认值 | 范围 | 说明 |
|------|------|--------|------|------|
| `FixtureID` | `int32` | `1` | ≥1 | 灯具编号（用于标识和地址标签显示） |
| `Universe` | `int32` | `1` | 1-256 | DMX Universe 编号 |
| `StartAddress` | `int32` | `1` | 1-512 | 起始 DMX 通道地址 |

### 使用说明

- `Universe` 对应 `USuperDMXSubsystem` 的**内部 Universe 编号**（受 `StartUniverse` 偏移影响）
- `StartAddress` 是该灯具占用的第一个 DMX 通道
- 灯具占用的通道范围 = `[StartAddress, StartAddress + ChannelSpan - 1]`
- `FixtureID` 不影响 DMX 通信，仅用于标识和显示

---

## 4. ASuperDmxActorBase — DMX 数据读取基类

**头文件**: `SuperDmxActorBase.h`  
**基类**: `ASuperBaseActor`  
**导出宏**: `SUPERSTAGE_API`

所有 DMX 控制 Actor 的**核心基类**，提供从 `USuperDMXSubsystem` 读取 DMX 数据并转换为可用值的完整接口。

### 继承体系

```
AActor
 └─ ASuperBaseActor           (元数据 / 方向预览)
     └─ ASuperDmxActorBase    (DMX 读取核心) ← 本类
         ├─ ASuperLightBase   (Pan/Tilt 运动控制)
         │   ├─ ASuperStageLight     (全功能电脑灯)
         │   ├─ ASuperStageVFXActor  (Niagara VFX)
         │   └─ ASuperLiftMatrix     (升降矩阵)
         ├─ ASuperLiftingMachinery   (升降机械)
         ├─ ASuperRailMachinery      (轨道机械)
         ├─ ASuperLightStripEffect   (LED 灯带)
         └─ ASuperDMXCamera          (DMX 摄像机)
```

---

### 4.1 属性

| 属性 | 类型 | 类别 | 访问 | 说明 |
|------|------|------|------|------|
| `SuperDMXFixture` | `FSuperDMXFixture` | `A.SuperDMX` | ReadWrite + Interp | DMX 地址配置（Universe + StartAddress + FixtureID） |
| `FixtureLibrary` | `USuperFixtureLibrary*` | `A.SuperDMX` | ReadOnly | 灯库数据资产引用（定义通道布局） |
| `bShowAddressLabel` | `bool` | `F.Preview` | ReadWrite | 是否显示地址码文本标签 |
| `bIncludeFixtureIdInLabel` | `bool` | `F.Preview` | ReadWrite | 标签是否包含 FixtureID |
| `AddressLabelOffset` | `FVector` | `F.Preview` | ReadWrite | 标签偏移量（默认 `(30,0,0)` cm） |

#### SuperDMXFixture

灯具的 DMX 地址配置。支持 `Interp` 元标记，可在 Sequencer 中进行动画（动态切换 Universe/地址）。

**蓝图中设置**：在 Actor 的 Details 面板 → `A.SuperDMX` → `SuperDMXFixture` 中配置 Universe 和 StartAddress。

#### FixtureLibrary

引用一个 `USuperFixtureLibrary` 数据资产，定义该灯具的所有 DMX 通道映射。**必须设置**，否则属性读取函数将无法工作。

**蓝图中设置**：在 Actor 的 Details 面板 → `A.SuperDMX` → `灯库` 中选择数据资产。

---

### 4.2 蓝图事件

#### SuperDMXTick

```cpp
UFUNCTION(BlueprintImplementableEvent, Category="A.Event", meta=(DisplayName="Super DMX Tick"))
void SuperDMXTick(float DeltaSeconds = 0.f);
```

**每帧触发**的蓝图可实现事件。这是用户编写 DMX 控制逻辑的**主要入口点**。

**参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `DeltaSeconds` | `float` | 帧间隔时间（秒）。`ForceRefreshDMX()` 调用时为 `0.0` |

**触发时机**：
- 正常 `Tick()` 每帧调用
- `ForceRefreshDMX()` 手动触发
- `OnConstruction()` 构建时触发（`DeltaSeconds = 0`）
- 编辑器视口暂停时**仍然触发**（`ShouldTickIfViewportsOnly() = true`）

**典型用法**（蓝图）：

```
Event SuperDMXTick(DeltaSeconds)
 ├─ SetLightingDefaultValue(...)     // 初始化（可选，仅首帧）
 ├─ SetLightingIntensity(Dimmer)     // 亮度
 ├─ SetLightingStrobe(Strobe)        // 频闪
 ├─ SetLightingColorRGB(R, G, B)     // 颜色
 ├─ YPan(Pan)                        // Pan 旋转
 ├─ YTilt(Tilt)                      // Tilt 旋转
 ├─ SetLightingZoom(Zoom)            // 变焦
 └─ ...
```

> **重要**: 编辑器中通过 `FEditorScriptExecutionGuard` 保护蓝图执行安全。

#### LightInitialization

```cpp
UPROPERTY(BlueprintAssignable, Category = "A.Event", meta = (DisplayName = "LightInitialization"))
FLightInitialization LightInitialization;
```

灯具初始化委托（`DECLARE_DYNAMIC_MULTICAST_DELEGATE`），可在蓝图中绑定回调。

---

### 4.3 控制函数

#### ForceRefreshDMX

```cpp
UFUNCTION(BlueprintCallable, Category = "A.Yunsio|SuperDMX|Control")
void ForceRefreshDMX();
```

强制刷新 DMX 状态。

**行为**：
1. 调用 `SuperDMXTick(0.0f)` — 触发蓝图控制逻辑
2. 调用 `MarkComponentsRenderStateDirty()` — 标记渲染状态需要更新
3. 调用 `PostEditChange()` — 触发材质参数等属性变化通知

**使用场景**：
- 编辑器中实时预览 DMX 控制效果
- `SuperConsoleSubsystem` 批量刷新灯具
- 手动触发单次 DMX 更新（不等待下一帧）

> **性能说明**: 此函数**不主动重绘视口**，由 `SuperConsoleSubsystem` 统一批量重绘，避免 N 个灯具 N 次重绘。

---

### 4.4 属性值读取（归一化）

这组函数使用 `FSuperDMXAttribute` 结构体查询属性，**自动根据通道精度归一化**到 `[0, 1]` 范围。

#### FSuperDMXAttribute 查询键

```cpp
USTRUCT(BlueprintType)
struct FSuperDMXAttribute
{
    int32 InstanceIndex;         // 模块实例索引（0 = 主模块）
    FName AttribName;            // 属性名称（必须与灯库定义一致）
    EDMXChannelType DMXChannelType; // 通道精度
};
```

| EDMXChannelType | 精度 | 值范围 | 归一化除数 |
|-----------------|------|--------|-----------|
| `Conarse` | 8-bit | 0-255 | ÷255 |
| `Fine` | 16-bit | 0-65535 | ÷65535 |
| `Ultra` | 24-bit | 0-16777215 | ÷16777215 |

---

#### GetSuperDmxAttributeValue

```cpp
UFUNCTION(BlueprintPure, Category = "A.Yunsio|SuperDMX|Read",
    meta = (DisplayName = "GetSuperDmxAttributeValue"))
void GetSuperDmxAttributeValue(
    const FSuperDMXAttribute& DmxAttribute,
    float& InOutDefault) const;
```

读取 DMX 属性值并**归一化**到 `[0, 1]`。

**参数**：

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `DmxAttribute` | `FSuperDMXAttribute` | 输入 | 属性查询键 |
| `InOutDefault` | `float&` | 输入/输出 | 输入为默认值；输出为归一化后的值 |

**归一化规则**：

| DMXChannelType | 公式 |
|----------------|------|
| `Conarse` | `Coarse / 255.0` |
| `Fine` | `(Coarse × 256 + Fine) / 65535.0` |
| `Ultra` | `(Coarse × 65536 + Fine × 256 + Ultra) / 16777215.0` |

**行为**：
- 读取失败（属性不存在、Universe 无数据）时 `InOutDefault` **保持不变**
- 内部使用 Universe 快照（`SnapshotUniverse`），单次批量读取整帧数据
- 包含权限检查（`SUPER_REQUIRE_PERMISSION_VOID`）

**示例**（蓝图伪代码）：

```
DmxAttr.InstanceIndex = 0
DmxAttr.AttribName = "Dimmer"
DmxAttr.DMXChannelType = Conarse

DefaultValue = 0.0
GetSuperDmxAttributeValue(DmxAttr, DefaultValue)
// DefaultValue 现在是 0.0 ~ 1.0 的亮度值
```

---

#### GetSuperDmxAttributeValueNoConversion

```cpp
UFUNCTION(BlueprintPure, Category = "A.Yunsio|SuperDMX|Read",
    meta = (DisplayName = "GetSuperDmxAttributeValueNoConversion"))
void GetSuperDmxAttributeValueNoConversion(
    const FSuperDMXAttribute& DmxAttribute,
    float& InOutDefault) const;
```

读取 DMX 属性的**原始整数值**（不归一化）。

**返回值范围**：

| DMXChannelType | 范围 |
|----------------|------|
| `Conarse` | 0 - 255 |
| `Fine` | 0 - 65535 |
| `Ultra` | 0 - 16777215 |

**使用场景**：
- 需要原始 DMX 值的场景（如子属性查找、ChannelSet 匹配）
- 自定义归一化逻辑

---

#### GetSuperDMXColorValue

```cpp
UFUNCTION(BlueprintPure, Category = "A.Yunsio|SuperDMX|Read",
    meta = (DisplayName = "GetSuperDMXColorValue"))
void GetSuperDMXColorValue(
    const FSuperDMXAttribute& DmxRed,
    const FSuperDMXAttribute& DmxGreen,
    const FSuperDMXAttribute& DmxBlue,
    FLinearColor& OutColor) const;
```

读取 RGB 三个 DMX 属性并组合为 `FLinearColor`。

**参数**：

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `DmxRed` | `FSuperDMXAttribute` | 输入 | 红色通道属性 |
| `DmxGreen` | `FSuperDMXAttribute` | 输入 | 绿色通道属性 |
| `DmxBlue` | `FSuperDMXAttribute` | 输入 | 蓝色通道属性 |
| `OutColor` | `FLinearColor&` | 输出 | 合成颜色（Alpha 固定为 1.0） |

**行为**：
- 分别调用 `GetSuperDmxAttributeValue()` 读取 R/G/B
- 将结果填入 `OutColor.R/G/B`，`OutColor.A = 1.0`
- 各通道独立读取，可使用不同的 `DMXChannelType`

---

#### GetSuperDmxAttributeRawValue

```cpp
UFUNCTION(BlueprintPure, Category = "A.Yunsio|SuperDMX|Read",
    meta = (DisplayName = "GetSuperDmxAttributeRawValue"))
void GetSuperDmxAttributeRawValue(
    const FSuperDMXAttribute& DmxAttribute,
    int32& OutRawValue) const;
```

获取属性的**原始 8-bit 值**（仅读取 Coarse 字节）。

**参数**：

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `DmxAttribute` | `FSuperDMXAttribute` | 输入 | 属性查询键（`DMXChannelType` 被忽略） |
| `OutRawValue` | `int32&` | 输出 | 原始值 0-255，读取失败时为 0 |

> **注意**: 此函数始终只读取 Coarse 字节（8-bit），不受 `DMXChannelType` 影响。如需 16/24-bit 原始值，使用 `GetSuperDmxAttributeValueNoConversion()`。

---

### 4.5 原始值读取（按实例索引）

这组函数直接使用 `InstanceIndex` + `AttribName` 定位属性，返回**原始整数值**。

#### GetAttributeRaw8ByIndex

```cpp
UFUNCTION(BlueprintPure, Category="A.Yunsio|SuperDMX|Read8")
int32 GetAttributeRaw8ByIndex(
    int32 InstanceIndex,
    FName AttribName,
    int32 DefaultValue = 0) const;
```

读取 **8-bit** 原始 DMX 值（仅 Coarse 字节）。

**参数**：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `InstanceIndex` | `int32` | — | 模块实例索引（0 = 主模块） |
| `AttribName` | `FName` | — | 属性名称（如 `"Dimmer"`, `"Pan"`, `"Tilt"`） |
| `DefaultValue` | `int32` | `0` | 读取失败时的返回值 |

**返回值**: `int32`，范围 `[0, 255]`

---

#### GetAttributeRaw16ByIndex

```cpp
UFUNCTION(BlueprintPure, Category="A.Yunsio|SuperDMX|Read16")
int32 GetAttributeRaw16ByIndex(
    int32 InstanceIndex,
    FName AttribName,
    int32 DefaultValue = 0) const;
```

读取 **16-bit** 原始 DMX 值（Coarse 为高 8 位，Fine 为低 8 位）。

**返回值**: `int32`，范围 `[0, 65535]`

**计算公式**: `(Coarse << 8) | Fine`

> **注意**: 需要灯库中该属性同时定义了 Coarse 和 Fine 通道偏移，否则返回 `DefaultValue`。

---

#### GetAttributeRaw24ByIndex

```cpp
UFUNCTION(BlueprintPure, Category="A.Yunsio|SuperDMX|Read24")
int32 GetAttributeRaw24ByIndex(
    int32 InstanceIndex,
    FName AttribName,
    int32 DefaultValue = 0) const;
```

读取 **24-bit** 原始 DMX 值。

**返回值**: `int32`，范围 `[0, 16777215]`

**计算公式**: `(Coarse << 16) | (Fine << 8) | Ultra`

**容错规则**：
- 仅需 Coarse 存在即可工作
- 缺少 Fine → Fine 视为 0
- 缺少 Ultra → Ultra 视为 0

---

#### GetAttributeBitDepthByIndex

```cpp
UFUNCTION(BlueprintPure, Category="A.Yunsio|SuperDMX|Info")
int32 GetAttributeBitDepthByIndex(
    int32 InstanceIndex,
    FName AttribName) const;
```

查询属性的有效位深度。

**返回值**：

| 值 | 说明 |
|----|------|
| `0` | 属性不存在 |
| `8` | 仅有 Coarse |
| `16` | Coarse + Fine |
| `24` | Coarse + Fine + Ultra |

---

### 4.6 矩阵批量读取

矩阵读取函数遍历灯库中**所有模块实例**，读取同名属性的值，返回排序后的数组。适用于 LED 矩阵灯、像素条等多模块设备。

#### GetMatrixAttributeRaw

```cpp
UFUNCTION(BlueprintPure, Category="A.Yunsio|SuperDMX|Matrix")
void GetMatrixAttributeRaw(
    FName AttribName,
    TArray<float>& OutValues,
    float DefaultValue = 0,
    bool bSortByPatch = true) const;
```

按 **8-bit** 精度读取所有模块实例的同名属性。

**参数**：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `AttribName` | `FName` | — | 属性名称 |
| `OutValues` | `TArray<float>&` | — | 输出数组，每个元素为 **[0, 1]** 归一化值 |
| `DefaultValue` | `float` | `0` | 无数据时的默认值 |
| `bSortByPatch` | `bool` | `true` | 是否按 Patch 偏移排序 |

**行为**：
1. 遍历 `FixtureLibrary->Modules` 所有模块实例
2. 查找包含 `AttribName` 的模块
3. 读取 Coarse 字节，归一化到 `[0, 1]`（`coarse / 255.0`）
4. 按 `Patch` 升序排序（如 `bSortByPatch = true`）

**返回值说明**：
- `OutValues[0]` 对应 Patch 最小的模块（通常是像素 1）
- 数组长度 = 包含该属性的模块数量
- 跳过不包含该属性的模块

---

#### GetMatrixAttributeRaw16

```cpp
UFUNCTION(BlueprintPure, Category="A.Yunsio|SuperDMX|Matrix")
void GetMatrixAttributeRaw16(
    FName AttribName,
    TArray<float>& OutValues,
    float DefaultValue = 0,
    bool bSortByPatch = true) const;
```

按 **16-bit** 精度读取所有模块实例的同名属性。

**归一化公式**: `((Coarse << 8) | Fine) / 65535.0`

> **注意**: 需要每个模块中该属性同时定义 Coarse 和 Fine，缺少 Fine 的模块将被跳过。

---

#### GetMatrixAttributeRawWithIndex / GetMatrixAttributeRaw16WithIndex

```cpp
void GetMatrixAttributeRawWithIndex(
    FName AttribName,
    TArray<float>& OutValues,
    TArray<int32>& OutModuleIndices,
    float DefaultValue = 0) const;

void GetMatrixAttributeRaw16WithIndex(
    FName AttribName,
    TArray<float>& OutValues,
    TArray<int32>& OutModuleIndices,
    float DefaultValue = 0) const;
```

与上述矩阵函数相同，但额外输出**模块索引数组**。

**额外输出**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `OutModuleIndices` | `TArray<int32>&` | 对应 `OutValues` 每个元素的原始模块索引 |

**使用场景**：
- 需要通过模块索引进一步查询子属性（`FindAttributeDef`）
- 被 `GET_SUPER_DMX_MATRIX_VALUE_WITH_SUBATTR` 宏内部使用

> **注意**: 这两个函数**非 BlueprintCallable**（无 UFUNCTION 宏），仅限 C++ 调用。

---

### 4.7 地址解析与信息查询

#### GetChannelValue

```cpp
UFUNCTION(BlueprintPure, Category="A.Yunsio|SuperDMX|Read8")
float GetChannelValue(int32 Address = 1, float DefaultValue = 1) const;
```

按**相对通道地址**读取单个 DMX 字节值。

**参数**：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `Address` | `int32` | `1` | 相对于 `StartAddress` 的通道偏移（**1 基**） |
| `DefaultValue` | `float` | `1` | 读取失败时的返回值 |

**返回值**: `float`，范围 `[0, 255]`（**注意：不是归一化值**）

**绝对地址计算**: `StartAddress + Address - 1`

**示例**：

```
// StartAddress = 100
GetChannelValue(1)  → 读取通道 100（Coarse 值 0-255）
GetChannelValue(2)  → 读取通道 101
GetChannelValue(5)  → 读取通道 104
```

---

#### ResolveAttributeAddressesByIndex

```cpp
UFUNCTION(BlueprintPure, Category="A.Yunsio|SuperDMX|Address")
bool ResolveAttributeAddressesByIndex(
    int32 InstanceIndex,
    FName AttribName,
    int32& OutCoarseAbs,
    int32& OutFineAbs,
    int32& OutUltraAbs) const;
```

解析属性在 Universe 中的**绝对通道地址**。

**参数**：

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `InstanceIndex` | `int32` | 输入 | 模块实例索引 |
| `AttribName` | `FName` | 输入 | 属性名称 |
| `OutCoarseAbs` | `int32&` | 输出 | Coarse 绝对地址（1 基），无效时为 0 |
| `OutFineAbs` | `int32&` | 输出 | Fine 绝对地址，不存在时为 0 |
| `OutUltraAbs` | `int32&` | 输出 | Ultra 绝对地址，不存在时为 0 |

**返回值**: `true` 表示找到属性且至少有 Coarse 地址

**地址计算公式**：

```
Base = StartAddress + Max(0, Patch - 1)
CoarseAbs = Base + Coarse - 1    (如果 Coarse > 0)
FineAbs   = Base + Fine - 1      (如果 Fine > 0)
UltraAbs  = Base + Ultra - 1     (如果 Ultra > 0)
```

---

#### FindAttributeDef

```cpp
// 按实例索引查找
const FSuperDMXAttributeDef* FindAttributeDef(int32 InstanceIndex, FName AttribName) const;

// 按 FSuperDMXAttribute 查找
const FSuperDMXAttributeDef* FindAttributeDef(const FSuperDMXAttribute& DmxAttribute) const;
```

查找属性定义，返回指向 `FSuperDMXAttributeDef` 的指针。

**返回值**: 找到时返回指针，未找到时返回 `nullptr`

**使用场景**：
- 访问子属性系统（`SubAttributes`）
- 获取 ChannelSet 信息
- 自定义属性解析逻辑

> **注意**: 这两个重载**非 BlueprintCallable**，仅限 C++ 调用。

---

#### GetModules

```cpp
UFUNCTION(BlueprintPure, Category="A.Yunsio|SuperDMX|Info",
    meta=(DisplayName="GetModules"))
const TArray<FSuperDMXModuleInstance>& GetModules() const;
```

获取灯库中的模块实例数组。

**返回值**: `FixtureLibrary->Modules` 的引用；`FixtureLibrary` 为空时返回静态空数组

---

#### GetFixtureChannelSpan

```cpp
UFUNCTION(BlueprintPure, Category="A.Yunsio|SuperDMX|Info",
    meta=(DisplayName="GetFixtureChannelSpan"))
int32 GetFixtureChannelSpan() const;
```

计算灯具占用的总通道跨度。

**返回值**: `最大绝对地址 - 最小绝对地址 + 1`；无有效通道时返回 `0`

**计算方式**：
- 遍历所有模块实例的所有属性定义
- 收集所有 >0 的 Coarse/Fine/Ultra 绝对地址
- 跨度 = Max - Min + 1

---

## 5. DMX 地址计算详解

### 基本概念

```
Universe  ──────────────────────────────── 512 通道
  │
  └─ StartAddress = 100 ─────── 灯具起始位置
       │
       ├─ Module[0] (Main), Patch = 0
       │   ├─ Dimmer:  Coarse=1  → 绝对地址 100
       │   ├─ Pan:     Coarse=3, Fine=4  → 绝对地址 102, 103
       │   ├─ Tilt:    Coarse=5, Fine=6  → 绝对地址 104, 105
       │   └─ ...
       │
       ├─ Module[1] (Pixel1), Patch = 17
       │   ├─ Dimmer:  Coarse=1  → 绝对地址 116
       │   ├─ Red:     Coarse=2  → 绝对地址 117
       │   └─ ...
       │
       └─ Module[2] (Pixel2), Patch = 20
           ├─ Dimmer:  Coarse=1  → 绝对地址 119
           └─ ...
```

### 地址公式

```
Base = StartAddress + Max(0, Patch - 1)
AbsoluteAddress = Base + RelativeOffset - 1
```

其中 `RelativeOffset` 是 `FSuperDMXAttributeDef` 中的 `Coarse`/`Fine`/`Ultra` 值。

### 示例计算

| 参数 | 值 | 说明 |
|------|-----|------|
| `StartAddress` | 100 | 灯具起始 |
| `Patch` | 17 | Pixel1 模块偏移 |
| `Coarse` | 2 | Red 属性的 Coarse 偏移 |
| **Base** | 100 + 16 = 116 | |
| **绝对地址** | 116 + 2 - 1 = **117** | Red 通道在 Universe 中的地址 |

---

## 6. 蓝图宏工具

以下 C++ 宏简化了矩阵属性的批量读取，供 C++ 插件开发者使用（蓝图用户应使用对应的 BlueprintCallable 函数）。

### GET_SUPER_DMX_MATRIX_VALUE

```cpp
GET_SUPER_DMX_MATRIX_VALUE(DMXAttribute, DefaultValue, LoopBody)
```

遍历矩阵属性值，自动根据 `DMXChannelType` 选择 8-bit 或 16-bit 读取。

**循环变量**：
- `LightIndex` (`int32`): 当前像素索引
- `DefaultValue`: 当前像素的归一化值（被就地更新）

### GET_SUPER_DMX_MATRIX_VALUE_WITH_SUBATTR

```cpp
GET_SUPER_DMX_MATRIX_VALUE_WITH_SUBATTR(DMXAttribute, DefaultValue, LoopBody)
```

增强版，额外提供子属性信息。

**循环变量**：
- `LightIndex` (`int32`): 当前像素索引
- `DefaultValue`: 当前像素的归一化值
- `DmxRawValue` (`int32`): 原始 DMX 值（0-255）
- `AttrDef` (`const FSuperDMXAttributeDef*`): 属性定义指针
- `SubAttr` (`const FSubAttribute*`): 当前值对应的子属性（可能为 `nullptr`）

### GET_SUPER_DMX_MATRIX_RGB

```cpp
GET_SUPER_DMX_MATRIX_RGB(DmxR, DmxG, DmxB, OutColor, LoopBody)
```

遍历矩阵 RGB 颜色。

**循环变量**：
- `LightIndex` (`int32`): 当前像素索引
- `OutColor` (`FLinearColor`): 当前像素的颜色

---

## 7. 使用示例

### 7.1 C++ 插件：读取灯具亮度

```cpp
#include "SuperDmxActorBase.h"

void AMyController::ReadFixtureDimmer(ASuperDmxActorBase* Light)
{
    if (!Light) return;

    // 方式1：通过 FSuperDMXAttribute 归一化读取
    FSuperDMXAttribute DimmerAttr;
    DimmerAttr.InstanceIndex = 0;
    DimmerAttr.AttribName = FName("Dimmer");
    DimmerAttr.DMXChannelType = EDMXChannelType::Conarse;

    float Dimmer = 0.0f;
    Light->GetSuperDmxAttributeValue(DimmerAttr, Dimmer);
    // Dimmer = 0.0 ~ 1.0

    // 方式2：通过实例索引原始值读取
    int32 RawDimmer = Light->GetAttributeRaw8ByIndex(0, FName("Dimmer"));
    // RawDimmer = 0 ~ 255

    // 方式3：通过相对通道读取
    float ChannelValue = Light->GetChannelValue(1);
    // ChannelValue = 0 ~ 255（通道 StartAddress + 0）
}
```

### 7.2 C++ 插件：读取矩阵像素颜色

```cpp
void AMyController::ReadMatrixPixels(ASuperDmxActorBase* Light)
{
    if (!Light) return;

    TArray<float> RedValues, GreenValues, BlueValues;
    Light->GetMatrixAttributeRaw(FName("Red"), RedValues);
    Light->GetMatrixAttributeRaw(FName("Green"), GreenValues);
    Light->GetMatrixAttributeRaw(FName("Blue"), BlueValues);

    int32 PixelCount = FMath::Min3(RedValues.Num(), GreenValues.Num(), BlueValues.Num());
    for (int32 i = 0; i < PixelCount; ++i)
    {
        FLinearColor PixelColor(RedValues[i], GreenValues[i], BlueValues[i], 1.0f);
        // 使用 PixelColor...
    }
}
```

### 7.3 C++ 插件：查询灯具通道信息

```cpp
void AMyController::InspectFixture(ASuperDmxActorBase* Light)
{
    if (!Light) return;

    // 获取通道跨度
    int32 Span = Light->GetFixtureChannelSpan();
    UE_LOG(LogTemp, Log, TEXT("Fixture uses %d channels"), Span);

    // 获取模块列表
    const TArray<FSuperDMXModuleInstance>& Modules = Light->GetModules();
    UE_LOG(LogTemp, Log, TEXT("Fixture has %d modules"), Modules.Num());

    // 查询属性位深
    int32 PanBitDepth = Light->GetAttributeBitDepthByIndex(0, FName("Pan"));
    UE_LOG(LogTemp, Log, TEXT("Pan bit depth: %d"), PanBitDepth); // 8 或 16

    // 解析绝对地址
    int32 CoarseAddr, FineAddr, UltraAddr;
    if (Light->ResolveAttributeAddressesByIndex(0, FName("Pan"), CoarseAddr, FineAddr, UltraAddr))
    {
        UE_LOG(LogTemp, Log, TEXT("Pan Coarse=Ch%d, Fine=Ch%d"), CoarseAddr, FineAddr);
    }
}
```

---

## 8. API 速查表

### ASuperBaseActor

| 属性 | 类型 | 说明 |
|------|------|------|
| `AssetData` | `FAssetMetaData` | 资产元数据 |
| `SceneBase` | `USceneComponent*` | 根组件 |
| `bPreviewAssetOrientation` | `bool` | 方向预览开关 |

### ASuperDmxActorBase — 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `SuperDMXFixture` | `FSuperDMXFixture` | DMX 地址（Universe + StartAddress） |
| `FixtureLibrary` | `USuperFixtureLibrary*` | 灯库引用 |
| `bShowAddressLabel` | `bool` | 显示地址标签 |

### ASuperDmxActorBase — 归一化读取

| 函数 | 返回 | 说明 |
|------|------|------|
| `GetSuperDmxAttributeValue(Attr, &Val)` | `void` | 归一化到 [0,1] |
| `GetSuperDmxAttributeValueNoConversion(Attr, &Val)` | `void` | 原始整数值 |
| `GetSuperDMXColorValue(R, G, B, &Color)` | `void` | RGB → FLinearColor |
| `GetSuperDmxAttributeRawValue(Attr, &Raw)` | `void` | 仅 Coarse (0-255) |

### ASuperDmxActorBase — 按索引原始读取

| 函数 | 返回范围 | 说明 |
|------|----------|------|
| `GetAttributeRaw8ByIndex(Idx, Name, Def)` | 0-255 | 8-bit |
| `GetAttributeRaw16ByIndex(Idx, Name, Def)` | 0-65535 | 16-bit |
| `GetAttributeRaw24ByIndex(Idx, Name, Def)` | 0-16777215 | 24-bit |
| `GetAttributeBitDepthByIndex(Idx, Name)` | 0/8/16/24 | 位深查询 |

### ASuperDmxActorBase — 矩阵读取

| 函数 | 精度 | 输出 |
|------|------|------|
| `GetMatrixAttributeRaw(Name, &Vals, Def, Sort)` | 8-bit | `TArray<float>` [0,1] |
| `GetMatrixAttributeRaw16(Name, &Vals, Def, Sort)` | 16-bit | `TArray<float>` [0,1] |

### ASuperDmxActorBase — 工具

| 函数 | 返回 | 说明 |
|------|------|------|
| `ForceRefreshDMX()` | `void` | 强制刷新 DMX |
| `GetChannelValue(Addr, Def)` | `float` 0-255 | 相对通道读取 |
| `ResolveAttributeAddressesByIndex(...)` | `bool` | 地址解析 |
| `GetModules()` | `TArray<FSuperDMXModuleInstance>&` | 模块列表 |
| `GetFixtureChannelSpan()` | `int32` | 通道跨度 |

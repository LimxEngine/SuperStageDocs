# SuperDMX 模块 API 文档

> **文档版本**: 1.0  
> **适用版本**: SuperStage Plugin 2025/2026  
> **目标读者**: 蓝图开发者 / C++ 插件集成开发者（无源码修改权限）  
> **最后更新**: 2026-03

---

## 目录

1. [模块概述](#1-模块概述)
2. [数据类型](#2-数据类型)
   - [ESuperDMXProtocol](#21-esuperdmxprotocol)
   - [FSuperDMXEndpoint](#22-fsuperdmxendpoint)
   - [FSuperDMXIOConfig](#23-fsuperdmxioconfig)
3. [USuperDMXSubsystem — DMX 通信子系统](#3-usuperdmxsubsystem--dmx-通信子系统)
   - [获取实例](#31-获取实例)
   - [生命周期](#32-生命周期)
   - [配置管理](#33-配置管理)
   - [数据接收与查询](#34-数据接收与查询)
   - [数据发送](#35-数据发送)
   - [诊断与监控](#36-诊断与监控)
   - [sACN 多播管理](#37-sacn-多播管理)
4. [Sequencer 集成](#4-sequencer-集成)
   - [UMovieSceneSuperDMXTrack](#41-umoviescenesuper dmxtrack)
   - [UMovieSceneSuperDMXSection](#42-umoviescenesuperdmxsection)
5. [线程安全模型](#5-线程安全模型)
6. [网络协议细节](#6-网络协议细节)
7. [Universe 编号与 StartUniverse 映射](#7-universe-编号与-startuniverse-映射)
8. [使用示例](#8-使用示例)
9. [常见问题与排错](#9-常见问题与排错)

---

## 1. 模块概述

**SuperDMX** 是 SuperStage 插件体系的底层通信模块，负责 **DMX over IP** 数据的收发与缓存。

| 项目 | 说明 |
|------|------|
| **模块类型** | Runtime |
| **导出宏** | `SUPERDMX_API` |
| **模块入口** | `FSuperDMXModule`（`SuperDMX.h`） |
| **核心子系统** | `USuperDMXSubsystem`（`UEngineSubsystem` 全局单例） |
| **协议支持** | Art-Net（UDP 6454）、sACN/E1.31（UDP 5568，代码预留） |
| **头文件目录** | `Source/SuperDMX/Public/` |

### 模块依赖

在 `.Build.cs` 中添加：

```csharp
PublicDependencyModuleNames.Add("SuperDMX");
```

### 公开头文件清单

| 头文件 | 内容 |
|--------|------|
| `SuperDMX.h` | 模块接口（`FSuperDMXModule`） |
| `SuperDMXTypes.h` | 数据类型（协议枚举、端点、I/O 配置） |
| `SuperDMXSubsystem.h` | DMX 子系统（收发/缓存/查询） |
| `Sequencer/MovieSceneSuperDMXTrack.h` | Sequencer DMX 轨道 |
| `Sequencer/MovieSceneSuperDMXSection.h` | Sequencer DMX 区段 |

---

## 2. 数据类型

### 2.1 ESuperDMXProtocol

**头文件**: `SuperDMXTypes.h`

DMX 网络协议枚举。

```cpp
UENUM(BlueprintType)
enum class ESuperDMXProtocol : uint8
{
    ArtNet  UMETA(DisplayName = "Art-Net"),
};
```

| 值 | 说明 | 默认端口 |
|----|------|----------|
| `ArtNet` | Art-Net 协议（OpDmx 0x5000） | 6454 |

> **备注**: sACN/E1.31 协议已在代码中预留（端口 5568），当前版本仅支持 Art-Net。

---

### 2.2 FSuperDMXEndpoint

**头文件**: `SuperDMXTypes.h`

DMX 网络端点配置结构体，定义一个输入或输出端的网络参数。

```cpp
USTRUCT(BlueprintType)
struct FSuperDMXEndpoint
```

#### 属性

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `bEnabled` | `bool` | `true` | 是否启用该端点。禁用时接收器/发送器不会创建 |
| `LocalIp` | `FString` | `""` | 本地绑定 IP。为空或 `"0.0.0.0"` 时绑定所有网卡 |
| `RemoteIp` | `FString` | `""` | 远端目标 IP。输出时为单播/广播地址；sACN 使用 `239.255.*.*` 多播 |
| `Port` | `int32` | `6454` | 网络端口。Art-Net 标准端口 6454，sACN 标准端口 5568 |
| `StartUniverse` | `int32` | `0` | 起始 Universe 偏移（详见 [第 7 节](#7-universe-编号与-startuniverse-映射)） |

#### 属性详解

##### bEnabled

控制端点是否激活。对于 **输入端** (`FSuperDMXIOConfig.Input`)：
- `true`：创建 UDP 接收器，开始监听 DMX 数据
- `false`：不创建接收器，不消耗网络资源

对于 **输出端** (`FSuperDMXIOConfig.Output`)：
- `true`：创建 UDP 发送器，`SendDMXBuffer()` 将数据发送到网络
- `false`：`SendDMXBuffer()` 仅更新本地缓存，不进行网络发送

##### LocalIp

指定绑定的本地网络接口。多网卡环境下，可指定特定 IP 以确保 DMX 数据仅从该网卡收发。

| 值 | 行为 |
|----|------|
| `""` 或 `"0.0.0.0"` | 监听所有网络接口 |
| `"192.168.1.100"` | 仅监听指定网卡 |
| `"10.0.0.1"` | 用于隔离 DMX 网段 |

##### RemoteIp

输出端的目标地址。

| 场景 | 值 |
|------|-----|
| 广播（所有设备） | `"255.255.255.255"` 或 `"2.255.255.255"`（Art-Net 子网广播） |
| 单播（指定设备） | `"192.168.1.200"` |
| sACN 多播 | `"239.255.0.1"`（Universe 1） |

##### StartUniverse

用于对齐不同控台的 Universe 编号习惯。详见 [第 7 节](#7-universe-编号与-startuniverse-映射)。

---

### 2.3 FSuperDMXIOConfig

**头文件**: `SuperDMXTypes.h`

完整的 DMX 输入/输出配置实体。

```cpp
USTRUCT(BlueprintType)
struct FSuperDMXIOConfig
```

#### 属性

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `Protocol` | `ESuperDMXProtocol` | `ArtNet` | 网络协议 |
| `Input` | `FSuperDMXEndpoint` | 见 Endpoint 默认 | 输入端配置（接收 DMX） |
| `Output` | `FSuperDMXEndpoint` | 见 Endpoint 默认 | 输出端配置（发送 DMX） |

#### 使用示例

```cpp
FSuperDMXIOConfig Config;
Config.Protocol = ESuperDMXProtocol::ArtNet;

// 输入配置：监听所有网卡的 Art-Net
Config.Input.bEnabled = true;
Config.Input.LocalIp = TEXT("");       // 0.0.0.0
Config.Input.Port = 6454;
Config.Input.StartUniverse = 0;        // MA2 默认对齐

// 输出配置：广播到整个子网
Config.Output.bEnabled = true;
Config.Output.RemoteIp = TEXT("255.255.255.255");
Config.Output.Port = 6454;
Config.Output.StartUniverse = 0;
```

---

## 3. USuperDMXSubsystem — DMX 通信子系统

**头文件**: `SuperDMXSubsystem.h`  
**基类**: `UEngineSubsystem`  
**导出宏**: `SUPERDMX_API`

DMX 通信的核心单例服务，**引擎启动时自动创建**，提供以下能力：
- 网络配置与生命周期管理
- Art-Net/sACN 数据接收与解析
- 多 Universe DMX 数据缓存
- DMX 数据查询（线程安全）
- DMX 数据发送（Sequencer 回放 / 程序化输出）
- 信号活跃度诊断

---

### 3.1 获取实例

`USuperDMXSubsystem` 是 `UEngineSubsystem`，全局唯一，**无需手动创建**。

**C++ 获取方式**：

```cpp
#include "SuperDMXSubsystem.h"

USuperDMXSubsystem* DMX = GEngine->GetEngineSubsystem<USuperDMXSubsystem>();
```

**蓝图获取方式**：

> 当前版本的 UFUNCTION 宏已被注释（`//UFUNCTION`），因此蓝图中**不能直接调用**这些函数。需要通过自定义蓝图函数库或 C++ 插件桥接。

---

### 3.2 生命周期

| 阶段 | 行为 |
|------|------|
| **引擎启动** | `Initialize()` 自动调用 → 以默认配置启动接收器和发送器 |
| **运行时** | 接收线程持续解析 DMX 数据包 → 写入缓存 |
| **引擎关闭** | `Deinitialize()` 自动调用 → 停止接收器 → 销毁发送器 → 释放网络资源 |

> **重要**: 引擎启动时会以默认配置（ArtNet, 端口 6454, 绑定 0.0.0.0）自动开始监听。如需更改配置，调用 `ApplyConfig()`。

---

### 3.3 配置管理

#### ApplyConfig

```cpp
void ApplyConfig(const FSuperDMXIOConfig& InConfig);
```

应用 I/O 配置并重新启动接收器和发送器。

**参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `InConfig` | `const FSuperDMXIOConfig&` | 完整的 I/O 配置 |

**行为**：
1. 保存配置到内部 `Config` 成员
2. 停止当前接收器 → 以新配置重新创建接收器
3. 销毁当前发送器 → 以新配置重新创建发送器

**注意事项**：
- 调用后**立即生效**，已缓存的 DMX 数据**不会被清除**
- 如果 `Input.bEnabled = false`，接收器不会启动
- 如果 `Output.bEnabled = false`，发送器不会创建
- 可在运行时多次调用以切换网络配置

**示例**：

```cpp
USuperDMXSubsystem* DMX = GEngine->GetEngineSubsystem<USuperDMXSubsystem>();

FSuperDMXIOConfig Config;
Config.Protocol = ESuperDMXProtocol::ArtNet;
Config.Input.bEnabled = true;
Config.Input.Port = 6454;
Config.Output.bEnabled = true;
Config.Output.RemoteIp = TEXT("192.168.1.255");
Config.Output.Port = 6454;

DMX->ApplyConfig(Config);
```

---

#### Stop

```cpp
void Stop();
```

停止所有网络活动（接收器和发送器）。

**行为**：
- 停止接收线程
- 关闭接收套接字
- 销毁发送套接字
- 已缓存的 DMX 数据保留不变

**使用场景**：
- 临时暂停 DMX 通信
- 切换网络环境前的清理

---

### 3.4 数据接收与查询

#### GetDMXBuffer

```cpp
bool GetDMXBuffer(int32 Universe, TArray<uint8>& OutBuffer) const;
```

获取指定 Universe 的**完整 DMX 缓冲**（最新一帧）。

**参数**：

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `Universe` | `int32` | 输入 | 目标 Universe 编号（内部编号，从 1 开始） |
| `OutBuffer` | `TArray<uint8>&` | 输出 | 接收缓冲，长度为该帧实际数据量（通常 512 字节） |

**返回值**：

| 值 | 说明 |
|----|------|
| `true` | 成功获取数据 |
| `false` | 该 Universe 尚未收到任何数据 |

**线程安全**: ✅ 内部使用 `FCriticalSection` 保护

**注意事项**：
- 返回的是 **数据拷贝**，调用后修改 `OutBuffer` 不影响内部缓存
- 缓冲长度取决于接收到的 Art-Net 包的 Length 字段（通常 512）
- 通道索引：`OutBuffer[0]` 对应 DMX 通道 1，`OutBuffer[511]` 对应通道 512

**示例**：

```cpp
TArray<uint8> Buffer;
if (DMX->GetDMXBuffer(1, Buffer))  // Universe 1
{
    // Buffer[0] = 通道1的值 (0-255)
    // Buffer[1] = 通道2的值
    // ...
    uint8 DimmerValue = Buffer[0];  // 假设 Dimmer 在通道 1
}
```

---

#### GetDMXValue

```cpp
bool GetDMXValue(int32 Universe, int32 Address, uint8& OutValue) const;
```

获取指定 Universe 和通道地址的**单个 DMX 值**。

**参数**：

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `Universe` | `int32` | 输入 | 目标 Universe 编号（内部编号，从 1 开始） |
| `Address` | `int32` | 输入 | DMX 通道地址（**1-512**，注意从 1 开始） |
| `OutValue` | `uint8&` | 输出 | 通道值（0-255） |

**返回值**：

| 值 | 说明 |
|----|------|
| `true` | 成功获取 |
| `false` | 地址越界（<1 或 >512）、Universe 无数据、或地址超出已接收范围 |

**线程安全**: ✅

**注意事项**：
- 地址从 **1** 开始，与 DMX 行业标准一致
- 如果 Address < 1 或 Address > 512，直接返回 `false`
- 如果 Universe 存在但 Address 超过接收到的数据长度，返回 `false`

**示例**：

```cpp
uint8 Value = 0;
if (DMX->GetDMXValue(1, 1, Value))  // Universe 1, Channel 1
{
    float Normalized = Value / 255.0f;  // 归一化到 0.0-1.0
}
```

---

#### GetKnownUniverses

```cpp
void GetKnownUniverses(TArray<int32>& OutUniverses) const;
```

获取当前已缓存数据的所有 Universe 编号列表。

**参数**：

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `OutUniverses` | `TArray<int32>&` | 输出 | Universe 编号列表（**升序排列**） |

**线程安全**: ✅

**使用场景**：
- 构建 DMX Activity Monitor（活动监视器）
- 动态发现活跃 Universe

**示例**：

```cpp
TArray<int32> Universes;
DMX->GetKnownUniverses(Universes);
for (int32 Uni : Universes)
{
    UE_LOG(LogTemp, Log, TEXT("Active Universe: %d"), Uni);
}
```

---

#### GetActiveUniverses

```cpp
void GetActiveUniverses(TArray<int32>& OutUniverses) const;
```

获取所有活跃 Universe 列表（与 `GetKnownUniverses` 功能相同，但不排序）。

**参数**：

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `OutUniverses` | `TArray<int32>&` | 输出 | Universe 编号列表（**无特定顺序**） |

**线程安全**: ✅

> **与 GetKnownUniverses 的区别**：`GetKnownUniverses` 返回升序排列的结果，`GetActiveUniverses` 返回无序结果（性能略优）。

---

#### ClearAllDMXBuffers

```cpp
void ClearAllDMXBuffers();
```

清除所有 Universe 的 DMX 缓冲数据。

**行为**：
- 清空内部 `UniverseToSlots` 映射表
- 不影响接收器/发送器的运行状态
- 清除后 `GetDMXBuffer()` / `GetDMXValue()` 将返回 `false`，直到收到新数据

**使用场景**：
- 重置 DMX 状态
- 切换场景/演出前的清理

---

#### ClearDMXBufferForUniverse

```cpp
void ClearDMXBufferForUniverse(int32 Universe);
```

清除指定 Universe 的 DMX 缓冲数据。

**参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `Universe` | `int32` | 要清除的 Universe 编号 |

**行为**：
- 从缓存中移除指定 Universe 的数据
- 其他 Universe 不受影响

---

### 3.5 数据发送

#### SendDMXBuffer

```cpp
bool SendDMXBuffer(int32 Universe, const TArray<uint8>& Buffer);
```

发送一帧 DMX 数据到指定 Universe。

**参数**：

| 参数 | 类型 | 方向 | 说明 |
|------|------|------|------|
| `Universe` | `int32` | 输入 | 目标 Universe 编号（内部编号，≥1） |
| `Buffer` | `const TArray<uint8>&` | 输入 | DMX 数据（长度 1-512 字节） |

**返回值**：

| 值 | 说明 |
|----|------|
| `true` | 发送成功（或输出未启用但本地缓存更新成功） |
| `false` | 参数无效、引擎未初始化、或网络发送失败 |

**行为流程**：
1. **参数校验**：`GEngine` 非空、`Universe > 0`、`Buffer.Num() > 0`
2. **Universe 映射**：计算 `NetworkUniverse = Universe + StartUniverse - 1`
3. **更新本地缓存**：将 Buffer 写入 `UniverseToSlots[Universe]`（供活动监视器实时反映）
4. **网络发送**（仅当 `Output.bEnabled = true`）：
   - 构建 Art-Net OpDmx 数据包（18 字节头 + DMX 数据）
   - 通过 UDP 发送到 `Output.RemoteIp:Output.Port`
5. 如果发送器未创建，自动尝试 `SetupSender()` 初始化

**Art-Net 包结构**（由 SendDMXBuffer 内部构建）：

| 偏移 | 字节数 | 内容 |
|------|--------|------|
| 0-7 | 8 | `"Art-Net\0"` 魔数 |
| 8-9 | 2 | OpCode `0x5000`（小端） |
| 10-11 | 2 | 协议版本 `0x000E` (14) |
| 12 | 1 | 序列号（固定 0） |
| 13 | 1 | 物理端口（固定 0） |
| 14-15 | 2 | Universe（小端） |
| 16-17 | 2 | 数据长度（大端） |
| 18+ | 1-512 | DMX 数据 |

**注意事项**：
- Buffer 长度将被 Clamp 到 1-512 字节
- 即使 `Output.bEnabled = false`，本地缓存也会更新
- 发送套接字启用了广播模式（`SetBroadcast(true)`）

**示例**：

```cpp
// 构建 DMX 帧并发送
TArray<uint8> Buffer;
Buffer.SetNumZeroed(512);
Buffer[0] = 255;  // 通道 1 = 全亮
Buffer[1] = 128;  // 通道 2 = 50%
Buffer[2] = 0;    // 通道 3 = 关闭

DMX->SendDMXBuffer(1, Buffer);  // 发送到 Universe 1
```

---

### 3.6 诊断与监控

#### GetSecondsSinceLastInput

```cpp
float GetSecondsSinceLastInput() const;
```

获取自上次收到**任何 Universe** 的 DMX 数据包以来经过的秒数。

**返回值**：

| 值 | 说明 |
|----|------|
| `float` | 自上次输入以来的秒数 |
| `BIG_NUMBER` | 从未收到过任何输入 |

**线程安全**: ✅

**使用场景**：
- 信号丢失检测
- 网络连接状态指示器
- 自动休眠/唤醒机制

**示例**：

```cpp
float Elapsed = DMX->GetSecondsSinceLastInput();
if (Elapsed > 5.0f)
{
    // 超过 5 秒没有收到 DMX 数据，可能信号丢失
    UE_LOG(LogTemp, Warning, TEXT("DMX signal lost for %.1f seconds"), Elapsed);
}
```

---

#### GetSecondsSinceLastInputForUniverse

```cpp
float GetSecondsSinceLastInputForUniverse(int32 Universe) const;
```

获取自上次收到**指定 Universe** 的 DMX 数据包以来经过的秒数。

**参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `Universe` | `int32` | 目标 Universe 编号 |

**返回值**：

| 值 | 说明 |
|----|------|
| `float` | 自该 Universe 上次输入以来的秒数 |
| `BIG_NUMBER` | 该 Universe 从未收到过输入 |

**线程安全**: ✅

---

#### GetProtocol

```cpp
ESuperDMXProtocol GetProtocol() const;
```

获取当前配置的 DMX 协议。

**返回值**：当前 `ESuperDMXProtocol` 值（`ArtNet`）

---

### 3.7 sACN 多播管理

#### SetSACNMulticastUniverses

```cpp
void SetSACNMulticastUniverses(const TArray<int32>& InUniverses);
```

设置 sACN 模式下需要加入的多播 Universe 列表。

**参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `InUniverses` | `const TArray<int32>&` | Universe 编号列表（>0） |

**行为**（sACN 模式下）：
1. 保存 Universe 列表
2. 停止接收器
3. 以新多播组重启接收器（自动计算 `239.255.{Hi}.{Lo}` 多播地址）

> **当前状态**: sACN 功能在当前版本中已预留但**未激活**（代码已注释）。调用此函数不会产生任何效果。

---

## 4. Sequencer 集成

SuperDMX 提供了与 UE5 Sequencer 的深度集成，允许在时间轴上录制和回放 DMX 动画。

### 4.1 UMovieSceneSuperDMXTrack

**头文件**: `Sequencer/MovieSceneSuperDMXTrack.h`  
**基类**: `UMovieSceneNameableTrack`, `IMovieSceneTrackTemplateProducer`  
**导出宏**: `SUPERDMX_API`

Sequencer 中的 DMX 轨道容器，管理多个 Universe 的 DMX 区段。

#### 类结构

```
UMovieSceneSuperDMXTrack (DMX 轨道)
 ├── Section[0]: Universe 1 的 DMX 区段
 ├── Section[1]: Universe 2 的 DMX 区段
 └── Section[N]: Universe N 的 DMX 区段
```

#### 公开方法

##### AddNewSection

```cpp
UMovieSceneSuperDMXSection* AddNewSection(int32 Universe);
```

添加指定 Universe 的区段。如果该 Universe 的区段已存在，返回现有区段。

**参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `Universe` | `int32` | 目标 Universe 编号 |

**返回值**: 指向新建或已存在的 `UMovieSceneSuperDMXSection` 指针

##### UMovieSceneTrack 接口

| 方法 | 说明 |
|------|------|
| `GetAllSections()` | 返回所有区段数组 |
| `CreateNewSection()` | 创建空白区段 |
| `AddSection(Section)` | 添加区段到轨道 |
| `RemoveSection(Section)` | 移除区段 |
| `HasSection(Section)` | 检查区段是否存在 |
| `IsEmpty()` | 轨道是否为空 |
| `RemoveAllAnimationData()` | 清除所有区段 |
| `SupportsType(SectionClass)` | 类型支持检查 |
| `CreateTemplateForSection(Section)` | 创建评估模板（`FMovieSceneSuperDMXTemplate`） |

---

### 4.2 UMovieSceneSuperDMXSection

**头文件**: `Sequencer/MovieSceneSuperDMXSection.h`  
**基类**: `UMovieSceneSection`  
**导出宏**: `SUPERDMX_API`

单个 Universe 的 DMX 动画区段，存储 1-512 个通道的关键帧曲线。

#### 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `Universe` | `int32` | 目标 Universe 编号 |
| `ChannelCurves` | `TMap<int32, FRichCurve>` | 通道曲线映射（通道号 1-512 → 动画曲线） |
| `bIsRecording` | `bool` | 录制状态标志 |

#### 公开方法

##### GetOrCreateCurve

```cpp
FRichCurve& GetOrCreateCurve(int32 Channel);
```

获取或创建指定通道的动画曲线。

**参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| `Channel` | `int32` | DMX 通道号（1-512） |

**返回值**: 对应通道的 `FRichCurve` 引用（如不存在则创建空曲线）

##### GetCurve

```cpp
const FRichCurve* GetCurve(int32 Channel) const;
```

获取指定通道的动画曲线（只读）。

**返回值**: 曲线指针，不存在时返回 `nullptr`

##### EvaluateAndSendDMX

```cpp
void EvaluateAndSendDMX(const FMovieSceneContext& Context) const;
```

根据 Sequencer 上下文的当前时间评估所有通道曲线，并通过 `USuperDMXSubsystem::SendDMXBuffer()` 发送。

**行为**：
1. 检查 `bIsRecording`，录制中跳过发送（避免回声冲突）
2. 读取当前 Universe 的最后状态作为基底缓冲
3. 遍历 `ChannelCurves`，根据时间评估每条曲线的当前值
4. 仅覆盖有曲线定义的通道（保留其他来源的通道值）
5. 调用 `SendDMXBuffer()` 发送合并后的 512 字节帧

##### 录制控制

```cpp
bool GetIsRecording() const;
void SetIsRecording(bool bIn);
```

控制录制状态。录制中 `EvaluateAndSendDMX()` 不会发送数据，避免录制时的回声/冲突。

#### 数据流

```
Sequencer 播放引擎
  │
  ▼
UMovieSceneSuperDMXTrack::CreateTemplateForSection()
  │ 创建 FMovieSceneSuperDMXTemplate
  ▼
FMovieSceneSuperDMXTemplate::Evaluate()
  │ 传入 FMovieSceneContext（含当前时间）
  ▼
UMovieSceneSuperDMXSection::EvaluateAndSendDMX()
  │ 遍历 ChannelCurves → 评估曲线值 → 构建 512B 缓冲
  ▼
USuperDMXSubsystem::SendDMXBuffer()
  │ 更新本地缓存 + 构建 Art-Net 包 + UDP 发送
  ▼
外部 DMX 设备 / 灯光 Actor 读取
```

---

## 5. 线程安全模型

`USuperDMXSubsystem` 在多线程环境下运行，采用以下保护机制：

### 线程角色

| 线程 | 职责 |
|------|------|
| **接收线程** (`FUdpSocketReceiver`) | 后台持续运行，接收 UDP 数据包 → 解析 → 写入缓存 |
| **游戏线程** | 调用 `GetDMXValue`/`GetDMXBuffer`/`SendDMXBuffer` 读写缓存 |

### 互斥保护

| 互斥锁 | 保护数据 | 使用方 |
|--------|----------|--------|
| `DataMutex` (`FCriticalSection`) | `UniverseToSlots`（DMX 数据缓存） | 读：`GetDMXBuffer/Value/Active/KnownUniverses`、`ClearAll/ClearForUniverse`<br>写：`HandleDatagram`、`SendDMXBuffer` |
| `TimeMutex` (`FCriticalSection`) | `LastAnyInputSeconds`、`UniverseToLastInputSeconds` | 读：`GetSecondsSinceLastInput*`<br>写：`HandleDatagram` |

### 锁使用模式

所有锁均使用 `FScopeLock` RAII 模式，确保异常安全：

```cpp
// 典型读取模式
bool USuperDMXSubsystem::GetDMXValue(...) const
{
    FScopeLock Lock(&DataMutex);
    // 在锁保护下读取 UniverseToSlots
    // FScopeLock 析构时自动释放锁
}
```

### 性能考量

- 接收器使用 **零节流** (`FTimespan::FromMicroseconds(0)`)，最大化吞吐量
- 接收缓冲区 **2 MB**，发送缓冲区 **256 KB**
- 锁粒度为**整个操作**（非通道级别），简单高效
- `GetDMXBuffer` 返回**数据拷贝**，持有锁时间极短

---

## 6. 网络协议细节

### 6.1 Art-Net OpDmx 包格式

SuperDMX 收发的 Art-Net 数据包遵循 Art-Net 4 协议：

```
偏移  大小  字段          值/说明
─────────────────────────────────────
0     8     Magic         "Art-Net\0" (ASCII + NULL)
8     2     OpCode        0x5000 (小端，DMX 数据包)
10    2     ProtVer       0x000E (14, 大端)
12    1     Sequence      0 (序列号，当前固定)
13    1     Physical      0 (物理端口，当前固定)
14    2     Universe      Universe 编号 (小端)
16    2     Length        DMX 数据长度 (大端, 1-512)
18    N     Data          DMX 通道数据 (N = Length)
```

### 6.2 sACN E1.31 包格式（预留）

```
偏移  大小  字段                  值/说明
──────────────────────────────────────────
4     12    ACN Packet ID         "ACN-E1.17\0\0\0"
113   2     Universe              Universe 编号 (大端)
123   2     Property Value Count  DMX数据长度+1 (大端, 含起始码)
125   1     Start Code            0x00
126   N     Data                  DMX 通道数据 (N = Count-1)
```

### 6.3 解析要点

| 项目 | Art-Net | sACN |
|------|---------|------|
| **最小包长** | 18 字节 | 126 字节 |
| **魔数验证** | `"Art-Net\0"` 8 字节 | ACN ID 12 字节 |
| **Universe 字节序** | 小端 (LE) | 大端 (BE) |
| **数据长度字节序** | 大端 (BE) | 大端 (BE) |
| **OpCode 验证** | `0x5000` | 无（通过 ACN ID 识别） |

---

## 7. Universe 编号与 StartUniverse 映射

### 问题背景

不同控台的 Universe 编号习惯不同：
- **MA2/MA3 默认**: 网络 Universe 从 0 开始（Art-Net 标准）
- **部分控台**: 网络 Universe 从 1 开始
- **SuperStage 内部**: Universe 从 1 开始

### 映射公式

`FSuperDMXEndpoint.StartUniverse` 用于统一编号：

**输入方向**（接收）：

```
InternalUniverse = NetworkUniverse - StartUniverse + 1
```

**输出方向**（发送）：

```
NetworkUniverse = InternalUniverse + StartUniverse - 1
```

### 配置示例

| 场景 | StartUniverse | 网络 Universe 0 | 网络 Universe 1 |
|------|---------------|-----------------|-----------------|
| **MA2 默认** | `0` | → 内部 1 | → 内部 2 |
| **从1开始的控台** | `1` | → 丢弃（<1） | → 内部 1 |
| **偏移对齐** | `10` | → 丢弃 | → 丢弃（<1） |

> **重要**: 如果计算得到的 `InternalUniverse < 1`，数据包将被**丢弃**。

### 最佳实践

| 控台 | 推荐 StartUniverse |
|------|-------------------|
| grandMA2 | `0` |
| grandMA3 | `0` |
| Hog4 | `0` |
| ETC Eos | `0` |
| 自定义/从1开始 | `1` |

---

## 8. 使用示例

### 8.1 基础接收示例

```cpp
#include "SuperDMXSubsystem.h"

void AMyActor::Tick(float DeltaTime)
{
    Super::Tick(DeltaTime);

    USuperDMXSubsystem* DMX = GEngine->GetEngineSubsystem<USuperDMXSubsystem>();
    if (!DMX) return;

    // 读取 Universe 1, Channel 1 的值
    uint8 DimmerRaw = 0;
    if (DMX->GetDMXValue(1, 1, DimmerRaw))
    {
        float Dimmer = DimmerRaw / 255.0f;
        // 应用亮度...
    }

    // 读取整帧
    TArray<uint8> Buffer;
    if (DMX->GetDMXBuffer(1, Buffer))
    {
        uint8 Red   = Buffer[1];  // Channel 2
        uint8 Green = Buffer[2];  // Channel 3
        uint8 Blue  = Buffer[3];  // Channel 4
    }
}
```

### 8.2 配置切换示例

```cpp
void AMyController::SwitchToArtNet(const FString& BindIp, int32 Port)
{
    USuperDMXSubsystem* DMX = GEngine->GetEngineSubsystem<USuperDMXSubsystem>();
    if (!DMX) return;

    FSuperDMXIOConfig Config;
    Config.Protocol = ESuperDMXProtocol::ArtNet;

    // 输入：绑定特定网卡
    Config.Input.bEnabled = true;
    Config.Input.LocalIp = BindIp;
    Config.Input.Port = Port;
    Config.Input.StartUniverse = 0;

    // 输出：广播
    Config.Output.bEnabled = true;
    Config.Output.RemoteIp = TEXT("255.255.255.255");
    Config.Output.Port = Port;
    Config.Output.StartUniverse = 0;

    DMX->ApplyConfig(Config);
}
```

### 8.3 DMX 发送示例（程序化控制）

```cpp
void AMyController::SendDMXFrame(int32 Universe)
{
    USuperDMXSubsystem* DMX = GEngine->GetEngineSubsystem<USuperDMXSubsystem>();
    if (!DMX) return;

    TArray<uint8> Frame;
    Frame.SetNumZeroed(512);

    // 设置通道值
    Frame[0] = 255;   // Channel 1: Dimmer 全亮
    Frame[1] = 128;   // Channel 2: Pan 中位
    Frame[2] = 64;    // Channel 3: Tilt 1/4
    Frame[3] = 200;   // Channel 4: Red
    Frame[4] = 100;   // Channel 5: Green
    Frame[5] = 50;    // Channel 6: Blue

    bool bSuccess = DMX->SendDMXBuffer(Universe, Frame);
    // bSuccess 为 true 表示发送成功
}
```

### 8.4 信号监控示例

```cpp
void AMyMonitor::CheckDMXHealth()
{
    USuperDMXSubsystem* DMX = GEngine->GetEngineSubsystem<USuperDMXSubsystem>();
    if (!DMX) return;

    // 全局信号检查
    float GlobalElapsed = DMX->GetSecondsSinceLastInput();
    bool bSignalActive = (GlobalElapsed < 2.0f);

    // 逐 Universe 检查
    TArray<int32> Universes;
    DMX->GetKnownUniverses(Universes);
    for (int32 Uni : Universes)
    {
        float UniElapsed = DMX->GetSecondsSinceLastInputForUniverse(Uni);
        if (UniElapsed > 5.0f)
        {
            UE_LOG(LogTemp, Warning, TEXT("Universe %d: signal lost (%.1fs)"), Uni, UniElapsed);
        }
    }
}
```

---

## 9. 常见问题与排错

### Q1: 收不到 DMX 数据

| 检查项 | 操作 |
|--------|------|
| 防火墙 | 确保 UDP 6454 端口未被阻止 |
| IP 绑定 | 尝试 `LocalIp = ""` 绑定所有网卡 |
| 控台输出 | 确认控台已启用 Art-Net 输出且网段正确 |
| StartUniverse | 检查偏移是否导致 InternalUniverse < 1（数据被丢弃） |
| Input.bEnabled | 确保为 `true` |
| 诊断 | `GetSecondsSinceLastInput()` 返回 `BIG_NUMBER` 说明从未收到数据 |

### Q2: 数据延迟高

| 检查项 | 操作 |
|--------|------|
| 网络拓扑 | DMX 设备与 UE 主机应在同一子网/VLAN |
| 交换机 | 使用千兆交换机，避免 IGMP snooping 干扰广播 |
| 帧率 | 确保 UE 帧率 ≥ DMX 刷新率（通常 44fps） |

### Q3: SendDMXBuffer 返回 false

| 原因 | 解决 |
|------|------|
| `Universe ≤ 0` | Universe 必须 ≥ 1 |
| `Buffer` 为空 | 至少包含 1 字节 |
| `Output.bEnabled = false` | 此时仅更新本地缓存（返回 `true`） |
| 发送器未初始化 | 检查 `Output.RemoteIp` 是否有效 |

### Q4: 多 Universe 性能

- 每个 Universe 独立缓存，查询互不干扰
- 锁粒度为操作级别，高并发下无显著争用
- 推荐使用 `GetDMXBuffer()` 批量读取而非逐通道 `GetDMXValue()`

### Q5: Sequencer 录制与回放冲突

- 录制时 `bIsRecording = true`，自动跳过回放发送
- 录制完成后调用 `SetIsRecording(false)` 恢复回放
- 回放时的缓冲合并策略：仅覆盖有曲线的通道，保留其他通道值

---

## 附录：API 速查表

### USuperDMXSubsystem

| 函数 | 类别 | 线程安全 | 说明 |
|------|------|----------|------|
| `ApplyConfig(Config)` | 配置 | — | 应用 I/O 配置并重启收发器 |
| `Stop()` | 配置 | — | 停止所有网络活动 |
| `GetDMXBuffer(Uni, Out)` | 查询 | ✅ | 获取整帧 DMX 缓冲 |
| `GetDMXValue(Uni, Addr, Out)` | 查询 | ✅ | 获取单通道值 |
| `GetKnownUniverses(Out)` | 查询 | ✅ | 获取已知 Universe 列表（排序） |
| `GetActiveUniverses(Out)` | 查询 | ✅ | 获取活跃 Universe 列表（无序） |
| `ClearAllDMXBuffers()` | 管理 | ✅ | 清除所有缓冲 |
| `ClearDMXBufferForUniverse(Uni)` | 管理 | ✅ | 清除单个 Universe 缓冲 |
| `SendDMXBuffer(Uni, Buffer)` | 发送 | ✅ | 发送 DMX 帧 |
| `GetSecondsSinceLastInput()` | 诊断 | ✅ | 全局最后输入时间 |
| `GetSecondsSinceLastInputForUniverse(Uni)` | 诊断 | ✅ | 指定 Universe 最后输入时间 |
| `GetProtocol()` | 查询 | — | 当前协议 |
| `SetSACNMulticastUniverses(Universes)` | sACN | — | 设置多播 Universe（预留） |

### UMovieSceneSuperDMXTrack

| 函数 | 说明 |
|------|------|
| `AddNewSection(Universe)` | 添加 Universe 区段 |
| `GetAllSections()` | 获取所有区段 |

### UMovieSceneSuperDMXSection

| 函数/属性 | 说明 |
|-----------|------|
| `Universe` | 目标 Universe |
| `ChannelCurves` | 通道→曲线映射 |
| `GetOrCreateCurve(Channel)` | 获取/创建通道曲线 |
| `GetCurve(Channel)` | 获取通道曲线（只读） |
| `EvaluateAndSendDMX(Context)` | 评估并发送 DMX |
| `SetIsRecording(bool)` | 设置录制状态 |
| `GetIsRecording()` | 查询录制状态 |

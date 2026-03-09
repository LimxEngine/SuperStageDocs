# SuperData 协议规范 v2.0

## 文档信息

| 项目 | 内容 |
|------|------|
| 协议名称 | SuperData Protocol |
| 版本 | 2.0.0 |
| 发布日期 | 2025-12-01 |
| 作者 | Yunsio SuperStage Team |
| 状态 | ✅ 正式发布 |

---

## 1. 概述

### 1.1 协议目的

SuperData 协议是一个跨引擎、跨平台的实时数据共享协议，专为舞台灯具数据同步设计。该协议允许不同软件平台（Unity、Unreal Engine、Vectorworks、GrandMA 等）之间实时共享灯具配置信息，包括：

- 灯具位置和旋转
- DMX 地址配置（Universe、Address、FixtureID）
- 灯具类型信息
- 自定义属性

### 1.2 架构模式：群聊模式

**v2.0 采用"群聊"架构**：

```
┌─────────────────────────────────────────────────────────────┐
│              SuperDataServer.exe (中央路由器)                │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              Session Manager (会话管理)              │   │
│   │         管理所有客户端连接，缓存各端数据              │   │
│   └─────────────────────────────────────────────────────┘   │
│                            │                                │
│                     TCP localhost:5966                      │
└─────────────────────────────────────────────────────────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
        Unity               UE               VW
       (客户端)           (客户端)          (客户端)
      "我的灯具数据"      "我的灯具数据"     "我的灯具数据"
```

**核心概念**：

1. **独立服务进程** - SuperDataServer.exe 作为中央路由器，无需各端重复实现协议层
2. **群聊模式** - 各客户端加入同一个"群"，共享各自的灯具数据
3. **选择性导入** - 客户端可以选择从群内任意其他客户端导入数据
4. **自动生命周期** - 第一个客户端连接时启动服务，最后一个断开时服务自动退出

**使用场景示例**：

```
1. Unity 用户打开连接面板 → 自动启动 Server → Unity 加入群聊
2. UE 用户打开连接面板 → 发现 Server 已运行 → UE 加入群聊
3. VW 用户打开连接面板 → VW 加入群聊
4. UE 用户想导入灯具 → 看到群内有 Unity 和 VW → 选择从 Unity 导入
5. 所有用户关闭连接 → Server 自动退出
```

### 1.3 设计原则

1. **简洁性** - 使用 JSON 作为负载格式，便于调试和跨语言实现
2. **可靠性** - TCP 用于数据传输
3. **兼容性** - 统一坐标系标准，各端自行转换
4. **扩展性** - 预留自定义属性字段，支持未来扩展
5. **零配置** - 客户端只需调用 exe，无需手动配置 IP/端口

### 1.4 支持平台

| 平台代码 | 平台名称 | 状态 |
|----------|----------|------|
| 1 | Unity | ✅ 已实现 |
| 2 | Unreal Engine | ✅ 已实现 |
| 3 | Vectorworks | 📋 计划中 |
| 4 | GrandMA2 | ✅ 已实现 |
| 255 | Custom | 预留 |

### 1.5 客户端示例

提供 8 种编程语言的完整客户端库实现：

| 语言 | 目录 | 特性 |
|------|------|------|
| Python | `Examples/Python/` | asyncio 异步、类型提示 |
| JavaScript | `Examples/JavaScript/` | Node.js、事件驱动 |
| TypeScript | `Examples/TypeScript/` | 强类型、完整类型定义 |
| C# | `Examples/CSharp/` | .NET 8、async/await |
| C++ | `Examples/Cpp/` | Asio 异步、跨平台 |
| Rust | `Examples/Rust/` | Tokio 异步、安全内存 |
| Java | `Examples/Java/` | Maven 构建、Gson JSON |
| Lua | `Examples/Lua/` | LuaSocket、GrandMA 兼容 |

所有客户端库都内置：
- ✅ 自动启动服务器（从注册表读取路径）
- ✅ 禁用系统代理
- ✅ TCP 粘包处理
- ✅ 心跳保活

---

## 2. 网络层规范

### 2.1 端口定义

| 用途 | 协议 | 端口 | 说明 |
|------|------|------|------|
| 数据传输 | TCP | 5966 | 可靠传输 |

> **v2.0 变更**：移除 UDP 5965 服务发现端口。采用独立服务进程模式，客户端直接连接 `localhost:5966`。

### 2.2 数据包格式

所有数据包采用统一的二进制头 + JSON 负载格式：

```
┌─────────────────────────────────────────────────────────────┐
│                     数据包结构 (Packet)                      │
├─────────────────────────────────────────────────────────────┤
│  偏移量  │  长度  │  字段名          │  类型      │  说明    │
├─────────────────────────────────────────────────────────────┤
│  0       │  4     │  Magic           │  char[4]   │  "SPDT"  │
│  4       │  2     │  Version         │  uint16    │  协议版本 │
│  6       │  2     │  PacketType      │  uint16    │  包类型   │
│  8       │  4     │  SequenceNumber  │  uint32    │  序列号   │
│  12      │  8     │  Timestamp       │  int64     │  时间戳   │
│  20      │  4     │  PayloadLength   │  uint32    │  负载长度 │
│  24      │  N     │  Payload         │  byte[N]   │  JSON数据 │
└─────────────────────────────────────────────────────────────┘
```

#### 2.2.1 字段说明

| 字段 | 类型 | 字节序 | 说明 |
|------|------|--------|------|
| Magic | char[4] | - | 固定值 "SPDT" (0x53, 0x50, 0x44, 0x54) |
| Version | uint16 | Little Endian | 当前版本: 1 |
| PacketType | uint16 | Little Endian | 见 [3. 数据包类型](#3-数据包类型) |
| SequenceNumber | uint32 | Little Endian | 递增序列号，用于追踪 |
| Timestamp | int64 | Little Endian | Unix 时间戳（毫秒） |
| PayloadLength | uint32 | Little Endian | JSON 负载的字节长度 |
| Payload | byte[] | UTF-8 | JSON 格式的负载数据 |

#### 2.2.2 头部大小

**固定头部大小: 24 字节**

### 2.3 字节序

所有多字节整数采用 **Little Endian（小端序）** 编码。

---

## 3. 数据包类型

### 3.1 类型定义

```
┌──────────────────────────────────────────────────────────┐
│                    数据包类型枚举                         │
├──────────────────────────────────────────────────────────┤
│  分类        │  代码    │  名称              │  方向     │
├──────────────────────────────────────────────────────────┤
│  连接管理    │  0x0010  │  Connect           │  C → S   │
│              │  0x0011  │  ConnectAck        │  S → C   │
│              │  0x0012  │  Disconnect        │  C ↔ S   │
│              │  0x0013  │  Heartbeat         │  C ↔ S   │
├──────────────────────────────────────────────────────────┤
│  客户端管理  │  0x0014  │  ClientJoined      │  S → C   │
│              │  0x0015  │  ClientLeft        │  S → C   │
├──────────────────────────────────────────────────────────┤
│  数据同步    │  0x0020  │  FixtureListRequest│  C → S   │
│              │  0x0021  │  FixtureListResponse│ S → C   │
│              │  0x0022  │  FixtureUpdate     │  C → S   │
│              │  0x0023  │  FixtureFullSync   │  C → S   │
│              │  0x0024  │  FixtureDelete     │  C → S   │
├──────────────────────────────────────────────────────────┤
│  错误        │  0xFF00  │  Error             │  S → C   │
└──────────────────────────────────────────────────────────┘

图例: S = Server, C = Client
```

> **v2.0 变更**：
> - 移除服务发现包（0x0001-0x0003）
> - 移除类型映射包（0x0030-0x0032），类型由客户端自行管理
> - 新增 ClientJoined/ClientLeft（0x0014-0x0015）用于群成员通知
> - 新增 FixtureFullSync（0x0023）用于客户端上报完整数据
> - 移除 FixtureCreate/FixtureSyncComplete，统一使用 FixtureFullSync

### 3.2 连接管理包

#### 3.2.1 Connect (0x0010)

客户端请求加入群聊。

**Payload 结构**:
```json
{
    "clientId": "550e8400-e29b-41d4-a716-446655440000",
    "clientName": "My UE Project",
    "platform": 2,
    "platformVersion": "5.6",
    "protocolVersion": "2.0.0"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| clientId | string | ✓ | 客户端唯一标识符（UUID，客户端生成） |
| clientName | string | ✓ | 客户端显示名称（用户可自定义） |
| platform | int | ✓ | 平台代码（见1.4） |
| platformVersion | string | ✓ | 平台版本号 |
| protocolVersion | string | ✓ | 协议版本 |

#### 3.2.2 ConnectAck (0x0011)

服务端响应连接请求，返回当前群内所有成员。

**Payload 结构**:
```json
{
    "accepted": true,
    "errorCode": 0,
    "errorMessage": "",
    "clients": [
        {
            "clientId": "...",
            "clientName": "Unity Project",
            "platform": 1,
            "fixtureCount": 42
        },
        {
            "clientId": "...",
            "clientName": "VW Project",
            "platform": 3,
            "fixtureCount": 28
        }
    ]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| accepted | bool | 是否接受连接 |
| errorCode | int | 错误代码（见7.1） |
| errorMessage | string | 错误描述 |
| clients | array | 当前群内所有客户端（不含自己） |

#### 3.2.3 Disconnect (0x0012)

主动断开连接通知。

**Payload**: `{}`

#### 3.2.4 Heartbeat (0x0013)

心跳保活包，双向发送。

**发送间隔**: 3000ms  
**超时时间**: 10000ms（未收到心跳则断开）

**Payload**: `{}`

---

### 3.3 客户端管理包

#### 3.3.1 ClientJoined (0x0014)

当有新客户端加入群聊时，Server 广播给所有其他客户端。

**Payload 结构**:
```json
{
    "clientId": "550e8400-e29b-41d4-a716-446655440000",
    "clientName": "New UE Project",
    "platform": 2,
    "fixtureCount": 0
}
```

#### 3.3.2 ClientLeft (0x0015)

当有客户端离开群聊时，Server 广播给所有其他客户端。

**Payload 结构**:
```json
{
    "clientId": "550e8400-e29b-41d4-a716-446655440000"
}
```

---

### 3.4 数据同步包

#### 3.4.1 FixtureListRequest (0x0020)

请求指定客户端的灯具列表。

**Payload 结构**:
```json
{
    "sourceClientId": "550e8400-e29b-41d4-a716-446655440000"
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| sourceClientId | string | ✓ | 要获取数据的客户端ID |

#### 3.4.2 FixtureListResponse (0x0021)

返回指定客户端的灯具列表。

**Payload 结构**:
```json
{
    "sourceClientId": "550e8400-e29b-41d4-a716-446655440000",
    "fixtures": [
        {
            "uuid": "fixture_001",
            "name": "Spot Light 1",
            "fixtureType": "BP_SpotLight",
            "universe": 1,
            "startAddress": 1,
            "fixtureID": 1,
            "position": [0.0, 5.0, 3.0],
            "rotation": [45.0, 0.0, 0.0],
            "scale": [1.0, 1.0, 1.0],
            "channelSpan": 16,
            "customProperties": {
                "manufacturer": "Robe",
                "model": "Robin 600"
            }
        }
    ],
    "totalCount": 42
}
```

#### 3.4.3 FixtureUpdate (0x0022)

客户端上报灯具增量更新（修改单个灯具），Server 更新缓存并广播给其他客户端。

**Payload 结构**:
```json
{
    "uuid": "fixture_001",
    "changes": {
        "position": [1.0, 5.0, 3.0],
        "universe": 2
    }
}
```

> **广播行为**：Server 收到后会转发给群内所有其他客户端，其他客户端可据此实时更新视图。

#### 3.4.4 FixtureFullSync (0x0023)

客户端上报完整的灯具列表（全量同步）。用于：
- 客户端首次连接后上报自己的数据
- 客户端数据发生批量变化时重新同步

**Payload 结构**:
```json
{
    "fixtures": [
        {
            "uuid": "fixture_001",
            "name": "Spot Light 1",
            "fixtureType": "BP_SpotLight",
            "universe": 1,
            "startAddress": 1,
            "fixtureID": 1,
            "position": [0.0, 5.0, 3.0],
            "rotation": [45.0, 0.0, 0.0],
            "scale": [1.0, 1.0, 1.0],
            "channelSpan": 16,
            "customProperties": {}
        }
    ],
    "totalCount": 42
}
```

> **说明**：Server 收到后会完全替换该客户端的缓存数据，并更新 fixtureCount。

#### 3.4.5 FixtureDelete (0x0024)

客户端上报删除灯具。

**Payload 结构**:
```json
{
    "uuids": ["fixture_001", "fixture_002"]
}
```

> **广播行为**：Server 收到后更新缓存并转发给群内所有其他客户端。

---

## 4. 数据结构定义

### 4.1 灯具数据 (Fixture)

```json
{
    "uuid": "string",           // 唯一标识符（必填，客户端生成UUID）
    "name": "string",           // 显示名称（必填）
    "fixtureType": "string",    // 灯具类型名（必填，客户端自定义）
    "universe": 1,              // DMX Universe (1-256)
    "startAddress": 1,          // DMX Address (1-512)
    "fixtureID": 1,             // 灯具编号 (1-65535)
    "position": [0, 0, 0],      // 位置 [X, Y, Z] 厘米（标准坐标系）
    "rotation": [0, 0, 0],      // 旋转 [X, Y, Z] 度（欧拉角）
    "scale": [1, 1, 1],         // 缩放 [X, Y, Z]
    "channelSpan": 0,           // DMX 通道跨度
    "customProperties": {}      // 自定义属性字典
}
```

> **UUID 生成**：由客户端在创建灯具时生成，推荐使用标准 UUID v4 格式。

### 4.2 客户端信息 (ClientInfo)

```json
{
    "clientId": "string",       // 客户端唯一ID（UUID）
    "clientName": "string",     // 客户端显示名称
    "platform": 1,              // 平台代码
    "fixtureCount": 0           // 当前灯具数量
}
```

> **v2.0 变更**：简化了原 ServiceInfo 结构，移除了与独立服务模式无关的字段。

---

## 5. 坐标系规范

### 5.1 标准坐标系定义

SuperData 协议使用 **右手坐标系，Z轴向上** 作为标准坐标系，与 MVR/GDTF 行业标准一致。

```
        Z (上)
        │
        │
        │
        └──────── Y (前/Downstage)
       /
      /
     X (右/Stage Right)
```

| 轴 | 方向 | 正值含义 | 舞台术语 |
|----|------|----------|----------|
| X | 右 | 向右移动 | Stage Right |
| Y | 前 | 向前移动 | Downstage（面向观众）|
| Z | 上 | 向上移动 | Up |

**单位**: 厘米 (cm) ⚠️ 重要  
**旋转**: 欧拉角（度），XYZ顺序
**手性**: 右手坐标系

> **为什么使用厘米？**
> - 与 UE 单位一致，减少转换误差
> - 与大多数 DCC 软件（3ds Max、Maya）一致
> - 精度更高，避免小数精度问题

### 5.1.1 手性与旋转方向（⚠️ 关键）

**左手系 vs 右手系的旋转方向相反！**

```
右手法则（Standard/MA/VW）：     左手法则（Unity/UE）：
拇指指向轴正方向                  拇指指向轴正方向
四指弯曲方向 = 正旋转方向         四指弯曲方向 = 正旋转方向

从轴正方向看：                    从轴正方向看：
正角度 = 逆时针                   正角度 = 顺时针
```

**结论**：当从左手系转换到右手系（或反之）时，**所有旋转角度都需要取反**！

```
左手系 → 右手系:
  stdRot.X = -localRot.X
  stdRot.Y = -localRot.Y  (轴重映射后)
  stdRot.Z = -localRot.Z  (轴重映射后)
```

### 5.2 各平台坐标系

> **重要原则**: 每个平台在发送数据时将本地坐标转换为标准坐标系，在接收数据时将标准坐标系转换为本地坐标系。这样可以确保跨平台数据的正确性。

| 平台 | 坐标系 | 单位 | 轴定义 | 需要单位转换 |
|------|--------|------|--------|--------------|
| **Standard** | 右手 Z-up | cm | X右, Y前, Z上 | - |
| Unity | 左手 Y-up | m | X右, Y上, Z前 | ✅ m→cm |
| UE | 左手 Z-up | cm | X前, Y右, Z上 | ❌ |
| MA | 右手 Z-up | m | X右, Y前, Z上 | ✅ m→cm |
| VW | 右手 Z-up | mm | X右, Y前, Z上 | ✅ mm→cm |

#### 5.2.1 Unity (左手 Y-up, 单位: m)

```
Unity:               Standard:
    Y (上)              Z (上)
    │                   │
    │                   │
    │                   │
    └──── X (右)        └──── Y (前)
   /                   /
  Z (前)              X (右)
```

**单位**: Unity 使用米 (m)，需要与标准坐标系 cm 转换

**转换公式**:

> ⚠️ **重要**：Unity 是左手系，Standard 是右手系，**所有旋转轴都需要取反**！

```
Unity → Standard:
  // 位置转换 (m → cm) + 轴重映射
  stdPos.X = unityPos.X * 100    // X(右) → X(右)
  stdPos.Y = unityPos.Z * 100    // Z(前) → Y(前)
  stdPos.Z = unityPos.Y * 100    // Y(上) → Z(上)
  
  // 旋转转换：轴重映射 + 手性修正（全部取反）
  stdRot.X = -unityRot.X         // 绕右轴，手性相反
  stdRot.Y = -unityRot.Z         // 绕前轴，手性相反
  stdRot.Z = -unityRot.Y         // 绕上轴，手性相反

Standard → Unity:
  // 位置转换 (cm → m) + 轴重映射
  unityPos.X = stdPos.X * 0.01   // X(右) → X(右)
  unityPos.Y = stdPos.Z * 0.01   // Z(上) → Y(上)
  unityPos.Z = stdPos.Y * 0.01   // Y(前) → Z(前)
  
  // 旋转转换：轴重映射 + 手性修正（全部取反）
  unityRot.X = -stdRot.X         // 绕右轴，手性相反
  unityRot.Y = -stdRot.Z         // 绕上轴，手性相反
  unityRot.Z = -stdRot.Y         // 绕前轴，手性相反
```

#### 5.2.2 Unreal Engine (左手 Z-up, 单位: cm)

```
UE:                  Standard:
    Z (上)              Z (上)
    │                   │
    │                   │
    │                   │
    └──── Y (右)        └──── Y (前)
   /                   /
  X (前)              X (右)
```

**单位**: UE 使用厘米 (cm)，与标准坐标系相同，无需转换单位

**转换公式**:

```
UE 欧拉角定义（FRotator）：
  Pitch (绕右轴/Y轴旋转)
  Yaw   (绕上轴/Z轴旋转)
  Roll  (绕前轴/X轴旋转)

Standard 欧拉角定义：
  X = 绕右轴旋转 (对应 UE Pitch)
  Y = 绕前轴旋转 (对应 UE Roll)
  Z = 绕上轴旋转 (对应 UE Yaw)

UE → Standard:
  // 位置转换（轴重映射，单位相同）
  stdPos.X = uePos.Y             // Y(右) → X(右)
  stdPos.Y = uePos.X             // X(前) → Y(前)
  stdPos.Z = uePos.Z             // Z(上) → Z(上)
  
  // 旋转转换（按轴物理意义映射 + 180°偏移）
  stdRot.X = ueRot.Pitch         // 绕右轴
  stdRot.Y = ueRot.Roll          // 绕前轴
  stdRot.Z = ueRot.Yaw - 180     // 绕上轴，减180°偏移

Standard → UE:
  // 位置转换（轴重映射，单位相同）
  uePos.X = stdPos.Y             // Y(前) → X(前)
  uePos.Y = stdPos.X             // X(右) → Y(右)
  uePos.Z = stdPos.Z             // Z(上) → Z(上)
  
  // 旋转转换（按轴物理意义映射 + 180°偏移）
  ueRot.Pitch = stdRot.X         // 绕右轴
  ueRot.Roll  = stdRot.Y         // 绕前轴
  ueRot.Yaw   = stdRot.Z + 180   // 绕上轴，加180°偏移
```

> **注意**：Yaw 需要 ±180° 偏移是因为 UE 和 Standard 的"前方"定义在经过轴重映射后存在 180° 的朝向差异。

#### 5.2.3 GrandMA2 (右手 Z-up, 单位: m)

```
MA2:                 Standard:
    Z (上)              Z (上)
    │                   │
    │                   │
    │                   │
    └──── Y (前)        └──── Y (前)
   /                   /
  X (右)              X (右)
```

**单位**: MA2 使用米 (m)，Standard 使用厘米 (cm)，需要转换单位

**转换公式**:

```
Standard → MA2:
  // 位置转换 (cm → m)
  ma2Pos.X = stdPos.X * 0.01   // X(右) → X(右)
  ma2Pos.Y = stdPos.Y * 0.01   // Y(前) → Y(前)
  ma2Pos.Z = stdPos.Z * 0.01   // Z(上) → Z(上)
  
  // 旋转转换（根据实测修正）
  ma2Rot.X = stdRot.X          // 绕右轴，不变
  ma2Rot.Y = stdRot.Y + 180    // 绕前轴，+180°
  ma2Rot.Z = stdRot.Z + 90     // 绕上轴，+90°

MA2 → Standard:
  // 位置转换 (m → cm)
  stdPos.X = ma2Pos.X * 100    // X(右) → X(右)
  stdPos.Y = ma2Pos.Y * 100    // Y(前) → Y(前)
  stdPos.Z = ma2Pos.Z * 100    // Z(上) → Z(上)
  
  // 旋转转换
  stdRot.X = ma2Rot.X          // 绕右轴，不变
  stdRot.Y = ma2Rot.Y - 180    // 绕前轴，-180°
  stdRot.Z = ma2Rot.Z - 90     // 绕上轴，-90°
```

> **注意**：MA2 的旋转偏移是根据实际测试得出的经验值，可能与灯具默认朝向有关。

### 5.3 坐标转换矩阵

#### Unity ↔ Standard

```
Unity→Standard 位置矩阵:     Standard→Unity 位置矩阵:
┌ 1  0  0 ┐                  ┌ 1  0  0 ┐
│ 0  0  1 │                  │ 0  0  1 │
└ 0  1  0 ┘                  └ 0  1  0 ┘
```

#### UE ↔ Standard

```
UE→Standard 位置矩阵:        Standard→UE 位置矩阵:
┌ 0  1  0 ┐                  ┌ 0  1  0 ┐
│ 1  0  0 │                  │ 1  0  0 │
└ 0  0  1 ┘                  └ 0  0  1 ┘

（单位相同，仅 X↔Y 交换）
```

---

## 6. 连接与数据同步流程

### 6.1 客户端启动流程

```
客户端打开连接面板
        │
        ▼
  尝试 TCP 连接 localhost:5966
        │
        ├─ 连接成功 → 服务已运行 → 发送 Connect
        │
        └─ 连接失败 → 启动 SuperDataServer.exe
                            │
                            ▼
                      轮询等待端口就绪（最多5秒）
                            │
                            ▼
                      TCP 连接 → 发送 Connect
```

### 6.2 加入群聊流程

> ⚠️ **重要提醒**：
> - 建立 TCP 连接后必须设置 `TCP_NODELAY = true`（见 8.8）
> - 接收数据时必须处理 TCP 粘包（见 8.7）

```
┌──────────────┐                              ┌─────────┐
│ 新客户端(UE)  │                              │ Server  │
└──────┬───────┘                              └────┬────┘
       │                                           │
       │  ═══════ TCP Connect ═══════════════════► │
       │         (设置 TCP_NODELAY=true)            │
       │                                           │
       │  ──── Connect (0x0010) ─────────────────► │
       │       {clientId, clientName, platform}    │
       │                                           │
       │  ◄─── ConnectAck (0x0011) ──────────────  │
       │       {clients: [Unity, VW, ...]}         │  返回当前群成员
       │                                           │
       │  ──── FixtureFullSync (0x0023) ────────►  │  上报自己的灯具
       │       {fixtures: [...]}                   │
       │                                           │
       │  ◄───────── Heartbeat ──────────────────► │  (每3秒)
       │                                           │
       ▼                                           ▼

同时，Server 广播给群内其他客户端：

┌─────────┐                              ┌──────────────┐
│ Server  │                              │ 已有客户端    │
└────┬────┘                              └──────┬───────┘
     │                                          │
     │  ──── ClientJoined (0x0014) ───────────► │
     │       {clientId, clientName, platform}   │
     │                                          │
     ▼                                          ▼
```

### 6.3 从其他客户端导入数据

```
┌─────────┐                              ┌─────────┐
│ UE 客户端│                              │ Server  │
└────┬────┘                              └────┬────┘
     │                                        │
     │  用户选择"从 Unity 导入"                │
     │                                        │
     │  ──── FixtureListRequest (0x0020) ───► │
     │       {sourceClientId: "unity-xxx"}    │
     │                                        │
     │  ◄─── FixtureListResponse (0x0021) ─── │
     │       {sourceClientId, fixtures: [...]}│  返回 Unity 的灯具数据
     │                                        │
     │  [UE 本地创建/更新灯具]                 │
     │                                        │
     ▼                                        ▼
```

### 6.4 数据变更广播

```
┌─────────┐                              ┌─────────┐
│ Unity   │                              │ Server  │
└────┬────┘                              └────┬────┘
     │                                        │
     │  用户移动了一个灯具                     │
     │                                        │
     │  ──── FixtureUpdate (0x0022) ────────► │
     │       {uuid, changes: {position: ...}} │
     │                                        │
     │                                        ▼
     │                              ┌─────────────────┐
     │                              │ 更新 Unity 缓存  │
     │                              │ 广播给其他客户端 │
     │                              └─────────────────┘
     │                                        │
     │                                        │  FixtureUpdate
     │                              ┌─────────┴─────────┐
     │                              ▼                   ▼
     │                            UE                   VW
     │                       (收到变更通知)        (收到变更通知)
     ▼
```

### 6.5 客户端离开与服务退出

```
┌─────────┐                              ┌─────────┐
│ Unity   │                              │ Server  │
└────┬────┘                              └────┬────┘
     │                                        │
     │  用户关闭连接                           │
     │                                        │
     │  ──── Disconnect (0x0012) ───────────► │
     │                                        │
     │  ═══════ TCP 断开 ═══════════════════► │
     │                                        │
                                              ▼
                                    ┌─────────────────┐
                                    │ 移除 Unity 会话  │
                                    │ 清除 Unity 数据  │
                                    │ 广播 ClientLeft  │
                                    └─────────────────┘
                                              │
                                              ▼
                                    检查连接数是否为0
                                              │
                                    ├─ 不为0 → 继续运行
                                    │
                                    └─ 为0 → 立即退出进程
```

---

## 7. 错误处理

### 7.1 错误代码

| 代码 | 名称 | 说明 |
|------|------|------|
| 0 | None | 无错误 |
| 1 | InvalidPacket | 无效数据包格式 |
| 2 | VersionMismatch | 协议版本不匹配 |
| 3 | ConnectionRefused | 连接被拒绝 |
| 4 | Timeout | 操作超时 |
| 5 | TypeMappingRequired | 需要类型映射 |
| 6 | FixtureNotFound | 灯具未找到 |
| 7 | PermissionDenied | 权限不足 |
| 255 | InternalError | 内部错误 |

### 7.2 错误响应格式

```json
{
    "errorCode": 1,
    "errorMessage": "Invalid packet format",
    "details": "Magic header mismatch"
}
```

---

## 8. 实现指南

### 8.1 服务端实现（SuperDataServer.exe）

服务端作为独立进程，由 C++ 实现，各平台客户端共享使用。

**核心功能**：
1. **TCP 服务器** - 监听 localhost:5966
2. **会话管理** - 管理所有客户端连接
3. **数据缓存** - 缓存每个客户端的灯具数据
4. **消息路由** - 转发/广播消息给相关客户端
5. **自动退出** - 最后客户端断开时退出进程
6. **单实例** - 使用命名互斥锁防止重复启动

**进程特性**：
- 无窗口后台运行（Windows: SUBSYSTEM:WINDOWS）
- 系统级安装，所有客户端共享

### 8.2 安装与部署

#### 8.2.1 下载安装

**下载地址**：[https://yunsio.com/superdata/download](https://yunsio.com/superdata/download)

**安装包**：`SuperDataSetup-2.0.0.exe`（约 2.2 MB）

**安装步骤**：
1. 下载并运行安装包
2. 选择语言（中文/English）
3. 按向导完成安装
4. 无需手动配置，客户端会自动启动服务

#### 8.2.2 安装位置

SuperDataServer 作为独立软件安装到系统，所有客户端共享使用：

**默认安装路径**：
```
C:\Program Files\Yunsio\SuperData\
├── SuperDataServer.exe    # 服务器主程序
├── version.txt            # 版本信息
└── unins000.exe           # 卸载程序
```

**注册表配置**（自动写入）：
```
HKEY_LOCAL_MACHINE\SOFTWARE\Yunsio\SuperData
├── InstallPath  (REG_SZ)  = "C:\Program Files\Yunsio\SuperData"
├── Version      (REG_SZ)  = "2.0.0"
└── ExePath      (REG_SZ)  = "C:\Program Files\Yunsio\SuperData\SuperDataServer.exe"
```

#### 8.2.3 客户端查找服务逻辑

```pseudocode
function FindServerExePath():
    // 1. 查询注册表
    regPath = "HKEY_LOCAL_MACHINE\\SOFTWARE\\Yunsio\\SuperData"
    exePath = Registry.GetValue(regPath, "ExePath")
    
    if exePath != null and FileExists(exePath):
        return exePath
    
    // 2. 检查 PATH 环境变量
    exePath = FindInPath("SuperDataServer.exe")
    if exePath != null:
        return exePath
    
    // 3. 未安装
    return Error("SuperData Server 未安装，请先安装: https://yunsio.com/superdata")
```

#### 8.2.4 安装包功能

| 功能 | 说明 |
|------|------|
| 多语言 | 支持中文/英文界面切换 |
| 安装 | 复制文件、写注册表、创建快捷方式 |
| 卸载 | 自动关闭进程、删除文件、清理注册表 |
| 升级 | 检测旧版本、自动替换 |
| 静默安装 | 支持 `/SILENT` 参数用于自动化部署 |

### 8.3 客户端最小实现要求

实现一个兼容的 SuperData 客户端，只需要：

1. **自动启动** - 从注册表查找并启动 SuperDataServer.exe（见 8.4）
2. **TCP 连接** - 连接到 localhost:5966
3. **数据包解析** - 解析 24 字节头 + JSON 负载
4. **心跳机制** - 每 3 秒发送心跳
5. **坐标转换** - 实现本地坐标系与标准坐标系转换
6. **TCP 粘包处理** - ⚠️ 必须处理 TCP 粘包（见 8.7）
7. **禁用 Nagle** - ⚠️ 必须设置 TCP_NODELAY（见 8.8）
8. **禁用代理** - ⚠️ 必须禁用系统代理（见 8.9）

> 💡 **提示**：所有 8 种语言的客户端示例库都已内置上述全部功能，可直接使用。

### 8.4 客户端启动服务逻辑

> **重要**：所有客户端示例库都已内置 `ensureServerRunning()` 函数，并在 `connect()` 方法中默认自动调用。通常情况下，你只需调用 `client.connect()` 即可，无需手动处理服务器启动逻辑。

```pseudocode
function EnsureServerRunning():
    // 1. 尝试连接（服务可能已运行）
    socket = TryConnect("127.0.0.1", 5966, timeout=100ms)
    if socket.connected:
        return socket
    
    // 2. 查找 exe 路径
    exePath = FindServerExePath()
    if exePath.error:
        return Error("SuperData Server 未安装")
    
    // 3. 启动服务
    LaunchProcess(exePath, hidden=true)
    
    // 4. 等待端口就绪
    for i in 1..50:  // 最多 5 秒
        sleep(100ms)
        socket = TryConnect("127.0.0.1", 5966, timeout=100ms)
        if socket.connected:
            return socket
    
    return Error("Failed to start server")
```

#### 8.4.1 各语言客户端 API

| 语言 | 导入 | 自动启动 connect |
|------|------|-----------------|
| Python | `from superdata import SuperDataClient, ensure_server_running` | `await client.connect()` |
| JavaScript | `const { SuperDataClient, ensureServerRunning } = require('superdata')` | `await client.connect()` |
| TypeScript | `import { Client, ensureServerRunning } from 'superdata'` | `await client.connect()` |
| C# | `using SuperData;` | `await client.ConnectAsync()` |
| C++ | `#include "ServerLauncher.h"` | `client.Connect()` |
| Rust | `use superdata::{Client, ensure_server_running}` | `client.connect("127.0.0.1", 5966).await` |
| Java | `import com.yunsio.superdata.ServerLauncher;` | `client.connect()` |
| Lua | `local ServerLauncher = require("superdata.server_launcher")` | `client:connect()` |

所有 `connect()` 方法都支持 `autoStartServer` 参数（默认 `true`），如需禁用自动启动：

```python
# Python
await client.connect(auto_start_server=False)
```

```javascript
// JavaScript/TypeScript
await client.connect('127.0.0.1', 5966, false);
```

```csharp
// C#
await client.ConnectAsync("127.0.0.1", 5966, autoStartServer: false);
```

**各平台实现**：

```csharp
// Unity (C#) - 查找并启动
string GetServerPath() {
    using var key = Registry.LocalMachine.OpenSubKey(@"SOFTWARE\Yunsio\SuperData");
    return key?.GetValue("ExePath") as string;
}

void StartServer() {
    var exePath = GetServerPath();
    if (string.IsNullOrEmpty(exePath)) {
        Debug.LogError("SuperData Server 未安装");
        return;
    }
    Process.Start(new ProcessStartInfo {
        FileName = exePath,
        CreateNoWindow = true,
        UseShellExecute = false
    });
}
```

```cpp
// UE (C++) - 查找并启动
FString GetServerPath() {
    FString ExePath;
    FWindowsPlatformMisc::QueryRegKey(
        HKEY_LOCAL_MACHINE,
        TEXT("SOFTWARE\\Yunsio\\SuperData"),
        TEXT("ExePath"),
        ExePath);
    return ExePath;
}

void StartServer() {
    FString ExePath = GetServerPath();
    if (ExePath.IsEmpty()) {
        UE_LOG(LogTemp, Error, TEXT("SuperData Server not installed"));
        return;
    }
    FPlatformProcess::CreateProc(*ExePath, nullptr, true, false, false,
        nullptr, 0, nullptr, nullptr);
}
```

```lua
-- MA2 (Lua) - 查找并启动
function getServerPath()
    local handle = io.popen('reg query "HKLM\\SOFTWARE\\Yunsio\\SuperData" /v ExePath 2>nul')
    local result = handle:read("*a")
    handle:close()
    local path = result:match("ExePath%s+REG_SZ%s+(.+)%s*$")
    return path and path:gsub("%s+$", "") or nil
end

function startServer()
    local exePath = getServerPath()
    if not exePath then
        print("SuperData Server not installed")
        return false
    end
    os.execute('start /B "" "' .. exePath .. '"')
    return true
end
```

### 8.5 伪代码示例

#### 8.5.1 发送数据包

```pseudocode
function SendPacket(socket, packetType, payload):
    // 构建头部
    header = new byte[24]
    header[0:4] = "SPDT"
    header[4:6] = PROTOCOL_VERSION  // Little Endian
    header[6:8] = packetType        // Little Endian
    header[8:12] = sequenceNumber++ // Little Endian
    header[12:20] = currentTimeMs() // Little Endian
    
    payloadBytes = UTF8.Encode(payload)
    header[20:24] = payloadBytes.Length // Little Endian
    
    // 发送
    socket.Send(header + payloadBytes)
```

#### 8.5.2 接收数据包

```pseudocode
function ReceivePacket(socket):
    // 读取头部
    header = socket.Receive(24)
    
    // 验证魔数
    if header[0:4] != "SPDT":
        return Error("Invalid magic")
    
    // 解析头部
    version = ReadUInt16LE(header, 4)
    packetType = ReadUInt16LE(header, 6)
    sequence = ReadUInt32LE(header, 8)
    timestamp = ReadInt64LE(header, 12)
    payloadLength = ReadUInt32LE(header, 20)
    
    // 读取负载
    payload = socket.Receive(payloadLength)
    payloadJson = UTF8.Decode(payload)
    
    return (packetType, payloadJson)
```

#### 8.5.3 客户端完整连接流程

```pseudocode
function ConnectToSuperData():
    // 1. 确保服务运行
    socket = EnsureServerRunning()
    if socket.error:
        return Error("Cannot start server")
    
    // 2. 发送 Connect
    myClientId = GenerateUUID()
    SendPacket(socket, PKT_CONNECT, {
        clientId: myClientId,
        clientName: GetProjectName(),
        platform: PLATFORM_CODE,
        platformVersion: GetPlatformVersion(),
        protocolVersion: "2.0.0"
    })
    
    // 3. 等待 ConnectAck
    response = ReceivePacket(socket)
    if response.type != PKT_CONNECT_ACK:
        return Error("Unexpected response")
    
    if not response.payload.accepted:
        return Error(response.payload.errorMessage)
    
    // 4. 保存群成员列表
    otherClients = response.payload.clients
    
    // 5. 上报自己的灯具数据
    myFixtures = GetAllLocalFixtures()
    SendPacket(socket, PKT_FIXTURE_FULL_SYNC, {
        fixtures: ConvertToStandardCoords(myFixtures),
        totalCount: myFixtures.length
    })
    
    // 6. 启动心跳线程
    StartHeartbeatThread(socket)
    
    return Success(socket, otherClients)
```

### 8.6 测试检查清单

- [ ] 能启动 SuperDataServer.exe
- [ ] 能建立 TCP 连接并完成握手
- [ ] 能正确发送/响应心跳
- [ ] 能上报自己的灯具数据
- [ ] 能获取其他客户端列表
- [ ] 能从指定客户端导入灯具
- [ ] 坐标转换结果正确
- [ ] 能处理网络断开并重连
- [ ] TCP 粘包处理正确
- [ ] 数据立即发送（无延迟）
- [ ] 收到 ClientJoined/ClientLeft 时正确更新 UI
- [ ] 已禁用系统代理（连接不被代理转发）

### 8.7 TCP 粘包处理（⚠️ 重要）

TCP 是流式协议，**多个小数据包可能被合并成一个大包发送**。这是 TCP 的正常行为，但如果不正确处理会导致数据丢失。

#### 8.7.1 问题场景

客户端连续发送两个请求：
```
TypeListRequest (0x0030) + FixtureListRequest (0x0020)
```

由于 TCP 合并，服务端可能在一次 `recv()` 中收到两个请求的数据。如果只解析第一个包，第二个请求将被丢弃！

#### 8.7.2 正确实现

**必须使用循环解析，直到缓冲区中没有完整数据包**：

```pseudocode
pendingBuffer = []  // 待处理缓冲区

function OnDataReceived(newData):
    pendingBuffer.append(newData)
    
    // 循环解析所有完整的数据包
    while pendingBuffer.length >= HEADER_SIZE:
        
        // 检查是否有完整的头部
        if pendingBuffer[0:4] != "SPDT":
            // 无效数据，尝试找到下一个有效的 magic
            findNextMagic()
            continue
        
        payloadLength = ReadUInt32LE(pendingBuffer, 20)
        totalPacketSize = HEADER_SIZE + payloadLength
        
        // 数据不完整，等待更多数据
        if pendingBuffer.length < totalPacketSize:
            break
        
        // 提取完整数据包
        packetData = pendingBuffer[0:totalPacketSize]
        pendingBuffer.removeRange(0, totalPacketSize)
        
        // 处理数据包
        ProcessPacket(packetData)
```

#### 8.7.3 常见错误

❌ **错误做法**：只解析一次
```pseudocode
data = socket.recv()
packet = ParsePacket(data)  // 只解析了第一个包！
HandlePacket(packet)
```

✅ **正确做法**：循环解析
```pseudocode
data = socket.recv()
buffer.append(data)
while (packet = TryParsePacket(buffer)) != null:
    HandlePacket(packet)
```

### 8.8 禁用 Nagle 算法（⚠️ 重要）

Nagle 算法会**缓冲小数据包**，等待更多数据后再发送，这会导致严重的延迟（可能长达数秒甚至数分钟）。

#### 8.8.1 必须设置 TCP_NODELAY

**所有 TCP 连接必须禁用 Nagle 算法**：

```cpp
// C/C++
int flag = 1;
setsockopt(socket, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag));
```

```csharp
// C#
tcpClient.NoDelay = true;
```

```python
# Python
socket.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
```

```lua
-- Lua (LuaSocket)
tcp:setoption("tcp-nodelay", true)
```

#### 8.8.2 不设置 TCP_NODELAY 的后果

| 设置 | 延迟 | 后果 |
|------|------|------|
| ❌ 未设置 | 200ms - 5分钟 | 客户端连接后长时间收不到数据 |
| ✅ 已设置 | < 10ms | 数据立即发送和接收 |

### 8.9 禁用系统代理（⚠️ 重要）

SuperData 使用本地 TCP 连接 (`127.0.0.1:5966`)，**必须确保连接不被系统代理转发**，否则可能导致连接失败或数据异常。

#### 8.9.1 为什么需要禁用代理

- 代理软件（如 Clash、V2Ray、Shadowsocks）可能会拦截本地 TCP 连接
- 即使是 localhost 连接，某些代理配置也会转发
- 转发后可能导致连接超时、数据丢失或协议解析错误

#### 8.9.2 实现方式

**服务端和客户端在启动时都应清除代理环境变量**：

```cpp
// C/C++
#ifdef _WIN32
    _putenv("http_proxy=");
    _putenv("HTTP_PROXY=");
    _putenv("https_proxy=");
    _putenv("HTTPS_PROXY=");
    _putenv("all_proxy=");
    _putenv("ALL_PROXY=");
    _putenv("no_proxy=*");
    _putenv("NO_PROXY=*");
#else
    unsetenv("http_proxy");
    unsetenv("HTTP_PROXY");
    // ... 其他代理变量
    setenv("no_proxy", "*", 1);
#endif
```

```csharp
// C#
string[] proxyVars = { "http_proxy", "HTTP_PROXY", "https_proxy", "HTTPS_PROXY", 
                       "all_proxy", "ALL_PROXY", "socks_proxy", "SOCKS_PROXY" };
foreach (var v in proxyVars)
    Environment.SetEnvironmentVariable(v, null);
Environment.SetEnvironmentVariable("no_proxy", "*");
```

```python
# Python
import os
proxy_vars = ["http_proxy", "HTTP_PROXY", "https_proxy", "HTTPS_PROXY",
              "all_proxy", "ALL_PROXY", "socks_proxy", "SOCKS_PROXY"]
for var in proxy_vars:
    os.environ.pop(var, None)
os.environ["no_proxy"] = "*"
```

```javascript
// JavaScript/Node.js
const proxyVars = ['http_proxy', 'HTTP_PROXY', 'https_proxy', 'HTTPS_PROXY',
                   'all_proxy', 'ALL_PROXY', 'socks_proxy', 'SOCKS_PROXY'];
for (const v of proxyVars)
    delete process.env[v];
process.env.no_proxy = '*';
```

#### 8.9.3 最佳实践

- 在模块/库初始化时立即禁用代理（静态构造函数或模块顶层）
- 设置 `no_proxy=*` 确保所有地址直连
- 服务端和客户端都需要处理

### 8.10 服务端数据缓存

SuperDataServer.exe 会为每个客户端维护独立的数据缓存：

```pseudocode
class Server:
    // 按客户端ID索引的数据缓存
    clientDataCache = Map<clientId, FixtureList>
    
    function OnFixtureFullSync(clientId, fixtures):
        // 完全替换该客户端的缓存
        clientDataCache[clientId] = fixtures
        UpdateClientFixtureCount(clientId, fixtures.length)
    
    function OnFixtureUpdate(clientId, uuid, changes):
        // 更新缓存中的单个灯具
        fixture = clientDataCache[clientId].find(uuid)
        fixture.applyChanges(changes)
        // 广播给其他客户端
        BroadcastToOthers(clientId, FixtureUpdate, {uuid, changes})
    
    function HandleFixtureListRequest(requesterId, sourceClientId):
        // 返回指定客户端的缓存数据
        fixtures = clientDataCache[sourceClientId]
        SendTo(requesterId, FixtureListResponse, {
            sourceClientId: sourceClientId,
            fixtures: fixtures,
            totalCount: fixtures.length
        })
    
    function OnClientDisconnect(clientId):
        // 清除该客户端的缓存
        clientDataCache.remove(clientId)
        // 广播客户端离开
        BroadcastToAll(ClientLeft, {clientId})
        // 检查是否需要退出
        if clientDataCache.isEmpty():
            ExitProcess()
```

---

## 9. 安全考虑

### 9.1 当前限制

当前版本（v1.0）不包含加密，仅适用于：
- 本地网络环境
- 受信任的内网环境

### 9.2 未来计划

后续版本可能添加：
- TLS 加密传输
- 令牌认证机制
- 权限控制系统

---

## 10. 故障排除

### 10.1 服务无法启动

#### 10.1.1 SuperData Server 未安装

如果客户端提示"SuperData Server 未安装"：

1. **下载安装包**：https://yunsio.com/superdata/download
2. **运行安装程序**：`SuperDataSetup.exe`
3. **验证安装**：
```powershell
reg query "HKLM\SOFTWARE\Yunsio\SuperData" /v ExePath
```

#### 10.1.2 注册表路径不正确

手动检查注册表：
```powershell
# 检查注册表
reg query "HKLM\SOFTWARE\Yunsio\SuperData"

# 如果需要手动修复（管理员权限）
reg add "HKLM\SOFTWARE\Yunsio\SuperData" /v ExePath /t REG_SZ /d "C:\Program Files\Yunsio\SuperData\SuperDataServer.exe" /f
```

#### 10.1.3 端口被占用

检查 TCP 5966 端口是否被占用：
```powershell
netstat -ano | findstr :5966
```

如果被占用，结束占用进程或等待其释放。

#### 10.1.4 防病毒软件阻止

某些防病毒软件可能阻止启动外部 exe，尝试添加信任或临时禁用。

### 10.2 连接建立失败

#### 10.2.1 服务未运行

确认 SuperDataServer.exe 是否正在运行：
```powershell
tasklist | findstr SuperDataServer
```

#### 10.2.2 防火墙阻止

如果需要远程连接（非 localhost），确保防火墙允许 TCP 5966：
```powershell
# 以管理员身份运行
netsh advfirewall firewall add rule name="SuperData TCP" dir=in action=allow protocol=TCP localport=5966
```

### 10.3 数据同步问题

#### 10.3.1 坐标不正确

确保各端正确实现了坐标系转换：
- 检查转换方向是否正确（导入/导出）
- 检查单位是否正确转换（UE使用厘米）

#### 10.3.2 JSON 解析错误

- 确保 JSON 使用 UTF-8 编码
- 检查特殊字符是否正确转义

### 10.4 数据接收延迟（常见问题）

#### 10.4.1 症状

- 连接成功，但等待很长时间（数秒甚至数分钟）才收到数据
- 只收到部分响应，其他响应丢失
- 数据偶尔正常，偶尔丢失

#### 10.4.2 原因与解决

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 数据延迟数秒/分钟 | 未禁用 Nagle 算法 | 设置 `TCP_NODELAY = true`（见 8.8）|
| 只收到第一个响应 | 未处理 TCP 粘包 | 循环解析缓冲区（见 8.7）|
| 连接超时/数据异常 | 被系统代理拦截 | 禁用系统代理（见 8.9）|

#### 10.4.3 快速诊断

1. **检查延迟**：发送请求后，记录收到响应的时间
   - < 100ms → 正常
   - > 1s → 可能未设置 TCP_NODELAY

2. **检查丢包**：连续发送多个请求，检查响应数量
   - 请求数 = 响应数 → 正常
   - 请求数 > 响应数 → 可能未处理粘包

### 10.5 代理导致的连接问题

#### 10.5.1 症状

- 连接超时或连接被拒绝
- 连接成功但无法收发数据
- 数据包格式错误（代理修改了数据）

#### 10.5.2 原因

系统代理（Clash、V2Ray 等）拦截了 localhost 连接。

#### 10.5.3 解决方案

1. **代码层面**：确保启动时清除代理环境变量（见 8.9）
2. **系统层面**：在代理软件中将 `127.0.0.1` 和 `localhost` 添加到绕过列表
3. **临时测试**：关闭代理软件后测试

### 10.6 客户端列表问题

#### 10.5.1 看不到其他客户端

- 确认其他客户端已成功连接（检查 ConnectAck.accepted = true）
- 确认其他客户端已发送 FixtureFullSync
- 检查是否正确处理 ClientJoined 消息

#### 10.5.2 客户端灯具数量为0

- 客户端连接后需要发送 FixtureFullSync 上报自己的数据
- 如果本地没有灯具，发送空列表即可

---

## 11. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 1.0 | 2025 | 初始版本 |
| 1.0.1 | 2025-11 | 添加 TCP 粘包处理说明 |
| 1.0.1 | 2025-11 | 添加 TCP_NODELAY 要求 |
| 1.0.1 | 2025-11 | 标准坐标系单位改为厘米（cm）|
| 1.0.2 | 2025-11 | 修正旋转转换公式（手性问题）|
| 1.0.3 | 2025-11 | 添加 GrandMA2 坐标转换公式 |
| **2.0.0** | **2025-12-01** | **🎉 正式发布：群聊架构** |
| 2.0.0 | 2025-12-01 | 引入独立服务进程 SuperDataServer.exe |
| 2.0.0 | 2025-12-01 | 移除 UDP 服务发现（0x0001-0x0003）|
| 2.0.0 | 2025-12-01 | 移除类型映射包（0x0030-0x0032）|
| 2.0.0 | 2025-12-01 | 新增 ClientJoined/ClientLeft（0x0014-0x0015）|
| 2.0.0 | 2025-12-01 | 新增 FixtureFullSync（0x0023）|
| 2.0.0 | 2025-12-01 | 简化客户端实现，协议层统一由 Server 处理 |
| 2.0.0 | 2025-12-01 | 系统级安装包（Inno Setup），支持中英文 |
| 2.0.0 | 2025-12-01 | 客户端自动启动服务器|
| 2.0.0 | 2025-12-01 | 提供 8 种语言客户端示例（Python/JS/TS/C#/C++/Rust/Java/Lua）|

---

## 附录 A: 常量定义

```c
// 协议常量
#define SUPERDATA_MAGIC         "SPDT"
#define SUPERDATA_VERSION       2
#define SUPERDATA_DATA_PORT     5966
#define SUPERDATA_HEARTBEAT_MS  3000
#define SUPERDATA_TIMEOUT_MS    10000
#define SUPERDATA_MAX_PACKET    65536

// 平台代码
#define PLATFORM_UNKNOWN        0
#define PLATFORM_UNITY          1
#define PLATFORM_UNREAL         2
#define PLATFORM_VECTORWORKS    3
#define PLATFORM_GRANDMA        4
#define PLATFORM_CUSTOM         255

// 数据包类型 - 连接管理
#define PKT_CONNECT             0x0010
#define PKT_CONNECT_ACK         0x0011
#define PKT_DISCONNECT          0x0012
#define PKT_HEARTBEAT           0x0013

// 数据包类型 - 客户端管理
#define PKT_CLIENT_JOINED       0x0014
#define PKT_CLIENT_LEFT         0x0015

// 数据包类型 - 数据同步
#define PKT_FIXTURE_LIST_REQ    0x0020
#define PKT_FIXTURE_LIST_RESP   0x0021
#define PKT_FIXTURE_UPDATE      0x0022
#define PKT_FIXTURE_FULL_SYNC   0x0023
#define PKT_FIXTURE_DELETE      0x0024

// 数据包类型 - 错误
#define PKT_ERROR               0xFF00
```

---

## 附录 B: JSON Schema

### B.1 Fixture Schema

```json
{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "object",
    "required": ["uuid", "name", "fixtureType", "universe", "startAddress", "fixtureID", "position", "rotation"],
    "properties": {
        "uuid": { "type": "string", "description": "客户端生成的UUID" },
        "name": { "type": "string" },
        "fixtureType": { "type": "string", "description": "客户端自定义的类型名" },
        "universe": { "type": "integer", "minimum": 1, "maximum": 256 },
        "startAddress": { "type": "integer", "minimum": 1, "maximum": 512 },
        "fixtureID": { "type": "integer", "minimum": 1 },
        "position": { "type": "array", "items": { "type": "number" }, "minItems": 3, "maxItems": 3, "description": "标准坐标系，单位cm" },
        "rotation": { "type": "array", "items": { "type": "number" }, "minItems": 3, "maxItems": 3, "description": "欧拉角，单位度" },
        "scale": { "type": "array", "items": { "type": "number" }, "minItems": 3, "maxItems": 3 },
        "channelSpan": { "type": "integer" },
        "customProperties": { "type": "object" }
    }
}
```

### B.2 ClientInfo Schema

```json
{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "object",
    "required": ["clientId", "clientName", "platform"],
    "properties": {
        "clientId": { "type": "string", "description": "客户端唯一标识符(UUID)" },
        "clientName": { "type": "string", "description": "客户端显示名称" },
        "platform": { "type": "integer", "description": "平台代码" },
        "fixtureCount": { "type": "integer", "description": "当前灯具数量" }
    }
}
```

---

**文档结束**

---

## 许可证

[![License: CC BY-NC 4.0](https://img.shields.io/badge/License-CC%20BY--NC%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-nc/4.0/)

本项目采用 [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/)（知识共享-署名-非商业性使用 4.0 国际）许可协议。

- ✅ 可自由共享、修改、演绎
- ✅ 需保留署名
- ❌ 禁止商业使用

详细条款请参阅 [LICENSE](LICENSE)（[English](LICENSE_EN)）| 商业授权请联系 [Yunsio](https://yunsio.com)

---

<p align="center">
  <b>SuperData Protocol v2.0.0</b><br>
  跨平台舞台灯具数据同步协议<br><br>
  © 2025 Yunsio SuperStage Team. All rights reserved.
</p>


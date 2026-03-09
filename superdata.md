# SuperData 协议文档

## 概述

SuperData 是 SuperStage 生态的开源跨平台数据同步协议，用于在不同灯光设计软件之间实时同步灯具参数、位置、编组等数据。协议基于 TCP 长连接，默认端口 **5966**，数据格式为 JSON。

## 设计目标

- **零门槛接入**：8 语言 SDK 全部开源（MIT 协议）
- **实时同步**：灯具参数变更 < 16ms 传输延迟
- **双向通信**：任意端既可以是 Server 也可以是 Client
- **可扩展**：自定义属性字段，向后兼容

## 支持平台

| 平台 | 角色 | 说明 |
|------|------|------|
| Unreal Engine 5 (SuperStage) | Server / Client | 原生 C++ 实现 |
| Vectorworks | Client | Python 插件 |
| grandMA2 / MA3 | Client | Lua 插件 |
| Unity | Client | C# SDK |
| TouchDesigner | Client | Python SDK |
| Web Browser | Client | TypeScript SDK |

## 协议架构

### 连接模型

```
┌──────────────┐     TCP 5966      ┌──────────────┐
│  SuperStage  │◄──────────────────►│   MA3 Plugin │
│   (Server)   │                   │   (Client)   │
└──────────────┘                   └──────────────┘
       ▲
       │  TCP 5966
       ▼
┌──────────────┐
│  Vectorworks │
│   (Client)   │
└──────────────┘
```

### 消息格式

所有消息采用 **长度前缀 + JSON** 格式：

```
[4 bytes: payload length (big-endian)] [N bytes: JSON payload]
```

### 消息类型

| 类型 | 方向 | 说明 |
|------|------|------|
| `FIXTURE_SYNC` | 双向 | 灯具参数同步 |
| `FIXTURE_LIST` | Server → Client | 灯具列表推送 |
| `GROUP_SYNC` | 双向 | 编组数据同步 |
| `POSITION_SYNC` | 双向 | 灯具位置同步 |
| `HEARTBEAT` | 双向 | 心跳保活 |
| `AUTH` | Client → Server | 认证握手 |

## 数据结构

### Fixture 灯具对象

```json
{
    "id": "fixture-001",
    "name": "Beam 230",
    "manufacturer": "Example",
    "model": "Beam 230W",
    "mode": "Standard",
    "universe": 1,
    "address": 1,
    "channels": 16,
    "position": { "x": 0.0, "y": 5.0, "z": 3.0 },
    "rotation": { "pitch": -45.0, "yaw": 0.0, "roll": 0.0 },
    "attributes": {
        "dimmer": 255,
        "pan": 32768,
        "tilt": 32768,
        "color_r": 255,
        "color_g": 200,
        "color_b": 100
    }
}
```

### Group 编组对象

```json
{
    "id": "group-01",
    "name": "Front Wash",
    "fixture_ids": ["fixture-001", "fixture-002", "fixture-003"],
    "color": "#00BFFF"
}
```

## SDK 使用

### Python

```python
from superdata import SuperDataClient

client = SuperDataClient("192.168.1.100", 5966)
client.connect()

# 获取灯具列表
fixtures = client.get_fixtures()

# 监听灯具变更
@client.on("fixture_sync")
def on_fixture_update(fixture):
    print(f"Fixture {fixture.name} updated: dimmer={fixture.attributes.dimmer}")

client.run()
```

### TypeScript

```typescript
import { SuperDataClient } from '@superstage/superdata'

const client = new SuperDataClient('192.168.1.100', 5966)
await client.connect()

// 获取灯具列表
const fixtures = await client.getFixtures()

// 监听灯具变更
client.on('fixtureSync', (fixture) => {
    console.log(`Fixture ${fixture.name} updated`)
})
```

### C# (Unity)

```csharp
using SuperData;

var client = new SuperDataClient("192.168.1.100", 5966);
await client.ConnectAsync();

// 获取灯具列表
var fixtures = await client.GetFixturesAsync();

// 监听灯具变更
client.OnFixtureSync += (fixture) => {
    Debug.Log($"Fixture {fixture.Name} updated");
};
```

## 错误处理

### 错误码

| 代码 | 说明 |
|------|------|
| `1001` | 认证失败 |
| `1002` | 灯具不存在 |
| `1003` | 地址冲突 |
| `1004` | 域超出范围 |
| `2001` | 服务器内部错误 |
| `2002` | 消息格式错误 |

### 重连策略

SDK 内置指数退避重连机制：

- 首次重连间隔：1 秒
- 最大重连间隔：30 秒
- 退避系数：2x
- 最大重试次数：无限制

## 安全性

### 认证

连接建立后，客户端必须发送 `AUTH` 消息完成握手：

```json
{
    "type": "AUTH",
    "token": "your-api-token",
    "client_name": "MA3 Plugin",
    "client_version": "1.0.0"
}
```

### 传输加密

生产环境建议使用 TLS 加密。SDK 支持 `tls: true` 配置项。

## 开源仓库

SuperData SDK 全部开源，采用 MIT 协议：

- **Python SDK**: `pip install superdata`
- **TypeScript SDK**: `npm install @superstage/superdata`
- **C# SDK**: NuGet `SuperData`
- **C++ SDK**: 包含在 SuperStage 插件中
- **Lua SDK**: grandMA2/MA3 插件包
- **Go SDK**: `go get github.com/superstage/superdata-go`
- **Rust SDK**: `cargo add superdata`
- **Java SDK**: Maven `com.superstage:superdata`

## 更新日志

请参阅 [合作与生态](/ecosystem) 页面了解更多生态信息。

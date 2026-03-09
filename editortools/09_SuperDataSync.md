# SuperStage SuperData 网络同步 用户手册

## 1. 概述

SuperData 同步面板（SuperData Sync v2.0）采用**群聊架构**，允许多台运行不同灯光软件的设备通过 SuperDataServer 交换灯具数据。你可以从其他客户端（Unity、MA3、Capture 等）获取灯具配置，经过类型映射后一键导入到当前 UE 场景中。

---

## 2. 打开方式

**主菜单路径**：工具栏 **SuperStage** 下拉菜单 → **SuperDMXTool** → **SuperDataImport**

打开后面板最小尺寸为 580 × 520 像素。

---

## 3. 界面说明

面板从上到下分为五个区域：

```
┌──────────────────────────────────────────────────┐
│  SuperData Sync  v2.0                            │
├──────────────────────────────────────────────────┤
│  ● Connected           12 local fixtures [Connect]│
├──────────────────────────────────────────────────┤
│  Remote Clients (2)                   [Fetch Data]│
│  ┌──────────────────────────────────────────┐    │
│  │ Client         │ Platform  │ Fixtures    │    │
│  │ MA3-Console    │ grandMA3  │ 128         │    │
│  │ Capture-Laptop │ Capture   │ 64          │    │
│  └──────────────────────────────────────────┘    │
├──────────────────────────────────────────────────┤
│  Type Mapping  3/5 selected         [All] [None] │
│  ┌──────────────────────────────────────────┐    │
│  │ ☑│ Remote Type  │ Count │ Local Class    │    │
│  │ ☑│ Spot380      │ 24    │ BP_Spot380  ▼  │    │
│  │ ☑│ WashLED      │ 16    │ BP_WashLED  ▼  │    │
│  │ ☐│ Unknown      │ 8     │ (None)      ▼  │    │
│  └──────────────────────────────────────────┘    │
├──────────────────────────────────────────────────┤
│  40 of 48 fixtures ready to import    [Import to │
│                                         Scene]   │
└──────────────────────────────────────────────────┘
```

---

## 4. 连接区域

### 4.1 连接按钮

面板右上角有一个 **Connect / Disconnect** 切换按钮：

| 状态 | 按钮文字 | 操作 |
|------|----------|------|
| 未连接 | **Connect** | 连接到 SuperDataServer（如未启动会自动启动） |
| 已连接 | **Disconnect** | 断开连接 |

> **自动启动**：点击 Connect 时，如果 SuperDataServer 未运行，插件会自动启动它。

### 4.2 状态指示

按钮左侧有一个圆形状态指示器和状态文字：

| 指示器颜色 | 状态文字 | 说明 |
|-----------|----------|------|
| 🟢 绿色 | **Connected** | 已成功连接到服务器 |
| 🔴 红色 | **Disconnected** | 未连接 |
| 🟡 橙色 | **Connecting...** | 正在连接中 |

### 4.3 本地灯具计数

状态栏中还会显示当前场景中 SuperStage DMX 灯具的数量，格式为 `{N} local fixtures`。

---

## 5. 远程客户端列表 (Remote Clients)

显示当前连接到同一 SuperDataServer 的**其他客户端**（不包含本机）。

### 5.1 列表字段

| 列名 | 说明 |
|------|------|
| **Client** | 客户端名称（Tooltip 显示 Client ID） |
| **Platform** | 客户端所在平台（如 grandMA3、Capture、Unity 等） |
| **Fixtures** | 该客户端拥有的灯具数量 |

### 5.2 操作

- **选中客户端**：在列表中单击选中一个客户端作为数据源
- **Fetch Data**：点击右上角按钮，向选中的客户端请求灯具数据

> 当无客户端连接时，列表区域显示提示："Click 'Connect' to join the SuperData network"  
> 已连接但无其他客户端时显示："Waiting for other clients to connect..."

---

## 6. 类型映射 (Type Mapping)

当从远程客户端获取到灯具数据后，Type Mapping 区域会列出所有接收到的灯具类型，供你进行本地映射。

### 6.1 列表字段

| 列名 | 说明 |
|------|------|
| **☑** | 勾选框，是否导入该类型的灯具 |
| **Remote Type** | 远程客户端中的灯具类型名称 |
| **Count** | 该类型的灯具数量 |
| **Local Actor Class** | 下拉选择本地对应的 SuperStage 灯具蓝图类 |

### 6.2 批量选择

| 按钮 | 功能 |
|------|------|
| **All** | 全选所有灯具类型 |
| **None** | 取消全选 |

标题行旁显示 `{已选}/{总数} selected` 计数。

> 当尚未获取数据时，区域显示提示："No fixture data received yet. Select a client above and click 'Fetch Data'."

---

## 7. 底部操作区域

### 7.1 导入计数

左侧显示实时统计：`{可导入数} of {总数} fixtures ready to import`

只有勾选了类型**且**指定了 Local Actor Class 的灯具才计入可导入数。

### 7.2 Import to Scene 按钮

点击 **"Import to Scene"** 按钮后：

1. 遍历所有勾选且有映射的灯具类型
2. 为每个灯具在场景中创建对应的 Actor
3. 设置灯具的位置、旋转、DMX 地址等属性
4. 导入操作在一个 UE Transaction 中执行，支持 **Ctrl+Z 撤销**
5. 完成后通过 UE 通知弹窗显示导入结果

按钮仅在以下条件全部满足时可用：
- 已获取灯具数据（`PendingFixtures` 非空）
- 至少有一个类型被勾选且指定了本地 Actor Class

---

## 8. 使用流程

### 场景 1：从其他软件导入灯具

1. 打开 SuperData 同步面板
2. 点击 **"Connect"** 连接到 SuperDataServer
3. 等待其他客户端（如 Unity、MA3）加入
4. 在 Remote Clients 列表中选中数据源客户端
5. 点击 **"Fetch Data"** 获取灯具数据
6. 在 Type Mapping 区域中为每种灯具类型选择本地 Actor Class
7. 勾选要导入的灯具类型
8. 点击 **"Import to Scene"** 导入到场景

### 场景 2：多人协作

1. 灯光设计师在 MA3/Capture 中完成灯具布局
2. UE 操作员打开 SuperData 面板并连接
3. 选中设计师的客户端，Fetch Data
4. 映射灯具类型后导入场景
5. 导入后可使用 SuperStage 的批量 Patch 工具进一步调整

---

## 9. 注意事项

- SuperDataServer 是独立的可执行程序，连接时会自动启动
- 所有设备必须在同一局域网中
- 确保防火墙允许 SuperDataServer 的网络通信
- 导入操作会在场景中创建新的 Actor（不会更新已有 Actor）
- 导入操作支持 Ctrl+Z 撤销
- 面板关闭时会自动断开连接并清理资源

---

## 10. 故障排查

| 问题 | 可能原因 | 解决方法 |
|------|----------|----------|
| 无法连接 | SuperDataServer 未启动 | 点击 Connect 会自动启动，若失败请手动启动 |
| Remote Clients 为空 | 其他客户端未连接 | 确认其他软件已打开 SuperData 面板并连接 |
| Fetch Data 无响应 | 未选中客户端 | 先在列表中选中一个客户端 |
| 导入按钮不可用 | 未指定 Local Actor Class | 在 Type Mapping 中为至少一个类型选择本地蓝图类 |
| 灯具位置不对 | 坐标系差异 | 检查源客户端的坐标单位和轴向 |
| 灯具类型无法映射 | 本地灯库缺少对应蓝图 | 先在 SuperStage 灯库中创建对应的灯具蓝图类 |

# SuperDroneLink 用户手册

**版本：** v1.0 | **最后更新：** 2026-03-04 | **适用版本：** SuperStage 26Q1.2+ | **作者：** LimxTeam

---

## 目录

- [第一章 概述](#第一章-概述)
- [第二章 快速入门](#第二章-快速入门)
- [第三章 网络配置详解](#第三章-网络配置详解)
- [第四章 地理锚点系统](#第四章-地理锚点系统)
- [第五章 DroneSwarmManager 编队管理器](#第五章-droneswarmmanager-编队管理器)
- [第六章 SuperDroneActor 单架无人机](#第六章-superdroneactor-单架无人机)
- [第七章 Sequencer 时间轴集成](#第七章-sequencer-时间轴集成)
- [第八章 LDLink 协议说明](#第八章-ldlink-协议说明)
- [第九章 无人机状态与编号](#第九章-无人机状态与编号)
- [第十章 坐标系统与转换](#第十章-坐标系统与转换)
- [第十一章 蓝图节点参考](#第十一章-蓝图节点参考)
- [第十二章 常见问题与故障排除](#第十二章-常见问题与故障排除)
- [附录 A 术语表](#附录-a-术语表)
- [附录 B 性能参考指标](#附录-b-性能参考指标)

---

# 第一章 概述

## 1.1 什么是 SuperDroneLink

**SuperDroneLink** 是 SuperStage 插件中的无人机编队实时可视化模块。它接收来自无人机编队设计软件（如 LimxDroneStudio）的实时数据，在 Unreal Engine 中以三维方式呈现成千上万架无人机的位置、姿态和 LED 灯光颜色，实现无人机表演的「数字孪生」预演。

**简单来说：** 编队设计软件负责「编排」，SuperDroneLink 负责「实时 3D 预览」。

### 核心能力

| 能力 | 说明 |
|------|------|
| 实时数据接收 | 通过 UDP 网络接收 LDLink 协议数据，延迟极低 |
| 大规模渲染 | 支持 10,000+ 架无人机同屏显示，保持 60FPS+ |
| LED 灯光同步 | 实时同步每架无人机的 RGB 颜色和亮度 |
| 位置平滑插值 | 网络数据 10-25Hz 也能保证画面丝滑 |
| GPS 坐标转换 | 自动将 GPS 经纬度转换为 UE 三维坐标 |
| Sequencer 集成 | 在时间轴中录制和回放无人机轨迹 |
| 蓝图支持 | 所有核心功能均可在蓝图中调用 |

## 1.2 应用场景

- **无人机表演预演：** 将编队路径通过 LDLink 协议发送到 SuperStage，实时预览飞行轨迹和灯光效果
- **灯光+无人机联合编排：** 同时显示舞台灯光和无人机编队，在统一 3D 环境中审查整体视觉效果
- **方案评审展示：** 向客户实时播放无人机表演效果，支持自由旋转视角
- **Sequencer 离线编排：** 在 UE 时间轴中直接编辑无人机位置、姿态和 LED 曲线

## 1.3 系统要求

| 项目 | 要求 |
|------|------|
| Unreal Engine | 5.3 或更高版本 |
| SuperStage 插件 | 26Q1.2 或更高版本 |
| 操作系统 | Windows 10/11（64位） |
| 网络 | UDP 端口 14555 可用（可修改） |
| 显卡 | 1000 架建议 GTX 1060+；10000 架建议 RTX 3060+ |
| 外部软件（可选） | LimxDroneStudio 或其他支持 LDLink 协议的地面站 |

## 1.4 核心概念

### DroneLink 子系统
整个模块的「大脑」，随 UE 引擎启动自动运行，全局唯一。负责监听网络端口、管理无人机状态、提供数据查询接口。**无需手动创建。**

### 编队管理器（DroneSwarmManager）
放置在场景中的 Actor，使用 HISM 技术高性能渲染无人机编队。**适合 100-10000+ 架的大规模编队。**

### 单架无人机 Actor（SuperDroneActor）
独立 Actor，绑定到特定无人机 ID 并跟随其移动。**适合需要挂载摄像机、特效等的少量无人机。**

### 地理锚点（GeoAnchor）
GPS 到 UE 坐标的转换原点。第一架连接的无人机 GPS 位置会自动成为锚点（UE 原点 0,0,0）。

### LDLink 协议
专有通信协议，通过 UDP 传输无人机的位置和 LED 数据。

### DroneId
每架无人机的唯一标识号（0-65535）。

---

# 第二章 快速入门

## 2.1 五分钟上手

### 第 1 步：确认插件已启用
Edit > Plugins，搜索 SuperStage，确认已启用并重启编辑器。SuperDroneLink 作为子模块自动加载。

### 第 2 步：放置编队管理器
在 Super浏览器 面板搜索 `DroneSwarmManager`，拖拽到关卡中。

### 第 3 步：配置（可选）
选中 DroneSwarmManager，在 Details 面板中可调整：
- **Auto Sync** = 开启（默认）
- **Drone Scale** = 50（默认，控制显示大小）
- **Interp Speed** = 0.9（默认，控制平滑度）

### 第 4 步：在 LimxDroneStudio 中配置输出
- 目标 IP：运行 UE 的电脑 IP（本机填 `127.0.0.1`）
- 目标端口：`14555`
- 协议：LDLink
- 点击开始输出

### 第 5 步：查看效果
在 LimxDroneStudio 中播放编队动画，UE 编辑器中会立即看到无人机出现并移动。

> **重要：** 无需按 UE 的 Play 按钮，编辑器模式下即可实时显示。

## 2.2 连接方式

| 方式 | 目标 IP 设置 |
|------|-------------|
| 同一台电脑 | `127.0.0.1` |
| 局域网两台电脑 | UE 电脑的局域网 IP |
| 跨网段 | 需放行 UDP 14555 防火墙规则 |

---

# 第三章 网络配置详解

默认配置适合大多数场景，特殊网络环境下可能需要调整。

## 3.1 配置参数一览

| 参数 | 类别 | 默认值 | 范围 | 说明 |
|------|------|--------|------|------|
| Port | Network | 14555 | 1-65535 | UDP 监听端口 |
| BindIP | Network | 空（所有接口） | 合法IP | 本地绑定 IP |
| bEnableInput | Network | 关闭 | 开/关 | 启用数据接收（预留） |
| bEnableOutput | Network | 关闭 | 开/关 | 启用数据发送（预留） |
| ProtocolVersion | Protocol | MAVLink 2.0 | V1/V2/None | 协议版本（预留） |
| HeartbeatTimeoutSeconds | Protocol | 5.0 | 1.0-30.0 | 心跳超时（秒） |
| PositionInterpSpeed | Rendering | 0.8 | 0.0-1.0 | 位置插值平滑系数 |

## 3.2 监听端口（Port）

默认 14555。修改场景：端口被占用、与外部软件端口不同、防火墙限制。

通过蓝图 `ApplyConfig` 节点修改，会自动重启接收器。建议使用 1024 以上端口号。

## 3.3 绑定 IP（BindIP）

| 场景 | 设置 | 说明 |
|------|------|------|
| 通用（推荐） | 留空 | 接受所有网卡数据 |
| 多网卡电脑 | `192.168.1.100` | 只接受指定网卡数据 |
| 仅本机调试 | `127.0.0.1` | 只接受本机数据 |

填写不存在的 IP 会导致接收器启动失败。

## 3.4 协议版本（ProtocolVersion）

当前版本主要使用 LDLink 协议，此参数为未来 MAVLink 原生支持预留。保持默认即可。

## 3.5 心跳超时（HeartbeatTimeoutSeconds）

无人机在此时间内无任何数据则被判定离线并自动移除。

| 网络环境 | 建议值 |
|----------|--------|
| 本机/局域网 | 3.0 - 5.0 |
| 跨网段/WiFi | 8.0 - 15.0 |
| 大型演出现场 | 10.0 - 20.0 |

超时后无人机从状态表移除，重新收到数据会自动重新添加。

## 3.6 位置插值平滑系数（PositionInterpSpeed）

| 值 | 效果 |
|-----|------|
| 0.0 | 无平滑，直接跳到最新位置 |
| 0.5 | 中等平滑 |
| 0.8（默认） | 高度平滑，适合大多数预览 |
| 1.0 | 最大平滑（可能有延迟感） |

DroneSwarmManager 和 SuperDroneActor 各有自己的插值参数，效果叠加。

## 3.7 输入/输出开关

bEnableInput 和 bEnableOutput 均为未来扩展预留，当前版本无需修改。LDLink 接收功能独立于这两个开关。

## 3.8 应用配置

子系统随引擎启动**自动监听** 14555 端口，通常无需额外操作。

如需修改：蓝图中获取 DroneLinkSubsystem > 创建 FDroneLinkConfig 结构体 > 调用 ApplyConfig。调用后接收器会短暂重启（约 100ms 中断）。

---

# 第四章 地理锚点系统

## 4.1 什么是地理锚点

GPS 坐标与 UE 坐标之间的「桥梁」。指定一个 GPS 坐标作为 UE 原点 (0,0,0)，所有无人机的 GPS 坐标都转换为相对偏移（厘米）。

**举例：** 锚点设为纬度 39.9042、经度 116.4074、海拔 50m。某无人机在锚点正北 100m、正东 50m、高 200m，其 UE 坐标约为 X=10000, Y=5000, Z=15000（厘米）。

## 4.2 自动锚点

默认行为：收到第一条有效 GPS 坐标时自动设为锚点。条件：锚点未设置 + 纬度非零 + 经度非零。

如果发送的是**本地坐标**（非 GPS），不会触发自动锚点，坐标直接使用。

## 4.3 手动设置锚点

蓝图调用 `SetGeoAnchor`，传入：
- **Latitude（纬度）：** 度，如 39.9042
- **Longitude（经度）：** 度，如 116.4074
- **Altitude（海拔）：** 米（MSL），如 50.0

设置后所有已连接无人机的坐标会立即重算。

## 4.4 使用第一架无人机作为锚点

蓝图调用 `SetGeoAnchorFromFirstDrone`，自动以 DroneId 最小的无人机 GPS 位置为锚点。

## 4.5 重置锚点

蓝图调用 `ResetGeoAnchor`，清除锚点。下一条 GPS 数据会重新触发自动设置。

## 4.6 锚点参数

| 参数 | 类型 | 说明 |
|------|------|------|
| Latitude | 双精度浮点 | WGS84 纬度，度。北纬正，南纬负。-90~+90 |
| Longitude | 双精度浮点 | WGS84 经度，度。东经正，西经负。-180~+180 |
| Altitude | 双精度浮点 | MSL 海拔，米 |
| bIsSet | 布尔（只读） | 锚点是否已设置 |

---

# 第五章 DroneSwarmManager 编队管理器

## 5.1 功能简介

使用 HISM（层级实例化静态网格）技术大规模渲染无人机编队。**100 架以上编队首选。**

特点：单次 DrawCall、GPU 驱动颜色、平滑插值、脏检测优化、编辑器实时预览。

## 5.2 放置到场景

在 Super浏览器 面板搜索 `DroneSwarmManager` 拖拽到视口中。

放置位置意义：
- LDLink 本地坐标：无人机坐标 = 管理器位置 + 接收偏移
- GPS 坐标：由锚点转换，与管理器位置无关

## 5.3 配置属性详解

### Auto Sync（自动同步）
- 类别：DroneSwarm > Config
- 默认：开启
- 说明：开启后自动与 DroneLinkSubsystem 同步。关闭时需蓝图手动调用 StartSync。

### Drone Scale（无人机缩放）
- 类别：DroneSwarm > Config
- 默认：50.0
- 说明：显示大小。值除以 100 作为缩放因子。

| 值 | 缩放 | 适用 |
|----|------|------|
| 10 | 0.1x | 远距离俯瞰 |
| 25 | 0.25x | 中远距离 |
| 50（默认） | 0.5x | 常规预览 |
| 100 | 1.0x | 近距离检查 |
| 200 | 2.0x | 特写展示 |

### Interp Speed（插值速度）
- 类别：DroneSwarm > Config
- 默认：0.9
- 范围：0.0 - 1.0
- 说明：位置平滑过渡速度。内部以 30fps 频率更新。

| 值 | 效果 |
|----|------|
| 0.0 | 无平滑，直接跳 |
| 0.3 | 轻度平滑 |
| 0.9（默认） | 高度平滑 |
| 1.0 | 最大平滑 |

### Drone Timeout Seconds（超时时间）
- 类别：DroneSwarm > Config
- 默认：5.0
- 范围：1.0 - 60.0
- 说明：无数据超时后隐藏渲染实例

### Preallocated Instance Count（预分配实例数）
- 类别：DroneSwarm > Config
- 默认：1000
- 范围：0 - 10000
- 说明：预创建的渲染实例，避免首次出现卡顿

| 预期数量 | 建议值 |
|----------|--------|
| 1-100 | 100 |
| 100-500 | 500 |
| 500-2000 | 1000 |
| 2000-5000 | 3000 |
| 5000-10000 | 5000 |

## 5.4 视觉属性

### Drone Mesh（网格体）
- 类别：DroneSwarm > Visual
- 默认：引擎内置球体
- 建议：大规模编队用低面数模型（<100 三角面）

### Drone Material（材质）
- 类别：DroneSwarm > Visual
- 默认：MI_DroneLed_Inst（内置 LED 材质）
- 要求：必须支持 PerInstanceCustomData

CustomData 通道分配：

| 通道 | 数据 | 范围 |
|------|------|------|
| [0] | Red | 0-1 |
| [1] | Green | 0-1 |
| [2] | Blue | 0-1 |
| [3] | Brightness | 0-1 |

自定义材质需读取这 4 个通道用于颜色输出。

## 5.5 同步机制

### 编辑器模式
构造时：清除旧实例 > 连接子系统 > 订阅事件 > 同步已有无人机。
每帧（30fps）：获取状态 > 插值 > 脏检测 > 批量 GPU 更新。

### PIE 模式
BeginPlay：重新获取子系统 > 重新订阅事件 > 刷新实例。Tick 行为同上。

## 5.6 蓝图控制节点

| 节点 | 功能 |
|------|------|
| Start Sync | 开始同步 |
| Stop Sync | 停止同步 |
| Refresh All Instances | 强制重建所有实例 |
| Get Drone Count | 获取管理的无人机数量 |
| Get Drone World Location | 获取指定无人机世界坐标 |

## 5.7 性能优化

- **30fps 更新频率：** 非每帧更新，视觉无感知差异
- **脏检测：** 位置阈值 0.5cm，颜色阈值 0.01，静止无人机零开销
- **实例池：** 移除的实例放入池中复用，避免频繁内存分配
- **禁用非必要功能：** 碰撞、阴影、贴花、距离剔除、密度缩放全部关闭

---

# 第六章 SuperDroneActor 单架无人机

## 6.1 功能简介

独立 Actor，通过 DroneId 与子系统数据同步，自动跟随位置、姿态和 LED 颜色。**适合需要挂载组件或蓝图交互的少量无人机。**

## 6.2 与 DroneSwarmManager 对比

| 特性 | DroneSwarmManager | SuperDroneActor |
|------|-------------------|-----------------|
| 渲染方式 | HISM 批量实例化 | 独立 Actor |
| 适合数量 | 100-10000+ | 1-50 |
| DrawCall | 1 次 | 每架 1 次 |
| 挂载子组件 | 不支持 | 支持 |
| 蓝图交互 | 有限 | 完全支持 |
| 材质 | PerInstanceCustomData | 动态材质实例 |

选择建议：预览编队用 SwarmManager，挂摄像机/特效用 DroneActor，可混用。

## 6.3 放置与设置

搜索 `SuperDroneActor`（显示名 Super Drone Actor），拖入视口，设置 Drone Id。每个 Actor 只跟踪一架。

## 6.4 基本属性

### Drone Id（无人机编号）
- 类别：SuperDroneLink
- 默认：0
- 范围：0-65535
- 须与外部软件发送的编号一致

### Auto Sync（自动同步）
- 类别：SuperDroneLink
- 默认：开启
- 自动开始同步数据

## 6.5 同步选项（SuperDroneLink > Sync）

### Sync Position（同步位置）
- 默认：开启
- 开启时 Actor 跟随无人机位置移动

### Sync Rotation（同步旋转）
- 默认：开启
- 开启时 Actor 反映无人机的 Roll/Pitch/Yaw 姿态

### Sync LED Color（同步 LED 颜色）
- 默认：开启
- 开启时材质颜色跟随无人机 LED 变化

### Position Interp Speed（位置插值速度）
- 默认：15.0
- 范围：>=0
- 设为 0 则瞬间移动到目标位置
- 值越大移动越快，默认 15 为适中平滑

### Rotation Interp Speed（旋转插值速度）
- 默认：10.0
- 范围：>=0
- 设为 0 则瞬间旋转，默认 10 为适中平滑

## 6.6 材质参数（SuperDroneLink > Material）

### LED Color Parameter Name
- 默认：`LedColor`
- 材质中的 Vector Parameter 名称，接收 LED 颜色

### LED Brightness Parameter Name
- 默认：`LedBrightness`
- 材质中的 Scalar Parameter 名称，接收 LED 亮度（0-1）

自定义材质需包含对应名称的参数节点，或修改这两个属性匹配你的材质。

## 6.7 挂载子组件

- **摄像机跟拍：** Add Component > Camera，调整位置角度，配合 Sequencer Camera Cut 使用
- **点光源：** Add Component > Point Light，绑定到 Get Led Color 实现环境照明
- **粒子特效：** Add Component > Niagara System，添加尾焰、轨迹等效果

## 6.8 蓝图节点

| 节点 | 类型 | 说明 |
|------|------|------|
| Get Drone Id | 纯函数 | 返回绑定的无人机编号 |
| Set Drone Id | 可调用 | 运行时修改编号（0-65535） |
| Get Led Color | 纯函数 | 返回当前 LED 颜色 |
| Start Sync | 可调用 | 开始同步 |
| Stop Sync | 可调用 | 停止同步 |

---

# 第七章 Sequencer 时间轴集成

## 7.1 功能概述

与 UE Sequencer 深度集成：录制实时数据为关键帧、回放轨迹、手动编辑曲线、多机编排。

## 7.2 添加 DroneLink 轨道

打开 Sequencer > 点击 + Track > 选择 DroneLink Track。一个轨道可包含多个无人机区段。

## 7.3 添加无人机区段

右键轨道 > Add Section > DroneLink Section，设置 Drone Id（0-65535）。同一轨道不能有重复 DroneId 的区段，每个区段自动放到不同行避免重叠。

## 7.4 曲线通道（每区段 10 条曲线）

### 位置曲线（3 条）

| 通道 | 单位 | 说明 |
|------|------|------|
| Position X | 厘米 | UE X 轴（北向） |
| Position Y | 厘米 | UE Y 轴（东向） |
| Position Z | 厘米 | UE Z 轴（上方） |

### 姿态曲线（3 条）

| 通道 | 单位 | 范围 |
|------|------|------|
| Roll | 度 | -180 ~ 180 |
| Pitch | 度 | -180 ~ 180 |
| Yaw | 度 | 0 ~ 360（正北为 0） |

### LED 曲线（4 条）

| 通道 | 范围 | 说明 |
|------|------|------|
| LED R | 0-255 | 红色 |
| LED G | 0-255 | 绿色 |
| LED B | 0-255 | 蓝色 |
| LED Brightness | 0-255 | 亮度 |

编辑：双击区段进入曲线编辑器，右键 Add Key 添加关键帧，可选线性/平滑/阶梯插值。

## 7.5 录制模式

区段的 **Is Recording** 标志：
- **true（录制中）：** 只记录数据，不播放。防止录制和播放互相干扰
- **false（正常）：** 播放时按曲线更新无人机状态

录制流程：设 Is Recording = true > 接收实时数据 > 写入曲线关键帧 > 设 Is Recording = false > 回放查看

## 7.6 播放行为

Sequencer 播放时，非录制区段会：评估所有曲线 > 写入位置/姿态/LED 到子系统。

- Sequencer 播放时曲线数据**覆盖**实时接收数据
- Sequencer 停止时实时数据正常工作
- 同一无人机同时有两个数据源时以最后写入为准

## 7.7 多机编排

一个轨道中为每架无人机添加独立区段，各自设置不同 DroneId，播放时所有区段同时评估。

> 100+ 架建议通过外部工具批量生成曲线数据导入，而非手动逐架编辑。

---

# 第八章 LDLink 协议说明

## 8.1 协议概述

LDLink（Limx Drone Link）是专为大规模编队数据传输设计的协议。

| 特性 | 值 |
|------|-----|
| 传输层 | UDP |
| 默认端口 | 14555 |
| 字节序 | 小端序 |
| 魔数 | `LDLK`（4字节 ASCII） |
| 版本 | 1.0 |

## 8.2 数据包结构

包头（12 字节）+ 载荷（Length 字节）

| 字段 | 偏移 | 大小 | 说明 |
|------|------|------|------|
| Magic | 0 | 4B | ASCII `LDLK` |
| Version | 4 | 1B | 协议版本 |
| OpCode | 5 | 1B | 操作码 |
| Universe | 6 | 2B | Universe 编号（小端序） |
| Sequence | 8 | 1B | 序列号（0-255 循环） |
| Length | 9 | 2B | 载荷长度（小端序） |
| Reserved | 11 | 1B | 保留 |

## 8.3 操作码

| 十六进制 | 名称 | 说明 |
|----------|------|------|
| 0x10 | LedData | LED 颜色 |
| 0x20 | PositionData | 位置（本地坐标） |
| 0x30 | StateData | 位置+LED（推荐） |
| 0x40 | Sync | 帧同步标记 |
| 0x50 | Poll | 轮询请求 |
| 0x61 | PollReply | 轮询回复 |
| 0x70 | Command | 控制指令 |

## 8.4 LED 数据（0x10）

### Universe 模式（Universe > 0）— 每架 5 字节

| 偏移 | 大小 | 说明 |
|------|------|------|
| 0 | 1B | Channel（0-255） |
| 1 | 1B | R（0-255） |
| 2 | 1B | G（0-255） |
| 3 | 1B | B（0-255） |
| 4 | 1B | Brightness（0-255） |

DroneId = Universe x 256 + Channel

### 全局模式（Universe = 0）— 每架 6 字节

| 偏移 | 大小 | 说明 |
|------|------|------|
| 0 | 2B | DroneId（小端序） |
| 2 | 1B | R |
| 3 | 1B | G |
| 4 | 1B | B |
| 5 | 1B | Brightness |

## 8.5 位置数据（0x20）— 每架 14 字节

| 偏移 | 大小 | 说明 |
|------|------|------|
| 0 | 2B | DroneId（小端序） |
| 2 | 4B | X（浮点，米） |
| 6 | 4B | Y（浮点，米） |
| 10 | 4B | Z（浮点，米） |

坐标单位为米，内部自动 x100 转厘米。始终使用全局模式。

## 8.6 完整状态（0x30）

### Universe 模式（Universe > 0）— 每架 17 字节

| 偏移 | 大小 | 说明 |
|------|------|------|
| 0 | 1B | Channel |
| 1 | 4B | X（米） |
| 5 | 4B | Y（米） |
| 9 | 4B | Z（米） |
| 13 | 4B | Color（RGBA 打包，小端序：R,G,B,Brightness） |

### 全局模式（Universe = 0）— 每架 18 字节

| 偏移 | 大小 | 说明 |
|------|------|------|
| 0 | 2B | DroneId（小端序） |
| 2 | 4B | X（米） |
| 6 | 4B | Y（米） |
| 10 | 4B | Z（米） |
| 14 | 4B | Color（RGBA 打包） |

## 8.7 Universe 寻址

类似 DMX 的 Universe 概念，对大量无人机分组：

| Universe | DroneId 范围 |
|----------|-------------|
| 0 | 全局模式 |
| 1 | 256 - 511 |
| 2 | 512 - 767 |
| ... | ... |
| 255 | 65280 - 65535 |

公式：DroneId = Universe x 256 + Channel

判断逻辑：Universe > 0 用 Universe 模式；Universe = 0 时根据数据长度能否整除条目大小自动判断。

## 8.8 与 LimxDroneStudio 对接

LimxDroneStudio 设置：
1. 目标地址：UE 电脑 IP
2. 目标端口：14555
3. 协议：LDLink
4. 数据内容：建议「完整状态」（StateData 0x30）
5. 发送频率：建议 25-30Hz

### 带宽估算（StateData 全局模式，25Hz）

| 无人机数量 | 每帧大小 | 25Hz 带宽 |
|-----------|----------|----------|
| 100 | 1.8 KB | 45 KB/s |
| 1000 | 18 KB | 450 KB/s |
| 5000 | 90 KB | 2.2 MB/s |
| 10000 | 180 KB | 4.5 MB/s |

---

# 第九章 无人机状态与编号

## 9.1 连接状态

| 状态 | 说明 |
|------|------|
| Disconnected | 从未收到数据，或已清理 |
| Connecting | 刚创建，尚未收到完整数据 |
| Connected | 正常接收数据 |
| Lost | 超时（当前版本超时即直接移除） |

流程：不存在 > 收到数据 > Connecting > 收到更多数据 > Connected > 超时 > 移除（回到不存在）

## 9.2 DroneId 编号规则

范围 0-65535，支持最多 65536 架。

### 扩展寻址公式

`DroneId = (ComponentId - 1) x 256 + SystemId`

| 编号方式 | DroneId 范围 | 说明 |
|----------|-------------|------|
| LDLink 直接编号 | 0-65535 | 直接使用 DroneId |
| MAVLink 标准 | 0-255 | ComponentId=1，DroneId=SystemId |
| MAVLink 扩展 | 0-65535 | 利用 ComponentId 扩展 |

反推：SystemId = DroneId % 256，ComponentId = DroneId / 256 + 1

## 9.3 位置数据

| 字段 | 说明 |
|------|------|
| Latitude/Longitude/Altitude | GPS 原始数据（度/米） |
| RelativeAltitude | 相对起飞点高度（米） |
| LocalPosition | 当前显示位置（厘米，经插值） |
| TargetPosition | 最新目标位置（厘米） |

UE 坐标：X=北，Y=东，Z=上

## 9.4 姿态数据

| 字段 | 范围 | 说明 |
|------|------|------|
| Roll | -180~180 度 | 横滚角 |
| Pitch | -180~180 度 | 俯仰角 |
| Yaw | 0~360 度 | 偏航角（正北为 0） |
| LocalRotation | FRotator | 已转换的 UE 旋转 |

## 9.5 LED 数据

| 字段 | 范围 | 说明 |
|------|------|------|
| LedColor | 各分量 0-1 | RGB 颜色（从 0-255 归一化） |
| LedBrightness | 0-1 | 亮度（从 0-255 归一化） |

## 9.6 统计信息（GetStats）

| 字段 | 说明 |
|------|------|
| ConnectedDrones | 已连接无人机数 |
| PacketsPerSecond | 每秒接收包数 |
| BytesPerSecond | 每秒接收字节数 |
| ParseErrors | 解析错误数 |
| LostPacketCount | 丢包数 |
| PacketLossRate | 丢包率（0-100%） |
| AverageLatencyMs | 平均延迟（毫秒） |

---

# 第十章 坐标系统与转换

## 10.1 两个坐标系

| 坐标系 | 轴定义 | 单位 |
|--------|--------|------|
| WGS84（GPS） | 经度/纬度/海拔 | 度/米 |
| UE 本地 | X=北, Y=东, Z=上 | 厘米 |

## 10.2 转换原理

1. **设定锚点：** 选择一个 GPS 坐标为 UE 原点
2. **计算偏移：** dLat/dLon 转弧度
3. **椭球投影：** 利用 WGS84 参数计算米级偏移
   - 北向 = dLat x 子午圈曲率半径(M)
   - 东向 = dLon x 卯酉圈曲率半径(N) x cos(锚点纬度)
   - 高度 = 目标海拔 - 锚点海拔
4. **转厘米：** x100

WGS84 参数：长半轴 6,378,137m，扁率 1/298.257。同一经纬度差在不同纬度对应不同实际距离，SuperDroneLink 使用精确椭球参数而非简单线性近似。

## 10.3 单位汇总

| 数据类型 | 外部输入 | 内部存储 | 说明 |
|----------|---------|---------|------|
| GPS 经纬度 | 度 | 度 | 直接存储 |
| GPS 海拔 | 米 | 米 | 直接存储 |
| LDLink 位置 | 米 | 厘米 | 接收时 x100 |
| UE 坐标 | 厘米 | 厘米 | 引擎标准 |
| MAVLink 姿态 | 弧度 | 度 | 接收时转换 |
| Sequencer 姿态 | 度 | 度 | 直接存储 |
| LED 颜色/亮度 | 0-255 | 0-1 | 接收时归一化 |

---

# 第十一章 蓝图节点参考

## 11.1 DroneLinkSubsystem 节点

### 配置类

| 节点 | 类型 | 输入 | 输出 | 说明 |
|------|------|------|------|------|
| Apply Config | 可调用 | FDroneLinkConfig | 无 | 应用配置并重启接收器 |
| Get Config | 纯函数 | 无 | FDroneLinkConfig | 获取当前配置 |

### 锚点类

| 节点 | 类型 | 输入 | 说明 |
|------|------|------|------|
| Set Geo Anchor | 可调用 | Lat, Lon, Alt | 手动设置锚点 |
| Set Geo Anchor From First Drone | 可调用 | 无 | 以首架无人机位置为锚点 |
| Get Geo Anchor | 纯函数 | 无 | 获取当前锚点 |
| Reset Geo Anchor | 可调用 | 无 | 重置锚点 |

### 查询类

| 节点 | 类型 | 输入 | 输出 | 说明 |
|------|------|------|------|------|
| Get All Drone Ids | 可调用 | 无 | TArray uint16 | 所有无人机 ID（已排序） |
| Get Drone State | 可调用 | DroneId | FDroneState, bool | 获取指定无人机状态 |
| Get All Drone States | 可调用 | 无 | TArray FDroneState | 所有无人机状态 |
| Get Connected Drone Count | 纯函数 | 无 | int32 | 已连接数量 |
| Get Stats | 纯函数 | 无 | FDroneLinkStats | 统计信息 |

### 坐标转换

| 节点 | 输入 | 输出 | 说明 |
|------|------|------|------|
| Convert Geo To Local | Lat, Lon, Alt | FVector | GPS 转 UE 坐标（厘米） |

## 11.2 DroneSwarmManager 节点

| 节点 | 类型 | 说明 |
|------|------|------|
| Start Sync | 可调用 | 开始同步 |
| Stop Sync | 可调用 | 停止同步 |
| Refresh All Instances | 可调用 | 强制重建所有实例 |
| Get Drone Count | 纯函数 | 当前管理数量 |
| Get Drone World Location | 纯函数 | 指定无人机的世界坐标 |

## 11.3 SuperDroneActor 节点

| 节点 | 类型 | 说明 |
|------|------|------|
| Get Drone Id | 纯函数 | 获取绑定的编号 |
| Set Drone Id | 可调用 | 修改绑定的编号 |
| Get Led Color | 纯函数 | 获取当前 LED 颜色 |
| Start Sync | 可调用 | 开始同步 |
| Stop Sync | 可调用 | 停止同步 |

---

# 第十二章 常见问题与故障排除

## 12.1 无人机不显示

**检查清单：**
1. 确认 SuperStage 插件已启用并重启了编辑器
2. 确认场景中已放置 DroneSwarmManager 且 Auto Sync 开启
3. 确认 LimxDroneStudio 的目标 IP 和端口正确（默认 14555）
4. 检查 Windows 防火墙是否放行了 UDP 14555 端口
5. 确认 LimxDroneStudio 已开始输出数据
6. 尝试调大 Drone Scale（默认 50 可能在大场景中看不到）

## 12.2 无人机位置偏移严重

**可能原因及解决方案：**
1. **锚点设置错误：** 调用 ResetGeoAnchor 重置后让系统自动设置
2. **坐标系不匹配：** 确认外部软件输出的是正确的坐标格式（GPS 或本地坐标）
3. **单位错误：** LDLink 位置数据单位是米，系统内部自动转厘米。确认发送端单位正确
4. **DroneSwarmManager 位置：** 如使用本地坐标，管理器的世界位置就是偏移原点

## 12.3 无人机闪烁或抖动

**可能原因及解决方案：**
1. **网络丢包严重：** 调高 HeartbeatTimeoutSeconds（如 10-15 秒）
2. **数据频率太低：** 提高 LimxDroneStudio 的发送频率（建议 25Hz+）
3. **插值不足：** 提高 Interp Speed（DroneSwarmManager 或 SuperDroneActor 的插值速度）
4. **无人机反复超时重连：** 调高超时时间

## 12.4 LED 颜色不变化

**检查清单：**
1. 确认外部软件正在发送 LED 数据（操作码 0x10 或 0x30）
2. DroneSwarmManager：确认材质支持 PerInstanceCustomData 且通道分配正确
3. SuperDroneActor：确认 Sync LED Color 已开启
4. SuperDroneActor：确认材质中有名为 `LedColor`（Vector）和 `LedBrightness`（Scalar）的参数
5. 使用默认内置材质 MI_DroneLed_Inst 测试是否为自定义材质问题

## 12.5 性能问题

**优化建议：**
1. 使用 DroneSwarmManager（单 DrawCall）而非大量 SuperDroneActor
2. 使用低面数模型（默认球体最佳）
3. 合理设置 Preallocated Instance Count 避免运行时动态创建
4. 确认 Drone Scale 不要设得过大导致大量面片渲染
5. 减少视口中其他高开销渲染元素

## 12.6 网络连接问题

**排查步骤：**
1. 在 UE 电脑上运行 `netstat -an | findstr 14555` 确认端口已监听
2. 从 LimxDroneStudio 电脑 `ping` UE 电脑确认网络可达
3. 检查两端防火墙设置
4. 尝试将 BindIP 留空（接受所有网卡）
5. 两台电脑的子网掩码是否匹配
6. 如使用 WiFi，尝试切换为有线连接

---

# 附录 A 术语表

| 术语 | 说明 |
|------|------|
| **DroneId** | 无人机唯一编号，0-65535 |
| **DroneLink Subsystem** | 引擎级子系统，全局单例，管理所有无人机数据 |
| **DroneSwarmManager** | 编队管理器 Actor，HISM 批量渲染 |
| **SuperDroneActor** | 单架无人机 Actor，独立跟踪 |
| **GeoAnchor** | 地理锚点，GPS 到 UE 坐标的转换原点 |
| **HISM** | Hierarchical Instanced Static Mesh，层级实例化静态网格 |
| **LDLink** | Limx Drone Link 协议 |
| **MAVLink** | Micro Air Vehicle Link，无人机通信标准协议 |
| **WGS84** | World Geodetic System 1984，GPS 坐标系标准 |
| **MSL** | Mean Sea Level，平均海平面 |
| **Universe** | 类似 DMX Universe 的分组概念，每组 256 架 |
| **Channel** | Universe 内的通道号，0-255 |
| **PerInstanceCustomData** | UE 实例化渲染的逐实例数据传递机制 |
| **PIE** | Play In Editor，UE 编辑器内播放模式 |
| **DrawCall** | GPU 绘制调用，越少性能越好 |
| **插值（Interpolation）** | 在两个已知值之间计算中间值，实现平滑过渡 |
| **心跳（Heartbeat）** | 定期发送的存活信号，用于检测连接状态 |
| **脏检测（Dirty Check）** | 检查数据是否发生变化，只更新变化的部分 |

---

# 附录 B 性能参考指标

以下数据基于 RTX 3060 显卡、i7-12700 CPU 测试环境，仅供参考：

| 指标 | 1000 架 | 5000 架 | 10000 架 |
|------|---------|---------|----------|
| FPS（DroneSwarmManager） | 120+ | 90+ | 60+ |
| DrawCall | 1 | 1 | 1 |
| CPU 每帧耗时 | <0.5ms | <1ms | <2ms |
| GPU 显存增量 | ~10MB | ~40MB | ~80MB |
| 网络带宽（25Hz StateData） | 45 KB/s | 2.2 MB/s | 4.5 MB/s |
| 推荐发送频率 | 25-30Hz | 20-25Hz | 15-20Hz |

### 性能优化优先级

1. **首选 DroneSwarmManager** 而非大量 SuperDroneActor
2. **使用低面数模型**（默认球体 < 100 面最佳）
3. **合理预分配** Preallocated Instance Count
4. **启用脏检测**（默认已启用，无需额外操作）
5. **控制发送频率** 在带宽和流畅度之间取平衡

---

*本手册涵盖 SuperDroneLink 模块的所有用户可操作功能。如有技术问题，请联系 LimxTeam 技术支持。*

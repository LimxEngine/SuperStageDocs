# SuperStage 产品文档

## 概述

SuperStage 是基于 Unreal Engine 5 的新一代舞美灯光预演与虚拟制作系统。它将灯光控台编程、激光控制、NDI 视频流、无人机编队、施工图绘制整合在统一的实时 3D 环境中，实现「一个引擎，全链路覆盖」。

## 系统要求

### 硬件要求

| 项目 | 最低配置 | 推荐配置 |
|------|---------|---------|
| CPU | Intel i7-10700 / AMD Ryzen 7 3700X | Intel i9-13900K / AMD Ryzen 9 7950X |
| GPU | NVIDIA RTX 3060 (12GB) | NVIDIA RTX 4080 (16GB) |
| 内存 | 32 GB DDR4 | 64 GB DDR5 |
| 存储 | 256 GB SSD | 1 TB NVMe SSD |
| 系统 | Windows 10 21H2 (64-bit) | Windows 11 23H2 (64-bit) |

### 软件依赖

- Unreal Engine 5.6+
- Visual Studio 2022（C++ 桌面开发工作负载）
- .NET 6.0 Runtime

## 快速开始

### 1. 安装插件

将 SuperStage 插件解压到项目的 `Plugins/` 目录：

```
YourProject/
├── Plugins/
│   └── SuperStage/
│       ├── SuperStage.uplugin
│       ├── Source/
│       └── Content/
```

### 2. 启用插件

在 UE5 编辑器中：**Edit → Plugins → 搜索 "SuperStage" → 勾选启用 → 重启编辑器**。

### 3. 创建第一个灯光场景

1. 打开 SuperBrowser 资产浏览器（`Window → SuperStage → SuperBrowser`）
2. 从灯库中拖入灯具到场景
3. 打开 SuperConsolePro 控台（`Window → SuperStage → Console`）
4. 选中灯具，调整参数

## 核心模块

### SuperConsolePro

内置专业灯光控台，支持 CUE 编程、Preset 预设、Effect 引擎、Timeline 时间线、Timecode 时间码同步。

**核心特性：**

- **CUE 系统**：支持 Fade / Delay / Snap / Override
- **Preset 系统**：颜色、位置、光束、效果四类预设
- **Effect 引擎**：正弦、锯齿、随机、步进等波形
- **Timeline**：多 CUE 多轨道可视化编排
- **Timecode**：SMPTE LTC / Art-Net Timecode / Internal

### SuperLaser

Beyond 激光控制系统，通过 TCP 直连 Pangolin Beyond 软件，实现 UE5 内激光点云实时渲染。

**核心特性：**

- 71.4% ILDA 点云压缩率
- Zone → Actor 自动映射
- 激光束照亮烟雾粒子

### SuperNdi

NDI 视频流集成系统，支持在 3D 场景中嵌入实时视频流。

**核心特性：**

- NDI 发现与接收
- 视频纹理实时更新
- Alpha 通道支持

### SuperDroneLink

无人机编队模块，接收 LimxDroneStudio 的编队数据并在 UE5 中实时可视化。

**核心特性：**

- UDP Limx Binary Protocol 接收
- 3D 编队实时渲染
- LED 灯光同步

### SuperShader

灯光着色器系统，将 DMX 通道数据转化为物理正确的光学渲染效果。

**核心特性：**

- 13 种属性分类（亮度、位置、颜色、图案、光束…）
- 8/16/24-bit 精度支持
- GDTF / MA 灯库导入

### SuperCAD

专业施工图绘制系统，在 UE5 编辑器内直接生成 CAD 级别灯位图、正立面图。

**核心特性：**

- 3D→2D 实时联动
- 正交描边渲染
- DXF / SVG / PDF 导出
- 智能标注系统

## 网络端口

| 协议 | 端口 | 方向 | 说明 |
|------|------|------|------|
| Art-Net | UDP 6454 | 双向 | 灯光控制信号 |
| sACN | UDP 5568 | 双向 | 流式 ACN |
| SuperData | TCP 5966 | 双向 | 跨平台数据同步 |
| Beyond | TCP 16062 | 发送 | 激光点云 |
| NDI | 动态 | 接收 | 视频流 |
| OSC | UDP 可配 | 双向 | 开放声音控制 |
| LimxDroneStudio | UDP 17000 | 接收 | 无人机编队 |

## 设计哲学

### Configuration over Customization

不为任何单一厂家写死代码。所有硬件特性通用化、配置化。

### Graceful Degradation

单台灯具离线不影响全局。Art-Net 中断保持最后有效状态。

### Single Source of Truth

所有状态判定以 SuperStage 为准，硬件端的状态只是"影射"。

### Zero Trust for External Systems

所有外部输入经过校验和容错处理。

## 更新日志

请参阅 [更新历史](/about) 页面了解版本变更详情。

## 联系支持

- **邮箱**: yunsio@yunsio.com
- **Bilibili**: SuperStage 官方频道
- **微信群**: 添加客服微信获取邀请

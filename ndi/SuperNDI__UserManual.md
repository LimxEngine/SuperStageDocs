# SuperNDI 用户手册

> **版本**：SuperStage 26Q1  
> **最后更新**：2026-03-05  
> **适用引擎**：Unreal Engine 5.6 ~ 5.7  
> **适用对象**：舞台灯光设计师、虚拟制作技术人员、LED 屏幕操作员

---

## 目录

1. [概述与核心概念](#1-概述与核心概念)
2. [系统要求与前置准备](#2-系统要求与前置准备)
3. [NDI 配置面板（SuperNDI Config）](#3-ndi-配置面板superndi-config)
4. [NDI 编辑器全局设置（Project Settings）](#4-ndi-编辑器全局设置project-settings)
5. [NDI 屏幕 Actor（SuperNDIScreen）](#5-ndi-屏幕-actorsuperndiscreen)
6. [Take Recorder 录制 NDI 视频](#6-take-recorder-录制-ndi-视频)
7. [Sequencer 回放 NDI 录制](#7-sequencer-回放-ndi-录制)
8. [常见问题与故障排除](#8-常见问题与故障排除)
9. [性能优化建议](#9-性能优化建议)
10. [术语表](#10-术语表)

---

## 1. 概述与核心概念

### 1.1 什么是 SuperNDI？

SuperNDI 是 SuperStage 插件中的 **NDI 视频流接入模块**，允许你在 Unreal Engine 编辑器和运行时中：

- **实时接收** 来自局域网中任何 NDI 源（如 OBS、vMix、NDI 摄像机、Resolume 等）的视频画面
- **将视频画面投射到 3D 场景中的 LED 屏幕/投影幕布** 上，实现虚拟制作预演
- **录制** NDI 视频流到 Sequencer 时间轴，保存为可回放的帧序列
- **回放** 预录制的 NDI 视频，无需真实 NDI 源即可预演效果

### 1.2 核心工作流程

```
外部 NDI 源（OBS/vMix/摄像机等）
        │
        ▼
   局域网 NDI 传输
        │
        ▼
  SuperNDI 配置面板（添加逻辑输入、映射源）
        │
        ▼
  SuperNDIScreen Actor（场景中的 LED 屏幕）
        │
        ├──→ 实时预览（编辑器/运行时）
        ├──→ Take Recorder 录制（保存帧序列）
        └──→ Sequencer 回放（离线预演）
```

### 1.3 "逻辑输入名"与"外部源"

SuperNDI 采用**逻辑名映射**机制，这是理解整个系统的关键：

| 概念 | 说明 | 示例 |
|------|------|------|
| **逻辑输入名**（InputName） | 你自定义的名字，用于在项目中引用一路 NDI 输入 | `MainScreen`、`CameraA`、`SidePanel` |
| **外部源**（ExternalSource） | 网络中实际存在的 NDI 设备/软件的显示名称 | `OBS (DESKTOP-ABC)`、`vMix - Output 1` |
| **映射** | 将一个逻辑名绑定到一个外部源 | `MainScreen` → `OBS (DESKTOP-ABC)` |

**为什么要这样设计？**

- 外部源名称可能随网络环境变化（换了电脑名、换了设备）
- 逻辑名在你的项目中保持不变，场景中的屏幕 Actor 只需要关心逻辑名
- 换一台 NDI 源设备时，只需要在配置面板重新映射，不需要修改场景中的任何 Actor

---

## 2. 系统要求与前置准备

### 2.1 硬件要求

| 项目 | 最低要求 | 推荐配置 |
|------|----------|----------|
| **CPU** | 支持 SSE 4.2 指令集 | Intel i7 / AMD Ryzen 7 以上 |
| **内存** | 16 GB | 32 GB 以上（录制时需要更多） |
| **网络** | 千兆以太网 | 万兆以太网（多路 4K NDI） |
| **显卡** | 支持 DirectX 12 | RTX 3060 以上 |

### 2.2 网络配置

NDI 基于 mDNS（UDP 端口 5353）进行源发现，基于 TCP 传输视频数据。请确保：

1. **所有 NDI 设备处于同一局域网/子网** 或已配置跨网段路由
2. **Windows 防火墙** 允许以下端口：
   - **UDP 5353**（mDNS 源发现）
   - **TCP/UDP 5960-5969**（NDI 数据传输）
3. **Bonjour 服务** 正在运行（通常随 NDI Runtime 自动安装）
4. 如果使用 VPN 或虚拟网卡，确认 mDNS 广播未被阻断

### 2.3 NDI 运行时

SuperNDI 插件**已内置 NDI SDK DLL**（`Processing.NDI.Lib.x64.dll`），无需额外安装 NDI Runtime。

DLL 文件查找顺序（优先级从高到低）：
1. `Plugins/SuperStage/Binaries/Win64/Processing.NDI.Lib.x64.dll`
2. `Plugins/SuperStage/Source/SuperNdi/ThirdParty/NDI/Bin/x64/`
3. `Plugins/SuperStage/Source/SuperNdi/ThirdParty/NDI/Bin/`（回退路径）

> **重要提示**：SuperNDI **不会使用** 系统 PATH 中安装的 NDI DLL（如 NDI Tools 6 自带的版本），以避免版本冲突。如果启动时看到 DLL 相关错误，请确认上述路径中存在正确的 DLL 文件。

### 2.4 与 NDI Tools 6 的兼容性

如果你的电脑上安装了 NDI Tools 6（包含 NDI Studio Monitor），可能出现以下冲突：

- **NDI Studio Monitor 独占源**：关闭 Studio Monitor 后再启动 UE 编辑器
- **DLL 版本冲突**：SuperNDI 已内置集成 DLL，会忽略系统安装的版本
- **Finder 冲突**：极少数情况下两个 NDI Finder 同时运行会干扰发现

**建议**：在使用 SuperNDI 时，关闭 NDI Studio Monitor 等独立 NDI 工具。

---

## 3. NDI 配置面板（SuperNDI Config）

配置面板是使用 SuperNDI 的**第一步**——在这里管理逻辑输入和映射外部源。

### 3.1 打开配置面板

在 UE 编辑器**主工具栏**中找到 **SuperNDI** 按钮（位于 SuperDMX 配置按钮旁边），单击即可打开配置面板。

面板会以独立标签页形式打开，可以停靠到编辑器的任意位置。

### 3.2 面板界面说明

配置面板由以下部分组成：

```
┌─────────────────────────────────────────────────┐
│  NDI Inputs   [ Enter logical input name... ]   │
│               [Add]    [Refresh Sources]         │
├─────────────────────────────────────────────────┤
│  Configured Inputs:                              │
│                                                  │
│  MainScreen    [▼ OBS (DESKTOP-ABC)]   [Remove]  │
│  CameraA       [▼ <None>            ]   [Remove]  │
│  SidePanel     [▼ vMix - Output 1   ]   [Remove]  │
└─────────────────────────────────────────────────┘
```

### 3.3 添加逻辑输入

1. 在顶部的文本输入框中输入一个**逻辑输入名称**（例如 `MainScreen`）
2. 点击 **Add** 按钮
3. 新的逻辑输入会出现在 "Configured Inputs" 列表中

**命名建议**：
- 使用有意义的英文名称（如 `MainScreen`、`CameraA`、`BackDrop`）
- 名称不能为空
- 名称不能重复（重复时会自动忽略）
- 名称在项目中全局唯一，所有 SuperNDIScreen Actor 共享同一套逻辑输入

### 3.4 映射外部 NDI 源

每个逻辑输入右侧有一个**下拉菜单**，用于选择要绑定的外部 NDI 源：

1. 点击下拉箭头 **▼**
2. 面板会自动刷新网络中的 NDI 源列表
3. 从列表中选择目标源（如 `OBS (DESKTOP-ABC)`）
4. 选择后立即生效，NDI 接收器会自动创建并开始接收视频

**下拉菜单中的选项**：
- **`<None>`**：不绑定任何外部源（清除映射）
- **`设备名 (主机名)`**：网络中发现的 NDI 源，格式为 `源名称 (主机名)`

### 3.5 刷新源列表

点击 **Refresh Sources** 按钮可以强制重新扫描网络中的 NDI 源。

- 刷新过程为**异步执行**，不会阻塞编辑器
- 首次扫描可能需要 2-3 秒才能发现所有源（mDNS 广播延迟）
- 面板打开时会自动执行一次初始扫描
- 打开下拉菜单时也会触发一次后台刷新

> **提示**：如果网络中有新的 NDI 源上线，等待约 2-5 秒后刷新即可看到。

### 3.6 删除逻辑输入

点击对应行末尾的 **Remove** 按钮即可删除该逻辑输入。

- 删除后，所有引用该逻辑名的 SuperNDIScreen Actor 将停止接收视频
- 删除操作会立即保存到项目设置中
- 如果该逻辑名正在被 Take Recorder 录制，录制数据不会丢失

### 3.7 配置的持久化

所有配置（逻辑输入名、外部源映射）保存在**项目的编辑器设置中**（EditorPerProjectUserSettings），这意味着：

- 关闭并重新打开编辑器后，配置仍然存在
- 配置是**项目级别**的，不同项目有不同配置
- 配置是**用户级别**的，同一项目的不同用户可以有不同配置

---

## 4. NDI 编辑器全局设置（Project Settings）

除了配置面板，还可以通过 UE 编辑器的 **项目设置** 来管理 SuperNDI 的全局参数。

### 4.1 打开设置页面

菜单栏 → **Edit** → **Project Settings** → 左侧列表找到 **Plugins** 分类 → **SuperNDI Settings**

### 4.2 设置项详解

#### 4.2.1 Inputs（输入列表）

与配置面板中的内容相同，这里以表格形式展示所有已配置的逻辑输入：

| 字段 | 类型 | 说明 |
|------|------|------|
| **InputName** | 名称 | 逻辑输入名称（与配置面板中添加的名称一致） |
| **ExternalSource** | 名称 | 映射的外部 NDI 源显示名（可选，为空表示未映射） |

可以在这里直接添加/编辑/删除逻辑输入，效果与配置面板完全一致。

#### 4.2.2 DefaultTargetFPS（默认录制帧率）

| 项目 | 说明 |
|------|------|
| **类型** | 整数 |
| **范围** | 1 ~ 120 |
| **默认值** | 30 |
| **作用** | 新建 Take Recorder 录制源时的默认帧率 |

> **建议**：30 fps 适用于大多数场景。如果 NDI 源本身是 60 fps 且需要高精度同步，可设为 60。

#### 4.2.3 DefaultDownscale（默认降采样系数）

| 项目 | 说明 |
|------|------|
| **类型** | 浮点数 |
| **范围** | 0.25 ~ 1.0 |
| **默认值** | 1.0 |
| **作用** | 新建录制源时的默认降采样倍率 |

> **重要**：此值在内部会**再除以 2**（`Scale = Downscale × 0.5`）。也就是说：
> - 设置为 **1.0**（默认）：实际录制分辨率为**原始的 50%**（1920×1080 → 960×540）
> - 设置为 **0.5**：实际录制分辨率为**原始的 25%**（1920×1080 → 480×270）
> - 设置为 **0.25**（最小值）：实际录制分辨率为**原始的 12.5%**（1920×1080 → 240×135）
>
> 如果需要**无降采样**的原始分辨率录制，请在 Take Recorder 录制源中将 Downscale 设为 **2.0**（该参数范围为 0.5 ~ 2.0，见 [6.3.3 节](#633-downscale降采样系数)）。

#### 4.2.4 DefaultCompressionLevel（默认压缩等级）

| 项目 | 说明 |
|------|------|
| **类型** | 整数 |
| **范围** | 0 ~ 9 |
| **默认值** | 0 |
| **作用** | 帧缓冲数据的压缩级别（0 = 不压缩） |

> **注意**：当前版本中此参数为**预留参数**，帧缓冲始终以未压缩 BGRA 原始像素格式存储。压缩功能将在后续版本中实现。

---

## 5. NDI 屏幕 Actor（SuperNDIScreen）

SuperNDIScreen 是放置在 3D 场景中的 Actor，用于将 NDI 视频画面显示到指定的屏幕网格上。

### 5.1 放置 SuperNDIScreen

在 UE 编辑器中：

1. 打开 **Place Actors** 面板（快捷键 Shift+1）
2. 搜索 `SuperNDIScreen`
3. 拖放到视口中

或者在内容浏览器中右键 → All Classes → 搜索 `SuperNDIScreen` → 拖入场景。

> **注意**：SuperNDIScreen 本身**不包含网格体**。它是一个"控制器"Actor，需要将视频输出**应用到你指定的静态网格体**上（见 5.3 节）。

### 5.2 属性面板 — 完整参数说明

选中场景中的 SuperNDIScreen Actor，在 **Details**（细节）面板中可以看到以下属性：

#### 5.2.1 NDIInputSelection（NDI 输入选择）

| 项目 | 说明 |
|------|------|
| **类型** | 下拉选择（FName） |
| **默认值** | 空（自动选择第一个已配置源） |
| **作用** | 选择此屏幕要接收的逻辑输入名 |

**下拉列表**中会显示所有在配置面板中添加的逻辑输入名。

**自动选择行为**：如果此项为空（未手动选择），Actor 会在启动时**自动绑定第一个已配置的逻辑输入**。如果你有多个逻辑输入，建议手动选择目标输入。

#### 5.2.2 OutputTexture（输出纹理）

| 项目 | 说明 |
|------|------|
| **类型** | UTexture2D（只读） |
| **作用** | 运行时自动创建的纹理对象，显示当前接收到的 NDI 帧 |

这是一个只读属性，用于调试和确认纹理是否已创建。你不需要也不应该手动修改它。

#### 5.2.3 ScreenMeshActors（屏幕网格体）

| 项目 | 说明 |
|------|------|
| **类型** | 数组（AStaticMeshActor 引用列表） |
| **默认值** | 空 |
| **作用** | 指定一个或多个静态网格体 Actor，NDI 视频将渲染到这些网格体上 |

**使用方法**：
1. 在场景中放置一个或多个 **Static Mesh Actor**（通常是平面/曲面网格，代表 LED 屏幕）
2. 回到 SuperNDIScreen 的细节面板
3. 在 ScreenMeshActors 数组中点击 **+** 添加元素
4. 使用吸管工具或下拉菜单选择场景中的 Static Mesh Actor

> **一对多支持**：一个 SuperNDIScreen 可以同时将同一路 NDI 视频输出到**多块屏幕**，适用于 LED 矩阵拼接或视频墙场景。

#### 5.2.4 Transparent（透明模式）

| 项目 | 说明 |
|------|------|
| **类型** | 布尔值（复选框） |
| **默认值** | 关闭（false） |
| **作用** | 切换屏幕材质为半透明模式 |

- **关闭**：使用不透明材质（`MI_Screen_Inst`），适用于实体 LED 屏幕
- **开启**：使用半透明材质（`MI_Screen_Transparent`），适用于投影效果、全息效果

切换此选项后，材质实例会自动重建。

#### 5.2.5 Transparency（透明度）

| 项目 | 说明 |
|------|------|
| **类型** | 浮点数 |
| **范围** | 0.0 ~ 1.0 |
| **默认值** | 0.95 |
| **作用** | 控制半透明材质的不透明度 |
| **可见条件** | 仅当 Transparent 开启时显示 |

- **0.0**：完全透明（不可见）
- **0.95**（默认）：几乎不透明，略微可以透视背后内容
- **1.0**：完全不透明

#### 5.2.6 Brightness（亮度）

| 项目 | 说明 |
|------|------|
| **类型** | 浮点数 |
| **最小值** | 0.0 |
| **默认值** | 1.0 |
| **作用** | 调整画面的整体亮度 |

- **0.0**：全黑
- **1.0**：原始亮度
- **> 1.0**：增强亮度（可以超过 1.0 实现过曝效果）

#### 5.2.7 Color（颜色）

| 项目 | 说明 |
|------|------|
| **类型** | 线性颜色（FLinearColor） |
| **默认值** | 白色 (1, 1, 1, 1) |
| **作用** | 对画面进行颜色叠加/染色 |

将颜色设为白色时，画面保持原始色彩。设为其他颜色时，画面会被该颜色"染色"（相乘混合）。

**应用场景**：
- 设为红色可以模拟红色 LED 面板效果
- 设为 (0.5, 0.5, 0.5) 可以整体压暗画面

#### 5.2.8 Contrast（对比度）

| 项目 | 说明 |
|------|------|
| **类型** | 浮点数 |
| **最小值** | 0.0（UIMin） |
| **默认值** | 1.5 |
| **作用** | 调整画面对比度 |

> **提示**：默认值 1.5 会稍微增强对比度，使 LED 屏幕效果更真实。设为 1.0 为原始对比度。
>
> **技术说明**：代码中 `ClampMax = 1`（编辑器 UI 滑块最大值为 1.0），但构造时默认值为 1.5。首次放置 Actor 时 Contrast 显示为 1.5（有效），若通过 Details 面板修改后将无法再设回超过 1.0 的值。

#### 5.2.9 Deformation（梯形校正）

用于模拟投影仪梯形校正或 LED 屏幕非正交安装时的画面变形。

| 子参数 | 类型 | 范围 | 默认值 | 说明 |
|--------|------|------|--------|------|
| **UpperLeftCorner** | 2D 向量 | (0,0) ~ (1,1) | (0, 0) | 左上角 UV 偏移量 |
| **LowerLeftCorner** | 2D 向量 | (0,0) ~ (1,1) | (0, 0) | 左下角 UV 偏移量 |
| **UpperRightCorner** | 2D 向量 | (0,0) ~ (1,1) | (0, 0) | 右上角 UV 偏移量 |
| **LowerRightCorner** | 2D 向量 | (0,0) ~ (1,1) | (0, 0) | 右下角 UV 偏移量 |

**工作原理**：
- 每个角点的偏移值控制对应角在 UV 空间中的移动量
- (0, 0) 表示角点在原始位置，不偏移
- 增大 X 值会将角点向中心水平移动
- 增大 Y 值会将角点向中心垂直移动
- 所有值最终通过材质 Shader 实现四点透视变换

**典型使用场景**：
- 投影融合时校正投影仪的梯形畸变
- 模拟倾斜安装的 LED 屏幕的画面校正
- 多屏拼接时的边缘融合调整

### 5.3 使用步骤（快速上手）

**步骤 1**：在配置面板中添加逻辑输入（如 `MainScreen`），并映射到外部 NDI 源。

**步骤 2**：在场景中放置一个 Static Mesh Actor（如一个 Plane），调整大小和位置代表 LED 屏幕。

**步骤 3**：在场景中放置一个 SuperNDIScreen Actor（位置无所谓，它不可见）。

**步骤 4**：选中 SuperNDIScreen Actor，在细节面板中：
- **NDIInputSelection**：选择 `MainScreen`
- **ScreenMeshActors**：添加步骤 2 中的 Static Mesh Actor

**步骤 5**：完成！如果 NDI 源正在发送视频，你应该能在场景中看到画面。

### 5.4 多屏幕配置

一个场景中可以有**多个 SuperNDIScreen Actor**，每个绑定不同的逻辑输入：

```
SuperNDIScreen_1 → InputName: MainScreen → 主 LED 屏幕
SuperNDIScreen_2 → InputName: CameraA    → 侧面监视器
SuperNDIScreen_3 → InputName: SidePanel  → 侧面 LED 面板
```

也可以让**多个 SuperNDIScreen Actor 绑定同一个逻辑输入**，实现同一路视频在不同位置同时显示。

### 5.5 编辑器中的实时预览

SuperNDIScreen 支持**编辑器实时预览**——不需要按 Play，在编辑状态下就能看到 NDI 视频画面。

这是因为 Actor 在以下生命周期节点都会执行绑定：
- 拖入场景时（OnConstruction）
- 运行时启动（BeginPlay）
- 编辑器加载关卡时（PostLoad）
- 组件注册完成时（PostRegisterAllComponents）
- 修改 InputName 或 Transparent 属性时（PostEditChangeProperty）

如果编辑器预览不显示画面，尝试：
1. 确认配置面板中已正确映射外部源
2. 选中 Actor 后在细节面板中重新选择 NDIInputSelection
3. 检查 OutputTexture 属性是否已创建（非 None）

---

## 6. Take Recorder 录制 NDI 视频

SuperNDI 与 UE 的 **Take Recorder** 系统深度集成，可以将实时 NDI 视频流录制为 Sequencer 可回放的帧序列。

### 6.1 打开 Take Recorder

菜单栏 → **Window** → **Cinematics** → **Take Recorder**

### 6.2 添加 SuperNDI 录制源

1. 在 Take Recorder 面板中，点击 **+ Source** 按钮
2. 在弹出的列表中找到 **Super NDI Input**
3. 点击添加

添加后，Sources 列表中会出现一个名为 **"Super NDI (Logical Inputs)"** 的录制源。

### 6.3 配置录制参数

选中该录制源，在右侧的 Details 面板中可以看到以下参数：

#### 6.3.1 InputsToRecord（要录制的输入）

| 项目 | 说明 |
|------|------|
| **类型** | 名称数组（下拉多选） |
| **默认值** | 空（录制所有已配置的逻辑输入） |
| **作用** | 指定要录制哪些逻辑输入 |

**下拉选项**来自配置面板中已添加的逻辑输入名。

**行为规则**：
- 如果**留空**：将自动录制**所有**已配置的逻辑输入
- 如果**手动选择**：仅录制选中的逻辑输入
- 支持**多选**：可以同时录制多路 NDI 输入

#### 6.3.2 TargetFPS（目标帧率）

| 项目 | 说明 |
|------|------|
| **类型** | 整数 |
| **范围** | 1 ~ 120 |
| **默认值** | 30 |
| **作用** | 每秒录制多少帧 NDI 视频 |

**FPS 节流机制**：即使 NDI 源的帧率高于此值，录制器也会按此帧率采样。这是基于**墙钟时间**（真实时间）计算的，不受编辑器帧率波动影响。

| 设置值 | 适用场景 |
|--------|----------|
| 15 | 低精度预览、长时间录制 |
| 24 | 电影帧率 |
| 30 | **推荐**。平衡质量与内存 |
| 60 | 高精度同步 |

#### 6.3.3 Downscale（降采样系数）

| 项目 | 说明 |
|------|------|
| **类型** | 浮点数 |
| **范围** | 0.5 ~ 2.0 |
| **默认值** | 1.0 |
| **作用** | 控制录制帧的分辨率降采样 |

> **重要**：此值在内部会**再除以 2**，以节省内存。实际对照表：

| 面板显示值 | 实际降采样 | 1080p 实际录制分辨率 | 单帧内存 |
|-----------|-----------|---------------------|---------|
| **2.0** | 1.0（无降采样） | 1920 × 1080 | ~8 MB |
| **1.0**（默认） | 0.5 | 960 × 540 | ~2 MB |
| **0.5** | 0.25 | 480 × 270 | ~0.5 MB |

#### 6.3.4 MaxDurationSeconds（最大录制时长）

| 项目 | 说明 |
|------|------|
| **类型** | 整数（秒） |
| **范围** | 1 ~ 3600 |
| **默认值** | 900（15 分钟） |
| **作用** | 超过此时长后自动停止录制 |

**内存估算**（1080p@30fps，默认 Downscale=1.0 即实际 0.5 降采样）：

| 录制时长 | 预估内存占用 |
|----------|-------------|
| 1 分钟 | ~3.5 GB |
| 5 分钟 | ~17 GB |
| 15 分钟（默认上限） | ~52 GB |

> **警告**：NDI 帧缓冲存储在内存中（非磁盘）。录制超长视频时请密切关注内存使用。

### 6.4 开始录制

1. 确认所有参数设置正确
2. 确认 NDI 源正在发送视频（配置面板中已映射并接收）
3. 在 Take Recorder 面板中点击 **Record** 按钮
4. 录制开始后，状态栏会显示录制时长

### 6.5 停止录制

- 点击 **Stop** 按钮手动停止
- 或等待达到 MaxDurationSeconds 自动停止

### 6.6 录制结果

录制完成后：

1. 系统会在当前 Level Sequence 中创建 **NDI 轨道**（每个录制的逻辑输入一条轨道）
2. 轨道命名为 `NDI_逻辑输入名`（如 `NDI_MainScreen`）
3. 每条轨道包含一个 **Section**，其中关联了帧缓冲资产
4. 帧缓冲资产会随 Level Sequence 自动保存

> **提示**：录制完成后务必 **Ctrl+S 保存**当前 Level Sequence，否则帧缓冲数据可能丢失。

---

## 7. Sequencer 回放 NDI 录制

录制完成后，可以在 Sequencer 中回放预录制的 NDI 视频。

### 7.1 打开 Sequencer

双击 Content Browser 中的 Level Sequence 资产，或在场景中选中 Level Sequence Actor → 点击 **Open in Sequencer**。

### 7.2 NDI 轨道结构

在 Sequencer 时间轴中，你会看到录制生成的 NDI 轨道：

```
▼ NDI_MainScreen  ──────────────────────────
  ├─ [Section: MainScreen] ████████████████
  │   InputName: MainScreen
  │   FrameBuffer: (帧缓冲资产引用)
  │   HoldLastFrame: ✓
```

### 7.3 Section 属性

选中 Sequencer 中的 NDI Section（紫色/蓝色条块），在 Details 面板中可以看到：

#### 7.3.1 InputName（逻辑输入名）

| 项目 | 说明 |
|------|------|
| **类型** | 名称（FName） |
| **作用** | 指定此 Section 回放时注入到哪个逻辑输入 |

通常由录制器自动设置，不需要修改。如果需要将录制的视频输出到不同的逻辑输入，可以手动更改。

#### 7.3.2 FrameBuffer（帧缓冲）

| 项目 | 说明 |
|------|------|
| **类型** | 资产引用（USuperNDIMultiSourceFrameBuffer） |
| **作用** | 指向存储录制帧数据的缓冲资产 |

**帧缓冲属性**（展开后可见）：

| 属性 | 类型 | 说明 |
|------|------|------|
| **TargetFPS** | 整数 | 录制时使用的帧率 |
| **Downscale** | 浮点数 | 录制时的降采样系数 |
| **MaxDurationSeconds** | 整数 | 录制时的时长上限 |
| **Sources** | 数组 | 各逻辑输入的帧序列数据（只读） |

#### 7.3.3 HoldLastFrame（保持末帧）

| 项目 | 说明 |
|------|------|
| **类型** | 布尔值 |
| **默认值** | 开启（true） |
| **作用** | 播放头超出录制帧范围时的行为 |

- **开启**（推荐）：如果当前时间点没有精确匹配的帧，显示最近的一帧（避免黑屏）
- **关闭**：如果没有精确帧，不注入任何画面（可能导致黑屏闪烁）

### 7.4 回放机制详解

当 Sequencer 播放时，NDI Section 的行为如下：

1. **播放头进入 Section 范围**：
   - 自动开启**环回模式**（Loopback）
   - 屏蔽该逻辑输入的真实 NDI 网络帧
   - 确保场景中的 SuperNDIScreen 只显示录制内容，不受实时 NDI 源干扰

2. **每帧更新**：
   - 根据当前时间从帧缓冲中查询对应帧（二分查找，高效）
   - 将查询到的帧注入子系统
   - 子系统广播帧数据到所有订阅该逻辑输入的 SuperNDIScreen Actor

3. **播放头离开 Section 范围**：
   - 关闭环回模式
   - 恢复真实 NDI 网络帧接收
   - SuperNDIScreen 恢复显示实时 NDI 画面

### 7.5 多轨道/多 Section

- **多轨道**：可以同时回放多个逻辑输入（如主屏幕 + 侧屏幕），每个逻辑输入一条轨道
- **多 Section**：同一轨道内可以有多个 Section，对应不同时间段播放不同的帧缓冲
- **Section 排列**：
  - 同一行的 Section 按时间顺序排列
  - 不同行的 Section 可以并行（垂直堆叠）

### 7.6 离线渲染

NDI 回放支持 Sequencer 的**离线渲染**模式。在渲染电影时：

- 不需要真实 NDI 源在线
- 回放完全依赖录制的帧缓冲数据
- 画面质量取决于录制时的降采样设置

---

## 8. 常见问题与故障排除

### 8.1 源发现相关

#### Q：配置面板下拉列表中看不到任何 NDI 源

**排查步骤**：
1. 确认 NDI 源设备/软件**正在运行并发送视频**（如 OBS 已开启 NDI 输出）
2. 确认 NDI 源与 UE 主机在**同一局域网/子网**
3. 检查 **Windows 防火墙**，确保 UDP 5353 和 TCP/UDP 5960-5969 未被阻止
4. 关闭 **NDI Studio Monitor**（可能独占 Finder 资源）
5. 等待 **2-5 秒**后点击 **Refresh Sources**（mDNS 发现有延迟）
6. 检查 **Bonjour 服务**是否在运行（服务名：`Bonjour Service`）
7. 查看 UE 的 **Output Log**，搜索 `SuperNDI` 关键字，查看诊断信息

#### Q：源发现时间过长

**可能原因**：
- 首次启动时 mDNS 广播需要时间积累
- 持久化 Finder 启动后约 3-5 秒开始发现源
- 如果超过 10 秒仍未发现，检查网络配置

### 8.2 视频显示相关

#### Q：SuperNDIScreen 显示黑屏

**排查步骤**：
1. 确认配置面板中该逻辑输入已**映射到有效的外部源**
2. 确认 SuperNDIScreen 的 **NDIInputSelection** 已选择正确的逻辑输入
3. 确认 **ScreenMeshActors** 数组中至少添加了一个 Static Mesh Actor
4. 检查 **OutputTexture** 属性是否为 None（应该不是 None）
5. 尝试重新选择 NDIInputSelection（触发重新绑定）

#### Q：画面卡顿/帧率低

**可能原因与解决**：
- NDI 源本身帧率低 → 检查源设备设置
- 网络带宽不足 → 使用千兆或更高带宽的有线网络
- UE 编辑器帧率低 → 降低编辑器画面质量或关闭不必要的窗口
- UYVY/UYVA 格式转换开销 → NDI 源尽量输出 BGRA/BGRX 格式（避免 CPU 侧 YUV→BGRA 逐像素转换）

#### Q：画面颜色偏差

SuperNDI 默认使用 **sRGB** 色彩空间。如果 NDI 源输出的是线性色彩空间，可能出现颜色过亮/偏色。请确认 NDI 源的输出色彩空间设置。

### 8.3 录制相关

#### Q：Take Recorder 中看不到 Super NDI Input 选项

确认 SuperStageEditor 模块已加载。在 Output Log 中搜索 `SuperNDI` 确认初始化成功。

#### Q：录制后帧缓冲数据丢失

录制完成后务必**保存 Level Sequence 资产**（Ctrl+S 或右键 → Save）。帧缓冲数据嵌入在 Level Sequence 的 Package 中，未保存则关闭后丢失。

#### Q：录制时内存飙升

这是正常现象——NDI 帧缓冲以**未压缩 BGRA 格式**存储在内存中。

**缓解方法**：
- 降低 **Downscale**（如设为 0.5，实际 0.25 降采样）
- 降低 **TargetFPS**（如 15 fps）
- 缩短 **MaxDurationSeconds**
- 关闭不需要的 NDI 输入（仅录制必要源）

### 8.4 回放相关

#### Q：Sequencer 回放时显示的是实时 NDI 画面而不是录制内容

检查 NDI Section 的 **FrameBuffer** 属性是否正确关联了帧缓冲资产。如果为空，回放无法注入帧。

#### Q：Sequencer 回放时某些时间点黑屏

- 开启 Section 的 **HoldLastFrame** 选项
- 检查帧缓冲中是否确实有对应时间范围的帧数据

### 8.5 DLL 与初始化相关

#### Q：Output Log 显示 "SuperNDI: CRITICAL - Integrated NDI DLL not found!"

NDI SDK 的 DLL 文件缺失。请确保以下任一路径中存在 `Processing.NDI.Lib.x64.dll`：
- `Plugins/SuperStage/Binaries/Win64/`
- `Plugins/SuperStage/Source/SuperNdi/ThirdParty/NDI/Bin/x64/`
- `Plugins/SuperStage/Source/SuperNdi/ThirdParty/NDI/Bin/`（回退路径）

> **诊断信息**：启动时 Output Log 会输出详细的 DLL 搜索路径和加载结果（搜索 `SuperNDI` 关键字），可以帮助定位具体是哪条路径失败。

#### Q：Output Log 显示 "NDIlib_initialize failed"

可能原因：
- CPU 不支持 SSE 4.2 指令集（极老的 CPU）
- DLL 文件损坏
- 运行时依赖缺失（Visual C++ Redistributable）

#### Q：Output Log 显示 "Failed to create Persistent Finder"

可能原因：
1. NDI Runtime 未安装（从 [ndi.video](https://ndi.video) 下载）
2. NDI Studio Monitor 占用资源 → 关闭后重试
3. 防火墙阻止 UDP 5353（mDNS）
4. Bonjour 服务未运行

---

## 9. 性能优化建议

### 9.1 网络优化

| 建议 | 说明 |
|------|------|
| 使用**有线千兆网络** | WiFi 丢包率高，不推荐用于 NDI |
| NDI 源与接收端使用**独立网段** | 避免其他流量干扰 |
| 减少 NDI 源数量 | 每路 1080p NDI 占用约 100-200 Mbps |

### 9.2 分辨率优化

| 场景 | 推荐分辨率 |
|------|-----------|
| 预演/彩排 | 960×540（Downscale=1.0） |
| 最终渲染 | 1920×1080（Downscale=2.0） |
| 长时间录制 | 480×270（Downscale=0.5） |

### 9.3 帧率优化

| 场景 | 推荐帧率 |
|------|---------|
| 静态画面/幻灯片 | 10-15 fps |
| 普通视频 | 30 fps |
| 高速运动/音乐同步 | 60 fps |

### 9.4 内存优化

- 录制完成后立即**保存并关闭**不需要的 Level Sequence
- 使用**最小必要的降采样和帧率**
- 长时间录制分多个 Take 分段录制
- 监控 Windows 任务管理器中 UE 的内存占用

### 9.5 NDI 源端优化

- OBS 设置：NDI 输出分辨率与 UE 中实际需要的分辨率保持一致
- vMix 设置：选择低带宽输出模式（如 SHQ 而非 Full NDI）
- NDI 源尽量输出 **BGRA/BGRX 格式**（避免 UYVY/UYVA 的 CPU 转换开销）

### 9.6 接收帧率上限

SuperNDI 子系统使用 **50Hz Ticker** 轮询 NDI 接收器（每 20ms 一次），因此即使 NDI 源输出 60fps，实际接收帧率上限约为 **50fps**。对于绝大多数舞台预演场景（30fps），此限制不构成影响。

---

## 10. 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| NDI | Network Device Interface | NewTek 开发的低延迟 IP 视频传输协议 |
| 逻辑输入名 | Logical Input Name | 用户自定义的名称，用于在项目中标识一路 NDI 输入 |
| 外部源 | External Source | 网络中实际存在的 NDI 设备/软件的显示名称 |
| 帧缓冲 | Frame Buffer | 存储录制的 NDI 视频帧序列的内存/资产 |
| 降采样 | Downscale | 降低录制帧分辨率以节省内存 |
| 环回模式 | Loopback Mode | Sequencer 回放时屏蔽真实 NDI 帧，仅使用注入的录制帧 |
| mDNS | Multicast DNS | NDI 用于源发现的网络协议（UDP 端口 5353） |
| Take Recorder | — | UE 内置的录制系统，将实时数据录制到 Sequencer |
| Sequencer | — | UE 内置的影片编辑器/时间轴系统 |
| Section | — | Sequencer 轨道中的一个时间段/片段 |
| MID | Material Instance Dynamic | 运行时创建的动态材质实例 |
| RHI | Render Hardware Interface | UE 的渲染硬件抽象层 |
| BGRA | — | 像素格式：蓝-绿-红-透明，每通道 8 位 |
| sRGB | — | 标准 RGB 色彩空间（伽马校正） |
| 梯形校正 | Keystone Correction / Deformation | 通过 UV 偏移校正投影/显示的几何畸变 |

---

> **文档维护**：如有任何问题或建议，请联系 SuperStage 技术支持团队。

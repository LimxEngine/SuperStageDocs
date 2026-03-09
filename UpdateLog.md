# SuperStage 更新日志

> 记录 SuperStage 各版本的功能更新、Bug 修复与性能优化。

---

## 版本 26Q2.0

> 📦 **版本号**：26Q2.0
> 📅 **发布日期**：2026年3月6日
> 🔧 **基于版本**：26Q1.1

### 📐 SuperCAD — 施工图绘制系统（全新模块）

在 UE 编辑器内直接绘制专业施工图，告别 3D 预演完还要回 CAD 软件重新画图的痛苦。

**打开方式**：Window 菜单 → "SuperCAD - Lighting Plot"

**核心功能：**

- **正交视口**：俯视图 / 正立面 / 侧立面 / 后立面，三种渲染模式（线条/着色/光照）
- **灯具信息自动显示**：地址码、灯具编号、类型、功率、重量 —— 场景改了图纸自动更新
- **尺寸标注**：线性标注 + 连续标注，自动计算真实距离，支持捕捉对齐
- **文字标注**：自由放置文字说明
- **引线标注**：带箭头的折线标注，指向灯具或位置
- **电源/信号线路**：逐点绘制电力线、信号线路径
- **区域标注**：多边形区域标记，自动计算面积
- **框选 & 点选**：Window 框选（左→右）和 Crossing 框选（右→左），支持 Shift/Ctrl 多选
- **图层管理**：按类别分层（灯具/标注/电源/信号等），可独立开关可见性和颜色
- **多页面**：一份文档包含多张图纸（灯位图 + 电源图 + 信号图）
- **图框 & 标题栏**：A0~A4 标准图幅 + 自定义尺寸，可编辑项目信息/设计者/版本号
- **右键菜单**：空白/灯具/标注三种场景的快捷操作

**导出格式：**

| 格式 | 说明 |
|------|------|
| **DXF** | AutoCAD 兼容，可导入 VW / DraftSight，分图层输出 |
| **PDF** | 可直接打印交付 |
| **PNG** | 快速截图分享 |

**统计报表：**

- **灯具清单 (BOM)**：按类型汇总数量、功率、重量
- **Universe 使用率**：各 Universe 灯具数、通道占用、空闲通道
- **电源负载**：按回路汇总负载、估算三相电流
- **挂点载荷**：逐灯具列出重量和挂载高度

**数据管理：**

- 施工图保存为项目资产（.uasset），Content Browser 可直接双击打开
- 支持 Ctrl+Z / Ctrl+Shift+Z 撤销重做
- 自动记忆上次打开的文档

### 🚂 轨道机械系统强化（全新功能）

SuperStage 轨道机械系统全面升级。

- **编辑器实时预览**：编辑器模式下即可预览轨道运动轨迹，无需进入 Play 模式
- **闭合轨道智能插值**：环形轨道自动选择最短路径运动
- **边界安全保护**：轨道端点越界自动限位

### 🔆 灯光组件亮度分控（全新功能）

所有灯光组件新增 **ComponentDimmer** 参数（0~1），支持多模组灯具独立调节单个组件的亮度。

- 与 Actor 级 Dimmer 乘法叠加
- 适用于：Spot / Beam / Rect / Effect / Matrix / LaserPro 全部组件
- 用途举例：一台灯具有多个发光模组，可以单独控制每个模组的亮度

### 🏗️ SuperTruss — 程序化桁架系统（全新）

在场景中直接放置参数化桁架，支持实时调整尺寸。

- **截面类型**：方管 / 三角 / 平板
- **造型模式**：门型 / T型 / 门洞 / 双跨
- **工程计算**：DIN 4113 标准荷载计算
- 所有参数实时可调

### 🎭 SuperDrape — 程序化幕布系统（全新）

在场景中直接放置参数化幕布。

- **6 种幕布类型**：Standard / Austrian / Brail / Contour / Venetian / Kabuki
- **4 种褶皱样式**：Box Pleat / Pinch Pleat / Pencil Pleat / Flat
- **4 种开合模式**：Fly / Traveller / Austrian / Static
- 支持面料参数（密度/垂坠/弹性）

### ✨ 激光 Pro 组件（全新）

基于程序化网格的高性能激光系统。

- 支持 Beyond 协议实时接收
- 支持 Sequencer 录制播放
- UV深度衰减 + HDR亮度控制 + 多层烟雾效果 + 光斑效果
- 高性能：内存复用 + 变更检测跳过无变化帧

### 🚁 SuperDroneLink 生产级修复

3 轮深度审查，共修复 **17 项问题**，包括：

- PIE 模式下无人机无法接收信号和控制
- 内存安全、空指针防护、资源泄漏
- 线程安全、生命周期管理
- 边界条件、异常恢复

### 🔫 SuperLaser 生产级加固

代码审核共修复 **12 项问题**（P0×3 / P1×4 / P2×5），包括：

- AsyncTask lambda use-after-free 修复（AliveFlag 守卫）
- FCanvas 空资源指针崩溃防护
- 递归锁、浮点溢出、SEH 保护
- Sequencer 时间单位不匹配修复
- 线性搜索优化为二分查找、除零防护

### ⚡ 频闪系统重构

频闪计算从材质着色器迁移到组件 Tick，所有灯光类型统一频闪逻辑。

- 性能提升：避免每帧每像素的材质计算
- 频闪关闭时自动禁用 Tick
- 6 种波形：Linear / Pulse / RampUp / RampDown / Sine / Random
- LightStripEffect 灯带特效频闪同步迁移至 C++ 计算

### 🔦 Spot 辅光矩阵控制（全新）

为 SuperStageLight 的 Spot 辅光添加完整的矩阵控制：RGB / RGBW / 冷暖混合 / Zoom / 频闪 / 雾化。

### 💡 光束遮挡系统重构

光束遮挡检测与衰减半径更新全面重构，解决 GPU Scene 崩溃和多个累加 bug。

- **GPU Scene 安全**：新增 `SetAttenuationRadiusSafe()` 绕过 `PushRadiusToRenderThread()`，避免与 GPU Scene 同帧冲突
- **遮挡与 Zoom 解耦**：遮挡检测以 30fps 独立限流运行，不再依赖 Zoom 触发
- **Frost 累加修复**：BeamComponent 的 `SetLightingFrost` 改用缓存基础值叠加，消除无限累加
- **防御性检查**：`UpdateBeamBlockDistance` 全面 NaN/Inf 校验、钳位、去抖

### � 光束棱镜效果增强

棱镜激活时自动增大光束和光斑角度，还原真实灯具棱镜散射效果。

- 角度增量与棱镜半径线性关联（PrismRadius × 20°）
- 瞬变应用，与棱镜状态同步（非线性渐变）
- 正确处理 Zoom / Frost / Iris 同时开启的叠加场景
- 防 de-jitter 累加保护：棱镜角度在任何调用顺序下均保持稳定

### 🔄 无极旋转优化

新增 Pan / Tilt 无极旋转矩阵控制，支持顺时针/逆时针/停止三种模式。

### � SuperNDI 稳定性修复

NDI 视频流模块深度修复，提升运行时稳定性。

- MoveTemp 时序修正，防止帧缓冲双释放
- bNDIReady 守卫，避免未就绪时访问资源
- 资源泄漏修复 + 帧缓冲性能优化

### 📡 SuperDMX 生产级加固

代码审核共修复 **14 项问题**（P0×4 / P1×3 / P2×7），包括：

- 线程安全与并发保护
- 网络协议解析健壮性
- HttpService 切换正式环境

### � 权益系统重构

从单一订阅制升级为模块级权益控制。

- 每个模块独立授权（SuperStage / SuperConsolePro / SuperDroneLink / SuperLaser / SuperNDI）
- AES-256-GCM 加密令牌解密，安全性升级

### � 灯库编辑器（全新）

基于 MA2/GDTF 标准的灯库数据资产编辑器。

- 13 种 DMX 属性分类
- 三级导航：属性 → 子属性 → 槽位
- 支持多模块/矩阵灯配置
- 多选操作：添加、删除、复制、粘贴、排序

### 🎛️ DMX 起始域设置

解决 MA2 与 UE Universe 编号对齐问题。

- 默认值改为 0：MA2 的 1.1 直接对应 UE 的 1.1，无需手动偏移
- 旧项目兼容：如需恢复旧行为，将起始域改为 1 即可

### �️ SuperConsole 模块移除

SuperConsole 基础控台已被 **SuperConsolePro** 完全替代，本版本移除全部旧模块代码。无需迁移，SuperConsolePro 继承并扩展了所有功能。

### 📖 全模块用户手册

新增完整的用户手册文档，覆盖所有模块：

- SuperStage — 灯光系统与场景管理
- SuperConsolePro — 控台操作指南
- SuperDroneLink — 无人机编队控制
- SuperLaser — 激光系统
- SuperNDI — NDI 视频流
- SuperStageEditor — 编辑器工具

### 🛠️ 开发者 API 文档

新增 7 篇开发者技术文档（Source/SuperStage/DevDocs/）：

- 架构总览 — 模块分层与数据流
- SuperDMX API — 网络通信与缓存接口
- SuperDmxActorBase API — DMX 数据读取与通道映射
- SuperLightBase / SuperStageLight API — 灯光 Actor 接口
- 其他 Actor API — 激光/NDI/投影/无人机等
- 组件 API — Spot/Beam/Rect/Effect/Matrix/Laser 组件接口
- 灯具开发指南 — 从零创建自定义灯具的完整流程

### 📋 工具栏菜单优化

- 新增 Modules 子菜单，集中管理 SuperCAD / SuperConsolePro 入口
- 移除 Window 菜单中多余的 SuperUI 条目

### 🐛 Bug 修复

- 修复 SuperSpot 属性名冲突
- 修复 Spot 辅光参数与主光源不一致
- 修复频闪算法在矩阵组件上的不一致
- 修复灯具初始化函数未被调用
- 修复 PatchTool 灯具排序：一键全选现在按自然排序（C_1 < C_2 < ... < C_10 < C_100）
- 修复 SuperConsolePro 属性轮和属性栏显示不全，移除硬编码属性名称

### 📥 升级说明

从 26Q1.1 升级到 26Q2.0：

1. 直接覆盖安装即可，项目文件完全兼容
2. **注意**：材质中的频闪参数已移除，如有自定义材质需同步更新
3. **注意**：SuperConsole 模块已移除，请使用 SuperConsolePro
4. **注意**：权益系统已重构，旧版订阅令牌需重新获取
5. **新增**：SuperCAD 施工图 → 工具栏 Modules 菜单 → SuperCAD
6. **新增**：SuperTruss / SuperDrape → 直接在场景中放置使用
7. **新增**：轨道机械编辑器预览 → 编辑器模式下可直接预览运动轨迹
8. **新增**：全模块用户手册文档

---
---
---

## 版本 26Q1.1

> 📦 **版本号**：26Q1.1
> 📅 **发布日期**：2026年2月3日
> 🔧 **基于版本**：26Q1.0

### 🚁 SuperDroneLink 无人机链接模块（全新）

全新推出 **SuperDroneLink** 无人机灯光控制模块，支持通过 LDLink 协议控制无人机编队灯光。

**核心功能：**

- **LDLink 协议支持**：实时接收无人机位置和状态数据
- **DroneSwarmManager**：高性能无人机群管理器，使用 HISM 优化渲染
- **SuperDroneActor**：单体无人机 Actor，支持独立控制
- **Sequencer 录制**：支持将无人机轨迹录制到 LevelSequence
- **活动监视器**：实时显示无人机连接状态和数据流

**技术特性：**

- 使用线性插值实现平滑运动
- 自动单位转换（米 → 厘米）
- 支持网卡选择和自动发现
- 继承 ASuperBaseActor 统一架构

### 💡 灯光控制增强

#### 颜色混合系统

**新增混色函数：**

- `SetLightingColorRGBW`：RGBW 四通道混色
- `SetLightingColorRGBWWithCTO`：RGBW + CTO 色温混色
- `SetLightingColorMixWithCTO`：多通道混色 + CTO 色温调节
- `SetLightingColorHSAndCTO`：色相饱和度 + CTO 控制
- `SetLightingColorHS` / `SetLightingColorHSV`：HSV 颜色控制
- `SetLightingColorWheel2AndCMYAndCTO`：CMY + CTO 色温滤片

**矩阵组件扩展颜色控制（全新）：**

- RGB + CTO / RGBW / RGBW + CTO
- RGB + CoolWarm / RGBW + CoolWarm
- CoolWarm 混合 / 多色混合 (`SetMatrixColorMix`)

#### 图案控制优化

**Gobo 抖动功能（全新）：**

- **EGoboMode 枚举**：固定 (Static)、流水 (Scroll)、抖动 (Shake)
- 材质端实现：抖动幅度 25 度，速度 0-10Hz
- 中文关键字支持：识别"抖动"、"摇摆"等关键字

**图案 / 棱镜旋转分离：**

- 分离图案旋转 (Gobo) 和棱镜旋转 (Prism)
- `SetBeamCuttingRotate` 只控制 Shaper + Gobo
- 所有 GoboMode 都计算 GoboIndex

#### 旋转速度控制优化

- 物理范围控制：PT Speed、图案 / 棱镜 / Shaper 旋转使用物理范围
- 无极旋转模式：新增 `EInfiniteRotationMode` 枚举

### 🎬 效果组件增强

**频闪系统修复：**

- 修复矩阵组件分段光源频闪算法与材质一致
- 修复矩阵频闪模式索引对应问题
- 初始化 Brightness 和 StrobeMode 参数

**效果速度控制：**

- `SetEffectWithSpeed`：效果 + 独立速度通道
- `SetEffectMatrixWithSpeed`：矩阵效果 + 速度通道

### 🔧 工具与优化

- 颜色 / 图案图集生成器命名规则优化
- 颜色轮流水效果修复
- 灯库从 Actor 内联改为数据资产
- AssetBrowser 按类扫描蓝图，启动时同步预热
- DroneSwarmManager HISM 性能优化

### 🐛 Bug 修复

- 修复 Iris 光圈 DMX 值反转和最小值限制
- 修复 FrameSubsystem 使用 IPluginManager 查找插件路径
- 修复 PatchSubsystem ScanScene 时始终更新 Modules 数据
- 修复 TemperatureToRGB 前置声明顺序
- 修复 SetBeamPrism 中 CalcGoboRotation 调用参数

### 📋 提交摘要

| 类别 | 数量 | 说明 |
|:-----|:----:|:-----|
| 新功能 | 25+ | 无人机模块、混色系统、抖动效果等 |
| Bug 修复 | 20+ | 频闪、颜色轮、索引对应等 |
| 优化重构 | 15+ | 性能优化、代码重构 |
| 灯具资产 | 10+ | 新增 / 更新灯具配置文件 |

### 📥 升级说明

1. 直接覆盖安装即可
2. 项目文件完全兼容，无需迁移
3. 建议重新编译项目以获得所有新功能

---

## 版本 26Q1.0

> 🔴 **强制更新**
> 📅 **发布日期**：2026年1月20日

### ⚠️ 重要提示

本次为**强制更新**版本。由于服务端全面重构，旧版本将无法连接服务器。

- 未更新用户将无法正常使用插件
- 本次更新不会影响已完成或正在进行中的项目文件

### 🔄 核心架构与版本变更

- **域名变更**：官网及插件 API 域名由 `ue.yunsio.com` 迁移至 `yunsio.com`
- **全新版本号命名**：弃用原 1.x/2.x 格式，启用年份 / 季度命名制（如 26Q1.0 = 2026年第一季度第0版）
- **更新计划**：后续保持每季度一次大版本迭代

### ✨ 新增功能与优化

**灯光与特效：**

- 高性能激光 (Beta)：全新激光系统，支持更高性能的实时渲染
- 升降矩阵灯：支持升降矩阵灯及升降无人机矩阵灯
- 高级混色：支持 RGBA / RGBL / RGBALM 等专业色彩模式
- 灯具性能深度优化，大规模灯光场景更流畅

**工具与系统：**

- DMX 相机：可通过 DMX 信号控制相机参数与运动
- 服务端全面重构，鉴权机制更严格，登录连接更稳定
- NDI 兼容性提升

---

## 版本 2.1.3

> 📅 **发布日期**：2025年12月16日

### 🎚️ SuperConsolePro 专业灯光控台（全新）

全新推出 **SuperConsolePro** 专业级灯光控台系统，为舞台灯光、演出现场、虚拟制作提供完整的灯光编程和回放解决方案。

**核心特性：**

- 模块化子系统架构：15 个独立子系统协同工作
- 专业级编程工作流：Programmer、预设、CUE 等概念
- 实时 DMX 输出：延迟 < 16ms
- 时间线编辑器：CUE 轨道 + 音频轨道多轨编辑
- 帧效果引擎：正弦、方波、锯齿波等波形
- 灵活窗口布局：多窗口自由布局，10 个预设槽位
- 演出文件管理：.ssshow 格式保存 / 加载

**主要功能模块：**

| 层级 | 模块 |
|------|------|
| 数据管理 | 演出文件、配接、属性子系统 |
| 编程工作流 | 选择、编程器、编码器、预设子系统 |
| 回放控制 | CUE、回放运行时、快捷键子系统 |
| 高级功能 | 时间线、帧效果、布局子系统 |
| 输出控制 | 控台 DMX、控制栏、时间码 CUE 子系统 |

**UI 面板系统（12+ 面板）：**

灯具表、回放、预设、时间线、帧编辑器、布局、颜色选择器、组、配接、运行中回放、DMX 监控、设置

---

## 版本 2.0.0

> 📅 **发布日期**：2025年12月6日

### 🚨 重要变更

- **废除启动器**，改用网盘下载（百度 / 夸克 / 谷歌）
- 改为 UE 编辑器内认证面板，使用邮箱验证码登录

### 🎯 主要更新

- UE 5.7 版本无缝兼容
- 新增永久包 3 年更新、单版本永久订阅、源码版商用版本

### 🔄 SuperData 协议

推出 **SuperData 协议**，支持多软件间灯光设备数据实时互通：

- 支持 UE、Unity、VW、MA2、Blender、C4D 等软件
- 使用网络通信技术，无需导出文件

### 📦 SuperStage 多版本计划

| 版本 | 状态 |
|:-----|:----:|
| SuperStageForUE | ✅ 更新 2.0 |
| SuperStageForLimx | 🔧 开发中 |
| SuperStageForUnity | 🧪 内测中 |
| SuperStageForVW | ✅ 已上线（免费） |
| SuperStageForMA | ✅ 已上线（免费） |
| SuperStageForBlender | 📋 计划中 |
| SuperStageForC4D | 📋 计划中 |

### 🎮 SuperStageForUnity（内测中）

- 插件架构、使用方法、参数与 UE 版本基本相同
- 支持舞台 DMX 设备、机械、灯带、SuperData 协议
- 暂不支持：激光灯、NDI、Super 内置控台

### 🎚️ Super 内置控台（预览版）

- 扫描灯具自动 Layout 布局
- 灯组管理 / CUE 列表 / 内置效果系统
- 关键帧效果系统（首尾帧编程）
- 时间线系统（声光同步，可转为 Sequence 渲染）

### 🔐 认证授权优化

- 登录状态最长可保留 7 天
- 网络连接失败不丢失登录状态
- 增加失败重连机制

---

## 版本 1.2.1

> 📅 **发布日期**：2025年11月

### 更新内容

- 废除导出 CSV / XML 导入功能
- 解除蓝图加密限制
- 新增 Beyond 激光、连接、渲染模块
- 修复认证鉴权 Bug、内存泄漏与后台进程 Bug
- 修复棱镜旋转问题，优化所有规范问题
- 新增 Windows 平台打包支持（所有用户均可打包）

> ⚠️ 更新前需手动删除旧版插件目录

---

## 版本 1.2.0

> 📅 **正式版发布**

### 开放内容

- 开放所有灯具
- 开放 DMX 一键配接、配接预览、导出至 MA2、CSV 导出工具
- 开放所有舞台资产与特效
- 开放 NDI 屏幕连接与渲染模块
- 开放 VWXml / Mvr 导入工具、观众资产
- 共计 **118 个资产**

---

## 版本 1.0.0

> 📅 **发布日期**：2025年9月20日

### 首次发布

- SuperStage 插件安装程序首次发布
- 支持 Unreal Engine 5.6
- 自动从 Windows 注册表检测 UE 安装路径
- 多语言支持（英文和中文）
- 向导式安装界面
- 插件安装至 `[UE路径]\Engine\Plugins\Marketplace\SuperStage`

---

**感谢使用 SuperStage！如有问题请访问 [yunsio.com](https://yunsio.com) 或发送邮件至 yunsio@yunsio.com**

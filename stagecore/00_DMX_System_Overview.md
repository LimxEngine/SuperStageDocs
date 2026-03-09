# SuperStage DMX 系统 — 用户手册总览

> **版本**: SuperStage 26Q1  
> **适用对象**: 舞美设计师、灯光编程师、虚拟制作技术人员  
> **最后更新**: 2026-03-06

---

## 一、SuperStage 是什么

SuperStage 是一款运行在 Unreal Engine 5 中的**专业舞美可视化插件**。它的核心使命是让灯光设计师能够在虚拟 3D 场景中，使用真实的 DMX 信号实时控制虚拟灯具，实现与真实演出完全一致的灯光效果预览。

简单来说：**你在 MA 控台上推推杆，UE 里的虚拟灯就跟着动**。

### 核心能力一览

| 能力 | 说明 |
|------|------|
| **DMX 实时接收** | Art-Net 协议，UDP 6454，毫秒级响应 |
| **DMX 实时输出** | Sequencer 回放时反向输出 Art-Net 到真实灯具 |
| **全品类灯具** | 电脑灯 / 染色灯 / 矩阵灯 / 切割灯 / LED灯带 / 激光 / 特效机 / 升降矩阵 / 舞台机械 / 摄像机 / 投影仪 / NDI屏幕 |
| **专业颜色系统** | RGB / RGBW / HSV / ColorWheel / CMY / CTO / CoolWarm / ColorMix（任意通道混色） |
| **图案与棱镜** | 最多3个Gobo轮 + 3个棱镜轮，静态/无极旋转 |
| **切割系统** | 4叶片8通道 Shaper，叠加旋转 |
| **矩阵控制** | 像素级独立颜色/亮度/频闪，支持任意规模矩阵 |
| **Sequencer 集成** | 录制 DMX → Sequencer 轨道，可回放/导出 |
| **MA 控台导出** | 一键生成 MA2/MA3 可导入的灯具配置文件 |
| **多平台协议** | Art-Net 输入/输出 + Pangolin Beyond（激光）+ NDI（视频） |

---

## 二、DMX 是什么（快速科普）

DMX512 是舞台灯光行业的标准控制协议。你需要了解以下几个核心概念：

| 概念 | 说明 |
|------|------|
| **Universe（域）** | 一条 DMX 线路，包含 512 个通道。可以理解为"一根信号线" |
| **Channel（通道）** | 每个 Universe 中的一个控制值，范围 0-255（8位）。每个通道控制灯具的一个功能（如亮度、颜色、位置等） |
| **Address（地址）** | 灯具在 Universe 中的起始通道编号（1-512）。类似"门牌号" |
| **Fixture（灯具）** | 一台灯。一台灯通常占用多个连续通道（例如一台电脑灯可能占用 30 个通道） |
| **Fixture ID** | 灯具的唯一编号，用于与 MA 控台对应 |
| **Art-Net** | 一种通过以太网传输 DMX 信号的协议。SuperStage 默认使用此协议，端口 6454 |

### DMX 通道精度

| 精度 | 字节数 | 值范围 | 用途 | SuperStage 支持 |
|------|--------|--------|------|-----------------|
| **8 位 (Coarse)** | 1 个通道 | 0 - 255 | 基本控制（颜色、Gobo 等） | ✅ `GetAttributeRaw8ByIndex` |
| **16 位 (Fine)** | 2 个通道 | 0 - 65,535 | 精细控制（Pan/Tilt 等需要平滑运动的功能） | ✅ `GetAttributeRaw16ByIndex` |
| **24 位 (Ultra)** | 3 个通道 | 0 - 16,777,215 | 超精细控制（极少使用） | ✅ `GetAttributeRaw24ByIndex` |

---

## 三、完整类继承体系

SuperStage 所有 Actor 类形成一棵清晰的继承树。理解这棵树是正确使用本系统的基础。

### 3.1 继承树总览

```
AActor (UE5 引擎基类)
  │
  └── ASuperBaseActor ·················· 所有 SuperStage Actor 的根基类
        │                                 ├ SceneBase（根组件）
        │                                 ├ FAssetMetaData（UUID/制造商/分组元数据）
        │                                 └ 方向预览箭头（ForwardArrow / UpArrow）
        │
        ├── ASuperDmxActorBase ·········· DMX 灯具基类（增加 DMX 读取能力）
        │     │                            ├ FSuperDMXFixture（Universe / StartAddress / FixtureID）
        │     │                            ├ USuperFixtureLibrary*（灯具库引用）
        │     │                            ├ GetChannelValue / GetAttributeRaw8/16/24ByIndex
        │     │                            ├ GetMatrixAttributeRaw / Raw16（矩阵批量读取）
        │     │                            ├ GetSuperDmxAttributeValue（归一化读取 0~1）
        │     │                            ├ SuperDMXTick（蓝图每帧事件）
        │     │                            ├ ForceRefreshDMX（编辑器强制刷新）
        │     │                            └ AddressLabel（地址码文本显示）
        │     │
        │     ├── ASuperLightBase ······· 电脑灯基类（增加 Pan/Tilt 控制层）
        │     │     │                      ├ SceneRocker → SceneHard（灯臂→灯头旋转轴）
        │     │     │                      ├ Pan/Tilt 范围控制（默认 ±270°/±135°）
        │     │     │                      ├ 无极旋转（PanRot/TiltRot 连续旋转模式）
        │     │     │                      ├ PTSpeed 速度控制
        │     │     │                      ├ 矩阵 Pan/Tilt（YMatrixPan/YMatrixTilt）
        │     │     │                      ├ Z 轴升降（SetLiftZ，LiftRange 默认 500cm）
        │     │     │                      └ 灯具角度（LampAngle 手动固定模式）
        │     │     │
        │     │     ├── ASuperStageLight · 全功能电脑灯（完整灯具实现）  → 文档 15
        │     │     │                      ├ 光束：Dimmer/Strobe/Zoom/Focus/Frost/Iris
        │     │     │                      ├ 颜色：RGB/RGBW/HSV/ColorWheel/CMY/CTO/CoolWarm/ColorMix
        │     │     │                      ├ 图案：最多 3 个 Gobo 轮 + 旋转
        │     │     │                      ├ 棱镜：最多 3 个 Prism 轮 + 旋转
        │     │     │                      ├ 切割：4 叶片 Shaper（A1/B1~A4/B4）+ 旋转
        │     │     │                      ├ 效果：Effect LUT 系统（10 种内置效果）
        │     │     │                      ├ 矩阵：SuperMatrixComponent（像素级 RGB/白光/频闪）
        │     │     │                      └ Spot 辅光：SuperSpotComponent（真实 SpotLight 补光）
        │     │     │
        │     │     ├── ASuperStageVFXActor 舞台特效 VFX                 → 文档 14
        │     │     │                      ├ UNiagaraComponent（粒子发射器）
        │     │     │                      ├ DMX → SpawnCount / Color
        │     │     │                      └ 继承 Pan/Tilt（可控制喷射方向）
        │     │     │
        │     │     └── ASuperLiftMatrix ·· 升降矩阵灯阵                 → 文档 12
        │     │                            ├ 5 层升降组件 + 4 根钢丝绳
        │     │                            ├ 10 个 SuperEffectComponent
        │     │                            ├ DMX 升降 + 矩阵效果/颜色控制
        │     │                            └ 继承 Pan/Tilt
        │     │
        │     ├── ASuperDMXCamera ········ DMX 摄像机                    → 文档 05
        │     │                            ├ UCineCameraComponent（电影摄像机）
        │     │                            ├ USceneCaptureComponent2D（渲染到 RT）
        │     │                            ├ 6 轴运动：PosX/Y/Z + RotX/Y/Z
        │     │                            └ 镜头参数：FOV / Aperture / FocusDistance
        │     │
        │     ├── ASuperLiftingMachinery · 升降机械（6 轴）              → 文档 10
        │     │                            ├ 3 轴位移 + 3 轴旋转
        │     │                            ├ 绝对模式 / 无极旋转模式
        │     │                            └ BootRefresh 初始化范围
        │     │
        │     ├── ASuperRailMachinery ···· 轨道机械（7 轴）              → 文档 10
        │     │                            ├ USplineComponent（样条轨道路径）
        │     │                            ├ RailMountPoint + OffsetComponent
        │     │                            ├ 轨道位置 + 3 轴偏移 + 3 轴旋转
        │     │                            └ 闭合循环 / 朝向锁定
        │     │
        │     └── ASuperLightStripEffect · LED 灯带特效                  → 文档 13
        │                                  ├ GPU 材质特效（10 种内置效果）
        │                                  ├ 9 通道 DMX 控制
        │                                  └ 多目标 StaticMesh 批量应用
        │
        ├── ASuperLaserActor ·············· 激光（纹理投射模式）         → 文档 11
        │                                  └ RenderTarget 纹理投射
        │
        ├── ASuperLaserProActor ··········· 激光（点数据精确模式）       → 文档 11
        │                                  └ 激光点坐标 → 线条渲染 + 碰撞检测
        │
        ├── ASuperProjector ··············· 投影仪
        │                                  └ 视频/图像投射
        │
        ├── ASuperNDIScreen ··············· NDI 屏幕
        │                                  └ NDI 视频流接收 → 材质纹理更新
        │
        ├── ASuperScaffold ················ 脚手架（参数化生成）
        ├── ASuperCurvedScaffold ·········· 弧形脚手架（参数化生成）
        ├── ASuperTruss ··················· 桁架（参数化生成）
        └── ASuperDrape ··················· 幕布（程序化褶皱生成）
```

### 3.2 各层级职责说明

| 层级 | 类名 | 职责 | 新增能力 |
|------|------|------|----------|
| **L0** | `AActor` | UE5 引擎基类 | Tick、Transform、组件系统 |
| **L1** | `ASuperBaseActor` | SuperStage 根基类 | SceneBase 根组件、资产元数据（UUID/制造商/分组）、方向预览箭头 |
| **L2** | `ASuperDmxActorBase` | DMX 数据读取层 | DMX Fixture 地址（Universe/Address/FixtureID）、灯具库引用、通道值读取（8/16/24位）、矩阵批量读取、归一化读取、蓝图事件 SuperDMXTick、地址标签显示 |
| **L3** | `ASuperLightBase` | Pan/Tilt 控制层 | SceneRocker→SceneHard 旋转轴、Pan/Tilt 范围映射、无极旋转模式、PTSpeed 速度控制、矩阵 Pan/Tilt、Z 轴升降、灯具角度固定 |
| **L4** | `ASuperStageLight` | 全功能灯具实现 | 光束/颜色/图案/棱镜/切割/效果/矩阵/Spot辅光 完整实现 |

> **设计原则**：每一层只增加该层该管的功能，不越界。例如 `ASuperDmxActorBase` 只管 DMX 数据读取，完全不知道 Pan/Tilt 是什么；`ASuperLightBase` 只管 Pan/Tilt 旋转，完全不知道颜色/图案是什么。这种分层设计使得你可以在任何层级派生新的 Actor 类型。

### 3.3 "我该用哪个类？" — 选型决策表

| 你想做什么 | 选哪个基类 | 典型场景 |
|------------|-----------|---------|
| 需要完整电脑灯功能（颜色/图案/棱镜/切割/效果/矩阵） | **直接使用 ASuperStageLight** | Martin MAC Viper、Robe MegaPointe、ClayPaky Sharpy |
| 只需要 DMX 控制 + Pan/Tilt 旋转 | **继承 ASuperLightBase** | 简易追光灯、旋转台灯 |
| 只需要 DMX 数据读取（无旋转） | **继承 ASuperDmxActorBase** | LED 条屏、DMX 继电器、自定义设备 |
| 不需要 DMX，只需要 SuperStage 基础功能 | **继承 ASuperBaseActor** | 舞台道具、桁架、幕布 |
| 激光可视化 | **ASuperLaserActor / ASuperLaserProActor** | Pangolin Beyond 对接 |
| Niagara 粒子特效 | **ASuperStageVFXActor** | 烟雾机、火焰、CO2、雪花 |
| 虚拟摄像机 | **ASuperDMXCamera** | 演播室/LED 虚拟拍摄 |
| 舞台升降/旋转 | **ASuperLiftingMachinery** | 升降台、旋转舞台 |
| 轨道运动 | **ASuperRailMachinery** | 轨道车、移动吊杆 |
| LED 灯带效果 | **ASuperLightStripEffect** | 灯条、轮廓灯 |
| 升降矩阵灯阵 | **ASuperLiftMatrix** | LED 升降球/管 |

---

## 四、全 Actor 功能对比矩阵

以下矩阵列出了所有 DMX 控制 Actor 的功能支持情况，帮助你快速对比选型：

| 功能 | StageLight | LightBase | VFXActor | LiftMatrix | DMXCamera | LiftMach | RailMach | StripEffect |
|------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **DMX 数据读取** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **灯具库 (Fixture Library)** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Pan/Tilt 旋转** | ✅ | ✅ | ✅ | ✅ | — | — | — | — |
| **无极旋转** | ✅ | ✅ | ✅ | ✅ | — | ✅ | ✅ | — |
| **Z 轴升降** | ✅ | ✅ | ✅ | ✅ | — | — | — | — |
| **亮度 (Dimmer)** | ✅ | — | — | ✅ | — | — | — | ✅ |
| **频闪 (Strobe)** | ✅ | — | — | — | — | — | — | ✅ |
| **RGB 颜色** | ✅ | — | ✅ | ✅ | — | — | — | ✅ |
| **RGBW 颜色** | ✅ | — | — | — | — | — | — | — |
| **HSV 颜色** | ✅ | — | — | — | — | — | — | — |
| **颜色轮 (ColorWheel)** | ✅ (1~3轮) | — | — | — | — | — | — | — |
| **CMY 混色** | ✅ | — | — | — | — | — | — | — |
| **CTO 色温** | ✅ | — | — | — | — | — | — | — |
| **冷暖光混色** | ✅ | — | — | — | — | — | — | — |
| **动态多通道混色 (ColorMix)** | ✅ | — | — | — | — | — | — | — |
| **Zoom 变焦** | ✅ | — | — | — | — | — | — | — |
| **Focus 调焦** | ✅ | — | — | — | — | — | — | — |
| **Frost 雾化** | ✅ | — | — | — | — | — | — | — |
| **Iris 光圈** | ✅ | — | — | — | — | — | — | — |
| **Gobo 图案轮** | ✅ (1~3轮) | — | — | — | — | — | — | — |
| **Prism 棱镜** | ✅ (1~3轮) | — | — | — | — | — | — | — |
| **Cutting 切割 (Shaper)** | ✅ | — | — | — | — | — | — | — |
| **Effect LUT 效果** | ✅ | — | — | ✅ | — | — | — | ✅ |
| **Matrix 矩阵** | ✅ | — | — | — | — | — | — | — |
| **Spot 辅光 (SpotLight)** | ✅ | — | — | — | — | — | — | — |
| **3 轴位移** | — | — | — | — | ✅ | ✅ | ✅(偏移) | — |
| **3 轴旋转** | — | — | — | — | ✅ | ✅ | ✅ | — |
| **轨道运动 (Spline)** | — | — | — | — | — | — | ✅ | — |
| **FOV / 光圈 / 对焦** | — | — | — | — | ✅ | — | — | — |
| **渲染到 RenderTarget** | — | — | — | — | ✅ | — | — | — |
| **Niagara 粒子** | — | — | ✅ | — | — | — | — | — |
| **GPU 材质特效** | — | — | — | — | — | — | — | ✅ |
| **多目标批量应用** | — | — | — | — | — | — | — | ✅ |

---

## 五、系统整体架构

SuperStage 的 DMX 系统由以下几个部分组成，它们协同工作：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           外部设备层                                     │
│   MA2/MA3 控台 │ grandMA3 onPC │ 其他 Art-Net 控台 │ Pangolin Beyond   │
└───────┬──────────────────┬──────────────────────────┬───────────────────┘
        │ Art-Net           │ Art-Net                  │ Beyond Laser Data
        │ UDP 6454          │ UDP 6454                 │ (TCP/共享内存)
        ▼                   ▼                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      SuperDMX 引擎子系统                                 │
│                                                                          │
│  ┌──────────┐    ┌──────────────┐    ┌───────────────────┐              │
│  │ UDP 接收  │───▶│ Art-Net 解析  │───▶│  Universe 缓冲区   │             │
│  │  (输入)   │    │  OpCode 判断  │    │  (512ch × N 个域)  │             │
│  └──────────┘    └──────────────┘    └─────────┬─────────┘              │
│                                                  │                       │
│  ┌──────────┐    ┌──────────────┐               │ GetDMXValue()         │
│  │ UDP 发送  │◀───│ Art-Net 构建  │◀──────────────┘ SendDMXBuffer()      │
│  │  (输出)   │    │  数据帧封装   │    (Sequencer 回放 / SuperConsolePro) │
│  └──────────┘    └──────────────┘                                       │
└────────────────────────┬────────────────────────────────────────────────┘
                         │ 每帧 Tick → 读取 DMX 通道值
                         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      虚拟灯具 Actor 层                                    │
│                                                                          │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐           │
│  │  电脑灯           │ │  DMX 摄像机      │ │  舞台机械         │          │
│  │  SuperStageLight  │ │  SuperDMXCamera  │ │  LiftingMach     │          │
│  │  ─────────────── │ │  ─────────────── │ │  RailMachinery   │          │
│  │  光束/颜色/图案   │ │  6轴位移/旋转    │ │  6轴/7轴运动     │          │
│  │  棱镜/切割/效果   │ │  FOV/光圈/对焦   │ │  绝对/无极模式   │          │
│  │  矩阵/Spot辅光    │ │  渲染到RT        │ │  样条轨道        │          │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘           │
│                                                                          │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐           │
│  │  LED 灯带特效     │ │  舞台 VFX 特效   │ │  升降矩阵         │          │
│  │  LightStripEffect│ │  StageVFXActor  │ │  LiftMatrix      │          │
│  │  ─────────────── │ │  ─────────────── │ │  ─────────────── │          │
│  │  10种GPU材质效果  │ │  Niagara粒子    │ │  5层升降+钢丝绳  │          │
│  │  9通道DMX控制     │ │  SpawnCount+RGB │ │  矩阵效果/颜色   │          │
│  │  多目标批量应用   │ │  Pan/Tilt继承   │ │  Pan/Tilt继承    │          │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘           │
│                                                                          │
│  ┌─────────────────┐ ┌─────────────────┐                                │
│  │  激光（纹理投射） │ │  激光（点数据）  │                                │
│  │  SuperLaserActor │ │  LaserProActor  │                                │
│  │  ─────────────── │ │  ─────────────── │                                │
│  │  RenderTarget    │ │  线条渲染+碰撞  │                                │
│  │  Beyond对接      │ │  Beyond对接      │                                │
│  └─────────────────┘ └─────────────────┘                                │
│                                                                          │
│  每台灯具通过「灯具库 (Fixture Library)」定义其通道表                      │
│  通过「DMX Patch」指定 Universe + 起始地址 + FixtureID                    │
└─────────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         工具链                                            │
│                                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │Patch 工具 │  │活动监视器 │  │DMX 录制   │  │MA 导出    │  │DMX 配置  │ │
│  │批量地址   │  │实时通道值 │  │Sequencer │  │MA2/MA3   │  │Art-Net   │ │
│  │分配       │  │查看      │  │回放输出   │  │灯具配置  │  │网络设置  │ │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### 核心数据流（信号怎样从控台到灯具）

```
① 控台发出 DMX 信号
   ↓ Art-Net UDP 数据包 (端口 6454)
② SuperDMX 子系统接收 → 解析 Art-Net OpCode → 提取 Universe 编号 + 512 通道值
   ↓ 存入 Universe 缓冲区（线程安全，游戏线程随时可读）
③ 灯具 Actor Tick
   ↓ 根据自身 Universe + StartAddress 从缓冲区读取通道值
④ 灯具库通道映射
   ↓ FSuperDMXModuleInstance → FSuperDMXAttributeDef → Coarse/Fine/Ultra
⑤ 参数应用
   ↓ DMX 值 → Pan 角度 / 颜色 / 亮度 / Gobo / 位置 等
⑥ Unreal Engine 实时渲染
```

### 模块依赖关系

```
┌──────────────────┐     ┌──────────────────┐
│    SuperDMX      │     │   SuperStage     │
│  (Runtime 模块)  │◀────│  (Runtime 模块)  │
│  ─────────────── │     │  ─────────────── │
│  USuperDMXSubsystem │  │  所有灯具 Actor  │
│  Art-Net 收发    │     │  灯具库资产      │
│  Universe 缓冲区 │     │  灯光组件        │
└──────────────────┘     └──────────────────┘
                                 ▲
                         ┌───────┴──────────┐
                         │ SuperConsolePro  │
                         │  (Editor 模块)   │
                         │  ─────────────── │
                         │  虚拟控台 UI     │
                         │  Phaser / Cue    │
                         └──────────────────┘
```

---

## 六、模块导航

SuperStage DMX 系统包含以下功能模块，每个模块都有独立的详细文档：

### 核心灯具模块

| 文档 | 模块 | 说明 |
|------|------|------|
| [03 - DMX 灯具基础](03_DMX_Actor_Base.md) | `ASuperDmxActorBase` | **所有 DMX 灯具的通用基础**：Universe、地址、FixtureID、灯具库引用、DMX 通道读取（8/16/24位）、矩阵批量读取、地址标签显示 |
| [04 - 电脑灯基类](04_Computer_Light.md) | `ASuperLightBase` | **Pan/Tilt 控制层**：水平/垂直旋转、无极旋转、速度控制、矩阵灯头、Z 轴升降、灯具角度 |
| [15 - 全功能电脑灯](15_SuperStageLight.md) | `ASuperStageLight` | **完整灯具实现**：光束（Dimmer/Strobe/Zoom/Focus/Frost/Iris）、颜色系统（RGB/RGBW/HSV/ColorWheel/CMY/CTO/CoolWarm/ColorMix）、图案（Gobo×3）、棱镜（Prism×3）、切割（Shaper 4叶片）、效果（Effect LUT 10种）、矩阵（像素级控制）、Spot辅光 |

### 专用灯具模块

| 文档 | 模块 | 说明 |
|------|------|------|
| [05 - DMX 摄像机](05_DMX_Camera.md) | `ASuperDMXCamera` | 6 轴位移/旋转 + FOV/光圈/对焦 + 渲染到 RenderTarget |
| [10 - 舞台机械](10_Stage_Machinery.md) | `ASuperLiftingMachinery` / `ASuperRailMachinery` | 升降台/旋转台（6轴）、轨道车（7轴，样条曲线路径） |
| [11 - 激光系统](11_Laser_System.md) | `ASuperLaserActor` / `ASuperLaserProActor` | Pangolin Beyond 对接：纹理投射模式 / 点数据精确线条模式 |
| [12 - 升降矩阵](12_Lift_Matrix.md) | `ASuperLiftMatrix` | 5 层升降组件 + 4 根钢丝绳，矩阵效果/颜色控制 |
| [13 - LED 灯带特效](13_Light_Strip_Effect.md) | `ASuperLightStripEffect` | 10 种 GPU 材质效果，9 通道 DMX 控制，多目标批量应用 |
| [14 - 舞台特效 VFX](14_Stage_VFX.md) | `ASuperStageVFXActor` | DMX 控制 Niagara 粒子系统：烟雾/火焰/雪花/CO2/烟花 |

### 系统配置与工具

| 文档 | 模块 | 说明 |
|------|------|------|
| [01 - DMX 网络配置](01_DMX_Network_Configuration.md) | DMX 配置面板 | Art-Net 协议设置、IP 地址、端口、输入/输出开关、Universe 起始偏移 |
| [02 - 灯具库](02_Fixture_Library.md) | Fixture Library | 创建和编辑灯具的通道定义表（属性名→通道偏移→精度→子属性） |
| [06 - DMX 活动监视器](06_DMX_Activity_Monitor.md) | Activity Monitor | 实时查看每个 Universe 每个通道的当前值 |
| [07 - Patch 工具](07_Patch_Tools.md) | Patch Tool & Preview | 批量分配 DMX 地址，预览和编辑所有灯具的 Patch 信息 |
| [08 - DMX 录制与回放](08_DMX_Recording_Playback.md) | Sequencer & Take Recorder | 将外部 DMX 输入录制为 Sequencer 动画，支持回放输出 |
| [09 - MA 控台导出](09_Export_To_MA.md) | Export to MA / CSV | 将虚拟灯具配置导出为 MA2/MA3 可导入的灯具配置文件 |

---

## 七、快速开始（5 分钟上手）

### 第 1 步：打开 DMX 配置面板

在 UE 编辑器顶部的 SuperStage 工具栏中，点击 **DMX 设置** 按钮打开 DMX 配置面板。

### 第 2 步：设置网络

- **协议**：选择 Art-Net（默认）
- **输入**：勾选「启用」，端口保持 6454
- **本地 IP**：选择与控台同一网段的网卡 IP（如果只有一个网卡，留空即可）
- 点击「应用」

### 第 3 步：放置灯具

从 Content Browser 中将灯具蓝图拖入场景。每台灯具都继承自 `ASuperDmxActorBase`，自带 DMX 控制能力。

### 第 4 步：分配 DMX 地址 (Patch)

选中灯具，在细节面板中设置：
- **Universe**：灯具所在的 DMX 域（与控台一致，范围 1-256）
- **Start Address**：起始地址（与控台中该灯的地址一致，范围 1-512）
- **Fixture ID**：灯具编号（与控台一致，用于导出到 MA）

> **提示**：多台灯具可以使用「Patch 工具」批量分配地址，参见 [07 - Patch 工具](07_Patch_Tools.md)。

### 第 5 步：确认信号

打开 **DMX 活动监视器**，从控台发送信号。如果配置正确，你会看到通道值在实时变化。此时场景中的虚拟灯具也会同步响应。

### 典型工作流程

```
设计阶段                          编程阶段                       演出阶段
───────────                    ───────────                   ───────────
1. 搭建虚拟舞台                2. 连接 MA 控台              5. 实时预览灯光效果
   ├ 放置灯具蓝图                 ├ 配置 Art-Net             6. 调整灯光编程
   ├ 设置灯具位置                 ├ 分配 DMX 地址            7. 录制 DMX → Sequencer
   └ 配置灯具库                   └ 确认信号连通             8. 导出到 MA 控台
3. Patch 批量分配地址
4. 导出灯具配置到 MA
```

---

## 八、系统要求与网络拓扑

### 硬件要求

| 项目 | 最低要求 | 推荐配置 |
|------|---------|---------|
| **CPU** | Intel i5 / AMD Ryzen 5 | Intel i7 / AMD Ryzen 7 以上 |
| **GPU** | NVIDIA GTX 1660 / AMD RX 5600 | NVIDIA RTX 3070 / AMD RX 6800 以上 |
| **内存** | 16 GB | 32 GB 以上 |
| **网络** | 百兆以太网 | 千兆以太网（多 Universe 场景必需） |
| **引擎** | Unreal Engine 5.6 | Unreal Engine 5.7 |

### 推荐网络拓扑

```
┌──────────────────┐                    ┌──────────────────┐
│   MA2/MA3 控台    │                    │  UE5 + SuperStage │
│   IP: 2.x.x.x    │◀──── 千兆交换机 ──▶│  IP: 2.x.x.x     │
│   Art-Net 输出    │         │          │  Art-Net 输入     │
└──────────────────┘         │          └──────────────────┘
                              │
                     ┌────────┴────────┐
                     │ Art-Net Node    │  (可选：接真实灯具)
                     │ IP: 2.x.x.x    │
                     └─────────────────┘
```

### 端口说明

| 端口 | 协议 | 方向 | 说明 |
|------|------|------|------|
| **6454** | Art-Net | 输入/输出 | DMX 信号收发（UDP） |

### Universe 编号对齐

SuperStage 提供「起始 Universe」偏移设置，用于与不同控台的编号习惯对齐：

| 控台 | 控台 Universe 1 | SuperStage 起始 Universe 设为 | SuperStage 内部 Universe |
|------|-----------------|------------------------------|------------------------|
| MA2/MA3 | Universe 1 | 0（默认） | 1 |
| MA2/MA3 | Universe 1 | 1 | 1 |

默认设置（起始 Universe = 0）已兼容 MA2/MA3 的常见配置，大多数情况下无需修改。

---

## 九、性能指南

### 灯具数量与性能

| 场景规模 | 灯具数量 | Universe 数量 | 预期帧率 (RTX 3070) |
|---------|---------|--------------|-------------------|
| 小型演出 | 50 台以下 | 1-4 | 60+ FPS |
| 中型演出 | 50-200 台 | 4-16 | 45-60 FPS |
| 大型演出 | 200-500 台 | 16-64 | 30-45 FPS |
| 超大型 | 500+ 台 | 64+ | 视优化而定 |

### 性能优化建议

| 优化项 | 方法 | 效果 |
|--------|------|------|
| **减少光束渲染** | 降低 `BeamQuality`（默认 75，可降至 50） | GPU 负载 ↓ 20-30% |
| **关闭阴影** | `LightSpotDefaultValue.bLightShadow = false` | GPU 负载 ↓ 15-25% |
| **降低体积雾** | `VolumetricScattering = 0` | GPU 负载 ↓ 10-15% |
| **减少光照距离** | 降低 `MaxLightDistance`（默认 2345cm） | GPU 负载 ↓ |
| **Spot 辅光按需** | 仅在需要真实光照交互时启用 SpotLight | 大幅降低光照计算 |
| **矩阵灯优化** | 减少矩阵规模或降低更新频率 | CPU 负载 ↓ |

---

## 十、常见问题 (FAQ)

### Q1: 虚拟灯具不响应控台信号？

**排查步骤（按顺序逐项检查）**：

1. **检查 DMX 输入开关** — DMX 配置面板 → 输入 → 确认「启用」已勾选
2. **检查网络连通** — 确认 UE 主机与控台在同一网段（ping 测试）
3. **检查端口** — 确认 UDP 6454 端口未被防火墙阻止
4. **检查活动监视器** — 打开 DMX 活动监视器，确认是否收到信号
5. **检查 Universe 对齐** — 确认灯具的 Universe 编号与控台一致（注意起始 Universe 偏移设置）
6. **检查地址匹配** — 确认灯具的 StartAddress 与控台中该灯的地址一致
7. **检查灯具库** — 确认灯具已关联灯具库，且灯具库中的属性名与蓝图中调用的一致
8. **检查蓝图逻辑** — 确认蓝图的 SuperDMXTick 事件中调用了正确的控制函数

### Q2: 多台灯具的地址冲突了怎么办？
使用「Patch 工具」可以自动计算不冲突的地址分配。参见 [07 - Patch 工具](07_Patch_Tools.md)。

### Q3: 如何让 UE 中的灯光效果输出到真实灯具？
在 DMX 配置面板中启用「输出」，设置远程 IP 为灯具/节点的地址。Sequencer 回放时会自动通过 Art-Net 输出 DMX 信号。

### Q4: SuperStage 支持多少个 Universe？
理论上支持 32,767 个 Universe（Art-Net 规范上限）。实际性能取决于硬件和场景复杂度，通常 256 个 Universe 以内无压力。

### Q5: 可以同时接收和发送 DMX 吗？
可以。输入和输出是独立的，可以同时启用。例如：从 MA 控台接收信号控制虚拟灯具，同时将 Sequencer 回放的另一些 Universe 输出到真实灯具。

### Q6: 灯光效果看起来很暗 / 光束不明显？

| 问题 | 解决方案 |
|------|---------|
| 灯光太暗 | 调高 `MaxLightIntensity`（默认 100，可调至 200-500） |
| 光束不可见 | 确认场景中有雾效（ExponentialHeightFog），调高 `BeamFogIntensity`（默认 20） |
| 光束太短 | 调高 `MaxLightDistance`（默认 2345cm，大场景可调至 5000-10000cm） |
| 没有阴影 | 启用 `LightSpotDefaultValue.bLightShadow = true` |
| 颜色不准 | 检查 Post Process Volume 的色调映射设置，建议使用 ACES 色调映射 |

### Q7: 如何创建自定义灯具？

1. 在 Content Browser 中创建新蓝图，父类选择 `ASuperStageLight`（或其他合适的基类）
2. 替换灯具 3D 模型（Hook/Base/Rocker/Hard 四个网格组件）
3. 创建灯具库（Fixture Library）定义通道表
4. 在蓝图的 `SuperDMXTick` 事件中调用控制函数
5. 调整默认参数（光束范围、颜色模式等）

详细步骤参见各模块文档中的「蓝图集成」章节。

---

## 十一、术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| 域 | Universe | 一条 512 通道的 DMX 信号线路 |
| 通道 | Channel | DMX 域中的一个控制值（0-255） |
| 地址 | Address | 灯具在域中的起始通道编号 |
| 补丁 | Patch | 为灯具分配 Universe + 起始地址的过程 |
| 灯具库 | Fixture Library | 定义灯具通道功能映射的数据资产 |
| 灯具 ID | Fixture ID | 灯具的唯一编号，对应控台中的灯具号 |
| 粗调 | Coarse | 8 位（1 通道）精度控制，值范围 0-255 |
| 精调 | Fine | 16 位（2 通道）精度控制，值范围 0-65535 |
| 超精调 | Ultra | 24 位（3 通道）精度控制，值范围 0-16777215 |
| 属性 | Attribute | 灯具的一个可控功能（如 Dimmer、Pan、Tilt、Color 等） |
| 模块实例 | Module Instance | 灯具库中的功能模块（用于矩阵灯的多灯头定义），包含 Patch 偏移和属性列表 |
| 通道集 | Channel Set | 通道中特定值范围对应的功能名称（类似 MA2 通道集） |
| 子属性 | Sub-Attribute | 属性中按值范围细分的功能区间（如 Gobo1 属性中 0-10=白光，11-20=图案1） |
| 灯臂 | Rocker | 电脑灯中负责 Pan（水平旋转）的机械臂（对应 `SceneRocker` 组件） |
| 灯头 | Hard | 电脑灯中负责 Tilt（垂直旋转）的灯头部分（对应 `SceneHard` 组件） |
| 灯钩 | Hook | 电脑灯悬挂在桁架上的挂钩部分（对应 `SM_Hook` 组件） |
| 频闪 | Strobe | 灯光的快速开关闪烁效果，通过 Strobe 通道控制 |
| 雾化 | Frost | 使光束边缘柔化的光学效果 |
| 光圈 | Iris | 控制光斑大小的机械光圈 |
| 色轮 | ColorWheel | 带有多个固定颜色滤片的旋转盘 |
| 图案轮 | Gobo Wheel | 带有多个图案模板的旋转盘 |
| 棱镜 | Prism | 将单束光分裂为多束的光学元件 |
| 切割 | Shaper/Blade | 通过叶片遮挡光束边缘，实现矩形/梯形光斑 |

---

> **下一步**：请阅读 [01 - DMX 网络配置](01_DMX_Network_Configuration.md) 了解如何设置网络连接，或直接跳到 [15 - 全功能电脑灯](15_SuperStageLight.md) 了解最常用的灯具类型。

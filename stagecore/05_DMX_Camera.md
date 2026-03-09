# 05 - DMX 摄像机 (SuperDMXCamera)

> **所属模块**: SuperStage 运行时 (ASuperDMXCamera)  
> **适用对象**: 灯光设计师、虚拟制作技术人员  
> **前置阅读**: [03 - DMX 灯具基础](/docs/stage-core/dmx-actor-base)  
> **最后更新**: 2026-03-06

---

## 一、概述

**SuperDMXCamera** 是一台通过 DMX 信号控制的虚拟电影摄像机。它继承自 SuperDmxActorBase，可以像控制灯具一样从 MA 控台控制摄像机的：

- **6 轴运动** — X/Y/Z 三轴位移 + Pitch/Yaw/Roll 三轴旋转
- **镜头参数** — FOV（视场角）、光圈（f-stop）、对焦距离
- **渲染输出** — 可将摄像机画面渲染到 RenderTarget，用于 LED 屏幕等显示

### 应用场景

- 舞台上的实时摄像机预览（控台操作员远程控制虚拟摄像机角度）
- 虚拟演播室的摄像机调度
- LED 屏幕内容的实时摄像机渲染
- 灯光秀中的视频内容与灯光同步

---

## 二、类继承链

```
AActor (UE5 引擎基类)
  │
  └── ASuperBaseActor ·············· SuperStage 根基类 (L1)
        │                           ├ SceneBase / ForwardArrow / UpArrow
        │                           └ AssetMetaData
        │
        └── ASuperDmxActorBase ····· DMX 灯具基类 (L2)
              │                      ├ DMX 地址（Universe/Address/FixtureID）
              │                      ├ 灯具库 / 通道读取 / 矩阵读取
              │                      └ SuperDMXTick / 地址标签
              │
              └── ASuperDMXCamera ·· DMX 摄像机 (L3) ← 本文档描述的类
                    ├ UCineCameraComponent  ← 电影摄像机（FOV/光圈/对焦）
                    ├ USceneCaptureComponent2D ← 场景捕获（渲染到 RT）
                    ├ 6 轴运动控制（位移 + 旋转）
                    ├ 镜头参数控制（FOV/Aperture/FocusDistance）
                    ├ Boot Refresh 初始化
                    └ CameraParamSpeed 插值平滑
```

> **设计要点**：`ASuperDMXCamera` 直接继承 `ASuperDmxActorBase` 而非 `ASuperLightBase`，因为摄像机不需要 Pan/Tilt 旋转轴结构（SceneRocker/SceneHard）。它使用自己的 6 轴自由运动系统，比 Pan/Tilt 模型更适合摄像机控制场景。

---

## 三、组件结构

```
ASuperDMXCamera
  └── SceneBase (USceneComponent)                    ← 根组件（继承）
        ├── ForwardArrow (UArrowComponent)            ← +X 方向预览（继承）
        ├── UpArrow (UArrowComponent)                 ← +Z 方向预览（继承）
        ├── AddressLabel (UTextRenderComponent)       ← DMX 地址标签（继承）
        │
        ├── CineCamera (UCineCameraComponent)         ← 电影摄像机组件
        │     ├ FilmbackSettings   ← 底片尺寸（Super 35mm 等）
        │     ├ LensSettings       ← 镜头参数（焦距/光圈/对焦）
        │     └ PostProcessSettings ← 后处理设置
        │
        └── SceneCapture (USceneCaptureComponent2D)   ← 场景捕获组件
              ├ TextureTarget      ← 渲染目标纹理
              ├ CaptureSource      ← 捕获源类型
              └ bCaptureEveryFrame ← 是否每帧捕获
```

| 组件 | 类型 | 说明 |
|------|------|------|
| **CineCamera** | UCineCameraComponent | 电影摄像机组件，提供专业摄像机功能（FOV/光圈/对焦） |
| **SceneCapture** | USceneCaptureComponent2D | 场景捕获组件，将画面渲染到 RenderTarget 供 LED 屏幕使用 |

---

## 四、默认参数配置

在细节面板的 **B.DefaultParameter** 分组中，可以设置摄像机的运动范围和镜头参数范围。

### 4.1 启动开关

| 参数 | 说明 | 默认值 |
|------|------|--------|
| **Start** | 摄像机启动开关。设为 True 后，摄像机开始响应 DMX 信号 | False |

> **重要**：放置到场景后，必须将 Start 设为 **True** 才能开始接受 DMX 控制。

### 4.2 初始化函数

| 操作 | 说明 |
|------|------|
| **Boot Refresh** | 点击此按钮（在编辑器中）可根据当前位置和 MovingRange/RotRange 自动计算 InitialPosition/EndPosition 和 InitialRotation/EndRotation |

**使用步骤**：
1. 将摄像机放置到场景中你希望的**中心位置**
2. 设置 MovingRange（位移范围）和 RotRange（旋转范围）
3. 点击 **Boot Refresh** 按钮
4. 系统自动计算 InitialPosition 和 EndPosition（基于当前位置 ± MovingRange/2）
5. 系统自动计算 InitialRotation 和 EndRotation（基于当前旋转 ± RotRange/2）

### 4.3 位移范围

| 参数 | 说明 | 默认值 | 单位 |
|------|------|--------|------|
| **Moving Range** | X/Y/Z 三轴的位移范围（总范围） | (0, 0, 0) | 厘米 |
| **Initial Position** | 位移范围的起点（系统自动计算，只读） | (0, 0, 0) | 厘米 |
| **End Position** | 位移范围的终点（系统自动计算，只读） | (0, 0, 0) | 厘米 |

**位移映射**：
```
实际位置 = Lerp(InitialPosition, EndPosition, DMX值)
```

其中 DMX 值为归一化的 0.0 ~ 1.0：
- 0.0 → 位于 InitialPosition（范围起点）
- 0.5 → 位于中心位置（初始放置位置）
- 1.0 → 位于 EndPosition（范围终点）

**示例**：
- 摄像机放在 (500, 0, 300)
- Moving Range 设为 (200, 100, 50)
- Boot Refresh 后：InitialPosition = (400, -50, 275)，EndPosition = (600, 50, 325)
- DMX PosX = 0.0 → X = 400，DMX PosX = 1.0 → X = 600

### 4.4 旋转范围

| 参数 | 说明 | 默认值 | 单位 |
|------|------|--------|------|
| **Rot Range** | Pitch/Yaw/Roll 三轴的旋转范围（总范围） | (180, 360, 90) | 度 |
| **Initial Rotation** | 旋转范围的起点（系统自动计算，只读） | (0, 0, 0) | 度 |
| **End Rotation** | 旋转范围的终点（系统自动计算，只读） | (0, 0, 0) | 度 |

**旋转映射**：
```
实际旋转 = Lerp(InitialRotation, EndRotation, DMX值)
```

**默认 Rot Range = (180, 360, 90) 的含义**：
- **Pitch**（俯仰）：±90°，总范围 180°
- **Yaw**（偏航）：±180°，总范围 360°（可全方位旋转）
- **Roll**（翻滚）：±45°，总范围 90°

### 4.5 镜头参数范围

| 参数 | 说明 | 默认值 | 单位 |
|------|------|--------|------|
| **FOV Range** | 视场角范围 (最小, 最大) | (15.0, 120.0) | 度 |
| **Aperture Range** | 光圈范围 (最小, 最大) | (1.2, 22.0) | f-stop |
| **Focus Distance Range** | 对焦距离范围 (最小, 最大) | (50.0, 100000.0) | 厘米 |

**映射方式**（与位移/旋转相同）：
```
实际FOV = Lerp(FOVRange.Min, FOVRange.Max, DMX值)
实际光圈 = Lerp(ApertureRange.Min, ApertureRange.Max, DMX值)
实际对焦距离 = Lerp(FocusDistanceRange.Min, FocusDistanceRange.Max, DMX值)
```

### 4.6 插值速度

| 参数 | 说明 | 默认值 | 范围 |
|------|------|--------|------|
| **Camera Param Speed** | 摄像机参数变化的插值速度 | 3.0 | 0.0 - 10.0 |

- 值越大 → 响应越快（更跟手）
- 值越小 → 响应越慢（更平滑，但有延迟）
- 值 = 0 → 不移动（锁定）
- 值 = 10 → 几乎无延迟

> **提示**：建议设为 3.0 ~ 5.0，在响应速度和平滑度之间取得平衡。

---

## 五、渲染输出配置

SuperDMXCamera 可以将摄像机画面渲染到 RenderTarget，用于 LED 屏幕、监视器等显示。

| 参数 | 说明 | 默认值 |
|------|------|--------|
| **Render Target** | 渲染目标纹理（UTextureRenderTarget2D） | 无 |
| **Render Resolution** | 渲染分辨率 | 1920 × 1080 |
| **Enable Scene Capture** | 是否启用场景捕获渲染 | False |

### 5.1 设置步骤

1. 在 Content Browser 中创建一个 **Render Target** 资产
2. 将其分配给摄像机的 **Render Target** 属性
3. 设置所需的 **Render Resolution**
4. 将 **Enable Scene Capture** 设为 **True**
5. 将 Render Target 应用到场景中的材质（例如 LED 屏幕面板的材质）

> **性能提示**：场景捕获会消耗额外的 GPU 资源。如果不需要渲染输出，请保持 Enable Scene Capture 为 False。

---

## 六、控制参数（实时值）

在细节面板的 **C.ControlParameter** 分组中，可以看到当前的控制参数值。这些值通常由 DMX 信号驱动，也可以在编辑器中手动调整进行测试：

### 6.1 位移控制

| 参数 | 说明 | 范围 | 默认值 |
|------|------|------|--------|
| **PosX** | X 轴位置（归一化） | 0.0 - 1.0 | 0.5（中心） |
| **PosY** | Y 轴位置（归一化） | 0.0 - 1.0 | 0.5（中心） |
| **PosZ** | Z 轴位置（归一化） | 0.0 - 1.0 | 0.5（中心） |

### 6.2 旋转控制

| 参数 | 说明 | 范围 | 默认值 |
|------|------|------|--------|
| **RotX** | Pitch 旋转（归一化） | 0.0 - 1.0 | 0.5（正前方） |
| **RotY** | Yaw 旋转（归一化） | 0.0 - 1.0 | 0.5（正前方） |
| **RotZ** | Roll 旋转（归一化） | 0.0 - 1.0 | 0.5（水平） |

### 6.3 镜头控制

| 参数 | 说明 | 范围 | 默认值 |
|------|------|------|--------|
| **FOV** | 视场角（归一化） | 0.0 - 1.0 | 0.5 |
| **Aperture** | 光圈（归一化） | 0.0 - 1.0 | 0.5 |
| **Focus Distance** | 对焦距离（归一化） | 0.0 - 1.0 | 0.5 |

> **所有控制参数都是 0.0 ~ 1.0 的归一化值**，系统根据前面设置的范围自动映射到实际物理值。

---

## 七、缓存信息（只读）

在细节面板的 **A.Cache Info** 分组中，可以查看当前的实际物理值（经过插值平滑后的值）：

| 参数 | 说明 | 单位 |
|------|------|------|
| **CurrentPosX/Y/Z** | 当前实际位置 | 厘米 |
| **CurrentRotX/Y/Z** | 当前实际旋转 | 度 |
| **CurrentFOV** | 当前实际视场角 | 度 |
| **CurrentAperture** | 当前实际光圈 | f-stop |
| **CurrentFocusDistance** | 当前实际对焦距离 | 厘米 |

这些值是只读的，主要用于调试和监控。

---

## 八、蓝图控制函数

SuperDMXCamera 提供两个核心蓝图函数用于 DMX 控制：

### 8.1 DMXCameraControl（运动控制）

控制摄像机的 6 轴运动：

| 输入参数 | 说明 |
|---------|------|
| **DmxXPos** | X 轴位移的 DMX 属性 |
| **DmxYPos** | Y 轴位移的 DMX 属性 |
| **DmxZPos** | Z 轴位移的 DMX 属性 |
| **DmxXRot** | Pitch 旋转的 DMX 属性 |
| **DmxYRot** | Yaw 旋转的 DMX 属性 |
| **DmxZRot** | Roll 旋转的 DMX 属性 |

### 8.2 DMXCameraParams（镜头参数控制）

控制摄像机的镜头参数：

| 输入参数 | 说明 |
|---------|------|
| **DmxFOV** | 视场角的 DMX 属性 |
| **DmxAperture** | 光圈的 DMX 属性 |
| **DmxFocusDistance** | 对焦距离的 DMX 属性 |

### 8.3 典型蓝图实现

```
Event SuperDMXTick(DeltaTime)
  │
  ├─ DMXCameraControl(PosX, PosY, PosZ, RotX, RotY, RotZ)
  └─ DMXCameraParams(FOV, Aperture, FocusDistance)
```

---

## 九、DMX 通道规划建议

一台 SuperDMXCamera 需要 9 个基本 DMX 属性。如果使用 16 位精度（推荐），需要 18 个通道：

| 通道 | 属性 | 精度 | 说明 |
|------|------|------|------|
| 1-2 | PosX | 16位 | X 轴位移 |
| 3-4 | PosY | 16位 | Y 轴位移 |
| 5-6 | PosZ | 16位 | Z 轴位移 |
| 7-8 | RotX | 16位 | Pitch 旋转 |
| 9-10 | RotY | 16位 | Yaw 旋转 |
| 11-12 | RotZ | 16位 | Roll 旋转 |
| 13-14 | FOV | 16位 | 视场角 |
| 15-16 | Aperture | 16位 | 光圈 |
| 17-18 | FocusDistance | 16位 | 对焦距离 |

> **提示**：如果不需要精细控制，可以使用 8 位精度（9 个通道），但运动可能不够平滑。

---

## 十、使用步骤总结

1. **放置摄像机** — 将 SuperDMXCamera 拖入场景，放到你想要的中心位置
2. **设置范围** — 配置 MovingRange、RotRange、FOVRange、ApertureRange、FocusDistanceRange
3. **初始化** — 点击 **Boot Refresh** 自动计算起止范围
4. **配置灯具库** — 创建或选择一个包含 9 个属性的灯具库
5. **设置 Patch** — 分配 Universe 和 Start Address
6. **启动** — 将 Start 设为 True
7. **（可选）配置渲染输出** — 设置 RenderTarget 和 Enable Scene Capture
8. **从控台发送信号** — 推动对应通道的推杆，摄像机应开始响应

---

## 十一、常见问题

### Q: 摄像机不响应 DMX 信号？
1. 检查 **Start** 是否设为 True
2. 检查是否点击了 **Boot Refresh**
3. 确认 DMX 网络配置正确（参见 [01 - 网络配置](/docs/stage-core/dmx-network)）
4. 确认灯具库和 Patch 配置正确

### Q: 摄像机运动抖动？
- 增大 **Camera Param Speed** 值（使运动更平滑）
- 或使用 16 位精度的 DMX 属性（减少量化误差）

### Q: 摄像机运动范围不对？
- 重新调整 MovingRange / RotRange
- 重新点击 **Boot Refresh** 更新计算

### Q: 渲染输出画面是黑色的？
- 确认 **Enable Scene Capture** 为 True
- 确认 **Render Target** 已正确分配
- 确认 Render Target 的分辨率设置正确

---

> **下一步**：请阅读 [06 - DMX 活动监视器](/docs/stage-core/dmx-monitor) 了解如何实时监控 DMX 通道数据。

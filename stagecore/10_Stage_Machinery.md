# 10 - 舞台机械 (Stage Machinery)

> **所属模块**: SuperStage 运行时 (ASuperLiftingMachinery / ASuperRailMachinery)  
> **适用对象**: 舞美设计师、机械控制技术人员  
> **前置阅读**: [03 - DMX 灯具基础](/docs/stage-core/dmx-actor-base)  
> **最后更新**: 2026-03-06

---

## 一、概述

SuperStage 提供两种 DMX 控制的舞台机械 Actor，用于在虚拟场景中模拟舞台升降台、旋转台、轨道车等机械设备的运动：

| 类型 | 类名 | 控制轴数 | 说明 |
|------|------|---------|------|
| **升降机械** | SuperLiftingMachinery | **6 轴** | 3 轴位移 + 3 轴旋转，适用于升降台、旋转台、吊杆 |
| **轨道机械** | SuperRailMachinery | **7 轴** | 沿样条曲线轨道运动 + 6 轴局部偏移/旋转，适用于轨道车、吊挂小车 |

两者都继承自 SuperDmxActorBase，具备完整的 DMX 接收能力。

### 类继承链

```
AActor (UE5 引擎基类)
  │
  └── ASuperBaseActor ·············· SuperStage 根基类 (L1)
        │                           ├ SceneBase / ForwardArrow / UpArrow
        │                           └ AssetMetaData
        │
        └── ASuperDmxActorBase ····· DMX 灯具基类 (L2)
              │                      ├ DMX 地址 / 灯具库 / 通道读取
              │                      └ SuperDMXTick / 地址标签
              │
              ├── ASuperLiftingMachinery ·· 升降机械 (L3)
              │     ├ 6 轴自由度（3 位移 + 3 旋转）
              │     ├ Boot Refresh 初始化
              │     ├ 绝对旋转 / 无极旋转模式
              │     └ PosSpeed / RotSpeed 插值平滑
              │
              └── ASuperRailMachinery ···· 轨道机械 (L3)
                    ├ 7 轴自由度（轨道位置 + 3 偏移 + 3 旋转）
                    ├ USplineComponent 样条曲线轨道
                    ├ RailMountPoint 挂载点 → OffsetComponent
                    ├ 闭合回路 + 最短弧插值
                    └ 朝向锁定到轨道切线
```

> **设计要点**：两种机械都直接继承 `ASuperDmxActorBase` 而非 `ASuperLightBase`，因为它们不需要 Pan/Tilt 旋转轴结构。它们使用自己的多轴自由运动系统，更适合舞台机械的控制需求。

### 应用场景

- **升降台** — 舞台地面上下升降（Z 轴位移）
- **旋转舞台** — 整个舞台区域水平旋转（Yaw 无极旋转）
- **吊杆 / 灯架** — 上下升降 + 前后移动
- **机械臂** — 多轴联动运动
- **轨道车** — 沿预设路径移动的灯光/摄像机小车
- **吊挂小车** — 沿 Truss 轨道移动的灯具挂载点
- **环形轨道** — 闭合回路上的循环运动

---

# 第一部分：升降机械 (SuperLiftingMachinery)

## 二、基本原理

升降机械提供 **6 轴自由度**：

```
3 轴位移                    3 轴旋转
┌─────────────┐            ┌─────────────┐
│ PosX (左右)  │            │ RotX (Pitch) │
│ PosY (前后)  │            │ RotY (Yaw)   │
│ PosZ (上下)  │            │ RotZ (Roll)  │
└─────────────┘            └─────────────┘
```

每个轴通过一个 DMX 属性（0.0 ~ 1.0 归一化值）控制，映射到预设的运动范围。

---

## 三、属性详解

### 3.1 启动开关

| 参数 | 位置 | 说明 | 默认值 |
|------|------|------|--------|
| **Start** | B.DefaultParameter | 机械启动开关。设为 True 后开始响应 DMX 信号 | **False** |

> **重要**：放置到场景后，必须将 Start 设为 **True** 才能开始运动。这是一个安全保护设计——防止误操作导致机械突然运动。

### 3.2 初始化按钮

| 操作 | 说明 |
|------|------|
| **Boot Refresh** | 点击此按钮，以当前 Actor 位置和旋转为基准，自动计算运动范围的起止点 |

**BootRefresh 执行的操作**：
1. 记录当前世界位置为 **InitialPosition**
2. 计算 **EndPosition** = InitialPosition + MovingRange
3. 计算 **InitialRotation** = 当前旋转 - RotRange × 0.5
4. 计算 **EndRotation** = 当前旋转 + RotRange × 0.5
5. 同步所有缓存值

> **使用步骤**：先将机械放到正确的起始位置 → 设置 MovingRange/RotRange → 点击 BootRefresh。

### 3.3 位移参数

| 参数 | 可编辑 | 说明 | 默认值 | 单位 |
|------|--------|------|--------|------|
| **MovingRange** | ✅ | X/Y/Z 三轴的位移量（从 InitialPosition 到 EndPosition 的偏移） | (0, 0, 0) | 厘米 |
| **InitialPosition** | ❌ 只读 | 位移范围的起始点（BootRefresh 计算） | — | 厘米 |
| **EndPosition** | ❌ 只读 | 位移范围的终点（BootRefresh 计算） | — | 厘米 |

**位移映射**：
```
目标位置 = Lerp(InitialPosition, EndPosition, DMX值)
```
- DMX 值 = 0.0 → Actor 在 InitialPosition（起始位置）
- DMX 值 = 0.5 → Actor 在 InitialPosition 与 EndPosition 的中点
- DMX 值 = 1.0 → Actor 在 EndPosition（终点位置）

**示例**：升降台只做上下运动
- 将升降台放在最低位置
- 设置 MovingRange = (0, 0, 300)（Z 轴上升 300 厘米 = 3 米）
- 点击 BootRefresh
- DMX PosZ = 0.0 → 最低位置；DMX PosZ = 1.0 → 升起 3 米

### 3.4 旋转参数（绝对模式）

以下参数仅在 **PolarRotation = False**（绝对旋转模式）时显示：

| 参数 | 可编辑 | 说明 | 默认值 | 单位 |
|------|--------|------|--------|------|
| **RotRange** | ✅ | Pitch/Yaw/Roll 三轴的旋转总范围 | (360, 360, 360) | 度 |
| **InitialRotation** | ❌ 只读 | 旋转范围的起始角度（BootRefresh 计算） | — | 度 |
| **EndRotation** | ❌ 只读 | 旋转范围的终止角度（BootRefresh 计算） | — | 度 |

**旋转映射**：
```
目标旋转 = Lerp(InitialRotation, EndRotation, DMX值)
```

**示例**：旋转舞台水平旋转 360°
- RotRange = (0, 360, 0)（只有 Yaw 旋转）
- BootRefresh 后：InitialRotation.Yaw = 当前Yaw - 180°, EndRotation.Yaw = 当前Yaw + 180°
- DMX RotY = 0.0 → 最左；DMX RotY = 0.5 → 正中；DMX RotY = 1.0 → 最右

### 3.5 速度参数

| 参数 | 条件 | 说明 | 范围 | 默认值 |
|------|------|------|------|--------|
| **PosSpeed** | 始终可见 | 位移插值速度 | 0.0 - 10.0 | 1.0 |
| **RotSpeed** | PolarRotation = False | 旋转插值速度（绝对模式） | 0.0 - 10.0 | 1.0 |
| **PolarRotationSpeed** | PolarRotation = True | 无极旋转最大转速 | 0.0 - 10.0 | 1.0 |

**速度含义**：
- **PosSpeed / RotSpeed**：FInterpTo 插值速度参数。值越大，运动越快越跟手；值越小，运动越平滑但有延迟
  - 0 → 完全不动（锁定）
  - 1 → 较慢平滑
  - 5 → 中等响应
  - 10 → 几乎无延迟
- **PolarRotationSpeed**：无极旋转的最大角速度。DMX = 0.0 时以 -PolarRotationSpeed 速度逆时针旋转，DMX = 1.0 时以 +PolarRotationSpeed 速度顺时针旋转

**调参建议**：

| 场景 | 推荐 PosSpeed | 推荐 RotSpeed | 说明 |
|------|--------------|--------------|------|
| 大型升降台（缓慢庄重） | 0.5 - 1.5 | 0.5 - 1.5 | 模拟液压升降台的沉稳运动 |
| 标准吊杆升降 | 1.0 - 3.0 | — | 灯架上下运动，中等速度 |
| 快速弹出效果 | 5.0 - 10.0 | — | 突然弹出的视觉效果（烟火/人员） |
| 旋转舞台（缓慢） | — | 0.3 - 1.0 | 慢速旋转展示 |
| 旋转舞台（快速切换） | — | 3.0 - 8.0 | 场景快速转换 |
| 机械臂多轴联动 | 2.0 - 5.0 | 2.0 - 5.0 | 需要多轴协调的运动 |

> **安全提示**：真实舞台机械的速度通常受安全法规限制。建议在虚拟预览中使用与真实设备相匹配的速度值，以确保预演效果与实际演出一致。

### 3.6 无极旋转模式

| 参数 | 说明 | 默认值 |
|------|------|--------|
| **PolarRotation** | 无极旋转模式开关 | **False** |

- **False（绝对模式）**：DMX 值映射到固定角度范围（InitialRotation ~ EndRotation）
- **True（无极模式）**：DMX 值控制旋转速度，机械可以**连续不停旋转**

**无极旋转 DMX 映射**：

| DMX 值（归一化） | 旋转行为 |
|-----------------|---------|
| 0.0 | 以最大速度**逆时针**旋转 |
| 0.5 | **停止**旋转 |
| 1.0 | 以最大速度**顺时针**旋转 |
| 0.0 ~ 0.5 | 逆时针旋转，速度递减 |
| 0.5 ~ 1.0 | 顺时针旋转，速度递增 |

> **提示**：无极模式适用于旋转舞台、旋转灯架等需要持续旋转的场景。

### 3.7 控制参数（实时值）

| 参数 | 说明 | 范围 | 默认值 |
|------|------|------|--------|
| **PosX** | X 轴位移控制 | 0.0 - 1.0 | 0.5 |
| **PosY** | Y 轴位移控制 | 0.0 - 1.0 | 0.5 |
| **PosZ** | Z 轴位移控制 | 0.0 - 1.0 | 0.5 |
| **RotX** | Pitch 旋转控制 | 0.0 - 1.0 | 0.5 |
| **RotY** | Yaw 旋转控制 | 0.0 - 1.0 | 0.5 |
| **RotZ** | Roll 旋转控制 | 0.0 - 1.0 | 0.5 |

这些值通常由 DMX 信号驱动，也可以在编辑器中手动拖拽进行测试。

### 3.8 缓存信息（只读）

| 参数 | 说明 | 单位 |
|------|------|------|
| **CurrentPosX/Y/Z** | 当前实际世界位置（经过插值平滑后） | 厘米 |
| **CurrentRotX/Y/Z** | 当前实际旋转角度（经过插值平滑后） | 度 |

---

## 四、蓝图控制

### 4.1 LiftingMachinery 函数

这是升降机械的主控制函数，在蓝图的 **SuperDMXTick** 事件中调用：

| 输入参数 | 说明 |
|---------|------|
| **DmxXPos** | X 轴位移的 DMX 属性 |
| **DmxYPos** | Y 轴位移的 DMX 属性 |
| **DmxZPos** | Z 轴位移的 DMX 属性 |
| **DmxXRot** | Pitch 旋转的 DMX 属性 |
| **DmxYRot** | Yaw 旋转的 DMX 属性 |
| **DmxZRot** | Roll 旋转的 DMX 属性 |

### 4.2 典型蓝图

```
Event SuperDMXTick(DeltaTime)
  │
  └─ LiftingMachinery(PosX, PosY, PosZ, RotX, RotY, RotZ)
```

---

## 五、使用步骤总结（升降机械）

1. **放置机械** — 将 SuperLiftingMachinery 蓝图拖入场景，放到起始位置
2. **设置位移范围** — MovingRange，例如 (0, 0, 300) 表示 Z 轴升降 3 米
3. **设置旋转范围** — RotRange（或启用 PolarRotation 无极旋转）
4. **设置速度** — PosSpeed / RotSpeed / PolarRotationSpeed
5. **点击 BootRefresh** — 自动计算起止范围
6. **配置 DMX** — 设置 Universe、Start Address、Fixture Library
7. **启用** — Start = True
8. **从控台控制** — 推动对应通道推杆

---

# 第二部分：轨道机械 (SuperRailMachinery)

## 六、基本原理

轨道机械在升降机械的基础上新增了一个**轨道运动轴**，共 **7 轴自由度**：

```
轨道层（第1层）                偏移层（第2层）
┌─────────────────┐          ┌─────────────────┐
│ RailPos          │          │ PosX / PosY / PosZ │
│ 沿样条曲线移动    │    +     │ RotX / RotY / RotZ │
│                  │          │ 在挂载点局部坐标中  │
└────────┬────────┘          └────────┬────────┘
         │                            │
    沿轨道路径移动              在轨道位置基础上
    （全局运动）               做局部偏移/旋转
```

**组件层级**：
```
Actor Root
  └── RailSpline (样条曲线 — 定义轨道路径)
        └── RailMountPoint (挂载点 — 沿样条移动)
              └── OffsetComponent (偏移组件 — 6轴局部变换)
                    └── 子Actor/灯具/摄像机 (挂载到这里)
```

### 与升降机械的区别

| 特性 | 升降机械 | 轨道机械 |
|------|---------|---------|
| 控制轴数 | 6 轴 | **7 轴** |
| 运动路径 | 直线（InitialPosition → EndPosition） | **样条曲线**（可任意弯曲） |
| 位移参考系 | 世界坐标 | 挂载点**局部坐标** |
| 轨道编辑 | 无 | 编辑器中**可视化拖拽**样条控制点 |
| 闭合回路 | 不支持 | **支持**（首尾相连循环） |
| 朝向锁定 | 不支持 | **支持**（锁定到轨道切线方向） |

---

## 七、轨道组件

### 7.1 RailSpline（样条曲线）

| 参数 | 说明 |
|------|------|
| **类型** | USplineComponent |
| **位置** | A.RailComponents 分组 |
| **编辑方式** | 在场景视口中选中 Actor → 选择样条组件 → 拖拽控制点 |

样条曲线定义了轨道的**路径形状**。你可以在编辑器中：
- **添加控制点** — 右键点击样条线 → Add Spline Point
- **移动控制点** — 直接拖拽控制点调整路径
- **调整切线** — 拖拽切线手柄控制曲率
- **删除控制点** — 选中控制点 → Delete

### 7.2 RailMountPoint（挂载点）

| 参数 | 说明 |
|------|------|
| **类型** | USceneComponent |
| **位置** | A.RailComponents 分组 |
| **行为** | 自动沿样条曲线移动，位置由 RailPos DMX 值决定 |

### 7.3 OffsetComponent（偏移组件）

| 参数 | 说明 |
|------|------|
| **类型** | USceneComponent |
| **位置** | A.RailComponents 分组 |
| **行为** | 在挂载点的局部坐标系中做 6 轴偏移/旋转 |

> **重要**：任何需要跟随轨道运动的子 Actor（灯具、摄像机等）应**挂载到 OffsetComponent** 上，而不是 Actor 根组件。

---

## 八、属性详解

### 8.1 启动与初始化

| 参数 | 说明 | 默认值 |
|------|------|--------|
| **Start** | 启动开关 | False |
| **Boot Refresh** | 初始化范围计算（按钮） | — |

BootRefresh 对轨道机械的操作：
1. 同步样条闭合状态
2. 记录 OffsetComponent 当前局部位置为基准
3. 计算 InitialOffset / EndOffset（基于 OffsetRange）
4. 计算 InitialRotation / EndRotation（基于 RotRange）
5. 同步轨道位置缓存

### 8.2 轨道参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| **LockOrientationToRail** | 锁定朝向到轨道切线方向 | False |
| **ClosedLoop** | 轨道是否闭合（首尾相连） | False |

**LockOrientationToRail**：
- **False** — 挂载点沿轨道移动但**不旋转**，朝向由 RotX/Y/Z 独立控制
- **True** — 挂载点的**朝向始终沿轨道切线方向**，旋转控制在此基础上叠加

**ClosedLoop**：
- **False** — 开放轨道。RailPos = 0.0 为起点，1.0 为终点
- **True** — 闭合轨道。起点和终点连接形成回路，支持循环运动

> **闭合轨道特殊行为**：当轨道闭合时，系统使用**最短弧路径插值**。例如从 0.9 到 0.1，系统不会倒转经过 0.5，而是正向经过 0.0/1.0 跨越点（只走 0.2 的距离而非 0.8）。

### 8.3 偏移参数

| 参数 | 可编辑 | 说明 | 默认值 | 单位 |
|------|--------|------|--------|------|
| **OffsetRange** | ✅ | 偏移组件在挂载点局部坐标中的偏移范围 | (0, 0, 0) | 厘米 |
| **InitialOffset** | ❌ 只读 | 偏移范围的起始值 | — | 厘米 |
| **EndOffset** | ❌ 只读 | 偏移范围的终止值 | — | 厘米 |

偏移是相对于**挂载点的局部坐标系**，不是世界坐标。这意味着：
- 当挂载点跟随轨道转弯时，偏移方向也会跟着旋转
- PosX 始终是挂载点的"前方"，PosY 是"右方"，PosZ 是"上方"

### 8.4 旋转参数

与升降机械完全相同（支持绝对模式和无极旋转模式），但旋转作用于 OffsetComponent 的**局部旋转**。

| 参数 | 条件 | 说明 | 默认值 |
|------|------|------|--------|
| **RotRange** | PolarRotation = False | 旋转总范围 | (360, 360, 360) |
| **InitialRotation** | PolarRotation = False | 起始旋转 | (0, 0, 0) |
| **EndRotation** | PolarRotation = False | 终止旋转 | (0, 0, 0) |
| **PolarRotation** | 始终 | 无极旋转开关 | False |

### 8.5 速度参数

| 参数 | 说明 | 范围 | 默认值 |
|------|------|------|--------|
| **RailSpeed** | 轨道移动插值速度 | 0.0 - 10.0 | 1.0 |
| **OffsetSpeed** | 偏移移动插值速度 | 0.0 - 10.0 | 1.0 |
| **RotSpeed** | 旋转插值速度（绝对模式） | 0.0 - 10.0 | 1.0 |
| **PolarRotationSpeed** | 无极旋转速度 | 0.0 - 10.0 | 1.0 |

### 8.6 控制参数

| 参数 | 说明 | 范围 | 默认值 |
|------|------|------|--------|
| **RailPos** | 轨道位置（0=起点, 1=终点） | 0.0 - 1.0 | 0.0 |
| **PosX** | X 轴局部偏移 | 0.0 - 1.0 | 0.5 |
| **PosY** | Y 轴局部偏移 | 0.0 - 1.0 | 0.5 |
| **PosZ** | Z 轴局部偏移 | 0.0 - 1.0 | 0.5 |
| **RotX** | Pitch 旋转 | 0.0 - 1.0 | 0.5 |
| **RotY** | Yaw 旋转 | 0.0 - 1.0 | 0.5 |
| **RotZ** | Roll 旋转 | 0.0 - 1.0 | 0.5 |

### 8.7 缓存信息（只读）

| 参数 | 说明 |
|------|------|
| **CurrentRailPosition** | 当前轨道位置（归一化 0~1） |
| **CurrentOffsetX/Y/Z** | 当前偏移量 |
| **CurrentRotX/Y/Z** | 当前旋转角度 |

---

## 九、蓝图控制

### 9.1 RailMachinery 函数（7 轴完整控制）

| 输入参数 | 说明 |
|---------|------|
| **DmxRailPos** | 轨道位置的 DMX 属性 |
| **DmxXPos** | X 轴偏移的 DMX 属性 |
| **DmxYPos** | Y 轴偏移的 DMX 属性 |
| **DmxZPos** | Z 轴偏移的 DMX 属性 |
| **DmxXRot** | Pitch 旋转的 DMX 属性 |
| **DmxYRot** | Yaw 旋转的 DMX 属性 |
| **DmxZRot** | Roll 旋转的 DMX 属性 |

### 9.2 RailPositionOnly 函数（仅轨道位置）

如果只需要沿轨道移动而不需要偏移/旋转：

| 输入参数 | 说明 |
|---------|------|
| **DmxRailPos** | 轨道位置的 DMX 属性 |

### 9.3 典型蓝图

```
Event SuperDMXTick(DeltaTime)
  │
  └─ RailMachinery(RailPos, PosX, PosY, PosZ, RotX, RotY, RotZ)
```

或简化版：

```
Event SuperDMXTick(DeltaTime)
  │
  └─ RailPositionOnly(RailPos)
```

---

## 十、使用步骤总结（轨道机械）

1. **放置轨道机械** — 将 SuperRailMachinery 蓝图拖入场景
2. **编辑轨道路径** — 选中样条组件，拖拽控制点设计路径形状
3. **（可选）设为闭合** — ClosedLoop = True（环形轨道）
4. **（可选）锁定朝向** — LockOrientationToRail = True（沿轨道方向）
5. **设置偏移范围** — OffsetRange（如果需要局部偏移）
6. **设置旋转范围** — RotRange（或启用 PolarRotation）
7. **设置速度** — RailSpeed / OffsetSpeed / RotSpeed
8. **点击 BootRefresh**
9. **配置 DMX** — Universe、Start Address、Fixture Library
10. **挂载子 Actor** — 将灯具/摄像机挂载到 OffsetComponent
11. **启用** — Start = True

---

## 十一、DMX 通道规划建议

### 升降机械（6 轴）

| 通道 | 属性 | 精度 | 说明 |
|------|------|------|------|
| 1-2 | PosX | 16位 | X 轴位移 |
| 3-4 | PosY | 16位 | Y 轴位移 |
| 5-6 | PosZ | 16位 | Z 轴位移 |
| 7-8 | RotX | 16位 | Pitch 旋转 |
| 9-10 | RotY | 16位 | Yaw 旋转 |
| 11-12 | RotZ | 16位 | Roll 旋转 |

如果只做升降（Z 轴），只需 1-2 个通道即可。

### 轨道机械（7 轴）

| 通道 | 属性 | 精度 | 说明 |
|------|------|------|------|
| 1-2 | RailPos | 16位 | 轨道位置 |
| 3-4 | PosX | 16位 | X 偏移 |
| 5-6 | PosY | 16位 | Y 偏移 |
| 7-8 | PosZ | 16位 | Z 偏移 |
| 9-10 | RotX | 16位 | Pitch |
| 11-12 | RotY | 16位 | Yaw |
| 13-14 | RotZ | 16位 | Roll |

> **提示**：如果不需要所有轴，可以在灯具库中只定义需要的属性，节省通道。

---

## 十二、编辑器预览

两种机械都支持**编辑器实时预览**——无需点击 Play 按钮：

- **升降机械**：运行时通过 BeginPlay 自动调用 BootRefresh
- **轨道机械**：编辑器中修改 RailPos 或拖拽样条控制点会**实时更新**挂载点位置
  - 修改任何属性 → 自动更新轨道位置
  - 拖拽 Actor → 自动同步
  - Undo/Redo → 自动同步

---

## 十三、常见问题

### Q: 机械不动？
1. 检查 **Start** 是否为 True
2. 检查是否已点击 **BootRefresh**
3. 确认 DMX 网络配置正确
4. 确认灯具库中定义了对应的属性

### Q: 运动方向反了？
调整 MovingRange / OffsetRange 中对应轴的正负号。例如将 (0, 0, 300) 改为 (0, 0, -300) 可以反转 Z 轴方向。

### Q: 运动抖动？
降低 PosSpeed / RotSpeed / RailSpeed 值（使插值更平滑），或使用 16 位精度的 DMX 属性。

### Q: 轨道机械的子 Actor 不跟随？
确认子 Actor 挂载到了 **OffsetComponent** 而不是 Actor 根组件。OffsetComponent 是通过 GetDefaultAttachComponent 返回的默认挂载点。

### Q: 闭合轨道上运动方向不对？
系统自动选择最短弧路径。如果从 0.9 到 0.1，会正向经过 1.0/0.0 跨越点（短路径），而不是反向经过 0.5（长路径）。这是设计行为。

### Q: 编辑器中拖拽样条后轨道不更新？
确认样条是 RailSplineComponent（不是其他组件）。选中 Actor 后在组件列表中选择 RailSpline 组件再编辑。

---

> **下一步**：请阅读 [11 - 激光系统](/docs/stage-core/laser-system) 了解 Beyond 激光可视化功能。

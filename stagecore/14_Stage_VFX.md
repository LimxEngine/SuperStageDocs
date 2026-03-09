# 14 - 舞台特效 VFX (Stage VFX)

> **所属模块**: SuperStage 运行时 (ASuperStageVFXActor)  
> **适用对象**: 灯光设计师、舞美特效师  
> **前置阅读**: [04 - 电脑灯](/docs/stage-core/computer-light)  
> **最后更新**: 2026-03-06

---

## 一、概述

**SuperStageVFXActor** 是一个通过 DMX 控制 **Niagara 粒子系统** 的舞台特效 Actor。它将 DMX 信号映射为粒子系统的参数，实现对烟雾、火焰、彩带、雪花、CO2 喷射、烟花等舞台特效的实时控制。

### 类继承链

```
AActor (UE5 引擎基类)
  │
  └── ASuperBaseActor ·············· SuperStage 根基类 (L1)
        │
        └── ASuperDmxActorBase ····· DMX 灯具基类 (L2)
              │                      ├ DMX 地址 / 灯具库 / 通道读取
              │                      └ SuperDMXTick / 地址标签
              │
              └── ASuperLightBase ·· 电脑灯基类 (L3)
                    │                 ├ Pan/Tilt 控制 → 投射方向
                    │                 └ SceneRocker / SceneHard 旋转轴
                    │
                    └── ASuperStageVFXActor · 舞台特效 (L4) ← 本文档
                          ├ UNiagaraComponent（挂载到 SceneHard）
                          ├ SpawnCount / Color → Niagara 用户变量
                          ├ MaxSpawnCount / MaxSpeed / LoopTime
                          └ Minimum / Maximum 范围参数
```

> **设计要点**：`ASuperStageVFXActor` 继承自 `ASuperLightBase`，因此 Niagara 粒子组件挂载到 `SceneHard`（灯头旋转轴）上。这意味着粒子的发射方向可以通过 Pan/Tilt 远程控制——例如烟雾机可以通过控台调整喷射方向。

### 核心特点

- **Niagara 驱动** — 使用 UE 的 Niagara 粒子系统引擎，提供高品质的实时粒子效果
- **DMX 控制** — 通过 DMX 通道控制粒子数量（SpawnCount）和颜色（RGB）
- **继承 Pan/Tilt** — 继承自 SuperLightBase，Niagara 组件挂载到 SceneHard 上，支持 Pan/Tilt 旋转控制投射方向
- **可配置参数** — 速度、循环时间、范围等参数可在编辑器中自定义
- **自动初始化** — BeginPlay 时自动激活粒子系统并设置默认参数

### 应用场景

| 特效类型 | 说明 |
|---------|------|
| **烟雾机 (Smoke)** | DMX 控制烟雾发射量和颜色 |
| **火焰效果 (Fire)** | DMX 控制火焰强度和颜色 |
| **彩带炮 (Confetti)** | DMX 触发彩带喷射 |
| **雪花机 (Snow)** | DMX 控制雪花密度 |
| **CO2 喷射 (CO2)** | DMX 控制 CO2 喷射量 |
| **烟花特效 (Fireworks)** | DMX 触发烟花发射 |
| **水雾 / 泡泡** | DMX 控制喷射密度和颜色 |

---

## 二、组件结构

```
Actor Root (SceneBase)
  └── SceneRocker（Pan 旋转）
        └── SceneHard（Tilt 旋转）
              └── Niagara（粒子系统组件）
```

**Niagara 组件挂载到 SceneHard 上**，这意味着粒子的发射方向会跟随 Pan/Tilt 旋转。例如：
- 烟雾机可以通过 Pan/Tilt 控制烟雾的发射方向
- CO2 喷射方向可以远程调整
- 火焰效果的投射角度可以实时变化

---

## 三、属性详解

### 3.1 默认参数 (B.DefaultParameter)

| 参数 | 说明 | 范围 | 默认值 | 单位 |
|------|------|------|--------|------|
| **MaxSpawnCount** | 最大粒子生成数量 | 0 - 100 | 1.0 | 个/帧 |
| **MaxSpeed** | 粒子最大运动速度 | 0 - 100 | 100.0 | — |
| **LoopTime** | 粒子循环时间 | 0 - 100 | 100.0 | 秒 |
| **Minimum** | 最小范围/尺寸 | 0 - 100,000 | 100.0 | 厘米 |
| **Maximum** | 最大范围/尺寸 | 0 - 100,000 | 100.0 | 厘米 |

#### MaxSpawnCount（最大生成数量）

控制 DMX 值 = 1.0 时每帧生成的最大粒子数量：
- DMX SpawnCount = 0.0 → 不生成粒子（关闭特效）
- DMX SpawnCount = 0.5 → 生成 MaxSpawnCount × 0.5 个粒子
- DMX SpawnCount = 1.0 → 生成 MaxSpawnCount 个粒子

> **提示**：根据 Niagara 系统的设计调整此值。烟雾机可能需要较大值（50~100），而烟花可能只需 1~5。

#### MaxSpeed（最大速度）

设置 Niagara 系统中 `Speed` 变量的值，控制粒子的运动速度。具体影响取决于 Niagara 系统的模块配置。

#### LoopTime（循环时间）

设置 Niagara 系统中 `LoopTime` 变量的值，控制粒子效果的循环周期。

#### Minimum / Maximum（范围）

设置 Niagara 系统中 `Minimum` 和 `Maximum` 变量的值。通常用于控制粒子的发射范围或扩散范围，单位为厘米。

> **注意**：这些参数作为 Niagara 系统的**用户变量**传入。具体效果取决于你使用的 Niagara 系统资产中如何引用这些变量。需要确保 Niagara 系统中定义了 `Speed`、`LoopTime`、`Minimum`、`Maximum` 这些 Float 类型的用户变量。

### 3.3 调参建议

| 特效类型 | 推荐 MaxSpawnCount | 推荐 MaxSpeed | 推荐 LoopTime | 推荐 Min/Max | 说明 |
|---------|-------------------|--------------|--------------|-------------|------|
| **烟雾机** | 30 - 100 | 50 - 150 | 10 - 30 | 50 / 300 | 持续大量粒子，中低速扩散 |
| **CO2 喷射** | 50 - 200 | 200 - 500 | 1 - 5 | 10 / 100 | 短时间高速喷射，小范围 |
| **火焰效果** | 10 - 50 | 100 - 300 | 5 - 15 | 20 / 150 | 中等密度，向上运动 |
| **彩带炮** | 1 - 10 | 300 - 800 | 0.5 - 2 | 50 / 500 | 少量高速发射，大扩散范围 |
| **雪花机** | 50 - 200 | 10 - 50 | 30 - 100 | 200 / 1000 | 大量低速粒子，大覆盖范围 |
| **烟花** | 1 - 5 | 500 - 1000 | 1 - 3 | 100 / 800 | 极少量爆发性发射 |
| **泡泡机** | 5 - 30 | 20 - 80 | 10 - 30 | 50 / 400 | 低密度，缓慢飘动 |

> **提示**：以上为建议起始值，实际效果取决于你使用的 Niagara 系统资产的设计。建议先使用推荐值，然后在编辑器中实时预览并微调。

### 3.2 控制参数 (C.ControlParameter)

| 参数 | 说明 | 范围 | 默认值 |
|------|------|------|--------|
| **SpawnCount** | 粒子生成量（归一化） | 0.0 - 1.0 | 0.0 |
| **Color** | 粒子颜色 (RGB) | 每通道 0.0 - 1.0 | 白色 (1,1,1,1) |

---

## 四、DMX 控制映射

### 4.1 SpawnCount（粒子数量控制）

```
DMX 属性值 [0.0 ~ 1.0]
  ↓
Lerp(0, MaxSpawnCount, DMX值) → RoundToInt → 整数粒子数
  ↓
Niagara->SetVariableInt("SpawnCount", 整数值)
```

| DMX 值 | MaxSpawnCount = 50 时 | 效果 |
|--------|---------------------|------|
| 0.0 | 0 | 特效关闭 |
| 0.2 | 10 | 少量粒子 |
| 0.5 | 25 | 中等密度 |
| 1.0 | 50 | 最大密度 |

### 4.2 Color（颜色控制）

```
DMX R/G/B 三个通道 [各 0.0 ~ 1.0]
  ↓
组合为 FLinearColor(R, G, B)
  ↓
Niagara->SetVariableLinearColor("Color", 颜色值)
```

> **注意**：Niagara 系统中需要定义一个名为 `Color` 的 LinearColor 类型用户变量。

---

## 五、蓝图控制

### 5.1 控制函数

#### SetNiagaraDefaultValue（初始化）

初始化并激活 Niagara 粒子系统，设置默认参数。**运行时自动在 BeginPlay 中调用**。

执行内容：
1. 激活 Niagara 组件（`Activate()`）
2. 设置 `Speed` = MaxSpeed
3. 设置 `LoopTime` = LoopTime
4. 设置 `Minimum` = Minimum
5. 设置 `Maximum` = Maximum

#### SetNiagaraSpawnCount(DmxAttribute)（数量控制）

读取一个 DMX 属性，映射到粒子生成数量：

| 输入参数 | 说明 |
|---------|------|
| **DmxAttribute** | 控制生成数量的 DMX 属性 |

#### SetNiagaraColor(DmxR, DmxG, DmxB)（颜色控制）

读取三个 DMX 属性（RGB），映射到粒子颜色：

| 输入参数 | 说明 |
|---------|------|
| **DMXAttR** | 红色通道的 DMX 属性 |
| **DMXAttG** | 绿色通道的 DMX 属性 |
| **DMXAttB** | 蓝色通道的 DMX 属性 |

### 5.2 典型蓝图实现

```
Event SuperDMXTick(DeltaTime)
  │
  ├─ SetNiagaraSpawnCount(SpawnCount)     ← 控制粒子数量
  └─ SetNiagaraColor(Red, Green, Blue)    ← 控制粒子颜色
```

如果需要 Pan/Tilt 控制投射方向（继承自 SuperLightBase）：

```
Event SuperDMXTick(DeltaTime)
  │
  ├─ SuperLightControl(Pan, Tilt)         ← 控制投射方向
  ├─ SetNiagaraSpawnCount(SpawnCount)     ← 控制粒子数量
  └─ SetNiagaraColor(Red, Green, Blue)    ← 控制粒子颜色
```

---

## 六、DMX 通道规划

### 基本配置（4 通道）

| 通道 | 属性 | 说明 |
|------|------|------|
| 1 | SpawnCount | 粒子数量 |
| 2 | Red | 红色 |
| 3 | Green | 绿色 |
| 4 | Blue | 蓝色 |

### 含 Pan/Tilt 控制（8 通道）

| 通道 | 属性 | 精度 | 说明 |
|------|------|------|------|
| 1-2 | Pan | 16位 | 水平旋转 |
| 3-4 | Tilt | 16位 | 垂直旋转 |
| 5 | SpawnCount | 8位 | 粒子数量 |
| 6 | Red | 8位 | 红色 |
| 7 | Green | 8位 | 绿色 |
| 8 | Blue | 8位 | 蓝色 |

---

## 七、创建自定义 Niagara 特效

SuperStageVFXActor 可以与**任何 Niagara 系统**配合使用，只需确保 Niagara 系统中定义了以下**用户变量**：

### 必须的用户变量

| 变量名 | 类型 | 说明 |
|--------|------|------|
| **SpawnCount** | Int32 | 粒子生成数量 |
| **Color** | LinearColor | 粒子颜色 |

### 可选的用户变量（初始化时设置）

| 变量名 | 类型 | 说明 |
|--------|------|------|
| **Speed** | Float | 运动速度 |
| **LoopTime** | Float | 循环时间 |
| **Minimum** | Float | 最小范围 |
| **Maximum** | Float | 最大范围 |

### 创建步骤

1. 在 Content Browser 中创建新的 **Niagara System**
2. 在系统中添加上述用户变量
3. 在模块中引用这些变量控制粒子行为
4. 将 Niagara System 资产分配给 SuperStageVFXActor 的 Niagara 组件
5. 调整 MaxSpawnCount、MaxSpeed 等默认参数
6. 配置 DMX 并测试

---

## 八、使用步骤

1. **放置 Actor** — 将 SuperStageVFXActor 蓝图拖入场景
2. **选择 Niagara 系统** — 在 Niagara 组件上设置你需要的粒子系统资产
3. **设置默认参数** — 调整 MaxSpawnCount、MaxSpeed、LoopTime、Minimum、Maximum
4. **调整位置和方向** — 放到特效发射位置
5. **配置 DMX** — 分配 Universe、Start Address、灯具库
6. **从控台控制** — 推动 SpawnCount 推杆触发特效，推动 RGB 推杆调整颜色

---

## 九、常见问题

### Q: 粒子不显示？
1. 确认 Niagara 组件上已**分配了 Niagara System 资产**
2. 确认 DMX 的 SpawnCount 值 > 0
3. 确认 MaxSpawnCount 不为 0
4. 检查 Niagara 系统中是否正确引用了 `SpawnCount` 用户变量

### Q: 颜色不变化？
确认 Niagara 系统中定义了名为 `Color` 的 **LinearColor** 类型用户变量，且在粒子渲染模块中使用了该变量。

### Q: 如何做"一键触发"效果（如烟花）？
- 将 MaxSpawnCount 设为较小值（如 1~3）
- 在控台上快速将 SpawnCount 推到满，然后拉回 0
- Niagara 系统应配置为"发射后自动播放"模式（非循环）

### Q: Pan/Tilt 方向控制不生效？
确认在蓝图中调用了 SuperLightControl（Pan/Tilt 控制函数），并且相关的 DMX 属性已正确配置。Pan/Tilt 功能继承自 SuperLightBase。

### Q: 如何让不同特效使用不同的 Niagara 系统？
每个 SuperStageVFXActor 的 Niagara 组件可以**独立分配不同的 Niagara System 资产**。创建多个 Actor，分别分配烟雾、火焰、雪花等不同的 Niagara 系统即可。

---

> **返回总览**：[00 - DMX 系统总览](/docs/stage-core/overview)

# 08 - DMX 录制与回放

> **所属模块**: SuperDMX Sequencer / SuperStageEditor Recording  
> **适用对象**: 灯光编程师、虚拟制作技术人员  
> **前置阅读**: [01 - DMX 网络配置](/docs/stage-core/dmx-network)  
> **最后更新**: 2026-03-06

---

## 一、概述

SuperStage 支持将外部 DMX 输入信号**录制**为 UE Sequencer 动画，并可以在之后**回放**这些动画，同时将 DMX 信号重新输出到网络。

### 核心能力

| 功能 | 说明 |
|------|------|
| **DMX 录制** | 将控台发送的实时 DMX 信号逐帧录制到 Sequencer 轨道 |
| **DMX 回放** | 播放 Sequencer 中录制的 DMX 动画，驱动虚拟灯具 |
| **DMX 输出** | 回放时将 DMX 数据通过 Art-Net 发送到外部真实灯具 |

### 典型工作流

```
控台实时编程灯光秀
       ↓
使用 Take Recorder 录制 DMX 信号
       ↓
在 Sequencer 中查看/编辑录制的动画
       ↓
播放 Sequencer → 虚拟灯具响应 + 输出到真实灯具
```

---

## 二、录制 DMX（使用 Take Recorder）

SuperStage 通过 UE 内置的 **Take Recorder** 系统录制 DMX 数据。

### 2.1 准备工作

1. **确保 DMX 输入已启用** — 参见 [01 - 网络配置](/docs/stage-core/dmx-network)
2. **确认 DMX 信号正常接收** — 在活动监视器中验证
3. **打开 Take Recorder** — 菜单栏 Window > Cinematics > Take Recorder

### 2.2 添加 DMX 录制源

1. 在 Take Recorder 面板中，点击 **+ Source** 按钮
2. 在弹出的列表中找到 **Super DMX Input**
3. 点击添加

添加后，你会看到一个新的录制源，可以配置以下参数：

| 参数 | 说明 | 默认值 | 范围 |
|------|------|--------|------|
| **Universe Min** | 要录制的最小 Universe 编号 | 1 | 1 - 任意正整数 |
| **Universe Max** | 要录制的最大 Universe 编号 | 16 | 1 - 任意正整数 |

**Universe 范围说明**：
- 系统会录制从 Universe Min 到 Universe Max（包含两端）的所有 Universe
- 例如设置 Min=1, Max=4，则录制 Universe 1、2、3、4
- 如果 Min > Max，系统会自动交换（例如 Min=4, Max=1 等效于 Min=1, Max=4）
- 只有实际有数据的 Universe 才会产生录制内容

> **建议**：只录制你需要的 Universe 范围，避免产生过大的录制文件。如果你的灯具只用了 Universe 1-3，就设置 Min=1, Max=3。

### 2.3 开始录制

1. 在控台上准备好你的灯光秀
2. 在 Take Recorder 中点击 **Record** 按钮
3. 在控台上执行灯光秀（推推杆、切 Cue 等）
4. 录制完成后，点击 **Stop** 按钮

### 2.4 录制过程中发生了什么

录制期间，系统执行以下操作：

1. **每帧采样** — 跟随引擎 Tick 频率（通常 30-120 fps），无节流
2. **读取所有通道** — 从 SuperDMX 子系统获取指定 Universe 范围内每个 Universe 的全部 512 通道数据
3. **写入关键帧** — 将每个通道的值作为关键帧添加到 Sequencer 曲线中
4. **常量插值** — 关键帧使用常量插值模式（RCIM_Constant），确保 DMX 值精确（不使用平滑插值）
5. **自动扩展** — Section（区段）的时间范围自动扩展到当前录制帧

### 2.5 录制保护机制

录制期间，系统会设置 **bIsRecording = true** 标记，阻止 Sequencer 回放发送 DMX 数据。这避免了"回声"问题——即录制的数据被再次发送出去，导致信号循环。

---

## 三、录制结果

录制完成后，Take Recorder 会生成一个 **Level Sequence** 资产，其中包含 DMX 轨道。

### 3.1 轨道结构

```
Level Sequence
  └── SuperDMX Track (每个 Universe 一个独立轨道)
        ├── Universes_1  ← Universe 1 的轨道
        │     └── Section (包含512个通道的曲线数据)
        ├── Universes_2  ← Universe 2 的轨道
        │     └── Section
        └── Universes_3  ← Universe 3 的轨道
              └── Section
```

### 3.2 轨道命名

每个 Universe 的轨道自动命名为 `Universes_{编号}`，例如：
- `Universes_1`
- `Universes_2`
- `Universes_3`

### 3.3 数据格式

每个 Section（区段）中包含：

| 数据 | 说明 |
|------|------|
| **Universe** | 该区段对应的 Universe 编号 |
| **Channel Curves** | 512 个 FRichCurve，每个曲线存储一个通道的关键帧数据 |

每条曲线的关键帧：
- **时间轴**：基于 MovieScene 的 TickResolution
- **值**：0-255（uint8 DMX 原始值）
- **插值**：常量模式（跳变，不做平滑）

---

## 四、回放 DMX（Sequencer 播放）

### 4.1 基本回放

1. 双击打开录制生成的 Level Sequence
2. 在 Sequencer 编辑器中点击 **Play** 按钮
3. Sequencer 会逐帧评估 DMX 曲线数据
4. 评估结果自动发送到 SuperDMX 子系统
5. 场景中的虚拟灯具实时响应

### 4.2 回放数据流

```
Sequencer 播放
  ↓
DMX Template 评估 (每帧)
  ↓
读取当前时间对应的通道值 (从 RichCurve 曲线)
  ↓
构建 512 字节 DMX 缓冲区
  ↓
调用 SendDMXBuffer 发送
  ↓
虚拟灯具响应 + Art-Net 输出到网络
```

### 4.3 缓冲合并策略

回放时的 DMX 缓冲区使用**合并策略**（不是覆盖）：

1. 首先读取该 Universe 的当前状态作为基础（"种子"）
2. 仅覆盖 Sequencer 曲线中定义的通道
3. 保留其他来源（如外部控台输入）的通道值

**这意味着**：你可以在 Sequencer 中只录制/编辑部分通道，其他通道仍然可以由外部控台实时控制。

### 4.4 输出到外部设备

如果 DMX 配置面板中**输出已启用**，回放时的 DMX 数据会同时通过 Art-Net 发送到网络，驱动真实灯具。

确保：
- DMX 配置面板中输出已启用
- 输出的本地 IP 和远程 IP 设置正确
- 真实灯具的 Universe 和地址与录制时一致

---

## 五、在 Sequencer 中编辑 DMX 数据

录制后的 DMX 数据存储为标准的 UE Sequencer 曲线，可以使用 Sequencer 编辑器进行编辑。

### 5.1 可以做的操作

| 操作 | 说明 |
|------|------|
| **移动关键帧** | 调整特定通道在某个时间点的值 |
| **删除关键帧** | 移除不需要的数据点 |
| **复制/粘贴** | 复制一段时间的数据到另一个时间点 |
| **修改时间范围** | 裁剪或扩展 Section 的播放范围 |
| **修改 Universe** | 在 Section 属性中更改目标 Universe |

### 5.2 手动创建 DMX 轨道

你也可以不经过录制，直接在 Sequencer 中手动创建 DMX 轨道：

1. 在 Sequencer 中点击 **+ Track** 按钮
2. 选择 **SuperDMX Track**
3. 系统创建一个空的 DMX 轨道
4. 使用 **Add New Section** 为指定 Universe 添加区段
5. 手动添加关键帧到通道曲线中

---

## 六、录制参数详解

### 6.1 Take Recorder 录制源参数

| 参数 | 说明 | 默认值 |
|------|------|--------|
| **Universe Min** | 录制的最小 Universe | 1 |
| **Universe Max** | 录制的最大 Universe | 16 |
| **Track Tint** | 轨道在 Sequencer 中的颜色标记 | 橙色 (255, 160, 0) |

### 6.2 录制精度

| 特性 | 值 |
|------|-----|
| **采样频率** | 跟随引擎 Tick（通常 30-120 fps） |
| **通道精度** | 8 位 (0-255) |
| **通道数量** | 每个 Universe 最多 512 通道 |
| **插值模式** | 常量 (RCIM_Constant) |
| **时间精度** | 跟随 MovieScene 的 TickResolution |

### 6.3 录制文件大小预估

录制文件大小取决于 Universe 数量、录制时长和帧率：

```
大约大小 = Universe数量 × 512通道 × 每帧关键帧大小 × 帧率 × 录制秒数
```

**参考值**（仅粗估）：

| Universe 数量 | 录制时长 | 帧率 | 预估大小 |
|--------------|---------|------|---------|
| 1 | 1 分钟 | 30 fps | ~10 MB |
| 4 | 5 分钟 | 30 fps | ~200 MB |
| 16 | 10 分钟 | 60 fps | ~3 GB |

> **建议**：只录制实际需要的 Universe 范围，避免录制空闲 Universe 浪费空间。

---

## 七、高级使用场景

### 7.1 录制 + 实时控制混合

你可以将 Sequencer 中的录制回放与控台实时控制混合使用：

- Sequencer 回放 Universe 1-4（预录的灯光秀）
- 控台实时控制 Universe 5-8（现场追光等）
- 两者互不干扰

### 7.2 多次录制叠加

1. 第一次录制 Universe 1-2（电脑灯）
2. 第二次录制 Universe 3-4（LED 面板）
3. 两次录制的轨道会出现在同一个 Level Sequence 中
4. 回放时同时驱动所有 Universe

### 7.3 录制后离线编辑

录制完成后，你可以断开控台连接，在 Sequencer 中离线编辑 DMX 曲线数据。编辑完成后再连接灯具进行回放输出。

---

## 八、常见问题

### Q: 录制后播放但灯具不动？
1. 确认 Sequencer 中的 DMX 轨道有数据（展开轨道查看关键帧）
2. 检查灯具的 Universe 和地址是否与录制时一致
3. 确认不处于录制状态（bIsRecording 应为 false）

### Q: 录制的数据与控台不一致？
1. 检查 DMX 配置面板中的起始 Universe 设置
2. 使用活动监视器确认接收到的原始数据是否正确
3. 确认录制的 Universe 范围覆盖了目标灯具

### Q: 录制文件太大？
- 减小录制的 Universe 范围（只录制有实际数据的 Universe）
- 录制完成后删除空白通道的关键帧
- 缩短录制时长

### Q: 回放时有卡顿？
- 大量 Universe + 高帧率录制可能导致性能问题
- 尝试降低 Sequencer 的播放帧率
- 减少同时回放的 Universe 数量

### Q: 回放时会影响控台的实时控制吗？
Sequencer 回放使用**合并策略**：只覆盖曲线中有数据的通道，其他通道保持不变。如果你需要完全隔离，使用不同的 Universe。

### Q: 可以将 Sequencer 中的 DMX 数据导出吗？
当前版本不支持直接将 Sequencer DMX 数据导出为 Art-Net 文件。但你可以通过回放 + 启用输出的方式，将数据实时输出到外部录制设备。

---

> **下一步**：请阅读 [09 - MA 控台导出](/docs/stage-core/export-ma) 了解如何将灯具配置导出到 MA 控台。

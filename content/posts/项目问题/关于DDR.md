+++

title = '关于DDR'
date = '2026-06-08T16:35:00+08:00'
draft = false

+++

## 一、从实际案例出发：1G → 2G 为什么不需要改代码？

### 1.1 实验现象

板子从 1G DDR 换到 2G DDR，**软件未做任何改动**，上电直接正常启动：

```
# cat /proc/meminfo | head -3
MemTotal:        2036132 kB          ← 2GB 全部识别
MemFree:         1901472 kB
MemAvailable:    1938064 kB

# cat /sys/class/aml_ddr/freq
504 MHz                             ← 频率不变

# dmesg | grep "Memory:"
Memory: 1999064K/2097152K available
Virtual kernel memory layout:
  memory  : 0xffffffa640000000 - 0xffffffa6c0000000   (2048 MB)

# cat /proc/iomem | head -1
00000000-7fffffff : System RAM       ← 2GB 地址空间全覆盖
```

### 1.2 为什么可以不改？

答案在于 Amlogic（以及多数现代嵌入式 SoC）的 DDR 初始化架构。整个 DDR 初始化流程分为两个阶段：

**第一阶段：BL2 负责 DDR 硬件初始化（决定性的阶段）**

BL2 是 SoC 上电后最早运行的代码，它内部集成了 **Synopsys DDR Firmware**（一个闭源的二进制 blob）。在我们的源码中：

```
bootloader/uboot-repo/fip/a5/
├── aml_ddr.fw          ← BL2 集成的 DDR 主固件（自动训练 + 初始化）
├── ddr4_1d.fw          ← DDR4 单 Rank 参数固件（1d = 1 Rank）
├── ddr4_2d.fw          ← DDR4 双 Rank 参数固件（2d = 2 Rank）
├── lpddr4_1d.fw
├── lpddr4_2d.fw
├── piei.fw             ← PHY 初始化固件
```

BL2 的 DDR 固件在上电时会执行以下流程：

```
上电 → 读取 DDR 颗粒的 Mode Register → 识别颗粒类型/密度/位宽
     → 执行 Write Leveling（写均衡）
     → 执行 Read DQS Gate Training（读门控训练）
     → 执行 Read/Write Data Eye Training（数据眼图训练）
     → 执行 VREF Training（参考电压训练）
     → 自动配置 DDR 控制器寄存器（时序、容量、地址映射）
     → 将检测到的内存大小写入硬件寄存器/共享内存
```

**BL2 的 DDR FW 是自适应训练的**。它不需要你在代码里写死"这颗 DDR 是 1GB 还是 2GB"——它会自动从颗粒的 Mode Register 中读出密度信息、自动训练信号时序、然后自动配置 DDR 控制器的地址解码窗口。

这就是你不改代码就能工作的根本原因：**DDR 类型没变（都是 DDR4）、Rank 配置没变（都是 1d/单 Rank）、位宽没变（都是 32-bit），唯一变化的是颗粒密度（从 4Gb die 换成 8Gb die，2 颗芯片 × 4Gb = 8Gb = 1GB → 2 颗 × 8Gb = 16Gb = 2GB），而 BL2 固件自动识别并适配了这一切。**

**第二阶段：U-Boot (BL33) 传递内存信息给内核**

U-Boot 从 BL2 那里拿到已经初始化好的内存容量信息，然后动态修改 Device Tree 中的 memory 节点，再传给内核：

```
U-Boot 读取 BL2 写好的 DDR 容量
    → 修改 /memory@0 节点的 reg = <0x0 0x0 0x0 0x80000000>;  （2GB）
    → 传递给 Linux kernel
    → 内核看到的就是完整的 2GB
```

从板子的 Device Tree 来看，内核确实拿到了 2GB：

```bash
# od -An -tx4 /proc/device-tree/memory@00000000/reg
 00000000 00000000 00000000 0000807f   # size = 0x7f800000 ≈ 2GB
```

### 1.3 什么情况下 1G→2G 就 **需要** 改软件？

先给出结论表，再逐一解释为什么。

#### 1.3.1 场景速查表

假设我们是一块 A113X2 开发板，DDR4、32-bit 位宽、贴 **2 颗 x16 芯片**（依据见 2.5 节源码证据：`dram_x4x8x16_mode = CONFIG_DRAM_MODE_X16`，32-bit ÷ 16-bit = 2 颗）。从 1G 总容量升级到 2G 总容量，有以下可能的实现方式：

| 编号 | 升级方式 | 物理变化 | 软件要不要改？ |
|------|---------|---------|-------------|
| **S1** | 1×4Gb die → 1×8Gb die | 每颗芯片换更大密度的单 Die | **❌ 不需要** |
| **S2** | 2×2Gb die → 2×4Gb die | 每颗芯片仍是双 Die 封装(DDP)，但 Die 密度翻倍 | **❌ 不需要** |
| **S3** | 1×4Gb die → 2×4Gb die | 每颗从单 Die 换成双 Die 封装(DDP) | **❌ 不需要** |
| **S4** | 2×2Gb die → 1×8Gb die | 每颗从双 Die 封装换成大容量单 Die | **❌ 不需要** |
| **S5** | 1 Rank 2 chips → 2 Rank 4 chips | PCB 上多贴 2 颗芯片，新增 CS1 信号 | **✅ 必须改** |
| **S6** | 1 Rank 2 chips@4Gb → 1 Rank 2 chips@8Gb | 同 S1，唯容量变 | **❌ 不需要** |

**核心规律：只要 Rank 数量不变、DDR 类型不变、位宽不变——不管 Die 怎么组合（单 Die 变大、DDP 换单 Die、DDP 内部 Die 变大），BL2 全部自动处理。因为 BL2 只认 CS 信号上的芯片的 Mode Register，不关心芯片内部封了几个 Die。**

下面逐个场景详细解释"为什么"。

---

#### 1.3.2 S1：1×4Gb die → 1×8Gb die（单 Die 换大密度）—— ❌ 不需要

**物理变化：**
```
升级前：                            升级后：
每颗芯片 = 1 个 4Gb Die             每颗芯片 = 1 个 8Gb Die
2 颗 × 4Gb = 8Gb = 1GB              2 颗 × 8Gb = 16Gb = 2GB
```

**BL2 做了什么（自动）：**
1. 上电后，BL2 的 DDR FW 通过 JEDEC 标准命令读取每颗芯片的 **Mode Register**（DDR4 MR2/MR3 中包含 density 字段）
2. 升级前，MR 回报 density = 4Gb → BL2 算得总容量 = 2 × 4Gb = 1GB
3. 升级后，MR 回报 density = 8Gb → BL2 算得总容量 = 2 × 8Gb = 2GB
4. BL2 根据新的 density 自动调整 DDR 控制器的地址映射（Column/Row/Bank 的地址位宽）
5. BL2 根据 density 自动调整 tRFC（刷新周期——大容量颗粒需要更长的刷新时间）

**为什么不需要改：** 从 DDR 控制器的视角看，它始终面对的是"1 个 CS 信号上，2 颗 x16 芯片，DDR4 协议"。CS 上的芯片密度变了，但芯片自己会通过 Mode Register 告诉控制器。整个过程是 JEDEC 标准化的——不是 Amlogic 的私有实现。

---

#### 1.3.3 S2：2×2Gb die → 2×4Gb die（DDP 内部 Die 升级）—— ❌ 不需要

**物理变化：**
```
升级前：                            升级后：
每颗芯片                            每颗芯片
┌──────────────┐                    ┌──────────────┐
│  2Gb Die 0   │                    │  4Gb Die 0   │
│  2Gb Die 1   │                    │  4Gb Die 1   │
│  ──────────  │   ← DDP 封装       │  ──────────  │   ← DDP 封装
│  总 = 4Gb    │                    │  总 = 8Gb    │
└──────────────┘                    └──────────────┘
2 颗 × 4Gb = 8Gb = 1GB              2 颗 × 8Gb = 16Gb = 2GB
```

**BL2 做了什么（自动）：** 和 S1 完全一样。芯片内部的 Die 数量/密度对外部透明——Mode Register 只报告总密度。升级前 MR 回报 4Gb，升级后 MR 回报 8Gb。BL2 不关心这 8Gb 是 1×8Gb 还是 2×4Gb 还是 4×2Gb 组成的。

**为什么不需要改：** 双 Die 封装（DDP）的芯片，内部两个 Die 共享同一组 DQ 总线，通过内部 Die 选择信号（通常由高位地址线充当）来切换。对 DDR 控制器来说，它只看到一个"容量=4Gb"的芯片接在 DQ 总线上。封装的内部结构对软件完全透明。

---

#### 1.3.4 S3：1×4Gb → 2×4Gb（单 Die 换 DDP，容量翻倍）—— ❌ 不需要

**物理变化：**
```
升级前：                            升级后：
┌──────────────┐                    ┌──────────────┐
│              │                    │  4Gb Die 0   │
│  1 个 4Gb    │                    │  4Gb Die 1   │
│  Die         │                    │  ──────────  │
│              │                    │  总 = 8Gb    │
└──────────────┘                    └──────────────┘
每颗 4Gb，2 颗 = 1GB                 每颗 8Gb，2 颗 = 2GB
```

**物理上**，封装从单 Die 变成了双 Die（DDP），PCB 上的 Pad 排布可能不同，但电气接口（DQ/DQS/CA）完全一致。

**BL2 做了什么（自动）：** 同上。MR 回报 density = 8Gb，BL2 自动配置。

**为什么不需要改：** 虽然硬件封装变了，但 JEDEC DDR4 协议规定，Mode Register 中的 density 字段代表的是"这颗芯片的总容量"，不管内部怎么实现。BL2 只读 MR，不关心封装。

---

#### 1.3.5 S4：2×2Gb → 1×8Gb（DDP 换成大单 Die）—— ❌ 不需要

**物理变化：**
```
升级前：                            升级后：
每颗芯片 = 2 个 2Gb Die (DDP)       每颗芯片 = 1 个 8Gb Die (单 Die)
总 = 4Gb/chip × 2 = 1GB             总 = 8Gb/chip × 2 = 2GB
```

这是 S3 的反向操作。从 DDP 换成单 Die（密度增大），对外表现完全一致。

**为什么不需要改：** 和 S3 同理。Mode Register 只回报容量，不回报封装形式。2×2Gb 和 1×4Gb 和 1×8Gb 对你的 DDR 控制器来说都只是 "MR density = X Gb"。这也是为什么在 DDR 适配中，软件工程师最关键的问题是"这颗料是几 Rank、什么类型、多大密度"，而不是"这颗料封了几个 Die"。

---

#### 1.3.6 S5：1 Rank → 2 Rank — **✅ 必须改**

这是唯一一个 1G→2G 中 **必须改软件** 的场景。

**物理变化：**
```
升级前（1 Rank）：                   升级后（2 Rank）：
CS0 ──→ 2 颗芯片 (每颗 4Gb)         CS0 ──→ 2 颗芯片 (每颗 4Gb)
CS1   （未接）                       CS1 ──→ 2 颗芯片 (每颗 4Gb)  ← 新增！
总 = 2 × 4Gb = 8Gb = 1GB            总 = 4 × 4Gb = 16Gb = 2GB
```

**为什么必须改 — 三个层面的原因：**

**第一层：BL2 不知道 CS1 上有芯片。**

BL2 初始化 DDR 时，必须明确告知它"你需要去训练 CS1 上的芯片"。这个信息不在 Mode Register 里——Mode Register 只能告诉你 CS0 上有多少容量的芯片，不会告诉你"CS1 上也贴了芯片"。CS1 是完全独立的片选信号，需要 BL2 主动去拉低它、初始化它、训练它。如果 BL2 按 1 Rank（`dram_rank_config = 0x2`）初始化，它根本不会触碰 CS1，那 CS1 上的 2 颗芯片相当于不存在，系统只看到 1GB。

类比：你办公室有两个房间（CS0 和 CS1）。BL2 是一个自动巡检员，你告诉它"只查 1 号房间"，它就只查 1 号房间、只统计 1 号房间里的人。2 号房间里就算坐满了人，巡检员也不会去开门。

**第二层：DDR Firmware 要换。**

在 Amlogic 的 BL2 打包脚本中（`build.sh`），1 Rank（1d）和 2 Rank（2d）使用的是不同的 Synopsys DDR 固件：

```bash
# build.sh 中：1d 和 2d 的 FW 分别打包
dd if=ddr4_1d.fw of=ddrfw_1d.bin ...    # 1 Rank 训练固件
dd if=ddr4_2d.fw of=ddrfw_2d.bin ...    # 2 Rank 训练固件
```

2 Rank 的初始化流程和 1 Rank 有本质区别：
- 需要分别对 CS0 和 CS1 上的芯片做 Mode Register 初始化
- Write Leveling（写均衡）需要分别在两个 Rank 上执行
- Read/Write Data Eye Training 需要分别扫描两个 Rank 的眼图
- ZQ Calibration 需要协调两个 Rank 的 ODT 行为（一个 Rank 工作时，另一个 Rank 的 ODT 是重要的终端匹配）

**第三层：地址映射要变。**

单 Rank 时，所有地址位都用于寻址一颗芯片内部的 Row/Bank/Column。双 Rank 时，需要额外的一个地址位（通常是高位地址线）作为 **Rank 选择位**——用来告诉 DDR 控制器"这次操作是发到 CS0 的 Rank 还是 CS1 的 Rank"。

这不是 DDR 颗粒自己能决定的——这是 DDR 控制器内部的地址映射逻辑，必须通过 `dram_rank_config` 寄存器告诉控制器。

```c
// 具体要改的参数（在 acs.bin 中）
dram_rank_config = 0x2   // 改为 → 0x7
// 0x2 = CONFIG_DDR0_32BIT_RANK0_CH0    ← 单 Rank
// 0x7 = CONFIG_DDR0_32BIT_RANK01_CH0   ← 双 Rank，共用 CH0
```

---

#### 1.3.7 S6：同 S1——不赘述。

---

#### 1.3.8 判定流程图

```
                     1G → 2G 升级
                          │
               ┌──────────┴──────────┐
               │                      │
         容量翻倍方式              容量翻倍方式
       (1 Rank 内密度变大)        (增加 Rank 数量)
               │                      │
       查 DDR 颗粒 Datasheet    查原理图上是否多了 CS1
               │                      │
      ┌────────┴────────┐            YES
      │                  │            │
   类型不变            类型变了        │
   (都是DDR4)        (DDR3→DDR4)      │
      │                  │            │
   位宽不变            位宽变了        │
   (都是32-bit)      (32→16-bit)      │
      │                  │            │
   同 Rank               │            │
      │                  │            │
     ✅                  │            │
  不需要改！          ✅ 需要改       ✅ 需要改
                     DDRFW_TYPE      dram_rank_config
                     DramType        DDR FW 从 1d→2d
                     PMIC电压        （可能还要改 ODT
                     DisabledDbyte    和 ZQ 配置）
```

**一句话总结：BL2 的 DDR 固件是一个"自适应密度"的智能初始化程序，但它的智能边界是 JEDEC Mode Register——MR 能告诉它的（密度、Bank 数、位宽配置），它能自动处理；MR 不能告诉它的（CS1 上有没有芯片、DDR 类型变了、位宽变了），你必须人工告诉它。**

---

## 二、Die、Rank、Channel、CS — 概念与封装对应关系

这是 DDR 适配中最容易搞混的概念。以下从"一颗芯片长什么样"的角度来梳理。

### 2.1 Die（裸片 / 晶粒）

**Die** 是晶圆切割后未封装的最小存储单元。一个 Die 有自己的存储阵列（Bank/Row/Column）、控制逻辑、Mode Register。

- 一个 DDR4 Die 的常见容量：512Mb、1Gb、2Gb、4Gb、8Gb（注意这里的 b 是 bit，8Gb = 1GB）
- Die 决定了颗粒的 **密度** 和 **内部 Bank 数量**

### 2.2 封装 (Package) — 一颗芯片里可能有几个 Die

DDR 芯片的封装可以容纳不同数量的 Die：

```
情况 A：单 Die 封装
┌─────────────────────┐
│  DDR4 芯片 (FBGA)    │
│  ┌───────────────┐  │
│  │  1 个 2Gb Die  │  │    ← 这颗芯片总容量 = 2Gb = 256MB
│  └───────────────┘  │
└─────────────────────┘

情况 B：双 Die 封装（DDP - Dual Die Package）
┌─────────────────────┐
│  DDR4 芯片 (FBGA)    │
│  ┌───────┐┌───────┐ │
│  │1Gb Die││1Gb Die│ │    ← 这颗芯片总容量 = 2Gb = 256MB
│  │ Die 0 ││ Die 1 │ │       但物理上是两个 Die 叠在一起
│  └───────┘└───────┘ │
└─────────────────────┘

情况 C：四 Die 封装 (QDP)
┌─────────────────────┐
│  DDR4 芯片 (FBGA)    │
│  ┌───┐┌───┐┌───┐┌──┐│
│  │Die││Die││Die││Di││    ← 4 个 Die 堆叠
│  │ 0 ││ 1 ││ 2 ││ 3││
│  └───┘└───┘└───┘└──┘│
└─────────────────────┘
```

**一颗封装好的 DDR 芯片（一颗料），里面可能是 1 个 2Gb Die，也可能是 2 个 1Gb Die。对外部来说都表现为"一颗 256MB 的 DDR 芯片"，但内部结构不同。**

### 2.3 Rank — 软件视角的"片选组"

> 一个DDR的封装可能存在多个Die，但是在设计之初就决定了他是采用多个Rank还是单个Rank
>
> SoC 的 DDR Controller 决定了SOC端有多少个CS引脚，一般是两个，高端可以到4个，也就是4Rank
>
> 同时多个DDR芯片的Rank可以接在同一个CS引脚上面，以提供足够的带宽
>
> 还有SOC上的多个CS并不一定要全部使用，具体根据DDR的设计来

**Rank** 是 DDR 控制器通过片选信号（CS/Chip Select）独立选中的一组芯片。一个 Rank 对应一个独立的 CS 信号。

- **单 Rank (1R / 1d)**：所有 DDR 芯片共享同一个 CS 信号，DDR 控制器一次只能对这一整组芯片操作
- **双 Rank (2R / 2d)**：两组芯片各有一个 CS 信号（CS0 和 CS1），控制器可以在两组之间切换。一个 Rank 工作时，另一个 Rank 可以做 Precharge/Refresh

**Die 和 Rank 的关系：**（以下为概念演示，以 x8 颗粒 4 颗为例；本板实际是 x16 颗粒 2 颗，换算关系相同）

```
场景 1：单 Rank 单 Die
  PCB 上贴 4 颗 DDR4 芯片，每颗是单 Die 封装
  4 颗共用 1 个 CS0 → 1 Rank
  DDR 控制器看到：1 Rank，32-bit 位宽，容量 = 4 × 单颗容量

场景 2：单 Rank 双 Die（DDP 单 Rank）
  PCB 上贴 4 颗 DDR4 芯片，每颗是双 Die 封装（DDP）
  但 4 颗仍然共用 1 个 CS0，两个 Die 共享同一组数据总线
  → 1 Rank（但内部 2 个 Die 共享同一个 Rank）
  DDR 控制器看到：1 Rank，32-bit 位宽，容量 = 4 × 2 × 单 Die 容量

场景 3：双 Rank（独立 CS）
  PCB 上贴 8 颗 DDR4 芯片，分成两组：
  组 A（4 颗）接 CS0 → Rank 0
  组 B（4 颗）接 CS1 → Rank 1
  DDR 控制器看到：2 Rank，32-bit 位宽，容量 = 8 × 单颗容量
```

### 2.4 Channel（通道）

Channel 是 DDR 控制器内部独立的物理接口。每个 Channel 有自己完整的数据总线（DQ）、地址/命令总线（CA）、控制信号。

- Amlogic A113X2 (A5) 的 DDR 子系统通常配置为 **单 Channel**，32-bit 位宽
- 部分高端 SoC 支持双 Channel 甚至是 4 Channel

```
A113X2 (A5) 典型配置：
┌──────────────────────────────────────┐
│             DDR 控制器 (DMC)          │
│                                      │
│  ┌────────────────────────────────┐  │
│  │          Channel 0             │  │
│  │   32-bit DQ (DQ[0:31])        │  │
│  │   4 DQS pairs (DQS[0:3])      │  │
│  │   CS0 (Rank 0)                │  │
│  │   CS1 (Rank 1) — 可选          │  │
│  └────────────────────────────────┘  │
│                                      │
└──────────────────────────────────────┘
```

### 2.5 从 PCB 到软件的对应关系

以 A113X2 (AV409) 开发板为例，DDR 配置直接来自板级源码 `bl33/v2019/board/amlogic/a5_av409/firmware/timing.c`：

```c
// timing.c 中 A5_1RANK_DDR4 配置块（当前 AV409 启用项）
.cfg_board_common_setting.DramType          = CONFIG_DDR_TYPE_DDR4,        // DDR4
.cfg_board_common_setting.dram_rank_config  = CONFIG_DDR0_32BIT_RANK0_CH0, // 32-bit 单 Rank
.cfg_board_common_setting.DisabledDbyte     = CONFIG_DISABLE_D32_D63,      // 0xf0 → 只用 byte0-3
.cfg_board_common_setting.dram_cs0_size_MB  = CONFIG_DDR0_SIZE_2048MB,     // CS0 = 2048MB = 2GB
.cfg_board_common_setting.dram_cs1_size_MB  = CONFIG_DDR1_SIZE_0MB,        // CS1 = 0 → 单 Rank
.cfg_board_common_setting.dram_x4x8x16_mode = CONFIG_DRAM_MODE_X16,        // X16 颗粒
.cfg_board_common_setting.Is2Ttiming        = CONFIG_USE_DDR_2T_MODE,      // 2T 命令速率
```

宏定义（`arch-a5/ddr_define.h`）：
```c
#define CONFIG_DRAM_MODE_X16       0    // X16 颗粒模式
#define CONFIG_DDR0_SIZE_2048MB    2048 // CS0 容量 = 2048MB
```

对应的物理连接（由此推导芯片颗数）：
```
PCB 上贴了 2 颗 DDR4 芯片，每颗 16-bit 位宽 (x16)
2 × 16-bit = 32-bit 总位宽
2 颗共用 CS0 → 1 Rank
如果每颗是 4Gb Die → 总容量 = 2 × 4Gb = 8Gb = 1GB
如果每颗是 8Gb Die → 总容量 = 2 × 8Gb = 16Gb = 2GB
```

> **关于芯片颗数的推导依据：** `dram_x4x8x16_mode = X16` 确认颗粒是 x16，32-bit 位宽 ÷ 16-bit/颗 = **2 颗芯片**。配套佐证：timing.c 中的 `CARMEL_BOARD_1G_1G_ADC_ID`（"1G_1G" = 两颗芯片各 1GB = 2GB），说明这颗板的多配置方案正是围绕 2 颗 x16 芯片设计的。**如果你手里的板子硬件与此不同（例如贴了 4 颗 x8），以下所有换算按"32-bit ÷ 颗粒位宽 = 颗数"重新推导即可，结论不变。**

**每颗芯片从 4Gb Die 换成 8Gb Die（同类型、同封装），PCB 走线完全不变，BL2 自动训练适配，软件零改动。这就是 1G→2G 不需要改代码的底层原因。**

---

## 三、DDR 适配真正需要关注的参数

以 Amlogic A113X2 平台为例，DDR 配置的核心数据文件是 **`acs.bin`**（又称 `device_acs.bin`），它位于：

```
bootloader/uboot-repo/fip/a5/bin/acs.bin  （编译时从 BL33 board 目录拷贝）
bootloader/uboot-repo/bl33/v2019/board/amlogic/a5_av409/firmware/acs.bin
```

### 3.1 设备 ACS 参数（Device ACS）— 必须关注的

这些参数是 `acs.bin` 的核心内容，对应源码中的 `ddr_set` 结构体：

| 参数 | 含义 | 1G→2G 是否要改 |
|------|------|---------------|
| `DramType` | DDR 类型：0=DDR3, 1=DDR4, 2=LPDDR4, 3=LPDDR3 | **不变**（都是 DDR4） |
| `dram_rank_config` | Rank 连接方式：见下方详细说明 | **不变**（都是 1 Rank） |
| `DisabledDbyte` | 禁用某些 Byte Lane（16/32-bit 选择） | **不变**（都是 32-bit） |
| `Is2Ttiming` | 1T / 2T 命令速率 | **不变** |
| `soc_data_drv_ohm_ps1` | SoC 端驱动强度（Ω） | 通常不变，但若颗粒要求不同则需改 |
| `dram_data_drv_ohm_ps1` | DRAM 端驱动强度（Ω） | **查新颗粒 Datasheet** |
| `soc_data_odt_ohm_ps1` | SoC 端 ODT 阻值 | 通常不变 |
| `dram_data_odt_ohm_ps1` | DRAM 端 ODT 阻值 | **查新颗粒 Datasheet** |

### 3.2 dram_rank_config 详解

这是最关键的一个参数，从 Amlogic 源码中直接摘录：

```
// #define CONFIG_DDR0_16BIT_CH0                 0x1   // 16-bit 位宽，只用 CS0
// #define CONFIG_DDR0_16BIT_RANK01_CH0          0x4   // 16-bit 位宽，用 CS0 + CS1
// #define CONFIG_DDR0_32BIT_RANK0_CH0           0x2   // 32-bit 位宽，只用 CS0    ← AV409 默认
// #define CONFIG_DDR0_32BIT_RANK01_CH01         0x3   // LPDDR4 专用：32-bit，CH A CS0+1, CH B CS0+1
// #define CONFIG_DDR0_32BIT_16BIT_RANK0_CH0     0x5   // 32-bit，CS0 32-bit 但高位地址 16-bit
// #define CONFIG_DDR0_32BIT_16BIT_RANK01_CH0    0x6   // 32-bit，CS0 32-bit, CS1 16-bit
// #define CONFIG_DDR0_32BIT_RANK01_CH0          0x7   // 32-bit 位宽，用 CS0 + CS1  ← 双 Rank
```

**当 1G→2G 是单 Rank 变双 Rank 时**，`dram_rank_config` 要从 `0x2` 改为 `0x7`，并且 BL2 要使用 `ddr4_2d.fw` 而非 `ddr4_1d.fw`。这在 build.sh 中是通过 `DDRFW_TYPE` 控制的：

```bash
# build.sh 中的关键逻辑
if [ "$ddr_type" == "ddr4" ]; then
    dd if=${INPUT_DDRFW}/ddr4_1d.fw of=${payload}/ddrfw_1d.bin skip=96 bs=1 count=36864
    dd if=${INPUT_DDRFW}/ddr4_2d.fw of=${payload}/ddrfw_2d.bin skip=96 bs=1 count=36864
fi
```

`ddrfw_1d.bin` 作为主固件打包进 BL2，`ddrfw_2d.bin` 作为备用打包进 ddr-fip.bin。BL2 先用 1d 固件初始化，如果检测到双 Rank 配置，会从 ddr-fip 中加载 2d 固件。

### 3.3 时序参数（Timing Parameters）— 通常通过 ACS 工具生成

ACS 工具（Amlogic 的 DDR 参数生成工具）会根据 DDR 颗粒的 Datasheet 自动计算出最优时序，写入 `acs.bin`。软件开发通常**不需要手动计算时序**，但需要理解这些参数的含义：

| 参数 | 含义 | 典型 DDR4-3200 值 |
|------|------|-------------------|
| tCK | 时钟周期 | 0.625 ns (1600MHz) |
| CL (CAS Latency) | 列地址选通延迟 | 22 个时钟周期 |
| tRCD | RAS 到 CAS 延迟 | 22 个时钟周期 |
| tRP | 行预充电时间 | 22 个时钟周期 |
| tRAS | 行激活时间 | 52 个时钟周期 |
| tRC | 行周期时间 | tRAS + tRP |
| tRFC | 刷新周期时间 | 350 ns（容量越大 tRFC 越长） |
| tFAW | 四窗口激活时间 | 限制短时间内的 Row 激活次数 |

**注意：tRFC 与容量正相关。大容量 DDR（如单颗 4Gb+）的 tRFC 会显著大于小容量颗粒。这是 1G→2G 时 BL2 自动适配的关键参数之一。**

### 3.4 容量升级的各种组合场景

下面总结实际工程中常见的场景及其软件改动需求：

```
场景 A：单 Rank 同类型，密度翻倍
  1G (1×4Gb die × 2pcs, CS0) → 2G (1×8Gb die × 2pcs, CS0)
  ✅ 软件不改。BL2 自动识别。

场景 B：双 Die 封装替换单 Die，仍单 Rank
  1G (1×4Gb die × 2pcs, CS0) → 2G (2×4Gb die × 2pcs, CS0)
  ✅ 软件不改。虽然封装变了（DDP），但对外仍是 1 Rank，BL2 自动适应。

场景 C：单 Rank 变双 Rank
  1G (1 Rank × 2pcs) → 2G (2 Rank × 2pcs+2pcs)
  ❌ 需要改！dram_rank_config 从 0x2→0x7，DDRFW_TYPE 指定 2d 为默认

场景 D：DDR 类型变化
  DDR3 1G → DDR4 2G
  ❌ 需要改！DDRFW_TYPE 从 ddr3→ddr4，DramType 从 0→1，
     PMIC 电压 1.5V→1.2V，所有时序重新计算

场景 E：1 Rank → 2 Rank，其中一个 Rank 是 16-bit
  32-bit 1R → 32-bit CS0 + 16-bit CS1
  ❌ 需要改！dram_rank_config 改为 0x6

场景 F：16-bit 变 32-bit（换颗粒拓扑）
  ❌ 需要大改！DisabledDbyte 要改，PCB 走线全变，
     原则上等于重新做 DDR 设计
```

---

## 四、基于 A113X2 平台的实操指南

### 4.1 确认当前 DDR 配置

```bash
# 1. 确认总容量
cat /proc/meminfo | head -1

# 2. 确认 DDR 频率
cat /sys/class/aml_ddr/freq

# 3. 确认 Memory 地址空间
cat /proc/iomem | grep "System RAM"

# 4. 确认 Device Tree 中内存大小
od -An -tx4 /proc/device-tree/memory@00000000/reg

# 5. 确认 DDR 带宽使用
cat /sys/class/aml_ddr/bandwidth
cat /sys/class/aml_ddr/usage_stat
```

### 4.2 升级 DDR 前的 Checklist

在做 DDR 升级/更换之前，与硬件工程师确认以下信息：

1. **新 DDR 颗粒的完整型号和 Datasheet**
   - Die 容量（单个 Die 多大）
   - 封装类型（单 Die / DDP / QDP）
   - Rank 数量（1R 还是 2R？CS0 和 CS1 如何连接？）
   - 数据位宽（x8 / x16？）

2. **PCB 上的连接拓扑**
   - 几颗芯片？接几个 CS？（本板：2 颗 x16，1 个 CS0）
   - 总位宽多少？（每颗 x16 × 2颗 = 32-bit，或每颗 x8 × 4颗 = 32-bit）
   - 原理图上的 CS/CKE/ODT 信号连接

3. **电源是否兼容**
   - 新颗粒的 VDD/VDDQ 是否和老的一致？
   - 峰值电流是否超出 PMIC 的能力？

4. **是否需要改 acs.bin**
   - 用 Amlogic 的 ACS 工具根据新颗粒 Datasheet 重新生成
   - 重点关注 `dram_rank_config`、驱动强度、ODT 值

### 4.3 如果在源码中需要改动，改哪里？

```
# 1. ACS 参数（DDR 训练和时序参数）
bootloader/uboot-repo/bl33/v2019/board/amlogic/<board>/firmware/acs.bin

# 2. DDR 固件选择（1d vs 2d）
#    在 build 配置中设置 CONFIG_DDRFW_TYPE
#    或在 build.sh 调用时传入 DDRFW_TYPE 环境变量

# 3. BL2 编译时 DDR FW 打包
bootloader/uboot-repo/fip/a5/build.sh
    → DDRFW_TYPE 决定主固件用哪个 fw

# 4. Device Tree 内存节点（通常不需要手动改，U-Boot 动态填入）
#    但如果 kernel 不通过 U-Boot 启动（例如直接加载），
#    则需要确保 DT 中的 memory node 正确
```

### 4.4 验证升级后的稳定性

```bash
# 1. memtester — 用户空间压力测试，建议跑 24 小时
memtester 1500M 10

# 2. stress — 全系统压力
stress --vm 2 --vm-bytes 1500M --timeout 3600s

# 3. 关注 DDR 带宽监控
cat /sys/class/aml_ddr/usage_stat
# 观察是否有异常波动

# 4. 检查内核日志是否有 DDR 相关错误
dmesg | grep -iE "dmc|ddr|edac|noc.*error|slave.*error"

# 5. 反复休眠唤醒测试（验证 Self-Refresh）
#    执行数百次 suspend/resume
```

---

## 五、总结

### 5.1 核心概念一句话

- **Die**：未封装的存储裸片，决定颗粒的基本密度
- **封装 (Package)**：一颗成品 DDR 芯片，内部可能有 1 个或多个 Die
- **Rank**：由 CS 信号独立选中的一组芯片，软件视角的操作单位
- **Channel**：DDR 控制器的独立物理接口（含完整 DQ/CA 总线）
- **1d / 2d**：Amlogic 术语，1d = 单 Rank（单 Die 或共 CS 的 DDP），2d = 双 Rank

### 5.2 升级时软件要不要改？一张表判断

| 变化 | 改代码？ |
|------|---------|
| 同类型、同 Rank、同位宽，仅密度变大 | **不改** |
| 单 Rank → 双 Rank | **改** `dram_rank_config` 和 DDR FW |
| DDR3 → DDR4 | **改** 类型 + 电压 + 时序 |
| 16-bit ↔ 32-bit | **改** `DisabledDbyte` |
| 仅更换颗粒品牌（规格不变） | **通常不改**，但要验证时序是否兼容 |

### 5.3 关键经验

> **DDR 适配的软件工作量，取决于"配置"变没变，而不是"容量"变没变。**
>
> 密度变大但 Rank/类型/位宽不变 → BL2 固件自动搞定。
> Rank 变了、类型变了、位宽变了 → 必须人工修改 ACS 参数和选择正确的 DDR FW。

在我们这个 A113X2 平台上，1G→2G 的升级之所以零代码改动，正是因为只换了更大密度的同类型颗粒，整个 DDR 子系统的物理拓扑（32-bit / 1 Rank / DDR4 / CS0 only）完全没变。BL2 在上电训练时自动完成了从 Mode Register 读密度、算地址窗口、训时序眼图的全部工作。

**但千万别以为"从 1G 升到 2G 永远不用改软件"。如果 2G 是通过加一颗 CS1 的颗粒变成双 Rank 实现的，那就必须改了。**

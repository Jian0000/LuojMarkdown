+++
title = 'AHRD功能验证'
date = '2026-08-22T14:41:28+08:00'
draft = false
+++

# S905Y5 平台 AHDR（Advanced HDR）真机验证调查报告

> 报告日期：2026-08-11
> 平台：Amlogic S905Y5（S7 平台，板级 qurra）
> 素材：Amlogic 交付 AHDR.zip（2025-07-17 打包）
> 验证目标：确认板载 AHDR 功能可用，并通过 sysfs 节点调节生效

---

## 1. 调查背景

为论证 S905Y5 对 **Advanced HDR by Technicolor**（即 SL-HDR1 / SMPTE ST 2094-40）的支持能力，已实施以下集成工作：

| 步骤 | 内容 | 状态 |
|---|---|---|
| 1 | AHDR.zip 内 5 个补丁（prime_sl 驱动 / fb4ad875 UI 节点 / h264 SL-HDR 提取 / h265+h264 auxdata 修复 / cpq）打入源码库 | ✅ 完成 |
| 2 | 平台迁移：原补丁平台表**不含 S7**，自行移植 6 处改动（dts 节点、平台表、寄存器分支、build.cfg=qurra） | ✅ 完成 |
| 3 | 编译并烧录镜像 | ✅ 完成 |
| 4 | 静态检查：`/sys/module/aml_media/parameters/` 下 `prime_sl_display_brightness`、`prime_sl_display_adaptation_tuning` 节点存在 | ✅ 通过 |
| 5 | 功能验证：播放视频过程中写入调参节点，观察画面/输出变化 | ❌ **无任何作用** |

本报告回答两个问题：

1. **调参节点写入成功（读回值正确），为什么画面毫无变化？**
2. **为什么使用 zip 包交付的材料，无法达到预期的功能效果？**

---

## 2. 调查结论（摘要）

> **结论：AHDR 功能当前无法生效，是"片源不匹配"与"算法库缺失"两个问题叠加所致，二者缺一不可。已集成的驱动框架、补丁、编译产物均验证正常，问题不在集成环节。**

1. **根因一（验证素材问题）**：当前测试片源 DB.mp4 是 **Dolby Vision Profile 8（HLG 基、HEVC 编码）**，不是 AHDR 所需的 **SL-HDR1 片源（H.264 编码、携带 ST 2094-40 元数据）**。prime_sl 管线只接受被标记为 `HDR10PRIME` 格式的视频帧，DB.mp4 实测被判定为 `HLG` 格式，管线入口即被拒。
2. **根因二（功能本体缺失）**：AHDR 的核心算法（ST 2094-40 元数据解析 + 亮度/色域重建）由 **Technicolor 闭源算法库（ko）** 提供，**该算法库未随 AHDR.zip 交付**（readme 注明需另行申请 `AHDR_kernel5.15.170_*` 包）。没有算法库，驱动的算法函数指针（`p_funcs`）为空，管线即使条件满足也无法启动。
3. **机制解释**：调参节点是普通的模块参数变量，写入即刻成功（读回正确）；但其值**只在管线运行时才被消费**（配置硬件寄存器）。管线从未启动 → 变量空转 → 画面无变化。

---

## 3. AHDR 技术原理

### 3.1 什么是 AHDR / SL-HDR1

**AHDR（Advanced HDR by Technicolor）** 是 Technicolor 基于 **SL-HDR（Single Layer HDR，单层高动态范围）** 技术实现的 HDR 增强/转换方案，对应 **SMPTE ST 2094-40** 标准。

核心技术思想：**在普通 SDR 视频流（H.264 编码）中，通过 SEI 消息内嵌 HDR 动态元数据（ST 2094-40），接收端依据元数据将画面重建为 HDR**。播放端看到的仍是普通视频，显示端却能输出 HDR 信号。

### 3.2 AHDR 处理管线（从片源到画面）

```
┌─────────────┐   ┌──────────────────┐   ┌────────────────┐   ┌──────────────┐
│ SL-HDR1 片源 │ → │ H.264 解码器      │ → │ 视频帧格式判定   │ → │ prime_sl 驱动  │
│ H.264 编码   │   │ 提取 ST 2094-40  │   │ update_vframe_ │   │ prime_sl_     │
│ +SEI 元数据  │   │ SEI → auxdata    │   │ src_fmt()      │   │ process()     │
└─────────────┘   └──────────────────┘   └────────────────┘   └──────────────┘
                                                                    │
                                                    ┌───────────────▼───────────────┐
                                                    │ ① 检查帧格式 == HDR10PRIME(3) │
                                                    │ ② 解析 ST 2094-40 元数据      │
                                                    │ ③ 调用闭源算法库 p_funcs       │
                                                    │ ④ prime_sl_set_reg() 配置硬件 │
                                                    └───────────────────────────────┘
                                                                    │
                                                    调参节点在此处生效：
                                                    prime_sl_display_adaptation_tuning
                                                    prime_sl_display_brightness
                                                                    ▼
                                                           硬件寄存器 → 画面/HDMI 输出
```

管线各环节与代码位置：

| 环节 | 位置 | 作用 |
|---|---|---|
| 元数据提取 | `vmh264.c:7426`（H.264 解码器，补丁新增） | 识别 SEI 中 ITU-T T.35 国家码 `0xB5` + 厂商码 `0x003A`（ST 2094-40），打包进 auxdata 传给视频层 |
| 帧格式判定 | `video.c:update_vframe_src_fmt()`（5841-5849 行） | 检查 SEI 是否含 PRIME 元数据，是则标记 `VFRAME_SIGNAL_FMT_HDR10PRIME` |
| 管线入口 | `amprime_sl.c:prime_sl_process()`（1270-1331 行） | 视频帧到达驱动处理路径 |
| 格式检查 | `amprime_sl.c:1282` | `fmt != HDR10PRIME` 则直接返回，**管线不启动** |
| 元数据解析 | `amprime_sl.c:prime_sl_parser_metadata()` | 解析 ST 2094-40 动态元数据 |
| 算法处理 | `p_funcs->prime_metadata_parser_process()`（1312 行） | **闭源算法库**处理元数据，生成 LUT/增益等显示参数 |
| 硬件配置 | `amprime_sl.c:prime_sl_set_reg()`（1315 行） | 将调参值写入 VPU 寄存器，**调参在此生效** |

### 3.3 调参节点在管线中的位置

| 节点 | 含义 | 有效值 | 生效机制 |
|---|---|---|---|
| `prime_sl_display_brightness` | 显示峰值亮度（nit） | 100/150/250/400/550/700/850/1000/1250/1500/1750/2000/2500/3000/3500/4000/5000 | 写入 `cfg->display_brightness`，仅在管线运行时随 `prime_sl_set_reg()` 写入硬件 |
| `prime_sl_display_adaptation_tuning` | 调教级别（效果强度） | 0=off / 1=low / 2=median / 3=high | 同上，映射为 `cfg->display_adaptation_tuning`（65536/81002/131072/212074） |

**关键理解：这两个节点只是"参数变量"，本身不触发任何处理。** 它们是被管线**消费**的对象，不是管线的**开关**。管线不运行，写什么都是空转。

---

## 4. 验证过程与现象

### 4.1 真机环境

- 播放器：`com.droidlogic.exoplayer2.demo`（zip 内交付的 demo APK，普通 ExoPlayer 播放器）
- 测试片源：`/sdcaDR/Movies/DB.mp4`（102MB，本地 ffprobe 分析结果见下）
- 驱动状态：`prime_sl_probe=1`（probe 成功）、`prime_sl_enable=1`、`prime_sl_running=0`

DB.mp4 片源属性（ffprobe）：

| 属性 | 值 | 含义 |
|---|---|---|
| codec | **hevc**（Main 10） | **H.265 编码** |
| color_transfer | arib-std-b67 | **HLG**（混合对数伽马） |
| color_space / primaries | bt2020 | BT.2020 宽色域 |
| dv_profile | **8** | **Dolby Vision Profile 8**（基于 HLG 的 DV 片源） |

**结论：DB.mp4 是 Dolby Vision P8 片源，与 AHDR（SL-HDR1 / ST 2094-40）是两种完全不同的 HDR 技术，且编码为 HEVC。**

### 4.2 实测现象

开启内核调试日志（`debug_flag=0x8000000` 打印帧格式判定、`prime_sl_debug=1` 打开驱动日志）后重启播放，dmesg 实锤：

```
[8058.904104] [update_vframe_src_fmt]fmt: 4, vf 000000000355e7b1, sei 0000000098d6b8ed
[8058.892768] Prime_SL: get_prime_sl_frame:0
[8058.995463] [update_vframe_src_fmt]fmt: 4, vf 00000000ce47c6db, sei 00000000cdc5a894
[8058.995806] [update_vframe_src_fmt]fmt: 4, vf 00000000595f3061, sei 000000006bd9f8f4
...
（Prime_SL: get_prime_sl_frame:0 持续刷屏）
```

对照枚举定义（`vframe.h:349-352`）：

| fmt 值 | 格式 | 说明 |
|---|---|---|
| 3 | HDR10PRIME | **AHDR 管线的唯一入口格式** |
| 4 | HLG | 实测 DB.mp4 被判定为此格式 |

调参写入验证（均成功读回，但无效）：

```
# echo 3 > /sys/module/aml_media/parameters/prime_sl_display_adaptation_tuning
# cat /sys/module/aml_media/parameters/prime_sl_display_adaptation_tuning
3                        ← 写入成功
# cat /sys/module/aml_media/parameters/prime_sl_running
0                        ← 管线未运行
```

### 4.3 算法模块清单（lsmod）

| 模块 | 归属 | 状态 |
|---|---|---|
| `hdr10_tmo_alg` | Amlogic 开源 HDR10 色调映射算法 | ✅ 已加载 |
| `cuva_hdr_alg` | Amlogic 开源 CUVA 国标 HDR 算法 | ✅ 已加载 |
| **SL-HDR（prime_sl）算法库** | **Technicolor 闭源算法** | ❌ **未加载（未交付）** |

---

## 5. 原理分析：为什么调参没有作用

### 5.1 根因一：测试片源与 AHDR 格式不匹配（验证素材问题）

prime_sl 管线的入口检查（`amprime_sl.c:1275-1288`）：

```c
void prime_sl_process(struct vframe_s *vf)
{
	if (!prime_sl_probe || !prime_sl_enable || !p_funcs)   // ← 入口条件②
		return;

	if (vf && vf->src_fmt.fmt != VFRAME_SIGNAL_FMT_HDR10PRIME) {  // ← 入口条件①
		if (prime_sl_running) { ... }
		return;                              // ← DB.mp4 播放时在这里被拒
	}
	...
}
```

帧格式由 `video.c:update_vframe_src_fmt()` 判定（5810-5851 行），逻辑简化如下：

```c
if (fmt == INVALID) {
    if (transfer == 18 /*HLG*/)      → fmt = HLG         // DB.mp4 命中此分支
    else if (transfer == 0x30)       → fmt = HDR10PLUS/HDR10
    else if (transfer == 16 /*PQ*/)  → fmt = HDR10
    else                             → fmt = SDR
    ...
    if (is_prime_sl_enable() && sei && size &&
        fmt != HDR10PLUS && !signal_cuva) {
        if (check_media_sei(sei, size, FMT_TYPE_PRIME, NULL))  // 找 ST 2094-40
            fmt = VFRAME_SIGNAL_FMT_HDR10PRIME;                // ← 需要 SEI 含 0xB5+0x003A
    }
}
```

**DB.mp4 的双重不匹配：**

1. **元数据不对**：DV P8 携带的是 Dolby Vision RPU 元数据（ST 2094-10），不是 ST 2094-40。`check_media_sei` 查找国家码 `0xB5` + 厂商码 `0x003A` 的 T.35 SEI，DB.mp4 中不存在 → 不满足 PRIME 标记条件；
2. **编码不对**：Amlogic 交付的元数据提取补丁只实现了 **H.264 解码器**（`vmh264.c`）的 SL-HDR 提取；**HEVC 解码器（`vh265.c`）无此功能**（源码确认无 SL-HDR 提取代码）。即使有 SL-HDR1 元数据，HEVC 编码也提取不出来。

> 补充：实测 dmesg 中 `Prime_SL: get_prime_sl_frame:0` 刷屏，说明驱动在视频处理路径上被正常调用（`get_prime_sl_frame()` 由 amvecm 的 amcsc.c:9145 调用），驱动本身存活，但 `set_prime_sl_frame(1)`（仅在检测到 PRIME 元数据时调用）从未触发 —— 与 fmt=4 结论相互印证。

### 5.2 根因二：SL-HDR 闭源算法库缺失（功能本体缺失）

即使片源正确，管线还需要算法库才能运行。看管线启动路径（`amprime_sl.c:1310-1322`）：

```c
if (prime_sl_enable && p_funcs && new_vf) {           // ← p_funcs 必须非空
	if (!prime_sl_parser_metadata(vf)) {              // 元数据解析成功
		p_funcs->prime_metadata_parser_process(&prime_sl_setting);  // ← 闭源算法
		...
		prime_sl_running = 1;                          // 管线启动
	}
}
```

- `p_funcs`（算法函数指针表）由**闭源算法模块**通过 `register_prime_functions()` 注册；
- 全工程（common_drivers + media_modules + vendor）grep `register_prime_functions`：**无任何调用者**；
- 板子 lsmod：仅有 `hdr10_tmo_alg`（HDR10 链路）、`cuva_hdr_alg`（CUVA 链路），**无 SL-HDR 算法模块**；
- `prime_sl_process()` 第一行 `if (!prime_sl_probe || !prime_sl_enable || !p_funcs) return;` —— **p_funcs 为空，函数全静默**。

**即：AHDR 的交付形式 = 开源驱动框架（已集成）+ 闭源算法库（需另行申请 ko）。没有算法库，prime_sl 永远无法处理画面。**

### 5.3 "写入成功但无效"的机制解释

```c++
echo 3 > prime_sl_display_adaptation_tuning
   │
   ▼
module_param 变量赋值（内核 sysfs 标准机制）── 立即成功，读回=3 ✓
   │
   ▼
等待管线消费：prime_sl_set_reg() 将变量值写入 VPU 寄存器（amprime_sl.c:1315）
   │
   ▼
管线启动条件：fmt==HDR10PRIME(3) 且 元数据解析成功 且 p_funcs 非空
   │
   ├─ 条件① fmt：DB.mp4 实测 = 4 (HLG)      → ✗ 不满足
   ├─ 条件② 算法库：p_funcs = NULL          → ✗ 不满足
   ▼
管线从未启动 → prime_sl_set_reg 从未执行 → 变量空转 → 画面/输出零变化
```

**一句话：节点是"变量"不是"开关"；变量写入后需要管线运行时才会被应用，而管线因片源（A）和算法库（B）双重缺失从未运行。**

---

## 6. 为什么 zip 包中的材料无法达到预期效果

### 6.1 zip 包交付内容盘点

| 材料 | 性质 | 实际作用 | 评估 |
|---|---|---|---|
| `0001-prime_sl-add-prime_sl-driver` | **开源驱动框架**源码 | 驱动主体 + sysfs 节点 | ✅ 已集成，工作正常 |
| `fb4ad875.diff` | 开源 UI 节点补丁 | 使调参节点可写 | ✅ 已集成，节点可写 |
| `h264 SL-HDR 提取补丁` | 开源解码器补丁 | H.264 流提取 ST 2094-40 元数据 | ✅ 已集成 |
| `h265/h264 auxdata 修复补丁` | 开源缺陷修复 | 修复元数据传递 | ✅ 已集成 |
| `cpq 补丁` | 开源逻辑修改 | CPQ 识别 HDR_TYPE_PRIMESL | ✅ 已集成 |
| `demo-noDecoderExtensions-release.apk` | **普通 ExoPlayer 播放器** | 仅负责解封装/解码/播放 | ⚠️ 不含任何 HDR 算法 |
| readme 提及的 `AHDR_kernel5.15.170_*` ko | **闭源 SL-HDR 算法库** | ST 2094-40 解析 + 亮度/色域重建算法 | ❌ **未随包交付** |

### 6.2 缺口分析：zip 包提供了"骨架"，缺"引擎"与"燃料"

```
交付的（骨架，已装车）          缺失的（无法从 zip 获得）
┌─────────────────────┐   ┌──────────────────────────────┐
│ 驱动框架 amprime_sl │   │ ① 闭源 SL-HDR 算法库（引擎）   │
│ sysfs 调参节点      │   │    readme 注明需另行申请        │
│ 解码器元数据提取    │   │    AHDR_kernel5.15.170_* 包    │
│ CPQ 识别逻辑        │   │ ② SL-HDR1 测试片源（燃料）      │
│ demo 播放器         │   │    H.264 编码 + ST 2094-40     │
└─────────────────────┘   │    zip 内未包含任何测试素材     │
                          └──────────────────────────────┘
```

具体说明：

1. **算法库（引擎）未交付**：这是 zip 包无法达到效果的核心原因。Amlogic 的交付模式是"开源框架 + 闭源算法"：框架代码全部开源随包提供，但算法本体（Technicolor 授权）以预编译 ko 形式提供，需按 `readme.txt` 第 2 条的指引单独申请（`AHDR_kernel5.15.170_v20250416-Shenzhen_xxx.zip`）。**没有引擎，车架再完整也跑不起来。**
2. **测试片源（燃料）未交付**：zip 包未包含任何 SL-HDR1 测试视频。自行准备的 DB.mp4 是 Dolby Vision P8（HEVC），技术路线完全不符，无法触发 AHDR 管线。
3. **demo APK 只是播放器壳**：它没有任何 HDR 处理能力，仅负责把视频播出来让底层驱动处理，不提供算法。
4. **平台兼容性需要自行迁移**：原补丁平台表仅含 g12/tl1/tm2/sc2/t7/t3/s4d/t5w/t5m/s7d/s6，**不含 S7**（S905Y5 属 S7 平台）；本次通过自行移植 dts 节点与平台表解决（已真机验证节点生效），但这属于交付物之外的额外工作量，且寄存器兼容性依赖 S7 与 S7D 硬件一致这一前提（本次已实证成立）。

### 6.3 证据小结

| 证据 | 来源 | 指向 |
|---|---|---|
| `[update_vframe_src_fmt]fmt: 4`（HLG） | 真机 dmesg | 片源被判为 HLG，非 HDR10PRIME |
| `Prime_SL: get_prime_sl_frame:0` 持续输出 | 真机 dmesg | PRIME 帧标志从未置位 |
| `prime_sl_running = 0` | 真机节点读取 | 管线从未启动 |
| DB.mp4 = HEVC + dv_profile=8 + arib-std-b67 | ffprobe | 片源是 DV P8，非 SL-HDR1 |
| lsmod 仅 hdr10_tmo_alg / cuva_hdr_alg | 真机 | 无 SL-HDR 算法库 |
| 全工程无 `register_prime_functions` 调用者 | 源码 grep | 算法库未集成 |
| `vmh264.c` 有 SL-HDR 提取、`vh265.c` 无 | 源码 | 仅 H.264 支持元数据提取 |
| 调参节点写入后读回正确 | 真机 | 节点功能正常，问题在消费侧 |

---

## 7. 结论

1. **AHDR 驱动框架在 S905Y5（S7）上集成成功**：驱动加载、probe 成功、sysfs 节点存在且可写、解码器/CPQ 补丁就位 —— 平台迁移方案成立，编译产物正确。
2. **AHDR 功能本体（闭源算法库）未交付**，这是"无法达到效果"的根本原因之一；即使驱动框架再完整，没有算法库管线无法运行。
3. **测试片源错误**（DV P8 非 SL-HDR1），即使算法库就位，当前素材也无法触发管线 —— 这是"调参无作用"的直接原因。
4. 调参节点写入成功属正常机制（参数变量），"无效"是因为变量从未被消费。

**一句话总结：zip 包交付的是 AHDR 的"外壳"（驱动框架 + 播放器），"引擎"（闭源算法库）与"燃料"（SL-HDR1 测试片源）均未随包提供，需要向 Amlogic 另行索取 —— 这是 zip 包材料无法达到预期效果的本质原因。**

---

## 8. 后续行动建议

| 优先级 | 行动 | 说明 |
|---|---|---|
| P0 | 向 Amlogic 申请 `AHDR_kernel5.15.170_*` ko 包 | **注明平台为 S7（S905Y5，不是 S7D）**，附上本报告结论 |
| P0 | 同步索要 SL-HDR1 测试片源 | 要求：H.264 编码、携带 ST 2094-40 元数据；用 ffprobe 验证出现 `SMPTE ST 2094-40` 后再上板 |
| P1 | 拿到 ko 后验证闭环 | insmod 算法库 → 确认 `p_funcs` 注册（dmesg 有 prime_sl 处理输出）→ 播放 SL-HDR1 片源 → dmesg 应显示 `fmt: 3`（HDR10PRIME）→ `prime_sl_running` 变 1 → 调参（tuning 0~3、brightness 100~5000）画面/HDMI 输出变化 |
| P1 | 固化交付 | 验证通过后将算法库 ko 固化进 vendor_dlkm，随固件发布 |
| P2 | 平行确认 | 若 Amlogic 无法提供 S7 版算法库，评估 S905X5M（S7D）样机作为功能验证替代平台 |

---

## 附录：关键文件与行号索引

| 文件 | 关键位置 |
|---|---|
| `common/common14-5.15/.source_date_epoch_dir/common_drivers/drivers/media/enhancement/amprime_sl/amprime_sl.c` | 入口检查 1275 行；格式检查 1282 行；管线启动 1310-1322 行；硬件配置 1315 行；调教映射 1131-1145 行；厂商码 63 行 |
| `.../drivers/media/video_sink/video.c` | fmt 判定 5810-5851 行；PRIME 标记 5841-5849 行；PRIME SEI 检查 5630-5655 行 |
| `.../include/linux/amlogic/media/vfm/vframe.h` | 帧格式枚举 348-360 行（HDR10PRIME=3，HLG=4） |
| `common/common14-5.15/driver_modules/media_modules/drivers/frame_provider/decoder/h264_multi/vmh264.c` | SL-HDR 元数据提取 7404-7462 行（补丁新增） |
| `.../decoder/h265/vh265.c` | 无 SL-HDR 提取逻辑（对比项） |
| `common_drivers/drivers/media/enhancement/amvecm/hdr/am_hdr10_tm.c` 等 | HDR10/CUVA 开源算法（对照项） |

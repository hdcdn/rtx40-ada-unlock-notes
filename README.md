# RTX 40 系（Ada）解锁 · 完整资料 + 逐卡实测 + 操作手册

基于 `JustAResearcher/gpu-unlock-ada`（v1.1.1）逐文件核对，配合 NVIDIA 官方规格整理：三处辟谣、支持矩阵、四张卡实测、机制原理、完整 Linux/Windows 操作手册。

**交流群**：QQ 群 **1128010665**（加群备注：GitHub）

---

基于 `JustAResearcher/gpu-unlock-ada`（v1.1.1，GPL-2.0-only）主仓库逐文件核对，配合 NVIDIA 官方规格整理。**不引用社媒二手数字**。

- 整理时间：2026-10-25
- 适用：RTX 40 系（Ada / AD10x）
- 交流群：QQ 1128010665

---

## 0. 三处先辟谣

| 社媒说法 | 仓库实际 |
|---|---|
| "从 4090 到 4060 Ti 整个 40 系全部解锁" | Linux 模块只启用 **3 个 device ID**：`10de:2684`(4090)、`10de:2705`(4070 Ti SUPER，实验)、`10de:2805`(4060 Ti 16G，实验) |
| "FP8 也翻倍"（4080S 201.8→376.1） | 4070 Ti SUPER 实测 E4M3 FP8/FP32 CUTLASS GEMM：**59.326 → 59.495 TF/s，+0.3%** |
| "指令级 2× = 应用 2×" | 4090 真实 cuBLAS GEMM：FP16 +12~34%，BF16 在 4096³ 基本不动 |

---

## 1. 支持矩阵（精确到 PCI device ID）

| 卡 | device ID | 芯片 | SM | CUDA 核心 | 显存 | 状态 |
|---|---|---|---:|---:|---|---|
| RTX 4090 | `10de:2684` | AD102 | 128 | 16,384 | 24 GB GDDR6X / 1008 GB/s | ✅ 正式支持 |
| RTX 4070 Ti SUPER | `10de:2705` | AD103 | 66 | 8,448 | 16 GB GDDR6X / 672 GB/s | ⚠️ 实验（1 张卡） |
| RTX 4060 Ti 16GB | `10de:2805` | AD106 | 34 | 4,352 | 16 GB GDDR6 / 288 GB/s | ⚠️ 实验（1 张卡，无同卡对比） |
| RTX 4070 Ti | `10de:2782` | AD104 | 60 | 7,680 | 12 GB GDDR6X / 504 GB/s | 🧪 仅 Windows 参考移植 |
| RTX 4080 | — | AD103 | 76 | 9,728 | 16 GB / 716.8 GB/s | ❌ 未启用 |
| RTX 4080 SUPER | — | AD103 | 80 | 10,240 | 16 GB / 736 GB/s | ❌ 未启用（PR #24 讨论中） |
| RTX 4070 | — | AD104 | 46 | 5,888 | 12 GB / 504 GB/s | ❌ 未启用 |
| RTX 4060 Ti 8GB | — | AD106 | 34 | 4,352 | 8 GB / 288 GB/s | ❌ 明确不支持 |

> 显存/带宽/CUDA 核心数为 NVIDIA 官方标称；SM 数来自仓库实测报告（2705=66、2805=34、2782=60、2684=128）。

**关键**：`gpu_unlock.targets=all` **只会选 4090 和 30 系**，永远不会选中 2705/2805——这两个必须写单个精确 PCI 地址 + 实验开关。

---

## 2. 锁了什么

SM speed-select 寄存器里的 9 个字段，编码 `0=满速 / 1=半速 / 2=四分之一`：

| 字段 | 含义 | 原厂 | 解锁后 |
|---|---|---|---|
| `FMLA32` | FP32 累加的 FP16/BF16 张量运算 | 1 | **0** |
| `IMLA2` | INT8/INT4 MMA 第 2 组 | 1 | **0** |
| `IMLA4` | INT8/INT4 MMA 第 4 组 | 1 | **0** |
| `IMLA0` `FMLA16` `FFMA` `IMLA1` `IMLA3` `DP` | 其余六项 | 0 | 0（不变） |

出厂时 FP16/BF16（FP32 累加）只有半速——这解释了为什么所有实测都是精确的 2 倍。

---

## 3. 逐卡实测

### 3.1 RTX 4090（AD102，正式支持）

驱动 615.78.08、450W 上限、同卡 A/B：

| 模式 | 原厂 | 解锁后 | 倍数 |
|---|---:|---:|---:|
| FP16 → FP32 | 165.6 TF/s | **331.6 TF/s** | 2.00× |
| BF16 → FP32 | 165.5 TF/s | **330.6 TF/s** | 2.00× |
| FP16 → FP16 | 325.7 TF/s | 306.3 TF/s | 0.94×（略降） |
| INT8 → INT32 | 661.6 TOP/s | 661.4 TOP/s | 不变 |
| `dp4a` | 164.5 TOP/s | 164.4 TOP/s | 不变 |

独立第三方复现（rev `8855e7b`，Arch 内核 7.2.11，615.78.08，450W）：FP16/FP32 **160.4 → 320.4**、BF16/FP32 **159.7 → 320.2**。

真实 cuBLAS GEMM：FP16 **+12~34%**（随 shape），BF16 在 4096³ **基本不变**。

### 3.2 RTX 4070 Ti SUPER（AD103，实验）

Ubuntu 5.15.0-177、驱动 580.178.04、240W 上限：

| 模式 | 原厂 | 解锁后 | 倍数 |
|---|---:|---:|---:|
| FP16 → FP32 | 95.0 TF/s | **189.9 TF/s** | 2.00× |
| BF16 → FP32 | 95.0 TF/s | **189.7 TF/s** | 2.00× |
| TF32 → FP32 | 47.49 TF/s | **94.51 TF/s** | 1.99× |
| 无符号 INT4 → INT32 | ~754.8 TOP/s | ~754.8 TOP/s | 不变 |
| INT8 → INT32 | 379.5 TOP/s | 379.5 TOP/s | 不变 |
| **E4M3 FP8 → FP32（CUTLASS 4096³）** | 59.326 TF/s | **59.495 TF/s** | **1.003×** |

独立验证：CUTLASS Profiler 4.8.1 实测 FP16/FP32 **177,543 GFLOP/s**、BF16/FP32 **176,321 GFLOP/s**（4096³，240W，无同卡原厂对照）。

可靠性：冷启动 2 次成功、普通热重启 2 次成功。

### 3.3 RTX 4060 Ti 16GB（AD106，实验）

Ubuntu 26.04.1 / 内核 7.0.0-38 / 驱动 595.99.02：

- 所有前置校验通过，`FMLA32/IMLA2/IMLA4` 由 1 变 0，FLR 后保持，日志 `gpu_unlock: unlocked`
- 驱动正常加载、CUDA 正常、无 NVRM Xid
- **但仓库没有做同卡原厂吞吐对比** → 网传"提升 38–39%"在仓库里**查不到出处**

### 3.4 RTX 4070 Ti（AD104，仅 Windows）

MSI Z490 / i9-10850K / Windows 26200.9168 / 驱动 616.56 / 285W：

| cuBLAS 4096² GEMM | 原厂 | 解锁后 | 变化 |
|---|---:|---:|---:|
| FP32 pedantic | 30.23 | 30.53 | +1.0% |
| **FP16/FP32** | 84.08 | **151.43** | **+80.1%** |
| **BF16/FP32** | 84.03 | **148.25** | **+76.4%** |

三次独立进程取中位，CPU FP64 全量校验通过，输出哈希与原厂一致。仅一块主板、一次手动启动，不构成通用支持。

---

## 4. 机制：为什么不用解密固件

1. 板载 ROM 的签名固件 `FWSEC-FRTS` 先在 GSP Falcon 上建立固件保留区
2. 把**原厂签名的 AD102 booter** 加载到 SEC2，但喂一份**构造过的 V67 签名数据**
   - SEC2 自己验签、解密——booter 是真的，验签通过
   - 签名数据里精心排布的字节**覆盖 booter 栈上的返回地址**
   - 执行流跳进 booter 内部**已认证的指令**（gadget），用固件自身特权完成受保护写入
3. 写入四个权限：`WPR_CFG`、`FBPA`、`WPR`、`FEAT`，请求 mailbox 标记 `0x67b00001` 并 HALT
4. 主机校验标记、SEC2 已停、PCI/BAR 可访问、权限回读符合
5. 主机写选择器：`SS0=0x88888888`、`SS1=0x8`
6. 触发 PCIe FLR，确认选择器**复位后仍保持**
7. 原厂驱动正常绑定

payload：64,000 字节，SHA-256 `44c36b2d6b3424e53e18f3e55db85c833655de071ce66ce42749588a4a7dc8cb`（来自 `TOTORONG/AD103_win_unlocker`）。

**"不刷 vBIOS"的含义**：改动只在本次上电的固件状态里，vBIOS 与驱动零改动；代价是**每次冷启动都要重跑**。

---

## 5. Linux 操作步骤

### 前提

- 受支持的卡 + 独立显卡
- 内核头文件、`dkms`、编译链、`python3`、`zstd`
- **Secure Boot 关闭**（或自行 MOK 签名）
- 能改内核启动参数

### 步骤

```bash
# 1. 找 PCI 地址与 device id
lspci -Dnn | grep -i nvidia
# 例：0000:01:00.0 ... [10de:2684]

# 2. 编译安装（本地下载 SHA 固定的 NVIDIA 610.43.03 输入并生成固件）
git clone https://github.com/JustAResearcher/gpu-unlock-ada.git
cd gpu-unlock-ada
make install          # 或 make install-arch / install-deb / install-rpm

# 3. 打印该卡对应的启动参数（只读）
python3 tools/print_boot_option.py
```

内核参数：

| 卡 | 参数 |
|---|---|
| 4090 | `gpu_unlock.targets=0000:01:00.0` |
| 4070 Ti SUPER | `gpu_unlock.targets=0000:01:00.0 gpu_unlock.ad103_experimental=1` |
| 4060 Ti 16G | `gpu_unlock.targets=0000:01:00.0 gpu_unlock.ad106_experimental=1` |

```bash
# GRUB
sudo nano /etc/default/grub      # 追加到 GRUB_CMDLINE_LINUX_DEFAULT 引号内
sudo update-grub

# 4. 断电冷启动（不是 reboot）
sudo poweroff
```

### 期望日志

```
SS0=0x1403041  SS1=0x3  WPR2 high 0x0  chip 0x196000a1
payload marker 0x67b00001 (expected 0x67b00001), SEC2 halted
WPR_CFG=0xfffff0ff  WPR=0xffffffff  FEAT=0xffffffff
selector writes read back SS0=0x88888888 SS1=0x8
after PCIe FLR: SS0=88888888 SS1=00000008
gpu_unlock: unlocked
```

### 三条硬约束

1. **第一次必须断电冷启动**，`reboot` 不算
2. 热重启不保证：部分板子 WPR2 不归零，模块会**主动拒绝**重跑
3. 手动尝试前**不要**跑 `nvidia-smi`/CUDA/NVML/`nvidia-modprobe`——它们会按名字加载驱动，卡一旦绑定就无法解锁（普通 modprobe 黑名单拦不住）

---

## 6. Windows 路线（AD104 参考移植）

- 位置：`windows/ad104-msi/`，报告 `docs/ad104-windows-latence-app.md`
- 状态：一块 MSI Z490、一次成功引导，绑定具体 ROM/FWSEC 几何
- 做法：UEFI 回调释放 TextOut / GraphicsConsole / NVIDIA GOP → 独占 PCI 声明 → 仅开 IO/MEM 解码 → 走同一套固件事务
- 风险：`Unsafe DMA` 保留到断电；USB 拓扑变化需精确分类（v0.5.4 才处理）
- **仓库不分发任何 Windows/ROM 二进制，也没有自动安装器**

---

## 7. 验证工具

| 工具 | 用途 |
|---|---|
| `tools/read_sm_rates.py` | 只读读 9 个速度字段 |
| `tools/print_boot_option.py` | 打印每张卡的启动参数 |
| `diagnostics/bench/mma_bench.cu` | 寄存器驻留 MMA 发射率（FP16/BF16/INT8/dp4a/TF32） |
| `diagnostics/bench/tensor_modes_driver_runner.py` | 驱动加载后跑 tensor modes 对比 |
| `diagnostics/bench/validate_gemm.cu` | GEMM 数值正确性校验 |
| `diagnostics/peek/gpu_peek.c` | 看 PCI BAR / 寄存器 |

---

## 8. 风险清单

1. 仅 3 个 Ada ID 启用，2705/2805 是单卡实验
2. 4060 Ti 16G 无同卡对比，8G 版完全没测
3. FP8 / INT8 / INT4 / dp4a **均无提升**
4. 指令级 ≠ 应用级（4090 真实 GEMM 只 +12~34%）
5. 热重启不保证（WPR2 可能不归零）
6. 显示交接未验证；FLR 前 framebuffer 被移除，屏幕会黑到 `nvidia-drm` 接手
7. Secure Boot 必须关，或自行 MOK 签名
8. 升级内核/驱动后需重新验证（测试用的是内核版本相关 initramfs）
9. **影响保修**；每次冷启动重跑

---

## 9. 数据来源

| 内容 | 来源 |
|---|---|
| 解锁机制、逐卡数字、限制 | `JustAResearcher/gpu-unlock-ada`（README + `docs/` 四份报告 + `evidence/` 原始输出） |
| CUDA 核心 / 显存 / 带宽 / TDP | NVIDIA 官方产品规格 |
| SM 数 | 仓库实测报告（4090 128 / 2705 66 / 2805 34 / 2782 60） |

*报告生成：2026-10-26*

---

## 附录 A · 完整操作手册（Linux）

### A.1 准备

```bash
# 依赖
sudo apt-get install -y git build-essential dkms linux-headers-$(uname -r) python3 zstd

# 关闭 Secure Boot（必须，否则模块加载失败）
mokutil --sb-state        # 若为 enabled，进 BIOS 或 mokutil --disable-validation
```

### A.2 确认卡与地址

```bash
lspci -Dnn | grep -i nvidia
# 0000:01:00.0 VGA ... [10de:2684]   ← 4090
#                     [10de:2705]   ← 4070 Ti SUPER（实验）
#                     [10de:2805]   ← 4060 Ti 16G（实验）
#                     [10de:2782]   ← 4070 Ti（仅 Windows 路线）
```

### A.3 编译安装

```bash
git clone https://github.com/JustAResearcher/gpu-unlock-ada.git
cd gpu-unlock-ada
make install                 # 自动识别 Arch / Debian / RHEL
# 或显式：make install-arch / make install-deb / make install-rpm
```

构建过程会：下载 SHA-256 固定的 NVIDIA 610.43.03 输入 → 本地生成 64 KB payload → 编出 DKMS 模块 → 装进 initramfs（位于 nvidia 之前）。

### A.4 写入内核参数

```bash
python3 tools/print_boot_option.py     # 只读，打印每张卡的参数
```

| 卡 | 内核参数 |
|---|---|
| 4090 | `gpu_unlock.targets=0000:01:00.0` |
| 4070 Ti SUPER | `gpu_unlock.targets=0000:01:00.0 gpu_unlock.ad103_experimental=1` |
| 4060 Ti 16G | `gpu_unlock.targets=0000:01:00.0 gpu_unlock.ad106_experimental=1` |

```bash
# GRUB
sudo sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT="\(.*\)"/GRUB_CMDLINE_LINUX_DEFAULT="\1 gpu_unlock.targets=0000:01:00.0"/' /etc/default/grub
sudo update-grub

# systemd-boot
sudo bootctl update
```

### A.5 冷启动（关键）

```bash
sudo poweroff        # 必须断电，不是 reboot
# 手动按电源键开机
```

### A.6 验证

```bash
journalctl -k -b | grep gpu_unlock
```

期望输出：

```
gpu_unlock: card 0000:01:00.0 (10de:2684) firmware state fresh, WPR2=0x0
gpu_unlock: payload marker 0x67b00001, SEC2 halted
gpu_unlock: permissions WPR_CFG=0xfffff0ff WPR=0xffffffff FEAT=0xffffffff
gpu_unlock: selectors SS0=0x88888888 SS1=0x8
gpu_unlock: post-FLR persistence OK
gpu_unlock: unlocked
```

```bash
# 读速率字段（不用 nvidia-smi）
python3 tools/read_sm_rates.py
# FMLA32 / IMLA2 / IMLA4 应为 0（满速）

# 跑发射率基准
cd diagnostics/bench && make && ./mma_bench
```

### A.7 回滚

```bash
# 去掉内核参数里的 gpu_unlock.* 即可；模块不加载就不生效
sudo sed -i 's/ gpu_unlock\.[^"]*//g' /etc/default/grub && sudo update-grub
sudo poweroff
```

---

## 附录 B · Windows 路线（AD104 参考移植）

B.1 备份当前 BIOS 设置（尤其是 Above 4G Decoding、Resizable BAR）
B.2 关闭 Secure Boot、关闭 Windows 快速启动
B.3 用仓库 `windows/ad104-msi/` 的 UEFI 参考流程（**绑定具体主板 ROM**，非通用）
B.4 释放 TextOut / GraphicsConsole / NVIDIA GOP → 独占 PCI → 仅 IO/MEM 解码
B.5 执行固件事务 → 重启 → 用 cuBLAS 基准验证（4096² FP16 应 +70~80%）
B.6 风险：Unsafe DMA 保留到断电；换主板/换 BIOS 版本即失效

---

*文档生成：2026-10-26　数据口径与来源见各节*

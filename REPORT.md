# RTX 40 系（Ada）解锁资料全整理

> 依据：`JustAResearcher/gpu-unlock-ada` 主仓库（v1.1.1，GPL-2.0-only），逐文件核对 README、QUICKSTART、四份技术报告与证据目录后整理。
> 整理时间：2026-10-25　仓库 star：1381

---

## 〇、先读这段（重要）

社交平台上那篇"40 系全系解锁、FP8 翻倍"的帖子，和仓库里的实测记录**有三处对不上**。先把差异摆出来，再谈怎么用：

| 推文说法 | 仓库实际情况 | 判断 |
|---|---|---|
| "从 4090 到 4060 Ti，整个 40 系全部解锁" | Linux 模块只启用 **3 个 device ID**：4090(2684)、4070 Ti SUPER(2705，实验)、4060 Ti 16G(2805，实验)；其余 Ada ID **未启用** | ⚠️ 严重夸大 |
| "连 FP8 都翻倍"（4080S 201.8→376.1、4070TiS 158.5→309.1） | 4070 Ti SUPER 上实测 E4M3 FP8/FP32 CUTLASS GEMM：**59.326 → 59.495 TF/s，只有 0.3% 差异** | ❌ 与仓库矛盾 |
| "同样不刷 vBIOS、不改驱动" | ✅ 正确。用的是签名 booter + 构造签名数据，不碰 vBIOS、不改驱动二进制 | ✅ 一致 |

**结论：FP16/BF16/TF32 的成倍提升是真的（有同卡 A/B 和逐位校验）；FP8 提升在这份仓库里没有证据；"全系支持"是宣传口径，实际只有三张卡被启用。**

---

## 一、支持型号（精确到 PCI device ID）

| 卡 | device ID | 芯片 | 状态 | 启用方式 |
|---|---|---|---|---|
| **RTX 4090** | `10de:2684` | AD102 / 128 SM | ✅ 主要支持，已有独立报告 | `gpu_unlock.targets=<PCI地址>` |
| **RTX 4070 Ti SUPER** | `10de:2705` | AD103 / 66 SM | ⚠️ 实验，仅 1 张卡验证 | `gpu_unlock.targets=<地址> gpu_unlock.ad103_experimental=1` |
| **RTX 4060 Ti 16GB** | `10de:2805` | AD106 / 34 SM | ⚠️ 实验，仅 1 张卡、1 次冷启动，**无同卡 stock 对比** | `gpu_unlock.targets=<地址> gpu_unlock.ad106_experimental=1` |
| RTX 4070 Ti（AD104） | `10de:2782` | AD104 / 60 SM | 🧪 **仅 Windows** 参考移植（MSI Z490 单板验证） | 不属于 Linux 模块 |
| 其余 40 系（4080/4080S/4070/4060 8G 等） | — | — | ❌ 未启用 | 需自行改模块，无官方验证 |

**关键细节**：`gpu_unlock.targets=all` **只会选 4090 和老的 30 系**，永远不会选中 2705/2805——这两个必须写**单个精确 PCI 地址**。多卡机器上千万别图省事用 `all`。

---

## 二、到底锁了什么

NVIDIA 在消费卡上通过 **SM speed-select** 寄存器把张量核心的"累加器速率"降半。模块读回的是 9 个字段，编码 `0=满速 / 1=半速 / 2=四分之一`：

| 字段 | 含义 | 原厂 | 解锁后 |
|---|---|---|---|
| `FMLA32` | FP32 累加的 FP16/BF16 张量运算 | 1（半速） | **0（满速）** |
| `IMLA2` | INT8/INT4 MMA 第 2 组 | 1 | **0** |
| `IMLA4` | INT8/INT4 MMA 第 4 组 | 1 | **0** |
| `IMLA0` `FMLA16` `FFMA` `IMLA1` `IMLA3` `DP` | 其余六项 | 0 | 0（不变） |

也就是说：**出厂时 FP16/BF16（FP32 累加）只有一半速率**，解锁就是把这一位打开。这解释了为什么所有实测都是"精确的 2 倍"。

---

## 三、原理：为什么不用解密固件

这是这个项目最漂亮的地方——**它没有分发任何解密后的固件，也没有在 GPU 里注入自定义代码**。

链路是这样的：

1. 板上 ROM 的签名固件 `FWSEC-FRTS` 先在 GSP Falcon 上跑起来，建立固件保留区；
2. 把**原厂签名的 AD102 booter** 加载到 SEC2，但给它喂一份**构造过的 V67 签名数据**；
   - SEC2 会自己验签、自己解密——验签是过的，因为 booter 是真的；
   - 而这份签名数据里被塞了精心排布的字节，会**覆盖 booter 栈上的返回地址**；
   - 于是执行流跳进 booter 里**本来就存在的、已认证的指令**（所谓 gadget），用固件自身的特权去完成受保护寄存器写入。
3. 写入的目标是四个权限：`WPR_CFG`、`FBPA`、`WPR`、`FEAT`，然后请求 mailbox 标记 `0x67b00001` 并 HALT；
4. 主机侧校验标记、SEC2 已停、PCI/BAR 可访问、三个权限回读符合预期；
5. 主机写速度选择器：`SS0=0x88888888`、`SS1=0x8`，再回读确认九个字段全为 0；
6. 触发 PCIe FLR（Function Level Reset），确认选择器**在复位后仍然保持**；
7. 让原厂驱动正常绑定、CUDA 正常工作。

**"不刷 vBIOS"的含义**：改动只发生在这次上电的固件状态里，vBIOS 和驱动文件一个字节都没动。代价是**每次冷启动都要重跑一遍**。

核心 payload 是 64,000 字节，SHA-256：
```
44c36b2d6b3424e53e18f3e55db85c833655de071ce66ce42749588a4a7dc8cb
```
它由 `TOTORONG/AD103_win_unlocker` 发布，本仓库把它接进了 `Zagitta/gpu-unlock` 的 Linux 模块与打包流程。**payload 是结果成立的关键，不是可选项。**

---

## 四、逐卡实测数据

### 4.1 RTX 4090（AD102，主支持卡）

同卡、同驱动（615.78.08）、同 450W 功耗上限，寄存器驻留 MMA 发射速率：

| 模式 | 原厂 | 解锁后 | 倍数 |
|---|---:|---:|---:|
| FP16 → FP32 累加 | 165.6 TF/s | **331.6 TF/s** | 2.00× |
| BF16 → FP32 累加 | 165.5 TF/s | **330.6 TF/s** | 2.00× |
| FP16 → FP16 累加 | 325.7 TF/s | 306.3 TF/s | **0.94×（略降）** |
| INT8 → INT32 累加 | 661.6 TOP/s | 661.4 TOP/s | 0.99×（不变） |
| `dp4a` | 164.5 TOP/s | 164.4 TOP/s | 不变 |

独立第三方复现（rev `8855e7b`，Arch 内核 7.2.11，同为 615.78.08、450W）：FP16/FP32 **160.4 → 320.4**、BF16/FP32 **159.7 → 320.2**。

**真实 cuBLAS GEMM 收益（不是发射率）**：FP16 提升 12–34%（随 shape 变化），BF16 在 4096³ 上**基本没变化**。也就是说：**指令级 2 倍，应用级远达不到 2 倍。**

> 注意 FP16/FP16 累加反而略降——原厂这条路本来就是满速，解锁后没有额外收益，测得轻微波动。

### 4.2 RTX 4070 Ti SUPER（AD103，实验）

单张卡、Ubuntu 5.15.0-177、驱动 580.178.04、240W 上限：

| 模式 | 原厂 | 解锁后 | 倍数 |
|---|---:|---:|---:|
| FP16 → FP32 | 95.0 TF/s | **189.9 TF/s** | 2.00× |
| BF16 → FP32 | 95.0 TF/s | **189.7 TF/s** | 2.00× |
| TF32 → FP32 | 47.49 TF/s | **94.51 TF/s** | 1.99× |
| 无符号 INT4 → INT32 | ~754.8 TOP/s | ~754.8 TOP/s | 不变 |
| INT8 → INT32 | 379.5 TOP/s | 379.5 TOP/s | 不变 |
| **E4M3 FP8 → FP32（CUTLASS GEMM 4096³）** | 59.326 TF/s | **59.495 TF/s** | **1.003×** |

独立验证：CUTLASS Profiler 4.8.1 实测 FP16/FP32 **177,543 GFLOP/s**、BF16/FP32 **176,321 GFLOP/s**（4096³，240W）——这是纯解锁后绝对值，没有同卡原厂对照。

可靠性：冷启动成功 2 次，普通热重启成功 2 次（最后一次用的是持久 EFI 默认项）。

### 4.3 RTX 4060 Ti 16GB（AD106，实验）

单张卡、Ubuntu 26.04.1 内核 7.0.0-38、驱动 595.99.02：

- 模块所有前置校验通过，`FMLA32/IMLA2/IMLA4` 由 1 变 0，FLR 后保持，日志 `gpu_unlock: unlocked`
- 驱动正常加载，CUDA 正常，无 NVRM Xid
- **但仓库明确写：没有做同卡原厂吞吐对比**，所以"提速 38–39%"这个数字在仓库里**找不到出处**

### 4.4 RTX 4070 Ti（AD104，仅 Windows 参考移植）

MSI RTX 4070 Ti（`10de:2782`/`1462:5132`）、MSI Z490、i9-10850K、Windows 26200.9168、驱动 616.56、285W：

| cuBLAS 4096² GEMM | 原厂 | 解锁后 | 变化 |
|---|---:|---:|---:|
| FP32 pedantic | 30.23 TF/s | 30.53 TF/s | +1.0% |
| **FP16/FP32** | 84.08 TF/s | **151.43 TF/s** | **+80.1%** |
| **BF16/FP32** | 84.03 TF/s | **148.25 TF/s** | **+76.4%** |

- 三次独立进程测量取中位，CPU FP64 全量校验通过，输出哈希与原厂一致
- 注意只有 +80%，不是 2 倍——Windows 这条路走的是另一套实现，且受功耗/温度影响
- 这是**一块主板、一次手动启动**的参考移植，不构成通用支持

---

## 五、Linux 操作步骤（完整版）

### 前置条件

- 一块受支持的卡，且是**独立显卡**（不是笔记本独显直连那种）
- Linux 内核头文件、`dkms`、编译工具链、`python3`、`zstd`
- **Secure Boot 必须关闭**（模块未签名；或自己用 MOK 签名）
- 能改内核启动参数（GRUB / systemd-boot / EFI 默认项）

### 步骤

```bash
# 1. 找卡的 PCI 地址和 device id
lspci -Dnn | grep -i nvidia
# 例：0000:01:00.0 VGA compatible controller [0300]: NVIDIA ... [10de:2684]

# 2. 拉源码并编译安装（会在本地下载 SHA 固定的 NVIDIA 610.43.03 输入并生成固件）
git clone https://github.com/JustAResearcher/gpu-unlock-ada.git
cd gpu-unlock-ada
make install
# 按发行版可分别用：make install-arch / install-deb / install-rpm

# 3. 打印该卡对应的启动参数（只读，安全）
python3 tools/print_boot_option.py
```

把打印出来的那一行加到内核命令行：

| 卡 | 内核参数 |
|---|---|
| 4090 | `gpu_unlock.targets=0000:01:00.0` |
| 4070 Ti SUPER | `gpu_unlock.targets=0000:01:00.0 gpu_unlock.ad103_experimental=1` |
| 4060 Ti 16G | `gpu_unlock.targets=0000:01:00.0 gpu_unlock.ad106_experimental=1` |

GRUB 用户：

```bash
sudo nano /etc/default/grub     # 追加到 GRUB_CMDLINE_LINUX_DEFAULT 的引号内
sudo update-grub
```

```bash
# 4. 断电冷启动（不是 reboot！）
sudo poweroff
# 再按电源键开机

# 5. 验证
journalctl -k -b | grep gpu_unlock
# 期望看到：gpu_unlock: unlocked
lspci -nnk -s 0000:01:00.0
```

### 冷启动 vs 热重启

- **第一次必须断电冷启动**：模块需要"固件状态是新鲜的"卡
- 热重启**不一定**能重现：在测试的 4070 Ti SUPER 上可以（WPR2 回读归零），但其他板子可能保留 WPR2，此时模块会**主动拒绝**重跑固件流程
- 模块会跳过已绑定或已下电的卡；如果板子固件留了 bus mastering，模块会清掉并校验

### 千万不要做的事

> 手动尝试前，**不要**跑 `nvidia-smi`、CUDA、NVML、`nvidia-modprobe`——它们会按名字把 NVIDIA 驱动加载起来，卡一旦被绑定就无法解锁。普通 modprobe 黑名单**拦不住**按名字显式加载。

---

## 六、Windows 路线（AD104 参考移植）

- 位置：`windows/ad104-msi/`，报告 `docs/ad104-windows-latence-app.md`
- 状态：**一块 MSI Z490 主板、一次成功引导**，绑定具体 ROM/FWSEC 几何
- 做法：UEFI 回调释放 TextOut / GraphicsConsole / NVIDIA GOP 占用 → 独占 PCI 声明 → 只开 IO/MEM 解码 → 走与 Linux 相同的固件事务
- 风险：`Unsafe DMA` 会保留到断电为止；USB 拓扑变化需要精确分类（v0.5.4 才开始处理）
- **仓库不分发任何 Windows/ROM 二进制，也没有自动安装器**

---

## 七、验证工具（仓库自带）

| 工具 | 用途 |
|---|---|
| `tools/read_sm_rates.py` | 只读方式读 9 个速度选择字段（不用 nvidia-smi） |
| `tools/print_boot_option.py` | 打印每张卡该用哪行内核参数 |
| `diagnostics/bench/mma_bench.cu` | 寄存器驻留 MMA 发射率基准（FP16/BF16/INT8/dp4a/TF32） |
| `diagnostics/bench/tensor_modes_driver_runner.py` | 驱动加载后跑 tensor modes 对比 |
| `diagnostics/bench/validate_gemm.cu` | GEMM 数值正确性校验 |
| `diagnostics/peek/gpu_peek.c` | 看 PCI BAR / 寄存器状态 |

期望的日志序列（2705 实测）：

```
SS0=0x1403041  SS1=0x3  WPR2 high 0x0  chip 0x196000a1
payload marker 0x67b00001 (expected 0x67b00001), SEC2 halted
WPR_CFG=0xfffff0ff  WPR=0xffffffff  FEAT=0xffffffff
selector writes read back SS0=0x88888888 SS1=0x8
after PCIe FLR: SS0=88888888 SS1=00000008
gpu_unlock: unlocked
```

---

## 八、限制与风险（仓库自己写的）

1. **只有 3 个 Ada ID 被启用**，且 2705/2805 是"单卡实验"，不构成对其他同芯片型号的支持承诺
2. **4060 Ti 16G 没有同卡原厂吞吐对比**，8G 版本完全没测
3. **FP8 无收益**：E4M3 FP8/FP32 GEMM 实测 +0.3%；INT8、无符号 INT4、dp4a 都不变
4. **指令级 ≠ 应用级**：4090 的 cuBLAS GEMM 只涨 12–34%（FP16）、BF16 在 4096³ 基本不动
5. **热重启不保证**：部分板子 WPR2 不归零，模块会拒绝重跑
6. **显示交接未验证**：固件 framebuffer 在 FLR 前被移除，屏幕会黑到 `nvidia-drm` 接手；多卡与显示输出不在 4090 验证范围内
7. **Secure Boot 必须关**，或自行 MOK 签名
8. **升级内核/驱动后要重新验证**：测试用的是内核版本相关的 initramfs 镜像
9. **不刷 vBIOS 的另一面**：每次冷启动都要重跑，重启策略/断电策略要配套

---

## 九、和《每千元算力》表对账

那张表里的"解锁"数字，现在可以逐条对上：

| 表中数值 | 仓库出处 | 是否一致 |
|---|---|---|
| 4090 331.6 TFLOPS | README 表：165.6 → **331.6** | ✅ |
| 4080S 202.7 TFLOPS | 推文数据，仓库**没有 4080S 报告** | ⚠️ 仓库未覆盖 |
| 4070 Ti SUPER 189.9 | 2705 报告：95.0 → **189.9** | ✅ |
| 3080 106 TFLOPS | 来自 30 系 `Zagitta/gpu-unlock`，另一条链 | ✅（不同项目） |

**4080 SUPER 是缺口**：推文说 PR #24 有支持，但主仓库当前分支没有 4080S 的验证报告。要用 4080S，先去 PR #24 看状态。

---

## 十、仓库文件索引（40 系相关）

```
README.md                        主文档：支持型号、实测、启动参数、限制
QUICKSTART.md                    手把手 5 步（推荐从这里开始）
rtx4090-register-report.md       4090 完整技术报告（机制/事务/基准/寄存器）
docs/ada-port.md                 移植状态：payload 来源、DMA 清理、工具链
docs/ada-2705-report.md          4070 Ti SUPER 单卡报告（最详细的一份）
docs/ada-2805-report.md          4060 Ti 16G 单卡报告（无同卡对比）
docs/ad104-windows-latence-app.md 4070 Ti Windows 参考移植
docs/How Ampere was unlocked.md  30 系是怎么被解开的（背景知识）
docs/evidence/rtx4090/           4090 原始证据
docs/evidence/rtx4090-pr1-matched/ 第三方复现的匹配数据
docs/evidence/ada2705/           2705 原始 benchmark 输出、sm-rates、CUTLASS 日志
docs/evidence/ada2805/           2805 启动日志、rate 读回
docs/evidence/ad104-windows/     Windows 侧 UEFI 事务日志、SHA256 清单
firmware/generate_v67.py         生成 64KB payload
firmware/ada-610.43.03.json      钉死的固件输入清单
firmware/decode_debug_booter.py  调试 booter 解码（研究用）
tools/read_sm_rates.py           只读读 9 个速度字段
tools/print_boot_option.py       打印每张卡的启动参数
tools/trace_booter_stack.py      booter 栈追踪（研究用）
diagnostics/bench/               MMA / GEMM 基准与校验
kernel/                          内核模块源码
```

---

## 十一、一句话总结

**能拿到真收益的部分**：4090（FP16/BF16/TF32 约 2×，指令级）、4070 Ti SUPER（同）、4060 Ti 16G（机制通过，缺对比数据）。
**拿不到的部分**：FP8、INT8、INT4、dp4a，以及任何"应用级也 2 倍"的期待。
**别信的部分**："全系 40 系都支持"和"FP8 翻倍"。

如果你只是想让自己的 4090 在本地推理里快一点：**值得试，但要接受冷启动依赖、Secure Boot 关闭、以及可能只有 10–30% 的实际应用提升。**

---

*本文所有数字均取自仓库内 README 与 `docs/` 下的四份报告及其 `evidence/` 原始输出，未做二次推算；与社交平台说法的差异已逐条标注。*

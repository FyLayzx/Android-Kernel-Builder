# kona (OPPO sm8250) 自动编译说明

本仓库是 [DogDayAndroid/Android-Kernel-Builder](https://github.com/DogDayAndroid/Android-Kernel-Builder)
的一个 fork，**为 OPPO Reno5 Pro+ / sm8250 (kona) 增加了专用自动化编译**。

## 用哪个 workflow

| Workflow | 用途 |
|---|---|
| **`build-kona-kernelsu.yml`** | ✅ **本机专用**：编译 kona 内核 + 集成最新 ReSukiSU/BakaSU，**直接产出可刷的 AnyKernel3 zip** |
| `build.yml` | 上游的通用模板（用官方 `tiann/KernelSU` 的 `setup.sh`，**不支持 non-GKI 4.19**），保留仅作参考 |

## 怎么用

1. 打开仓库的 **Actions** 页面
2. 左侧选 **Build kona kernel (ReSukiSU)** → 右侧 **Run workflow**
3. 保持默认参数直接运行（默认就是：本人 fork 的 `stock/13.1` + `kona-resukisu-defconfig`）
4. 约 15~25 分钟后，在该次运行的 **Artifacts** 里下载 `kona-ResuKSU-<commit>.zip`
5. 这就是**可直接刷入**的 AnyKernel3 包（刷前先备份 boot 分区）

可选参数：

| 参数 | 默认 | 说明 |
|---|---|---|
| `use_proxy` | `true` | 经 `v4.gh-proxy.org` 下载内核/ReSukiSU/工具链；失败会自动回退直连 |
| `kernel_repo` | 本人 fork | 想编别人的树时改这里 |
| `kernel_branch` | `stock/13.1` | |
| `defconfig` | `kona-resukisu-defconfig` | 内核仓库 `arch/arm64/configs/` 下的文件名 |

## 这个 workflow 固化了哪些"坑"

这几条都是**实测踩出来的**，改动前请先读：

1. **KSU 必须用 ReSukiSU/BakaSU** —— KernelSU 官方早已不支持 non-GKI（4.19）。
   workflow 每次构建都拉取 **ReSukiSU 最新代码**，并建立 `drivers/kernelsu -> ../KernelSU/kernel` 软链。
2. **只能用手动钩子**：`CONFIG_KSU_MANUAL_HOOK=y`。
   `CONFIG_KSU_TRACEPOINT_HOOK`（自动钩子）在 non-GKI 上会被 BakaSU 的 Kbuild 直接拒绝：
   `*** TP hooks are incompatible with Non-GKI/GKI 1.0 kernels. Stop.`
3. **不要用 `make <defconfig名>`**：本树上 kbuild 的 defconfig 目标解析不可靠
   （会报 `No rule to make target` 或静默不生成 `.config`）。
   正确做法是**直接铺 `.config` + `olddefconfig`**（workflow 第 5 步即如此）。
4. **必须传 `CROSS_COMPILE=aarch64-linux-gnu-`**，
   否则 ZyC 的 clang 会按 x86_64 主机 target 编译，报一堆
   `unknown register name 'x0' in asm` / `value out of range for constraint 'I'`。
5. **工具链固定为 ZyC clang 11.1.0 整包**（内含 LLD 11.1.0 + binutils 2.39.50），
   与内核作者 youngguo18 完全一致。LLD 链接才会启用 RELR（与能开机镜像同形态）。
6. **WiFi 依赖 `CONFIG_QCA_CLD_WLAN=y`**：该符号 Kconfig 默认 `n`，
   而设备 dump 生成的配置里没有它，`olddefconfig` 会**静默关掉** → 编译成功但 WiFi/热点失效。
   `kona-resukisu-defconfig` 已包含此项（`CONFIG_QCA_CLD_WLAN_PROFILE="qca6390"`）。

## 内核侧改动在哪

内核源码与改动在另一个仓库：
**https://github.com/FyLayzx/android_kernel_OPPO_sm8250**（分支 `stock/13.1`）

其 `KONA_RESUKISU_README.md` 记录了：清理旧 KSU/SuSFS、7 个手动钩子的位置与签名、
工具链获取、构建步骤、以及设备配置里 12 个"源码中不存在"的符号清单。

## 本地复现

不想用 Actions 时，按内核仓库 `KONA_RESUKISU_README.md` 的"构建步骤"操作即可；
本 workflow 的每一步都可直接抄到本地 shell 执行。

# 用近期发布的 PCC 固件中的 VPHONE600AP 组件构建一台虚拟 iPhone（中文对照版）

> 英文原题：**Building virtual iPhone using VPHONE600AP component of recently released PCC firmware**

本文是仓库 [README.md](README.md) 的**中文对照版（inline bilingual）**，做三件事：

1. **逐节保留原 writeup 的英文原文**（含代码块、截图、链接，未删改）；
2. **逐句对照**：每一句英文原文的下面，紧跟一行以 `→` 开头的中文说明；
3. 再补上**面向普通 iOS 工程师的原理与复现细节**：术语表、Apple 启动链、每一步的检查点、
   固件补丁速查表、故障排查清单、法律与风险提示。

配套的可运行脚本（原作者后来整理出的可执行版本）在
[wh1te4ever/super-tart-vphone](https://github.com/wh1te4ever/super-tart-vphone)
（**最后一次提交为 2026-02-25；仓库现已归档 → 只读**）以及它的
[GUIDE.md](https://github.com/wh1te4ever/super-tart-vphone/blob/main/GUIDE.md)。
本文会频繁引用这两处，建议对照阅读。

## 阅读约定

每个小节统一为两层结构：

- **原文 · 译文（逐行对照）** —— 原 writeup 的英文原文**逐句**给出，**每一句英文下面紧跟一行
  以 `→` 开头的中文说明**；代码块、截图、链接保持原样、不重复翻译。
  标注为「本文新增」的小节没有对应原文。
- **补充解读** —— 本文新增（原 writeup 没有的部分），解释「为什么这样做」和「具体怎么做」。

> 原 writeup 是「作者自言自语式」的研究记录：结论和代码片段都在，但大量背景知识、隐含前提、
> 操作顺序和动机没有写出来。**「补充解读」就是为补齐这些空白而写的。**
> 若发现本文与原始脚本/仓库有出入，**以原始仓库为准**。

---

## 目录

- [0. 行前须知：结论、术语与准备](#0-行前须知结论术语与准备)
  - [0.1 一句话结论与整体流水线](#01-一句话结论与整体流水线)
  - [0.2 术语表与 Apple 启动链](#02-术语表与-apple-启动链)
  - [0.3 宿主机环境与工具链（含 Step 1）](#03-宿主机环境与工具链含-step-1)
  - [0.4 用 pccvre 造一台 PCC 研究虚拟机（Step 2）](#04-用-pccvre-造一台-pcc-研究虚拟机step-2)
- [1. 致谢 / Special Thanks](#1-致谢--special-thanks)
- [2. 动机 / Motivation](#2-动机--motivation)
- [3. 改造 super-tart 以启动虚拟 iPhone / Modifying super-tart](#3-改造-super-tart-以启动虚拟-iphone--modifying-super-tart)
- [4. 改造固件 / Modifying the Firmware](#4-改造固件--modifying-the-firmware)
- [5. 构建自定义固件 / Building Custom Firmware](#5-构建自定义固件--building-custom-firmware)
- [6. 打补丁 AVPBooter.vresearch1.bin / Patch AVPBooter.vresearch1.bin](#6-打补丁-avpbootervresearch1bin--patch-avpbootervresearch1bin)
- [7. 改造并编译 libirecovery / Modifying and Building libirecovery](#7-改造并编译-libirecovery--modifying-and-building-libirecovery)
- [8. 给固件组件打补丁 / Patching Firmware Components](#8-给固件组件打补丁--patching-firmware-components)
- [9. 恢复固件 / Restoring Firmware](#9-恢复固件--restoring-firmware)
- [10. 用 SSH Ramdisk 修复启动问题 / Fixing Boot Issue with SSH Ramdisk](#10-用-ssh-ramdisk-修复启动问题--fixing-boot-issue-with-ssh-ramdisk)
- [11. 第一次启动尝试 / First Boot Attempt](#11-第一次启动尝试--first-boot-attempt)
- [12. 实现 Metal / Implementing Metal](#12-实现-metal--implementing-metal)
- [13. 第二次启动尝试 / Second Boot Attempt](#13-第二次启动尝试--second-boot-attempt)
- [14. 兼容性 / Compatibility](#14-兼容性--compatibility)
- [15. 在 Sequoia 上启用触摸交互 / Enabling Touch Interaction on Sequoia](#15-在-sequoia-上启用触摸交互--enabling-touch-interaction-on-sequoia)
- [16. 项目源码 / Project SRC](#16-项目源码--project-src)
- [17. vphone-cli 实现剖析 / Implementation Deep Dive](#17-vphone-cli-实现剖析--implementation-deep-dive)
  - [17.1 一句话结论与规模](#171-一句话结论与规模--scale)
  - [17.2 进程与产物结构](#172-进程与产物结构--processes-and-bundle-layout)
  - [17.3 固件准备：`fw prepare`](#173-固件准备fw-prepare--firmware-preparation)
  - [17.4 固件打补丁：`fw patch`](#174-固件打补丁fw-patch--patching-the-boot-chain)
  - [17.5 恢复固件：`restore`](#175-恢复固件restore--in-process-restore)
  - [17.6 CFW 安装：`cfw install`](#176-cfw-安装cfw-install--guest-system-installation)
  - [17.7 虚拟机运行](#177-虚拟机运行--running-the-vm)
  - [17.8 guest 控制面：`vphoned`](#178-guest-控制面vphoned--guest-control-plane)
  - [17.9 guest 侧组件与注入链](#179-guest-侧组件与注入链--guest-components)
  - [17.10 模板、快照与磁盘](#1710-模板快照与磁盘--templates-snapshots-disk)
  - [17.11 安全模型与权限边界](#1711-安全模型与权限边界--security-model)
  - [17.12 网络四模式](#1712-网络四模式--networking)
  - [17.13 Launchpad：图形化的那一层](#1713-launchpad图形化的那一层--the-app)
  - [17.14 与本文手工流程的逐条对照](#1714-与本文手工流程的逐条对照--writeup-vs-implementation)
  - [17.15 复现命令速查](#1715-复现命令速查--command-cheat-sheet)
  - [17.16 本节结论](#1716-本节结论--takeaways)
- [附录 A：固件补丁速查表](#附录-a固件补丁速查表)
- [附录 B：故障排查清单](#附录-b故障排查清单)
- [附录 C：法律、伦理与风险提示](#附录-c法律伦理与风险提示)
- [附录 D：参考链接](#附录-d参考链接)

---

## 0. 行前须知：结论、术语与准备

### 0.1 一句话结论与整体流水线

**原文 · 译文（逐行对照）**：本节为本文新增，原 writeup 无对应内容。

**中文总结**：

一句话：**Apple 给「Private Cloud Compute（PCC）」用的固件里，混进了一套名为 `vphone600ap` 的
「虚拟 iPhone」硬件定义。作者把 PCC 固件里的这些组件，和一台真实 iPhone 16（iPhone17,3）的
iOS 26.1 固件「拼」在一起，再用改造过的虚拟机软件（super-tart）跑了起来，得到一台能启动、
能进设置界面、甚至有 Metal 加速的虚拟 iPhone。**

整条流水线（也是全文的骨架）：

```
① pccvre / vrevm（Apple 官方 PCC 研究工具，走 Virtualization.framework 私有 API）
        │  产出一台 "pcc-research" 虚拟机的磁盘/SEP/NVRAM
        ▼
② super-tart（第三方 macOS 虚拟机工具，被改造为 super-tart-vphone）
        │  用 vresearch101 平台 + AVPBooter.vresearch1.bin 启动，进入 DFU
        ▼
③ 自定义固件（iPhone 16 的 iOS 26.1 + PCC 的 vphone 内核/驱动，混合 + 打补丁）
        │  用 idevicerestore 把它恢复到虚拟机里
        ▼
④ SSH Ramdisk（从 DFU 手动加载 krnl/ramdisk/dtree 等，拿到 root shell）
        │  用来修补被 SSV 保护、无法正常写入的根文件系统
        ▼
⑤ 启动补丁（Cryptex、seputil、launchd_cache_loader、mobileactivationd…）
        ▼
⑥ 正常启动 → 设置界面 ✅（+ Metal 加速 / SSH / VNC）
```

**重要前提**：这套东西是「研究向、脏活、硬编码」。原作者自己在仓库 README 里明确写了
「This is not for end-user, really messing/dirty project」。所以：

- 它只对**一个特定版本组合**（cloudOS 26.1 / iOS 26.1 build 23B85）有效；
- 所有补丁地址都是**写死的文件偏移**，换个固件版本就全废；
- 有一部分代码（`libAppleParavirtCompilerPluginIOGPUFamily` 的源码）作者因版权顾虑**没有公开**；
- 官方项目已归档，存在一个更现代的替代品（见 [第 14 节](#14-兼容性--compatibility)）。

> 本节出现的 SSV / DFU / AVPBooter / Cryptex / SSH Ramdisk 等术语会在 [0.2](#02-术语表与-apple-启动链)
> 统一解释，第一遍读不通很正常，先记住这条流水线即可。

复现前请先接受以上前提，否则很容易卡在「为什么我的版本跑不通」。

### 0.2 术语表与 Apple 启动链

**原文 · 译文（逐行对照）**：本节为本文新增，原 writeup 无对应内容。

**中文解读**：这一段是让「普通 iOS 开发」跨到「固件研究」的桥梁。建议先通读一遍再动手。

**（1）镜像与签名格式**

| 术语 | 说明 |
| --- | --- |
| **IM4P** | Image4 Payload，Apple 固件的标准封装，带类型标签（`ibss`/`ibec`/`illb`/`ibot`/`krnl`/`trxm`/`rdsk`…）和压缩（一般 lzfse） |
| **IM4M** | Image4 Manifest，即签名（含 APTicket），里面是对各 IM4P 的摘要 |
| **IM4P + IM4M = IMG4** | 一个「已签名」的固件镜像，可被 iBoot / irecovery 加载 |
| **SHSH / APTicket** | Apple 服务器根据设备 ECID、APNonce、要恢复的组件生成的签名票据。没有它，iBoot 不会加载你上传的镜像 |
| **ECID** | 每台设备（这里指虚拟机）的唯一芯片 ID。super-tart-vphone 默认写死为 `0x1de1518ecffe2725`（见 [第 3 节](#3-改造-super-tart-以启动虚拟-iphone--modifying-super-tart)）；改动它会导致已有 SHSH 失效 |
| **APNonce** | 参与 SHSH 的随机数，用于防重放 |
| **PAYP** | IM4P 末尾的一段附属结构（bling），**转换 IM4P → RAW → IM4P 时必须原样保留并修正长度**，否则签名/加载会失败 |
| **AEA** | Apple Encrypted Archive，Cryptex（`SystemOS`/`AppOS`）等镜像的加密分发格式，需要 key 才能解密（本项目用的是 iPhone17,3 恢复目录里的 `.dmg.aea`） |

**（2）启动链（Boot Chain）**

真实 iPhone 的启动顺序，对应到本项目的文件：

```
             (真实设备)                         (本项目虚拟机)
Stage 0  BootROM (硬件，不可改)        →   AVPBooter.vresearch1.bin   ← 我们要打补丁
Stage 1  LLB (Low-Level Boot)          →   LLB.vresearch101.RESEARCH_RELEASE.im4p
Stage 2  iBoot / iBSS / iBEC           →   iBSS / iBEC（DFU/恢复用）、iBoot
             ↓ 加载
         SEP (安全隔区)                 →   AVPSEPBooter.vresearch1.bin + SEPStorage
             ↓
         SPTM（安全页表监控，取代 PPL）→   sptm.vresearch1.release.im4p
         TXM（可信执行监控，管代码签名）→   txm.iphoneos.research.im4p
             ↓
         kernelcache（内核 + 内建 kext）→   kernelcache.research.vphone600
             ↓
         userland（launchd → 设置界面） →   rootfs + Cryptex
```

- **iBSS**：DFU 阶段用，负责进入 DFU 上传镜像；
- **iBEC**：恢复/ramdisk 阶段用；
- **LLB**：正常启动（非 DFU）用，SSV 校验主要发生在这里；
- **SPTM / TXM**：iOS 17 起引入、逐步取代 PPL（PPL 又是 KTRR 的后继）的「新一代安全监控组件」，
  分别负责页表保护与代码签名；
- **TrustCache**：设备上一份「允许运行的二进制哈希白名单」，TXM 用它做校验；
- **SSV（Signed System Volume）**：根卷被密封，任何改动都会导致「seal is broken」而 panic。

**（3）其它名词**

| 术语 | 说明 |
| --- | --- |
| **DFU 模式** | 最底层刷机模式，host 可直接通过 USB 上传并加载 IMG4 |
| **Cryptex** | iOS 里独立于系统卷的加密应用卷（`SystemOS` / `AppOS`），**dyld shared cache、系统 App 都在里面** |
| **dyld shared cache (dsc)** | 预合并的动态库缓存，`libSystem.B.dylib` 等都在其中 |
| **SEPROM / SEPStorage** | 安全隔区的引导与持久化存储，等价于 macOS VM 的 `AuxiliaryStorage` |
| **Gigalocker (.gl)** | SEP 管理的密钥文件（如 `AA.gl`） |
| **launchd / launchd.plist** | 系统第一个用户态进程；`/System/Library/xpc/launchd.plist` 是它的任务数据库 |
| **AMFI** | AppleMobileFileIntegrity，用户态代码签名检查 |
| **jetsam** | iOS 的内存回收机制，panic 里常出现它的报错 |
| **ldid** | 给 Mach-O 重新做（伪）签名的小工具，本项目大量使用 |

### 0.3 宿主机环境与工具链（含 Step 1）

**原文 · 译文（逐行对照）**：本节为本文新增，原 writeup 无对应内容（原 writeup 只在「Compatibility」一节提到机型）。

**中文解读**：

**（1）硬件与系统**

- **必须是 Apple Silicon Mac**（M 系列）。Intel 不行。
- 内存建议 **16GB 起**，磁盘留足 **40GB+**（固件、ramdisk、`pcc-research` 与 `vphone` 两台虚拟机都很占空间）。
- 原作者验证过的组合：
  - Apple M3 / 16GB / **Sequoia 15.7.4**
  - Apple M1 Pro / 32GB / **Tahoe 26.3**
- 理论上任何支持 `pccvre` 的机型都可以。

**（2）关闭宿主安全限制（Step 1，仅限专用机器）**

super-tart 需要加载自定义 bootrom、DFU、GDB 调试，以及用自定义 bootrom 替换
Virtualization.framework 默认使用的 `AVPBooter`（改的是虚拟机目录里的 ROM 文件，不是框架本身），这要求：

```sh
# 进入恢复模式（开机按住电源键 → 选项 → 终端），执行：
csrutil disable                       # 关闭 SIP（super-tart / 私有 API 必需）
csrutil allow-research-guests enable  # 允许运行 PCC 研究用虚拟机（VRE 官方要求）
# 可选：只有当你要「就地」替换宿主机的 /System 下 AVPBooter 时才需要
csrutil authenticated-root disable
# 回到正常系统后：
sudo nvram amfi_get_out_of_my_way=1   # 关闭 AMFI（加载未签名二进制/kext）
```

> 本文流程是把 `AVPBooter.vresearch1.bin` **拷进虚拟机目录**再打补丁的，所以
> `authenticated-root disable` 通常用不到；SIP + AMFI + research-guests 是必需项。

> ⚠️ 这些设置会显著削弱这台 Mac 的安全性。**强烈建议用一台专用测试机**，别在主力机上做。
> 恢复办法：`csrutil enable` / `sudo nvram -d amfi_get_out_of_my_way` / `bless --mount / --last-sealed-snapshot`。
>
> 对比一下：更现代的 [vphone-cli](#14-兼容性--compatibility) 只需要
> `csrutil enable --without debug` + `allow-research-guests enable`，**SIP 可以保持开启**，
> 这一点上更友好。

**检查点**：`csrutil status` 显示 disabled；恢复模式里 `allow-research-guests` 已开启。
**先做完重启，再做后面的事。**

**（3）工具链**

几乎所有「偏门」工具（`img4`、`img4tool`、`ldid`、`sshpass`、`gtar`、`trustcache`、`bspatch`…）
原作者**已经打包在 `super-tart-vphone/CFW/tools/`** 里，直接 clone 就能用。
需要自己装的只有下面这些：

```sh
# 基础包管理
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Apple 固件研究全家桶
brew install pkg-config autoconf automake libtool
brew install libimobiledevice libirecovery   # libusbmuxd 提供 iproxy；irecovery 稍后会被改造版覆盖
brew install blacktop/tap/ipsw               # ipsw 工具，用来解密 AEA / 分析固件
brew install vbindiff                        # 对比补丁前后的字节差异
brew install --cask ghidra                   # 反汇编（或自备商业版 IDA Pro）
brew install python3                         # 若系统没有
python3 -m pip install pyimg4 requests       # pyimg4 用于 IM4P/IM4M/IMG4；requests 供 get_fw.py 下载固件

# Xcode：编译 swift/tart 需要 macOS SDK（GUIDE 里作者用的是 Xcode 26.2）

# idevicerestore：Brew 核心没有，用官方构建脚本或 MacPorts
#   官方脚本：https://github.com/libimobiledevice/idevicerestore  （按 README 的 build script）
#   或：sudo port install idevicerestore
```

还需要两个**改造版**仓库（后面[第 7 节](#7-改造并编译-libirecovery--modifying-and-building-libirecovery)会编译）：

- `wh1te4ever/super-tart-vphone`（改造过的虚拟机软件）
- `wh1te4ever/libirecovery`（支持 `vresearch101ap` 型号的上传工具）

以及 `aea`（macOS 自带的 `/usr/bin/aea`，`fix_boot.py` 用它解密 Cryptex）。

**磁盘布局建议**（后面所有脚本都用相对路径硬编码，别乱改目录名）：

```
super-tart-vphone/
├── CFW/
│   ├── get_fw.py, patch_fw.py, get_rd.py, fix_boot.py
│   ├── custom_26.1/          # 关键的「混合后」BuildManifest.plist / Restore.plist
│   ├── tools/                # 预打包工具
│   └── iPhone17,3_26.1_23B85_Restore/   # get_fw.py 生成的工作目录
```

### 0.4 用 pccvre 造一台 PCC 研究虚拟机（Step 2）

**原文 · 译文（逐行对照）**：本节为本文新增，原 writeup 无对应内容（原 writeup 直接假设你已有一台 PCC 研究虚拟机）。

**中文解读**：

```sh
cd /System/Library/SecurityResearch/usr/bin

./pccvre release list                  # 看看有哪些可用 release
./pccvre release download --release 35622
./pccvre instance create -N pcc-research -R 35622 --variant research
```

这一步会下载 cloudOS 固件并自动恢复，**最终虚拟机会自动关机，这是正常的**。
生成物在：

```
~/Library/Application Support/com.apple.security-research.vrevm/VM-Library/pcc-research.vm/
├── config.plist
├── Disk.img
├── AuxiliaryStorage        # ≈ macOS 虚拟机的 NVRAM
└── SEPStorage              # SEP 的持久化存储
```

**检查点**：上面四个文件都存在，且 `pccvre instance list` 能看到 `pcc-research`。

> 这一步的意义：它给我们提供了一台**已经用 vresearch101 平台正常启动过**的虚拟机骨架，
> 包括正确的磁盘、SEP 存储和 NVRAM。后面搭 `vphone` 虚拟机时要直接复用这些文件。

---

## 1. 致谢 / Special Thanks

**原文 · 译文（逐行对照）**

> **Special Thanks / Acknowledgements**
>
> → **致谢**
>
> - [dlevi309](https://github.com/dlevi309) (Provided ideas for touch interaction on virtual iPhone)
>
> → [dlevi309](https://github.com/dlevi309)（提供了虚拟 iPhone 触摸交互的思路）
>
> - [khanhduytran0](https://github.com/khanhduytran0), [34306](https://github.com/34306), [asdfugil](https://github.com/asdfugil), [verygenericname](https://github.com/verygenericname) (Provided other ideas for building virtual iPhone, including Cryptex, Device Activation, Ramdisk booting, etc.)
>
> → [khanhduytran0](https://github.com/khanhduytran0)、[34306](https://github.com/34306)、[asdfugil](https://github.com/asdfugil)、[verygenericname](https://github.com/verygenericname)（提供了构建虚拟 iPhone 的其他思路，包括 Cryptex、设备激活、Ramdisk 启动等）
>
> - [ma4the](https://github.com/ma4the), [Mard](https://github.com/Mardcelo), [SwallowS](https://github.com/swollows) (Testing if works well on other environment)
>
> → [ma4the](https://github.com/ma4the)、[Mard](https://github.com/Mardcelo)、[SwallowS](https://github.com/swollows)（帮忙测试在其他环境下是否可用）

---

## 2. 动机 / Motivation

**原文 · 译文（逐行对照）**

> **Motivation**
>
> → **动机**
>
> Around late 2024, Apple began introducing [Private Cloud Compute](https://security.apple.com/blog/private-cloud-compute/), claiming to open new horizon for cloud-based AI privacy.
>
> → 2024 年底左右，Apple 开始推出 [Private Cloud Compute](https://security.apple.com/blog/private-cloud-compute/)，宣称要为云端 AI 隐私开辟新天地。
>
> Then, around late 2025, some interesting news appeared: Apple had newly added vphone600ap-related components to PCC firmware, starting with cloudOS 26.
>
> → 然后到了 2025 年底左右，出现了一条有意思的消息：Apple 从 cloudOS 26 开始，往 PCC 固件里**新增了与 `vphone600ap` 相关的组件**。
>
> ![Source: [https://x.com/matteyeux/status/2006339694783848660/photo/1](https://x.com/matteyeux/status/2006339694783848660/photo/1)](contents/image.png)
>
> Source: [https://x.com/matteyeux/status/2006339694783848660/photo/1](https://x.com/matteyeux/status/2006339694783848660/photo/1)
>
> → 图源见上链接。
>
> **"iPhone Research Environment Virtual Machine”?**
>
> → **「iPhone 研究环境虚拟机（iPhone Research Environment Virtual Machine）」？**
>
> Is this a planned move by Apple to build and distribute a virtual iPhone environment for other security researchers in the future, or was it simply a mistake?
>
> → 这是 Apple 计划中的一步——未来为其他安全研究者构建并分发一套虚拟 iPhone 环境？还是仅仅是一次失误？
>
> Given that DEVELOPMENT/KASAN build kernel was once discovered in the iOS 15.0 beta to 15.1 beta3 OTAs back in 2021, the possibility of a slip-up cannot be ruled out.
>
> → 考虑到 2021 年 iOS 15.0 beta 到 15.1 beta3 的 OTA 里曾被发现混入 DEVELOPMENT/KASAN 版本的内核，**「手滑」的可能性也不能排除**。
>
> At that time, the kernel remained included for about 4 months, roughly from June to October 2021.
>
> → 当时那个内核大约在包里保留了 4 个月，大致是 2021 年 6 月到 10 月。
>
> Then, around January of this year, a tweet was posted showing a virtual iPhone booting up utilizing these vphone600ap-related components.
>
> → 再后来，今年 1 月左右，有人发推展示了一台**利用这些 `vphone600ap` 相关组件启动起来的虚拟 iPhone**。
>
> ![Source: [https://x.com/_inside/status/2008951845725548783](https://x.com/_inside/status/2008951845725548783)](contents/Screenshot_2026-02-24_at_7.37.31_PM.png)
>
> Source: [https://x.com/_inside/status/2008951845725548783](https://x.com/_inside/status/2008951845725548783)
>
> → 图源见上链接。
>
> ![Screenshot 2026-02-24 at 7.39.03 PM.png](contents/Screenshot_2026-02-24_at_7.39.03_PM.png)
>
> From what I saw, almost everything worked truly elegantly.
>
> → 从我看到的来看，几乎一切都运行得相当优雅。
>
> Compared to the [QEMUAppleSilicon(Inferno) project](https://github.com/ChefKissInc/Inferno) I had seen previously, it runs much snappier and smoother.
>
> → 相比我之前见过的 [QEMUAppleSilicon(Inferno) 项目](https://github.com/ChefKissInc/Inferno)，它明显更快更流畅。
>
> Furthermore, it even appeared to support Metal acceleration.
>
> → 而且，它**似乎还支持 Metal 加速**。
>
> Ultimately, completely captivated by it, I dove right in and started building my own virtual iPhone on January 31st.
>
> → 最终我彻底被它吸引，一头扎了进去，在 1 月 31 日开始动手搭自己的虚拟 iPhone。
>
> ![Screenshot 2026-02-24 at 7.46.41 PM.png](contents/Screenshot_2026-02-24_at_7.46.41_PM.png)

**补充解读**

普通 iOS 开发者平时接触的是「设备 → .ipa / .ipsw」。这里要先认识三个新东西：

**Private Cloud Compute（PCC）**
Apple 为 Apple Intelligence 搭的云端计算集群，主打隐私。2024 年起 Apple 公开了它的安全文档，
并在 [security.apple.com](https://security.apple.com/documentation/private-cloud-compute) 发布了
研究计划，其中包括一个叫 **VRE（Virtual Research Environment，虚拟研究环境）** 的东西：
让研究者在自己的 Mac 上**启动一个真实的 PCC 节点镜像**来做安全分析。

**cloudOS**
PCC 服务器上运行的操作系统，可以理解成「服务器版 iOS」。它的固件包（类似 iOS 的 ipsw）
托管在 `updates.cdn-apple.com/private-cloud-compute/...`。作者用的版本是 **cloudOS 26.1（23B85）**。

**vphone600ap / vresearch101**

| 名称 | 含义 |
| --- | --- |
| `vphone600ap` | 一个「虚拟 iPhone」的 BoardConfig（主板配置），出现于 cloudOS 26 起 |
| `vresearch101ap` | 对应的「研究用虚拟芯片」平台标识（Platform = `vresearch1`） |
| `iPhone99,11` | 这套虚拟设备被识别出来的 ProductType（真实设备没有这个型号） |
| CPID = `65025`（0xFE01） | 芯片 ID；`vrevm` 源码里 `PlatformType.vresearch101.chipID` 正是 `0xfe01` |

`vphone600ap` 之所以能被「虚拟化」，是因为 Apple 官方工具 **`vrevm`（在
`/System/Library/SecurityResearch/usr/bin/`，源码在 [apple/security-pcc](https://github.com/apple/security-pcc)
的 `srd_tools/vre/`）** 用 **Virtualization.framework 的私有 API** 显式指定了平台类型
（同目录下的 `pccvre` 是给研究者的命令行前端，负责创建/管理 VRE 实例；`vrevm` 是底层启动器）：

```swift
// 摘自 apple/security-pcc  srd_tools/vre/vrevm/VM/VM+Config.swift
// 「vresearch101」平台对应的硬件模型
case .vresearch101:
    let hw_descriptor = _VZMacHardwareModelDescriptor()
    hw_descriptor.setPlatformVersion(3)          // 3 = vrevm 里 PlatformType.vresearch101 的原始值
    hw_descriptor.setISA(.appleInternal4)
    hw_model = VZMacHardwareModel._hardwareModel(with: hw_descriptor)
```

而它配套的**分辨率**也被写死为 `1290 × 2796 @ 460 ppi`——正好是 iPhone 14 Pro Max / 15 Plus /
15 Pro Max / 16 Plus 的规格：

```swift
// 同一文件
let displays_config = VZMacGraphicsDisplayConfiguration(
    widthInPixels: 1290,
    heightInPixels: 2796,
    pixelsPerInch: 460
)
```

**这两个事实就是整篇 writeup 的起点**：既然官方工具能指定「平台类型 = vresearch101」并用
`AVPBooter.vresearch1.bin` 启动，那我们只要改一下第三方虚拟机软件（super-tart），就能用同样的
私有 API 启动同一套固件。

---

## 3. 改造 super-tart 以启动虚拟 iPhone / Modifying super-tart

**原文 · 译文（逐行对照）**

> **Modifying super-tart to boot virtual iPhone**
>
> → **改造 super-tart 以启动虚拟 iPhone**
>
> The referenced project is [security-pcc](https://github.com/apple/security-pcc).
>
> → 参考的项目是 [security-pcc](https://github.com/apple/security-pcc)。
>
> It corresponds to the source code of the /System/Library/SecurityResearch/usr/bin/vrevm binary.
>
> → 它对应 `/System/Library/SecurityResearch/usr/bin/vrevm` 这个二进制的源码。
>
> An interesting point is that it uses private methods provided by Virtualization.framework.
>
> → 有意思的一点是：**它用到了 Virtualization.framework 的私有方法**。
>
> In the virtual machine used for PCC research, you can see that the ISA and PlatformVersion are explicitly specified during the hardware model initialization process.
>
> → 在 PCC 研究用的虚拟机里可以看到，硬件模型初始化时会**显式指定 ISA 和 PlatformVersion**。
>
> ![Screenshot 2026-02-24 at 8.27.01 PM.png](contents/Screenshot_2026-02-24_at_8.27.01_PM.png)
>
> For the bootrom, AVPBooter.vresearch1.bin is used (/System/Library/Frameworks/Virtualization.framework/Resources/AVPBooter.vresearch1.bin)
>
> → bootrom 用的是 `AVPBooter.vresearch1.bin`（位于 `/System/Library/Frameworks/Virtualization.framework/Resources/AVPBooter.vresearch1.bin`）。
>
> ![Screenshot 2026-02-24 at 8.32.08 PM.png](contents/Screenshot_2026-02-24_at_8.32.08_PM.png)
>
> and for the SEPROM (avpsepbooter), AVPSEPBooter.vresearch1.bin is used, which separately loads a SEPStorage file that functions similarly to [AuxiliaryStorage](https://developer.apple.com/documentation/virtualization/vzmacplatformconfiguration/auxiliarystorage).
> (/System/Library/Frameworks/Virtualization.framework/Versions/A/Resources/AVPSEPBooter.vresearch1.bin)
>
> → 而 SEPROM（avpsepbooter）用的是 `AVPSEPBooter.vresearch1.bin`，它会额外加载一个 `SEPStorage` 文件，作用类似于 [AuxiliaryStorage](https://developer.apple.com/documentation/virtualization/vzmacplatformconfiguration/auxiliarystorage)。（位于 `/System/Library/Frameworks/Virtualization.framework/Versions/A/Resources/AVPSEPBooter.vresearch1.bin`）
>
> Another interesting point is that if you look at the code for setting the resolution, it is set to 1290x2796, which corresponds to the iPhone 14 Pro Max, 15 Plus, 15 Pro Max, and 16 Plus devices.
>
> → 另一个有意思的点：如果你去看**设置分辨率**的那段代码，它写的是 **1290x2796**，正好对应 iPhone 14 Pro Max、15 Plus、15 Pro Max 和 16 Plus 这些机型。
>
> ![Screenshot 2026-02-24 at 8.34.11 PM.png](contents/Screenshot_2026-02-24_at_8.34.11_PM.png)
>
> With just this information, it should be more than enough to modify [super-tart](https://github.com/JJTech0130/super-tart) to boot the virtual iPhone.
>
> → 有了这些信息，改造 [super-tart](https://github.com/JJTech0130/super-tart) 让它启动虚拟 iPhone 就绰绰有余了。
>
> I made the modifications as shown below.
>
> → 我做的修改如下。
>
> - /Sources/tart/VM.swift
>
> → （文件：`/Sources/tart/VM.swift`，代码见下方）
>
> ```swift
> ...
> class VM: NSObject, VZVirtualMachineDelegate, ObservableObject {
> ...
>   // vzHardwareModel derives the VZMacHardwareModel config specific to the "platform type"
>   // of the VM (currently only vresearch101 supported)
>   static private func vzHardwareModel_VRESEARCH101() throws -> VZMacHardwareModel {
>     var hw_model: VZMacHardwareModel
>
>     guard let hw_descriptor = _VZMacHardwareModelDescriptor() else {
>       fatalError("Failed to create hardware descriptor")
>     }
>     hw_descriptor.setPlatformVersion(3) // .appleInternal4 = 3
>     hw_descriptor.setBoardID(0x90)
>     hw_descriptor.setISA(2)
>     hw_model = VZMacHardwareModel._hardwareModel(withDescriptor: hw_descriptor)
>
>     guard hw_model.isSupported else {
>         fatalError("VM hardware config not supported (model.isSupported = false)")
>     }
>
>     return hw_model
>   }
>
>   static func craftConfiguration(
>     diskURL: URL,
>     nvramURL: URL,
>     romURL: URL,
>     sepromURL: URL? = nil,
>     vmConfig: VMConfig,
>     network: Network = NetworkShared(),
>     additionalStorageDevices: [VZStorageDeviceConfiguration],
>     directorySharingDevices: [VZDirectorySharingDeviceConfiguration],
>     serialPorts: [VZSerialPortConfiguration],
>     suspendable: Bool = false,
>     nested: Bool = false,
>     audio: Bool = true,
>     clipboard: Bool = true,
>     sync: VZDiskImageSynchronizationMode = .full,
>     caching: VZDiskImageCachingMode? = nil
>   ) throws -> VZVirtualMachineConfiguration {
>     let configuration: VZVirtualMachineConfiguration = .init()
>
>     // Boot loader
>     let bootloader = try vmConfig.platform.bootLoader(nvramURL: nvramURL)
>     Dynamic(bootloader)._setROMURL(romURL)
>     configuration.bootLoader = bootloader
>
>     // SEP ROM
>     let homeURL = FileManager.default.homeDirectoryForCurrentUser
>     var sepstoragePath = homeURL.appendingPathComponent(".tart/vms/vphone/SEPStorage").path
>     let sepstorageURL = URL(fileURLWithPath: sepstoragePath)
>     let sep_config = Dynamic._VZSEPCoprocessorConfiguration(storageURL: sepstorageURL)
>     if let sepromURL { // default AVPSEPBooter.vresearch1.bin from VZ framework
>         sep_config.romBinaryURL = sepromURL
>     }
>     sep_config.debugStub = Dynamic._VZGDBDebugStubConfiguration(port: 8001)
>     configuration._setCoprocessors([sep_config.asObject])
>     
>     // Some vresearch101 config
>     let pconf = VZMacPlatformConfiguration()
>     pconf.hardwareModel = try vzHardwareModel_VRESEARCH101()
>
>     let serial = Dynamic._VZMacSerialNumber.initWithString("AAAAAA1337")
>     let identifier = Dynamic.VZMacMachineIdentifier._machineIdentifierWithECID(0x1111111111111111, serialNumber: serial.asObject)
>     pconf.machineIdentifier = identifier.asObject as! VZMacMachineIdentifier
>
>     pconf._setProductionModeEnabled(true)
>     var auxiliaryStoragePath = homeURL.appendingPathComponent(".tart/vms/vphone/nvram.bin").path
>     let auxiliaryStorageURL = URL(fileURLWithPath: auxiliaryStoragePath)
>     pconf.auxiliaryStorage = VZMacAuxiliaryStorage(url: auxiliaryStorageURL)
>
>     if #available(macOS 14, *) {
>       let keyboard = VZUSBKeyboardConfiguration()
>       configuration.keyboards = [keyboard]
>     }
>
>     if #available(macOS 14, *) {
>       let touch = _VZUSBTouchScreenConfiguration()
>       configuration._setMultiTouchDevices([touch])
>     }
>     ...
>     configuration.platform = pconf
>
>     // Display
>     let graphics_config = VZMacGraphicsDeviceConfiguration()
>     let displays_config = VZMacGraphicsDisplayConfiguration(
>         widthInPixels: 1179,
>         heightInPixels: 2556,
>         pixelsPerInch: 460
>     )
>     graphics_config.displays.append(displays_config)
>     configuration.graphicsDevices = [graphics_config]
>  ...   
> ```

**补充解读**

**（1）Step 3：先用原版 super-tart 搭出 `vphone` 虚拟机骨架**

先用**原版** super-tart 建一台名叫 `vphone` 的虚拟机（原作者说 super-tart-vphone 当时还不能
独立创建 VM，所以借用原版）：

```sh
git clone --recursive https://github.com/JJTech0130/super-tart
cd super-tart
./scripts/run-signed.sh
# 只是借它生成 VM 骨架；磁盘随后会被 pcc-research 的镜像替换，所以随便一个 macOS IPSW 即可
# （可在 https://ipsw.me 下载 UniversalMac 的 12.0.1 21A559 Restore IPSW）
# OS 和磁盘大小随便选，但名字必须是 vphone
./.build/debug/tart create vphone --disk-size 32 \
    --from-ipsw /path/to/UniversalMac_12.0.1_21A559_Restore.ipsw
```

然后清掉不需要的文件，把 [0.4](#04-用-pccvre-造一台-pcc-研究虚拟机step-2) 里 pcc-research 的
磁盘/SEP/NVRAM 拷进来，再放入 vresearch101 的 bootrom：

```sh
cd ~/.tart/vms/vphone
rm -f AVPBooter.vmapple2.bin disk.img nvram.bin          # 清掉 tart 自带的磁盘/固件文件

SRC=~/Library/Application\ Support/com.apple.security-research.vrevm/VM-Library/pcc-research.vm
cp "$SRC/config.plist"     config.plist    # 保留一份原始 PCC 配置备查（tart 实际读的是 config.json）
cp "$SRC/Disk.img"         disk.img
cp "$SRC/AuxiliaryStorage" nvram.bin
cp "$SRC/SEPStorage"       SEPStorage

cp /System/Library/Frameworks/Virtualization.framework/Versions/A/Resources/AVPBooter.vresearch1.bin .
cp /System/Library/Frameworks/Virtualization.framework/Versions/A/Resources/AVPSEPBooter.vresearch1.bin .
```

**检查点**：`~/.tart/vms/vphone/` 下同时有 `config.json`、`config.plist`、`disk.img`、`nvram.bin`、
`SEPStorage`、`AVPBooter.vresearch1.bin`、`AVPSEPBooter.vresearch1.bin`。

**（2）Step 4：编译 super-tart-vphone**

```sh
# 先回到一个干净的开发目录再克隆（不要留在 ~/.tart/vms/vphone）
cd ~/Developer

# 1) 改造版虚拟机软件
git clone --recursive https://github.com/wh1te4ever/super-tart-vphone
cd super-tart-vphone
./scripts/run-signed.sh
# 用改造版跑起来（--dfu 表示进入 DFU 模式）
./.build/debug/tart run vphone --dfu
```

**检查点**：运行 `./.build/debug/tart run vphone --dfu` 后，在
**系统信息 App → 硬件 → USB** 里能看到一个 DFU 设备。看到它，才说明虚拟化和 bootrom 都接上了。

**（3）逐行解释上面那段 VM.swift**

```swift
hw_descriptor.setPlatformVersion(3)   // 3 = vrevm 里 PlatformType.vresearch101 的原始值
hw_descriptor.setBoardID(0x90)        // 0x90 = 144 = vresearch101ap 的 BDID
hw_descriptor.setISA(2)               // 原作者这里用字面量 2；Apple 的 vrevm 源码写的是 .appleInternal4。若启动失败可试 3
hw_model = VZMacHardwareModel._hardwareModel(withDescriptor: hw_descriptor)

// 固定 ECID + 序列号，让 SHSH 可复现
let serial = Dynamic._VZMacSerialNumber.initWithString("AAAAAA1337")
let identifier = Dynamic.VZMacMachineIdentifier._machineIdentifierWithECID(
    0x1de1518ecffe2725, serialNumber: serial.asObject)   // writeup 正文里曾用 0x1111111111111111
pconf.machineIdentifier = identifier.asObject as! VZMacMachineIdentifier

// SEP（对应 AVPSEPBooter + SEPStorage），开 GDB 调试桩
let sep_config = Dynamic._VZSEPCoprocessorConfiguration(storageURL: sepstorageURL)
sep_config.romBinaryURL = sepromURL
sep_config.debugStub = Dynamic._VZGDBDebugStubConfiguration(port: 8001)
configuration._setCoprocessors([sep_config.asObject])

// 触摸（Tahoe/26 上直接可用；Sequoia 需另走 VNC，见 [第 13 节](#13-第二次启动尝试--second-boot-attempt)）
let touch = _VZUSBTouchScreenConfiguration()
configuration._setMultiTouchDevices([touch])

// 显示：注意这里 writeup 正文写 1179×2556，官方 vrevm 是 1290×2796（iPhone 16 Plus 规格）
let displays_config = VZMacGraphicsDisplayConfiguration(
    widthInPixels: 1179, heightInPixels: 2556, pixelsPerInch: 460)
```

> 三个「不一致」值得记住，别当成笔误：
> 1. `setISA(2)` vs 官方 `.appleInternal4`；
> 2. ECID 正文写 `0x1111111111111111`，可运行脚本里是 `0x1de1518ecffe2725`；
> 3. 分辨率正文 `1179×2556`（iPhone 14 Pro）vs 官方 `1290×2796`。
> 这些差异**不影响启动**，但会影响 SHSH 是否可复现、以及显示比例。

**vphone-cli 对应实现**（详见 [17.7](#177-虚拟机运行--running-the-vm)）

原文这段改造 `super-tart` 的 `vzHardwareModel_VRESEARCH101()`，在 vphone-cli 里是一段几乎逐行对应的代码：

```swift
// VPhoneVirtualization/UI/VirtualMachine/VPhoneVirtualMachineHardwareModel.swift
static func createModel() throws -> VZMacHardwareModel {
    let desc = Dynamic._VZMacHardwareModelDescriptor()
    desc.setPlatformVersion(NSNumber(value: UInt32(3)))   // PV=3
    desc.setBoardID(NSNumber(value: UInt32(0x90)))        // vresearch101
    desc.setISA(NSNumber(value: Int64(2)))
    let model = Dynamic.VZMacHardwareModel
        ._hardwareModelWithDescriptor(desc.asObject).asObject as! VZMacHardwareModel
    guard model.isSupported else { throw VPhoneVirtualMachineError.hardwareModelNotSupported }
    return model
}
```

原文里那三处「不一致」，工程实现给了明确答案：

| 原文的分歧 | vphone-cli 的选择 | 位置 |
| --- | --- | --- |
| `setISA(2)` vs `.appleInternal4` | **`2`**（数值一样，写法不同） | `VPhoneVirtualMachineHardwareModel.swift` |
| ECID `0x1111…` vs `0x1de1…` | **不写死**，由 `machineIdentifier` 推导，首次启动后写入 `udid-prediction.txt` | `VPhoneVirtualMachine.swift` |
| 分辨率 `1179×2556` vs `1290×2796` | **`1290×2796 @ 460 ppi`**（`ScreenConfig.default`） | `VPhoneVirtualMachineManifest.swift` |

分辨率这一项比原文更进一步：它不是常量，而是 `config.plist` 里的 `screenConfig`。
iPad guest 会换成对应面板（比如 iPad mini 的 1488×2266 @ 326 ppi），
所以同一个二进制既跑 iPhone 也跑 iPad。

触摸也一样。原文第 15 节手写的鼠标事件重写，在 vphone-cli 里长成
`VPhoneVirtualMachineView`（`VZVirtualMachineView` 的子类），而且代码注释直接引用了
本文的 `ScreenSharingVNC.swift`：

```swift
// VPhoneVirtualMachineView.swift:125
/// https://github.com/wh1te4ever/super-tart-vphone-writeup/blob/main/contents/ScreenSharingVNC.swift
private var multiTouchDevice: AnyObject? { … Dynamic(vm)._multiTouchDevices … }
```

设备清单也是同一套私有 API：`_VZUSBTouchScreenConfiguration`（触摸）、
`_VZPL011SerialPortConfiguration`（串口，对应原文的 `serial=3` 日志）、
`_VZSEPCoprocessorConfiguration`（SEP + `SEPStorage`）、
`_VZGDBDebugStubConfiguration`（对应原文说的「支持实时内核调试」）。

---

## 4. 改造固件 / Modifying the Firmware

**原文 · 译文（逐行对照）**

> **Modifying the Firmware**
>
> → **改造固件**
>
> The referenced project is [vma2pwn](https://github.com/nick-botticelli/vma2pwn).
>
> → 参考的项目是 [vma2pwn](https://github.com/nick-botticelli/vma2pwn)。
>
> Specifically for version 12.0.1, it boots a Mac virtual machine with almost the entire bootchain modified.
>
> → 具体针对 12.0.1 版本，它启动了**一台几乎整条 bootchain 都被修改过的 Mac 虚拟机**。
>
> Let's look at the [prepare.sh](https://github.com/nick-botticelli/vma2pwn/blob/main/prepare.sh) script first.
>
> → 先看它的 [prepare.sh](https://github.com/nick-botticelli/vma2pwn/blob/main/prepare.sh) 脚本。
>
> It extracts firmware components, such as the bootloader and kernel compressed in IM4P format, into RAW format and patches instructions/data at specific hardcoded addresses.
>
> → 它把固件组件（比如以 IM4P 格式压缩的 bootloader 和内核）解包成 RAW 格式，并在**写死的地址上**打指令/数据补丁。
>
> RestoreRamdisk is the root filesystem used when restoring the firmware, and AVPBooter is the BootROM used in the virtual machine.
>
> → 其中 `RestoreRamdisk` 是恢复固件时用的根文件系统，`AVPBooter` 是虚拟机里用的 BootROM。
>
> To summarize, it extracts the individual files included in the firmware and patches integrity checks to allow the restoration of custom firmware, or modifies the boot-args parameters to make it easier to view boot-related logs.
>
> → 总结一下：它把固件里的各个文件抽出来，然后**补掉完整性校验**，从而允许恢复自定义固件；或者修改 **boot-args**，以便更容易查看启动相关的日志。
>
> Finally, [vma2pwn.sh](https://github.com/nick-botticelli/vma2pwn/blob/main/vma2pwn.sh) is responsible for restoring the custom firmware.
>
> → 最后，[vma2pwn.sh](https://github.com/nick-botticelli/vma2pwn/blob/main/vma2pwn.sh) 负责恢复自定义固件。
>
> It does this by entering DFU mode beforehand.
>
> → 做法是**先进入 DFU 模式**。
>
> Here, the virtual machine uses something called super-tart.
>
> → 这里虚拟机用的是叫 **super-tart** 的东西。
>
> This is a version of the existing tart virtual machine with added features like a custom bootrom, serial output, DFU mode, and GDB debugging.
>
> → 它是现有 tart 虚拟机的一个改造版，加了自定义 bootrom、串口输出、DFU 模式、GDB 调试等特性。
>
> (Note that SIP/AMFI must be disabled for this to work.)
>
> → （注意：为此必须关闭 SIP/AMFI。）
>
> I have used it quite usefully recently while I [studying XNU kernel 1-day vulnerabilities (CVE-2021-30937, CVE-2021-30955)](https://github.com/wh1te4ever/xnu_1day_practice).
>
> → 我最近在[研究 XNU 内核 1-day 漏洞（CVE-2021-30937、CVE-2021-30955）](https://github.com/wh1te4ever/xnu_1day_practice)时把它用得挺顺手。
>
> It is fantastic because it supports live kernel debugging.
>
> → 它很棒，因为支持**实时内核调试**。

**补充解读**

- **vma2pwn 是「同类思路的先行者」**：同样是「解包 IM4P → 打补丁 → 重打包 → DFU 恢复」，
  只是对象是 macOS 虚拟机。理解了它，本项目的 `patch_fw.py` 就不难懂。
- **为什么必须关 SIP/AMFI**：super-tart 要加载自定义 bootrom、走私有 API，AMFI 会拦住未签名
  二进制/kext；SIP 会拦住对系统关键路径的替换。这正是 [0.3](#03-宿主机环境与工具链含-step-1) 里那几条命令的原因。
- **bootchain 与各名词的对应关系**见 [0.2（2）](#02-术语表与-apple-启动链)。

**vphone-cli 对应实现**（详见 [17.3](#173-固件准备fw-prepare--firmware-preparation)）

原文这一节是「读懂 vma2pwn 的思路」加上「照抄 vma2pwn 的做法」，没有可运行代码。
vphone-cli 把它拆成了两个明确的动作：

| 原文的说法 | vphone-cli 的对应 |
| --- | --- |
| 「解包成 RAW，在写死的地址打补丁」 | 保留（这是 `fw patch` 的核心），但**地址不再写死**，见[第 8 节](#8-给固件组件打补丁--patching-firmware-components)的补充 |
| 「补掉完整性校验」 | `avpbooter-boot-dgst_bypass`、`*-boot-image4_callback`、`txm-boot-trustcache_bypass` 等声明式补丁 |
| 「改 boot-args 方便看日志」 | `ibec-boot-boot_args` / `llb-boot-boot_args`，字符串**和原文一字不差** |
| 「进入 DFU 模式再恢复」 | `vm launch --dfu` + `restore`（进程内完成，见[第 9 节](#9-恢复固件--restoring-firmware)） |
| 「SIP/AMFI 必须关」 | 有替代路径：SIP 可保持开启，只需 allowlist，见 [17.11](#1711-安全模型与权限边界--security-model) |

原文提到的「支持实时内核调试」，vphone-cli 默认就开着：
`VPhoneVirtualMachine` 在组装配置时挂上 `_VZGDBDebugStubConfiguration`，
不指定端口就由系统分配，指定了就走 `--kernel-debug-port`（6000–65535）。
这是 vma2pwn 那套「改造 tart 加 GDB 调试」在工程实现里的等价物。

---

## 5. 构建自定义固件 / Building Custom Firmware

**原文 · 译文（逐行对照）**

> **Building Custom Firmware**
>
> → **构建自定义固件**
>
> I mixed the components of cloudOS 26.1 (23B85) and iOS 26.1 (iPhone17,3; 23B85), uh,,, but... I can't remember the exact details.
>
> → 我把 **cloudOS 26.1（23B85）** 和 **iOS 26.1（iPhone17,3；23B85）** 的组件混在了一起——呃……但**我记不清具体细节了**。
>
> To be precise, I had to properly mix the iPhone 16 and vphone-related components to create the custom firmware, but I've forgotten exactly which ones I ended up mixing. From what I recall:
>
> → 准确说，我必须把 iPhone 16 和 vphone 相关组件正确地混合，才能做出这个自定义固件，但我已经忘了最后到底混了哪些。凭印象：
>
> - BuildManifest.plist:
>
> → - BuildManifest.plist：
>
> I modified the dictionary elements under the Manifest key. I configured it so that during the restore process, the SystemVolume, SystemVolumeCanonicalMetadata, OS, StaticTrustCache, RestoreTrustCache, and RestoreRamDisk from iPhone 16(iOS 26.1) model would be used. The rest were set up to use vphone-related files from PCC firmware.
>
> → 我修改了 `Manifest` 键下的字典元素。配置成：恢复过程中，`SystemVolume`、`SystemVolumeCanonicalMetadata`、`OS`、`StaticTrustCache`、`RestoreTrustCache`、`RestoreRamDisk` 使用 **iPhone 16（iOS 26.1）** 的；**其余**使用 PCC 固件里的 vphone 相关文件。
>
> - Restore.plist:
>
> → - Restore.plist：
>
> I believe I added properties related to DeviceMap or SupportedProductTypes, or changed the SystemRestoreImageFileSystems element.
>
> → 我印象里是加了 `DeviceMap` 或 `SupportedProductTypes` 相关的属性，或者改了 `SystemRestoreImageFileSystems` 元素。
>
> The files below are the final result of my mix.
>
> → 下面两个文件就是我混合后的最终结果：
>
> [Restore.plist](contents/Restore.plist)
>
> [BuildManifest.plist](contents/BuildManifest.plist)
>
> → [contents/Restore.plist](contents/Restore.plist) / [contents/BuildManifest.plist](contents/BuildManifest.plist)
>
> - get_fw.py (Partial)
>
> → - get_fw.py（节选）
>
> ```python
> ...
>
> # 3. Import things from cloudOS
> # kernelcache
> os.system("cp 399b664dd623358c3de118ffc114e42dcd51c9309e751d43bc949b98f4e31349_extracted/kernelcache.* iPhone17,3_26.1_23B85_Restore")
> # agx, all_flash, ane, dfu, pmp...
> os.system("cp 399b664dd623358c3de118ffc114e42dcd51c9309e751d43bc949b98f4e31349_extracted/Firmware/agx/* iPhone17,3_26.1_23B85_Restore/Firmware/agx")
> os.system("cp 399b664dd623358c3de118ffc114e42dcd51c9309e751d43bc949b98f4e31349_extracted/Firmware/all_flash/* iPhone17,3_26.1_23B85_Restore/Firmware/all_flash")
> os.system("cp 399b664dd623358c3de118ffc114e42dcd51c9309e751d43bc949b98f4e31349_extracted/Firmware/ane/* iPhone17,3_26.1_23B85_Restore/Firmware/ane")
> os.system("cp 399b664dd623358c3de118ffc114e42dcd51c9309e751d43bc949b98f4e31349_extracted/Firmware/dfu/* iPhone17,3_26.1_23B85_Restore/Firmware/dfu")
> os.system("cp 399b664dd623358c3de118ffc114e42dcd51c9309e751d43bc949b98f4e31349_extracted/Firmware/pmp/* iPhone17,3_26.1_23B85_Restore/Firmware/pmp")
> # sptm, txm, etc...
> os.system("cp 399b664dd623358c3de118ffc114e42dcd51c9309e751d43bc949b98f4e31349_extracted/Firmware/*.im4p iPhone17,3_26.1_23B85_Restore/Firmware")
>
> # 4. TODO: parse what things needed from BuildManifest.plist, Restore.plist in cloudOS 26.1
> # It will be really complicated, so import things from already parse completed
> os.system("sudo cp custom_26.1/BuildManifest.plist iPhone17,3_26.1_23B85_Restore")
> os.system("sudo cp custom_26.1/Restore.plist iPhone17,3_26.1_23B85_Restore")
>
> os.system("echo 'Done, grabbed all needed components for restoring'")
> ```

**补充解读**

原作者坦言「记不清确切混了哪些」，但仓库给出了**可运行脚本**，所以直接用它就好：

```sh
cd ~/Developer/super-tart-vphone/CFW   # 脚本都按 CFW/ 相对路径硬编码，务必在仓库根的 CFW 目录下执行

# ① 下载 iPhone 16 (iOS 26.1) 与 cloudOS 26.1 固件，解包，并做初步混合
python3 get_fw.py

# ② 给各组件打补丁，并重打包成 IM4P/IMG4
python3 patch_fw.py
```

`get_fw.py` 做的事，用大白话翻译：

1. 下载 `iPhone17,3_26.1_23B85_Restore.ipsw`（真机 iOS 26.1）与 cloudOS 固件包；
2. 解包到 `iPhone17,3_26.1_23B85_Restore/`；
3. 把 **cloudOS 的 vphone 组件**（kernelcache、`Firmware/agx`、`all_flash`、`ane`、`dfu`、`pmp`、
   `*.im4p` 如 sptm/txm 等）**覆盖进** iPhone 16 的恢复目录；
4. 用仓库里 **`custom_26.1/BuildManifest.plist` 和 `Restore.plist`** 覆盖（这正是原 writeup 里
   「我忘了具体怎么改」的那部分——**答案就在仓库里**）。

这两个 plist 的关键差异（`contents/Restore.plist` 可见）：

- **DeviceMap** 同时登记了三种主板：`d47ap`（iPhone 16）、`vphone600ap`、`vresearch101ap`；
- **SupportedProductTypes** 同时包含 `iPhone17,3`、`iPhone99,11`、`ComputeModule14,x`、`Mac14,14`；
- **SystemRestoreImageFileSystems** 指向两个 APFS 的 `.dmg.aea`（即 Cryptex SystemOS / AppOS）。

`patch_fw.py` 做的事（**所有偏移都是 26.1 / 23B85 / vphone600 专用**）详见
[第 8 节](#8-给固件组件打补丁--patching-firmware-components)。

**检查点**：`patch_fw.py` 跑完不报错；`iPhone17,3_26.1_23B85_Restore/` 下留有一堆 `.bak` 备份。

**vphone-cli 对应实现**（详见 [17.3](#173-固件准备fw-prepare--firmware-preparation)）

原文作者坦言「记不清具体混了哪些」，只留下 `custom_26.1/BuildManifest.plist` 和
`Restore.plist` 两个黑盒结果。vphone-cli 把这件「记不清」的事写成了可读规则，
落在 `VPhoneFirmwarePreparer.prepare()` 里：

| 组件 | 来自 | 原文的做法 |
| --- | --- | --- |
| LLB / iBSS / iBEC / iBoot | PCC `vresearch101ap` | `cp Firmware/dfu/*`、`Firmware/all_flash/*` |
| sptm / txm | PCC（`Firmware/*.im4p`） | 同一句 `cp Firmware/*.im4p` |
| DeviceTree / SEP / KernelCache / RecoveryMode | PCC `vphone600ap` | 混在 `all_flash/*` 里一起拷 |
| `SystemVolume` / `OS` / `StaticTrustCache` / `RestoreRamDisk` | iPhone 17,3 | 保留原恢复目录，不覆盖 |

两个 plist 的处理方式也变了。原文是整份覆盖，vphone-cli 是在 Swift 里**重新生成**
混合 BuildManifest，并且**会先校验**：

- 虚拟机的 DFU 身份必须是 `vresearch101ap`（BDID `0x90`），否则 TSS 拿不到票据；
- 运行时组件必须来自 `vphone600ap`，因为只有它的设备树把 MKB 的 `dt` 置 1，
  允许无系统 keybag 启动；
- 目标 cloudOS 里如果没有 `vphone600ap`，**在解包前就拒绝**，
  而不是等 restore 阶段才炸。原文读者拿 26.4 之后的 cloudOS 去试，
  会在恢复中途失败；vphone-cli 直接说「这个 cloudOS 里没有 `vphone600ap`」。

合并过程也有区别。原文是 `os.system("cp …")` 直接往恢复目录里覆盖，
中途失败就留下半个树；vphone-cli 在**一次性临时目录**里完成全部解包与合并，
整体成功后才移进 VM 目录。

---

## 6. 打补丁 AVPBooter.vresearch1.bin / Patch AVPBooter.vresearch1.bin

**原文 · 译文（逐行对照）**

> **Patch AVPBooter.vresearch1.bin**
>
> → **打补丁 AVPBooter.vresearch1.bin**
>
> I referenced [that post](https://gist.github.com/steven-michaud/fda019a4ae2df3a9295409053a53a65c#iboot-stage-0-avpbootervmapple2binorg).
>
> → 我参考了[那篇 gist](https://gist.github.com/steven-michaud/fda019a4ae2df3a9295409053a53a65c#iboot-stage-0-avpbootervmapple2binorg)。
>
> You must patch `image4_validate_property_callback` in order to load  custom bootloader afterward.
>
> → 为了之后能加载**自定义 bootloader**，你必须 patch `image4_validate_property_callback`。
>
> Simply use "Text-search (slow!)" feature in IDA Pro to search for "0x4447", and patch the epilogue of the corresponding function to always return 0.
>
> → 只要用 IDA Pro 的 `Text-search (slow!)` 功能搜 `0x4447`，然后把对应函数的 epilogue patch 成**永远返回 0** 即可。
>
> ![image.png](contents/image%201.png)

**补充解读**（Step 5）

这是让「自定义固件」能被加载的第一步。用 IDA Pro / Ghidra：

1. 打开 `~/.tart/vms/vphone/AVPBooter.vresearch1.bin`，处理器类型设为 **ARM Little-endian / 64-bit**；
2. 用 `Text-search (slow!)` 搜 `0x4447`（`DG`，DGST 的 magic），定位到
   `image4_validate_property_callback`；
3. 找到函数 **epilogue**，把返回前的 `mov x0, x20`（或类似）改成 `mov x0, #0`，让它永远返回成功；
4. 保存。

> 更稳妥的传法是「只改返回值、不破坏函数体」（gist 作者推荐的做法）；原作者这里用的是
> 「直接让 epilogue 返回 0」。两种都行，前者更安全。
>
> 顺便记录一下 APNonce（GUIDE 里给了示例 `7c2db09d…`）；[第 9 节](#9-恢复固件--restoring-firmware)
> 抓 SHSH 若报 nonce 不匹配，回到这里核对。

**检查点**：用二进制 diff 工具（`vbindiff`）确认只有一处 4 字节被改动。

> **原理提示**：见 [第 8 节](#8-给固件组件打补丁--patching-firmware-components)开头「为什么是这把钥匙」。

**vphone-cli 对应实现**（详见 [17.7](#177-虚拟机运行--running-the-vm)「（4）OpenAVB / OpenSEP」）

原文这一节的整套手工流程，在 vphone-cli 里有两条路，而且**默认走的是不做这件事的那条**。

**（1）Apple ROM 上：同一种定位法，但不是同一段代码**

`AVPBooterPatcher` 的策略和原文的 IDA 流程几乎一致，
但它把「人眼在 IDA 里找 epilogue」换成了固定的五步：

1. 反汇编整个二进制；
2. 找第一条引用 DGST 标记（`0x4447`，即 `0x44475354` 的低半）的指令；
3. 向后最多 512 条，找最近的 `ret` / `retaa` / `retab`；
4. 从 `ret` 向前最多 32 条，找最后一条写 `x0` / `w0` 的 `mov`，或条件选择（`cset`/`csinc`/`csinv`/`csneg`）；
5. 把那条指令改成 `mov x0, #0`。

原文提醒「更稳妥的传法是只改返回值、不破坏函数体」，vphone-cli 的算法正是这个传法：
它**不动函数体**，只改返回前那一条写 `x0` 的指令。补丁名是 `avpbooter-boot-dgst_bypass`。

**（2）新机器上：换掉 ROM，这一节整节的工作直接消失**

vphone-cli 新建的虚拟机默认启动 [OpenAVB](https://github.com/owngoal-dev/OpenAVB)，
它**不校验任何镜像摘要**。于是 `fw patch` 会读 manifest 判断 ROM 类型，
然后**跳过整个 AVPBooter 组件**：

```
[*] Boot ROM:          OpenAVB, which needs no ROM patch
```

补丁标识仍然保持被选中、被规划，这样 `fw patches` 不会误报漂移。
两种 ROM 可以随时互换，不需要重新恢复，因为**它们加载的是同一个已打补丁的 LLB**：

```sh
vphone-cli vm config myphone --rom openavb --seprom opensep   # 开源 ROM
vphone-cli vm config myphone --rom apple --seprom apple       # Apple 的 ROM + 本节的补丁
```

> 代价是**多了一段 DFU 协议**：OpenAVB 不携带任何 iBoot 版本信息，
> 宿主必须先把 iBSS 的身份（IM4P 大小、SHA-256、解压后大小、加载地址、异常级）
> 通过 vendor 请求写进它的 patch-store stage 0，它才肯接收。
> 这段代码在 `MobileRestoreCore/Recovery/openavb.c`，原文没有对应物。

---

## 7. 改造并编译 libirecovery / Modifying and Building libirecovery

**原文 · 译文（逐行对照）**

> **Modifying and Building libirecovery**
>
> → **改造并编译 libirecovery**
>
> Before restoring the firmware, a few modifications were necessary to support the vresearch101ap model.
>
> → 在恢复固件之前，需要做一些修改以支持 **vresearch101ap** 型号。
>
> Once built, firmware restoration becomes possible using the [idevicerestore](https://github.com/libimobiledevice/idevicerestore) tool.
>
> → 编译好之后，就可以用 [idevicerestore](https://github.com/libimobiledevice/idevicerestore) 工具来恢复固件了。
>
> [https://github.com/wh1te4ever/libirecovery](https://github.com/wh1te4ever/libirecovery)
>
> → 项目地址：[wh1te4ever/libirecovery](https://github.com/wh1te4ever/libirecovery)
>
> ![Screenshot 2026-02-24 at 9.52.14 PM.png](contents/Screenshot_2026-02-24_at_9.52.14_PM.png)

**补充解读**（Step 4 的另一半）

```sh
# 2) 支持 vresearch101ap 的 irecovery
git clone --recursive https://github.com/wh1te4ever/libirecovery
cd libirecovery
./autogen.sh && make -j8 && sudo make install
```

- **为什么必须用改造版**：官方 libirecovery 不认识 `vresearch101ap` 这个型号，无法在 DFU 下
  正确识别并上传镜像。改造版加了该型号的支持。
- **它同时服务于两件事**：`idevicerestore` 依赖 libirecovery 来和 DFU 设备通信；
  [第 10 节](#10-用-ssh-ramdisk-修复启动问题--fixing-boot-issue-with-ssh-ramdisk) 的 `irecovery -f ...`
  也用它。
- 安装后可用 `irecovery -q` 验证是否能看到设备信息。

**检查点**：`irecovery -q` 能打印出 `vresearch101ap` 相关字段。

**vphone-cli 对应实现**（详见 [17.5](#175-恢复固件restore--in-process-restore)）

原文为了支持 `vresearch101ap` 而去改造并编译 libirecovery，vphone-cli 的做法是
**把它编进自己的进程**：

| 原文 | vphone-cli |
| --- | --- |
| `git clone wh1te4ever/libirecovery && make install` | `VPhoneRestore/MobileRecoveryCore`（vendored libirecovery，Xcode 直接编译） |
| Homebrew 或源码编译的 `idevicerestore` | `VPhoneRestore/MobileRestoreCore`（vendored idevicerestore） |
| `irecovery -q` 验证 | `vphone-cli restore recovery-probe --ecid <hex>` |

「为什么要改造 libirecovery」这个原因仍然成立：
官方实现不认识 `vresearch101ap`，无法在 DFU 下正确识别并上传镜像。
vphone-cli 只是在 vendored 的那份里做了同样的支持，而不需要一个全局安装的库。

这一步带来的实际差别是「不需要装任何东西」：
`vphone-cli restore` 在当前进程内完成恢复，没有外部 `idevicerestore`、`irecovery` 可依赖，
也没有版本漂移的可能。原文里 `irecovery -f …` 那一串上传命令，
对应的是 `restore` 内部的组件发送序列。

> 顺带一个原文没有的选项：`vphone-cli restore --offline`（`-O`）是离线恢复，
> 就地解密 AEA 镜像并使用缓存的 `.shsh`。原文要绕开在线取票，靠的是
> `fake_tss_server.py`；工程实现里把这个需求做成了一个受支持的 flag，
> 而不是一个需要自己搭的假 TSS 服务器。

---

## 8. 给固件组件打补丁 / Patching Firmware Components

**原文 · 译文（逐行对照）**

> **Patching Firmware Components**
>
> → **给固件组件打补丁**
>
> Similar to AVPBooter, the bootloaders used for restoration, iBSS and iBEC, were patched to bypass signature verification.
>
> → 和 AVPBooter 类似，恢复时用到的 bootloader——**iBSS 和 iBEC**——都被打了补丁以**绕过签名校验**。
>
> I also enabled serial log output so that if there are any booting issues, the cause can be identified immediately.
>
> → 我顺便打开了**串口日志输出**，这样一旦启动出问题，能立刻定位原因。
>
> As you will see later, bypassing [SSV (Signed System Volume) verification](https://support.apple.com/fr-lu/guide/security/secd698747c9/web) is required to load an arbitrary Cryptex.
>
> → 后面你会看到：为了加载一个任意（被修改过的）Cryptex，必须绕过 [SSV（Signed System Volume）校验](https://support.apple.com/fr-lu/guide/security/secd698747c9/web)。
>
> This is performed in the LLB, which is loaded when booting in normal mode rather than DFU mode, and verification is also sometimes performed in the kernel.
>
> → 这一步发生在 **LLB** 里（LLB 在正常启动而非 DFU 模式下加载），**有时内核里也会做校验**。
>
> Additionally, I patched the TXM so that even if a binary/library is not registered in the Trustcache, it is recognized as if it were.
>
> → 此外，我还 patch 了 **TXM**，使得即使某个二进制/库**没有登记在 TrustCache 里**，也能被当作「已登记」来处理。
>
> - patch_fw.py (Partial content, Part 1)
>
> → - patch_fw.py（节选，第 1 部分）
>
> ```python
> # Patch iBSS
> # patch image4_validate_property_callback
> patch(0x9D10, 0xd503201f)   #nop
> patch(0x9D14, 0xd2800000)   #mov x0, #0
>
> # Patch iBEC
> # patch image4_validate_property_callback
> patch(0x9D10, 0xd503201f)   #nop
> patch(0x9D14, 0xd2800000)   #mov x0, #0
> # patch boot-args with "serial=3 -v debug=0x2014e %s"
> patch(0x122d4, 0xd0000082)  #adrp x2, #0x12000
> patch(0x122d8, 0x9101c042)  #add x2, x2, #0x70
> patch(0x24070, "serial=3 -v debug=0x2014e %s")
>
> # Patch LLB
> # patch image4_validate_property_callback
> patch(0xA0D8, 0xd503201f)   #nop
> patch(0xA0DC, 0xd2800000)   #mov x0, #0
> # patch boot-args with "serial=3 -v debug=0x2014e %s"
> patch(0x12888, 0xD0000082)  #adrp x2, #0x12000
> patch(0x1288C, 0x91264042)  #add x2, x2, #0x990
> patch(0x24990, "serial=3 -v debug=0x2014e %s")
> # make possible load edited rootfs (needed to command snaputil -n later)
> patch(0x2BFE8, 0x1400000b)
> patch(0x2bca0, 0xd503201f)
> patch(0x2C03C, 0x17ffff6a)
> patch(0x2fcec, 0xd503201f)
> patch(0x2FEE8, 0x14000009)
> # some unknown patch, bypass panic
> patch(0x1AEE4, 0xd503201f)  #nop
>
> # 6. Grab & Patch TXM
> # Patch TXM for make running binary which is not registered in trustcache
> # TXM [Error]: CodeSignature: selector: 24 | 0xA8 | 0x30 | 1
> # Some trace: FFFFFFF01702B018->sub_FFFFFFF0170306E4->sub_FFFFFFF01703059C->sub_FFFFFFF01703037C->sub_FFFFFFF017030164->sub_FFFFFFF01702EC70 (base: 0xFFFFFFF017004000)
> patch(0x2c1f8, 0xd2800000)      #FFFFFFF0170301F8
> patch(0x2bef4, 0xd2800000)      #FFFFFFF01702FEF4
> patch(0x2c060, 0xd2800000)      #FFFFFFF017030060
>
> # 7. Grab & patch kernelcache
> # ========= Bypass SSV =========
> # _apfs_vfsop_mount: Prevent panic "Failed to find the root snapshot. Rooting from the live fs ..."
> patch(0x2476964, 0xd503201f)  #FFFFFE000947A964
> # _authapfs_seal_is_broken: Prevent panic "root volume seal is broken ..."
> patch(0x23cfde4, 0xd503201f) #FFFFFE00093D3DE4 
> # _bsd_init: Prevent panic "rootvp not authenticated after mounting ..."
> patch(0xf6d960, 0xd503201f)    #FFFFFE0007F71960
> ...
> ```
>
> After converting to the RAW format and patching it, you need to convert it back to IM4P.
>
> → 把它转成 RAW 格式并打补丁之后，还需要再**转回 IM4P**。
>
> In the case of the kernel or TXM, a PAYP structure exists, so it was necessary to preserve that structure.
>
> → 对于内核或 TXM，存在一个 **PAYP** 结构，所以必须把该结构保留下来。
>
> Below is the code that converts IM4P → RAW → IM4P using [pyimg4](https://pypi.org/project/pyimg4/), [img4tool](https://github.com/tihmstar/img4tool), [img4](https://github.com/xerub/img4lib) tool.
>
> → 下面是用 [pyimg4](https://pypi.org/project/pyimg4/)、[img4tool](https://github.com/tihmstar/img4tool)、[img4](https://github.com/xerub/img4lib) 做 IM4P → RAW → IM4P 转换的代码。
>
> - patch_fw.py (Partial content, Part 2)
>
> → - patch_fw.py（节选，第 2 部分）
>
> ```python
> ...
>
> # Patch iBSS
> if not os.path.exists("iPhone17,3_26.1_23B85_Restore/Firmware/dfu/iBSS.vresearch101.RELEASE.im4p.bak"):
>     os.system("cp iPhone17,3_26.1_23B85_Restore/Firmware/dfu/iBSS.vresearch101.RELEASE.im4p iPhone17,3_26.1_23B85_Restore/Firmware/dfu/iBSS.vresearch101.RELEASE.im4p.bak")
> os.system("tools/img4 -i iPhone17,3_26.1_23B85_Restore/Firmware/dfu/iBSS.vresearch101.RELEASE.im4p.bak -o iBSS.vresearch101.RELEASE")
> ... # patch things from raw
> os.system("tools/img4tool -c iPhone17,3_26.1_23B85_Restore/Firmware/dfu/iBSS.vresearch101.RELEASE.im4p -t ibss iBSS.vresearch101.RELEASE")
>
> # Patch iBEC
> if not os.path.exists("iPhone17,3_26.1_23B85_Restore/Firmware/dfu/iBEC.vresearch101.RELEASE.im4p.bak"):
>     os.system("cp iPhone17,3_26.1_23B85_Restore/Firmware/dfu/iBEC.vresearch101.RELEASE.im4p iPhone17,3_26.1_23B85_Restore/Firmware/dfu/iBEC.vresearch101.RELEASE.im4p.bak")
> os.system("tools/img4 -i iPhone17,3_26.1_23B85_Restore/Firmware/dfu/iBEC.vresearch101.RELEASE.im4p.bak -o iBEC.vresearch101.RELEASE")
> ... # patch things from raw
> os.system("tools/img4tool -c iPhone17,3_26.1_23B85_Restore/Firmware/dfu/iBEC.vresearch101.RELEASE.im4p -t ibec iBEC.vresearch101.RELEASE")
>
> # Patch LLB
> if not os.path.exists("iPhone17,3_26.1_23B85_Restore/Firmware/all_flash/LLB.vresearch101.RESEARCH_RELEASE.im4p.bak"):
>     os.system("cp iPhone17,3_26.1_23B85_Restore/Firmware/all_flash/LLB.vresearch101.RESEARCH_RELEASE.im4p iPhone17,3_26.1_23B85_Restore/Firmware/all_flash/LLB.vresearch101.RESEARCH_RELEASE.im4p.bak")
> os.system("tools/img4 -i iPhone17,3_26.1_23B85_Restore/Firmware/all_flash/LLB.vresearch101.RESEARCH_RELEASE.im4p.bak -o LLB.vresearch101.RESEARCH_RELEASE")
> ... # patch things from raw
> os.system("tools/img4tool -c iPhone17,3_26.1_23B85_Restore/Firmware/all_flash/LLB.vresearch101.RESEARCH_RELEASE.im4p -t illb LLB.vresearch101.RESEARCH_RELEASE")
>
> # 6. Grab & Patch TXM
> if not os.path.exists("iPhone17,3_26.1_23B85_Restore/Firmware/txm.iphoneos.research.im4p.bak"):
>     os.system("cp iPhone17,3_26.1_23B85_Restore/Firmware/txm.iphoneos.research.im4p iPhone17,3_26.1_23B85_Restore/Firmware/txm.iphoneos.research.im4p.bak")
> os.system("pyimg4 im4p extract -i iPhone17,3_26.1_23B85_Restore/Firmware/txm.iphoneos.research.im4p.bak -o txm.raw")
> ... # patch things from raw
> #create im4p
> os.system("pyimg4 im4p create -i txm.raw -o txm.im4p -f trxm --lzfse")
> # preserve payp structure
> txm_im4p_data = Path('iPhone17,3_26.1_23B85_Restore/Firmware/txm.iphoneos.research.im4p.bak').read_bytes()
> payp_offset = txm_im4p_data.rfind(b'PAYP')
> if payp_offset == -1:
>     print("Couldn't find payp structure !!!")
>     sys.exit()
>
> with open('txm.im4p', 'ab') as f:
>     f.write(txm_im4p_data[(payp_offset-10):])
>
> payp_sz = len(txm_im4p_data[(payp_offset-10):])
> print(f"payp sz: {payp_sz}")
>
> txm_im4p_data = bytearray(open('txm.im4p', 'rb').read())
> txm_im4p_data[2:5] = (int.from_bytes(txm_im4p_data[2:5], 'big') + payp_sz).to_bytes(3, 'big')
> open('txm.im4p', 'wb').write(txm_im4p_data)
> os.system("mv txm.im4p iPhone17,3_26.1_23B85_Restore/Firmware/txm.iphoneos.research.im4p")
>
> # 7. Grab & patch kernelcache
> if not os.path.exists("iPhone17,3_26.1_23B85_Restore/kernelcache.research.vphone600.bak"):
>     os.system("cp iPhone17,3_26.1_23B85_Restore/kernelcache.research.vphone600 iPhone17,3_26.1_23B85_Restore/kernelcache.research.vphone600.bak")
> os.system("pyimg4 im4p extract -i iPhone17,3_26.1_23B85_Restore/kernelcache.research.vphone600.bak -o kcache.raw")
> ... # patch things from raw
> #create im4p
> os.system("pyimg4 im4p create -i kcache.raw -o krnl.im4p -f krnl --lzfse")
>
> # preserve payp structure
> kernel_im4p_data = Path('iPhone17,3_26.1_23B85_Restore/kernelcache.research.vphone600.bak').read_bytes()
> payp_offset = kernel_im4p_data.rfind(b'PAYP')
> if payp_offset == -1:
>     print("Couldn't find payp structure !!!")
>     sys.exit()
>
> with open('krnl.im4p', 'ab') as f:
>     f.write(kernel_im4p_data[(payp_offset-10):])
>
> payp_sz = len(kernel_im4p_data[(payp_offset-10):])
> print(f"payp sz: {payp_sz}")
>
> kernel_im4p_data = bytearray(open('krnl.im4p', 'rb').read())
> kernel_im4p_data[2:5] = (int.from_bytes(kernel_im4p_data[2:5], 'big') + payp_sz).to_bytes(3, 'big')
> open('krnl.im4p', 'wb').write(kernel_im4p_data)
>
> os.system("mv krnl.im4p iPhone17,3_26.1_23B85_Restore/kernelcache.research.vphone600")
> ...
> ```

**补充解读**

**（1）原理：为什么需要这么多补丁**

Apple 的安全模型是「逐级校验、任何一级被改就停」。我们要跑一个**被修改过的固件**，就必须
逐级把这些校验「骗过去」或「绕过去」。作者用的策略不是「删掉整个校验函数」，而是
**只改返回值 / 打 `nop` / 放行特定路径**，尽量少地破坏原有逻辑。

- **`image4_validate_property_callback` —— 一切的钥匙**：iBoot 每一级都调用它去校验下一级 IMG4
  的摘要与属性。把它改成「直接返回 0（成功）」，就能加载任意（被我们改过的）下一级镜像。
  AVPBooter 里用 `Text-search "0x4447"` 定位后再改 epilogue；iBSS / iBEC / LLB 里直接
  `nop` + `mov x0, #0`。
- **为什么要 bypass SSV**：正常启动时 LLB 会验证根卷的密封（SSV）。但我们**必须修改根文件系统**
  （补 Cryptex、补 Metal 库、改 launchd），一旦改动，SSV 就判定「seal is broken」并 panic。
  于是：在 LLB 里放开「允许加载被编辑的 rootfs」的几处判断（为后面 `snaputil -n` 重命名快照铺路）；
  在 kernelcache 里把三处 panic 检查打 `nop`（`_apfs_vfsop_mount`、`_authapfs_seal_is_broken`、`_bsd_init`）。
- **为什么要 patch TXM / TrustCache**：我们往根文件系统里塞进去的二进制**不在 Apple 的 TrustCache 里**，
  TXM 会拒绝执行。所以把 TXM 中「未注册即拒绝」的几个分支改成「当作已注册」。作者在补丁注释里
  保留了触发时的报错原文（`TXM [Error]: CodeSignature: selector: 24 | 0xA8 | 0x30 | 1`），方便对着日志定位。

**（2）IM4P → RAW → IM4P 的坑（PAYP）**

kernelcache 和 TXM 的 IM4P 末尾有 **PAYP** 结构。直接用 `pyimg4 im4p create` 重打包会丢掉它，必须：

- 从原始 `.bak` 里找到 `PAYP` 偏移，把**从 PAYP magic 前 10 字节开始**的整段尾部
  （`data[payp_offset-10:]`）**追加**回去；
- 这 10 字节的长度也要加进 IM4P 头部长度字段（`data[2:5]`，大端）。

`patch_fw.py` 里已经写好了这段逻辑，照抄即可。

**（3）补丁速查表**：见 [附录 A](#附录-a固件补丁速查表)。

**vphone-cli 对应实现**（详见 [17.4](#174-固件打补丁fw-patch--patching-the-boot-chain)）

原文这一节的 12 行 `patch(0x…. , 0x….)` 是最能体现「两代实现差异」的地方。
vphone-cli 干的**是同一件事**，但没有任何一个偏移是写死的，而且每个补丁都有了名字：

| 原文的调用 | vphone-cli 的补丁标识 | 找位置的方法 |
| --- | --- | --- |
| iBSS/LLB `patch(0x9D10, nop)` + `0x9D14, mov x0,#0` | `ibss-boot-image4_callback` / `llb-boot-image4_callback` | 找 `b.ne` 紧跟 `mov x0, x22`，前 8 条内有 `cmp` |
| iBSS `patch(0x1b544, b)` | `ibss-boot-skip_generate_nonce` | 分支模式匹配 |
| iBEC/LLB 写 boot-args 指针 + 字符串 | `ibec-boot-boot_args` / `llb-boot-boot_args` | 字符串锚点 + 写指针的指令序列 |
| LLB 五处 `0x2BFE8` 等（放行被编辑的 rootfs） | `llb-boot-rootfs`（多条记录） | 分支模式 + 上下文 |
| TXM 三处 `mov x0,#0` | `txm-boot-trustcache_bypass` | 二分查找调用点 → `mov x0,#0` |
| kernel `0x2476964`（`_apfs_vfsop_mount`） | `kernel-boot-apfs_vfsop_mount` | panic 字符串 + 调用图 |
| kernel `0x23cfde4`（`_authapfs_seal_is_broken`） | `kernel-boot-apfs_seal_broken` | 同上 |
| kernel `0xf6d960`（`_bsd_init`） | `kernel-boot-bsd_init_rootvp` | 见下 |

以 iBSS 那条补丁为例，工程实现长这样：

```swift
// FirmwarePatcher/IBoot/BasePatches/IBootPatchImage4Callback.swift
guard a.mnemonic == "b.ne" else { continue }
guard b.mnemonic == "mov", b.operandString == "x0, x22" else { continue }
let hasCmp = insns[lookback ..< i].contains { $0.mnemonic == "cmp" }
// 打分：前 64 条内出现 movn w22 / mov w22,#-1 的优先（错误返回路径）
emit(off,     ARM64.nop,     id: "\(component)-boot-image4_callback.bne")
emit(off + 4, ARM64.movX0_0, id: "\(component)-boot-image4_callback.mov")
```

**为什么这一改很重要**：原文 8.2 节的 `patch(0x9D10, …)` 只要固件版本一变就全废，
所以原文只能锁定 26.1 / 23B85。vphone-cli 的定位是语义的，
因此能覆盖 iOS 26.0 到 27.0.1 这么宽的范围。

`_bsd_init` 那条尤其值得对照。原文是「打一个 `nop`」；工程实现的第一步是把补丁点
**重新找出来**：定位 `bsd_init` → 找 rootvp panic 块 → 在该函数内找唯一的 `call` 指令
→ 其后紧跟 `cbnz` → 再跟 `bl imageboot_needed`，最后只改那条分支门控。
这段定位流程还作为规范写进了仓库的 `AGENTS.md`，防止后来人退回硬编码。

**PAYP 那件事依然成立**。`IM4PHandler` 会为 `trxm` 和 `krnl` 这两个类型
保留 PAYP：从原始数据里找到 `PAYP` magic，**从它前 10 字节开始**截取整段尾部，
追加到重建的 IM4P 后面，并把长度加进顶层 DER 长度字段。
原文 8.2 节说的那个坑，在工程实现里是一段带名字的函数（`appendPAYPIfPresent`）。

**补丁变成数据之后多出来的能力**：

```sh
vphone-cli fw patches myphone                        # 看全部声明、预设、以及哪些还没落到 guest
vphone-cli fw set-patches myphone --block kernel-cfw-debugger
vphone-cli fw patch myphone                          # 重新解析并落盘
```

原文里「想关掉某个补丁」只能去改 Python 源码；现在它是一次勾选，
而且因为 `FirmwareOriginals/` 保存了未修改的原始字节，
**关闭补丁是真的回到原始字节**，不是反向应用一遍。

> 还有一条原文没有、但很实用的约束：版本门。比如
> `kernel-boot-thread_guard_violation` 被钉死在 iOS 18，
> `dyld-boot-maxslide` 钉死在 iOS 27。预设可以关掉一个补丁，
> 但不能把一个补丁**强行放宽**到它没有验证过的版本上。

---

## 9. 恢复固件 / Restoring Firmware

**原文 · 译文（逐行对照）**

> **Restoring Firmware**
>
> → **恢复固件**
>
> Once everything is ready, let's put the virtual machine into DFU mode and try restoring it.
>
> → 一切就绪后，把虚拟机切到 **DFU 模式**，试着恢复它。
>
> Below is a screenshot of the panic that occurs if the SEP is not configured properly. If you set it up correctly, it should pass this point without any issues.
>
> → 下面这张截图是 **SEP 没配置正确**时会出现的 panic。如果配置正确，就能顺利通过这一步。
>
> ![image.png](contents/image%202.png)
>
> After the restore is complete, it reboots automatically.
>
> → 恢复完成后虚拟机会**自动重启**。
>
> However, a panic occurs in the launchd process because the /usr/lib/libSystem.B.dylib library is missing.
>
> → 然而 `launchd` 进程会 panic，原因是 **`/usr/lib/libSystem.B.dylib` 库缺失**。
>
> This library is located within the dyld_shared_cache on the Cryptex partition, and for some reason, the Cryptex partition could not be restored.
>
> → 这个库位于 Cryptex 分区上的 `dyld_shared_cache` 里，而因为某些原因**Cryptex 分区没能被恢复**。
>
> As a temporary workaround, you must create an [SSH Ramdisk](https://github.com/verygenericname/SSHRD_Script) to modify the root file system and inject the necessary files.
>
> → 作为临时对策，你必须做一个 [SSH Ramdisk](https://github.com/verygenericname/SSHRD_Script) 来修改根文件系统并注入需要的文件。
>
> That is exactly why the patch related to SSV verification was needed.
>
> → **这正是之前需要打 SSV 相关补丁的原因。**
>
> ![Screenshot 2026-02-24 at 10.24.33 PM.png](contents/Screenshot_2026-02-24_at_10.24.33_PM.png)
>
> ![image.png](contents/image%203.png)

**补充解读**（Step 7）

```sh
# 注意：从本节起，tart 命令在仓库根目录 super-tart-vphone/ 下执行，
# idevicerestore / 各 python 脚本在 CFW/ 下执行；shsh 会生成在 CFW/shsh/

# 1) 让 vphone 停在 DFU（切到仓库根目录）
cd ~/Developer/super-tart-vphone
./.build/debug/tart run vphone --dfu

# 2) 抓本机的 SHSH（签名票据），-t 表示只抓 ticket（切回 CFW/ 目录）
cd CFW
idevicerestore -e -y ./iPhone17,3_26.1_23B85_Restore -t
#    生成 CFW/shsh/<ECID>-iPhone99,11-26.1.shsh（通常是 gzip 压缩）

mv shsh/<ECID>-iPhone99,11-26.1.shsh shsh/<ECID>-iPhone99,11-26.1.shsh.gz
gunzip shsh/<ECID>-iPhone99,11-26.1.shsh.gz

# 3) 正式恢复被我们改过的固件
idevicerestore -e -y ./iPhone17,3_26.1_23B85_Restore
```

- `-e` = erase（全新恢复）；`-y` = 全部自动确认。
- 恢复过程中 **SEP 配置错误会导致 panic**（见 [`contents/image 2.png`](contents/image%202.png)）。
  如果 panic 里指向 SEP，回去检查 [第 3 节](#3-改造-super-tart-以启动虚拟-iphone--modifying-super-tart)
  的 `SEPStorage` / `AVPSEPBooter` 是否拷对。

恢复完成后虚拟机会自动重启，然后 **panic**：原因是 `/usr/lib/libSystem.B.dylib` 缺失
（它属于 Cryptex 的 dsc）。**这是预期行为**，我们用下一节的 SSH Ramdisk 来修。

> 注：仓库里还有 `CFW/fake_tss_server.py` + `tss_response`，用于在无法联网/无法拿正规
> ticket 时伪造 TSS 响应，属于进阶用法。

**vphone-cli 对应实现**（详见 [17.5](#175-恢复固件restore--in-process-restore)）

原文那三条 `idevicerestore` 命令，在 vphone-cli 里是：

```sh
vphone-cli vm launch myphone --dfu &
vphone-cli restore myphone                 # 在线取票 + 恢复，全在进程内
vphone-cli restore myphone --get-shsh      # 只要票据（对应原文的 -t）
```

| 原文 | vphone-cli |
| --- | --- |
| `-t`（只抓 ticket） | `--get-shsh` / `-t` |
| `-e`（erase 全新恢复） | 默认行为，**没有**「不擦除」模式（见下） |
| `-y`（全部确认） | 不需要，非交互 |
| `gunzip` 处理 SHSH | 内部处理 |
| `fake_tss_server.py` | `--offline` + 缓存的 `.shsh` |

**原文这一节那个 panic，工程实现给出了根因判断**：Cryptex 分区没能被恢复，
导致 `libSystem.B.dylib` 缺失。vphone-cli 不能靠 SSH Ramdisk 去救，
它改成**在宿主机离线把 Cryptex 补进去**（下一步 `cfw install`），
所以恢复完成后**不会再出现这个 panic**。

还有一个原文没有的坑被显式化：`--no-erase` 这个 flag **存在但被拒绝使用**。
代码注释里写了实测结论：在 iPhone 27.0 (24A435) + cloudOS 26.4 上尝试就地恢复，
失败三次（混合 manifest 没有 Upgrade 身份；补上之后 cloudOS 的恢复 ramdisk
仍会重建分区图和系统 keybag、擦掉 Data 卷；iPhone 的 update ramdisk 在 cloudOS
恢复内核上 panic）。保留这个 flag 是为了「让用它的脚本拿到这个原因，
而不是一个用法错误」。

> 想重现原文那种「SEP 配置错误导致 panic」的场景，可以直接对比：
> vphone-cli 的 `SEPStorage`、`nvram.bin`、`Disk.img` 三件套是**同一次恢复**产出的，
> 互相之间必须匹配。把它们在虚拟机之间互换，同样会 panic。
> 参见 [17.10](#1710-模板快照与磁盘--templates-snapshots-disk) 里快照为什么必须三件一起搬。

---

## 10. 用 SSH Ramdisk 修复启动问题 / Fixing Boot Issue with SSH Ramdisk

**原文 · 译文（逐行对照）**

> **Fixing Boot Issue by Booting with SSH Ramdisk**
>
> → **用 SSH Ramdisk 启动来修好启动问题**
>
> I am going to try fixing the boot failure issue by utilizing the ramdisk used in https://github.com/verygenericname/SSHRD_Script.
>
> → 我打算借助 https://github.com/verygenericname/SSHRD_Script 里用的 ramdisk 来修复启动失败的问题。
>
> To upload and load components like the bootloader or kernel using the [irecovery](https://github.com/libimobiledevice/libirecovery) tool in DFU mode, an IMG4 image is needed, which requires an IM4M file.
>
> → 在 DFU 模式下，用 [irecovery](https://github.com/libimobiledevice/libirecovery) 上传并加载 bootloader、内核等组件时，需要 **IMG4** 镜像，而 IMG4 又需要一个 **IM4M** 文件。
>
> Therefore, I first fetched shsh file using idevicerestore tool, and then converted it into an IM4M file.
>
> → 所以先用 idevicerestore 工具抓取 shsh 文件，再把它转成 IM4M 文件。
>
> ```bash
> idevicerestore -e -y ./iPhone17,3_26.1_23B85_Restore -t
>
> mv shsh/[ECID]-iPhone99,11-26.1.shsh shsh/[ECID]-iPhone99,11-26.1.shsh.gz
>
> gunzip shsh/[ECID]-iPhone99,11-26.1.shsh.gz
>
> ...
>
> pyimg4 im4m  extract -i shsh/[ECID]-iPhone99,11-26.1.shsh -o vphone.im4m
> ```
>
> Then, using that IM4M file, I generated several IMG4 files for each of the firmware components used, such as iBSS, iBEC, and the devicetree.
>
> → 然后，用这个 IM4M 文件，为每个要用的固件组件（iBSS、iBEC、devicetree 等）**分别生成 IMG4**。
>
> ```python
> # 1. Grab & Patch iBSS 
> if not os.path.exists("iPhone17\\,3_26.1_23B85_Restore/Firmware/dfu/iBSS.vresearch101.RELEASE.im4p.bak"):
>     os.system("cp iPhone17\\,3_26.1_23B85_Restore/Firmware/dfu/iBSS.vresearch101.RELEASE.im4p iPhone17\\,3_26.1_23B85_Restore/Firmware/dfu/iBSS.vresearch101.RELEASE.im4p.bak")
> os.system("tools/img4 -i iPhone17\\,3_26.1_23B85_Restore/Firmware/dfu/iBSS.vresearch101.RELEASE.im4p.bak -o iBSS.vresearch101.RELEASE")
> ... # patch things from raw
> os.system("tools/img4tool -c iBSS.vresearch101.RELEASE.im4p -t ibss iBSS.vresearch101.RELEASE")
> os.system("tools/img4 -i iBSS.vresearch101.RELEASE.im4p -o ./Ramdisk/iBSS.vresearch101.RELEASE.img4 -M ./vphone.im4m")
>
> # 2. Grab & Patch iBEC
> if not os.path.exists("iPhone17\\,3_26.1_23B85_Restore/Firmware/dfu/iBEC.vresearch101.RELEASE.im4p.bak"):
>     os.system("cp iPhone17\\,3_26.1_23B85_Restore/Firmware/dfu/iBEC.vresearch101.RELEASE.im4p iPhone17\\,3_26.1_23B85_Restore/Firmware/dfu/iBEC.vresearch101.RELEASE.im4p.bak")
> os.system("tools/img4 -i iPhone17\\,3_26.1_23B85_Restore/Firmware/dfu/iBEC.vresearch101.RELEASE.im4p -o iBEC.vresearch101.RELEASE")
> ... # patch things from raw
> os.system("tools/img4tool -c iBEC.vresearch101.RELEASE.im4p -t ibec iBEC.vresearch101.RELEASE")
> os.system("tools/img4 -i iBEC.vresearch101.RELEASE.im4p -o Ramdisk/iBEC.vresearch101.RELEASE.img4 -M vphone.im4m")
>
> # 3. Grab SPTM
> os.system("tools/img4 -i iPhone17\\,3_26.1_23B85_Restore/Firmware/sptm.vresearch1.release.im4p -o Ramdisk/sptm.vresearch1.release.img4 -M vphone.im4m -T sptm")
>
> # 4. Grab devicetree
> os.system("tools/img4 -i iPhone17\\,3_26.1_23B85_Restore/Firmware/all_flash/DeviceTree.vphone600ap.im4p -o Ramdisk/DeviceTree.vphone600ap.img4 -M vphone.im4m -T rdtr")
>
> # 5. Grab sep
> os.system("tools/img4 -i iPhone17\\,3_26.1_23B85_Restore/Firmware/all_flash/sep-firmware.vresearch101.RELEASE.im4p -o Ramdisk/sep-firmware.vresearch101.RELEASE.img4 -M vphone.im4m -T rsep")
>
> # 6. Grab & Patch TXM
> if not os.path.exists("iPhone17\\,3_26.1_23B85_Restore/Firmware/txm.iphoneos.release.im4p.bak"):
>     os.system("cp iPhone17\\,3_26.1_23B85_Restore/Firmware/txm.iphoneos.release.im4p iPhone17\\,3_26.1_23B85_Restore/Firmware/txm.iphoneos.release.im4p.bak")
> os.system("pyimg4 im4p extract -i iPhone17\\,3_26.1_23B85_Restore/Firmware/txm.iphoneos.release.im4p.bak -o txm.raw")
> ... # patch things from raw
> #create im4p
> os.system("pyimg4 im4p create -i txm.raw -o txm.im4p -f trxm --lzfse")
> # preserve payp structure
> txm_im4p_data = Path('iPhone17,3_26.1_23B85_Restore/Firmware/txm.iphoneos.release.im4p.bak').read_bytes()
> payp_offset = txm_im4p_data.rfind(b'PAYP')
> if payp_offset == -1:
>     print("Couldn't find payp structure !!!")
>     sys.exit()
>
> with open('txm.im4p', 'ab') as f:
>     f.write(txm_im4p_data[(payp_offset-10):])
>
> payp_sz = len(txm_im4p_data[(payp_offset-10):])
> print(f"payp sz: {payp_sz}")
>
> txm_im4p_data = bytearray(open('txm.im4p', 'rb').read())
> txm_im4p_data[2:5] = (int.from_bytes(txm_im4p_data[2:5], 'big') + payp_sz).to_bytes(3, 'big')
> open('txm.im4p', 'wb').write(txm_im4p_data)
>
> # sign
> os.system("pyimg4 img4 create -p txm.im4p -o Ramdisk/txm.img4 -m vphone.im4m")
>
> # 7. Grab & patch kernelcache
> if not os.path.exists("iPhone17\\,3_26.1_23B85_Restore/kernelcache.research.vphone600.bak"):
>     os.system("cp iPhone17\\,3_26.1_23B85_Restore/kernelcache.research.vphone600 iPhone17\\,3_26.1_23B85_Restore/kernelcache.research.vphone600.bak")
> os.system("pyimg4 im4p extract -i iPhone17\\,3_26.1_23B85_Restore/kernelcache.research.vphone600.bak -o kcache.raw")
> ... # patch things from raw
>
> #create im4p
> os.system("pyimg4 im4p create -i kcache.raw -o krnl.im4p -f rkrn --lzfse")
>
> # preserve payp structure
> kernel_im4p_data = Path('iPhone17,3_26.1_23B85_Restore/kernelcache.research.vphone600.bak').read_bytes()
> payp_offset = kernel_im4p_data.rfind(b'PAYP')
> if payp_offset == -1:
>     print("Couldn't find payp structure !!!")
>     sys.exit()
>
> with open('krnl.im4p', 'ab') as f:
>     f.write(kernel_im4p_data[(payp_offset-10):])
>
> payp_sz = len(kernel_im4p_data[(payp_offset-10):])
> print(f"payp sz: {payp_sz}")
>
> kernel_im4p_data = bytearray(open('krnl.im4p', 'rb').read())
> kernel_im4p_data[2:5] = (int.from_bytes(kernel_im4p_data[2:5], 'big') + payp_sz).to_bytes(3, 'big')
> open('krnl.im4p', 'wb').write(kernel_im4p_data)
>
> # sign
> os.system("pyimg4 img4 create -p krnl.im4p -o Ramdisk/krnl.img4 -m vphone.im4m")
>
> # 8. Grab ramdisk & build custom ramdisk
> os.system("pyimg4 im4p extract -i iPhone17,3_26.1_23B85_Restore/043-53775-129.dmg -o ramdisk.dmg")
> os.system("mkdir SSHRD")
> os.system("sudo hdiutil attach -mountpoint SSHRD ramdisk.dmg -owners off")
> os.system("sudo hdiutil create -size 254m -imagekey diskimage-class=CRawDiskImage -format UDZO -fs APFS -layout NONE -srcfolder SSHRD -copyuid root ramdisk1.dmg")
> os.system("sudo hdiutil detach -force SSHRD")
> os.system("sudo hdiutil attach -mountpoint SSHRD ramdisk1.dmg -owners off")
>
> ... #remove unneccessary files for expand space
>
> #resign all things preserving ents
> target_path= [
>     "SSHRD/usr/local/bin/*", "SSHRD/usr/local/lib/*",
>     "SSHRD/usr/bin/*", "SSHRD/bin/*",
>     "SSHRD/usr/lib/*", "SSHRD/sbin/*", "SSHRD/usr/sbin/*", "SSHRD/usr/libexec/*"
> ]
> for pattern in target_path:
>     for path in glob.glob(pattern):
>         if os.path.isfile(path) and not os.path.islink(path):
>             if "Mach-O" in subprocess.getoutput(f"file \"{path}\""):
>                 os.system(f"tools/ldid_macosx_arm64 -S -M -Cadhoc \"{path}\"")
>
> #8-2. Grab & build custom ramdisk's trustcache while building custom ramdisk
> os.system("pyimg4 im4p extract -i iPhone17,3_26.1_23B85_Restore/Firmware/043-53775-129.dmg.trustcache -o trustcache.raw")
> os.system("tools/trustcache_macos_arm64 create sshrd.tc SSHRD")
> os.system("pyimg4 im4p create -i sshrd.tc -o trustcache.im4p -f rtsc")
> # sign
> os.system("pyimg4 img4 create -p trustcache.im4p -o Ramdisk/trustcache.img4 -m vphone.im4m")
> #8-2. end
>
> os.system("sudo hdiutil detach -force SSHRD")
> os.system("sudo hdiutil resize -sectors min ramdisk1.dmg")
> # sign
> os.system("pyimg4 im4p create -i ramdisk1.dmg -o ramdisk1.dmg.im4p -f rdsk")
> os.system("pyimg4 img4 create -p ramdisk1.dmg.im4p -o Ramdisk/ramdisk.img4 -m vphone.im4m")
> ```
>
> Once all the IMG4 images are created, let's load them one by one and boot with the Ramdisk.
>
> → 所有 IMG4 都生成好之后，逐个加载并用 Ramdisk 启动。
>
> - boot_rd.sh
>
> → - boot_rd.sh（**顺序很重要**，见下方代码块）
>
> ```bash
> #!/bin/zsh
> irecovery -f Ramdisk/iBSS.vresearch101.RELEASE.img4
> irecovery -f Ramdisk/iBEC.vresearch101.RELEASE.img4
> irecovery -c go
>
> sleep 1;
> irecovery -f Ramdisk/sptm.vresearch1.release.img4
> irecovery -c firmware
>
> irecovery -f Ramdisk/txm.img4
> irecovery -c firmware
>
> irecovery -f Ramdisk/trustcache.img4
> irecovery -c firmware
> irecovery -f Ramdisk/ramdisk.img4
> irecovery -c ramdisk
> irecovery -f Ramdisk/DeviceTree.vphone600ap.img4
> irecovery -c devicetree
> irecovery -f Ramdisk/sep-firmware.vresearch101.RELEASE.img4
> irecovery -c firmware
> irecovery -f Ramdisk/krnl.img4
> irecovery -c bootx
>
> ```
>
> Then, you will see the Creeper face from Minecraft in the third window from the left, as shown below.
>
> → 接着，你会看到下面截图里「左边数第三个窗口」出现的 Minecraft **Creeper 脸**。
>
> If you check the USB menu in the System Information app and see "iPhone Research...", you can now access the virtual iPhone shell using [iproxy](https://github.com/libimobiledevice/libusbmuxd/blob/master/tools/iproxy.c) tool. (`iproxy 2222 22 &`)
>
> → 如果你在「系统信息」App 的 USB 菜单里看到 “iPhone Research...”，就可以用 [iproxy](https://github.com/libimobiledevice/libusbmuxd/blob/master/tools/iproxy.c) 工具（`iproxy 2222 22 &`）访问虚拟 iPhone 的 shell 了。
>
> ![image.png](contents/image%204.png)
>
> To modify the root file system, rename the snapshot.
>
> → 要修改根文件系统，需要**重命名快照**。
>
> ```python
> ssh root@127.0.0.1 -p2222
> #pw: alpine
>
> mount_apfs -o rw /dev/disk1s1 /mnt1
>
> snaputil -l /mnt1
> # (then will output will be printed with hash, result may be differ)
> com.apple.os.update-8AAB8DBA5C8F1F756928411675F4A892087B04559CFB084B9E400E661ABAD119
>
> snaputil -n <com.apple.os.update-hash> orig-fs /mnt1
>
> umount /mnt1
>
> exit
> ```
>
> Decrypt the AEA file using the [ipsw](https://github.com/blacktop/ipsw) tool to create a dmg file, mount it, and then transfer the files from the Cryptex partition to the virtual machine.
>
> → 用 [ipsw](https://github.com/blacktop/ipsw) 工具**解密 AEA 文件**得到 dmg，挂载它，然后把 **Cryptex 分区**里的文件传进虚拟机。
>
> Along with the file transfer, specific patches were required.
>
> → 除了传文件，还需要一些特定补丁。
>
> For convenience, I added three specific processes to start on boot: bash, dropbear, and [trollvnc](https://github.com/OwnGoalStudio/TrollVNC).
>
> → 为了方便，我加了三个开机自启的进程：**bash、dropbear 和 [trollvnc](https://github.com/OwnGoalStudio/TrollVNC)**。
>
> seputil had an issue where it couldn't properly find the gigalocker file, so I patched it to always look for AA.gl.
>
> → **seputil** 有个问题：它找不到正确的 gigalocker 文件，所以我把它 patch 成**永远找 `AA.gl`**。
>
> Furthermore, I patched launchd_cache_loader to ensure that the modified /System/Library/xpc/launchd.plist loads correctly.
>
> → 此外，我还 patch 了 **launchd_cache_loader**，确保被修改过的 `/System/Library/xpc/launchd.plist` 能正确加载。
>
> ```python
> ...
>  ========= INSTALL CRYPTEX(SystemOS, AppOS) =========
> # Grab and Decrypt Cryptex(SystemOS) AEA
> key = subprocess.check_output("ipsw fw aea --key iPhone17,3_26.1_23B85_Restore/043-54303-126.dmg.aea", shell=True, text=True).strip()
> print(f"key: {key}")
> os.system(f"aea decrypt -i iPhone17,3_26.1_23B85_Restore/043-54303-126.dmg.aea -o CryptexSystemOS.dmg -key-value '{key}'")
>
> # Grab Cryptex(AppOS)
> os.system(f"cp iPhone17,3_26.1_23B85_Restore/043-54062-129.dmg CryptexAppOS.dmg")
>
> # Mount CryptexSystemOS
> os.system("mkdir CryptexSystemOS")
> os.system("sudo hdiutil attach -mountpoint CryptexSystemOS CryptexSystemOS.dmg -owners off")
>
> # Mount CryptexAppOS
> os.system("mkdir CryptexAppOS")
> os.system("sudo hdiutil attach -mountpoint CryptexAppOS CryptexAppOS.dmg -owners off")
>
> # Prepare
> remote_cmd("/sbin/mount_apfs -o rw /dev/disk1s1 /mnt1")
>
> remote_cmd("/bin/rm -rf /mnt1/System/Cryptexes/App")
> remote_cmd("/bin/rm -rf /mnt1/System/Cryptexes/OS")
>
> remote_cmd("/bin/mkdir -p /mnt1/System/Cryptexes/App")
> remote_cmd("/bin/chmod 0755 /mnt1/System/Cryptexes/App")
> remote_cmd("/bin/mkdir -p /mnt1/System/Cryptexes/OS")
> remote_cmd("/bin/chmod 0755 /mnt1/System/Cryptexes/OS")
>
> # send Cryptex files to device
> print("Copying cryptexs to vphone! Will take about 3 mintues...")
> os.system("tools/sshpass -p 'alpine' scp -q -r -ostricthostkeychecking=false -ouserknownhostsfile=/dev/null -o StrictHostKeyChecking=no -P 2222 CryptexSystemOS/. 'root@127.0.0.1:/mnt1/System/Cryptexes/OS'")
> os.system("tools/sshpass -p 'alpine' scp -q -r -ostricthostkeychecking=false -ouserknownhostsfile=/dev/null -o StrictHostKeyChecking=no -P 2222 CryptexAppOS/. 'root@127.0.0.1:/mnt1/System/Cryptexes/App'")
>
> # Thanks nathan for idea
> # /System/Library/Caches/com.apple.dyld -> /System/Cryptexes/OS/System/Library/Caches/com.apple.dyld/
> remote_cmd("/bin/ln -sf ../../../System/Cryptexes/OS/System/Library/Caches/com.apple.dyld /mnt1/System/Library/Caches/com.apple.dyld")
> # /System/DriverKit/System/Library/dyld -> /System/Cryptexes/OS/System/DriverKit/System/Library/dyld
> remote_cmd("/bin/ln -sf ../../../../System/Cryptexes/OS/System/DriverKit/System/Library/dyld /mnt1/System/DriverKit/System/Library/dyld")
>
> # ========= PATCH SEPUTIL =========
> # remove if already exist
> os.system("rm custom_26.1/seputil 2>/dev/null")
> os.system("rm custom_26.1/seputil.bak 2>/dev/null")
> # backup seputil before patch
> file_path = "/mnt1/usr/libexec/seputil.bak"
> if not check_remote_file_exists(file_path): 
>      print(f"Created backup {file_path}")
>      remote_cmd("/bin/cp /mnt1/usr/libexec/seputil /mnt1/usr/libexec/seputil.bak")
> # grab seputil
> os.system("tools/sshpass -p 'alpine' scp -q -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -P 2222 root@127.0.0.1:/mnt1/usr/libexec/seputil.bak ./custom_26.1")
> os.system("mv custom_26.1/seputil.bak custom_26.1/seputil")
> # patch seputil; prevent error "seputil: Gigalocker file (/mnt7/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX.gl) doesn't exist: No such file or directory"
> fp = open("custom_26.1/seputil", "r+b")
> patch(0x1B3F1, "AA")
> fp.close()
> # sign
> os.system("tools/ldid_macosx_arm64 -S -M -Ksigncert.p12 -Icom.apple.seputil custom_26.1/seputil")
> # send to apply
> os.system("tools/sshpass -p 'alpine' scp -q -r -ostricthostkeychecking=false -ouserknownhostsfile=/dev/null -o StrictHostKeyChecking=no -P 2222 custom_26.1/seputil 'root@127.0.0.1:/mnt1/usr/libexec/seputil'")
> remote_cmd("/bin/chmod 0755 /mnt1/usr/libexec/seputil")
> # clean
> os.system("rm custom_26.1/seputil 2>/dev/null")
>
> # Change gigalocker filename to AA.gl
> remote_cmd("/sbin/mount_apfs -o rw /dev/disk1s3 /mnt3")
> remote_cmd("/bin/mv /mnt3/*.gl /mnt3/AA.gl")
>
> ... # ========= INSTALL AppleParavirtGPUMetalIOGPUFamily =========
>
> # ========= INSTALL iosbinpack64 =========
> # Send to rootfs
> os.system("tools/sshpass -p 'alpine' scp -q -r -ostricthostkeychecking=false -ouserknownhostsfile=/dev/null -o StrictHostKeyChecking=no -P 2222 jb/iosbinpack64.tar 'root@127.0.0.1:/mnt1'")
> # Unpack 
> remote_cmd("/usr/bin/tar --preserve-permissions --no-overwrite-dir -xvf /mnt1/iosbinpack64.tar  -C /mnt1")
> remote_cmd("/bin/rm /mnt1/iosbinpack64.tar")
> # Setup initial dropbear after normal boot
> '''
> /iosbinpack64/bin/mkdir -p /var/dropbear
> /iosbinpack64/bin/cp /iosbinpack64/etc/profile /var/profile
> /iosbinpack64/bin/cp /iosbinpack64/etc/motd /var/motd
> '''
>
> # ========= PATCH launchd_cache_loader (patch required if modifying /System/Library/xpc/launchd.plist) =========
> # remove if already exist
> os.system("rm custom_26.1/launchd_cache_loader 2>/dev/null")
> os.system("rm custom_26.1/launchd_cache_loader.bak 2>/dev/null")
> # backup launchd_cache_loader before patch
> file_path = "/mnt1/usr/libexec/launchd_cache_loader.bak"
> if not check_remote_file_exists(file_path): 
>      print(f"Created backup {file_path}")
>      remote_cmd("/bin/cp /mnt1/usr/libexec/launchd_cache_loader /mnt1/usr/libexec/launchd_cache_loader.bak")
> # grab launchd_cache_loader
> os.system("tools/sshpass -p 'alpine' scp -q -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -P 2222 root@127.0.0.1:/mnt1/usr/libexec/launchd_cache_loader.bak ./custom_26.1")
> os.system("mv custom_26.1/launchd_cache_loader.bak custom_26.1/launchd_cache_loader")
> # patch to apply launchd_unsecure_cache=1
> fp = open("custom_26.1/launchd_cache_loader", "r+b")
> patch(0xB58, 0xd503201f)
> fp.close()
> # sign
> os.system("tools/ldid_macosx_arm64 -S -M -Ksigncert.p12 -Icom.apple.launchd_cache_loader custom_26.1/launchd_cache_loader")
> # send to apply
> os.system("tools/sshpass -p 'alpine' scp -q -r -ostricthostkeychecking=false -ouserknownhostsfile=/dev/null -o StrictHostKeyChecking=no -P 2222 custom_26.1/launchd_cache_loader 'root@127.0.0.1:/mnt1/usr/libexec/launchd_cache_loader'")
> remote_cmd("/bin/chmod 0755 /mnt1/usr/libexec/launchd_cache_loader")
> # clean
> os.system("rm custom_26.1/launchd_cache_loader 2>/dev/null")
>
> # ========= MAKE RUN bash, dropbear, trollvnc automatically when boot =========
> # Send plist to /System/Library/LaunchDaemons
> os.system("tools/sshpass -p 'alpine' scp -q -r -ostricthostkeychecking=false -ouserknownhostsfile=/dev/null -o StrictHostKeyChecking=no -P 2222 jb/LaunchDaemons/bash.plist 'root@127.0.0.1:/mnt1/System/Library/LaunchDaemons'")
> os.system("tools/sshpass -p 'alpine' scp -q -r -ostricthostkeychecking=false -ouserknownhostsfile=/dev/null -o StrictHostKeyChecking=no -P 2222 jb/LaunchDaemons/dropbear.plist 'root@127.0.0.1:/mnt1/System/Library/LaunchDaemons'")
> os.system("tools/sshpass -p 'alpine' scp -q -r -ostricthostkeychecking=false -ouserknownhostsfile=/dev/null -o StrictHostKeyChecking=no -P 2222 jb/LaunchDaemons/trollvnc.plist 'root@127.0.0.1:/mnt1/System/Library/LaunchDaemons'")
> remote_cmd("/bin/chmod 0644 /mnt1/System/Library/LaunchDaemons/bash.plist")
> remote_cmd("/bin/chmod 0644 /mnt1/System/Library/LaunchDaemons/dropbear.plist")
> remote_cmd("/bin/chmod 0644 /mnt1/System/Library/LaunchDaemons/trollvnc.plist")
>
> # Edit /System/Library/xpc/launchd.plist 
> # remove if already exist
> os.system("rm custom_26.1/launchd.plist 2>/dev/null")
> os.system("rm custom_26.1/launchd.plist.bak 2>/dev/null")
> # backup launchd.plist before patch
> file_path = "/mnt1/System/Library/xpc/launchd.plist.bak"
> if not check_remote_file_exists(file_path): 
>      print(f"Created backup {file_path}")
>      remote_cmd("/bin/cp /mnt1/System/Library/xpc/launchd.plist /mnt1/System/Library/xpc/launchd.plist.bak")
> # grab launchd.plist
> os.system("tools/sshpass -p 'alpine' scp -q -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -P 2222 root@127.0.0.1:/mnt1/System/Library/xpc/launchd.plist.bak ./custom_26.1")
> os.system("mv custom_26.1/launchd.plist.bak custom_26.1/launchd.plist")
>
> # Inject bash, dropbear, trollvnc to launchd.plist
> os.system("plutil -convert xml1 custom_26.1/launchd.plist")
>
> # 1. bash
> target_file = 'custom_26.1/launchd.plist'
> source_file = 'jb/LaunchDaemons/bash.plist'
> insert_key  = '/System/Library/LaunchDaemons/bash.plist'
>
> with open(target_file, 'rb') as ft, open(source_file, 'rb') as fs:
>     target_data = plistlib.load(ft)
>     source_data = plistlib.load(fs)
>
> target_data.setdefault('LaunchDaemons', {})[insert_key] = source_data
>
> with open(target_file, 'wb') as f:
>     plistlib.dump(target_data, f, sort_keys=False)
>
> # 2. dropbear
> source_file = 'jb/LaunchDaemons/dropbear.plist'
> insert_key  = '/System/Library/LaunchDaemons/dropbear.plist'
>
> with open(target_file, 'rb') as ft, open(source_file, 'rb') as fs:
>     target_data = plistlib.load(ft)
>     source_data = plistlib.load(fs)
>
> target_data.setdefault('LaunchDaemons', {})[insert_key] = source_data
>
> with open(target_file, 'wb') as f:
>     plistlib.dump(target_data, f, sort_keys=False)
>
> # 3. trollvnc
> source_file = 'jb/LaunchDaemons/trollvnc.plist'
> insert_key  = '/System/Library/LaunchDaemons/trollvnc.plist'
>
> with open(target_file, 'rb') as ft, open(source_file, 'rb') as fs:
>     target_data = plistlib.load(ft)
>     source_data = plistlib.load(fs)
>
> target_data.setdefault('LaunchDaemons', {})[insert_key] = source_data
>
> with open(target_file, 'wb') as f:
>     plistlib.dump(target_data, f, sort_keys=False)
>
> # send to apply
> os.system("tools/sshpass -p 'alpine' scp -q -r -ostricthostkeychecking=false -ouserknownhostsfile=/dev/null -o StrictHostKeyChecking=no -P 2222 custom_26.1/launchd.plist 'root@127.0.0.1:/mnt1/System/Library/xpc'")
> remote_cmd("/bin/chmod 0644 /mnt1/System/Library/xpc/launchd.plist")
> # clean
> os.system("rm custom_26.1/launchd.plist 2>/dev/null")
> # ========= End of MAKE RUN bash, dropbear, trollvnc automatically when boot =========
>
> ...
> remote_cmd("/sbin/halt")
> ...
> ```

**补充解读**

**（1）Step 8：制作并启动 SSH Ramdisk**

```sh
cd ~/Developer/super-tart-vphone/CFW

# 把上一节抓到的 shsh 放到 CFW/vphone.shsh（上一节是在 CFW/ 下执行的）
cp ./shsh/*-iPhone99,11-26.1.shsh ./vphone.shsh

python3 get_rd.py        # 生成 Ramdisk/*.img4（含自建 trustcache、重签名）
./boot_rd.sh             # 逐个 irecovery -f ... 上传并启动

iproxy 2222 22 &         # 把虚拟机 22 端口映射到本机 2222
```

`get_rd.py` 里两个值得注意的点：

- **自建 trustcache**：ramdisk 里的可执行文件都要能被 TXM 认可，所以对它们重新 `ldid` 签名，
  并用 `trustcache_macos_arm64` 重新生成 trustcache 再打回固件；
- **重新打包 ramdisk**：`hdiutil create` 成一个更大的 APFS 镜像，塞进 SSH 工具，再 `resize -sectors min`。

**检查点**：屏幕上出现「Minecraft Creeper 脸」（ramdisk 已启动的标志）；
系统信息 → USB 里能看到 “iPhone Research…”。然后 `ssh root@127.0.0.1 -p2222`（密码 `alpine`）。

**（2）Step 9：修好首次启动（Cryptex / seputil / launchd）**

先重命名 rootfs 快照（这会打破 SSV，**正是我们需要 LLB 补丁的原因**）：

```sh
mount_apfs -o rw /dev/disk1s1 /mnt1
snaputil -l /mnt1
# 记下输出的哈希，例如 com.apple.os.update-8AAB8DBA...
snaputil -n <com.apple.os.update-哈希> orig-fs /mnt1
umount /mnt1
exit
```

然后运行修复脚本（`fix_boot.py`，它会通过 SSH 往虚拟机里灌文件）：

```sh
python3 fix_boot.py
```

它依次完成：

1. **解密并安装 Cryptex**：从 **iPhone17,3 恢复目录**取 Cryptex 镜像
   （`043-54303-126.dmg.aea` / `043-54062-129.dmg`），用 `ipsw fw aea --key` 拿 key →
   `aea decrypt` → 挂载 → `scp` 把 `SystemOS` / `AppOS` 拷进 `/mnt1/System/Cryptexes/{OS,App}`；
   再把 `/System/Library/Caches/com.apple.dyld` 等软链到 Cryptex 里；
2. **patch seputil**：改一小段让它固定找 `AA.gl`，重新用 `ldid` 签名后写回；
   并把 `/mnt3/*.gl` 重命名为 `AA.gl`；
3. **安装 `AppleParavirtGPUMetalIOGPUFamily.bundle`**
   （需要你先自行准备 `custom_26.1/AppleParavirtGPUMetalIOGPUFamily.tar`，仓库里没有；
   Metal 见 [第 12 节](#12-实现-metal--implementing-metal)）；
4. **安装 `iosbinpack64`**（一个轻量越狱工具集，含 bash/dropbear 等）；
5. **patch `launchd_cache_loader`**（允许加载被修改的 `launchd.plist`）；
6. **patch `mobileactivationd`**（绕过激活）；
7. **patch `launchd`**（防 jetsam panic）；
8. **注入 LaunchDaemons**：把 `bash.plist` / `dropbear.plist` / `trollvnc.plist` 加入
   `launchd.plist`，让它们开机自启（这样正常启动后也能 SSH / VNC）；
9. 收尾：在虚拟机内卸载 `/mnt1`、`/mnt3`，宿主机侧 detach 两个 Cryptex 镜像。

**检查点**：脚本无报错。注意 `fix_boot.py` **不会自动关机**，需要你再执行一次
`ssh root@127.0.0.1 -p2222 halt`（GUIDE 第 7 节），虚拟机应自动关机。

**（3）原理：Cryptex / seputil / launchd_cache_loader / mobileactivationd**

- **Cryptex 为什么必须手动补**：PCC 固件/虚拟 iPhone 没有像真机那样把 Cryptex 正常恢复到设备上，
  导致启动后**缺 `libSystem.B.dylib`（它在 Cryptex 的 dsc 里）**，launchd 直接挂掉。
  解决办法就是用 SSH Ramdisk 拿到 root shell，手动把 Cryptex 解密、挂载、拷进
  `System/Cryptexes/{OS,App}` 并建好软链接。
- **seputil**：报 `Gigalocker file (/mnt7/XXXX.gl) doesn't exist`，是因为虚拟机的 `.gl` 文件名
  和固件里写死的期望不一致 → patch 成固定找 `AA.gl`，并真的把文件重命名为 `AA.gl`。
- **launchd_cache_loader**：会加载缓存的 `launchd.plist`。我们改了 plist，所以要允许「不安全缓存」
  （源码注释：`launchd_unsecure_cache=1`）。
- **mobileactivationd**：绕过激活锁（hactivation = 激活状态标记；让 `should_hactivate` 永远返回真）。
- **launchd** 本体：防止 jetsam 属性未初始化导致的 `initproc exited` panic。

**vphone-cli 对应实现**（详见 [17.6](#176-cfw-安装cfw-install--guest-system-installation)）

原文这一节有 500 多行，是整个 writeup 里最重的一段：做 SSH Ramdisk、
拿 root shell、`snaputil -n`、然后跑 `fix_boot.py` 的九个步骤。
vphone-cli 用**一条命令**替代了整节：

```sh
vphone-cli cfw install myphone      # 需要 root
```

**换掉了什么**：原来必须「启动一个 ramdisk、SSH 进去、在客户机里改盘」，
现在改成「**在宿主机上离线挂载虚拟机的磁盘镜像**」。VM 必须关机，
`Disk.img` 被 APFS clone 到私有工作目录，改完整体 rename 覆盖，失败时原盘不动。
整个过程不需要 ramdisk、不需要 SSH、不需要网络。

九个步骤的对应关系：

| 原文 `fix_boot.py` 的步骤 | vphone-cli 的实现 | 差异 |
| --- | --- | --- |
| 1. 解密并安装 Cryptex（`SystemOS`/`AppOS`） | `FirmwarePatcher/CryptexFilesystem/` 七个文件 | 多了**重新封卷**（`…Sealing.swift`）与 manifest 哈希更新 |
| 2. `snaputil -n` 重命名根快照 | `vphone-cli cfw flip-snapshot` | 完全离线，不挂载、不调 `fs_snapshot` |
| 3. patch `seputil`（固定 `AA.gl`） | `system-seputil-boot-gigalocker_uuid` | 同样是改 cstring 里的 uuid 字段为 `AA` |
| 4. 安装 GPU bundle | 从 cloudOS 自动提取（[17.3](#173-固件准备fw-prepare--firmware-preparation)） | 不再需要自己准备 tar |
| 5. 安装 `iosbinpack64` | **不做** | 工程实现不装任何 bootstrap，用 `vphoned` 取代 |
| 6. patch `launchd_cache_loader` | `system-launchd_cache_loader-boot-unsecure_cache_gate` | 同样是把那个 `cbz` 打掉 |
| 7. patch `mobileactivationd` | `system-mobileactivationd-boot-should_hactivate` | 同样是 `mov x0,#1` 覆盖 getter |
| 8. patch `launchd`（防 jetsam panic） | `system-launchd-boot-jetsam_panic_guard_bypass` | 同名语义 |
| 9. 注入 bash/dropbear/trollvnc 自启 | **不做** | 控制面换成 VSOCK + `vphoned`，见[第 13 节](#13-第二次启动尝试--second-boot-attempt) |

**seputil 那一步的解释，工程实现比原文更细**。原文说「gigalocker 文件名和固件里写死的
期望不一致」，代码注释把机理写清楚了：`seputil` 用
`snprintf(path, n, "%s/%s.gl", mountpoint, uuid)` 拼路径，缓存的是**设备自己的 UUID**；
vphone guest 没有那个 UUID，所以找不到文件。把 format string 的第二个 `%s`
改成常量 `AA`，就能解析到安装时放好的 `AA.gl`。
注意它**故意保留第一个 `%s`**：mountpoint 在运行时会在 `/mnt7` 和 `/private/xarts`
之间二选一，两者都得能工作。

**为什么不需要 ramdisk 也能改根卷**：原文必须靠 SSV 的补丁让「被编辑的 rootfs」能启动；
vphone-cli 因为是在宿主机直接改盘，改完自己**重新封卷**并把哈希写回 BuildManifest，
所以 guest 侧看到的仍是一个「自洽」的系统卷。

---

## 11. 第一次启动尝试 / First Boot Attempt

**原文 · 译文（逐行对照）**

> **First Boot Attempt**
>
> → **第一次启动尝试**
>
> The boot process should work fine now, but when you try to proceed past the black setup screen, it resprings and won't go any further.
>
> → 现在启动流程应该能正常工作了，但当你试图**越过（黑底的）设置界面**继续时，它会 **respring（SpringBoard 重启、回到主屏）**，无法再往下走。
>
> ![image.png](contents/image%205.png)
>
> ![image.png](contents/image%206.png)

**补充解读**

- 「respring」= iOS 的 SpringBoard（桌面/窗口管理器）重启，不是整机重启，表现为闪一下回到主屏。
- 这里卡住的原因，是**缺用户态的 GPU/Metal 驱动**（内核已经认得 `AppleParavirtGPU`，
  但用户态缺 `AGXMetal*` 一类库）。→ 见[第 12 节](#12-实现-metal--implementing-metal)。

**vphone-cli 对应实现**

这一段「越过黑底设置页就 respring」的根因判断是对的，工程实现沿着同一条线索继续往下走，
并且把「为什么明明内核认得 GPU、用户态还是没有 Metal」拆成了两层：

| 现象 | 原因 | vphone-cli 的处理 |
| --- | --- | --- |
| 内核 `ioreg` 里有 `AppleParavirtGPU`，`MTLCreateSystemDefaultDevice()` 却返回 null | 用户态缺 `AppleParavirtGPUMetalIOGPUFamily.bundle` | 从 cloudOS 恢复卷自动提取并装进 `System/Library/Extensions` |
| 装了 bundle 仍黑屏、`backboardd` 报 Metal 编译中断 | bundle 里**没有** `libAppleParavirtCompilerPluginIOGPUFamily.dylib` | 用源码重新实现该插件并编进 bundle |
| 只有 App 拿到 Metal，命令行工具拿到 null | 26.x sandbox 不认识研究板的 paravirtual 设备 | `kernel-cfw-paravirt_user_clients`（27 上是引导必需的 `kernel-boot-iouc_sandbox_gate`） |

第三条是原文完全没有的一层。原文的排查停在「装 bundle」，而工程实现发现
即使 bundle 装好，`AppleParavirtDeviceUserClient` 也会被 sandbox 拒绝：

```
System Policy: MetalTest(648) deny(1) iokit-open-user-client AppleParavirtDeviceUserClient
IOServiceOpen returned kIOReturn(0xE00002E2)
```

同一个门禁还会拒绝 `com.apple.WebKit` 的硬解、`spotlightknowledged` 的 ANE、
以及 `vphoned` 的缩放器。所以这条补丁只放行**四个按名字列出的 paravirt 设备类**，
其他拒绝一概保留。

一个连带影响：`cameracaptured` 每次启动都会在 GPU 上预热拍摄着色器，
被拒绝后会崩在 paravirtual Metal 驱动里并 crash-loop；
第一个碰 AVCapture 的进程会阻塞在这个同步 XPC 上，而主线程的 SpringBoard
正好是其中之一。这也是第 12 节那条相机补丁必须和这条共存的原因。

至于原文说的「respring」，vphone-cli 有对应的免重启手段：
`system.respring`（RPC）与 `vphoned` 的 `system.uicache`，
不用整机重启就能重载 SpringBoard 和图标缓存。

---

## 12. 实现 Metal / Implementing Metal

**原文 · 译文（逐行对照）**

> **Implementing Metal**
>
> → **实现 Metal**
>
> When checking with a custom program called MetalTest, it shows that Metal is not supported.
>
> → 用一个叫 **MetalTest** 的自制程序检查时，显示 **Metal 不受支持**。
>
> ```python
> #import <stdio.h>
> #import <Metal/Metal.h>
> #import <Foundation/Foundation.h>
>
> int main(int argc, char *argv[], char *envp[]) {
>     id<MTLDevice> device = MTLCreateSystemDefaultDevice();
>     NSLog(@"device: %@", device);
>
>     if (device) {
>         NSLog(@"Metal Device Create Success: %@", [device name]);
>     } else {
>         NSLog(@"Metal Not Supported!");
>     }
>
>     return 0;
> }
> ```
>
> - Running result
>
> → - 运行结果
>
> ```python
> -bash-4.4# ./MetalTest 
> 2026-02-08 22:49:02.293 MetalTest[633:9434] device: (null)
> 2026-02-08 22:49:02.294 MetalTest[633:9434] Metal Not Supported!
> -bash-4.4# sysctl kern.version
> kern.version: Darwin Kernel Version 25.1.0: Thu Oct 23 11:11:48 PDT 2025; root:xnu-12377.42.6~55/RELEASE_ARM64_VRESEARCH1
> ```
>
> Normally, the output should have looked like the result below.
>
> → 正常情况下，输出**应该**是这样：
>
> ```python
> seo@seos-Virtual-Machine Desktop % sysctl kern.version
> kern.version: Darwin Kernel Version 25.0.0: Mon Aug 25 21:17:21 PDT 2025; root:xnu-12377.1.9~3/RELEASE_ARM64_VMAPPLE
> seo@seos-Virtual-Machine Desktop % ./MetalTest        
> 2026-02-08 23:16:56.846 MetalTest[682:5810] device: <AppleParavirtDevice: 0x102c48fe0>
>     name = Apple Paravirtual device
> 2026-02-08 23:16:56.847 MetalTest[682:5810] Metal Device Create Success: Apple Paravirtual device
> seo@seos-Virtual-Machine Desktop % 
> ```
>
> Checking with `ioreg -l`, as you can see, the kernel was actually recognizing AppleParavirtGPU.
>
> → 用 `ioreg -l` 检查可以看到，**内核其实已经认出了 `AppleParavirtGPU`**。
>
> ![image.png](contents/image%207.png)
>
> When checking on an iPad 7th Gen running iOS 16.6.1, calling the `MTLCreateSystemDefaultDevice` function internally accesses the IOGPU driver through a specific library called `AGXMetalA10`.
>
> → 在一台运行 iOS 16.6.1 的 iPad 7 上检查时，`MTLCreateSystemDefaultDevice` 内部是通过一个叫 `AGXMetalA10` 的库去访问 IOGPU 驱动的。
>
> This `AGXMetalA10` library is located in `/System/Library/Extensions`.
>
> → 这个 `AGXMetalA10` 库就位于 `/System/Library/Extensions`。
>
> A thought suddenly crossed my mind here: wouldn't there be GPU/Metal-related libraries used for virtual iPhone as well?
>
> → 这时我脑子里突然冒出一个念头：**虚拟 iPhone 是不是也有对应的 GPU/Metal 库？**
>
> ![image.png](contents/image%208.png)
>
> Checking that same path in the PCC virtual machine reveals that 7 files exist there.
>
> → 去 PCC 虚拟机里看同一个路径，发现那里**有 7 个文件**。
>
> I took the /System/Library/Extensions/AppleParavirtGPUMetalIOGPUFamily.bundle used in PCC and placed it directly into the virtual iPhone. (I used the SSH Ramdisk for this.)
>
> → 我把 PCC 里用的 `/System/Library/Extensions/AppleParavirtGPUMetalIOGPUFamily.bundle` 直接搬进了虚拟 iPhone（用的是 SSH Ramdisk）。
>
> ![image.png](contents/image%209.png)
>
> Checking MetalTest again, the `MTLCreateSystemDefaultDevice` function now works properly.
>
> → 再测 MetalTest，`MTLCreateSystemDefaultDevice` 就正常了。
>
> ![image.png](contents/image%2010.png)
>
> However, because a specific dylib file does not exist in the iPhone 16 model's dsc(dyld shared cache), I needed to separately reverse-engineer and implement it from the dsc in the PCC.
>
> → 但是，因为 iPhone 16 型号的 **dsc（dyld shared cache）里不存在某个特定的 dylib**，我需要单独从 PCC 的 dsc 里**逆向并实现**它：
>
> - /System/Library/Extensions/AppleParavirtGPUMetalIOGPUFamily.bundle/libAppleParavirtCompilerPluginIOGPUFamily.dylib
>
> → - `/System/Library/Extensions/AppleParavirtGPUMetalIOGPUFamily.bundle/libAppleParavirtCompilerPluginIOGPUFamily.dylib`
>
> ![Screenshot 2026-02-25 at 1.19.40 PM.png](contents/Screenshot_2026-02-25_at_1.19.40_PM.png)
>
> ![image.png](contents/image%2011.png)

**补充解读**（Step 10）

```sh
cd ~/Developer/super-tart-vphone
./.build/debug/tart run vphone --vnc-experimental
```

现在应该能进到有背景的**设置界面**了。Metal 部分：

- 先用原作者提供的 `MetalTest`（`CFW/jb/MetalTest/main.m`）验证：
  `MTLCreateSystemDefaultDevice()` 一开始返回 `null` → Metal 不可用；
- `ioreg -l` 能看到内核已识别 `AppleParavirtGPU`，缺的是用户态驱动；
- 你需要**自己**从 PCC 虚拟机里取出 `/System/Library/Extensions/AppleParavirtGPUMetalIOGPUFamily.bundle`，
  打包成 `custom_26.1/AppleParavirtGPUMetalIOGPUFamily.tar`（仓库里没有这个文件，
  [第 10 节](#10-用-ssh-ramdisk-修复启动问题--fixing-boot-issue-with-ssh-ramdisk) 的脚本
  只负责把它解开到 `/mnt1/System/Library/Extensions/`）；
- 再跑 `MetalTest`，应输出 `Apple Paravirtual device`。

> ⚠️ **诚实提示**：原作者因为版权顾虑，**没有公开** `libAppleParavirtCompilerPluginIOGPUFamily`
> 的源码（仓库里只有 `build.sh` 和证书，没有 `main.m`）。所以严格意义上，**完整 Metal 复现
> 需要你自己参照 PCC 的 dsc 去反向实现这个 dylib**。如果你只想跑起系统，跳过它即可。

**原理小结**：内核其实**已经认得** `AppleParavirtGPU`（`ioreg -l` 能看到），缺的是用户态的
`AGXMetal*` 类驱动。真机上是 `/System/Library/Extensions/AGXMetalA10` 之类。
于是作者把 PCC 里的 `/System/Library/Extensions/AppleParavirtGPUMetalIOGPUFamily.bundle`
整体搬进虚拟机，Metal 就通了；其中还缺一个 dylib，作者**反向重写**了一个
`libAppleParavirtCompilerPluginIOGPUFamily.dylib`（这也是仓库里唯一缺源码的部分）。

**vphone-cli 对应实现**（详见 [17.3](#173-固件准备fw-prepare--firmware-preparation)「（3）GPU 驱动从哪来」）

原文这一节的两次排查（先用 `MetalTest` 确认不支持，再发现内核其实认了 GPU，
最后找到 `AGXMetalA10` 的对应物），在工程实现里变成了一个**自动化的固定流程**：

1. 建一台**无网络设备**的临时虚拟机；
2. 启动到 DFU；
3. 用进程内恢复后端恢复所选 cloudOS IPSW；
4. 只读挂载恢复出来的 System 卷，取出 `AppleParavirtGPUMetalIOGPUFamily.bundle`；
5. 删掉临时虚拟机；
6. 按 cloudOS build 缓存到 `~/.vphone/gpu-drivers/<version>_<build>/`。

所以原文里「需要你自己准备 `custom_26.1/AppleParavirtGPUMetalIOGPUFamily.tar`」
这件事不再存在，bundle 一定和内核同源。装进 guest 的位置也和原文一致：
`System/Library/Extensions/AppleParavirtGPUMetalIOGPUFamily.bundle`。

**关于作者没公开的那个 dylib**：原文最后卡在
`libAppleParavirtCompilerPluginIOGPUFamily.dylib` 缺源码，只能黑屏。
vphone-cli 的解法是**用源码重写了一份**
（`VPhoneGuestComponents/GraphicLoader/main.mm`，源自 0xjohnnydev 的 metal-patch），
在 Xcode 构建期编进 bundle 的 `guest-resources`，`fw prepare` 时再拷进暂存的 GPU bundle。

这就补上了原文最后缺的那一环：cloudOS 26.4 的 bundle 里有
`DTPlatformVersion=26.4`、`CFBundleVersion=64.4.4`，**确实没有**这个插件。
没有它，`MTLCompilerService` 反复 abort、`backboardd` 报 Metal 编译中断，
表现就是「宿主窗口全黑，但 vphoned 连得上」，和原文第 11 节的症状同源。

GPU 真正跑起来的量化结果（M5 Pro，iPadOS 26.6.2 guest + cloudOS 26.4 内核）：

| 项目 | guest | host |
| --- | --- | --- |
| 2048×2048 compute pass | 2.3–3.8 ms | 2.0–3.3 ms |
| 空命令缓冲 + `waitUntilCompleted` | 0.12–0.27 ms | 0.02 ms |
| 着色器编译（热） | 80–220 ms | 265 ms（未缓存） |
| 同一次启动的首次编译 | **4.3–8.3 s** | — |

也就是说 GPU 工作量本身接近原生速度，代价在**同步往返**（贵 6–13 倍）和
**首次编译**。这解释了原文读者常问的「为什么刚开机时卡一阵」。

> 想自己确认驱动在位：`device.ioreg` 能列出研究板的 paravirtual GPU 设备，
> 而 `MetalTest` 这类**非 App 沙盒**的程序即使在驱动正常时也可能拿到 null，
> 这和原文第 11、12 节看到的现象不是一回事。要验证用户态 Metal，
> 应该在 guest 的 App 里试，或者直接观察窗口能不能渲染出设置界面。

---

## 13. 第二次启动尝试 / Second Boot Attempt

**原文 · 译文（逐行对照）**

> **Second Boot Attempt**
>
> → **第二次启动尝试**
>
> Once implemented, you are now greeted by the setup screen with a background.
>
> → 实现之后，迎接你的就是**有背景的设置界面**了。
>
> Since I couldn't properly implement the home button, I resolved this using a temporary workaround by controlling it via iproxy/VNC.
>
> → 由于我没能正确实现 Home 键，我用一个临时方案解决：通过 **iproxy / VNC** 来控制它。
>
> ![image.png](contents/image%2012.png)

**补充解读**（Step 11：交互、VNC 与可选越狱）

- **Home 键**：键盘 **CMD + H** 模拟 Home。
- **触摸**：
  - Tahoe（26）上，`VZVirtualMachineView` 直接支持触摸（`_VZUSBTouchScreenConfiguration`）；
  - Sequoia（15）上不行，原作者通过**重写鼠标事件**实现，代码见
    [`contents/ScreenSharingVNC.swift`](contents/ScreenSharingVNC.swift)（详见[第 15 节](#15-在-sequoia-上启用触摸交互--enabling-touch-interaction-on-sequoia)）。
- **VNC**：`--vnc-experimental` 启动后，把 VNC 端口 5901 用 `iproxy` 映射出来即可（密码 `alpine`）。
- **SSH**：正常启动后 dropbear 监听 **22222** 端口（注意不是 ramdisk 的 22）。

**可选：越狱（Procursus + Ellekit）**，详见 `GUIDE.md` 第 8 节，概括为：

```sh
cd ~/Developer/super-tart-vphone
./.build/debug/tart run vphone --dfu
cd CFW && ./boot_rd.sh
iproxy 2222 22 &
python3 install_jb_procursus.py
ssh root@127.0.0.1 -p2222 halt
# 正常启动后，SSH 到 22222，按 GUIDE 建立 /var/jb 软链、装 Sileo、加 Ellekit 源；
# 之后还要再进一次 DFU + boot_rd.sh，跑 python3 install_jb_basebin.py 收尾（GUIDE 第 8 节后半段）。
```

> 越狱属于完全可选的「附加题」，与「跑起虚拟 iPhone」无关，建议先确保前 12 步稳定再做。

**vphone-cli 对应实现**（详见 [17.8](#178-guest-控制面vphoned--guest-control-plane)）

原文为了解决三件具体的事，装了三个开机自启进程：`bash`、`dropbear`、`trollvnc`。
vphone-cli 把它们换成了**一个 root 守护进程 + VSOCK**，三件事各有对应物：

| 原文的进程 | 解决什么 | vphone-cli 的对应 |
| --- | --- | --- |
| `bash`（开机自启） | 要有个能敲命令的 shell | `vphoned` 的 `processes.list` / `apps.launch`，或装 Irisin bootstrap 后自带 |
| `dropbear`（22222 端口） | 要能远程执行、传文件 | `vphoned` 的 `files.*` 与 157 个 RPC 方法 |
| `trollvnc`（5901 端口，密码 `alpine`） | 要看屏幕、点按 | `vphone.sock` 的 `screenshot` / `tap` / `swipe`，或宿主 VM 窗口本身 |

**Home 键**这件事原文是「没实现，用 iproxy/VNC 绕」。vphone-cli 直接做了：

```sh
# 窗口菜单 Keys → Home，或者走 RPC
{"t":"rpc","method":"input.button","params":{"name":"home"}}
```

原文说「键盘 CMD + H 模拟 Home」，工程实现里 Home 是独立的按钮输入，
不依赖键盘映射。

**VNC 与 SSH 都不装了**。这不是功能缩水：`vphone.sock` 和 HTTP API 都在
**不需要 guest 网络**的情况下工作，因此 `tunnel` 模式下照样可用；
而 dropbear/trollvnc 依赖 guest 网络栈和端口转发。

**关于可选越狱**：原文这一步是「装 Procursus + ElleKit / Sileo」。
vphone-cli 的公开工作流**不装任何 bootstrap**，`cfw install` 只往 `/usr/lib` 放它自己的
hook dylib，注入靠 `launchdhook-vphone` + `SystemHook-vphone` 这条自建链，
不需要 ElleKit。如果确实需要包管理器，走的是另一个入口：

```sh
# 菜单栏 Apps > Install Bootstrap…，选 roothide（rootless 已弃用）
# 然后在 Irisin 里装 owngoal-bootstrap-vphone
```

> 注意这里的取舍：`owngoal-bootstrap-vphone` 是一次装齐 apt/dpkg/bash/zsh/sudo/
> openssh-server 等全套，因为其中几个包互相依赖、`openssh-server` 甚至有循环依赖，
> 逐个装会中途失败。这和原文「先装 Procursus 再补 basebin」是同一个坑的两种解法。

---

## 14. 兼容性 / Compatibility

**原文 · 译文（逐行对照）**

> **Compatibility**
>
> → **兼容性**
>
> It is only compatible with Apple Silicon Macs, and the devices/versions confirmed to work are as follows:
>
> → 它**只兼容 Apple Silicon Mac**，已经确认可用的设备/版本如下：
>
> - Apple M3, 16GB RAM, Sequoia 15.7.4
>
> → - Apple M3 / 16GB 内存 / Sequoia 15.7.4
>
> - Apple M1 Pro, 32GB RAM, Tahoe 26.3
>
> → - Apple M1 Pro / 32GB 内存 / Tahoe 26.3
>
> I expect it will probably work on any target that supports pccvre.
>
> → 我预计**任何支持 `pccvre` 的机型**应该都可以。
>
> ![Source: [https://security.apple.com/documentation/private-cloud-compute/vresetup](https://security.apple.com/documentation/private-cloud-compute/vresetup)](contents/image%2013.png)
>
> Source: [https://security.apple.com/documentation/private-cloud-compute/vresetup](https://security.apple.com/documentation/private-cloud-compute/vresetup)
>
> → 图源见上链接。

**补充解读**

- **硬件**：仅 Apple Silicon；两台已实测的机器见 [0.3](#03-宿主机环境与工具链含-step-1)。
- **软件**：只针对 cloudOS 26.1 (23B85) + iOS 26.1 (iPhone17,3, 23B85) 这一组合。

**如果你只是想「拥有一台能跑的虚拟 iPhone」，而不是复现研究过程**：
强烈建议直接用 **[Lakr233/vphone-cli](https://github.com/Lakr233/vphone-cli)**。它：

- 把「下载固件 → 打补丁 → 恢复 → 启动」打包成一个图形化 **Launchpad**，几乎零命令行；
- 用开源的 **OpenAVB / OpenSEP** 替换 Apple 的 bootrom，**不需要 patch AVPBooter**；
- **SIP 可以保持开启**（只需 `csrutil enable --without debug` + `allow-research-guests enable`）；
- 支持备份、克隆、HTTP/WebSocket 自动化 API；
- 运行时不依赖 Xcode / Python / Homebrew。

它明确把本 writeup 列为致谢来源，可以理解为本文所讲内容的「工程化重做版」。
**想学习原理 → 读本文；想快速用 → 上 vphone-cli。**

**vphone-cli 对应实现**（详见 [17.3](#173-固件准备fw-prepare--firmware-preparation) 与 [17.14](#1714-与本文手工流程的逐条对照--writeup-vs-implementation)）

原文的兼容性是「我试过这两台机器能用」，vphone-cli 把它变成了一张**会更新、会拒绝**的表。

**（1）cloudOS 侧的窗口变了**

原文锁定的 cloudOS 是 `26.1 (23B85)`。vphone-cli 现在能用的是 **`26.4-23E5207q`（beta）及更早**，
因为 Apple 在之后的版本里删掉了 `vphone600ap`：

| cloudOS | `vresearch101ap` | `vphone600ap` |
| --- | --- | --- |
| `26.4-23E5207q` (beta) | 有 | **有** |
| `26.4-23E244` 及之后全部 | 有 | 无 |

`vphone-cli fw catalog` 可以查当前目录里哪些组合可用；`fw prepare` 会在解包前拒绝
没有 `vphone600ap` 的镜像。

**（2）iPhone 侧反而变宽了**

原文只能配 iPhone 16（iPhone17,3）。vphone-cli 已经验证过的组合包括
iPhone 17 Pro / Pro Max、iPhone 18 Pro，以及多款 iPad（iPad mini A17 Pro、
iPad A16、iPad Air M3、iPad Pro M4/M5）。差别在于原文的偏移是写死的，
vphone-cli 是语义定位，所以能跟着固件走。

**（3）一条原文没有的硬限制**

iPhone18,1 和 iPhone18,2 在 iOS 27 上**必须**开 `kernel-boot-shared_region_size`。
原因是它们的 dyld shared cache 记录了 `sharedRegionSize = 0x185804000`
（27.0.1），比 cloudOS 26.4 内核的 `0x180000000` 共享区还大，即使把 slide 去掉也映射不进去，
launchd 会在首次启动 panic 并报 `Library not loaded: /usr/lib/libSystem.B.dylib`。
iPhone17,3 的 27.0.1 cache 是 `0x17D508000`，塞得下。

这条限制在工程实现里是**前置拒绝**：`cfw install` 会拿 cache 的实际尺寸去比内核的共享区，
放不下就直接说原因，不会留下一个一启动就 panic 的 guest。

**（4）宿主侧**

原文说「预计任何支持 `pccvre` 的机型都可以」。vphone-cli 的硬要求是
**实体 Apple Silicon Mac + macOS 15 或更新**，并且明确指出
**PV=3 guest 不能在 macOS 虚拟机里嵌套运行**。这一条原文没有写，
但确实是很多人在 VM 里试然后卡住的原因。

---

## 15. 在 Sequoia 上启用触摸交互 / Enabling Touch Interaction on Sequoia

**原文 · 译文（逐行对照）**

> **Enabling Touch Interaction on Sequoia**
>
> → **在 Sequoia 上启用触摸交互**
>
> Unlike Tahoe version 26, touch interaction is not possible using only the VZVirtualMachineView object, so it was necessary to override the mouse event functions.
>
> → 和 Tahoe（版本 26）不同，在 Sequoia 上**只靠 `VZVirtualMachineView` 对象无法实现触摸交互**，所以必须**重写鼠标事件函数**。
>
> [ScreenSharingVNC.swift](contents/ScreenSharingVNC.swift)
>
> → 代码见：[`contents/ScreenSharingVNC.swift`](contents/ScreenSharingVNC.swift)

**补充解读**

- 这段代码是本仓库里少数**有完整源码**的部分，值得一读：它把鼠标事件翻译成多点触摸事件
  发给虚拟机。
- 在 Tahoe 26 上不需要它（系统直接支持）。

**vphone-cli 对应实现**（详见 [17.7](#177-虚拟机运行--running-the-vm)「（3）触摸」）

原文这段 `ScreenSharingVNC.swift` 是整套研究里少数公开了完整源码的部分，
vphone-cli 的 `VPhoneVirtualMachineView` 是它的直系后代，连代码注释都引用了它：

```swift
// VPhoneVirtualMachineView.swift:125
/// https://github.com/wh1te4ever/super-tart-vphone-writeup/blob/main/contents/ScreenSharingVNC.swift
private var multiTouchDevice: AnyObject? {
    guard let vm = virtualMachine else { return nil }
    guard let devices = Dynamic(vm)._multiTouchDevices.asObject as? NSArray,
          devices.count > 0 else { return nil }
    return devices.object(at: 0) as AnyObject
}
```

拿触摸设备的方式一样，都是 `Dynamic(vm)._multiTouchDevices` 取第一个。
但它没有停在「重写鼠标事件」，而是把这套重写扩成了一个完整的输入模型：

| 宿主输入 | guest 侧行为 |
| --- | --- |
| 左键点击 | 单指触摸（phase 0 按下 → 3 抬起） |
| 右键 | iPhone：触摸长按（模拟 iOS 的上下文菜单）；iPad：真正的第二键 |
| 滚轮 / 触控板双指 | 合成一根手指拖动，或走原生 `input.scroll`（带 AppKit 动量淡出） |
| 双指捏合 | 两根合成手指做 pinch |
| 拖文件进窗口 | 文件传输进 guest |
| ⌘C / ⌘V | 走 guest 剪贴板 |
| Esc | iPhone：返回手势；iPad：真正的 Esc |

两个原文没有提到、但实测必须处理的细节：

- **触摸更新被限制在约 70 Hz**。每帧都发会让 guest 里的触摸变成抖动；
- **翻页与滚动的边界处理**。guest 会把手指钳在屏幕边缘，一旦钳住，
  继续累加的位移就丢了，所以代码里维护了 `scrollOverflow` 在回到屏内时补回去。

原文第 13 节提到「Sequoia 上只靠 `VZVirtualMachineView` 无法触摸」，
vphone-cli 的做法确实是重写 `mouseDown` / `mouseDragged` / `mouseUp` 一族，
不过它同时用 `_VZUSBTouchScreenConfiguration` + `_setMultiTouchDevices` 把触摸设备挂上去，
两者配合才成立。

---

## 16. 项目源码 / Project SRC

**原文 · 译文（逐行对照）**

> **Project SRC**
>
> → **项目源码**
>
> - https://github.com/wh1te4ever/super-tart-vphone
>
> → - [wh1te4ever/super-tart-vphone](https://github.com/wh1te4ever/super-tart-vphone)

**补充解读**

- 该仓库里有本文所有步骤对应的**可运行脚本**（`CFW/*.py`、`boot_rd.sh`）与 `GUIDE.md`。
- 注意：**仓库现已归档（只读）**，最后一次提交为 2026-02-25。

**vphone-cli 对应实现**

原文在这里只列了一个仓库地址，而且该项目**已于 2026-02-25 归档**。
vphone-cli 是它之后仍在维护的实现，两者的关系可以直接用目录对上：

| 原文的项目结构 | vphone-cli 的对应 |
| --- | --- |
| `CFW/patch_fw.py`、`patch_fw.py` 里的补丁函数 | `VPhoneExecutable/VPhoneCommand/FirmwarePatcher/`（Swift，9 个 patch set） |
| `CFW/fix_boot.py` | `VPhoneCommand/Firmware/VPhoneCustomFirmwareInstaller.swift` |
| `CFW/get_rd.py` + `boot_rd.sh` | 不再需要（改为宿主机离线挂载磁盘） |
| `CFW/get_fw.py` | `VPhoneCommand/Firmware/VPhoneFirmwarePreparer.swift` |
| 改造过的 `super-tart`（`VM.swift`） | `VPhoneExecutable/VPhoneVirtualization/` |
| 改造过的 `libirecovery` | `VPhoneCommand/VPhoneRestore/MobileRecoveryCore`（vendored） |
| `CFW/custom_26.1/*.plist` | 由 `fw prepare` 在 Swift 里重新生成 |
| `contents/ScreenSharingVNC.swift` | `UI/VirtualMachine/VPhoneVirtualMachineView.swift` |
| `GUIDE.md` 里的越狱步骤 | 移出项目范围（见[第 13 节](#13-第二次启动尝试--second-boot-attempt)） |

规模上也不是一个量级：原文的产出是几个 Python 脚本加一份 GUIDE，
vphone-cli 是 712 个 Swift 文件、213 个 C / Objective-C 文件，
运行时不需要 Xcode、Python 或 Homebrew，
产出是一个签名过的 `VPhone.bundle`。

拆解细节见新增的[第 17 节](#17-vphone-cli-实现剖析--implementation-deep-dive)，
如果只想尽快用起来，[17.15](#1715-复现命令速查--command-cheat-sheet) 有命令速查。

---

## 17. vphone-cli 实现剖析 / Implementation Deep Dive

**原文 · 译文（逐行对照）**：本节为本文新增，原 writeup 无对应内容。

**中文解读**：前面 16 节讲的是「一个人怎么用 Python 脚本 + 改造过的 tart 把虚拟 iPhone 拼出来」。
本节讲的是**同一件事被工程化之后长什么样**，也就是 [Lakr233/vphone-cli](https://github.com/Lakr233/vphone-cli)
的源码实现。它不是本文的替代品，而是本文的**答案对照表**：你在这里能读到「原作者手工敲的
那条 `patch_fw.py`，在工程实现里对应哪个文件、用什么策略重新定位、为什么不再硬编码偏移」。

> 本节所有结论均来自对 vphone-cli 仓库的**直接读码**，基线为 commit `aa76eef8`
> （2026-10-21），Launchpad 版本 `2.10.1`。仓库在持续演进，若与最新代码有出入，
> **以仓库为准**（和本文其他部分的约定一致）。

### 17.1 一句话结论与规模 / Scale

**中文解读**：

一句话：**vphone-cli 把本文的「一次性脏活」重写成了一个零运行时依赖的 Swift 产品，
并且把其中最脆弱的环节（硬编码文件偏移）全部换成了「按语义重新定位」。**

先看规模，这决定了它不可能靠读几个文件就理解：

| 指标 | 数值 | 说明 |
| --- | --- | --- |
| Swift 源文件 | **712** 个 | 约 174,000 行 |
| C / Objective-C / 头文件 | **213** 个 | 恢复后端（vendored）+ guest 侧 hook dylib |
| Markdown 文档 | **149** 个 | `Research/` 下每个补丁都有独立研究记录 |
| Python 文件 | **0** 个 | `git ls-files '*.py'` 返回空，这是仓库的**常驻检查项** |
| 顶层模块 | 5 个 | `VPhoneExecutable` / `VPhoneKit` / `VPhoneDaemon` / `VPhoneGuestComponents` / `VPhoneLaunchpad` |

对比本文：本文的产出是 `CFW/*.py` + `boot_rd.sh` + 一份 GUIDE；vphone-cli 的产出是
一个**签名过的 `VPhone.bundle`**，运行时不需要 Xcode、Python 或 Homebrew。

**为什么「零 Python」值得单列**：本文的 `patch_fw.py`、`fix_boot.py`、`get_rd.py` 都依赖
`pyimg4`、`requests` 等第三方包，而依赖链本身会漂移。vphone-cli 把 `pyimg4` 换成
`libimg4-spm`、把 keystone 汇编器换成预生成常量 + Capstone 校验、把 `ipsw` 的 AEA/IM4P
功能换成自研命令（`fw aea-key`、`fw im4p-create`），目的是让「一个 bundle 拷到别人的
干净 Mac 上就能跑」。

### 17.2 进程与产物结构 / Processes and Bundle Layout

**中文解读**：

**（1）为什么拆成两个可执行文件**

这是本文完全没有的一层设计，也是理解 vphone-cli 的入口。本文的做法是把
`AVPBooter` 拷进虚拟机目录再打补丁，宿主侧只需要「关 SIP / 关 AMFI」；vphone-cli
则要面对一个更严格的现实：**带 Apple 私有 entitlement 的 ad-hoc 签名二进制，amfid 会直接拒绝，
内核在 exec 阶段就把它杀掉**。

于是它做了**二进制拆分**：

| 二进制 | entitlement | 职责 |
| --- | --- | --- |
| `vphone-cli` | **无** | 参数解析、编排。在任何 Mac 上都能启动 |
| `vphone-vm` | **6 项 entitlement**（含 3 项 `com.apple.private.*`） | 唯一持有虚拟化权限的进程，跑 `NSApplication` + `VZVirtualMachine` |
| `vphone-escalator` | 无（需 root） | 只负责 AMFI cdhash 准入，见 [17.11](#1711-安全模型与权限边界--security-model) |

`vphone-vm` 的 entitlement 清单（`VPhoneExecutable/VPhoneVirtualization/Resources/VPhoneVirtualization.entitlements`）：

```
com.apple.private.bmk.allow
com.apple.private.virtualization
com.apple.private.virtualization.security-research
com.apple.security.get-task-allow
com.apple.security.virtualization
com.apple.vm.networking
```

关键点：`com.apple.private.virtualization.security-research` 就是「允许运行 PCC 研究用
虚拟机（PV=3）」的那一项。`vphone-cli` 自己**一项都没有**，所以它永远能启动，
也就永远能在 amfid 拒绝 `vphone-vm` 时**把原因和解决办法打印出来**，
这解决了本文读者最痛苦的一类问题：`Killed: 9` 且没有任何输出。

`vphone-cli` 的 `main.swift` 因此只有 40 行，核心逻辑是：

```swift
// VPhoneExecutable/VPhoneCommand/VPhoneCommand/main.swift:23-37
let command = try VPhoneCommand.parseAsRoot()
switch command {
case let boot as VPhoneBootCommand:
    // `boot` 不是本进程的活，交给 vphone-vm
    let status = try VPhoneGuestLaunchPlanner().run(boot.bootArguments)
    if status != 0 { throw ExitCode(status) }
default:
    var runnable = command
    try runnable.run()
}
```

注意 `VPhoneBootCommand` 的定义在 **`VPhoneCoreKit`** 里，被 `vphone-cli` 和 `vphone-vm`
**共享同一份声明**。这样 `vphone-cli boot --help` 与 `vphone-vm --help` 不可能漂移，
`--dfu` 之类的语义天然一致。这是本文的「手工改 super-tart」无法保证的一致性。

**（2）`VPhone.bundle` 的目录布局**

```
VPhone.bundle/Contents/
├── MacOS/                       # 宿主机程序，只在 Mac 上跑
│   ├── vphone-cli
│   ├── vphone-vm
│   ├── vphone-escalator
│   └── libswiftCompatibilitySpan.vphone.dylib
├── Frameworks/
│   └── VPhonePatchKit.framework # 补丁 API（外部 patch set 也链接这一份）
├── Helpers/
│   └── VPhoneLocation.app       # 读宿主机定位的辅助 app（locationd 只认 .app 内的客户端）
└── Resources/
    ├── OpenAVB/OpenAVB.bin      # 开源 boot ROM
    ├── OpenSEP/OpenSEP.bin      # 开源 SEP ROM
    ├── patches_presets/*.plist  # 预置 patch 预设
    └── guest-resources/         # 装进虚拟机的全部内容，绝不在 Mac 上执行
        ├── vphoned, vphoned-tvos, vphoned.plist
        ├── launchdhook-vphone.dylib, SystemHook-vphone.dylib
        ├── libcamfix.dylib, libvcamcaptured.dylib, libmisfix.dylib
        ├── libhapticsfix.dylib, libbatteryhealthfix.dylib, libdevicehubfix.dylib
        ├── libdevicename.dylib, libprefsfix.dylib, libsigninfix.dylib
        ├── libvphonegyro.dylib, libvphoneaccelerometer.dylib, libvphoneattitude.dylib
        ├── libAppleParavirtCompilerPluginIOGPUFamily.dylib
        └── VPhoneVirtIOSound.driver
```

`Contents/MacOS` 与 `guest-resources` 的分界线是**硬约束**，由构建期脚本
`Build/ValidateBundle.sh` 强制检查：guest 目录里出现非 iOS 平台的 Mach-O 就构建失败；
`Contents/MacOS`、`Contents/Frameworks`、`VPhoneLocation.app`、`guest-resources`
之外出现 Mach-O 也失败。

**（3）构建与签名流水线**

`Build/StageBundle.sh` 是唯一产出 bundle 的地方，它做的事值得记住：

1. 用 `xcodebuild` 依次构建 4 个子工程：`VPhoneRestore`（C 恢复后端）、`VPhoneCommand`
   （CLI）、`vphoned`（**iOS SDK**）、`VPhoneEscalator`（**arm64e**）；
2. 用 `make -C VPhoneGuestComponents all` 编译 15 个 guest dylib 与 1 个音频驱动；
3. 检查 `vphoned` 没有引入 iOS 26 缺失的 `_swift_initBorrow` 符号（否则 guest 里会 dyld 崩溃循环）；
4. 用 `vtool -set-build-version tvos` 从 `vphoned` 复制出 `vphoned-tvos`；
5. **逐个二进制 ad-hoc 签名，并且只带自己的 entitlement**：
   `vphone-vm` 带虚拟化 entitlement，`vphoned` 带 daemon entitlement，
   `vphone-cli` / `vphone-escalator` **不带**；
6. 最后对整个 bundle 再签一次，然后调 `ValidateBundle.sh` 验收。

> 这一条对应本文 0.3 节的「关闭 SIP + AMFI」。vphone-cli 把它降级为：
> **SIP 保持开启，只放宽调试限制**（`csrutil enable --without debug` +
> `csrutil allow-research-guests enable`），再用 allowlist 精确放行那一个二进制。
> 安全性上有本质区别，见 [17.11](#1711-安全模型与权限边界--security-model)。

### 17.3 固件准备：`fw prepare` / Firmware Preparation

**中文解读**：

本文第 4、5 节讲的是「下载 iPhone 16 与 cloudOS 26.1 固件 → 解包 → 按需挑组件 → 拼 BuildManifest」。
vphone-cli 把这一步做成 `fw prepare`，实现集中在
`VPhoneExecutable/VPhoneCommand/VPhoneCommand/Firmware/VPhoneFirmwarePreparer.swift`。

**（1）混合固件的来源分配**

和本文一样是**三源混合**，但工程实现把「哪个组件来自哪个身份」写成了明确规则：

| 来源 | 身份 | 提供什么 |
| --- | --- | --- |
| PCC `vresearch101ap` | `ComputeModule14,2` / board `0x90` | 引导链（LLB / iBSS / iBEC / iBoot）、SPTM、TXM |
| PCC `vphone600ap` | `iPhone99,11` / board `0x91` | DeviceTree、SEP 固件、KernelCache、RecoveryMode |
| iPhone 17,3 | `iPhone17,3` / `d47ap` | OS 镜像、trust cache、文件系统 |

为什么必须这么分，`Research/Firmware/firmware_manifest_and_origins.md` 写得很清楚：

- 虚拟机在 **DFU 模式**下以 `vresearch101ap`（BDID `0x90`）出现，所以
  **BuildManifest 的签名身份必须是 vresearch101ap**，否则 TSS/SHSH 拿不到；
- 但**运行时组件必须用 vphone600ap**：只有它的 DeviceTree 把 MKB 的 `dt` 置为 1
  （允许无系统 keybag 启动），SEP 固件也和 vphone600 的设备树匹配。

本文是靠「先解包 cloudOS，再把几个目录 copy 进 iPhone 恢复目录」来做的；vphone-cli 的
`fw prepare` 则是在一个**一次性临时目录**里完成解包与合并，全部成功后才整体移入 VM 目录，
中途失败不会留下半个恢复树。

**（2）拒绝无效组合**

`fw prepare` 会在解包前就拒绝「不可能启动」的 cloudOS。这是本文没有的一层保护：
本文读者如果拿 cloudOS 26.4 之后的版本去试，会在 restore 阶段才炸；vphone-cli 直接说
「这个 cloudOS 里没有 `vphone600ap`」。

实测记录（`Documents/Guides/compatibility.md`）显示：**`26.4-23E5207q`（beta）是最后一个
仍包含 `vphone600ap` 的 cloudOS**。之后的版本只剩 PCC 板与 `vresearch101ap`，
`SupportedProductTypes` 里没有 `iPhone99,11`，也没有
`kernelcache.*.vphone600` / `sep-firmware.vphone600` / `DeviceTree.vphone600ap`。

| cloudOS | `vresearch101ap` | `vphone600ap` |
| --- | --- | --- |
| `26.4-23E5207q` (beta) | 有 | **有** |
| `26.4-23E244` 及之后全部 | 有 | 无 |

> 这一点直接解释了本文 14 节的「只对 26.1 有效」为什么在工程实现里变成了
> 「只对 26.4 beta 及更早有效」，因为中间 Apple 删掉了 vphone600ap。

**（3）GPU 驱动从哪来：一台用完即删的临时虚拟机**

这是本文第 12 节「实现 Metal」的工程化答案，也是整套流程里最巧妙的一步。

本文的做法是：手动从 cloudOS 恢复目录里翻出
`AppleParavirtGPUMetalIOGPUFamily.bundle`，拷进 guest 的 `/System/Library/Extensions`。

vphone-cli 的做法是**自动复现这个过程**：

1. 建一台**无网络设备**的临时虚拟机；
2. 启动到 DFU；
3. 用自研的 in-process restore 后端恢复所选 cloudOS IPSW；
4. 把恢复出来的 System 卷**只读挂载**，取出 GPU bundle；
5. **删掉临时虚拟机**；
6. 把 bundle 按 cloudOS build 缓存到 `~/.vphone/gpu-drivers/<version>_<build>/`。

缓存的意义：同一 cloudOS 上的后续虚拟机**跳过这次临时恢复**（省一次完整 restore）。
也可以用 `--gpu-driver-bundle /path/to/....bundle` 直接复用已有的。

还有一个细节：cloudOS 的 bundle 里**没有**
`libAppleParavirtCompilerPluginIOGPUFamily.dylib`。没有它，`MTLCompilerService` 会反复
abort、`backboardd` 报 Metal 编译中断，**窗口全黑但 vphoned 能连上**。这正是本文 12 节
里「黑底设置页、一操作就 respring」的现代版本。vphone-cli 的解法是：把编译器插件
**用源码重新实现并编进 bundle**
（`VPhoneGuestComponents/GraphicLoader/main.mm`，来自 0xjohnnydev 的 metal-patch），
`fw prepare` 时再把它拷进暂存的 GPU bundle。

### 17.4 固件打补丁：`fw patch` / Patching the Boot Chain

**中文解读**：本节是本文第 6、8 节（打补丁 AVPBooter / 给固件组件打补丁）的工程化对照，
也是整个项目**工程含量最高**的部分。

**（1）流水线顺序**

`FirmwarePipeline.swift` 第 5 行就把顺序写死了，和本文的手工顺序一致：

```
AVPBooter → iBSS → iBEC → LLB → TXM → Kernel → DeviceTree
```

每个组件是一个 `ComponentDescriptor`，声明：文件在恢复树里的**搜索 glob**、
加载方式（IM4P 还是裸数据）、以及**一组 patcher 工厂**。这比本文的
`patch(0x9D10, ...)` 多了一层抽象：同一个组件可以挂多个 patcher 顺序执行。

**（2）`FirmwareOriginals/`：为什么补丁要「每次从原始字节重打」**

这是本文没有、但很关键的设计。`FirmwarePipelineOriginals.swift` 在**第一次**打补丁前，
把每个引导链组件的**未修改字节**存进 VM 目录下的 `FirmwareOriginals/`；之后每次
`fw patch` 都从这份原始字节重新开始。

为什么必须这样：

- 本文的脚本是「就地改」，跑第二次会**在已打补丁的字节上再找一次模式**，
  轻则找不到、重则改错位置；
- vphone-cli 允许你随时开关单个补丁（见下文 patch set），关闭时必须能**回到原始字节**，
  而不是「反向应用一遍补丁」，后者对分支类补丁几乎不可能正确。

两个组件**不参与**这套机制（`restorable = false`）：`Filesystem`（Cryptex 镜像）
与 `Manifest`（BuildManifest 哈希）。原因写在代码注释里：它们改的是「别的步骤产出的文件
所对应的哈希」，把 manifest 单独还原会描述一个已经不存在的树。

顺带一提，`fw prepare` 会主动**删除**上一个固件留下的 `FirmwareOriginals/`。理由很实在：
`AVPBooter*.bin` 的文件名**不含版本号**，如果不删，下一次 `fw patch` 会把**上一个固件**的
字节补完再覆盖到新固件上。

**（3）Patch Set / Preset / Selection / Plan / Receipt 五层模型**

这是 vphone-cli 相对本文最大的一次架构升级。本文的补丁是「脚本里按顺序调用的一串函数」，
**没有名字、没有开关、没有记录**。vphone-cli 把它拆成五层：

| 层 | 文件 | 回答什么问题 |
| --- | --- | --- |
| **Patch Set** | `FirmwarePatcher/PatchSets/*.swift` | 「这个 bundle 声明了哪些补丁？」（编译进二进制，共 **9 个 set**） |
| **Preset** | `Resources/patches_presets/*.plist` | 「哪一套组合是推荐配置？」（随 bundle 发布，共 **4 个**） |
| **Selection** | `<VM>/PatchSelection.plist` | 「这台 VM 的机主改过哪些勾？」（**只记录被改动的项**） |
| **Plan** | `<VM>/PatchPlan.plist` | 「上一次 `fw patch` 实际解析出什么？」（含 iOS/cloudOS 版本、不适用的补丁） |
| **Receipt** | `<VM>/PatchReceipt.plist` | 「guest 里现在真正生效的是什么？」（按 part 分组） |

九个 patch set 及其职责：

| Set 标识 | 内容 |
| --- | --- |
| `bootchain` | AVPBooter / iBSS / iBEC / LLB / TXM |
| `kernel.base` | 内核基础补丁（APFS、启动校验、dyld 策略、sandbox） |
| `kernel.cfw` | 内核 CFW 补丁（调试器、sandbox 扩展、内存） |
| `kernel.hypervisor` | `hv_vmm_present` 隐藏（成对声明） |
| `kernel.frida` | Frida Stalker 放宽（默认关闭） |
| `devicetree` | 设备树：板型呈现、相机几何、身份重写、iPad 适配 |
| `guest.system` | guest 用户态文件与 Mach-O（launchd、seputil、mobileactivationd…） |
| `guest.display` | dyld shared cache 的显示相关补丁 |
| `guest.identity` | Preboot 设备树身份重写 |

**补丁标识符的命名规则**（这条规则在 `Skills/authoring-patch-sets/SKILL.md` 里有完整说明）：
形如 `<target>-<stage>-<what>`，比如：

```
kernel-boot-apfs_root_snapshot        内核 · 引导必需 · APFS 根快照
system-seputil-boot-gigalocker_uuid   seputil · 引导必需 · gigalocker UUID
dyld-boot-maxslide                    dyld · 引导必需 · slide 上限
devicetree-cfw-ipad_product           设备树 · CFW · iPad 产品信息
```

当前 bundle 里**声明的补丁共 158 个**。一个声明可以覆盖多个写入点：比如
`kernel-boot-sandbox_ext` 一个标识符覆盖全部 `kernel-boot-sandbox_ext.<index>` 记录，
`kernel-boot-kcall10` 一个标识符覆盖它的 4 条记录。

**四个预置**（`Resources/patches_presets/`）：

| 预设 | 用途 |
| --- | --- |
| `standard` | 默认。**屏蔽 20 个补丁**：2 个 Frida 放宽、3 个 `hv_vmm_present` 隐藏、8 个设备树身份重写、Preboot 身份重写、`mis_trust_auth`、Settings 行隐藏、Software Update 行、tvOS 相关 |
| `experimental` | 不屏蔽任何补丁 |
| `tvos` | 标准 + 两个 TV TPRO 补丁，关掉 iOS 的显示/相机/音频/传感器/GPU 补丁 |
| `tvos-4k` | `tvos` + GPU bundle + OTA userland 替换 |

`standard` 为什么默认关掉 `hv_vmm_present` 隐藏？研究记录写得很直接：**它会让刚恢复的
26.4 guest 直接变砖**。`bluetoothd` 把 `sysctlbyname("kern.hv_vmm_present")` 的答案缓存进
`dispatch_once`，拿到 ENOENT 后缓存 0，芯片选择单例匹配不到虚拟 iPhone 就 NULL 解引用、
crash-loop；`locationd` 又同步阻塞在这个被 throttle 的 XPC 上，
`com.apple.locationd.migrator` 卡住，SpringBoard 等迁移，结果是**黑屏，无 panic**。

> 这条是本文读者最容易踩的坑：本文附录 A 里没有 `hv_vmm_present` 系列补丁，
> 而 vphone-cli 曾经把它设为默认开启，然后被真实故障打了回来。

**（4）漂移检测（drift）**

`FirmwarePatchDrift.swift` 把上面三个记录（selection / plan / receipt）做**纯函数比较**，
回答「我勾的这个补丁，到 guest 里了没有」。它同时定义了**四种投递方式**：

| 投递方式 | 适用补丁 | 机制 |
| --- | --- | --- |
| `fw-patch` | AVPBooter | 文件在 VM 目录里，每次启动读取，重打即可 |
| `update-kernel` | KernelCache | 从 `FirmwareOriginals` 重打，把 IM4P 换进 Preboot，**不擦数据** |
| `restore` | iBSS / iBEC / LLB / TXM / DeviceTree | 只能重新恢复（擦除式） |
| `update-environment` | DSC / guest 可执行文件 / entitlement / plist / Preboot DT | `cfw install` 或环境更新写入 |

这套「改一个补丁需要多重的一步」的建模，是本文完全没有的。
本文读者改一个内核偏移后只能整机重恢复。

**（5）补丁定位：从「硬编码偏移」到「语义锚点」**

这是本节最值得本文读者学习的一点。本文附录 A 的表格是这样的：

| 组件 | 偏移 | 补丁 |
| --- | --- | --- |
| iBSS | `0x9D10` / `0x9D14` | `nop` / `mov x0,#0` |
| kernel | `0x2476964` | `nop` |

vphone-cli **禁止**这种写法。`AGENTS.md` 的「Kernel patcher guardrails」把它列为硬性规则：

> - 绝不在补丁逻辑里硬编码文件偏移、虚拟地址或预汇编指令字节。
> - 所有指令匹配必须来自 Capstone 解码结果（mnemonic / operands / control flow），
>   能语义判断时不要用格式化后的操作数字符串。
> - 所有替换字节必须来自 `ARM64Encoder.encode*` 或 `ARM64` 常量，**绝不写指令字面量**。

它用的定位手法有三类：

1. **字符串锚点 + ADRP+ADD 交叉引用**：找 `"display-scale"` 字面量，回溯加载它的
   ADRP+ADD，再匹配后面的 `ubfx / add / str` 序列；
2. **指令模式 + 上下文打分**：比如 iBSS 的 image4 回调补丁，找 `b.ne` 紧跟 `mov x0, x22`、
   且前 8 条指令内有 `cmp` 的位置；再用「前 64 条内是否有 `movn w22, #-1`」给候选**打分**，
   优先选错误返回路径那个；
3. **调用图相关性**：内核补丁通过 `bl` 调用频次、panic 字符串、XNU 源码语义来确认
   函数身份（比如 `bsd_init` 的 rootvp panic 块 → 唯一的 `call` → `cbnz` → `bl imageboot_needed`）。

具体例子（本文附录 A 的 iBSS 第一条补丁在工程实现里的样子）：

```swift
// FirmwarePatcher/IBoot/BasePatches/IBootPatchImage4Callback.swift
// 找 b.ne + mov x0, x22，且前面 8 条内有 cmp
guard a.mnemonic == "b.ne" else { continue }
guard b.mnemonic == "mov", b.operandString == "x0, x22" else { continue }
let hasCmp = insns[lookback ..< i].contains { $0.mnemonic == "cmp" }
// 打分：前 64 条内出现 movn w22 / mov w22,#-1 的优先
emit(off,     ARM64.nop,     id: "\(component)-boot-image4_callback.bne")
emit(off + 4, ARM64.movX0_0, id: "\(component)-boot-image4_callback.mov")
```

对比本文的 `patch(0x9D10, nop)`：**偏移一个都不写**，但效果完全一致，
而且换固件版本时**自动重定位**。这就是为什么 vphone-cli 能支持 iOS 26.0 到 27.0.1
这么宽的范围，而本文只能锁定 26.1。

还有一个细节值得注意：boot-args 字符串**和本文完全一样**：

```swift
// IBoot/IBootPatcher.swift:30
static let bootArgs = "serial=3 -v debug=0x2014e %s"
```

这说明本文的结论是对的，工程实现只是换了找位置的方法。

### 17.5 恢复固件：`restore` / In-Process Restore

**中文解读**：本文第 7、9 节讲的是「改造并编译 libirecovery → 抓 SHSH → 用
`idevicerestore` 恢复」。vphone-cli 把这一整条链**搬进了自己的进程**。

**（1）不再需要外部工具**

本文需要：

- 改造版 `wh1te4ever/libirecovery`（支持 `vresearch101ap`）；
- Homebrew 或源码编译的 `idevicerestore`；
- `iproxy`（SSH ramdisk 阶段）；
- `python3` + `pyimg4` + `requests`。

vphone-cli 的 `vphone-cli restore` 在**当前进程内**完成恢复，底层是两份 vendored C：

| 目录 | 来源 | 作用 |
| --- | --- | --- |
| `VPhoneRestore/MobileRecoveryCore` | libirecovery | DFU/恢复模式 USB 通信、IMG4 上传 |
| `VPhoneRestore/MobileRestoreCore` | idevicerestore | 恢复流程编排、组件发送、ASR |
| `VPhoneRestore/VPhoneRestore` | 本项目 Swift（约 1,000 行） | 桥接、进度、错误 |

`libimobiledevice` / `libplist` / `libusbmuxd` / `libtatsu` / OpenSSL 这些**没有** vendored，
而是通过 `Lakr233/AppleMobileDeviceLibrary` 提供的预编译静态 xcframework 链接进来。
研究记录算过账：那 5 个库加起来 73,193 行，属于「本项目永远不会读也永远不会改」的代码。

**（2）`openavb.c`：本文没有的一段 DFU 协议**

这是 OpenAVB 带来的**新增实现**，也是本文「打补丁 AVPBooter」在工程实现里的替代路径。

OpenAVB **不携带任何 iBoot 版本信息**，所以它拒绝通过 DFU 接收 iBSS，
除非宿主先把「这张镜像是什么」告诉它。`MobileRestoreCore/Recovery/openavb.c` 在发送 iBSS 之前：

1. 用**只读**报告请求（`0xc0 0x73`，48 字节）探测 ROM 类型。
   **Apple 的 ROM 会让这个请求 stall**，于是什么都不会写；这就是区分两种 ROM 的方法；
2. 从个性化后的 iBSS 构造 112 字节的 `AVPPatchStageHeader`：
   遍历 IMG4 → IM4P → 第一个 OCTET STRING，要求 `bvx2` 压缩，
   用 libcompression 解压，从偏移 `0x300` 读出 iBoot 链接基址
   （vresearch101 是 `0x7006c000`），entry level 2；
3. 打开 stage（`0x41 0x70`）→ 追加数据块（`0x41 0x71`）→ 封口（`0x41 0x72`，0 行）；
4. 再报告一次，要求 stage 0 返回 `{0, 112, sealed}`。

> 这意味着一个连带改动：OpenAVB **只解 LZFSE**，所以 `fw patch` 现在把 iBSS
> 按 LZFSE 写回（和 Apple 原厂一致，Apple 的 ROM 也接受）。
> 本文读者如果自己写脚本重打包 iBSS，这一点必须注意。

**（3）SHSH / 离线恢复**

`vphone-cli restore` 的关键 flag：

| Flag | 作用 |
| --- | --- |
| `--get-shsh` / `-t` | 只抓签名票据，不恢复 |
| `--offline` / `-O` | 离线恢复：就地解密 AEA 镜像，使用缓存的 `.shsh` |
| `--ecid` | 指定 ECID（默认从 bundle 的 `udid-prediction.txt` 读） |
| `-v` / `-vv` / `-vvv` | 工具细节 / guest 串口 / 内部 trace |

在线恢复在进程内取票，等价于本文的 `tsschecker` / `idevicerestore -t`。

**（4）一个明确「不支持」的选项**

`--no-erase` 这个 flag **存在但被拒绝**，而且拒绝理由写进了代码注释：
在 iPhone 27.0 (24A435) + cloudOS 26.4 上实测，就地恢复失败三次：

1. 混合 manifest 没有 Upgrade 身份，系统卷写完后就失败；
2. 即使补一个，cloudOS 的恢复 ramdisk（vphone guest 唯一能启动的 ramdisk）
   仍会重建分区图与系统 keybag，**擦掉 Data 卷**；
3. iPhone 的 update ramdisk 在 cloudOS 恢复内核上 panic（initproc exits）。

保留这个 flag 是为了「让用它的脚本拿到这个原因，而不是一个用法错误」。
这种「把失败实验固化成明确的拒绝」的做法，值得本文读者学习。

### 17.6 CFW 安装：`cfw install` / Guest System Installation

**中文解读**：本节对应本文第 10、13 节，即用 SSH Ramdisk 进去补 Cryptex、装 Sleigh/ElleKit、
打 `fix_boot.py` 那些补丁。vphone-cli 把「进 ramdisk 手工改」换成了
**宿主机离线挂载磁盘**，实现集中在
`VPhoneExecutable/VPhoneCommand/VPhoneCommand/Firmware/VPhoneCustomFirmwareInstaller.swift`
（3,000+ 行）。

**（1）三种模式**

| 模式 | 做什么 | 需要恢复树 |
| --- | --- | --- |
| `full` | 完整安装：Cryptex、GPU bundle、全部 DSC 与 Mach-O 补丁、guest payload | 需要 |
| `environmentOnly` | 只把 bundle 里的 guest 文件（vphoned、dylib、音频驱动）覆盖回原位 | **不需要** |
| `kernelUpdate` | 只换 Preboot 里的 kernelcache IM4P | 不需要 |

`environmentOnly` 的存在理由很有意思：恢复树在 VM 首次启动后就被删了，
所以「给一台正常使用的 VM 更新一个 guest dylib」**没有别的路径**：
通过 API 替换运行中 guest 的文件，改不了已经被守护进程映射的库。

**（2）root 与安全边界**

`cfw install` **需要 root**（`sudo` 或 Launchpad 的 helper）。它面对的是一个
「调用者完全可控、权限 0777」的目录 + 一个「内容可能被 guest 或导入改过」的 `Disk.img`，
所以代码里有一整套防御，值得单列：

- bundle 只按**文件描述符**固定一次，之后**绝不用路径重建**；
- `Disk.img`、恢复树、Cryptex 镜像必须是**调用者拥有的普通单链接文件**，
  只在 root-only 工作目录里挂载**私有副本**；
- guest 卷挂载在 0700 目录内，**所有 guest 读写都走 `VPhoneConfinedDirectory`（相对描述符）**，
  这样 guest 内的符号链接永远无法把 root 指向宿主机路径；
- root 只按描述符交回自己创建的文件，**从不遍历调用者目录去 chown/chmod**。

工作目录默认 `/private/var/tmp`（root 拥有、sticky），用 `mkdtemp` 创建，
名字不可预测。可以用 `--work-parent` 改。

**（3）磁盘操作流程**

1. 用 `lsof` 确认没有别的进程打开 `Disk.img`（`vm stop` 之外，Spotlight / 备份工具也会占）；
2. 在私有工作目录里对 `Disk.img` 做 **APFS clone**，改完再整体 rename 覆盖原文件，
   所以**失败时原盘不动**；
3. `diskutil image attach -noMount` 挂载镜像；
4. 挂载 guest 卷，执行 Cryptex / GPU / DSC / Mach-O / plist 各项补丁；
5. 写完 `PatchReceipt.plist`，卸载、eject。

> 注意 macOS 27 的一个坑：`diskutil image attach` 现在会先打印它合成的 APFS
> container，再打印镜像自己的 disk。代码里专门处理了这一点：取
> `Apple_APFS` 分区的**父盘**，否则后续所有设备引用都会指向 container 而不是镜像。

**（4）Cryptex 与封卷**

本文第 10 节的核心难点是：SSV 把根卷封了，改文件就 panic，所以要用 SSH Ramdisk 把
`SystemOS` / `AppOS` 两个 Cryptex 补进去。vphone-cli 用
`FirmwarePatcher/CryptexFilesystem/` 下的 7 个文件完成同一件事：

- `CryptexFilesystemPatcherAEA.swift`：AEA 解密
- `CryptexFilesystemPatcherDiskImage.swift`：磁盘镜像操作
- `CryptexFilesystemPatcherSealing.swift`：**重新封卷**
- `CryptexFilesystemPatcherManifest.swift`：更新 BuildManifest 里的哈希
- `CryptexFilesystemPatcherGuestPayload.swift`：写入 guest payload
- `CryptexFilesystemPatcherFileOps.swift`：文件级操作
- `CryptexFilesystemPatcherProcess.swift`：流程编排

**（5）`orig-fs` 快照**

本文用 `snaputil -n` 处理根快照，vphone-cli 用 `cfw flip-snapshot`：
离线重命名 APFS 里 `com.apple.os.update-<hash>` 这个快照，
让「封卷校验被 patch 掉的内核」找不到具名根快照，转而从 live volume 起。

两个实现细节：**只改 APFS 校验和验证通过的块内记录**（所以卷上二进制里
相同的字符串不会被误改）；**完全离线**，不挂载、不调 `fs_snapshot` 系统调用。

**（6）可逆：`.bak` 机制**

这是本文完全没有的能力。`patchCopy` 在第一次改动某个 guest 文件时，
把**改动前**的字节存成 `<file>.bak`，之后**永不覆盖**。于是：

- 关闭某个补丁时，可以**真的还原**成原始字节；
- 但只有「本次运行前没有其他补丁碰过这个文件」时才存 `.bak`，
  否则存下来的就是「已打过补丁的字节」，会被误当成原始文件；
- 早于这个机制打补丁的文件没有 `.bak`，代码会**明确报告不可还原**，而不是猜。

### 17.7 虚拟机运行 / Running the VM

**中文解读**：本节对应本文第 3 节（改造 super-tart）和第 15 节（触摸交互）。

**（1）PV=3 硬件模型**

本文的 `vzHardwareModel_VRESEARCH101()` 在工程实现里几乎是**逐行对应**的：

```swift
// VPhoneExecutable/VPhoneVirtualization/UI/VirtualMachine/VPhoneVirtualMachineHardwareModel.swift:19-35
static func createModel() throws -> VZMacHardwareModel {
    let desc = Dynamic._VZMacHardwareModelDescriptor()
    desc.setPlatformVersion(NSNumber(value: UInt32(3)))   // PV=3
    desc.setBoardID(NSNumber(value: UInt32(0x90)))        // vresearch101
    desc.setISA(NSNumber(value: Int64(2)))
    let model = Dynamic.VZMacHardwareModel
        ._hardwareModelWithDescriptor(desc.asObject).asObject as! VZMacHardwareModel
    guard model.isSupported else { throw ... }
    return model
}
```

注释里还写了 entitlement 的门槛（这是本文没有的信息）：

```
default_configuration_for_platform_version(3) 的有效性字节 =
    (entitlements & 0x12) != 0
    位 1 = com.apple.private.virtualization
    位 4 = com.apple.private.virtualization.security-research
```

**（2）设备清单**

`VPhoneVirtualMachine.swift` 组装的配置，可以看成「一台虚拟 iPhone 由哪些虚拟硬件构成」的
完整清单。标 **私有** 的需要 `Dynamic()` 走运行时方法派发：

| 类别 | 配置类 | 备注 |
| --- | --- | --- |
| 平台 | `VZMacPlatformConfiguration` + 私有硬件模型 | machineIdentifier 即 ECID 来源 |
| 引导 | `VZMacOSBootLoader` + **私有** `_setROMURL` | 换 OpenAVB / Apple ROM |
| 显示 | `VZMacGraphicsDeviceConfiguration` | 默认 1290×2796 @ 460 ppi，scale 3.0 |
| 音频 | `VZVirtioSoundDeviceConfiguration` | 输入 + 输出流，宿主音频源/汇 |
| 存储 | `VZVirtioBlockDeviceConfiguration` | `Disk.img` |
| 网络 | `VZNetworkDeviceConfiguration` | nat / bridged / tunnel / none |
| 串口 | **私有** `_VZPL011SerialPortConfiguration` | 双向管道，用于启动检测与 `-v` 日志 |
| 加速器 | **私有** `_VZMacVideoToolboxDeviceConfiguration`、`_VZMacNeuralEngineDeviceConfiguration`、`_VZMacScalerAcceleratorDeviceConfiguration` | 硬解 / ANE / 缩放器 |
| 触摸 | **私有** `_VZUSBTouchScreenConfiguration` + `_setMultiTouchDevices` | 见下文 |
| 键盘 | `VZUSBKeyboardConfiguration` | 可在 manifest 里关掉 |
| 熵 | `VZVirtioEntropyDeviceConfiguration` | |
| 电池 | **私有** `_VZMacSyntheticBatterySource` + `_VZMacBatteryPowerSourceDeviceConfiguration` | 固定 100%、充电中 |
| 调试 | **私有** `_VZGDBDebugStubConfiguration` | 内核 GDB stub，默认系统分配端口 |
| 协处理器 | **私有** `_VZSEPCoprocessorConfiguration` | SEP + `SEPStorage` + SEP ROM |
| 控制通道 | `VZVirtioSocketDeviceConfiguration` | VSOCK，vphoned 走这里 |

**（3）触摸：本文 `ScreenSharingVNC.swift` 的直系后代**

这是本文与 vphone-cli 之间**最直接的血缘关系**，代码注释里就挂着本文的链接：

```swift
// VPhoneVirtualization/UI/VirtualMachine/VPhoneVirtualMachineView.swift:125
/// https://github.com/wh1te4ever/super-tart-vphone-writeup/blob/main/contents/ScreenSharingVNC.swift
private var multiTouchDevice: AnyObject? {
    guard let vm = virtualMachine else { return nil }
    guard let devices = Dynamic(vm)._multiTouchDevices.asObject as? NSArray,
          devices.count > 0 else { return nil }
    return devices.object(at: 0) as AnyObject
}
```

本文第 15 节说「Sequoia 上必须重写鼠标事件函数」，vphone-cli 的
`VPhoneVirtualMachineView` 正是这么做的，并且把它扩成了**完整的输入模型**：

| 宿主输入 | guest 侧效果 |
| --- | --- |
| 左键点击 | 单指触摸（phase 0 → 3） |
| 右键 | iPhone：触摸长按；iPad：真正的第二键 |
| 滚轮 / 触控板双指 | 合成一根手指拖动，或原生 `input.scroll`（带动量淡出） |
| 双指捏合 | 两根合成手指做 pinch |
| 拖拽文件进窗口 | 文件传输进 guest |
| ⌘C / ⌘V | 走 guest 剪贴板 |
| Esc | iPhone：返回手势；iPad：真正的 Esc |

一个很细的工程点：触摸更新被限制在约 70 Hz（`scrollLastSend`），
因为「每帧都发」在 guest 里会让触摸变成抖动。

**（4）OpenAVB / OpenSEP：本文「打补丁 AVPBooter」的终点**

本文第 6 节整节都在讲怎么给 `AVPBooter.vresearch1.bin` 打 DGST 校验绕过补丁。
vphone-cli 的做法是**换掉它**：

| ROM | 替代 | 说明 |
| --- | --- | --- |
| boot ROM | [OpenAVB](https://github.com/owngoal-dev/OpenAVB) | 开源 AArch64 boot ROM，兼容 AVPBooter 契约 |
| SEP ROM | [OpenSEP](https://github.com/owngoal-dev/OpenSEP) | 开源 SEP boot ROM |

两者的意义：**在 iBoot 和 SEPOS 之前运行的代码不再出自 Apple**。

工程上的连带变化：

- OpenAVB **不校验镜像摘要**，所以 `avpbooter-boot-dgst_bypass` 补丁在它上面
  **无事可做**。`fw patch` 会读 manifest 判断 ROM 类型，然后**跳过 AVPBooter 组件**，
  打印 `Boot ROM: OpenAVB, which needs no ROM patch`；
- 但补丁**仍然保持被选中、被规划**，这样 `fw patches` 不会报「漂移」，
  因为机器的行为确实符合这个补丁的意图；
- 两种 ROM **加载同一个已打补丁的 LLB**，所以「用 Apple ROM 恢复的机器」
  可以无缝切到 OpenAVB 启动，**不需要重新恢复**：

```sh
vphone-cli vm config myphone --rom openavb --seprom opensep   # 开源 ROM
vphone-cli vm config myphone --rom apple --seprom apple       # Apple 的 ROM
```

两个 ROM 都以**预编译**形式随 bundle 发布，构建自固定的上游 commit，
可以用 `Build/UpdateOpenAVB.sh` / `UpdateOpenSEP.sh` 重建并**逐字节核对**。
唯一的例外：PCC GPU 恢复用的那台临时虚拟机**保留 Apple ROM**，
因为它恢复的是 Apple 签名的 PCC 固件，OpenAVB 没在这个场景验证过。

### 17.8 guest 控制面：`vphoned` / Guest Control Plane

**中文解读**：本文第 10 节里，作者「为了方便」加了三个开机自启进程：
**bash、dropbear、trollvnc**。vphone-cli 用**一个 root 守护进程 + VSOCK** 取代了这三样。

**（1）进程与传输**

`vphoned` 以 launchd 守护进程运行（`VPhoneDaemon/Configuration/vphoned.plist`）：

```xml
<key>Label</key>            <string>com.vphone.vphoned</string>
<key>ProgramArguments</key> <array><string>/usr/bin/vphoned</string></array>
<key>RunAtLoad</key>        <true/>
<key>KeepAlive</key>        <true/>
<key>ProcessType</key>      <string>Interactive</string>
```

监听两个 **VSOCK** 端口（不是 TCP，不经过 guest 网络栈）：

| 端口 | 用途 |
| --- | --- |
| **1339** | HTTP/1.1 + WebSocket（SwiftNIO），控制面 |
| **1338** | 相机帧数据面（BGRA 帧 + 共享内存 + Darwin 通知） |

启动顺序在 `main.swift` 里是有讲究的，每一步都为了「在某个系统进程读它之前」就位：

```swift
GuestLockScreenIdle.startOnStartup()        // SpringBoard 读锁屏超时之前
GuestMobileGestaltCache.dropIfStaleOnStartup()
GuestVirtualAudioProduct.storeOnStartup()   // audiomxd 请求声音之前
vp_vcam_start()
_ = vp_hid_load()                           // 预解析多指注入符号
GuestIrisinInstaller.refreshBootstrapOnStartup()
GuestAPI.restoreUSBSerialOnStartup()
GuestStaticNames.shared.restoreOnStartup()
GuestLocationdWatchdog.shared.startOnStartup()  // locationd 启动死锁时重启它
```

**（2）157 个 RPC 方法**

按 `GuestAPI+<Area>.swift` 一个文件一个领域组织，共 **157 个方法名**。
完整清单见 `Research/vphoned_http_api.md`，这里按领域概括：

| 领域 | 代表方法 | 能力 |
| --- | --- | --- |
| 设备 | `device.info` `device.snapshot` `device.screen` `device.ioreg` `device.network` | 一次拿全状态 |
| 输入 | `input.tap` `input.swipe` `input.drag` `input.key` `input.type` `input.touch_sequence` | 手势 / 键盘 / 指针 |
| UI | `ui.tree` `ui.element_at` `ui.tap_element` `ui.wait` `ui.ocr` `ui.describe` | 无障碍树 + OCR |
| 应用 | `apps.list` `apps.install` `apps.launch` `apps.terminate` `apps.remove_system` `apps.restore_system` | 含系统 App 移除/还原 |
| 文件 | `files.list` `files.read` `files.write` `files.plist_set` `files.find` `files.chown` | 描述符相对、可 root |
| launchd | `services.list` `services.profile.apply` `services.kickstart` `launchd.setenv` | 服务画像 |
| 进程 | `processes.list` `processes.kill` `memory.jetsam` `memory.pressure` | |
| 日志 | `logs.syslog` `logs.crashes` `logs.crash` | 有界抓取（≤60 s） |
| 显示/音频 | `display.brightness` `display.rotation` `display.auto_lock` `screen.unlock` `audio.volume` | |
| 网络 | `network.capture`（写 pcap）`network.resolve` `network.hostname.set` `network.static_names.set` | |
| 运动 | `motion.gyroscope.set` `motion.attitude.set` | 虚拟陀螺仪 / 姿态 |
| 位置 | `location.set` `location.current` `location.clear` | |
| 钥匙串 | `keychain.list` `keychain.add` `keychain.database` | |
| 设置 | `settings.get/set/delete` `clipboard.get/set/clear` | |
| 快照 | `apfs.snapshots` `apfs.snapshot.delete` | 只允许删 `orig-fs.disabled.rn-*` |
| 环境 | `environment.status` `environment.install` | 热更新 guest dylib |
| Bootstrap | `bootstrap.install` `bootstrap.status` `bootstrap.uninstall` | Irisin |
| Setup | `setup.status` `setup.skip` `setup.settle` | 跳过设置向导 |
| 系统 | `system.reboot` `system.shutdown` `system.respring` `system.uicache` | |
| UDID | `udid.get` `udid.set` `udid.clear` | 改 profile 看到的 UDID |

**危险方法需要显式 `"force": true`**：`processes.kill`、`system.reboot`、
`system.shutdown`、`system.respring`、`apps.uninstall`、`apps.remove_system`、
`services.profile.apply` 等。

**（3）两层访问控制**

`vphoned` 以 **root** 运行，能读写任意 guest 文件、列钥匙串、加载 launch daemon。
它的模型很坦诚：**它无法区分「被代理的连接」和「guest 自己的 VSOCK 客户端」**，
所以两道检查分别做在两层：

- **宿主代理层**：`vphone-vm` 启动时用 `SecRandomCopyBytes` 生成 32 字节 token（hex），
  每次启动都换。可用 `VPHONE_API_TOKEN` 固定（16–256 字符，`A-Z a-z 0-9 - . _ ~`）。
  只读前 16 KiB 请求头、最多等 10 秒，token 可以放在
  `Authorization: Bearer`、`?token=` 查询项、或 `Sec-WebSocket-Protocol: vphone-token.<token>`。
  **常量时间比较**；缺失/错误/超长/畸形一律 401 并直接关闭，**不会**打开 guest 连接；
- **guest 层**：`vphoned` 自己的检查。

**（4）`vphone.sock`：不经过网络的本地控制口**

除了 HTTP API，每台 VM 还有一个 **Unix domain socket**：`<VM bundle>/vphone.sock`。
协议极简：**一行 JSON 进，一行 JSON 出**，并发服务：

```json
{"t":"tap","x":100,"y":200}
{"t":"swipe","fromX":100,"fromY":600,"toX":100,"toY":200}
{"t":"key","name":"cmd+v"}
{"t":"screenshot","path":"/tmp/shot.png"}
{"t":"rpc","method":"input.type","params":{"text":"hello"}}
```

它在 `--no-graphics` 下同样存在，此时 `tap` / `swipe` 走 vphoned 的触摸注入，
截图来自 guest 自身。**这条通道不走 guest 网络**，所以 `tunnel` 模式下照样能用。

> 对比本文：作者装 dropbear + trollvnc 是为了「能操作 guest」。
> vphone-cli 的方案在**不需要 SSH、不需要 VNC、不需要网络**的前提下提供了
> 严格更强的能力（结构化 UI 树、OCR、钥匙串、服务画像）。

**（5）相机：VSOCK 1338 + 共享内存**

`libvcamcaptured.dylib` 与 `libcamfix.dylib` 的组合，把宿主摄像头画面喂给 guest：

1. 宿主 `VPhoneCameraServer` 通过 VSOCK 1338 发 BGRA 帧；
2. `vphoned` 收到后写入**序列号保护的共享内存帧** `vphone-vcam-frame.shm`，
   再 post `com.vphone.vcam.frame`；
3. `libvcamcaptured` 读该帧，供给 camera 客户端。

所有相机运行时文件都在 `/var/mobile/Media/SimulatedCamera`。
注意文档里明确写了：宿主窗口的 **Camera server: connected** 只证明 VSOCK 连上了，
**不证明** camera App 或 hook 装好、加载了、出图了。

### 17.9 guest 侧组件与注入链 / Guest Components

**中文解读**：本文第 13 节装 Sileo、ElleKit，用它们注入 tweak。vphone-cli 的 guest hook
**不依赖任何 bootstrap**：`cfw install` 时就把 dylib 放进 `/usr/lib`，靠自己的注入链加载。

**（1）注入链**

```
launchd
  └─ DYLD_INSERT_LIBRARIES=/usr/lib/launchdhook-vphone.dylib
       └─ 每次 spawn 时判断目标路径
            ├─ App（/Applications/**.app/**, /var/containers/Bundle/Application/**）
            ├─ bootstrap 程序（/var/jb/** 或 .jbroot-*）
            ├─ /usr/libexec/cameracaptured
            └─ /usr/libexec/backboardd
                 ↓ 再注入
            /usr/lib/SystemHook-vphone.dylib
                 ↓ 按目标进程选择
            libcamfix / libvcamcaptured / libprefsfix / libsigninfix /
            libbatteryhealthfix / libhapticsfix / libdevicehubfix /
            libdevicename / libmisfix / libvphonegyro / libvphoneaccelerometer /
            libvphoneattitude
```

（Flutter AOT 重映射不是独立 dylib，它编在 `SystemHook` 里，见下表。）

日志落在 `/var/mobile/Library/Caches/vphone-launchdhook-injection.log`
与 `vphone-launchdhook-spawn.log`，形如：

```
event=inserted+batteryhealthfix child=1586 path=/Applications/Preferences.app/Preferences status=0
```

一个很实际的坑写在代码注释里：**每个进程都会加载这个 hook（包括非 root 进程）**，
所以日志文件必须对所有人可写。第一个写入者总是 root 的 `xpcproxy`，
它创建的 0644 文件会**静默丢弃**所有 mobile 进程的行，看起来像 hook 根本没生效。
所以每次打开都 `fchmod` 成 world-writable。

**（2）各 dylib 的职责**

| 组件 | 修什么问题 |
| --- | --- |
| `launchdhook-vphone` | 注入总闸；同时负责 arm64e X1 指令兼容的 spawn 路径 |
| `SystemHook-vphone` | 按目标进程分发具体 hook；相机/运动守护进程的注入点；内含 Flutter AOT 重映射兼容 |
| `libcamfix` | 前摄查询返回唯一 vcam 设备；给已加载 AVFoundation 的 app 用 |
| `libvcamcaptured` | 在 `cameracaptured` 里供给相机帧 |
| `libbatteryhealthfix` | 电池健康读数在虚拟硬件上不崩 |
| `libhapticsfix` | 无振动硬件的机型不因触觉调用异常 |
| `libdevicehubfix` | DeviceHub 相关界面 |
| `libdevicename` | **设备名锁定**：让 Xcode / `devicectl` 看到 VM 名 |
| `libprefsfix` | 隐藏 Settings 里不该出现的行（Apple Account 横幅、Software Update 等） |
| `libsigninfix` | 抑制 Apple ID / Media & Purchases 登录 sheet（app 进程专用） |
| `libmisfix` | 拦 `libmis` 的签名/安装策略，让 IPA 安装走得通 |
| `libvphonegyro` / `libvphoneaccelerometer` / `libvphoneattitude` | 虚拟陀螺仪 / 加速度计 / 姿态，经 HID provider |
| `libAppleParavirtCompilerPluginIOGPUFamily` | Metal 着色器编译（见 17.3） |
| `VPhoneVirtIOSound.driver` | 音频 HAL 插件（含时钟、格式转换、延迟补偿） |

**（3）服务画像与瘦身**

模板的 setup boot 里会应用 `services.profile.apply`，**关掉约 140 个 launchd job**：
Siri 与智能相关、Find My、Wallet/NFC/SE、Watch、Home、健康、地图后台、
iCloud 与 iMessage 同步、广告与推荐、遥测、配件固件更新、OTA。

实测收益（`Research/Guest/service_trimming.md`）：

| Guest | 进程内存 | 宿主空闲 CPU | 宿主空闲写入 |
| --- | --- | --- | --- |
| iOS 27.0 | −29% | −26% | −60% |
| iOS 26.6.2 | −30% | −39% | −94% |

机制上它不是固件补丁：用的是 `launchctl disable` 写的 override
（`/private/var/db/com.apple.xpc.launchd/disabled.plist`），
**系统卷和 `launchd.plist` 一个字节都没动**，可完整回滚
（vphoned 把禁用的 label 记在 `/var/db/vphoned/service-profile.plist`）。

一个 iOS 27 的坑：launchd 会忽略被板型 jetsam 属性标记
`DisableAllowed = false` 的 job，iOS 27 有 **19 个**这样的守护进程。
`system-jetsamproperties-cfw-disable_allowed` 补丁把这些标记去掉，
**不 patch 任何可执行文件**，只改 plist，并且备份/可还原。

### 17.10 模板、快照与磁盘 / Templates, Snapshots, Disk

**中文解读**：本文读者建一台虚拟机要重跑整条流水线（约 40–60 分钟，几十 GB 读写）。
vphone-cli 的模板机制把它变成「第一次付全价，之后近乎免费」。

**（1）一台虚拟机占多少**

| 项 | 大小 |
| --- | --- |
| `Disk.img`（64 GB 逻辑） | 装完约 18.6 GB，首次启动后约 **20.5 GB** |
| `nvram.bin` | 33.6 MB |
| `SEPStorage` | 512 KB |
| `.vphoned.signed` | 9.3 MB |
| 恢复树（构建期临时） | 约 **11 GB** |
| IPSW 缓存（库外，共享） | 12.3 GB + 1.2 GB |

**（2）模板键**

模板是「一台完整机器，只启动一次（setup boot），之后冻结」。它的标识符是
**规范化描述的 SHA-256 前 12 位十六进制**，规范化描述包含：

```
format=2
device=iPhone17,3
ios=27.0/24A435
cloudos=26.4/23E5207q
preset=standard
bootchain=<补丁计划摘要>
series=2.10
disk=64
trim=standard/1
setup=1
services=trimmed
service-groups=...
removed-apps=...
```

于是「同样的选择」自动命中同一个模板，「不同的瘦身」各自并存，互不干扰。
CPU、内存、屏幕、网络**不在**键里，它们是克隆后再设的。

**（3）离线瘦身三档**

| 档位 | 删什么 | 省 |
| --- | --- | --- |
| `none` | 不删 | 0 |
| `conservative` | `usr/standalone/update` 内容（软件更新 ramdisk、基带固件） | 0.35 GB |
| `standard` | conservative + AirPods/Beats 配对资源 + 表盘 + 未保留语言的 `LinguisticData` | **1.2 GB** |
| `aggressive` | 保留档，**未在 guest 验证前拒绝使用** | — |

关键细节：**删掉的文件一开始不释放空间**。`cfw install` 留下的
`orig-fs.disabled.rn-*` 快照仍然持有所有块，而宿主**即使 root 也删不掉**它。
只有 setup boot 的第一步把它删掉，空间才回来。所以：
**没做 setup boot 的模板不允许 trim；trim 过的模板必须在快照消失后才能冻结/采纳。**

**（4）setup boot 的 7 步**

模板的 setup boot 就是把「每个克隆都应该有的起点」固化下来：

1. 删 `orig-fs.disabled.rn-*` 快照；
2. `setup.skip` 跳过设置向导；
3. `setup.settle` 等首次启动的后台工作稳定（最多 10 分钟，
   因为 installd 会在首次启动展开可移除系统 App）；
4. `apps.remove_system` 移除系统 App（备份留在 guest 里）；
5. `services.profile.apply` 应用瘦身画像（含关掉登录跟进守护进程）；
6. 重启后校验：无残留快照、Setup 完成、画像里的服务没在跑、移除的 App 确实没了、vphoned 有应答；
7. 清掉本次启动固定的设备名，干净关机。

每一步都有 deadline。失败就停 VM、**不记录任何东西**、构建保持未冻结状态。

**（5）克隆为什么几乎免费**

模板和克隆之间是 **APFS `clonefile`** 关系：克隆后首次启动完约只占 **0.35 GB**，
而不是 20.5 GB。代价是 `du` 和 Finder 会把每份都算成全尺寸，
**只有 `df` 的空闲空间才是诚实的指标**。

同样的机制也用于：

- `vm clone`：普通克隆（保留身份，不能同时运行）或 `--new-identity`（新 ECID/UDID/MAC，可并行）；
- `vm rebase`：让两台**分别恢复**的 VM 共享相同块（实测一对约省 8.7 GB）；
- `vm snapshot`：磁盘 + `SEPStorage` + `nvram.bin` **三件套一起**，因为 SEP 的反重放计数器
  在 `SEPStorage` 里，磁盘与它必须匹配，否则下次启动 SEP panic。

**（6）模板的共享密钥代价（重要警告）**

所有从同一模板克隆的机器**共享**：SEP 根密钥、Data 卷密钥、以及恢复和 setup boot 写入的数据。
**一个克隆在原理上可以解密另一个的 Data 卷。**

如果需要独立密钥（比如要登录 Apple 服务做多设备研究），必须用 `--no-template` 创建。

### 17.11 安全模型与权限边界 / Security Model

**中文解读**：本文 0.3 节的做法是「`csrutil disable` + `amfi_get_out_of_my_way=1`」。
vphone-cli 提供了一条**保持 SIP 开启**的路径，这也是它相对本文最实质的安全改进。

**（1）两条路径**

| 路径 | 设置 | 代价 |
| --- | --- | --- |
| **A. 关闭 SIP + AMFI** | `csrutil disable` + `nvram boot-args="amfi_get_out_of_my_way=1 -v"` | 更宽松，全局削弱 |
| **B. SIP 开启 + 放宽调试**（推荐） | `csrutil enable --without debug` + `allow-research-guests enable` + allowlist | 只放宽调试限制 |

两条都**必须**先 `csrutil allow-research-guests enable`（这是 PV=3 的硬要求）。

**（2）`vphone-escalator` 到底做了什么**

这是整个项目里最精巧的一段系统编程。它解决的问题是：

> 一个 ad-hoc 签名、带 `com.apple.private.*` entitlement 的二进制，amfid 会拒绝，
> 内核在 exec 时杀掉。怎么办？

常见答案是**改 amfid 的代码**（下断点、改 `ldrb`），但代码注释里明确写了为什么这不行：

> 两种做法都会让 amfid 里出现一个 dirty、unsigned 的可执行页。在
> `vm.cs_system_enforcement = 1` 的宿主上，内核下次 fault 进这个页会校验、
> 发现没有签名、**把 amfid 杀掉**（`CODESIGNING / "Invalid Page"`）。
> 这个 sysctl 运行时只读，所以「改代码」这条路走不通。

它换了个思路：**用 AMFI 自己已有的功能**。

AMFI 会读 `/Library/Preferences/com.apple.security.coderequirements.plist` 的两个键：

| 键 | 作用 |
| --- | --- |
| `Entitlements` | 一个 code requirement 字符串，**替换** `_restrictedRequirement` |
| `AllowUnsafeDynamicLinking` | BOOL，成为 validator 的 `_shouldUnrestrict`，让 `DYLD_INSERT_LIBRARIES` 重新被接受 |

于是 `Entitlements` 里的 `cdhash H"..."` 就是一份**按二进制粒度的 allowlist**，
由 Apple 自己的代码来评估。

但整个 preference 被一个 BOOL ivar 门控：

```c
_isRunningInternalBuild = (csr_check(CSR_ALLOW_APPLE_INTERNAL) == 0)
```

所以这个工具**只写一个字节**：amfid 堆上那个单例里的 ivar。

- 单例指针在 dyld shared cache 的一个静态槽里，**每个进程地址相同**。
  工具通过解码 `+[AMFIRequirementsManager sharedManager]`、把每个它加载的地址
  和本进程刚创建的单例比对来**找到这个槽**，**不硬编码任何地址**，
  布局变化是「拒绝」，而不是「野写」；
- 目标是 **malloc 出来的堆**：没有 `mach_vm_protect`、没有共享页 COW、
  没有可执行内存，代码签名监控没有可反对的东西；
- ivar 偏移来自**活的 ObjC 运行时**，不是常量。

它需要 root + `task_for_pid`（即 SIP 放宽调试），这和让 amfid 尊重 preference 是同一个前提。

`off` 只移除**这个工具加过的** hash，然后重启 amfid，保留 preference 和其他值。

**（3）必须重复执行**

因为 cdhash **每次签名都会变**，所以 `allow` 是**每次构建都要重跑**的一步，
不是「一台机器做一次」。Launchpad 的做法是：helper 先复查宿主安全设置、
校验 root 拥有的 bundle 和它记录的 cdhash，再以 root 跑那个 bundle 的
`vphone-escalator allow`，然后重跑 preflight。

**（4）`vphone-cli` 如何在没有权限的情况下诊断**

`VPhoneGuestLaunchPlanner` 负责：解析运行中可执行文件旁边的 `vphone-vm`、
检查它的 entitlement、**探测 AMFI**、并在被拒时**打印完整的 allowlist 命令**。
它自己**从不获取 root**。

对比本文：读者遇到 `Killed: 9` 只能自己猜。vphone-cli 把「谁拒绝了我、我该敲什么」
直接印出来。这是二进制拆分的全部意义。

### 17.12 网络四模式 / Networking

**中文解读**：本文没有系统讲网络（只有 SSH ramdisk 阶段的 `iproxy`）。vphone-cli 有四种模式：

| 模式 | guest 怎么上网 | 什么时候用 |
| --- | --- | --- |
| `nat` | Virtualization.framework 内置 NAT | 默认。Mac 没有 VPN，或 VPN 不需要承载 guest |
| `bridged` | 直接挂在 Mac 的某个物理接口上，拿该网段的地址 | guest 要能被局域网其他机器访问 |
| `tunnel` | 由 `vphone-vm` 用**普通 socket** 代理出去 | **Mac 走 VPN 或代理，guest 流量必须跟随** |
| `none` | 不挂网络设备 | guest 必须离线 |

**`tunnel` 模式为什么存在**：`nat` 和 `bridged` 都会让 guest 的包走物理接口出去。
当 VPN 拥有 Mac 的默认路由（WireGuard、WARP，或 Surge / Clash 的 TUN 模式），
这些包**绕过 VPN**：要么明文出去，要么根本出不去。

`tunnel` 模式下 guest 的网卡**实现在 `vphone-vm` 进程里**：它自己应答 DHCP、ARP、网关 ping，
把 guest 的每条 TCP 连接和 UDP 流转成 Mac 上的普通 socket。那些 socket 和任何 App 一样
走 Mac 的路由表，于是 VPN 自然承载它们。**不需要 root、不需要新网卡、不改 Mac 网络设置。**

guest 看到的固定值：

| 项 | 值 |
| --- | --- |
| 地址 | `192.168.127.3/24`（DHCP） |
| 网关 / DNS | `192.168.127.1` |
| MTU | 1500 |
| 到网关的其他连接 | Mac 的 loopback `127.0.0.1`（TCP/UDP） |

限制也说得很清楚：**仅 IPv4**；除转发端口外**只出不进**；ICMP 只到网关
（`ping 1.1.1.1` 会失败，但 TCP/UDP 正常）；吞吐受 `vphone-vm` 单进程限制。

**其他网络能力**（本文都没有）：

- **固定地址**：`vm config --ip`，NAT 模式须在 Mac 共享 NAT 网段内，vphoned 写进 guest；
- **端口转发**：`--forward tcp:8022:22`，实现是宿主侧 `VPhoneTCPForwarder`；
- **MAC 地址**：首次启动随机生成并存入 `config.plist`，之后保持同一身份与同一 DHCP 租约；
- **mDNS**：`--mdns on` 让 guest 响应 `<vm-name>.local`，四种模式都支持；
- **Mac 名解析**：`--mac-name`（默认开）让 guest **立刻**把 Mac 的 `<LocalHostName>.local`
  解析到「guest 到达 Mac 的那个地址」，而不是去和 mDNS 应答赛跑。

### 17.13 Launchpad：图形化的那一层 / The App

**中文解读**：本文完全没有 GUI 层。vphone-cli 的 `vphone-launchpad.app` 把
「下载固件 → 打补丁 → 恢复 → CFW → 首次启动」变成四个按钮。

**（1）组件**

| 组件 | 作用 |
| --- | --- |
| `vphone-launchpad.app` | SwiftUI Mac App |
| `com.vphone.launchpad.helper` | **SMJobBless** 特权 helper，唯一的 root 面 |
| `vphone-launchpad-cli` | 无状态客户端，通过同用户 Unix socket 和运行中的 App 通信 |
| `vphone-launchpad-launcher` | 每台 VM 的 **responsible process** |

**（2）helper 的能力被严格限定**

helper 只做这几件事（`VPhoneLaunchpadHelperProtocol.swift`）：

| 方法 | 作用 |
| --- | --- |
| `installBundle` | 把校验过的发布版装进 root 拥有的 store |
| `allowVirtualMachine` | 跑那个 bundle 的 `vphone-escalator allow`（针对 receipt 固定的 `vphone-vm`） |
| `installCustomFirmware` | 从 store 里跑 `cfw install`（先复查记录的 cdhash） |
| `updateGuestEnvironment` | 同上，`cfw update-environment` |
| `updateKernel` | 同上，`cfw update-kernel` |
| `releaseOrphanedLeases` | 对**调用者自己拥有**的库目录跑 `vm leases --release-orphans` |

**没有通用命令动词**。store 路径是
`/Library/Application Support/vphone-launchpad/Bundles`。

**（3）`vphone-launchpad-launcher` 为什么存在**

Launchpad 通过它**分离启动** `vm launch`（独立 session、disclaim responsibility）。
launcher 作为 app `Contents/MacOS` 里的工具，**归属于 app**，所以：
麦克风、定位这些隐私提示和授权是 **app 的**，**按 Developer ID 签名问一次**，
而不是按每个 ad-hoc 签名的 bundle 构建各问一次。

**（4）系列匹配规则**

Launchpad 和 `VPhone.bundle` 按**系列**匹配，也就是版本号前两段：

- Launchpad `2.10.3` 可以和任意 `VPhone.bundle` `2.10.x` 配合，反之亦然；
- 同系列内的补丁版本**必须双向可互换**；
- 一旦改动 Launchpad 依赖的 bundle 侧任何东西（`vphone-cli` 参数或输出、
  `vphone-escalator`、bundle 布局、receipt），就**必须升 minor 版本**
  并提高 `VPhoneLaunchpadBundleStore.swift` 里的 `minimumBundleComponents`。

### 17.14 与本文手工流程的逐条对照 / Writeup vs. Implementation

**中文解读**：这张表是本节的核心交付物，把本文每一节的「手工动作」映射到工程实现。

| 本文小节 | 手工做法 | vphone-cli 的实现 | 关键差异 |
| --- | --- | --- | --- |
| 0.3 宿主环境 | 关 SIP、关 AMFI、`authenticated-root disable` | `csrutil enable --without debug` + allowlist | **SIP 可保持开启** |
| 0.4 PCC 虚拟机 | 手跑 `pccvre instance create` | 不需要，`fw prepare` 自己建临时 VM 并删掉 | 全自动 |
| 3 改造 super-tart | 改 `VM.swift` 的 `vzHardwareModel_VRESEARCH101()` | `VPhoneVirtualMachineHardwareModel.createModel()` | 代码几乎逐行对应 |
| 4–5 混合固件 | `get_fw.py` 解包 + 手工 copy 组件 | `fw prepare` 在临时目录合并，整体移入 | 三源分配规则固化 |
| 6 打补丁 AVPBooter | `patch(0x..., nop)` | **OpenAVB 替代**，无需补丁；`fw patch` 自动跳过 | 换 ROM 而非打补丁 |
| 7 改造 libirecovery | 编译 `wh1te4ever/libirecovery` | vendored C，编进 `VPhoneRestore` | 无外部工具 |
| 8 打补丁固件组件 | 硬编码偏移 + `vbindiff` 人工核对 | **语义锚点定位** + `PatchRecord` 记录偏移与前后状态 | 换版本自动重定位 |
| 9 恢复固件 | `idevicerestore` + `tsschecker` | `vphone-cli restore`（进程内） | 在线取票，`--offline` 可选 |
| 10 SSH Ramdisk 修 Cryptex | 手动 `boot_rd.sh` + `iproxy` + scp | `cfw install` 离线挂载磁盘 | **不需要 ramdisk / SSH** |
| 11 首次启动尝试 | 看串口日志猜 panic | `-v` / `-vv` / `-vvv` + 结构化错误 | |
| 12 实现 Metal | 手工装 GPU bundle + 缺 dylib 就黑屏 | 自动从 cloudOS 提取 + 编译器插件用源码重建 | 不再依赖未公开 dylib |
| 13 第二次启动 + 越狱 | 装 Sileo / ElleKit / dropbear / trollvnc | 不需要；`vphoned` + RPC 取代 | 无 bootstrap 依赖 |
| 14 兼容性 | 锁死 26.1 (23B85) | 26.0–27.0.1 多组合，有兼容矩阵 | 靠动态定位而非固定偏移 |
| 15 Sequoia 触摸 | `ScreenSharingVNC.swift` 重写鼠标事件 | `VPhoneVirtualMachineView` 完整输入模型 | 代码注释直接引用本文 |
| 附录 A 补丁速查 | 偏移表 | 158 个**有名字、可开关、有版本门**的补丁声明 | 从「表」变成「模型」 |

### 17.15 复现命令速查 / Command Cheat Sheet

**中文解读**：如果你只是想用起来，这一节够了；想理解原理，回到前 16 节。

```sh
# ── 宿主准备（只需一次，需进 macOS 恢复模式）
csrutil enable --without debug
csrutil allow-research-guests enable
# 重启后，放行当前构建的 vphone-vm（每次重新构建都要重跑）
sudo "<bundle>/Contents/MacOS/vphone-escalator" allow "<bundle>/Contents/MacOS/vphone-vm"
"<bundle>/Contents/MacOS/vphone-escalator" status
"<bundle>/Contents/MacOS/vphone-cli" host preflight

# ── 一条命令建机（模板化，第二次起近乎瞬时）
vphone-cli vm create myphone \
  --iphone-source /path/to/iPhone17,3_Restore.ipsw \
  --cloudos-source /path/to/cloudOS.ipsw

# ── 手动分步（研究用）
vphone-cli vm new myphone
vphone-cli fw prepare myphone --iphone-source ... --cloudos-source ...
vphone-cli fw patch myphone
vphone-cli vm launch myphone --dfu &
vphone-cli restore myphone
vphone-cli vm stop myphone
vphone-cli cfw install myphone          # 需要 root
vphone-cli vm launch myphone

# ── 补丁管理
vphone-cli fw patches myphone           # 看声明、预设、漂移
vphone-cli fw set-patches myphone --block kernel-cfw-debugger
vphone-cli fw patch myphone             # 重新解析并落盘
vphone-cli cfw update-environment myphone   # 只更新 guest 侧（需要 root）
vphone-cli cfw update-kernel myphone        # 只换 Preboot 内核（需要 root）

# ── 日常运维
vphone-cli vm list / info / config / clone / rename / delete
vphone-cli vm snapshot create myphone clean --note "fresh"
vphone-cli vm snapshot revert myphone clean
vphone-cli vm rebase second --onto first --dry-run
vphone-cli vm template list / show / trim / setup / adopt / delete
vphone-cli vm export myphone --out myphone.tzst
vphone-cli vm import myphone.tzst --name restored

# ── 自动化
vphone-cli vm launch myphone --api-listen 127.0.0.1:8765
# 输出里有 [api] token: …
curl -H "Authorization: Bearer $TOKEN" http://127.0.0.1:8765/v1/health
vphone-launchpad-cli guest rpc myphone device.info
```

**目录位置**：

| 路径 | 内容 |
| --- | --- |
| `~/.vphone/machines/<name>/` | 单台 VM（磁盘、`config.plist`、补丁工作文件） |
| `~/.vphone/machines/.templates/<id>/` | 模板（`vm list` 不显示，但确实占空间） |
| `~/.vphone/ipsws/` | 远程 IPSW 缓存，所有 VM 共享 |
| `~/.vphone/gpu-drivers/<version>_<build>/` | 从 cloudOS 提取的 GPU bundle |
| `~/.vphone/patchsets/` | 导入的外部 patch set |
| `~/.vphone/patches_presets/` | 用户自定义预设 |

`VPHONE_ROOT` 可整体重定位；`VPHONE_LIBRARY_ROOT` 只重定位库。

### 17.16 本节结论 / Takeaways

**中文解读**：

1. **本文的结论是对的，方法不是唯一的。** boot-args 字符串
   `serial=3 -v debug=0x2014e %s`、三源混合的组件分配、TXM/kernel 的补丁点，
   在 vphone-cli 里全部复现，但**没有一个硬编码偏移**。

2. **最大的工程升级是「把补丁变成数据」。** 本文的补丁是代码里的一串调用；
   vphone-cli 的补丁是有名字、有版本门、有预设、有选择、有计划、有回执的**声明**，
   因此可以「开关、审计、检测漂移、可逆」。

3. **最实质的安全改进是「不用关 SIP」。** 本文要求 `csrutil disable` +
   `amfi_get_out_of_my_way=1`；vphone-cli 用 allowlist 精确放行那一个二进制。

4. **最巧的一步是「换掉 boot ROM 而不是打补丁」。** OpenAVB / OpenSEP 让
   iBoot 和 SEPOS 之前的代码不再出自 Apple，本文第 6 节整节的补丁工作因此消失。

5. **最容易踩的坑从「偏移错了」变成了「组合不兼容」。** 本文读者担心的是
   「换版本偏移全废」；vphone-cli 读者要担心的是
   「这个 cloudOS 还有没有 `vphone600ap`」「这个 iOS 的 dyld cache 共享区放不放得下」。
   后者有 `fw catalog`、兼容矩阵和 `cfw install` 的前置拒绝来兜底。

> **一句话**：本文教你**为什么能跑起来**，vphone-cli 展示**怎么让它一直跑得下去**。

---

## 附录 A：固件补丁速查表

> 所有偏移均针对 **iOS/cloudOS 26.1 (23B85)、vphone600、research 内核**。换版本请重新定位。
> 这些是 `CFW/patch_fw.py` 里的精简版；完整版（含越狱补丁）见仓库。

| 组件 | 偏移 | 补丁 | 目的 |
| --- | --- | --- | --- |
| iBSS | `0x9D10` / `0x9D14` | `nop` / `mov x0,#0` | 绕过 `image4_validate_property_callback` |
| iBSS | `0x1b544` | `b` | 不重新生成 APNonce（保持 nonce 可复现） |
| iBEC | `0x9D10/0x9D14` | 同上 | 同上 |
| iBEC | `0x122d4/0x122d8` + `0x24070` | 写指针 + 字符串 | 设置 boot-args `serial=3 -v debug=0x2014e %s` |
| LLB | `0xA0D8/0xA0DC` | 同上 | 绕过签名校验 |
| LLB | `0x12888/0x1288C` + `0x24990` | 写指针 + 字符串 | 设置 boot-args `serial=3 -v debug=0x2014e %s` |
| LLB | `0x2BFE8`,`0x2bca0`,`0x2C03C`,`0x2fcec`,`0x2FEE8` | 分支/`nop` | 允许加载被编辑的 rootfs（为 `snaputil -n` 铺路） |
| LLB | `0x1AEE4` | `nop` | 绕过未知 panic |
| TXM | `0x2c1f8`,`0x2bef4`,`0x2c060` | `mov x0,#0` | 让未注册于 TrustCache 的二进制可运行 |
| kernel | `0x2476964` | `nop` | `_apfs_vfsop_mount`：防 “Failed to find the root snapshot” |
| kernel | `0x23cfde4` | `nop` | `_authapfs_seal_is_broken`：防 “root volume seal is broken” |
| kernel | `0xf6d960` | `nop` | `_bsd_init`：防 “rootvp not authenticated” |

`fix_boot.py` 里的二进制 patch：

| 目标 | 偏移 | 补丁 | 目的 |
| --- | --- | --- | --- |
| seputil | `0x1B3F1` | `"AA"` | gigalocker 固定找 `AA.gl` |
| launchd_cache_loader | `0xB58` | `nop` | 允许 `launchd_unsecure_cache=1` |
| mobileactivationd | `0x2F5F84` | `mov x0,#1` | `should_hactivate` 永远返回真（绕过激活） |
| launchd | `0xd73c` | `b` | 防 jetsam 属性未初始化的 panic |

> 每一步 patch 后都建议用 `vbindiff` 之类工具确认「只有预期的那几个字节变了」。

---

## 附录 B：故障排查清单

| 现象 | 可能原因 / 处理 |
| --- | --- |
| `tart run vphone --dfu` 后 USB 里没有设备 | 宿主 SIP/AMFI/研究来宾未正确设置；`AVPBooter`/`SEPStorage` 没拷对 |
| 恢复阶段 panic，日志提到 SEP | `SEPStorage` 或 `AVPSEPBooter.vresearch1.bin` 缺失/路径不对 |
| 恢复完成重启后 panic，缺 `libSystem.B.dylib` | **正常现象**，必须走 SSH Ramdisk 补 Cryptex |
| SSH 连不上 2222 | `iproxy` 是否在跑；ramdisk 是否真的起来了（看 creeper 界面） |
| `seputil` 报 `Gigalocker ... doesn't exist` | seputil 未打补丁，或 `.gl` 未重命名为 `AA.gl` |
| 改了 `launchd.plist` 后不生效 | `launchd_cache_loader` 未打补丁 |
| MetalTest 返回 null | `AppleParavirtGPUMetalIOGPUFamily.bundle` 未安装，或缺作者未公开的 dylib |
| 首次正常启动能进（黑底）设置页，但一操作就 respring | 缺用户态 GPU/Metal 驱动（见[第 12 节](#12-实现-metal--implementing-metal)）；先装 `.bundle`，仍不行说明缺作者未公开的 `libAppleParavirtCompilerPluginIOGPUFamily.dylib` |
| panic: `initproc exited … jetsam property category (Daemon) is not initialized` | `launchd` 的 jetsam patch 未生效（`fix_boot.py` 第 7 步） |
| `idevicerestore` 报签名/ticket 错误 | SHSH 未抓到或 APNonce 变了；可考虑 `fake_tss_server.py` |
| 换固件版本后全部 panic | **预期**：所有偏移都是硬编码，需重新定位 |

---

## 附录 C：法律、伦理与风险提示

1. **仅在你拥有或有权处置的设备上操作。** 关 SIP / AMFI、改宿主机根卷、改 bootrom 都会
   降低这台 Mac 的安全性，请用专用测试机。
2. 固件、Cryptex、GPU 驱动等**版权归 Apple 所有**。原作者已因版权顾虑下架了一部分源码；
   本文档只解释原理，不提供任何 Apple 二进制。
3. 该技术属于**安全研究**范畴。请遵守 Apple 的 PCC 研究计划条款与当地法律，不要用于
   攻击真实用户设备或绕过他人设备的激活。
4. 你可能会**变砖或无法联网**（Xcode 调试所需的开发者设置、TSS 等），提前做好备份与恢复计划。

---

## 附录 D：参考链接

**本仓库**
- [README.md](README.md)（英文原版）
- [README_ko.md](README_ko.md)（韩文版）
- [contents/ScreenSharingVNC.swift](contents/ScreenSharingVNC.swift)
- [contents/BuildManifest.plist](contents/BuildManifest.plist) / [contents/Restore.plist](contents/Restore.plist)

**核心项目**
- [wh1te4ever/super-tart-vphone](https://github.com/wh1te4ever/super-tart-vphone)（可运行脚本 + GUIDE，已归档）
- [JJTech0130/super-tart](https://github.com/JJTech0130/super-tart)（原版虚拟机工具）
- [wh1te4ever/libirecovery](https://github.com/wh1te4ever/libirecovery)（支持 vresearch101ap）

**Apple 官方**
- [Private Cloud Compute 安全文档](https://security.apple.com/documentation/private-cloud-compute)
- [VRE 上手（vresetup）](https://security.apple.com/documentation/private-cloud-compute/vresetup)
- [apple/security-pcc](https://github.com/apple/security-pcc)（`vrevm` / `pccvre` 源码）

**工具与背景**
- [pyimg4](https://github.com/m1stadev/pyimg4) / [img4tool](https://github.com/tihmstar/img4tool) / [img4lib](https://github.com/xerub/img4lib)
- [idevicerestore](https://github.com/libimobiledevice/idevicerestore) / [libirecovery](https://github.com/libimobiledevice/libirecovery)
- [verygenericname/SSHRD_Script](https://github.com/verygenericname/SSHRD_Script)
- [nick-botticelli/vma2pwn](https://github.com/nick-botticelli/vma2pwn)（macOS 上的同类思路）
- [steven-michaud 的 iBoot patch gist](https://gist.github.com/steven-michaud/fda019a4ae2df3a9295409053a53a65c)
- [blacktop/ipsw](https://github.com/blacktop/ipsw)

**更省事的替代**
- [Lakr233/vphone-cli](https://github.com/Lakr233/vphone-cli)
- [owngoal-dev/OpenAVB](https://github.com/owngoal-dev/OpenAVB) / [owngoal-dev/OpenSEP](https://github.com/owngoal-dev/OpenSEP)

---

*本文为社区中文对照版，非 Apple 官方文档。如果你在复现中发现本文与原始脚本/仓库有出入，
以原始仓库为准，并欢迎修正本文。*

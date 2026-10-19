# 用近期发布的 PCC 固件中的 VPHONE600AP 组件构建一台虚拟 iPhone（中文对照版）

> 英文原题：**Building virtual iPhone using VPHONE600AP component of recently released PCC firmware**

本文是仓库 [README.md](README.md) 的**中文对照版（inline bilingual）**，做三件事：

1. **逐节保留原 writeup 的英文原文**（含代码块、截图、链接，未删改）；
2. 在每节下方给出**对应的中文翻译**；
3. 再补上**面向普通 iOS 工程师的原理与复现细节**：术语表、Apple 启动链、每一步的检查点、
   固件补丁速查表、故障排查清单、法律与风险提示。

配套的可运行脚本（原作者后来整理出的可执行版本）在
[wh1te4ever/super-tart-vphone](https://github.com/wh1te4ever/super-tart-vphone)
（**最后一次提交为 2026-02-25；仓库现已归档 → 只读**）以及它的
[GUIDE.md](https://github.com/wh1te4ever/super-tart-vphone/blob/main/GUIDE.md)。
本文会频繁引用这两处，建议对照阅读。

## 阅读约定

每个小节统一为三层结构，用加粗小标题区分：

- **原文 / Original** —— 原 writeup 该节内容（英文，逐字保留）；
- **中文翻译** —— 上面对应英文的中文；
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
- [附录 A：固件补丁速查表](#附录-a固件补丁速查表)
- [附录 B：故障排查清单](#附录-b故障排查清单)
- [附录 C：法律、伦理与风险提示](#附录-c法律伦理与风险提示)
- [附录 D：参考链接](#附录-d参考链接)

---

## 0. 行前须知：结论、术语与准备

### 0.1 一句话结论与整体流水线

**原文 / Original**：本节为本文新增，无对应原文。

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

**原文 / Original**：本节为本文新增，无对应原文。

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

**原文 / Original**：本节为本文新增，无对应原文（原 writeup 只在「Compatibility」一节提到机型）。

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

**原文 / Original**：本节为本文新增，无对应原文（原 writeup 直接假设你已有一台 PCC 研究虚拟机）。

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

**原文 / Original**

> # Special Thanks / Acknowledgements
>
> - [dlevi309](https://github.com/dlevi309) (Provided ideas for touch interaction on virtual iPhone)
> - [khanhduytran0](https://github.com/khanhduytran0), [34306](https://github.com/34306), [asdfugil](https://github.com/asdfugil), [verygenericname](https://github.com/verygenericname) (Provided other ideas for building virtual iPhone, including Cryptex, Device Activation, Ramdisk booting, etc.)
> - [ma4the](https://github.com/ma4the), [Mard](https://github.com/Mardcelo), [SwallowS](https://github.com/swollows) (Testing if works well on other environment)

**中文翻译**

- [dlevi309](https://github.com/dlevi309)（提供了虚拟 iPhone 触摸交互的思路）
- [khanhduytran0](https://github.com/khanhduytran0)、[34306](https://github.com/34306)、[asdfugil](https://github.com/asdfugil)、[verygenericname](https://github.com/verygenericname)（提供了构建虚拟 iPhone 的其他思路，包括 Cryptex、设备激活、Ramdisk 启动等）
- [ma4the](https://github.com/ma4the)、[Mard](https://github.com/Mardcelo)、[SwallowS](https://github.com/swollows)（帮忙测试在其他环境下是否可用）

---

## 2. 动机 / Motivation

**原文 / Original**

> # Motivation
>
> Around late 2024, Apple began introducing [Private Cloud Compute](https://security.apple.com/blog/private-cloud-compute/), claiming to open new horizon for cloud-based AI privacy. Then, around late 2025, some interesting news appeared: Apple had newly added vphone600ap-related components to PCC firmware, starting with cloudOS 26.
>
> ![Source: [https://x.com/matteyeux/status/2006339694783848660/photo/1](https://x.com/matteyeux/status/2006339694783848660/photo/1)](contents/image.png)
>
> Source: [https://x.com/matteyeux/status/2006339694783848660/photo/1](https://x.com/matteyeux/status/2006339694783848660/photo/1)
>
> **"iPhone Research Environment Virtual Machine”?**
>
> Is this a planned move by Apple to build and distribute a virtual iPhone environment for other security researchers in the future, or was it simply a mistake? Given that DEVELOPMENT/KASAN build kernel was once discovered in the iOS 15.0 beta to 15.1 beta3 OTAs back in 2021, the possibility of a slip-up cannot be ruled out. At that time, the kernel remained included for about 4 months, roughly from June to October 2021.
>
> Then, around January of this year, a tweet was posted showing a virtual iPhone booting up utilizing these vphone600ap-related components.
>
> ![Source: [https://x.com/_inside/status/2008951845725548783](https://x.com/_inside/status/2008951845725548783)](contents/Screenshot_2026-02-24_at_7.37.31_PM.png)
>
> Source: [https://x.com/_inside/status/2008951845725548783](https://x.com/_inside/status/2008951845725548783)
>
> ![Screenshot 2026-02-24 at 7.39.03 PM.png](contents/Screenshot_2026-02-24_at_7.39.03_PM.png)
>
> From what I saw, almost everything worked truly elegantly. Compared to the [QEMUAppleSilicon(Inferno) project](https://github.com/ChefKissInc/Inferno) I had seen previously, it runs much snappier and smoother. Furthermore, it even appeared to support Metal acceleration. Ultimately, completely captivated by it, I dove right in and started building my own virtual iPhone on January 31st.
>
> ![Screenshot 2026-02-24 at 7.46.41 PM.png](contents/Screenshot_2026-02-24_at_7.46.41_PM.png)

**中文翻译**

2024 年底左右，Apple 开始推出 [Private Cloud Compute](https://security.apple.com/blog/private-cloud-compute/)，宣称要为云端 AI 隐私开辟新天地。然后到了 2025 年底左右，出现了一条有意思的消息：Apple 从 cloudOS 26 开始，往 PCC 固件里**新增了与 `vphone600ap` 相关的组件**。

（截图来源见上方链接）

**「iPhone Research Environment Virtual Machine（iPhone 研究环境虚拟机）」？**

这是 Apple 计划中的一步——未来为其他安全研究者构建并分发一套虚拟 iPhone 环境？还是仅仅是一次失误？考虑到 2021 年 iOS 15.0 beta 到 15.1 beta3 的 OTA 里曾被发现混入 DEVELOPMENT/KASAN 版本的内核，**「手滑」的可能性也不能排除**。当时那个内核大约在包里保留了 4 个月，大致是 2021 年 6 月到 10 月。

再后来，今年 1 月左右，有人发推展示了一台**利用这些 `vphone600ap` 相关组件启动起来的虚拟 iPhone**。

（截图见上方）

从我看到的来看，几乎一切都运行得相当优雅。相比我之前见过的 [QEMUAppleSilicon(Inferno) 项目](https://github.com/ChefKissInc/Inferno)，它明显更快更流畅，而且**似乎还支持 Metal 加速**。最终我彻底被它吸引，一头扎了进去，在 1 月 31 日开始动手搭自己的虚拟 iPhone。

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

**原文 / Original**

> # **Modifying super-tart to boot virtual iPhone**
>
> The referenced project is [security-pcc](https://github.com/apple/security-pcc). It corresponds to the source code of the /System/Library/SecurityResearch/usr/bin/vrevm binary. An interesting point is that it uses private methods provided by Virtualization.framework. In the virtual machine used for PCC research, you can see that the ISA and PlatformVersion are explicitly specified during the hardware model initialization process.
>
> ![Screenshot 2026-02-24 at 8.27.01 PM.png](contents/Screenshot_2026-02-24_at_8.27.01_PM.png)
>
> For the bootrom, AVPBooter.vresearch1.bin is used (/System/Library/Frameworks/Virtualization.framework/Resources/AVPBooter.vresearch1.bin)
>
> ![Screenshot 2026-02-24 at 8.32.08 PM.png](contents/Screenshot_2026-02-24_at_8.32.08_PM.png)
>
> and for the SEPROM (avpsepbooter), AVPSEPBooter.vresearch1.bin is used, which separately loads a SEPStorage file that functions similarly to [AuxiliaryStorage](https://developer.apple.com/documentation/virtualization/vzmacplatformconfiguration/auxiliarystorage).
> (/System/Library/Frameworks/Virtualization.framework/Versions/A/Resources/AVPSEPBooter.vresearch1.bin)
>
> Another interesting point is that if you look at the code for setting the resolution, it is set to 1290x2796, which corresponds to the iPhone 14 Pro Max, 15 Plus, 15 Pro Max, and 16 Plus devices.
>
> ![Screenshot 2026-02-24 at 8.34.11 PM.png](contents/Screenshot_2026-02-24_at_8.34.11_PM.png)
>
> With just this information, it should be more than enough to modify [super-tart](https://github.com/JJTech0130/super-tart) to boot the virtual iPhone. I made the modifications as shown below.
>
> - /Sources/tart/VM.swift
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

**中文翻译**

参考的是 [security-pcc](https://github.com/apple/security-pcc) 项目——它对应
`/System/Library/SecurityResearch/usr/bin/vrevm` 这个二进制的源码。有意思的一点是，
**它用到了 Virtualization.framework 的私有方法**。在 PCC 研究用的虚拟机里可以看到，
硬件模型初始化时会显式指定 **ISA 和 PlatformVersion**。

（截图见上）

bootrom 用的是 `AVPBooter.vresearch1.bin`
（位于 `/System/Library/Frameworks/Virtualization.framework/Resources/AVPBooter.vresearch1.bin`）。

（截图见上）

而 SEPROM（avpsepbooter）用的是 `AVPSEPBooter.vresearch1.bin`，它会额外加载一个 `SEPStorage` 文件，
作用类似于 [AuxiliaryStorage](https://developer.apple.com/documentation/virtualization/vzmacplatformconfiguration/auxiliarystorage)。
（位于 `/System/Library/Frameworks/Virtualization.framework/Versions/A/Resources/AVPSEPBooter.vresearch1.bin`）

另一个有意思的点：如果你去看**设置分辨率**的那段代码，它写的是 **1290x2796**，
正好对应 iPhone 14 Pro Max、15 Plus、15 Pro Max 和 16 Plus 这些机型。

（截图见上）

有了这些信息，改造 [super-tart](https://github.com/JJTech0130/super-tart) 让它启动虚拟 iPhone
就绰绰有余了。我做的修改如下（`/Sources/tart/VM.swift`，见上方代码块）。

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

---

## 4. 改造固件 / Modifying the Firmware

**原文 / Original**

> # Modifying the Firmware
>
> The referenced project is [vma2pwn](https://github.com/nick-botticelli/vma2pwn). Specifically for version 12.0.1, it boots a Mac virtual machine with almost the entire bootchain modified.
>
> Let's look at the [prepare.sh](https://github.com/nick-botticelli/vma2pwn/blob/main/prepare.sh) script first. It extracts firmware components, such as the bootloader and kernel compressed in IM4P format, into RAW format and patches instructions/data at specific hardcoded addresses. RestoreRamdisk is the root filesystem used when restoring the firmware, and AVPBooter is the BootROM used in the virtual machine.
>
> To summarize, it extracts the individual files included in the firmware and patches integrity checks to allow the restoration of custom firmware, or modifies the boot-args parameters to make it easier to view boot-related logs.
>
> Finally, [vma2pwn.sh](https://github.com/nick-botticelli/vma2pwn/blob/main/vma2pwn.sh) is responsible for restoring the custom firmware. It does this by entering DFU mode beforehand. Here, the virtual machine uses something called super-tart. This is a version of the existing tart virtual machine with added features like a custom bootrom, serial output, DFU mode, and GDB debugging. (Note that SIP/AMFI must be disabled for this to work.)
>
> I have used it quite usefully recently while I [studying XNU kernel 1-day vulnerabilities (CVE-2021-30937, CVE-2021-30955)](https://github.com/wh1te4ever/xnu_1day_practice). It is fantastic because it supports live kernel debugging.

**中文翻译**

参考的项目是 [vma2pwn](https://github.com/nick-botticelli/vma2pwn)。具体针对 12.0.1 版本，
它启动了**一台几乎整条 bootchain 都被修改过的 Mac 虚拟机**。

先看它的 `prepare.sh` 脚本：它把固件组件（比如以 IM4P 格式压缩的 bootloader 和内核）解包成
RAW 格式，并在**写死的地址上**打指令/数据补丁。其中 `RestoreRamdisk` 是恢复固件时用的根文件系统，
`AVPBooter` 是虚拟机里用的 BootROM。

总结一下：它把固件里的各个文件抽出来，然后**补掉完整性校验**，从而允许恢复自定义固件；
或者修改 **boot-args**，以便更容易查看启动相关的日志。

最后，`vma2pwn.sh` 负责恢复自定义固件。做法是**先进入 DFU 模式**。这里虚拟机用的是叫
**super-tart** 的东西——它是现有 tart 虚拟机的一个改造版，加了自定义 bootrom、串口输出、
DFU 模式、GDB 调试等特性。（注意：为此必须关闭 SIP/AMFI。）

我最近在[研究 XNU 内核 1-day 漏洞（CVE-2021-30937、CVE-2021-30955）](https://github.com/wh1te4ever/xnu_1day_practice)
时把它用得挺顺手。它很棒，因为支持**实时内核调试**。

**补充解读**

- **vma2pwn 是「同类思路的先行者」**：同样是「解包 IM4P → 打补丁 → 重打包 → DFU 恢复」，
  只是对象是 macOS 虚拟机。理解了它，本项目的 `patch_fw.py` 就不难懂。
- **为什么必须关 SIP/AMFI**：super-tart 要加载自定义 bootrom、走私有 API，AMFI 会拦住未签名
  二进制/kext；SIP 会拦住对系统关键路径的替换。这正是 [0.3](#03-宿主机环境与工具链含-step-1) 里那几条命令的原因。
- **bootchain 与各名词的对应关系**见 [0.2（2）](#02-术语表与-apple-启动链)。

---

## 5. 构建自定义固件 / Building Custom Firmware

**原文 / Original**

> ## Building Custom Firmware
>
> I mixed the components of cloudOS 26.1 (23B85) and iOS 26.1 (iPhone17,3; 23B85), uh,,, but... I can't remember the exact details. To be precise, I had to properly mix the iPhone 16 and vphone-related components to create the custom firmware, but I've forgotten exactly which ones I ended up mixing. From what I recall:
>
> - BuildManifest.plist:
> I modified the dictionary elements under the Manifest key. I configured it so that during the restore process, the SystemVolume, SystemVolumeCanonicalMetadata, OS, StaticTrustCache, RestoreTrustCache, and RestoreRamDisk from iPhone 16(iOS 26.1) model would be used. The rest were set up to use vphone-related files from PCC firmware.
> - Restore.plist:
> I believe I added properties related to DeviceMap or SupportedProductTypes, or changed the SystemRestoreImageFileSystems element.
>
> The files below are the final result of my mix.
>
> [Restore.plist](contents/Restore.plist)
>
> [BuildManifest.plist](contents/BuildManifest.plist)
>
> - get_fw.py (Partial)
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

**中文翻译**

我把 **cloudOS 26.1（23B85）** 和 **iOS 26.1（iPhone17,3；23B85）** 的组件混在了一起——
呃……但**我记不清具体细节了**。准确说，我必须把 iPhone 16 和 vphone 相关组件正确地混合，
才能做出这个自定义固件，但我已经忘了最后到底混了哪些。凭印象：

- **BuildManifest.plist**：我修改了 `Manifest` 键下的字典元素。配置成：恢复过程中，
  `SystemVolume`、`SystemVolumeCanonicalMetadata`、`OS`、`StaticTrustCache`、`RestoreTrustCache`、
  `RestoreRamDisk` 使用 **iPhone 16（iOS 26.1）** 的；**其余**使用 PCC 固件里的 vphone 相关文件。
- **Restore.plist**：我印象里是加了 `DeviceMap` 或 `SupportedProductTypes` 相关的属性，
  或者改了 `SystemRestoreImageFileSystems` 元素。

下面两个文件就是我混合后的最终结果：
[Restore.plist](contents/Restore.plist)、[BuildManifest.plist](contents/BuildManifest.plist)

（`get_fw.py` 片段见上方代码块）

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

---

## 6. 打补丁 AVPBooter.vresearch1.bin / Patch AVPBooter.vresearch1.bin

**原文 / Original**

> ## Patch AVPBooter.vresearch1.bin
>
> I referenced [that post](https://gist.github.com/steven-michaud/fda019a4ae2df3a9295409053a53a65c#iboot-stage-0-avpbootervmapple2binorg). You must patch `image4_validate_property_callback` in order to load  custom bootloader afterward. Simply use "Text-search (slow!)" feature in IDA Pro to search for "0x4447", and patch the epilogue of the corresponding function to always return 0.
>
> ![image.png](contents/image%201.png)

**中文翻译**

我参考了[那篇 gist](https://gist.github.com/steven-michaud/fda019a4ae2df3a9295409053a53a65c#iboot-stage-0-avpbootervmapple2binorg)。
为了之后能加载**自定义 bootloader**，你必须 patch `image4_validate_property_callback`。
只要用 IDA Pro 的 `Text-search (slow!)` 功能搜 `0x4447`，然后把对应函数的 epilogue
patch 成**永远返回 0** 即可。

（截图见上）

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

---

## 7. 改造并编译 libirecovery / Modifying and Building libirecovery

**原文 / Original**

> ## Modifying and Building libirecovery
>
> Before restoring the firmware, a few modifications were necessary to support the vresearch101ap model. Once built, firmware restoration becomes possible using the [idevicerestore](https://github.com/libimobiledevice/idevicerestore) tool.
>
> [https://github.com/wh1te4ever/libirecovery](https://github.com/wh1te4ever/libirecovery)
>
> ![Screenshot 2026-02-24 at 9.52.14 PM.png](contents/Screenshot_2026-02-24_at_9.52.14_PM.png)

**中文翻译**

在恢复固件之前，需要做一些修改以支持 **vresearch101ap** 型号。编译好之后，
就可以用 [idevicerestore](https://github.com/libimobiledevice/idevicerestore) 工具来恢复固件了。

项目地址：[wh1te4ever/libirecovery](https://github.com/wh1te4ever/libirecovery)（截图见上）

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

---

## 8. 给固件组件打补丁 / Patching Firmware Components

**原文 / Original**

> ## **Patching Firmware Components**
>
> Similar to AVPBooter, the bootloaders used for restoration, iBSS and iBEC, were patched to bypass signature verification. I also enabled serial log output so that if there are any booting issues, the cause can be identified immediately.
>
> As you will see later, bypassing [SSV (Signed System Volume) verification](https://support.apple.com/fr-lu/guide/security/secd698747c9/web) is required to load an arbitrary Cryptex. This is performed in the LLB, which is loaded when booting in normal mode rather than DFU mode, and verification is also sometimes performed in the kernel.
>
> Additionally, I patched the TXM so that even if a binary/library is not registered in the Trustcache, it is recognized as if it were.
>
> - patch_fw.py (Partial content, Part 1)
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
> In the case of the kernel or TXM, a PAYP structure exists, so it was necessary to preserve that structure.
> Below is the code that converts IM4P → RAW → IM4P using [pyimg4](https://pypi.org/project/pyimg4/), [img4tool](https://github.com/tihmstar/img4tool), [img4](https://github.com/xerub/img4lib) tool.
>
> - patch_fw.py (Partial content, Part 2)
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

**中文翻译**

和 AVPBooter 类似，恢复时用到的 bootloader——**iBSS 和 iBEC**——都被打了补丁以**绕过签名校验**。
我顺便打开了**串口日志输出**，这样一旦启动出问题，能立刻定位原因。

后面你会看到：为了加载一个任意（被修改过的）Cryptex，必须绕过
[SSV（Signed System Volume）校验](https://support.apple.com/fr-lu/guide/security/secd698747c9/web)。
这一步发生在 **LLB** 里（LLB 在正常启动而非 DFU 模式下加载），**有时内核里也会做校验**。

此外，我还 patch 了 **TXM**，使得即使某个二进制/库**没有登记在 TrustCache 里**，
也能被当作「已登记」来处理。

（两段 `patch_fw.py` 片段见上方代码块）

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

---

## 9. 恢复固件 / Restoring Firmware

**原文 / Original**

> # Restoring Firmware
>
> Once everything is ready, let's put the virtual machine into DFU mode and try restoring it.
>
> Below is a screenshot of the panic that occurs if the SEP is not configured properly. If you set it up correctly, it should pass this point without any issues.
>
> ![image.png](contents/image%202.png)
>
> After the restore is complete, it reboots automatically.
> However, a panic occurs in the launchd process because the /usr/lib/libSystem.B.dylib library is missing. This library is located within the dyld_shared_cache on the Cryptex partition, and for some reason, the Cryptex partition could not be restored. As a temporary workaround, you must create an [SSH Ramdisk](https://github.com/verygenericname/SSHRD_Script) to modify the root file system and inject the necessary files. That is exactly why the patch related to SSV verification was needed.
>
> ![Screenshot 2026-02-24 at 10.24.33 PM.png](contents/Screenshot_2026-02-24_at_10.24.33_PM.png)
>
> ![image.png](contents/image%203.png)

**中文翻译**

一切就绪后，把虚拟机切到 **DFU 模式**，试着恢复它。

下面这张截图是 **SEP 没配置正确**时会出现的 panic。如果配置正确，就能顺利通过这一步。

（截图见上）

恢复完成后虚拟机会**自动重启**。然而 `launchd` 进程会 panic，原因是
**`/usr/lib/libSystem.B.dylib` 库缺失**。这个库位于 Cryptex 分区上的 `dyld_shared_cache` 里，
而因为某些原因**Cryptex 分区没能被恢复**。作为临时对策，你必须做一个
[SSH Ramdisk](https://github.com/verygenericname/SSHRD_Script) 来修改根文件系统并注入需要的文件。
**这正是之前需要打 SSV 相关补丁的原因。**

（截图见上）

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

---

## 10. 用 SSH Ramdisk 修复启动问题 / Fixing Boot Issue with SSH Ramdisk

**原文 / Original**

> # Fixing Boot Issue by Booting with SSH Ramdisk
>
> I am going to try fixing the boot failure issue by utilizing the ramdisk used in https://github.com/verygenericname/SSHRD_Script.
>
> To upload and load components like the bootloader or kernel using the [irecovery](https://github.com/libimobiledevice/libirecovery) tool in DFU mode, an IMG4 image is needed, which requires an IM4M file. Therefore, I first fetched shsh file using idevicerestore tool, and then converted it into an IM4M file.
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
> - boot_rd.sh
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
> If you check the USB menu in the System Information app and see "iPhone Research...",
> you can now access the virtual iPhone shell using [iproxy](https://github.com/libimobiledevice/libusbmuxd/blob/master/tools/iproxy.c) tool. (`iproxy 2222 22 &`)
>
> ![image.png](contents/image%204.png)
>
> To modify the root file system, rename the snapshot.
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
> Along with the file transfer, specific patches were required. For convenience, I added three specific processes to start on boot: bash, dropbear, and [trollvnc](https://github.com/OwnGoalStudio/TrollVNC).
>
> seputil had an issue where it couldn't properly find the gigalocker file, so I patched it to always look for AA.gl. Furthermore, I patched launchd_cache_loader to ensure that the modified /System/Library/xpc/launchd.plist loads correctly.
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

**中文翻译**

我打算借助 https://github.com/verygenericname/SSHRD_Script 里用的 ramdisk 来修复启动失败的问题。

在 DFU 模式下，用 [irecovery](https://github.com/libimobiledevice/libirecovery) 上传并加载 bootloader、
内核等组件时，需要 **IMG4** 镜像，而 IMG4 又需要一个 **IM4M** 文件。所以先用 idevicerestore 抓取
shsh 文件，再把它转成 IM4M：

```bash
idevicerestore -e -y ./iPhone17,3_26.1_23B85_Restore -t
mv shsh/[ECID]-iPhone99,11-26.1.shsh shsh/[ECID]-iPhone99,11-26.1.shsh.gz
gunzip shsh/[ECID]-iPhone99,11-26.1.shsh.gz
pyimg4 im4m extract -i shsh/[ECID]-iPhone99,11-26.1.shsh -o vphone.im4m
```

然后，用这个 IM4M 文件，为每个要用的固件组件（iBSS、iBEC、devicetree 等）**分别生成 IMG4**。

（生成脚本见上方代码块，覆盖：iBSS / iBEC / SPTM / devicetree / SEP / TXM / kernelcache / 自建 ramdisk / trustcache）

所有 IMG4 都生成好之后，逐个加载并用 Ramdisk 启动：

- `boot_rd.sh`（见上方代码块，**顺序很重要**）

接着，你会看到下面截图里「左边数第三个窗口」出现的 Minecraft **Creeper 脸**。
如果你在「系统信息」App 的 USB 菜单里看到 “iPhone Research...”，
就可以用 [iproxy](https://github.com/libimobiledevice/libusbmuxd/blob/master/tools/iproxy.c) 工具
（`iproxy 2222 22 &`）访问虚拟 iPhone 的 shell 了。

（截图见上）

要修改根文件系统，需要**重命名快照**：

```python
ssh root@127.0.0.1 -p2222
#pw: alpine

mount_apfs -o rw /dev/disk1s1 /mnt1
snaputil -l /mnt1
# 会打印出一串哈希（结果可能不同）
com.apple.os.update-8AAB8DBA5C8F1F756928411675F4A892087B04559CFB084B9E400E661ABAD119
snaputil -n <com.apple.os.update-hash> orig-fs /mnt1
umount /mnt1
exit
```

用 [ipsw](https://github.com/blacktop/ipsw) 工具**解密 AEA 文件**得到 dmg，挂载它，
然后把 **Cryptex 分区**里的文件传进虚拟机。除了传文件，还需要一些特定补丁。
为了方便，我加了三个开机自启的进程：**bash、dropbear 和 [trollvnc](https://github.com/OwnGoalStudio/TrollVNC)**。

**seputil** 有个问题：它找不到正确的 gigalocker 文件，所以我把它 patch 成**永远找 `AA.gl`**。
此外，我还 patch 了 **launchd_cache_loader**，确保被修改过的 `/System/Library/xpc/launchd.plist`
能正确加载。

（完整的 `fix_boot.py` 片段见上方代码块）

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

---

## 11. 第一次启动尝试 / First Boot Attempt

**原文 / Original**

> # First Boot Attempt
>
> The boot process should work fine now, but when you try to proceed past the black setup screen, it resprings and won't go any further.
>
> ![image.png](contents/image%205.png)
>
> ![image.png](contents/image%206.png)

**中文翻译**

现在启动流程应该能正常工作了，但当你试图**越过（黑底的）设置界面**继续时，它会 **respring
（SpringBoard 重启、回到主屏）**，无法再往下走。

（两张截图见上）

**补充解读**

- 「respring」= iOS 的 SpringBoard（桌面/窗口管理器）重启，不是整机重启，表现为闪一下回到主屏。
- 这里卡住的原因，是**缺用户态的 GPU/Metal 驱动**（内核已经认得 `AppleParavirtGPU`，
  但用户态缺 `AGXMetal*` 一类库）。→ 见[第 12 节](#12-实现-metal--implementing-metal)。

---

## 12. 实现 Metal / Implementing Metal

**原文 / Original**

> # Implementing Metal
>
> When checking with a custom program called MetalTest, it shows that Metal is not supported.
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
> ![image.png](contents/image%207.png)
>
> When checking on an iPad 7th Gen running iOS 16.6.1, calling the `MTLCreateSystemDefaultDevice` function internally accesses the IOGPU driver through a specific library called `AGXMetalA10`. This `AGXMetalA10` library is located in `/System/Library/Extensions`.
>
> A thought suddenly crossed my mind here: wouldn't there be GPU/Metal-related libraries used for virtual iPhone as well?
>
> ![image.png](contents/image%208.png)
>
> Checking that same path in the PCC virtual machine reveals that 7 files exist there.
>
> I took the /System/Library/Extensions/AppleParavirtGPUMetalIOGPUFamily.bundle used in PCC and placed it directly into the virtual iPhone. (I used the SSH Ramdisk for this.)
>
> ![image.png](contents/image%209.png)
>
> Checking MetalTest again, the `MTLCreateSystemDefaultDevice` function now works properly.
>
> ![image.png](contents/image%2010.png)
>
> However, because a specific dylib file does not exist in the iPhone 16 model's dsc(dyld shared cache), I needed to separately reverse-engineer and implement it from the dsc in the PCC.
>
> - /System/Library/Extensions/AppleParavirtGPUMetalIOGPUFamily.bundle/libAppleParavirtCompilerPluginIOGPUFamily.dylib
>
> ![Screenshot 2026-02-25 at 1.19.40 PM.png](contents/Screenshot_2026-02-25_at_1.19.40_PM.png)
>
> ![image.png](contents/image%2011.png)

**中文翻译**

用一个叫 **MetalTest** 的自制程序检查时，显示 **Metal 不受支持**。

```objc
#import <stdio.h>
#import <Metal/Metal.h>
#import <Foundation/Foundation.h>

int main(int argc, char *argv[], char *envp[]) {
    id<MTLDevice> device = MTLCreateSystemDefaultDevice();
    NSLog(@"device: %@", device);
    if (device) {
        NSLog(@"Metal Device Create Success: %@", [device name]);
    } else {
        NSLog(@"Metal Not Supported!");
    }
    return 0;
}
```

**运行结果（不支持时）**：

```text
-bash-4.4# ./MetalTest
2026-02-08 22:49:02.293 MetalTest[633:9434] device: (null)
2026-02-08 22:49:02.294 MetalTest[633:9434] Metal Not Supported!
-bash-4.4# sysctl kern.version
kern.version: Darwin Kernel Version 25.1.0: ... root:xnu-12377.42.6~55/RELEASE_ARM64_VRESEARCH1
```

正常情况下，输出**应该**是这样：

```text
seo@seos-Virtual-Machine Desktop % ./MetalTest
2026-02-08 23:16:56.846 MetalTest[682:5810] device: <AppleParavirtDevice: 0x102c48fe0>
    name = Apple Paravirtual device
2026-02-08 23:16:56.847 MetalTest[682:5810] Metal Device Create Success: Apple Paravirtual device
```

用 `ioreg -l` 检查可以看到，**内核其实已经认出了 `AppleParavirtGPU`**。

（截图见上）

在一台运行 iOS 16.6.1 的 iPad 7 上检查时，`MTLCreateSystemDefaultDevice` 内部是通过一个叫
`AGXMetalA10` 的库去访问 IOGPU 驱动的，而这个 `AGXMetalA10` 就位于 `/System/Library/Extensions`。

这时我脑子里突然冒出一个念头：**虚拟 iPhone 是不是也有对应的 GPU/Metal 库？**

（截图见上）

去 PCC 虚拟机里看同一个路径，发现那里**有 7 个文件**。

我把 PCC 里用的 `/System/Library/Extensions/AppleParavirtGPUMetalIOGPUFamily.bundle` 直接搬进了
虚拟 iPhone（用的是 SSH Ramdisk）。

（截图见上）

再测 MetalTest，`MTLCreateSystemDefaultDevice` 就正常了。

（截图见上）

但是，因为 iPhone 16 型号的 **dsc（dyld shared cache）里不存在某个特定的 dylib**，
我需要单独从 PCC 的 dsc 里**逆向并实现**它：

- `/System/Library/Extensions/AppleParavirtGPUMetalIOGPUFamily.bundle/libAppleParavirtCompilerPluginIOGPUFamily.dylib`

（截图见上）

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

---

## 13. 第二次启动尝试 / Second Boot Attempt

**原文 / Original**

> # Second Boot Attempt
>
> Once implemented, you are now greeted by the setup screen with a background.
> Since I couldn't properly implement the home button, I resolved this using a temporary workaround by controlling it via iproxy/VNC.
>
> ![image.png](contents/image%2012.png)

**中文翻译**

实现之后，迎接你的就是**有背景的设置界面**了。
由于我没能正确实现 Home 键，我用一个临时方案解决：通过 **iproxy / VNC** 来控制它。

（截图见上）

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

---

## 14. 兼容性 / Compatibility

**原文 / Original**

> # Compatibility
>
> It is only compatible with Apple Silicon Macs, and the devices/versions confirmed to work are as follows:
>
> - Apple M3, 16GB RAM, Sequoia 15.7.4
> - Apple M1 Pro, 32GB RAM, Tahoe 26.3
>
> I expect it will probably work on any target that supports pccvre.
>
> ![Source: [https://security.apple.com/documentation/private-cloud-compute/vresetup](https://security.apple.com/documentation/private-cloud-compute/vresetup)](contents/image%2013.png)
>
> Source: [https://security.apple.com/documentation/private-cloud-compute/vresetup](https://security.apple.com/documentation/private-cloud-compute/vresetup)

**中文翻译**

它**只兼容 Apple Silicon Mac**，已经确认可用的设备/版本如下：

- Apple M3 / 16GB 内存 / Sequoia 15.7.4
- Apple M1 Pro / 32GB 内存 / Tahoe 26.3

我预计**任何支持 `pccvre` 的机型**应该都可以。

（截图来源见上）

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

---

## 15. 在 Sequoia 上启用触摸交互 / Enabling Touch Interaction on Sequoia

**原文 / Original**

> ## Enabling Touch Interaction on Sequoia
>
> Unlike Tahoe version 26, touch interaction is not possible using only the VZVirtualMachineView object, so it was necessary to override the mouse event functions.
>
> [ScreenSharingVNC.swift](contents/ScreenSharingVNC.swift)

**中文翻译**

和 Tahoe（版本 26）不同，在 Sequoia 上**只靠 `VZVirtualMachineView` 对象无法实现触摸交互**，
所以必须**重写鼠标事件函数**。

代码见：[`contents/ScreenSharingVNC.swift`](contents/ScreenSharingVNC.swift)

**补充解读**

- 这段代码是本仓库里少数**有完整源码**的部分，值得一读：它把鼠标事件翻译成多点触摸事件
  发给虚拟机。
- 在 Tahoe 26 上不需要它（系统直接支持）。

---

## 16. 项目源码 / Project SRC

**原文 / Original**

> ### Project SRC
> - https://github.com/wh1te4ever/super-tart-vphone

**中文翻译**

项目源码：[wh1te4ever/super-tart-vphone](https://github.com/wh1te4ever/super-tart-vphone)

**补充解读**

- 该仓库里有本文所有步骤对应的**可运行脚本**（`CFW/*.py`、`boot_rd.sh`）与 `GUIDE.md`。
- 注意：**仓库现已归档（只读）**，最后一次提交为 2026-02-25。

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

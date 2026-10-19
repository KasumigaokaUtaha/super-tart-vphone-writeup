# 在 Apple Silicon Mac 上跑起一台「虚拟 iPhone」——面向普通 iOS 开发者的完整复现指南

> 本文是对仓库 [super-tart-vphone-writeup](README.md) 的中文补充解读。
> 原 README 是一篇「作者自言自语式」的研究记录：结论和代码片段都在，但大量背景知识、隐含前提、
> 操作顺序和「为什么这么做」没有写出来。本文的目标是：**一个没接触过 Apple 固件/iBoot 逆向的
> 普通 iOS 工程师，照着本文能理解全局，并独立复现出同样的结果。**

配套可运行脚本（原作者后来整理的、可执行版本）在：
[wh1te4ever/super-tart-vphone](https://github.com/wh1te4ever/super-tart-vphone)（**最后一次提交为 2026-02-25；仓库现已归档 → 只读**）
和它的 [GUIDE.md](https://github.com/wh1te4ever/super-tart-vphone/blob/main/GUIDE.md)。
本文会频繁引用这两个仓库，请对照阅读。

---

## 目录

- [0. 先说结论：这事到底在干什么](#0-先说结论这事到底在干什么)
- [1. 背景：PCC、cloudOS 与 vphone600ap](#1-背景pcccloudos-与-vphone600ap)
- [2. 必备术语与 Apple 启动链](#2-必备术语与-apple-启动链)
- [3. 原理：为什么需要这么多补丁](#3-原理为什么需要这么多补丁)
- [4. 环境准备](#4-环境准备)
- [5. 分阶段复现](#5-分阶段复现)
  - [Step 1. 宿主安全设置](#step-1-宿主安全设置)
  - [Step 2. 用 pccvre 造一台 PCC 研究虚拟机](#step-2-用-pccvre-造一台-pcc-研究虚拟机)
  - [Step 3. 用 super-tart 搭出 vphone 虚拟机骨架](#step-3-用-super-tart-搭出-vphone-虚拟机骨架)
  - [Step 4. 编译 super-tart-vphone 与改造版 libirecovery](#step-4-编译-super-tart-vphone-与改造版-libirecovery)
  - [Step 5. 打补丁：AVPBooter.vresearch1.bin](#step-5-打补丁avpbootervresearch1bin)
  - [Step 6. 组装「混血」自定义固件](#step-6-组装混血自定义固件)
  - [Step 7. 进入 DFU，抓 SHSH，恢复固件](#step-7-进入-dfu抓-shsh恢复固件)
  - [Step 8. 制作并启动 SSH Ramdisk](#step-8-制作并启动-ssh-ramdisk)
  - [Step 9. 修好首次启动（Cryptex / seputil / launchd）](#step-9-修好首次启动cryptex--seputil--launchd)
  - [Step 10. 第二次启动与 Metal 加速](#step-10-第二次启动与-metal-加速)
  - [Step 11. 交互、VNC 与（可选）越狱](#step-11-交互vnc-与可选越狱)
- [6. 固件补丁速查表](#6-固件补丁速查表)
- [7. 故障排查清单](#7-故障排查清单)
- [8. 兼容性与更省事的替代方案](#8-兼容性与更省事的替代方案)
- [9. 法律、伦理与风险提示](#9-法律伦理与风险提示)
- [10. 参考链接](#10-参考链接)

---

## 0. 先说结论：这事到底在干什么

一句话：**Apple 给「Private Cloud Compute（PCC）」用的固件里，混进了一套名为 `vphone600ap` 的
「虚拟 iPhone」硬件定义。作者把 PCC 固件里的这些组件，和一台真实 iPhone 16（iPhone17,3）的
iOS 26.1 固件「拼」在一起，再用改造过的虚拟机软件（super-tart）跑了起来，得到一台能启动、
能进设置界面、甚至有 Metal 加速的虚拟 iPhone。**

整条流水线（这也是全文的骨架）：

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
- 官方项目已归档，存在一个更现代的替代品（见 [第 8 节](#8-兼容性与更省事的替代方案)）。

> 本节出现的 SSV / DFU / AVPBooter / Cryptex / SSH Ramdisk 等术语会在[第 2 节](#2-必备术语与-apple-启动链)
> 统一解释，第一遍读不通很正常，先记住这条流水线即可。

复现前请先接受以上前提，否则很容易卡在「为什么我的版本跑不通」。

---

## 1. 背景：PCC、cloudOS 与 vphone600ap

普通 iOS 开发者平时接触的是「设备 → .ipa / .ipsw」。这里要认识三个新东西：

**Private Cloud Compute（PCC）**
Apple 为 Apple Intelligence 搭的云端计算集群，主打隐私。2024 年起 Apple 公开了它的安全文档，
并在 [security.apple.com](https://security.apple.com/documentation/private-cloud-compute) 发布了
研究计划，其中包括一个叫 **VRE（Virtual Research Environment，虚拟研究环境）** 的东西：
让研究者在自己的 Mac 上**启动一个真实的 PCC 节点镜像**来做安全分析。

**cloudOS**
PCC 服务器上运行的操作系统，可以理解成「服务器版 iOS」。它的固件包（类似 iOS 的 ipsw）
托管在 `updates.cdn-apple.com/private-cloud-compute/...`。作者用的版本是 **cloudOS 26.1（23B85）**。

**vphone600ap / vresearch101**
这是 cloudOS 26.1 固件里新出现的一套硬件定义：

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

## 2. 必备术语与 Apple 启动链

这一段是让「普通 iOS 开发」跨到「固件研究」的桥梁。建议先通读一遍再动手。

### 2.1 镜像与签名格式

| 术语 | 说明 |
| --- | --- |
| **IM4P** | Image4 Payload，Apple 固件的标准封装，带类型标签（`ibss`/`ibec`/`illb`/`ibot`/`krnl`/`trxm`/`rdsk`…）和压缩（一般 lzfse） |
| **IM4M** | Image4 Manifest，即签名（含 APTicket），里面是对各 IM4P 的摘要 |
| **IM4P + IM4M = IMG4** | 一个「已签名」的固件镜像，可被 iBoot / irecovery 加载 |
| **SHSH / APTicket** | Apple 服务器根据设备 ECID、APNonce、要恢复的组件生成的签名票据。没有它，iBoot 不会加载你上传的镜像 |
| **ECID** | 每台设备（这里指虚拟机）的唯一芯片 ID。super-tart-vphone 默认写死为 `0x1de1518ecffe2725`（见 Step 4）；改动它会导致已有 SHSH 失效 |
| **APNonce** | 参与 SHSH 随机数，用于防重放 |
| **PAYP** | IM4P 末尾的一段附属结构（bling），**转换 IM4P → RAW → IM4P 时必须原样保留并修正长度**，否则签名/加载会失败 |
| **AEA** | Apple Encrypted Archive，Cryptex（`SystemOS`/`AppOS`）等镜像的加密分发格式，需要 key 才能解密（本项目用的是 iPhone17,3 恢复目录里的 `.dmg.aea`） |

### 2.2 启动链（Boot Chain）

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

### 2.3 其它名词

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

---

## 3. 原理：为什么需要这么多补丁

理解这一节，你就能看懂后面每一个 patch 的目的，而不是盲目抄偏移。

Apple 的安全模型是「逐级校验、任何一级被改就停」。我们要跑一个**被修改过的固件**，就必须
逐级把这些校验「骗过去」或者「绕过去」。作者用的策略不是「删掉整个校验函数」，而是
**只改返回值 / 打 `nop` / 放行特定路径**，尽量少地破坏原有逻辑。

### 3.1 `image4_validate_property_callback` —— 一切的钥匙

iBoot 每一级都调用 `image4_validate_property_callback()` 去校验下一级 IMG4 的摘要与属性。
把它改成「直接返回 0（成功）」，就能加载任意（被我们改过的）下一级镜像。

- AVPBooter 里：`Text-search "0x4447"`（`DG`，即 DGST 的 magic）→ 找到函数 → patch epilogue 返回 0。
- iBSS / iBEC / LLB 里：直接 `nop` + `mov x0, #0`。

这是理解整篇的**关键概念**：只要控制了 Stage 0（AVPBooter），后面的链条就全部「可加载」。

### 3.2 为什么要 bypass SSV

正常启动时 LLB 会验证根卷的密封（SSV）。但我们**必须修改根文件系统**（补 Cryptex、补 Metal 库、
改 launchd），一旦改动，SSV 就判定「seal is broken」并 panic。于是：

- 在 **LLB** 里放开「允许加载被编辑的 rootfs」的几处判断（为后面 `snaputil -n` 重命名快照铺路）；
- 在 **kernelcache** 里把顺序相关的三处 panic 检查打 `nop`：
  - `_apfs_vfsop_mount` → 防止 “Failed to find the root snapshot…”
  - `_authapfs_seal_is_broken` → 防止 “root volume seal is broken…”
  - `_bsd_init` → 防止 “rootvp not authenticated after mounting…”

（源码里注释得很清楚，见后面的[补丁速查表](#6-固件补丁速查表)。）

### 3.3 为什么要 patch TXM / TrustCache

我们往根文件系统里塞进去的二进制**不在 Apple 的 TrustCache 里**，TXM 会拒绝执行。
所以把 TXM 中「未注册即拒绝」的几个分支改成「当作已注册」。作者在补丁注释里保留了
触发时的报错原文（`TXM [Error]: CodeSignature: selector: 24 | 0xA8 | 0x30 | 1`），
方便你对着日志定位。

### 3.4 Cryptex 为什么必须手动补

PCC 固件/虚拟 iPhone 没有像真机那样把 Cryptex 正常恢复到设备上，导致启动后
**缺 `libSystem.B.dylib`（它在 Cryptex 的 dsc 里）**，launchd 直接挂掉。
解决办法：用 **SSH Ramdisk 拿到 root shell**，手动把 **iPhone17,3 恢复目录里**的 Cryptex
解密、挂载、拷贝进 `System/Cryptexes/{OS,App}`，并建好软链接。

### 3.5 seputil / launchd_cache_loader / mobileactivationd

- **seputil**：报 `Gigalocker file (/mnt7/XXXX.gl) doesn't exist`，是因为虚拟机的
  `.gl` 文件名和固件里写死的期望不一致 → 把它 patch 成固定找 `AA.gl`，并真的把文件重命名为 `AA.gl`。
- **launchd_cache_loader**：会加载缓存的 `launchd.plist`。我们改了 plist，所以要允许「不安全缓存」
  （源码注释：`launchd_unsecure_cache=1`）。
- **mobileactivationd**：绕过激活锁（hactivation = 激活状态标记；让 `should_hactivate` 永远返回真）。
- **launchd** 本体：防止 jetsam 属性未初始化导致的 `initproc exited` panic。

### 3.6 Metal 为什么一开始不可用

内核其实**已经认得** `AppleParavirtGPU`（`ioreg -l` 能看到），缺的是用户态的
`AGXMetal*` 类驱动。真机上是 `/System/Library/Extensions/AGXMetalA10` 之类。
于是作者把 PCC 里的 `/System/Library/Extensions/AppleParavirtGPUMetalIOGPUFamily.bundle`
整体搬进虚拟机，Metal 就通了；其中还缺一个 dylib，作者**反向重写**了一个
`libAppleParavirtCompilerPluginIOGPUFamily.dylib`（这也是仓库里唯一缺源码的部分）。

---

## 4. 环境准备

### 4.1 硬件与系统

- **必须是 Apple Silicon Mac**（M 系列）。Intel 不行。
- 内存建议 **16GB 起**，磁盘留足 **40GB+**（固件、ramdisk、`pcc-research` 与 `vphone` 两台虚拟机都很占空间）。
- 原作者验证过的组合：
  - Apple M3 / 16GB / **Sequoia 15.7.4**
  - Apple M1 Pro / 32GB / **Tahoe 26.3**
- 理论上任何支持 `pccvre` 的机型都可以。

### 4.2 关闭宿主安全限制（**仅限专用机器**）

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
> 对比一下：更现代的 [vphone-cli](#8-兼容性与更省事的替代方案) 只需要
> `csrutil enable --without debug` + `allow-research-guests enable`，**SIP 可以保持开启**，
> 这一点上更友好。

### 4.3 工具链

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

还需要两个**改造版**仓库（后面 Step 4 会编译）：

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

---

## 5. 分阶段复现

> 下面的命令与 `GUIDE.md` / `CFW/*.py` 保持一致。每完成一步都给了「检查点」，方便你确认没走偏。

### Step 1. 宿主安全设置

见 [4.2](#42-关闭宿主安全限制仅限专用机器)。**先做完重启，再做后面的事。**

**检查点**：`csrutil status` 显示 disabled；恢复模式里 `allow-research-guests` 已开启。

### Step 2. 用 pccvre 造一台 PCC 研究虚拟机

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
> 包括正确的磁盘、SEP 存储和 NVRAM。

### Step 3. 用 super-tart 搭出 vphone 虚拟机骨架

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

然后清掉不需要的文件，把 Step 2 里 pcc-research 的磁盘/SEP/NVRAM 拷进来，
再放入 vresearch101 的 bootrom：

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

### Step 4. 编译 super-tart-vphone 与改造版 libirecovery

```sh
# 先回到一个干净的开发目录再克隆（不要留在 ~/.tart/vms/vphone）
cd ~/Developer

# 1) 改造版虚拟机软件
git clone --recursive https://github.com/wh1te4ever/super-tart-vphone
cd super-tart-vphone
./scripts/run-signed.sh
# 用改造版跑起来（--dfu 表示进入 DFU 模式）
./.build/debug/tart run vphone --dfu

# 2) 支持 vresearch101ap 的 irecovery
git clone --recursive https://github.com/wh1te4ever/libirecovery
cd libirecovery
./autogen.sh && make -j8 && sudo make install
```

改造点在 `Sources/tart/VM.swift`（对照原 README 的代码块），核心是：

```swift
// 平台类型 = vresearch101
hw_descriptor.setPlatformVersion(3)   // 3 = vrevm 里 PlatformType.vresearch101 的原始值
hw_descriptor.setBoardID(0x90)        // 0x90 = 144 = vresearch101ap 的 BDID
hw_descriptor.setISA(2)               // 原作者这里用字面量 2；Apple 源码写的是 .appleInternal4。若启动失败可试 3
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

// 触摸（Tahoe/26 上直接可用；Sequoia 需另走 VNC，见 Step 11）
let touch = _VZUSBTouchScreenConfiguration()
configuration._setMultiTouchDevices([touch])

// 显示：注意这里 writeup 正文写 1179×2556，官方是 1290×2796（iPhone 16 Plus 规格）
let displays_config = VZMacGraphicsDisplayConfiguration(
    widthInPixels: 1179, heightInPixels: 2556, pixelsPerInch: 460)
```

**检查点**：运行 `.build/debug/tart run vphone --dfu` 后，在
**系统信息 App → 硬件 → USB** 里能看到一个 DFU 设备。看到它，才说明虚拟化和 bootrom 都接上了。

### Step 5. 打补丁：AVPBooter.vresearch1.bin

这是让「自定义固件」能被加载的第一步。参考 [steven-michaud 的 gist](https://gist.github.com/steven-michaud/fda019a4ae2df3a9295409053a53a65c)
（iBoot Stage 0 一节）的思路，用 IDA Pro / Ghidra：

1. 打开 `~/.tart/vms/vphone/AVPBooter.vresearch1.bin`，处理器类型设为 **ARM Little-endian / 64-bit**；
2. 用 `Text-search (slow!)` 搜 `0x4447`（`DG`，DGST 的 magic），定位到
   `image4_validate_property_callback`；
3. 找到函数 **epilogue**，把返回前的 `mov x0, x20`（或类似）改成 `mov x0, #0`，让它永远返回成功；
4. 保存。

> 更稳妥的传法是「只改返回值、不破坏函数体」（gist 作者推荐的做法）；原作者这里用的是
> 「直接让 epilogue 返回 0」。两种都行，前者更安全。
>
> 顺便记录一下 APNonce（GUIDE 里给了示例 `7c2db09d…`）；Step 7 抓 SHSH 若报 nonce 不匹配，
> 回到这里核对。

**检查点**：用二进制 diff 工具确认只有一处 4 字节被改动。

### Step 6. 组装「混血」自定义固件

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
4. 用仓库里 **`custom_26.1/BuildManifest.plist` 和 `Restore.plist`** 覆盖（这正是 README 里
   「我忘了具体怎么改」的那部分——答案就在仓库里）。

这两个 plist 的关键差异（`contents/Restore.plist` 可见）：

- **DeviceMap** 同时登记了三种主板：`d47ap`（iPhone 16）、`vphone600ap`、`vresearch101ap`；
- **SupportedProductTypes** 同时包含 `iPhone17,3`、`iPhone99,11`、`ComputeModule14,x`、`Mac14,14`；
- **SystemRestoreImageFileSystems** 指向两个 APFS 的 `.dmg.aea`（即 Cryptex SystemOS / AppOS）。

`patch_fw.py` 做的事（**所有偏移都是 26.1 / 23B85 / vphone600 专用**）：

- iBSS / iBEC / LLB：把 `image4_validate_property_callback` 改成返回 0；
- iBSS 另外打一处（`0x1b544`）不重新生成 APNonce，保持 nonce 可复现；
- iBEC 与 LLB 另外写入 boot-args `"serial=3 -v debug=0x2014e %s"`（打开串口日志，方便看启动过程）；
  LLB 再额外放开 rootfs 编辑；
- TXM：放行未注册二进制 + 打开 debugger / developer mode 相关；
- kernelcache：bypass SSV + 一堆给越狱用的 hook。

> **IM4P → RAW → IM4P 的坑**：kernelcache 和 TXM 的 IM4P 末尾有 **PAYP** 结构。
> 直接用 `pyimg4 im4p create` 重打包会丢掉它，必须：
> 从原始 `.bak` 里找到 `PAYP` 偏移，把**从 PAYP magic 前 10 字节开始**的整段尾部
> （`data[payp_offset-10:]`）**追加**回去；这 10 字节的长度也要加进 IM4P 头部长度字段
> （`data[2:5]`，大端）。`patch_fw.py` 里已经写好了这段逻辑，照抄即可。

**检查点**：`patch_fw.py` 跑完不报错；`iPhone17,3_26.1_23B85_Restore/` 下留有一堆 `.bak` 备份。

### Step 7. 进入 DFU，抓 SHSH，恢复固件

```sh
# 注意：Step 7 起，tart 命令在仓库根目录 super-tart-vphone/ 下执行，
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
- 恢复过程中 **SEP 配置错误会导致 panic**（见
  [`contents/image 2.png`](contents/image%202.png)）。如果 panic 里指向 SEP，
  回去检查 Step 3 的 `SEPStorage` / `AVPSEPBooter` 是否拷对。

恢复完成后虚拟机会自动重启，然后 **panic**：原因是 `/usr/lib/libSystem.B.dylib` 缺失
（它属于 Cryptex 的 dsc）。**这是预期行为**，我们用下一步的 SSH Ramdisk 来修。

> 注：仓库里还有 `CFW/fake_tss_server.py` + `tss_response`，用于在无法联网/无法拿正规
> ticket 时伪造 TSS 响应，属于进阶用法。

### Step 8. 制作并启动 SSH Ramdisk

思路借鉴 [verygenericname/SSHRD_Script](https://github.com/verygenericname/SSHRD_Script)：
在 DFU 下，把「打过补丁的 iBSS/iBEC + SPTM + TXM + devicetree + SEP + kernelcache + 一个
带 SSH 的 ramdisk」逐个用 `irecovery` 上传并加载。

```sh
cd ~/Developer/super-tart-vphone/CFW

# 把 Step 7 抓到的 shsh 放到 CFW/vphone.shsh（Step 7 是在 CFW/ 下执行的）
cp ./shsh/*-iPhone99,11-26.1.shsh ./vphone.shsh

python3 get_rd.py        # 生成 Ramdisk/*.img4（含自建 trustcache、重签名）
./boot_rd.sh             # 逐个 irecovery -f ... 上传并启动

iproxy 2222 22 &         # 把虚拟机 22 端口映射到本机 2222
```

`boot_rd.sh` 的内容（顺序很重要）：

```sh
irecovery -f Ramdisk/iBSS.vresearch101.RELEASE.img4
irecovery -f Ramdisk/iBEC.vresearch101.RELEASE.img4
irecovery -c go
sleep 1
irecovery -f Ramdisk/sptm.vresearch1.release.img4      ; irecovery -c firmware
irecovery -f Ramdisk/txm.img4                          ; irecovery -c firmware
irecovery -f Ramdisk/trustcache.img4                   ; irecovery -c firmware
irecovery -f Ramdisk/ramdisk.img4                      ; irecovery -c ramdisk
irecovery -f Ramdisk/DeviceTree.vphone600ap.img4       ; irecovery -c devicetree
irecovery -f Ramdisk/sep-firmware.vresearch101.RELEASE.img4 ; irecovery -c firmware
irecovery -f Ramdisk/krnl.img4                         ; irecovery -c bootx
```

`get_rd.py` 里两个值得注意的点：

- **自建 trustcache**：ramdisk 里的可执行文件都要能被 TXM 认可，所以对它们重新 `ldid` 签名，
  并用 `trustcache_macos_arm64` 重新生成 trustcache 再打回固件；
- **重新打包 ramdisk**：`hdiutil create` 成一个更大的 APFS 镜像，塞进 SSH 工具，再 `resize -sectors min`。

**检查点**：屏幕上出现「Minecraft Creeper 脸」（即所谓“creeper 界面”，ramdisk 已启动的
标志）；系统信息 → USB 里能看到 “iPhone Research…”。然后：

```sh
ssh root@127.0.0.1 -p2222      # 密码：alpine
```

### Step 9. 修好首次启动（Cryptex / seputil / launchd）

保持 `iproxy` 和 SSH 连接，先重命名 rootfs 快照（这会打破 SSV，正是我们需要 LLB 补丁的原因）：

```sh
mount_apfs -o rw /dev/disk1s1 /mnt1
snaputil -l /mnt1
# 记下输出的哈希，例如：
# com.apple.os.update-8AAB8DBA5C8F1F756928411675F4A892087B04559CFB084B9E400E661ABAD119
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
   Metal，见 Step 10）；
4. **安装 `iosbinpack64`**（一个轻量越狱工具集，含 bash/dropbear 等）；
5. **patch `launchd_cache_loader`**（允许加载被修改的 `launchd.plist`）；
6. **patch `mobileactivationd`**（绕过激活）；
7. **patch `launchd`**（防 jetsam panic）；
8. **注入 LaunchDaemons**：把 `bash.plist` / `dropbear.plist` / `trollvnc.plist` 加入
   `launchd.plist`，让它们开机自启（这样正常启动后也能 SSH / VNC）；
9. 收尾：在虚拟机内卸载 `/mnt1`、`/mnt3`，宿主机侧 detach 两个 Cryptex 镜像。

**检查点**：脚本无报错。注意 `fix_boot.py` **不会自动关机**，需要你再执行一次
`ssh root@127.0.0.1 -p2222 halt`（GUIDE 第 7 节），虚拟机应自动关机。

### Step 10. 第二次启动与 Metal 加速

```sh
cd ~/Developer/super-tart-vphone
./.build/debug/tart run vphone --vnc-experimental
```

现在应该能进到有背景的**设置界面**了。

**Metal** 部分：
- 先用原作者提供的 `MetalTest`（`CFW/jb/MetalTest/main.m`）验证：
  `MTLCreateSystemDefaultDevice()` 一开始返回 `null` → Metal 不可用；
- `ioreg -l` 能看到内核已识别 `AppleParavirtGPU`，缺的是用户态驱动；
- 你需要**自己**从 PCC 虚拟机里取出 `/System/Library/Extensions/AppleParavirtGPUMetalIOGPUFamily.bundle`，
  打包成 `custom_26.1/AppleParavirtGPUMetalIOGPUFamily.tar`（仓库里没有这个文件，Step 9 的脚本
  只负责把它解开到 `/mnt1/System/Library/Extensions/`）；
- 再跑 `MetalTest`，应输出 `Apple Paravirtual device`。

> ⚠️ **诚实提示**：原作者因为版权顾虑，**没有公开** `libAppleParavirtCompilerPluginIOGPUFamily`
> 的源码（仓库里只有 `build.sh` 和证书，没有 `main.m`）。所以严格意义上，**完整 Metal 复现
> 需要你自己参照 PCC 的 dsc 去反向实现这个 dylib**。如果你只想跑起系统，跳过它即可。

### Step 11. 交互、VNC 与（可选）越狱

- **Home 键**：键盘 **CMD + H** 模拟 Home。
- **触摸**：
  - Tahoe（26）上，`VZVirtualMachineView` 直接支持触摸（`_VZUSBTouchScreenConfiguration`）；
  - Sequoia（15）上不行，原作者通过**重写鼠标事件**实现，代码见
    [`contents/ScreenSharingVNC.swift`](contents/ScreenSharingVNC.swift)。
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

> 越狱属于完全可选的「附加题」，与「跑起虚拟 iPhone」无关，建议先确保前 10 步稳定再做。

---

## 6. 固件补丁速查表

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

## 7. 故障排查清单

| 现象 | 可能原因 / 处理 |
| --- | --- |
| `tart run vphone --dfu` 后 USB 里没有设备 | 宿主 SIP/AMFI/研究来宾未正确设置；`AVPBooter`/`SEPStorage` 没拷对 |
| 恢复阶段 panic，日志提到 SEP | `SEPStorage` 或 `AVPSEPBooter.vresearch1.bin` 缺失/路径不对 |
| 恢复完成重启后 panic，缺 `libSystem.B.dylib` | **正常现象**，必须走 SSH Ramdisk 补 Cryptex |
| SSH 连不上 2222 | `iproxy` 是否在跑；ramdisk 是否真的起来了（看 creeper 界面） |
| `seputil` 报 `Gigalocker ... doesn't exist` | seputil 未打补丁，或 `.gl` 未重命名为 `AA.gl` |
| 改了 `launchd.plist` 后不生效 | `launchd_cache_loader` 未打补丁 |
| MetalTest 返回 null | `AppleParavirtGPUMetalIOGPUFamily.bundle` 未安装，或缺作者未公开的 dylib |
| 首次正常启动能进（黑底）设置页，但一操作就 respring（SpringBoard 重启、回到主屏） | 缺用户态 GPU/Metal 驱动（见 Step 10）；先装 `AppleParavirtGPUMetalIOGPUFamily.bundle`，仍不行说明缺作者未公开的 `libAppleParavirtCompilerPluginIOGPUFamily.dylib` |
| panic: `initproc exited … jetsam property category (Daemon) is not initialized` | `launchd` 的 jetsam patch 未生效（`fix_boot.py` 第 7 步） |
| `idevicerestore` 报签名/ticket 错误 | SHSH 未抓到或 APNonce 变了；可考虑 `fake_tss_server.py` |
| 换固件版本后全部 panic | **预期**：所有偏移都是硬编码，需重新定位 |

---

## 8. 兼容性与更省事的替代方案

- **硬件**：仅 Apple Silicon；两台已实测的机器见 [4.1](#41-硬件与系统)。
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

## 9. 法律、伦理与风险提示

1. **仅在你拥有或有权处置的设备上操作。** 关 SIP / AMFI、改宿主机根卷、改 bootrom 都会
   降低这台 Mac 的安全性，请用专用测试机。
2. 固件、Cryptex、GPU 驱动等**版权归 Apple 所有**。原作者已因版权顾虑下架了一部分源码；
   本文档只解释原理，不提供任何 Apple 二进制。
3. 该技术属于**安全研究**范畴。请遵守 Apple 的 PCC 研究计划条款与当地法律，不要用于
   攻击真实用户设备或绕过他人设备的激活。
4. 你可能会**变砖或无法联网**（Xcode 调试所需的开发者设置、TSS 等），提前做好备份与恢复计划。

---

## 10. 参考链接

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

*本文为社区中文解读，非 Apple 官方文档。如果你在复现中发现本文与原始脚本/仓库有出入，
以原始仓库为准，并欢迎修正本文。*

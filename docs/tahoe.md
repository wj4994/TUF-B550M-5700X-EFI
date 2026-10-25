# macOS 26 Tahoe 支持评估（15 → 26 升级）

> 结论先行：**内核补丁表和 OpenCore/kext 底座已经够用，不需要为 26 改**。真正要动的是三处——模拟音频、CPU 传感器 kext、OTA 的 `revpatch=sbvmm`；外加 WhateverGreen 在 26 上的 GPU 风险项。本文所有判断基于本仓库已提交的 `OC/config.plist` 实测比对，非推测。

## TL;DR

| 项目 | 现状 | 跑 26 要不要改 |
|---|---|---|
| 内核补丁表（25 条） | `MaxKernel=25.99.99` | **不用改**，Darwin 25 覆盖 26.0 ~ 26.7.x |
| OpenCore 1.0.8 / Lilu 1.7.2 / RestrictEvents 1.1.6 | 均含 26 修复 | **不用改** |
| SMBIOS `MacPro7,1` | 26 支持机型含 Mac Pro (2019) | **不用改** |
| `SecureBootModel=Disabled` | 已满足 26 的 OTA 前提 | **不用改** |
| `boot-args` | 缺 `revpatch=sbvmm` | **要加**，否则收不到 26 更新 |
| 模拟音频 `alcid=1` + AppleALC | 26 删除了 `AppleHDA.kext` | **会失效**，需绕过 |
| `AMDRyzenCPUPowerManagement` / `SMCAMDProcessor` | 上游停在 Ventura 时代 | **要换 fork 或禁用** |
| `WhateverGreen` + `agdpmod=pikera` | 26 有已知 connector patch 问题 | **先试，panic 再退** |

## 一、15 与 26 的版本差异

### 版本定位

| | macOS 15 Sequoia | macOS 26 Tahoe |
|---|---|---|
| 发布 | 2024-09 | 2025-09-15（本文写作时最新 26.7.1） |
| Darwin 内核 | 24.x | 25.x |
| 支持的 Intel Mac | MBP 2018+、MBA 2020+、iMac 2019+、mini 2018+、Mac Pro 2019 | **仅 4 款**：MBP 16" 2019、MBP 13" 2020（4×TB3）、iMac 2020、Mac Pro 2019 |
| 版本号 | 15（旧体系） | 26（统一到年份季，跳过 11 个号） |
| 生命周期 | 正常滚动 | **最后一版支持 Intel**；macOS 27「Golden Gate」仅 Apple Silicon |
| 升级途径 | App Store 可取完整安装包 | 首个无法从 App Store 升级的版本，只能走软件更新/恢复 |

### 功能差异

| 类别 | Tahoe 26 新增 |
|---|---|
| 界面 | Liquid Glass 全面重设计（半透明面板、菜单栏可全透明、图标着色） |
| 个性化 | 文件夹单独改色、加 emoji/符号 |
| 控制中心 | 支持大量第三方与 iPhone 小组件 |
| Spotlight | 重构，支持直接执行动作、剪贴板历史 |
| 连续性 | Live Activities、Mac 上的 Phone app（配合 iPhone Mirroring） |
| 通讯 | Messages 陌生联系人筛选/群聊背景、FaceTime 实时翻译 |
| 其他 | Games app、Journal、Password app 改进、Accessibility Reader |
| 图形 API | **Metal 4 仅 Apple silicon**；Metal 3 名单不变（AMD 5000/6000 系、Vega 仍是 Metal 3） |

### Intel 上拿不到的

26 的新功能大半长在 Apple Intelligence 上，而 AI 从 Sequoia 起就要求 M1+：Live Translation、AI Shortcuts、Genmoji 增强、Image Playground + ChatGPT、Messages 智能建议、Repair Assistant 一律无缘。所以 Intel 用户升 26，实际到手的是 Liquid Glass、重构版 Spotlight、Phone app / Live Activities 这几项。

## 二、对本机（B550M + 5700X + RX 5600 XT）的具体差异

| 维度 | 15 Sequoia | 26 Tahoe |
|---|---|---|
| 内核补丁 | Darwin 24 分支 | Darwin 25 分支，同一份表即可 |
| 模拟音频（ALCS1200A） | `alcid=1` 正常 | **失效**，只剩 HDMI/DP 数字输出 |
| 显卡 | `AMDRadeonX6000` + `agdpmod=pikera` | 驱动仍在，但 WEG 有 panic 风险；Metal 上限仍是 Metal 3 |
| CPU 频率/温度 | AMDRyzenCPUPowerManagement 可用 | 上游无 26 支持 |
| OTA | 14.4+ 需 `revpatch=sbvmm` | 同样需要，且 26 只能靠 OTA |
| SIP | 可全开 | 若要救回音频必须降 SIP + root patch |
| 稳定性 | 已实测 15.8.1 通过 | 未验证；Liquid Glass 更吃 GPU |

## 三、要做的改动

### 1. boot-args 加 `revpatch=sbvmm`（生产版就该加）

macOS 14.4 起 OTA 必须 RestrictEvents + 该参数，26 只能走 OTA，所以这是硬前提。`RestrictEvents 1.1.6` 与 `SecureBootModel=Disabled` 本仓库已满足。

```
agdpmod=pikera alcid=1
→ agdpmod=pikera alcid=1 revpatch=sbvmm
```

想让「关于本机」也正常显示，可用 `revpatch=auto,sbvmm`。

用 ProperTree/Xcode 手改，或临时脚本（仓库里的 `tools/build_config.py` 目前缺失，见文末「已知坑」）：

```bash
python3 - <<'PY'
import plistlib
p = 'OC/config.plist'
d = plistlib.load(open(p, 'rb'))
k = '7C436110-AB2A-4BBB-A880-FE41995C9F82'
d['NVRAM']['Add'][k]['boot-args'] = 'agdpmod=pikera alcid=1 revpatch=sbvmm'
plistlib.dump(d, open(p, 'wb'), sort_keys=False)
PY
plutil -lint OC/config.plist
```

### 2. 音频：26 没有 AppleHDA，`alcid=1` 变成空转

`AppleHDA.kext` 在 26 中被移除，AppleALC 无从挂载，ALCS1200A 的模拟输出直接失效。三条路：

| 方案 | 代价 |
|---|---|
| OCLP 3.0 nightly / OCLP-Mod 打 root patch 重注入 AppleHDA | 需把 `csr-active-config` 从 `00000000` 降到 `03000000` 起；每次系统更新后要重打 |
| VoodooHDA（降 SIP + 放 `/Library/Extensions`） | 音质下降，系统变"脏" |
| 放弃模拟音频，只走 HDMI/DP / USB 声卡 | 零成本 |

保留 AppleALC kext 与 `alcid=1` 无害（数字输出仍依赖音频栈），不必删。

### 3. CPU 传感器 kext 二选一

`AMDRyzenCPUPowerManagement 0.7.2` + `SMCAMDProcessor 1.0.1` 无 26 支持声明：

- 换 **DrogaBox/SMCAMDProcessor-personal**（声明 Zen1~Zen5、Ventura ~ Tahoe）；
- 或把两者在 `Kernel/Add` 里置 `Enabled=false`——只丢 AMD Power Gadget 读数，稳定性更保险。

### 4. GPU：先试，panic 再退

26 上 WhateverGreen 的 AMD connector patch 有已知问题。仓库用的是 `WhateverGreen 1.7.1`（当前最新正式版），社区有 26.3 + 1.7.1 正常运行的报告，**先按现状升级**；若出现 AMD GPU / WEG 相关 panic，按官方建议彻底移除 WEG，并把 `agdpmod=pikera` 换成 `Kernel/Patch` 里的手工 `AppleGraphicsDevicePolicy` 补丁。

### 5. 不需要动的（已核对）

- **内核补丁表**：与 AMD_Vanilla 官方 `patches.plist` 逐条比对，25 条完全一致，仅 4 条 `cpuid_cores_per_package` 因 8 核（`0x08`）不同；`MaxKernel=25.99.99` 即 Darwin 25，覆盖 26 全系列。
- **OpenCore 1.0.8 / Lilu 1.7.2**：后者 release notes 明确「Fixed AMDSupport lockups & panics in macOS 26」。
- **RestrictEvents 1.1.6**：release notes「Added constants for macOS 26 support」。
- **SMBIOS / SecureBootModel / APFS jumpstart / USBMap / LucyRTL8125 / VoodooPS2**：无 26 相关问题。
- `run-efi-updater=No` 保留；`FixupAppleEfiImages=false` 在不动固件更新策略时也无需改。

### 复现核对的方法

```bash
curl -sSLo /tmp/av.plist https://raw.githubusercontent.com/AMD-OSX/AMD_Vanilla/master/patches.plist
python3 - <<'PY'
import plistlib
repo = plistlib.load(open('OC/config.plist','rb'))['Kernel']['Patch']
up   = plistlib.load(open('/tmp/av.plist','rb'))['Kernel']['Patch']
sig = lambda x: (x.get('Comment'), x.get('Find'), x.get('Replace'), x.get('Mask'),
                 x.get('ReplaceMask'), x.get('Count'), x.get('MinKernel',''),
                 x.get('MaxKernel',''), x.get('Enabled'))
rs, us = {sig(x) for x in repo}, {sig(x) for x in up}
print('only in repo:', sorted(c[0] for c in rs-us))
PY
# 预期只剩 4 条 cpuid_cores_per_package（核心数 8 核定制）
```

## 四、升级步骤与回滚

1. 先只加 `revpatch=sbvmm`，其余不动；
2. 用调试配置做首启：`cp tools/config.debug.plist OC/config.plist`（首启务必带 `-v`）；
3. 收工切回生产版：`git restore OC/config.plist`，确认 `revpatch` 那行已在提交里；
4. 升级后依次验：能否进桌面（Navi 10 + WEG）、模拟音频是否如预期失效、AMD Power Gadget 是否有读数；
5. 再决定音频走 OCLP 还是放弃、传感器 kext 换 fork 还是禁用；
6. 全程保留 15.8.1 快照——WEG 若 panic，得靠旧快照回来改 EFI。

## 五、不可解决项

- MT7921 Wi-Fi：26 依旧无驱动。
- 换 Broadcom 免驱卡的常规路线 `AppleBCMWLANCompanion` 需要 VT-d，AMD 平台不可用，只能走 OCLP root patch。
- `Intel Bluetooth` 的 `-ibtcompatbeta` 与本机无关。

## 六、已知坑

- **`tools/build_config.py` 不在仓库里**（`tools/` 只有 `config.debug.plist`），README「维护」一节的 OC 升版流程目前跑不通。本次因 OC 已是最新无需重建配置，但下次 OC 升版会卡住。
- 补丁上限已到顶：`25.99.99` 之后不存在有意义的 `26.99.99`，因为 Darwin 26 = macOS 27 已不含 x86_64 内核。

## 参考

- Dortania：macOS 26 Tahoe 前置条件（AppleHDA 移除、SMBIOS 支持列表、OTA、WhateverGreen）<https://dortania.github.io/OpenCore-Install-Guide/extras/tahoe.html>
- AMD_Vanilla 官方补丁表 <https://github.com/AMD-OSX/AMD_Vanilla>
- Apple：macOS Tahoe 26 兼容机型 <https://support.apple.com/en-us/122867>
- Apple：Metal 支持列表（Metal 4 仅 Apple silicon）<https://support.apple.com/en-us/102894>
- Apple 确认 Tahoe 为最后一个支持 Intel 的 macOS（Bloomberg / Ars Technica）

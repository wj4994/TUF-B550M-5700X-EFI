# TUF GAMING B550M-PLUS WIFI II - OpenCore EFI

适用于 ASUS TUF GAMING B550M-PLUS WIFI II（华硕 B550M 重炮手 II）+ AMD Ryzen 7 5700X + RX 5600 XT 的 OpenCore EFI。

![screenshot.jpg](https://s2.loli.net/2022/09/25/fW8p3qxkQYnzVhI.jpg)

## 配置

| 组件 | 型号 |
|------|-----|
| 主板 | ASUS TUF GAMING B550M-PLUS WIFI II（BIOS 3621） |
| CPU | AMD Ryzen 7 5700X 8-Core Processor（8C/16T，Zen3） |
| 内存 | 32 GB DDR4 |
| 显卡 | Sapphire RX 5600 XT（Navi 10，`1002:731F`） |
| 硬盘 | Solidigm P44 Pro / WD SN550（NVMe） |
| 声卡 | Realtek ALCS1200A（`alcid=1`） |
| 有线网卡 | Realtek RTL8125 2.5GbE（`10EC:8125`） |
| 无线 | MediaTek MT7921 Wi-Fi 6（**无 macOS 驱动**） |

## 状态

- **已实机验证：macOS Sequoia 15.8.1**（本仓库 `OC/config.plist` 原样引导，无需临时改配置）。
- 支持范围：macOS Monterey 12 ~ Sequoia 15（12 ~ 15.8.1 之间未逐一回归；内核补丁上限 `25.99.99` 理论覆盖到 macOS 26 Tahoe，但 26 未验证）。
- 除 Wi-Fi、蓝牙外功能正常；MT7921 无解，需要无线请更换为 BCM94360NG 等免驱卡（主板 M.2 E-Key 可拆换）。
- 验证范围仅限上文「配置」表里的硬件（5700X + RX 5600 XT + RTL8125 + ALCS1200A）。换核心数、换非 Navi 10 显卡、换主板时，先按「故障排查」和「维护」两节调整再引导。
- 升到 macOS 26 Tahoe 需要改什么，见 [`docs/tahoe.md`](docs/tahoe.md)（结论：补丁表与 OC/kext 底座已够用，只剩音频、CPU 传感器、`revpatch=sbvmm` 三处）。

## 组件版本

| 组件 | 版本 | 说明 |
|------|------|------|
| OpenCore | 1.0.8 | `OpenCore.efi` + `BOOT/BOOTx64.efi` |
| Lilu | 1.7.2 | |
| VirtualSMC | 1.3.8 | AMD 上只提供 SMC 模拟 |
| WhateverGreen | 1.7.1 | 负责 Navi 10 的 `agdpmod=pikera` 处理 |
| AppleALC | 1.9.8 | ALCS1200A，layout 1 |
| RestrictEvents | 1.1.6 | MacPro7,1 上的内存/PCI 警告与 CPU 名 |
| LucyRTL8125Ethernet | 1.2.300 | RTL8125 rev 05 |
| VoodooPS2Controller | 2.3.7 | PS/2 键鼠（USB 键鼠无需） |
| AMDRyzenCPUPowerManagement | 0.7.2 | Ryzen 频率/温度 |
| SMCAMDProcessor | 1.0 | 配合上一个读取传感器 |
| AppleMCEReporterDisabler | 1.2 | 屏蔽 `AppleMCEReporter` 导致的随机重启 |
| USBMap | 1.0（自制） | 本主板端口映射，别换 |

内核补丁来自 [AMD_Vanilla](https://github.com/AMD-OSX/AMD_Vanilla) 官方 `patches.plist`（快照见 `tools/patches-amd-vanilla.plist`），共 25 条、启用 22 条，核心数已按 8 核（`0x08`）配置。

## 引导前必做

### 1. BIOS 设置

| 项目 | 值 |
|------|-----|
| Above 4G Decoding | Enabled |
| Re-Size BAR Support | **Disabled**（若保持开启，需依赖本配置里的 `ResizeAppleGpuBars=0`） |
| Secure Boot | Disabled / Other OS |
| CSM / Launch CSM | Disabled |
| Fast Boot | Disabled |
| Serial Port | Disabled（避免 0x3F8 冲突） |
| XHCI Hand-off | Enabled |
| Legacy USB Support | Enabled |
| IOMMU | Auto（若有问题改成 Disabled） |
| SVM Mode | Disabled（只影响 macOS；需要 KVM/Hyper-V 时保持开启，遇到虚拟化相关 panic 再关） |

### 2. SMBIOS

`PlatformInfo/Generic` 当前用的是**本机（5700X 这台）生成的 MacPro7,1 三码**：

| 键 | 值 | 说明 |
|---|---|---|
| `SystemSerialNumber` | `F5KKT8YPP7QM` | macserial 生成，解码为 MacPro7,1 / 2023 年第 23 周 |
| `MLB` | `F5K323200GUK3F7CB` | 与序列号配套，MLB 校验和已通过 |
| `SystemUUID` | `2930631C-B1FA-45A2-914E-3FA14058D7B5` | 随机 v4 UUID |
| `ROM` | `08:bf:b8:00:96:6e` | 本机 RTL8125 有线网卡 MAC |

换机器、或者想用自己的三码时，用 OpenCore 发行包自带的 `macserial`（Linux 版在 `Utilities/macserial/macserial.linux`，macOS 版用同一目录下的 `macserial`）：

```bash
# 1. 生成 5 组 序列号 | MLB（-m 指定机型，-g 生成，-n 数量）
./macserial -m MacPro7,1 -g -n 5
# 2. 校验：序列号能解码出 MacPro7,1，MLB 校验和有效
./macserial -i F5KKT8YPP7QM
./macserial --verify F5K323200GUK3F7CB
# 3. 取网卡 MAC（Linux: cat /sys/class/net/enp7s0/address；macOS: ifconfig en0 | grep ether）
# 4. 写回 config.plist（会顺带重建整份配置并跑通校验）
python3 tools/build_config.py --sample Docs/Sample.plist \
  --serial F5KKT8YPP7QM --mlb F5K323200GUK3F7CB \
  --uuid 2930631C-B1FA-45A2-914E-3FA14058D7B5 --rom 08:bf:b8:00:96:6e
```

四个参数都可省略，省略时保留 `config.plist` 里现有的值；`ROM` 若还是占位值 `11:22:33:44:55:66`，脚本会提示。

### 3. 准备安装盘与磁盘空间

本机现状（实测）：`nvme0n1` 953.9G 已被 Windows 占满；`nvme1n1` 931.5G = 4G ESP（挂载 `/boot`）+ 927.5G btrfs（仅用 73G）。

建议：

1. Btrfs 缩到 150–200G，腾出 700G+ 给 macOS；
2. macOS 建独立 APFS 容器（不要与 btrfs 混用同一分区）；
3. EFI 放进 `nvme1n1p1`（4G ESP，还剩 2.7G），或直接留在安装 U 盘上先用。

```bash
# 缩 btrfs（示例，操作前务必备份）
sudo btrfs filesystem resize 150G /
```

### 4. 制作安装 U 盘（Linux 下，无需 macOS）

macOS 15 的完整安装包 `InstallAssistant.pkg` 约 14.58 GiB，16 GB U 盘放不下（macOS 官方要求 ≥16 GB，建议 32 GB）。手上只有小容量 U 盘时，用**恢复模式 U 盘**：写入约 850 MB 的 `BaseSystem.dmg`，进恢复环境后联网下载系统。

```bash
# 1. 取恢复镜像（Board ID 对应 MacPro7,1）
curl -sSLo macrecovery.py \
  https://raw.githubusercontent.com/acidanthera/OpenCorePkg/master/Utilities/macrecovery/macrecovery.py
python3 macrecovery.py -b Mac-27AD2F918AE68F61 -m 00000000000000000 -o recovery download
# 产出 recovery/BaseSystem.dmg + BaseSystem.chunklist（下载后会自动用 chunklist 校验）

# 2. 分区 + 格式化：单个 FAT32 分区即可（GPT 的 EF00 或 MBR 的 0x0C 都能被 UEFI 识别）
sudo parted -s /dev/sdX mklabel gpt mkpart OPENCORE fat32 1MiB 100% set 1 esp on
sudo mkfs.vfat -F 32 -n OPENCORE /dev/sdX1

# 3. 拷贝：根目录下 EFI/ 与 com.apple.recovery.boot/
sudo mount /dev/sdX1 /mnt
sudo mkdir -p /mnt/EFI /mnt/com.apple.recovery.boot
sudo cp -r BOOT OC /mnt/EFI/
sudo cp recovery/BaseSystem.dmg recovery/BaseSystem.chunklist /mnt/com.apple.recovery.boot/
sync && sudo umount /mnt
```

`tools/make_macos_usb.sh` 是上面 2–3 步的自动化版本（需 root，默认 `/dev/sdb`，带容量与可移动性校验）。

开机按 F8 选 `UEFI: <U 盘>`，OpenCore 菜单里选 **macOS Base System**，进恢复模式后先用「磁盘工具」分区，再「安装 macOS Sequoia」。安装器会联网下载约 15 GB；RTL8125 由 `LucyRTL8125Ethernet.kext` 在恢复环境中即可用，前提是网线已接、路由器有 DHCP。

### 5. 不使用 U 盘安装（从内置盘引导恢复系统）

OpenCore 支持直接启动放在**任意 FAT32 分区根目录** `com.apple.recovery.boot/` 下的恢复镜像（菜单里带 `(dmg)` 后缀），所以不必插 U 盘——把 EFI 和恢复镜像放进内置 NVMe 的 ESP 即可。

```bash
# 0) 只读检查（无需 root）
./tools/install_to_internal.sh --check

# 1) 缩 btrfs，给 macOS 腾出 200G 空闲空间（保留 PARTUUID，fstab 不受影响）
sudo ./tools/install_to_internal.sh --shrink 200G

# 2) 把 OC + 恢复镜像拷进 ESP，并注册 UEFI 启动项（会自动备份原 EFI 目录）
sudo ./tools/install_to_internal.sh --esp
```

重启 → BIOS `F8` 选 `OpenCore`（或让它成为默认项）→ 菜单里选 **macOS Base System (dmg)** → 磁盘工具新建 APFS 容器 → 安装（需要网线，恢复镜像要下载约 14 GB 安装器）。

前置条件（脚本 `--check` 会自动验证）：`Misc/Security/DmgLoading = Signed`、`UEFI/Drivers` 启用 `OpenHfsPlus.efi`（官方 FAQ：恢复卷都是 HFS+）、`ScanPolicy = 0`（全盘扫描）、`UEFI/APFS/EnableJumpstart = true`（装完后能读 APFS 卷）。

## 目录结构

```
BOOT/BOOTx64.efi          OpenCore Bootstrap
OC/OpenCore.efi           主程序
OC/config.plist           配置（ocvalidate 1.0.8 校验通过）
OC/ACPI/                  SSDT-CPUR.aml、SSDT-EC-USBX.aml
OC/Kexts/                 11 个 kext（含 VoodooPS2 的 4 个插件）
OC/Drivers/               OpenRuntime / OpenHfsPlus / OpenCanopy / ResetNvramEntry
OC/Tools/                 OpenShell / ResetSystem（均默认不启用）
OC/Resources/             OpenCanopy 主题（Chardonnay / GoldenGate / Syrah）
tools/build_config.py     用官方 Sample.plist 重新生成 config 的脚本
tools/patches-amd-vanilla.plist  AMD_Vanilla 补丁快照
docs/tahoe.md             15 → 26 差异与升级改动清单
```

合计约 17 MB（`OC/Resources/Audio` 的 6 MB 开机音资源可删，`AudioSupport=false` 用不到）。

## 相对 main 的改动

### 适配本机硬件

1. **核心数补丁 12 → 8**：`0C` 改 `08`，对应 Ryzen 7 5700X。
2. **移除 `SSDT-NAVI.aml`**：原作者用它把 RX 6650 XT 伪装成 RX 6600/6600 XT（`device-id=0x73FF`），注入路径写死 `_SB.PCI0.GPP8.X161/PBR0/GFX0`。RX 5600 XT 是 Navi 10（`1002:731F`），由 `AMDRadeonX6000` 原生驱动，只需 WhateverGreen + `agdpmod=pikera`。
   - 若你用的是 RX 6600/6650 XT 等 RDNA2 型号，请按本机真实 ACPI 路径重写该 SSDT 而不是直接套用。

### 升级到 OpenCore 1.0.8 / 支持 Sequoia

3. OpenCore 0.8.x（作者自编译，无版本号）→ **1.0.8**，`config.plist` 按 1.0.8 的 `Sample.plist` 重建（补齐 `FixupAppleEfiImages`、`ClearTaskSwitchBit`、`DisableIoMapperMapping`、`GopBurstMode` 等新键）。
4. 内核补丁表换成 AMD_Vanilla 官方版：17 条 → **25 条**，上限 `25.99.99`。新增/更新了 13.3+ 的 5 字节核心数补丁、15.0+ 的 leaf7 补丁、Sequoia 的 PAT 补丁，以及 B550/X570 在 12+ 上必须的 CaseySJ `probeBusGated` / `IOPCIIsHotplugPort`、Visual 的两条非单调时钟补丁。
5. Kext 全部升级到最新版（见上表），新增 `AMDRyzenCPUPowerManagement`/`SMCAMDProcessor` 更新到 0.7.2f1。
6. 驱动精简为 4 个并移除 `OpenUsbKbDxe.efi`——它与 `UEFI/Input/KeySupport=true` 冲突（ocvalidate 报错），1.0.x 的输入栈不再需要它。
7. `Booter/Quirks/ResizeAppleGpuBars` 设为 `0`：BIOS 已开启 Re-Size BAR 时，让 macOS 只看到最小 BAR。
8. `NVRAM`：`prev-lang:kbd=zh-Hans:252`（首启中文键盘）；`boot-args` 默认生产态 `agdpmod=pikera alcid=1`，调试态带 `-v keepsyms=1 debug=0x100`（见「调试配置」一节）。
9. `Misc/Boot/Timeout` 2 → 5 秒，`Misc/Tools` 只保留真实存在的两个工具。

## 故障排查

| 现象 | 处理 |
|------|------|
| 卡 `[EB|#LOG:EXITBS:START]` | 依次试：关 BIOS 的 SVM/IOMMU、`DevirtualiseMmio=true`、`SetupVirtualMap=true` |
| 卡 `apfs_module_start` | 检查 `SetApfsTrimTimeout=-1`、BIOS SATA/AHCI、`EnableWriteUnprotector` 保持 false |
| 有 Apple 进度条但黑屏/无加速 | 确认 `SSDT-NAVI.aml` 已删除，`agdpmod=pikera` 在 boot-args 中 |
| 启动即随机重启 | 先禁用 `AMDRyzenCPUPowerManagement` / `SMCAMDProcessor` 试；保留 `AppleMCEReporterDisabler` |
| USB 端口不对（前面板/后置错位） | 用 CorpNewt `USBMap` 在本机重新映射（`WIFI II` 与作者同型号，正常可直接用） |
| 需要看 OpenCore/boot.efi 日志 | 用调试配置：`cp tools/config.debug.plist OC/config.plist`（见下节） |
| 装完系统后想关调试输出 | 已默认关闭；从调试配置回退即恢复 `-v keepsyms=1 debug=0x100` + `AppleDebug/ApplePanic` |
| Secure Boot 相关报错 | `Misc/Security/SecureBootModel` 改 `Disabled`，同时把 `Booter/Quirks/FixupAppleEfiImages` 设为 `true`（两者必须成对，见配置文档 8.10）。当前配置是 `Disabled` + `false`，因为 `run-efi-updater=No` 已挡住 Apple 固件更新，15.8.1 实测不受影响；一旦把 `run-efi-updater` 去掉或改成 `Yes`，就要同时把 `FixupAppleEfiImages` 设为 `true` |

## 调试配置

仓库里保留两份配置，`OC/config.plist` 是**生产版**（默认使用）：

| 项 | 生产版（`OC/config.plist`） | 调试版（`tools/config.debug.plist`） |
|---|---|---|
| `boot-args` | `agdpmod=pikera alcid=1` | `-v keepsyms=1 debug=0x100 agdpmod=pikera alcid=1` |
| `Misc/Debug/AppleDebug` | `false` | `true` |
| `Misc/Debug/ApplePanic` | `false` | `true` |

切换方式（`tools/config.debug.plist` 只作留档，不要覆盖它）：

```bash
cp tools/config.debug.plist OC/config.plist   # 排错：切到调试版
git restore OC/config.plist                   # 收工：回到生产版（需已提交）
```

调试版当前的 `Misc/Debug/Target = 0`，日志会被丢弃；真要抓日志还需把 `Target` 设为 `67`（`0x01|0x02|0x40`，日志+屏幕+写文件）并换用 DEBUG 版 `OpenCore.efi`/`BOOTx64.efi`/`OpenRuntime.efi`，日志会落在 ESP 根目录 `opencore-YYYY-MM-DD-HHMMSS.txt`。

## 维护

升级 OpenCore 后重新生成配置：

```bash
# 解压新版 OpenCore 发行包后
python3 tools/build_config.py --sample /path/to/OpenCore-*/Docs/Sample.plist
# 校验（Linux/macOS 均可，官方发行包自带 Linux 版）
/path/to/OpenCore-*/Utilities/ocvalidate/ocvalidate.linux OC/config.plist
```

`build_config.py` 会以新版 `Sample.plist` 为骨架自动补齐新增键、丢弃已删除键，并保留本仓库的硬件适配项（8 核、Navi 10、ReBAR、boot-args）。换上不同核心数的 CPU 时改脚本里的 `CORES = 0x08`（例如 5900X 用 `0x0C`，3950X 用 `0x10`）。

# Acer Nitro 5 AN515 — macOS Tahoe 26 启动案例

个人黑苹果启动与驱动记录。**不是通用教程**；硬件不同请勿照抄。

仓库**不含**完整 EFI、**不含**序列号 / ROM / MLB。

对照来源：2026-10-24 从 USB `1401` 实盘读取的 `EFI/OC/config.plist`（NullMoth 装 NV 驱动后的版本）。

## 硬件

| 项目 | 规格 |
| --- | --- |
| 机型 | Acer Nitro 5 **AN515** |
| CPU | Intel Core **i5-12500H** (Alder Lake) |
| iGPU | Intel Iris Xe (`8086:46A6`) |
| dGPU | **NVIDIA RTX 3060 Laptop** |
| 存储 | 512 GB NVMe：Windows + Data + **macOS APFS ~107 GB** |
| Wi‑Fi | Killer AX1650i / Intel AX201 类 |
| SMBIOS（仅型号） | `MacBookPro16,2` |

## 目标系统

- **macOS Tahoe 26.7.1**（x86_64）
- OpenCore USB 标签 **1401**；后已将 `EFI/OC` 拷入内置 ESP（与 `Microsoft` 并存）

## 当前可用状态（2026-10-24）

| 功能 | 状态 |
| --- | --- |
| 进桌面 | 可以 |
| **NVIDIA** | [NullMoth](https://github.com/nullmoth/nvidia-macos-driver) 已装；`com.nullmoth.NVRM*` / `NVAccel` 已加载；**外接屏**进桌面；**内屏**仍 verbose / 无加速（Optimus） |
| Wi‑Fi | 可用（`AirportItlwm` Enabled） |
| 内置 EFI | `OC` 已拷入；BIOS 启动项需自测 |
| iGPU 加速 | **未完成**（NootedGreen 曾挂在 `encodeFailureStack`，已回滚） |
| AppleALC / VoodooI2C / VoodooPS2 | 当前 config 均为 **Disabled** |
| Apple ID | 未登录 |

## 实盘 boot-args（脱敏）

```
-v debug=0x100 keepsyms=1 -vi2c-force-polling ipc_control_port_options=0 npci=0x2000 nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth amfi_get_out_of_my_way=0x1 amfi=0x80
```

- `csr-active-config`：`430a0000`（NullMoth 安装后）
- `SecureBootModel`：`Disabled`；`ScanPolicy`：`0`

NV / AMFI 相关参数由 **1401 Mac app** 写入，不是手搓核显参数。

## 实盘 Kernel → Add（Enabled）

| Enabled | Bundle |
| --- | --- |
| true | Lilu.kext |
| true | AMFIPass.kext（MinKernel 20.0.0） |
| true | VirtualSMC.kext |
| true | AirportItlwm.kext（MinKernel 24.0.0） |

其余常见 kext（AppleALC、WEG、VoodooI2C 整树、VoodooPS2、RestrictEvents、各 SMC 插件、itlwm 等）在当前 plist 为 **false**。U 盘 `Kexts` 目录里仍有 `NootedGreen.kext` 文件，但 **未加入启用列表**。

## DeviceProperties（仅路径）

- `PciRoot(0x0)/Pci(0x1,0x0)/Pci(0x0,0x0)`：`acpi-wake-type`
- `PciRoot(0x0)/Pci(0x1f,0x3)`：`layout-id`
- `PciRoot(0x0)/Pci(0x2,0x0)`：**空**（iGPU 未填加速属性）

## Quirks 摘录（实盘）

**Booter：** AvoidRuntimeDefrag / DevirtualiseMmio / EnableSafeModeSlide / ProvideCustomSlide / RebuildAppleMemoryMap / SyncRuntimePermissions = true；**SetupVirtualMap = false**。

**Kernel：** AppleXcpmCfgLock / DisableIoMapper / PanicNoKextDump / PowerTimeoutKernelPanic / ProvideCurrentCpuInfo = true。

> 本机约定：排障时不要随意改 MAT / SetupVirtualMap / DevirtualiseMmio / MMIO 相关项。上面是读盘结果，不是建议你再改。

## 安装与排障时间线（摘要）

1. USB OpenCore 单变量排障；关各 SMC 插件后越过 VirtualSMC。
2. 关 WEG / AirportItlwm / AppleALC 仍可能 EXITBS → 掉 Windows。
3. 关整棵 **VoodooI2C** → 进入 Recovery；装 **Tahoe 26** 到 ~107 GB APFS。
4. NootedGreen 核显路线：APFS 上编译安装后 EFI 注入 → 卡 `encodeFailureStack` → 恢复 `bak-20261024-020308`。
5. [1401](https://github.com/nullmoth/1401) 装 NullMoth NV 驱动；外接屏桌面 + Wi‑Fi 可用。
6. 将 U 盘 `EFI/OC` 拖入内置 `ESP/EFI/`（保留 `Microsoft`）。

## 显卡两条路线

### A. 独显（外接屏已验证）

- [nullmoth/1401](https://github.com/nullmoth/1401) + [nvidia-macos-driver](https://github.com/nullmoth/nvidia-macos-driver)
- 在 macOS 用 **1401.app**；坏了选 `1401: Remove NVIDIA driver`

### B. 核显（未完成）

- Apple GPUDriversIntel + 修补 NootedGreen（勿只用 `device-id 9A49`）
- WEG 不当加速主路径；失败时保留可启动备份

## 分区示意

```
disk1 (512 GB GPT)
  disk1s1  EFI ESP
  disk1s3  Windows
  disk1s5  Data
  disk1s6  APFS ~107 GB → macOS Tahoe
```

## 参考

- [Dortania OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/)
- [nullmoth/1401](https://github.com/nullmoth/1401)
- [nullmoth/nvidia-macos-driver](https://github.com/nullmoth/nvidia-macos-driver)

## 免责

Hackintosh 可能不稳定。本仓库仅个人实验记录，使用后果自负。

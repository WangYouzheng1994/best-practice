# VMware Workstation 安装与 Windows Server 虚拟机部署教程

> **Author:** 王有政<br>
> **编码:** UTF-8<br>
> **适用宿主:** Windows 10 / Windows 11 x64<br>
> **文档定位:** 本机实验与测试环境搭建教程，不用于生产环境

本文从零开始完成以下目标：在本地 Windows 宿主机上安装 VMware Workstation Pro，创建一台满足 Windows Server 2025 硬件要求的虚拟机，完成 Windows Server 安装、驱动配置和网络配置，并能够从宿主机通过远程桌面稳定访问。

阅读顺序建议按章节进行。首次操作大约需要 1.5 至 2 小时，其中 ISO 下载和系统安装占大部分时间。

## 1. 最终结果与参数规划

完成本文全部步骤后，应得到如下环境。

| 项目 | 规划值 |
| --- | --- |
| 宿主机操作系统 | Windows 10 Pro 22H2（19045）或 Windows 11 |
| 虚拟化平台 | VMware Workstation Pro，Broadcom 官网当前版本 |
| 客户机操作系统 | Windows Server 2025 Standard（Desktop Experience），180 天评估版 |
| 虚拟机名称 | `WS2025-TEST` |
| 虚拟机存放路径 | `D:\VMs\WS2025-TEST` |
| 虚拟 CPU | 4 vCPU（1 插槽 × 4 核心） |
| 虚拟内存 | 8 GB |
| 虚拟磁盘 | 80 GB，NVMe 或 SCSI，单文件 |
| 固件 | UEFI + 安全启动 |
| 虚拟 TPM | 2.0（Windows Server 2025 必需） |
| 网络模式 | NAT（VMnet8） |
| 虚拟机 IP | `192.168.100.10/24` |
| 默认网关 | `192.168.100.2` |
| DNS | `223.5.5.5`、`119.29.29.29` |
| 远程桌面 | 启用，宿主通过 `192.168.100.10:3389` 访问 |

> 表中的 IP 段、内存和磁盘大小均为示例值。正式使用前必须确认与宿主所在局域网不冲突，并按宿主实际硬件调整。

## 2. 宿主机准备

### 2.1 硬件要求

| 资源 | 最低 | 推荐 | 说明 |
| --- | --- | --- | --- |
| CPU | 4 核，支持 VT-x/AMD-V | 8 核及以上 | 需在 BIOS/UEFI 中开启硬件虚拟化 |
| 内存 | 8 GB | 16 GB 及以上 | 宿主保留 4 GB，虚拟机分配 8 GB |
| 磁盘 | 60 GB 空闲 | 120 GB 空闲 SSD | 虚拟磁盘实际占用会随使用增长 |
| 网络 | 可访问互联网 | 有线或稳定 Wi-Fi | 用于下载安装包和系统更新 |

### 2.2 确认 CPU 虚拟化已开启

以管理员身份打开 PowerShell，执行：

```powershell
Get-CimInstance Win32_Processor |
    Select-Object Name, NumberOfCores, NumberOfLogicalProcessors, VirtualizationFirmwareEnabled
```

`VirtualizationFirmwareEnabled` 为 `True` 表示固件虚拟化已开启。也可以打开任务管理器，切换到“性能 → CPU”，确认“虚拟化”显示为“已启用”。

如果显示为 `False`，重启进入 BIOS/UEFI，开启以下选项之一：

- Intel 平台：`Intel Virtualization Technology`、`VT-x`、`Intel VMX`
- AMD 平台：`SVM Mode`、`AMD-V`

部分品牌机还需要同时开启 `VT-d` 或 `IOMMU`。

### 2.3 处理 Hyper-V 冲突

VMware Workstation 17.x 及以上可以在 Hyper-V 存在时运行，但会切换到用户模式监视器（ULM），导致性能下降，并且不支持嵌套虚拟化和部分 3D 功能。实验环境建议关闭 Hyper-V 相关功能，换取完整功能和最佳性能。

先检查当前状态：

```powershell
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All | Select-Object FeatureName, State
Get-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform | Select-Object FeatureName, State
Get-WindowsOptionalFeature -Online -FeatureName HypervisorPlatform | Select-Object FeatureName, State
bcdedit /enum "{current}" | Select-String hypervisorlaunchtype
```

如果 `hypervisorlaunchtype` 为 `Auto`，说明 Hyper-V 虚拟机监控程序正在运行。关闭方式如下。

第一步，关闭 Windows 功能。以管理员身份运行 PowerShell：

```powershell
Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All -NoRestart
Disable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -NoRestart
Disable-WindowsOptionalFeature -Online -FeatureName HypervisorPlatform -NoRestart
```

第二步，关闭引导项中的 hypervisor 启动：

```powershell
bcdedit /set hypervisorlaunchtype off
```

第三步，关闭内核隔离中的内存完整性：

打开“Windows 安全中心 → 设备安全性 → 内核隔离详细信息”，将“内存完整性”设为关闭。

完成后重启宿主机。再次执行 `bcdedit /enum "{current}"`，确认 `hypervisorlaunchtype` 为 `Off`。

> 关闭 Hyper-V 后，WSL2、Docker Desktop 的 WSL2 后端、Windows 沙盒和 Windows 容器将不可用。如果这些功能必须保留，可以跳过本节，接受 VMware 的性能损失；后续如需改回，执行 `bcdedit /set hypervisorlaunchtype auto` 并重新启用对应 Windows 功能。

### 2.4 确认磁盘与系统版本

```powershell
Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion, OsBuildNumber, CsTotalPhysicalMemory
Get-PSDrive C, D | Select-Object Name, @{N='FreeGB';E={[math]::Round($_.Free/1GB,1)}}
```

Windows 10 已于 2025 年 10 月 14 日结束常规支持。宿主机仍可运行 VMware Workstation，但应通过 ESU 或升级到 Windows 11 获得安全更新。本教程在 Windows 10 Pro 19045 上验证操作路径。

## 3. 下载并安装 VMware Workstation Pro

### 3.1 版本与授权说明

Broadcom 于 2024 年 5 月将 VMware Workstation Pro 对个人使用免费开放，同年 11 月进一步宣布对所有用户免费，包括商业使用。但 Broadcom 在此之后多次调整授权与产品条款，实际条款必须以下载页面当次显示的许可说明为准，不能沿用旧资料。

| 使用场景 | 授权处理 |
| --- | --- |
| 个人学习、实验、非商业用途 | 免费，启动时选择 Personal Use |
| 公司设备、商业环境 | 下载前核对 Broadcom 页面当次许可条款；如果要求订阅，则购买并输入许可证密钥 |

版本命名方面，Broadcom 已将 Workstation 改为日历版本（例如 `25H2`），不再沿用 `17.x` 的编号。下载时以 Broadcom 官网当前发布版本为准。本文的操作步骤同时适用于 `17.5` 及以上版本和 `25H2` 及之后的版本。

### 3.2 注册账号并下载

1. 打开浏览器访问 Broadcom 支持门户：`https://support.broadcom.com`。
2. 点击右上角 `Login`，选择 `Register` 注册账号。建议使用公司邮箱或常用邮箱，部分免费邮箱域名可能被拒绝。
3. 登录后进入 `VMware Cloud Foundation Division` 分区，打开 `My Downloads`。
4. 在搜索框输入 `VMware Workstation Pro`，选择对应产品。
5. 选择当前版本，同意许可协议，下载 Windows 版安装包，文件名形如 `VMware-workstation-full-<版本号>.exe`。
6. 同时记录页面提供的 SHA256 校验值，下载完成后校验。

下载完成后校验安装包：

```powershell
Get-FileHash -Algorithm SHA256 "D:\soft\VMware-workstation-full-<版本号>.exe"
```

将输出与官网页面显示的校验值逐字符比对。不一致时重新下载，不要安装。

如果 Broadcom 门户访问困难，也可以使用国内镜像站或软件分发平台获取安装包，但必须自行校验数字签名：

```powershell
Get-AuthenticodeSignature "D:\soft\VMware-workstation-full-<版本号>.exe" |
    Select-Object Status, SignerCertificate
```

`Status` 应为 `Valid`，签名主体应包含 `Broadcom` 或 `VMware`。

### 3.3 安装

1. 右键安装包，选择“以管理员身份运行”。
2. 在安装向导中点击“下一步”，接受许可协议。
3. 安装位置保持默认 `C:\Program Files (x86)\VMware\VMware Workstation`，或改为非系统盘。
4. “用户体验设置”中取消勾选“加入 VMware 客户体验提升计划”。
5. 快捷方式按需勾选，点击“下一步 → 安装”。
6. 安装过程中会安装虚拟网络驱动，屏幕可能短暂闪烁或网络中断，属于正常现象。
7. 安装完成后点击“完成”，按提示重启宿主机。

### 3.4 首次启动与授权选择

1. 启动 VMware Workstation Pro。
2. 如果弹出许可提示，选择 `Personal Use`（个人使用）免费模式；商业环境输入购买的许可证密钥。
3. 进入主界面后，确认左侧“库”面板正常显示。

### 3.5 安装后检查

以管理员身份打开 PowerShell，确认虚拟网卡和服务已注册：

```powershell
ipconfig /all | Select-String -Pattern "VMnet","描述","Description"
Get-Service | Where-Object { $_.Name -like "VMware*" } | Format-Table Name, Status, StartType
```

应看到以下虚拟网卡和服务。

| 网卡 | 默认用途 | 默认网段 |
| --- | --- | --- |
| `VMnet1` | 仅主机（Host-only） | `192.168.x.0/24` |
| `VMnet8` | NAT | `192.168.x.0/24` |

| 服务 | 作用 |
| --- | --- |
| `VMware Authorization Service` | 虚拟机授权，必须运行 |
| `VMware DHCP Service` | 为 NAT 和仅主机网络分配地址 |
| `VMware NAT Service` | NAT 转发，VM 访问外网依赖此服务 |
| `VMware USB Arbitration Service` | USB 设备直通 |

上述服务启动类型应为 `Automatic` 且状态为 `Running`。如果 NAT 服务未运行，虚拟机将无法访问外网。

### 3.6 汉化（待补充）

> 本节预留位置，等待补充 VMware Workstation Pro 的汉化方式。填写时请说明适用版本、汉化包来源与校验方式、安装步骤、回退方法，并注明汉化仅影响界面语言，不影响虚拟机功能和客机系统。

<!-- ===== 汉化内容开始：可直接替换以下占位 ===== -->

#### 3.6.1 适用版本与前提

（待补充）

#### 3.6.2 汉化包获取与校验

（待补充）

#### 3.6.3 汉化操作步骤

（待补充）

#### 3.6.4 验证与回退

（待补充）

<!-- ===== 汉化内容结束 ===== -->

## 4. 选择 Windows Server 版本

### 4.1 版本生命周期

Windows Server 采用 LTSC 模式，每个版本提供 5 年主流支持和 5 年扩展支持。选型首先看生命周期，其次看应用兼容性。

| 版本 | 正式发布 | 主流支持结束 | 扩展支持结束 | 建议 |
| --- | --- | --- | --- | --- |
| Windows Server 2025 | 2024-11 | 2029-11 | 2034-11 | 新环境首选 |
| Windows Server 2022 | 2021-08 | 2026-10 | 2031-10 | 仅用于兼容存量应用 |
| Windows Server 2019 | 2018-11 | 2024-01 | 2029-01 | 仅用于兼容旧应用 |
| Windows Server 2016 | 2016-10 | 2022-01 | 2027-01 | 不建议新部署 |

截至本文编写时，Windows Server 2022 已结束主流支持，进入仅提供安全更新的扩展支持阶段。学习和实验环境应直接选择 Windows Server 2025。

> 生命周期日期以微软官方生命周期页面为准，选型前应再次核对。

### 4.1.1 三种授权版本速查

Windows Server 按授权分为 Essentials、Standard、Datacenter 三档。其中 Essentials 仅存在于 2016 和 2019，微软自 Windows Server 2022 起已取消该版本，新部署只能在 Standard 和 Datacenter 之间选择。

| 对比项 | Essentials | Standard | Datacenter |
| --- | --- | --- | --- |
| 可用版本 | 仅 2016 / 2019 | 2016 / 2019 / 2022 / 2025 | 2016 / 2019 / 2022 / 2025 |
| 适用规模 | 25 用户 / 50 设备以内的小型企业 | 中小规模，单机或少量虚拟化 | 中大规模，私有云与高密度虚拟化 |
| 虚拟化授权 | 不支持作为虚拟化宿主 | 每许可证覆盖 2 个虚拟机 | 不限虚拟机数量 |
| CPU 授权 | 按服务器，不限核心 | 按核心授权，最低 16 核 | 按核心授权，最低 16 核 |
| 内存上限 | 64 GB | 按版本，2025 为 4 TB | 按版本，2025 为 4 TB |
| 存储空间直通（S2D） | 不支持 | 不支持 | 支持 |
| 软件定义网络（SDN） | 不支持 | 不支持 | 支持 |
| 屏蔽虚拟机 | 不支持 | 不支持 | 支持 |
| 存储副本 | 不支持 | 仅单伙伴 | 不限伙伴 |
| 图形界面 | 仅 Desktop Experience | Core / Desktop 可选 | Core / Desktop 可选 |
| 远程访问 | 简化管理控制台（无 RDS） | 支持 RDS（需另购 CAL） | 支持 RDS（需另购 CAL） |
| 许可成本 | 最低 | 中 | 最高 |
| 推荐场景 | 存量小型办公环境 | 单机部署、实验、少量虚机 | 大规模虚拟化、私有云 |

选型原则：

1. 新项目不选 Essentials。该版本已停止在新版本中提供，且不支持虚拟化，功能受限。
2. 实验和单机部署选 Standard，成本低且功能完整。
3. 需要在一台物理机上运行超过 2 个 Windows Server 虚机，或需要 S2D、SDN、屏蔽虚拟机时选 Datacenter。
4. 授权按核心计算，采购前必须核对物理 CPU 核心数、CAL 数量和微软当期的授权条款。

> 上表的能力差异以 Windows Server 2025 为准。2016/2019 的 Essentials 在内存上限和用户数限制上与 2025 的 Standard 不同，实际采购应以微软产品条款和授权指南为准。

### 4.2 Standard 与 Datacenter 的区别

| 能力 | Standard | Datacenter |
| --- | --- | --- |
| 虚拟化授权 | 每个许可证覆盖 2 个虚拟机 | 不限虚拟机数量 |
| 存储空间直通（S2D） | 不支持 | 支持 |
| 软件定义网络（SDN） | 不支持 | 支持 |
| 屏蔽虚拟机 | 不支持 | 支持 |
| 存储副本 | 仅单伙伴 | 不限伙伴 |
| 许可成本 | 低 | 高 |

单机实验场景使用 Standard 即可。Datacenter 面向大规模虚拟化和私有云，本教程不需要。

### 4.3 Server Core 与 Desktop Experience

| 安装选项 | 界面 | 资源占用 | 攻击面 | 适用场景 |
| --- | --- | --- | --- | --- |
| Server Core | 无桌面，仅命令提示符和 PowerShell | 低 | 小 | 生产服务器、远程管理 |
| Desktop Experience | 完整图形桌面 | 高 | 大 | 学习、实验、需要 GUI 工具 |

两者的系统内核完全相同，功能组件可以互相转换。首次学习建议选择 Desktop Experience，便于直观操作；后续切换到 Server Core 可通过 PowerShell 卸载图形组件。

本教程选择：**Windows Server 2025 Standard（Desktop Experience）**。

### 4.4 版本号与硬件要求

Windows Server 2025 的内核版本为 `10.0.26100`。虚拟机必须满足以下要求，否则安装程序会拒绝继续。

| 项目 | 要求 |
| --- | --- |
| CPU | 1.4 GHz 及以上，x64 架构，支持 NX、DEP、CMPXCHG16b、LAHF/SAHF、PrefetchW、二级地址转换 |
| CPU 指令集 | 支持 SSE4.2 和 POPCNT |
| 内存 | Server Core 最低 512 MB，Desktop Experience 最低 2 GB，建议 4 GB 以上 |
| 磁盘 | 最低 32 GB，建议 60 GB 以上 |
| 固件 | UEFI 2.3.1c，支持安全启动 |
| TPM | TPM 2.0 |
| 网络 | 千兆以太网 |
| 显示 | 1024 × 768 及以上 |

其中 TPM 2.0 和安全启动是 Windows Server 2025 相比旧版新增的强制要求，也是虚拟机安装失败最常见的原因。VMware Workstation 通过软件方式模拟 TPM 2.0，宿主机本身没有 TPM 芯片也可以满足要求，但添加虚拟 TPM 时必须加密虚拟机。

## 5. 下载 Windows Server 镜像

### 5.1 下载渠道

| 渠道 | 授权 | 适用场景 |
| --- | --- | --- |
| Microsoft Evaluation Center | 180 天免费评估 | 学习、测试、功能验证 |
| Visual Studio Subscriptions | 订阅内包含正式授权 | 开发测试 |
| 批量许可服务中心（VLSC） | 企业批量授权 | 企业正式部署 |
| Azure 市场 | 按用量计费 | 云端部署 |

本教程使用评估版，免费且功能完整，仅限制使用期限。

### 5.2 从评估中心下载

1. 访问 `https://www.microsoft.com/en-us/evalcenter/download-windows-server-2025`。
2. 选择语言，推荐 `English (United States)` 或 `中文(简体)`。中文界面便于学习，英文界面便于对照官方文档。
3. 选择 `64-bit edition`，下载 ISO 文件，大小约 5.5 GB。
4. 下载页面同时提供 SHA256 校验值，复制保存。

下载地址和页面结构可能随微软站点改版变化，如果链接失效，从 `https://www.microsoft.com/evalcenter` 首页搜索 `Windows Server 2025` 进入。

### 5.3 校验镜像

```powershell
Get-FileHash -Algorithm SHA256 "D:\soft\windows-server-2025.iso"
```

将结果与下载页面公布的 SHA256 比对。不一致说明文件损坏或被篡改，必须重新下载。

也可以在 VMware 中直接使用 ISO，不需要解压或制作启动盘。

### 5.4 评估版的限制与续期

- 评估期为 180 天，从首次启动开始计算。
- 到期后系统每小时自动关机一次，不影响数据，但无法正常使用。
- 查看剩余时间：

```powershell
slmgr /dlv
```

- 评估版不能通过简单命令无限续期。到期后应重装，或购买正式授权并使用 `DISM` 转换为正式版本：

```powershell
DISM /Online /Set-Edition:ServerStandard /ProductKey:XXXXX-XXXXX-XXXXX-XXXXX-XXXXX /AcceptEula
```

版本转换只能从评估版转为对应正式版，Standard 与 Datacenter 之间不能直接互转。

## 6. 创建虚拟机

### 6.1 新建向导

1. 打开 VMware Workstation Pro，选择“文件 → 新建虚拟机”，或按 `Ctrl+N`。
2. 配置类型选择“自定义（高级）”，点击“下一步”。自定义模式可以控制固件、磁盘控制器和网络适配器类型。
3. 硬件兼容性选择当前 Workstation 版本对应的最高版本。如果虚拟机需要在旧版 Workstation 或 ESXi 上打开，选择较低版本。
4. 安装来源选择“稍后安装操作系统”。不要选择“安装程序光盘映像文件”，因为简易安装会跳过版本选择、分区和 Administrator 密码设置，不利于理解安装过程。
5. 客户机操作系统选择 `Microsoft Windows`，版本选择 `Windows Server 2025`。如果列表中没有 2025，选择 `Windows Server 2022`，安装后功能不受影响。
6. 虚拟机名称填写 `WS2025-TEST`，位置填写 `D:\VMs\WS2025-TEST`。路径不要包含中文和空格，避免后续脚本和共享目录出错。
7. 固件类型选择 `UEFI`，勾选“安全引导”。Windows Server 2025 必须使用 UEFI。
8. 处理器数量设置为 `1` 个插槽、`4` 个内核。Windows Server 按物理插槽和核心数授权，单插槽多核心更符合授权模型。
9. 内存设置为 `8192 MB`。宿主机内存为 16 GB 时分配 8 GB，为 8 GB 时分配 4 GB。
10. 网络类型选择 `NAT`，网卡类型保持默认的 `e1000e`。安装完成并可联网后，可在虚拟机设置中改为 `VMXNET3` 以获得更好性能。
11. I/O 控制器类型选择 `LSI Logic SAS`。如果客户机版本支持且追求性能，可以选择 `NVMe`。
12. 磁盘类型选择 `SCSI`，磁盘选择“创建新虚拟磁盘”。
13. 最大磁盘大小填写 `80 GB`，选择“将虚拟磁盘存储为单个文件”。不要勾选“立即分配所有磁盘空间”，使用精简置备节省宿主机空间。
14. 磁盘文件名保持默认，点击“下一步 → 完成”。

### 6.2 添加虚拟 TPM

Windows Server 2025 强制要求 TPM 2.0，必须在安装前添加。

1. 选中虚拟机，点击“编辑虚拟机设置”。
2. 切换到“硬件”选项卡，点击“添加”。
3. 选择“可信平台模块”，点击“下一步 → 完成”。
4. VMware 会提示必须加密虚拟机才能使用 TPM。点击“确定”，设置加密密码。
5. 加密密码一旦丢失无法恢复，虚拟机将永久无法启动。必须记录到密码管理系统。
6. 加密采用 AES-256，会对虚拟机性能产生轻微影响，实验环境可以接受。

如果添加 TPM 时提示“此虚拟机不支持可信平台模块”，检查以下项目：

- 固件类型是否为 UEFI。
- 是否勾选了安全引导。
- 虚拟机硬件版本是否过低，尝试升级硬件兼容性。

### 6.3 挂载安装镜像

1. 打开“编辑虚拟机设置 → 硬件 → CD/DVD (SATA)”。
2. 选择“使用 ISO 映像文件”，点击“浏览”，选择已校验的 Windows Server 2025 ISO。
3. 勾选“启动时连接”，点击“确定”。

### 6.4 确认最终硬件配置

| 硬件 | 配置 |
| --- | --- |
| 内存 | 8192 MB |
| 处理器 | 1 插槽 × 4 内核 |
| 硬盘 | 80 GB，NVMe 或 SCSI，单文件 |
| CD/DVD | 已挂载 Windows Server 2025 ISO，启动时连接 |
| 网络适配器 | NAT，e1000e |
| 声卡 | 可移除 |
| 打印机 | 可移除 |
| USB 控制器 | USB 3.1 |
| 可信平台模块 | 已添加 |
| 固件 | UEFI，已启用安全引导 |
| 显示 | 自动，显存 1 GB |

声卡和打印机对服务器无用，移除可以减少资源占用和启动时的设备枚举。

## 7. 安装 Windows Server

### 7.1 启动安装程序

1. 点击“开启此虚拟机”。
2. 虚拟机窗口出现 `Press any key to boot from CD or DVD` 时，立即点击窗口内部并按任意键。错过提示会进入 UEFI Shell，重启虚拟机即可。
3. 鼠标被虚拟机捕获后，按 `Ctrl+Alt` 释放回宿主机。
4. 选择语言、时间和货币格式、键盘输入法，点击“下一步 → 现在安装”。

### 7.2 选择安装版本

安装程序列出四个选项：

| 选项 | 说明 |
| --- | --- |
| Windows Server 2025 Standard Evaluation | 标准版，带桌面体验 |
| Windows Server 2025 Standard Evaluation (Server Core) | 标准版，无桌面 |
| Windows Server 2025 Datacenter Evaluation | 数据中心版，带桌面体验 |
| Windows Server 2025 Datacenter Evaluation (Server Core) | 数据中心版，无桌面 |

选择 **Windows Server 2025 Standard Evaluation**（不带 Server Core 字样），点击“下一步”，接受许可条款。

### 7.3 磁盘分区

1. 安装类型选择“自定义：仅安装 Microsoft Windows（高级）”。
2. 列表中显示一块 80 GB 的未分配空间，选中后点击“新建 → 应用”。
3. 安装程序提示需要创建额外的分区，点击“确定”。
4. 自动生成 EFI 系统分区、Microsoft 保留分区和主分区。确认主分区大小约为 79 GB。
5. 选中主分区，点击“下一步”，开始复制文件。

如果安装程序提示“无法将 Windows 安装到此磁盘”，检查：

- 磁盘是否为 GPT 分区表。UEFI 固件要求 GPT，安装程序在空磁盘上会自动初始化。
- 磁盘控制器驱动是否被识别。LSI Logic SAS 有内置驱动，VMware Paravirtual 需要额外加载驱动。

### 7.4 设置 Administrator 密码

安装完成后系统自动重启，进入密码设置界面。

Windows Server 强制要求 Administrator 密码满足复杂度：至少 8 位，包含大写字母、小写字母、数字和符号中的至少三类。

示例密码仅用于演示，实际使用必须替换：

```text
P@ssw0rd!2026
```

密码丢失无法找回，只能重装系统。虚拟机环境中建议记录到密码管理器，并在完成配置后创建快照。

### 7.5 首次登录

1. 密码设置完成后进入登录界面。
2. 在 VMware 中按 `Ctrl+Alt+Insert` 发送 `Ctrl+Alt+Delete`。
3. 输入 Administrator 密码登录。
4. 首次登录会自动打开“服务器管理器”，并可能弹出“本地服务器”配置页面。
5. 如果提示网络位置，选择“是”以允许网络发现，实验环境可接受；正式环境保持“否”。

## 8. 安装 VMware Tools

VMware Tools 提供显卡驱动、鼠标无缝切换、复制粘贴、拖放、时间同步和虚拟机心跳，是虚拟机可用性的基础。

### 8.1 安装

1. 在 VMware 菜单选择“虚拟机 → 安装 VMware Tools”。
2. 进入 Windows Server，打开“此电脑”，双击 `VMware Tools` 光驱。
3. 运行 `setup64.exe`，选择“典型安装”，点击“下一步 → 安装”。
4. 安装完成后重启虚拟机。

### 8.2 静默安装

如果需要在多台虚拟机上批量部署，可在客户机 PowerShell 中执行：

```powershell
$drive = (Get-Volume | Where-Object { $_.FileSystemLabel -eq 'VMware Tools' }).DriveLetter
& "${drive}:\setup64.exe" /S /v"/qn REBOOT=R"
```

### 8.3 安装后验证

重启后检查以下能力：

- 虚拟机窗口大小改变时，客户机分辨率自动适配。
- 宿主机与虚拟机之间可以复制粘贴文本。
- 鼠标移出虚拟机窗口时不再需要按 `Ctrl+Alt`。
- “设备管理器”中显示 `VMware SVGA II Adapter`、`VMware VMXNET3 Ethernet Adapter`（切换网卡后）等设备。

如果复制粘贴不生效，检查“虚拟机 → 设置 → 选项 → 客户机隔离”，确认“启用复制粘贴”和“启用拖放”已勾选。

### 8.4 常用快捷键

| 快捷键 | 作用 |
| --- | --- |
| `Ctrl+G` | 将输入焦点切换到虚拟机 |
| `Ctrl+Alt` | 释放鼠标和键盘回宿主机 |
| `Ctrl+Alt+Insert` | 向虚拟机发送 `Ctrl+Alt+Delete` |
| `Ctrl+Alt+Enter` | 切换全屏 |
| `Ctrl+Alt+F` | 切换全屏（部分版本） |

## 9. 虚拟机网络配置

### 9.1 四种网络模式对比

| 模式 | VMware 设备 | 虚拟机获取的地址 | 能否访问宿主机 | 能否访问外网 | 局域网能否访问虚拟机 | 适用场景 |
| --- | --- | --- | --- | --- | --- | --- |
| 桥接 | VMnet0 | 与宿主同网段的局域网地址 | 是 | 是 | 是 | 需要作为局域网独立主机 |
| NAT | VMnet8 | `192.168.x.0/24` 私有地址 | 是 | 是（经宿主转发） | 需端口映射 | 默认推荐，隔离性好 |
| 仅主机 | VMnet1 | `192.168.x.0/24` 私有地址 | 是 | 否 | 否 | 纯隔离实验 |
| 自定义 | VMnet2 等 | 按自定义网段 | 视配置 | 视配置 | 视配置 | 多网段实验 |

实验环境推荐 **NAT**：虚拟机拥有独立网段，不占用局域网 IP，不暴露到局域网，同时可以正常上网和安装更新。需要局域网其他设备直接访问虚拟机时，再改用桥接或配置端口转发。

### 9.2 规划 NAT 网段

先查看宿主机现有网段，避免冲突：

```powershell
Get-NetIPAddress -AddressFamily IPv4 |
    Where-Object { $_.IPAddress -notlike "127.*" } |
    Select-Object InterfaceAlias, IPAddress, PrefixLength
```

如果宿主机局域网使用 `192.168.1.0/24`，NAT 网段不要再选 `192.168.1.0/24`。本文使用 `192.168.100.0/24`。

| 项目 | 值 |
| --- | --- |
| 子网 | `192.168.100.0` |
| 子网掩码 | `255.255.255.0` |
| 宿主侧 VMnet8 地址 | `192.168.100.1` |
| NAT 网关 | `192.168.100.2` |
| DHCP 范围 | `192.168.100.128` 至 `192.168.100.254` |
| 虚拟机静态地址 | `192.168.100.10` |

静态地址放在 DHCP 范围之外，避免地址冲突。

### 9.3 配置虚拟网络编辑器

1. 在 VMware 菜单选择“编辑 → 虚拟网络编辑器”。
2. 点击“更改设置”按钮，通过 UAC 提权。不提权无法修改网络配置。
3. 选中 `VMnet8`，确认“NAT 模式”已选中。
4. 将“子网 IP”设为 `192.168.100.0`，“子网掩码”设为 `255.255.255.0`。
5. 点击“NAT 设置”，确认网关为 `192.168.100.2`。如需为虚拟机指定固定 DNS，可在此页面配置，但更推荐在 Windows Server 内部配置。
6. 点击“DHCP 设置”，确认起始地址为 `192.168.100.128`，结束地址为 `192.168.100.254`。如果所有虚拟机都使用静态地址，可以取消勾选“将 DHCP 服务用于该网络”，减少一个服务依赖。
7. 点击“确定”保存。

### 9.4 设置虚拟机网卡

1. 选中虚拟机，打开“编辑虚拟机设置 → 硬件 → 网络适配器”。
2. 选择“自定义”，下拉框选择 `VMnet8`；或直接选择“NAT”模式，两者等价。
3. 勾选“启动时连接”。
4. 如果宿主机使用笔记本并在有线和无线网络间切换，勾选“复制物理网络连接状态”，可以避免网络状态不同步。
5. 点击“确定”。

### 9.5 在 Windows Server 内配置静态 IP

#### 9.5.1 图形界面

1. 打开“服务器管理器 → 本地服务器 → 以太网”，或按 `Win+R` 输入 `ncpa.cpl`。
2. 右键网卡，选择“属性”。
3. 双击“Internet 协议版本 4 (TCP/IPv4)”。
4. 选择“使用下面的 IP 地址”，填写：

| 项目 | 值 |
| --- | --- |
| IP 地址 | `192.168.100.10` |
| 子网掩码 | `255.255.255.0` |
| 默认网关 | `192.168.100.2` |
| 首选 DNS | `223.5.5.5` |
| 备用 DNS | `119.29.29.29` |

5. 点击“确定”保存。

#### 9.5.2 PowerShell

以管理员身份打开 PowerShell：

```powershell
$if = Get-NetAdapter | Where-Object { $_.Status -eq 'Up' -and $_.Virtual -eq $false }
$if | Format-Table Name, InterfaceDescription, Status

New-NetIPAddress -InterfaceAlias $if.Name -IPAddress 192.168.100.10 -PrefixLength 24 -DefaultGateway 192.168.100.2
Set-DnsClientServerAddress -InterfaceAlias $if.Name -ServerAddresses 223.5.5.5,119.29.29.29
```

如果网卡已有 DHCP 地址，先移除：

```powershell
Remove-NetIPAddress -InterfaceAlias $if.Name -Confirm:$false
Remove-NetRoute -InterfaceAlias $if.Name -Confirm:$false
```

验证配置：

```powershell
Get-NetIPConfiguration -InterfaceAlias $if.Name
Get-DnsClientServerAddress -InterfaceAlias $if.Name
```

#### 9.5.3 设置计算机名

计算机名应使用大写字母、数字和短横线，长度不超过 15 个字符：

```powershell
Rename-Computer -NewName "WS2025-TEST" -Restart
```

重启后生效。

#### 9.5.4 设置时区

```powershell
Set-TimeZone -Id "China Standard Time"
Get-TimeZone
```

#### 9.5.5 启用 ICMP 回显

Windows Server 默认阻止 ping。实验环境需要宿主测试连通性时，启用入站 ICMP：

```powershell
New-NetFirewallRule -DisplayName "Allow ICMPv4-In" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow
```

生产环境不建议开放 ICMP 给任意来源，应限定远程地址范围。

### 9.6 从宿主机访问虚拟机

#### 9.6.1 直接访问

NAT 模式下，宿主机上存在一块 `VMnet8` 虚拟网卡，地址为 `192.168.100.1`。宿主机可以直接访问虚拟机的 NAT 地址，不需要任何端口映射：

```powershell
ping 192.168.100.10
Test-NetConnection 192.168.100.10 -Port 3389
```

#### 9.6.2 启用远程桌面

在 Windows Server 中执行：

```powershell
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
Get-NetFirewallRule -DisplayGroup "Remote Desktop" | Select-Object DisplayName, Enabled
```

随后在宿主机执行：

```powershell
mstsc /v:192.168.100.10
```

使用 `Administrator` 账号登录。首次连接会提示证书不受信任，实验环境选择“是”继续。

#### 9.6.3 局域网其他设备访问（端口转发）

如果局域网中的其他电脑需要访问虚拟机，而虚拟机使用 NAT，需要在 VMware 中配置端口转发。

1. 打开“编辑 → 虚拟网络编辑器”，提权后选中 `VMnet8`。
2. 点击“NAT 设置 → 端口转发 → 添加”。
3. 填写映射：

| 项目 | 值 |
| --- | --- |
| 主机端口 | `33890` |
| 类型 | `TCP` |
| 虚拟机 IP 地址 | `192.168.100.10` |
| 虚拟机端口 | `3389` |
| 描述 | `WS2025 RDP` |

4. 保存后，局域网设备通过 `宿主局域网IP:33890` 连接虚拟机。

宿主机查看自身局域网地址：

```powershell
Get-NetIPAddress -AddressFamily IPv4 |
    Where-Object { $_.InterfaceAlias -notlike "VMnet*" -and $_.IPAddress -notlike "127.*" } |
    Select-Object InterfaceAlias, IPAddress
```

如果宿主机防火墙阻止入站连接，需要放行对应端口：

```powershell
New-NetFirewallRule -DisplayName "VMware NAT RDP 33890" -Direction Inbound -Protocol TCP -LocalPort 33890 -Action Allow
```

### 9.7 使用桥接模式

需要虚拟机像局域网独立主机一样被访问时，改用桥接。

1. 打开“编辑 → 虚拟网络编辑器”，提权后选择 `VMnet0`。
2. “桥接到”选择宿主实际联网的物理网卡。多网卡、VPN 或虚拟网卡环境下不要使用“自动”，否则可能桥接到错误的网卡。
3. 虚拟机设置中将网络适配器改为“桥接（自动）”或“自定义 VMnet0”。
4. 在 Windows Server 中改为与局域网同网段的地址，或保留 DHCP 自动获取。

桥接模式下虚拟机直接暴露在局域网中，必须设置强密码、启用防火墙并及时安装补丁。

### 9.8 连通性验证清单

按顺序执行，任一步失败先解决再继续。

| 步骤 | 位置 | 命令 | 预期结果 |
| --- | --- | --- | --- |
| 1 | 虚拟机 | `ipconfig` | 显示 `192.168.100.10`，网关 `192.168.100.2` |
| 2 | 虚拟机 | `ping 192.168.100.2` | 有回复 |
| 3 | 虚拟机 | `ping 223.5.5.5` | 有回复，说明 NAT 出网正常 |
| 4 | 虚拟机 | `nslookup www.microsoft.com` | 返回解析结果，说明 DNS 正常 |
| 5 | 宿主机 | `ping 192.168.100.10` | 有回复 |
| 6 | 宿主机 | `Test-NetConnection 192.168.100.10 -Port 3389` | `TcpTestSucceeded : True` |
| 7 | 宿主机 | `mstsc /v:192.168.100.10` | 出现登录界面 |

如果第 3 步失败但第 2 步成功，检查 VMware NAT Service 是否运行，以及宿主机是否能正常上网。

如果第 5 步失败但第 1、2 步成功，检查虚拟机 Windows 防火墙是否放行了 ICMP，以及宿主机 `VMnet8` 网卡是否被禁用。

## 10. 系统初始化与激活

### 10.1 安装更新

```powershell
Install-Module PSWindowsUpdate -Force
Get-WindowsUpdate
Install-WindowsUpdate -AcceptAll -AutoReboot
```

如果无法访问 PowerShell Gallery，可以使用“设置 → 更新和安全 → Windows 更新”手动检查。

### 10.2 查看激活与评估状态

```powershell
slmgr /dlv
slmgr /xpr
```

评估版显示剩余天数。正式授权版本显示激活到期时间。

### 10.3 关闭 IE 增强的安全配置

Windows Server 默认开启 IE 增强的安全配置（IE ESC），会阻止大部分网站访问。仅实验环境建议关闭：

1. 打开“服务器管理器 → 本地服务器”。
2. 点击“IE 增强的安全配置”。
3. 管理员和用户两组都选择“关闭”，点击“确定”。

Windows Server 2025 已移除 Internet Explorer，默认浏览器为 Microsoft Edge，IE 增强的安全配置在新版本中的行为也已调整。若控制台仍显示该入口，可按上述步骤关闭；若不存在则跳过本节，直接检查 Edge 的安全策略。

### 10.4 常用管理入口

| 功能 | 命令或位置 |
| --- | --- |
| 服务器管理器 | `servermanager` |
| 网络连接 | `ncpa.cpl` |
| 系统属性 | `sysdm.cpl` |
| 服务 | `services.msc` |
| 事件查看器 | `eventvwr.msc` |
| 组策略 | `gpedit.msc` |
| 注册表 | `regedit` |
| 防火墙 | `wf.msc` |
| 任务管理器 | `taskmgr` |

Server Core 版本使用 `sconfig` 完成网络、计算机名、远程管理和更新配置：

```cmd
sconfig
```

## 11. 快照与日常维护

### 11.1 创建基线快照

系统安装完成并验证网络后，立即创建快照，作为后续实验的回滚点。

1. 关闭虚拟机，或在运行状态下操作（建议关机后创建，快照体积更小、一致性更好）。
2. 选择“虚拟机 → 快照 → 拍摄快照”。
3. 名称填写 `baseline-clean-install`，描述填写“Windows Server 2025 初始安装完成，网络已配置”。
4. 点击“拍摄快照”。

后续每次重大变更前创建快照，命名包含日期和变更内容，例如 `2026-10-21-before-iis`。

### 11.2 快照的注意事项

- 快照不是备份。快照文件保存在虚拟机目录中，磁盘损坏时会一起丢失。
- 快照链越长，磁盘占用越大，性能越差。完成验证后应删除中间快照，只保留基线。
- 删除快照时 VMware 需要合并磁盘文件，耗时较长，期间不要强制关闭宿主机。
- 虚拟机加密后，快照同样受加密保护，密码丢失则无法恢复。

### 11.3 回收磁盘空间

虚拟机删除文件后，虚拟磁盘文件不会自动缩小。回收步骤：

1. 在 Windows Server 中清理磁盘，执行磁盘碎片整理（仅机械盘）或 `Optimize-Volume`。
2. 关闭虚拟机。
3. 在 VMware 中选择“虚拟机 → 设置 → 硬盘 → 实用工具 → 压缩”。
4. 或者在宿主机执行：

```powershell
& "C:\Program Files (x86)\VMware\VMware Workstation\vmware-vdiskmanager.exe" -k "D:\VMs\WS2025-TEST\WS2025-TEST.vmdk"
```

### 11.4 克隆虚拟机

需要多台相同配置的服务器时，使用克隆而不是复制目录。

1. 右键虚拟机，选择“管理 → 克隆”。
2. 选择“完整克隆”得到独立虚拟机，或“链接克隆”节省空间但依赖父虚拟机。
3. 克隆后必须修改计算机名、IP 地址和 SID，避免冲突：

```powershell
Rename-Computer -NewName "WS2025-NODE2" -Restart
```

Windows Server 使用 `sysprep` 可以重置 SID：

```cmd
C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown
```

### 11.5 虚拟机文件说明

| 文件 | 作用 |
| --- | --- |
| `.vmx` | 虚拟机配置文件，记录硬件参数 |
| `.vmdk` | 虚拟磁盘文件 |
| `.nvram` | UEFI 变量存储 |
| `.vmsd` | 快照元数据 |
| `.vmsn` | 快照状态文件 |
| `.vmem` | 挂起时的内存镜像 |
| `.log` | 虚拟机运行日志，排错时查看 |

迁移虚拟机时，应关闭虚拟机后复制整个目录，不要只复制 `.vmdk`。

## 12. 常见问题

### 12.1 VMware 提示“此主机支持 Intel VT-x，但 Intel VT-x 处于禁用状态”

原因：BIOS/UEFI 未开启硬件虚拟化，或 Hyper-V 占用了 VT-x。

处理：

1. 进入 BIOS/UEFI 开启 `VT-x` 或 `SVM`。
2. 按第 2.3 节关闭 Hyper-V 和内存完整性。
3. 执行 `bcdedit /set hypervisorlaunchtype off` 后重启。

### 12.2 提示“VMware Workstation 与 Hyper-V 不兼容”

原因：宿主机启用了 Hyper-V、虚拟机平台、WSL2 或内存完整性。

处理：同 12.1。如果必须保留 Hyper-V，VMware 17.5 及以上会自动使用 ULM 模式，但虚拟机性能会下降，且不支持嵌套虚拟化。

### 12.3 Windows Server 安装时提示需要 TPM 2.0 或安全启动

原因：虚拟机未添加 TPM，或固件仍为 BIOS。

处理：

1. 关机后打开虚拟机设置，确认“选项 → 高级 → 固件类型”为 UEFI，并勾选安全引导。
2. 在“硬件”中添加“可信平台模块”，按提示加密虚拟机。
3. 重新启动安装。

仅实验环境需要绕过检查时，在安装界面按 `Shift+F10` 打开命令提示符，执行 `regedit`，创建以下键值后返回继续安装：

```text
HKEY_LOCAL_MACHINE\SYSTEM\Setup\LabConfig
BypassTPMCheck      DWORD = 1
BypassSecureBootCheck DWORD = 1
```

绕过检查只适用于学习，不能用于任何正式环境。

### 12.4 添加 TPM 时提示必须加密虚拟机

这是 VMware 的设计要求：虚拟 TPM 的状态文件必须加密存储。按提示设置加密密码即可。密码丢失无法恢复，必须妥善保存。

### 12.5 虚拟机无法访问外网

按顺序检查：

1. 虚拟机网卡是否勾选“启动时连接”。
2. 虚拟机网络模式是否为 NAT，是否连接到 `VMnet8`。
3. 宿主机 `VMware NAT Service` 是否运行：

```powershell
Get-Service "VMware NAT Service" | Start-Service
```

4. 虚拟机 IP、网关是否与 VMnet8 子网一致。
5. 宿主机是否能正常上网。
6. 虚拟机 DNS 是否配置正确，尝试 `nslookup www.microsoft.com 223.5.5.5`。

### 12.6 宿主机 ping 不通虚拟机

1. 确认虚拟机防火墙允许 ICMP，参考 9.5.5。
2. 确认宿主机 `VMnet8` 网卡未被禁用：

```powershell
Get-NetAdapter | Where-Object { $_.Name -like "VMnet*" } | Format-Table Name, Status
Enable-NetAdapter -Name "VMnet8"
```

3. 确认虚拟机和宿主机在同一 NAT 子网。

### 12.7 安装 VMware Tools 后分辨率仍然很小

1. 确认 VMware Tools 服务正在运行：

```powershell
Get-Service VMTools
```

2. 在虚拟机中打开“显示设置”，手动选择推荐分辨率。
3. 重新安装 VMware Tools，选择“修改”并确保勾选显卡驱动。

### 12.8 复制粘贴和拖放不生效

1. 确认 VMware Tools 已安装并运行。
2. 检查“虚拟机 → 设置 → 选项 → 客户机隔离”，勾选“启用复制粘贴”和“启用拖放”。
3. 重启虚拟机。
4. Windows Server 的 UAC 提权窗口内无法复制粘贴，这是系统限制。

### 12.9 虚拟机时间与宿主机不一致

VMware Tools 默认开启时间同步。如果仍不一致：

```powershell
Get-Service W32Time | Set-Service -StartupType Automatic
Start-Service W32Time
w32tm /resync
```

确保虚拟机设置中“选项 → VMware Tools → 同步客户机时间与主机时间”已勾选。

### 12.10 磁盘空间不足

1. 在虚拟机内清理临时文件和旧更新：

```powershell
cleanmgr /sageset:1
cleanmgr /sagerun:1
```

2. 使用 `DISM` 清理组件存储：

```powershell
DISM /Online /Cleanup-Image /StartComponentCleanup /ResetBase
```

3. 关闭虚拟机后压缩虚拟磁盘，参考 11.3。

### 12.11 虚拟机启动提示“此虚拟机已加密”

输入创建 TPM 时设置的加密密码。密码连续输错不会锁死，但无法找回。如果没有记录密码，只能删除虚拟机重建。

### 12.12 评估版到期后频繁关机

评估版到期后每小时关机一次。处理方式：

- 重装系统并重新开始 180 天评估。
- 购买正式授权后使用 `DISM` 转换版本。
- 在实验环境中从基线快照恢复，但快照不会重置评估计时。

## 13. 附录：初始化脚本

以下脚本用于在 Windows Server 首次登录后快速完成基础配置。执行前按实际环境修改变量。

```powershell
# 以管理员身份运行

# 1. 基础变量
$ComputerName = "WS2025-TEST"
$IPAddress    = "192.168.100.10"
$PrefixLength = 24
$Gateway      = "192.168.100.2"
$DnsServers   = @("223.5.5.5", "119.29.29.29")
$TimeZone     = "China Standard Time"

# 2. 重命名计算机
Rename-Computer -NewName $ComputerName -Force

# 3. 配置静态 IP
$adapter = Get-NetAdapter | Where-Object { $_.Status -eq "Up" -and $_.Virtual -eq $false } | Select-Object -First 1
if (-not $adapter) { throw "未找到可用物理网卡，请检查 VMware 网络适配器是否已连接" }

Remove-NetIPAddress -InterfaceAlias $adapter.Name -Confirm:$false -ErrorAction SilentlyContinue
Remove-NetRoute     -InterfaceAlias $adapter.Name -Confirm:$false -ErrorAction SilentlyContinue

New-NetIPAddress -InterfaceAlias $adapter.Name -IPAddress $IPAddress -PrefixLength $PrefixLength -DefaultGateway $Gateway
Set-DnsClientServerAddress -InterfaceAlias $adapter.Name -ServerAddresses $DnsServers

# 4. 时区
Set-TimeZone -Id $TimeZone

# 5. 允许 ICMP
New-NetFirewallRule -DisplayName "Allow ICMPv4-In" -Protocol ICMPv4 -IcmpType 8 -Direction Inbound -Action Allow -ErrorAction SilentlyContinue

# 6. 启用远程桌面
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

# 7. 关闭 IE 增强的安全配置
$adminKey = "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A7-37EF-4b3f-8CFC-4F3A74704073}"
$userKey  = "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A8-37EF-4b3f-8CFC-4F3A74704073}"
Set-ItemProperty -Path $adminKey -Name "IsInstalled" -Value 0 -ErrorAction SilentlyContinue
Set-ItemProperty -Path $userKey  -Name "IsInstalled" -Value 0 -ErrorAction SilentlyContinue

# 8. 输出结果
Get-NetIPConfiguration -InterfaceAlias $adapter.Name
Write-Host "配置完成，请重启计算机使计算机名生效。" -ForegroundColor Green
```

重启后验证：

```powershell
Restart-Computer
```

```powershell
hostname
Get-NetIPConfiguration
Test-NetConnection 223.5.5.5 -Port 53
```

## 14. 操作检查表

| 序号 | 检查项 | 完成标准 |
| --- | --- | --- |
| 1 | 宿主 CPU 虚拟化 | 任务管理器显示“虚拟化：已启用” |
| 2 | Hyper-V 冲突 | `hypervisorlaunchtype` 为 `Off` |
| 3 | VMware 安装 | 四个 VMware 服务均为 Running |
| 4 | 虚拟网络 | `VMnet1`、`VMnet8` 正常存在 |
| 5 | ISO 校验 | SHA256 与官网一致 |
| 6 | 虚拟机固件 | UEFI + 安全引导 |
| 7 | 虚拟 TPM | 已添加，加密密码已记录 |
| 8 | 系统安装 | 进入桌面，服务器管理器正常 |
| 9 | VMware Tools | 分辨率自适应、复制粘贴可用 |
| 10 | 静态 IP | `192.168.100.10/24`，网关 `192.168.100.2` |
| 11 | 出网 | 虚拟机可 ping 通 `223.5.5.5` 并解析域名 |
| 12 | 远程桌面 | 宿主机 `mstsc /v:192.168.100.10` 可登录 |
| 13 | 基线快照 | 已创建 `baseline-clean-install` |

## 15. 参考链接

- Broadcom 支持门户：`https://support.broadcom.com`
- VMware Workstation Pro 文档：`https://techdocs.broadcom.com/us/en/vmware-cis/desktop-hypervisors/workstation-pro.html`
- Microsoft Evaluation Center：`https://www.microsoft.com/evalcenter`
- Windows Server 2025 评估版下载：`https://www.microsoft.com/en-us/evalcenter/download-windows-server-2025`
- Windows Server 生命周期：`https://learn.microsoft.com/lifecycle/products/windows-server-2025`
- Windows Server 硬件要求：`https://learn.microsoft.com/windows-server/get-started/hardware-requirements`

> 上述站点可能改版，链接失效时请从对应厂商首页搜索产品名称进入。下载前必须核对当前版本号和校验值。




---
### windows 2022
Windows Server 2022 Datacenter（WX4NM-KYWYW-QJJR4-XV3QB-6VM33）
Windows Server 2022 Standard（VDYBN-27WPP-V4HQT-9VMD4-VMK7H）
---
### windows 2019
windows server 2019 + SN
http://www.winwin7.com/soft/54938.html  
windows server 2019 下载地址
https://www.microsoft.com/en-us/evalcenter/download-windows-server-2019


Windows Server 2019 Datacenter
［Key］：WMDGN-G9PQG-XVVXX-R3X43-63DFG  
Windows Server 2019 Standard
［Key］：N69G4-B89J2-4G8F4-WWYCC-J464C  
Windows Server 2019 Essential
［Key］：WVDHN-86M7X-466 P 6-VHXV7-YY726


### windows 2016
Windows Server 2016 Datacenter 数据中心版安装密钥：CB7KF-BWN84-R7R2Y-793K2-8XDDG
Windows Server 2016 Standard标准版安装密钥：WC2BQ-8NRM3-FDDYY-2BFGV-KHKQY
Windows Server 2016 嵌入式版安装密钥：JCKRF-N37P4-C2D82-9YXRT-4M63B
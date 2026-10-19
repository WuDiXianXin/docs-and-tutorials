# KVM 安装 Win10

> [!IMPORTANT]
> 核心原则：**能用 VirtIO 就用 VirtIO**。
> 磁盘和网络使用 VirtIO 驱动，CPU 使用 `host-passthrough` 模式，Windows 10 虚拟机即可获得接近原生的性能体验。

## 目录

- [前置条件](#前置条件)
- [创建虚拟机](#创建虚拟机)
- [安装过程中的关键操作-加载VirtIO存储驱动](#安装过程中的关键操作-加载VirtIO存储驱动)
- [安装后的驱动与工具安装](#安装后的驱动与工具安装)
- [性能微调与进阶优化](#性能微调与进阶优化)
- [常见问题速查](#常见问题速查)

## 前置条件

- 准备 Win10_22H2_*.iso 镜像，如 /kvm/iso/Win10_22H2_Chinese_Simplified_x64v1.iso。
- 下载 [Fedora VirtIO-Win 驱动 ISO](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/) 并放在 /kvm/iso 目录中。

## 创建虚拟机

打开 virt-manager，点击“创建新虚拟机”，选择“本地安装介质（ISO 镜像）”，指定 Windows 10 ISO。
**在向导最后一页，务必勾选“在安装前自定义配置”**，然后进入硬件详情页逐项调整。

### 操作系统类型与版本

- **类型**：`Windows`
- **版本**：`Microsoft Windows 10`

### 芯片组与固件

- **芯片组（Chipset）** ：选择 **Q35**。Q35 是更现代的虚拟芯片组，对 UEFI、PCIe 直通等特性支持更好。
- **固件**：选择 **UEFI x86_64**，推荐使用带 SMM / Secure Boot 支持的 OVMF 变体，例如 `OVMF_CODE.secboot.4m.fd`。
  路径通常位于 `/usr/share/edk2/x64/` 或 `/usr/share/OVMF/` 下。

> [!TIP]
> Windows 10 本身不强制要求 Secure Boot。
> 如果后续没有启用 BitLocker 或安全启动的需求，可以选择不带 secboot 的普通 OVMF 固件以简化配置；
> 如果选择 secboot 固件但未正确配置 Microsoft 密钥，可能影响启动。

### CPU配置

- **CPU 模式**：**`host-passthrough`**。
- **CPU 拓扑**：建议勾选“复制主机 CPU 配置”（Copy host CPU configuration），virt-manager 会自动将模式设为 `host-passthrough` 并匹配拓扑。

`host-passthrough` 让虚拟机看到的 CPU 与宿主机物理 CPU 完全一致，包括指令集和硬件特性，是性能最佳的选择。代价是该虚拟机**无法在线迁移到 CPU 不同的宿主机**。

安装后可用以下命令验证 CPU 模式是否生效：

```bash
virsh dumpxml <虚拟机名> | grep "cpu mode"
# 预期输出类似：<cpu mode='host-passthrough' check='none'/>
```

### 内存与CPU核心

| 资源 | 最低建议 | 推荐 |
|---|---|---|
| 内存 | 4 GB | 8 GB 或更多 |
| vCPU | 2 核 | 4 核或更多 |

### 磁盘

- **总线（Bus）** ：**VirtIO**。这是性能最好的磁盘总线，远优于 IDE 或 SATA。
- **格式**：**qcow2**。支持动态分配和快照。
- **容量**：建议 **60 GB 以上**（Windows 10 系统盘会随更新持续增长，50 GB 是下限，60 GB 以上更从容）。

> [!TIP]
> 1. Windows 10 安装镜像**不包含 VirtIO 存储驱动**。在安装过程中必须手动加载驱动，否则磁盘列表将为空白。这是正常现象，不是配置错误。
> 
> 2. 后续可以扩张这个 qcow2 存储大小。
>   ```bash
>   qemu-img resize /path/to/disk.qcow2 +20G   # 关机后
>   # 再进 Windows 磁盘管理 → 扩展卷
>   ```

### 网络

- **网络设备模型（NIC Model）** ：**`virtio`**。将默认的 `e1000` 或 `rtl8139` 改为 `virtio`，可获得接近物理网卡的吞吐量和更低的 CPU 占用。
- **网络源**：默认 NAT 即可满足大多数场景；如需虚拟机直接暴露在局域网中，可配置桥接网络（bridge）。

### 添加VirtIO驱动ISO作为第二个光驱

这是安装成功的关键步骤。在 virt-manager 虚拟机详情页中：

1. 点击 **“添加硬件” → “存储”**。
2. **设备类型**：`CDROM 设备`。
3. **总线类型**：`SATA`。
4. **存储源**：选择或浏览到 `virtio-win.iso`。
5. 勾选 **“只读”**。

确保虚拟机此时有两个光驱：一个指向 Windows 10 ISO，另一个指向 virtio-win ISO。

### 显示协议 

默认即可。

- 类型：Spice 服务器。
- 监听类型：地址。
- 地址：虚拟机管理程序默认。
- 端口：自动。

密码 和 OpenGL 选项和选项框 均为空。

### 显卡

- 型号：Virtio
- 3d加速：不启用

### TPM（可选，或者直接删除这个设备）

- 类型：模拟
- 型号：TIS
- 版本：2.0

### 引导顺序

在“引导选项”中，确保包含 Windows 10 ISO 的光驱和包含 virtio-win ISO 的光驱都处于可引导状态。

通常将 Windows 安装镜像放在首位；VirtIO 驱动光驱只需要能被 Windows 安装程序读取到即可，不要求从它引导。

## 安装过程中的关键操作-加载VirtIO存储驱动

点击左上角的 “开始安装”按钮。

启动虚拟机，一直保持快速按下 回车键，正常进入 Windows 10 安装程序。

当进行到 **“你想将 Windows 安装在哪里？”** 步骤时，**磁盘列表会是空的**——因为安装程序看不到 VirtIO 磁盘。

此时：

1. 点击 **“加载驱动程序”** 。
2. 点击 **“浏览”** ，定位到 virtio-win 光驱。
3. 对于 Windows 10 x64，进入：

   ```
   viostor\w10\amd64\

   或者

   amd64\w10\
   ```

   选择并安装 `viostor.inf`。

4. 驱动加载完成后，虚拟磁盘会出现在列表中，即可正常分区和安装。

如果此时还想一并加载网卡驱动，可以点击“加载驱动程序” → 浏览到 `NetKVM\w10\amd64\`，加载 `netkvm.inf`，这样系统安装完成后网卡即可立即工作。

## 安装后的驱动与工具安装

### 首选方案：运行 virtio-win-guest-tools.exe

进入 Windows 桌面后，打开 virtio-win 光驱，直接运行根目录下的 **`virtio-win-guest-tools.exe`**。该程序会自动安装：

- 所有剩余 VirtIO 驱动（网卡 `NetKVM`、Balloon 内存气球、串口等）
- **QEMU Guest Agent**（`qemu-ga`）

这是最省事、最不容易遗漏的方式。

### 备用方案：设备管理器手动安装

如果 `virtio-win-guest-tools.exe` 因某种原因不可用，可以在设备管理器中找到带黄色感叹号的设备（如“以太网控制器”），
右键“更新驱动程序” → “浏览计算机以查找驱动程序”，指向 virtio-win 光驱中对应目录：


| 设备 | 驱动目录 |
|---|---|
| 网卡 | `NetKVM\w10\amd64\` |
| Balloon（内存气球） | `Balloon\w10\amd64\` |
| 串口 / 其他 | 对应 `w10\amd64` 子目录 |


手动安装网卡驱动时，**不要通过远程桌面操作**——安装过程会更新网卡驱动，可能导致当前网络连接中断。应通过 virt-manager 的图形控制台（SPICE/VNC）操作。

### 验证驱动是否全部就位

安装完成后，在设备管理器中确认 **没有黄色感叹号设备**。也可以用 PowerShell 快速查看是否有未识别设备：

```powershell
Get-PnpDevice -Status Error, Unknown | Format-Table -AutoSize
```

验收清单：

- [ ] 设备管理器无黄色感叹号
- [ ] `virsh dumpxml <vm> | grep "cpu mode"` 显示 `host-passthrough`
- [ ] `virsh dumpxml <vm> | grep -E "model type='virtio'"` 网卡/磁盘均为 virtio
- [ ] Windows 内 `Get-Service qemu-ga` 为 Running
- [ ] `virsh dumpxml <vm> | grep hypervclock` 有输出

## 性能微调与进阶优化

### 确认CPU模式为host-passthrough

已在 [CPU配置](#CPU配置) 一节设置

```bash
virsh dumpxml <虚拟机名> | grep -i "cpu mode"
```

```xml
  <cpu mode="host-passthrough" check="none" migratable="on">
    <topology sockets="1" dies="1" clusters="1" cores="4" threads="2"/>
  </cpu>
```

### 启用Hyper-V enlightenments（显著提升 Windows 性能）

目前 KVM / QEMU 默认启动了很多优化特性。

这里使用新版支持的 `reset` 和 `reenlightenment`。

**推荐做法**：在 virt-manager 中打开虚拟机的 XML 编辑（`virsh edit <虚拟机名>`），在 `<features>` 段中添加。

> [!TIP]
> reset、reenlightenment 需 libvirt ≥ 7.x / QEMU ≥ 5.x；
> 若 virsh edit 后启动报 "unknown feature"，删掉对应行即可，其余项不受影响。

我的 `<features>` 段如下：

```xml
  <features>
    <acpi/>
    <apic/>
    <hyperv mode="custom">
      <relaxed state="on"/>
      <vapic state="on"/>
      <spinlocks state="on" retries="8191"/>
      <vpindex state="on"/>
      <runtime state="on"/>
      <synic state="on"/>
      <stimer state="on">
        <direct state="on"/>
      </stimer>
      <reset state="on"/>
      <frequencies state="on"/>
      <reenlightenment state="on"/>
      <tlbflush state="on"/>
      <ipi state="on"/>
      <evmcs state="on"/>
      <avic state="on"/>
    </hyperv>
    <kvm>
      <hidden state="on"/>
    </kvm>
    <vmport state="off"/>
    <smm state="off"/>
    <ioapic driver="kvm"/>
  </features>
```

> [!TIP]
> `evmcs` 仅 Intel 平台有效，`avic` 仅 AMD 平台有效。
> 保留不匹配的项通常无害（可设为 `off` 或直接删除）；若启动报错再删。
> 
> <smm> 是 Secure Boot 所需；若不启用 Secure Boot 可设为 off，略微减少开销。
> 
> vendor_id 与 <kvm><hidden> 用于隐藏虚拟化特征（如规避 NVIDIA 驱动检测），与性能无关，无此需求可删除。

### 配置时钟源（hypervclock）

在虚拟机的 `<clock>` 段中启用 `hypervclock`，让 Windows 使用 Hyper-V 半虚拟化时钟源，减少时间漂移和定时器轮询带来的 CPU 开销。

> [!TIP]
> 通常已默认启用，可用以下命令确认；若未启用再手动添加。 

```bash
virsh dumpxml <vm> | grep -A2 "<clock"
```


```xml
<clock offset='localtime'>
  <timer name='hypervclock' present='yes'/>
</clock>
```

我的 `<clock>` 段如下：

```xml
  <clock offset="localtime">
    <timer name="rtc" present="no" tickpolicy="catchup"/>
    <timer name="pit" present="no" tickpolicy="delay"/>
    <timer name="hpet" present="no"/>
    <timer name="kvmclock" present="no"/>
    <timer name="hypervclock" present="yes"/>
  </clock>
```

## 常见问题速查

### 问题 1：安装界面磁盘列表为空

**原因**：Windows 安装镜像不含 VirtIO 存储驱动。

**处理**：确认已添加 virtio-win ISO 光驱 → 在磁盘界面点击“加载驱动程序” → 浏览到 `viostor\w10\amd64\`（或 `vioscsi\w10\amd64\`）加载 `.inf` 文件。

### 问题 2：安装完成后没有网络，设备管理器显示“以太网控制器”黄色感叹号

**原因**：VirtIO 网卡驱动未安装。

**处理**：运行 `virtio-win-guest-tools.exe`，或在设备管理器中手动指向 `NetKVM\w10\amd64\`。

### 问题 3：虚拟机启动后直接进入 UEFI Shell，而不是 Windows 安装程序

**原因**：引导顺序不正确，或 OVMF 固件未能从光驱引导。
**处理**：
virt-manager 控制台点 "Send Key → Esc"。
检查 virt-manager “引导选项”中的设备顺序，确保 Windows 10 ISO 光驱排在首位；
确认固件为 UEFI 且光驱设备在引导列表中。

### 问题 4：`host-passthrough` 是否影响迁移？

**是**。`host-passthrough` 将宿主机 CPU 特性完全暴露给虚拟机，虚拟机只能在 CPU 相同的宿主机之间迁移。
如果未来有迁移需求，可改用 `host-model`（性能略低，但迁移兼容性更好），或在计划迁移时临时调整。

### 问题 5：是否必须启用 Secure Boot / TPM？

对于 **Windows 10**，通常**不需要**。除非有 BitLocker、Windows Hello 或特定安全合规需求，
否则可以保持 Secure Boot 和 TPM 关闭，简化安装与后续维护。Windows 11 则强制要求 vTPM 2.0 和 Secure Boot。

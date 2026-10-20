# KVM 使用 Virtio-FS 共享主机-虚拟机文件系统

> [!NOTE]
> By WuDiXianXin

Virtio-FS 是专为虚拟机设计的共享文件系统，性能接近本地文件系统。但 Windows 客户机（虚拟机系统）需要额外安装驱动和服务。

## 目录

- [主机端配置](#主机端配置)
- [Windows 10 虚拟机端配置](#Windows10虚拟机端配置)
- [理解 Target Path](#理解Target-Path)
- [Btrfs 重要优化：关闭 CoW](#Btrfs重要优化--关闭CoW)
- [相关链接](#相关链接)

## 主机端配置

1. **关闭虚拟机**。
2. 在 virt-manager 中打开虚拟机详情，在 **内存（Memory）** 选项卡中勾选 **“启用共享内存（Enable shared memory）”**。
3. 点击 **添加硬件（Add Hardware） → 文件系统（File System）**，按以下设置添加：
   - **驱动程序（Driver）**：`virtiofs`
   - **源路径（Source Path）**：主机上要共享的目录路径
   - **目标路径（Target Path）**：一个标识符，例如 `share_tag`，供客户机（虚拟机系统）挂载时使用
4. 应用后启动虚拟机。

## Windows10虚拟机端配置

### 准备 virtio-win 驱动 ISO

将 **virtio-win 驱动 ISO** 挂载到虚拟机的光驱，步骤如下：

1. 在 virt-manager 中打开虚拟机详情。
2. 点击 **添加硬件（Add Hardware） → 存储（Storage）**。
3. 按以下设置添加：
   - 选择或创建自定义存储，选择 virtio-win 驱动 ISO。
   - 如果还没有，可以从 [Fedora 项目页面](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/) 下载。
   - **设备类型（Device type）**：**CDROM 设备**
   - **总线类型（Bus type）**：**SATA**
   - **高级选项**：只读（勾选）

### 安装 WinFsp

- 从 [WinFsp 下载页面](https://winfsp.dev/rel/) 下载最新的 `.msi` 安装包。
- 运行安装程序，例如 `winfsp-*.msi`，安装时至少勾选 **“Core”** 功能。
- 完成安装后，重启 Windows 虚拟机。

### 安装 virtio-win 驱动与客户机工具

- 确保 virtio-win ISO 已挂载到虚拟机的光驱。
- 进入光驱目录，运行 `virtio-win-guest-tools.exe`，按照提示完成安装。
- 安装完成后，再次重启虚拟机。

### 手动启动 VirtioFS 服务

这是最关键的一步，很多教程会漏掉。`virtio-win-guest-tools.exe` 可能不会自动启动服务。

1. 在 Windows 中按 `Win + R`，输入 `services.msc` 并回车，打开“服务”窗口。
2. 在列表中找到 **“VirtIO-FS Service”**，服务名通常是 `VirtioFsSvc`。
3. 右键点击它，选择 **“属性”**。
4. 将 **“启动类型”** 改为 **“自动”**，然后点击 **“启动”**。
5. 确认服务状态变为“正在运行”后，点击 **“应用”** 和 **“确定”**。

### 验证共享目录

- 打开文件资源管理器，点击“此电脑”。
- 如果一切正常，你应该能看到一个新的驱动器盘符，通常从 `Z:` 开始，它对应的就是你在 virt-manager 中设置的共享文件夹。
- 如果没有自动出现，可以手动运行 `virtiofs.exe` 服务，并指定 `share_tag` 来挂载。

> 根据 Red Hat 的文档，每次虚拟机重启后，可能需要重新启动 VirtioFS 服务才能访问共享目录。将其设置为“自动”启动可以简化这个过程。

## 理解Target-Path

“Target Path” 在这里**不是虚拟机里的真实目录路径**，而是给这个 Virtio-FS 共享起的一个**标签名 / 别名 / 门牌号**。客户机挂载时靠这个名字来找到主机共享出来的目录。

可以这样对应理解：

| virt-manager 字段 | 实际含义 | 例子 |
|---|---|---|
| Source Path | 主机 Linux 上真实要共享的目录 | `/home/你的用户名/vm-share` |
| Target Path | Virtio-FS 的挂载标签，随便起名，但要和客户机挂载时一致 | `share_tag` |

也就是说：

- **Source Path**：告诉 KVM：“把主机上这个目录共享出去。”
- **Target Path**：告诉客户机：“这个共享叫 `share_tag`，你以后用这个名字来挂载它。”

例如你在 virt-manager 里这样填：

```text
Source Path: /home/tom/vm-share
Target Path: share_tag
```

那么在虚拟机内部，你看到的不是 `/home/tom/vm-share`，而是要通过 `share_tag` 这个标签去挂载。

如果是 Linux 客户机，命令类似：

```bash
sudo mount -t virtiofs share_tag /mnt/share
```

如果是 Windows 客户机，安装好 WinFsp 和 `virtio-win-guest-tools` 后，Virtio-FS 服务会识别这个标签，通常会把它映射成一个盘符，比如 `Z:`。如果手动配置，也是指定 `share_tag` 这个标签，而不是指定主机路径。

命名建议：

- `share_tag` 只是一个示例名字，你可以改成 `myshare`、`vm_share`、`sharedocs` 等。
- 不要用中文、空格、斜杠，建议用英文、数字、下划线或连字符。
- 它必须和你在客户机里挂载时使用的名字一致。
- 它不决定 Windows 里最终是 `Z:` 还是 `Y:`，盘符由 Windows 那边分配或选择。

在 libvirt 的 XML 里，它对应的是：

```xml
<filesystem type="mount" accessmode="passthrough">
  <driver type="virtiofs"/>
  <source dir="/kvm/share_dir"/>
  <target dir="share_tag"/>
  <address type="pci" domain="0x0000" bus="0x06" slot="0x00" function="0x0"/>
</filesystem>
```

这里的 `<target dir='share_tag'/>` 就是 virt-manager 界面里的 “Target Path”。它本质上是 Virtio-FS 的 **mount tag**，不是路径。

## Btrfs重要优化--关闭CoW

如果共享目录或虚拟机磁盘镜像存放在 Btrfs 分区上，**强烈建议关闭该目录的 Copy-on-Write（CoW）功能**。

Btrfs 的 CoW 特性在处理虚拟机频繁读写的文件时，会导致严重的性能下降和碎片化。

创建共享目录或存放虚拟机镜像的目录后，立即执行：

```bash
sudo chattr +C /path/to/your/shared/folder
```

查看是否关闭 CoW：

```bash
sudo lsattr -d /kvm/iso
sudo lsattr -d /kvm/share_dir/
```

输出示例：

```text
---------------C------ /kvm/iso
---------------C------ /kvm/share_dir/
```

说明：

- 此命令会对该目录下**之后创建**的文件禁用 CoW。
- 对于已存在的文件，可能需要手动处理。
- 一个更彻底的实践是：专门为虚拟机创建一个 Btrfs 子卷，并对其禁用 CoW。

## 相关链接

- [virtio-win 驱动 ISO（Fedora 项目页面）](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/)
- [WinFsp 下载页面](https://winfsp.dev/rel/)

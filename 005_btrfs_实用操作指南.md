# Btrfs 实用操作指南

> 原作者：WuDiXianXin  
> 整理补充：附带目录、常用命令、维护与故障排查  
> 适用环境：Linux + `btrfs-progs`，示例设备 `/dev/nvme0n1p2` 请替换为你的实际分区  
> 警告：涉及删除子卷、回滚系统、`send/receive`、`balance`、`defragment` 等操作前，请先确认路径与备份。

---

## 目录

- [Btrfs简介](#Btrfs简介)
- [核心概念](#核心概念)
- [安装与准备](#安装与准备)
- [子卷规划建议](#子卷规划建议)
- [创建与挂载子卷](#创建与挂载子卷)
- [快照管理](#快照管理)
- [系统回滚恢复](#系统回滚恢复)
- [挂载单个快照提取文件](#挂载单个快照提取文件)
- [备份与迁移](#备份与迁移)
- [维护、检查与优化](#维护、检查与优化)
- [常用命令速查](#常用命令速查)
- [自动化工具推荐](#自动化工具推荐)
- [最佳实践与注意事项](#最佳实践与注意事项)
- [故障排查](#故障排查)
- [附录：快照脚本示例](#附录：快照脚本示例)

---

## Btrfs简介

Btrfs 是一种现代的 Copy-on-Write 文件系统，支持：

- 子卷（subvolume）
- 快照（snapshot）
- 压缩（zstd、lzo、zlib 等）
- RAID / 多设备
- 校验和（checksum）
- 自愈（配合 RAID1/RAID10 等）
- 发送/接收（send/receive）增量备份

其 CoW 特性使得快照和增量备份非常高效：快照创建几乎瞬间完成，实际新增物理空间仅为后续修改的差异部分。

> 重要：Btrfs 快照不是备份。快照与源子卷通常位于同一文件系统、同一块磁盘上。磁盘损坏、误删文件系统、勒索软件等仍可能导致数据全部丢失。重要数据请遵循 3-2-1 备份原则。

---

## 核心概念

| 概念 | 说明 |
|---|---|
| 子卷 subvolume | Btrfs 内可独立挂载的树，类似“轻量分区”，但共享底层空间 |
| 顶层子卷 top-level | `subvolid=5`，通常不直接挂载为 `/`，用于创建其他子卷 |
| 快照 snapshot | 子卷的 CoW 副本，可只读或可写 |
| 只读快照 | 常用于备份、回滚、`send/receive`，防止误改历史版本 |
| 默认子卷 | 挂载 Btrfs 时不指定 `subvol=` 时挂载的子卷 |
| CoW | 写时复制，修改数据时写新块，旧数据保留给快照引用 |
| scrub | 校验数据与校验和，发现并尝试修复损坏块 |
| balance | 重新分布数据块，可回收空间、调整 RAID 布局 |
| qgroup | 配额组，用于统计子卷/快照空间占用 |

---

## 安装与准备

常见发行版安装 `btrfs-progs`：

```bash
# Arch Linux / Manjaro
sudo pacman -S btrfs-progs

# Debian / Ubuntu
sudo apt update
sudo apt install btrfs-progs

# Fedora
sudo dnf install btrfs-progs

# openSUSE
sudo zypper install btrfsprogs
```

准备操作：

```bash
# 查看磁盘与文件系统
lsblk -f

# 查看 Btrfs 文件系统信息
sudo btrfs filesystem show
```

建议：

- 系统回滚、修复、迁移时，优先使用 Arch/Linux Live ISO，自带 `btrfs-progs`。
- 操作前确认设备名，例如 `/dev/nvme0n1p2`、`/dev/sda2`。
- 不要在已挂载为 `/` 且正在运行的系统上随意删除根子卷。

---

## 子卷规划建议

最常见的布局如下，便于快照管理，并与 `snapper`、`timeshift`、`grub-btrfs` 等工具兼容：

| 子卷 | 挂载点 | 说明 |
|---|---|---|
| `@` | `/` | 根文件系统主体 |
| `@home` | `/home` | 用户数据，建议独立快照 |
| `@log` | `/var/log` | 日志变化频繁，可不纳入根快照 |
| `@cache` | `/var/cache` | 缓存可丢弃，可不纳入快照 |
| `.snapshots` | `/.snapshots` | 集中存放快照，推荐点开头隐藏，且作为独立子卷 |

推荐：

- 根快照子卷用 `@`。
- 快照存储用 `.snapshots`，并作为独立子卷挂载到 `/.snapshots`。
- 这样回滚根子卷时，不会误删快照本身。
- 2025–2026 年社区常见做法与 openSUSE、Fedora Silverblue、Ubuntu 部分方案、Arch snapper 指南保持一致。

创建与挂载详细步骤可参考：  
[《001_安装_Arch_Linux.md》 → “btrfs 文件系统” 章节](https://gitee.com/wudixianxin/open/blob/main/001_安装_Arch_Linux.md)

---

## 创建与挂载子卷

以下示例适合在 Live ISO 或安装阶段操作。假设 Btrfs 分区为 `/dev/nvme0n1p2`。

### 创建子卷

```bash
# 挂载顶层子卷
sudo mount /dev/nvme0n1p2 /mnt

# 创建推荐子卷
sudo btrfs subvolume create /mnt/@
sudo btrfs subvolume create /mnt/@home
sudo btrfs subvolume create /mnt/@log
sudo btrfs subvolume create /mnt/@cache
sudo btrfs subvolume create /mnt/.snapshots

# 查看结果
sudo btrfs subvolume list /mnt
```

### 挂载子卷

```bash
sudo umount /mnt

# 挂载根子卷 @
sudo mount -o subvol=@,compress=zstd:3,noatime,space_cache=v2 /dev/nvme0n1p2 /mnt

# 创建挂载点
sudo mkdir -p /mnt/home
sudo mkdir -p /mnt/var/log
sudo mkdir -p /mnt/var/cache
sudo mkdir -p /mnt/.snapshots

# 挂载其他子卷
sudo mount -o subvol=@home,compress=zstd:3,noatime /dev/nvme0n1p2 /mnt/home
sudo mount -o subvol=@log,compress=zstd:3,noatime /dev/nvme0n1p2 /mnt/var/log
sudo mount -o subvol=@cache,compress=zstd:3,noatime /dev/nvme0n1p2 /mnt/var/cache
sudo mount -o subvol=.snapshots,compress=zstd:3,noatime /dev/nvme0n1p2 /mnt/.snapshots
```

### `/etc/fstab` 示例

```fstab
UUID=你的UUID /             btrfs subvol=@,compress=zstd:3,noatime,space_cache=v2 0 0
UUID=你的UUID /home         btrfs subvol=@home,compress=zstd:3,noatime             0 0
UUID=你的UUID /var/log      btrfs subvol=@log,compress=zstd:3,noatime              0 0
UUID=你的UUID /var/cache    btrfs subvol=@cache,compress=zstd:3,noatime            0 0
UUID=你的UUID /.snapshots   btrfs subvol=.snapshots,compress=zstd:3,noatime        0 0
```

> `space_cache=v2` 在新内核中通常已是默认值。旧内核若不支持可省略。  
> 使用 `UUID` 更稳妥，可用 `blkid` 查看。

---

## 快照管理

### 创建只读快照

日常备份强烈建议使用只读快照 `-r`，防止误操作修改历史版本。

命名建议：

- `YYYYMMDD-HHMM_用途`
- `YYYYMMDD-序号_用途`

便于排序和识别。

```bash
# 根子卷 @ 的只读快照
sudo btrfs subvolume snapshot -r / /.snapshots/$(date +%Y%m%d-%H%M)_root

# home 子卷的只读快照
sudo btrfs subvolume snapshot -r /home /.snapshots/$(date +%Y%m%d-%H%M)_home
```

输出示例：

```text
create readonly snapshot of '/' in '/.snapshots/20260120-1427_root'
```

小技巧：可写脚本自动化，配合 `cron` 或 `systemd timer` 定时创建。

### 查看快照与子卷

```bash
# 列出当前文件系统中的子卷
sudo btrfs subvolume list /

# 以表格形式查看
sudo btrfs subvolume list -t /

# 查看某个子卷/快照详情
sudo btrfs subvolume show /.snapshots/20260120-1427_root
```

### 删除快照

```bash
sudo btrfs subvolume delete /.snapshots/20260120-1427_root
```

批量清理建议：

- 写脚本按时间保留，例如保留最近 7 天、最近 4 周、最近 6 个月。
- 使用 `snapper`、`btrbk` 等工具自动管理保留策略。
- 删除快照后空间不一定立即释放，可能仍被其他快照引用；可结合 `btrfs filesystem usage` 观察。

---

## 系统回滚恢复

最安全方式：使用 Arch/Linux Live ISO 启动，自带 `btrfs-progs`。

> 前提：`.snapshots` 最好是独立子卷。否则删除 `@` 时可能连同快照一起删除。

### 确认分区

```bash
lsblk -f
```

### 挂载 Btrfs 文件系统

不指定 `subvol`，让它挂载默认子卷；通常可看到顶层子卷内容。

```bash
sudo mount /dev/nvme0n1p2 /mnt
```

### 查看所有子卷/快照

```bash
sudo btrfs subvolume list /mnt
```

### 恢复根子卷 `@`

```bash
# 先删除损坏的 @（如果 delete 失败，先清空内容）
sudo btrfs subvolume delete /mnt/@

# 如果删除失败，可先清空再删，但极度小心路径
# sudo rm -rf /mnt/@/* /mnt/@/.[!.]*
# sudo btrfs subvolume delete /mnt/@

# 从只读快照重建可写 @
sudo btrfs subvolume snapshot /mnt/.snapshots/20260120-1427_root /mnt/@
```

### 同理处理 `@home`

如果 `/home` 也需要回滚：

```bash
sudo btrfs subvolume delete /mnt/@home
sudo btrfs subvolume snapshot /mnt/.snapshots/20260120-1427_home /mnt/@home
```

### 卸载并重启

```bash
sudo umount /mnt
reboot
```

> 重要：只要子卷名字和原来一样，`/etc/fstab` 通常无需修改，因为 `subvol=@` 等选项不变。  
> 如果使用默认子卷启动，则可能需要 `btrfs subvolume set-default` 调整默认子卷。

---

## 挂载单个快照提取文件

只提取文件、不回滚整个系统时，可单独挂载某个快照。

```bash
sudo mkdir -p /mnt/snap

# 方式1：直接用 subvol= 选项挂载特定快照
sudo mount -t btrfs -o subvol=.snapshots/20260120-1427_root,ro \
     /dev/nvme0n1p2 /mnt/snap

# 方式2：如果已挂载根文件系统，可用 bind 方式
sudo mount --bind /.snapshots/20260120-1427_root /mnt/snap
sudo mount -o remount,ro,bind /mnt/snap

# 用完记得卸载
sudo umount /mnt/snap
```

> 注意：`subvol=` 路径是相对于 Btrfs 顶层子卷的路径。  
> 如果 `.snapshots` 是独立子卷，通常路径为 `.snapshots/快照名`。

---

## 备份与迁移

优点：

- 支持增量
- 保留 CoW 特性
- 校验和可靠

缺点：

- 目标也必须是 Btrfs
- 操作稍复杂
- 增量依赖父快照

### 完整备份（第一次）

源快照必须是只读的。

```bash
# 发送到文件
sudo btrfs send /.snapshots/20260120-1427_root > /path/to/external/20260120_root.btrfs

# 或直接管道到 receive，节省本地空间
sudo btrfs send /.snapshots/20260120-1427_root | sudo btrfs receive /mnt/backup
```

### 增量备份（后续）

```bash
# 先创建新只读快照
sudo btrfs subvolume snapshot -r / /.snapshots/$(date +%Y%m%d-%H%M)_root

# 增量发送，-p 指定共同祖先
sudo btrfs send -p /.snapshots/20260120-1427_root \
     /.snapshots/20260121-0930_root \
     | sudo btrfs receive /mnt/backup
```

常见错误提醒：

- 增量发送时，源和目标上必须存在相同的父快照，即 `-p` 指定的快照也要在接收端存在。
- 快照必须保持只读，否则 `send` 会失败或数据不一致。
- 接收端路径通常是已存在的普通目录，`receive` 会自动在该目录下创建同名子卷。
- 增量备份要按顺序接收：先基础，后增量。

### 从备份文件恢复（迁移 / 换盘）

1. 在目标机器挂载新的 Btrfs 分区到 `/mnt`。
2. 接收完整备份：

```bash
sudo btrfs receive /mnt < /path/to/20260120_root.btrfs

# 或 cat 方式
cat 20260120_root.btrfs | sudo btrfs receive /mnt
```

3. 增量备份按顺序依次接收。
4. 接收完成后，将子卷重命名为 `@` 或你原来的名字：

```bash
# 如果已有旧 @，先删除
sudo btrfs subvolume delete /mnt/@

# 重命名，视 btrfs-progs 版本可用以下之一
sudo btrfs subvolume rename /mnt/20260120-1427_root /mnt/@
# 或
sudo mv /mnt/20260120-1427_root /mnt/@
```

---

## 维护、检查与优化

### 查看空间使用

```bash
sudo btrfs filesystem df /
sudo btrfs filesystem usage -h /
sudo btrfs filesystem show
```

### scrub 检查数据完整性

```bash
# 前台运行并等待完成
sudo btrfs scrub start -B /

# 查看状态
sudo btrfs scrub status /
```

建议定期执行，尤其是 RAID1/RAID10 环境。发现坏块时，若有多副本，Btrfs 可尝试自愈。

### balance 重新分布数据

```bash
# 只整理使用率低于 50% 的数据块
sudo btrfs balance start -dusage=50 /

# 只整理元数据
sudo btrfs balance start -musage=50 /

# 查看状态
sudo btrfs balance status /
```

> 不要频繁全量 `balance`。它可能耗时很长，并对 SSD 产生大量写入。

### defragment 与压缩

```bash
# 递归整理并尝试 zstd 压缩
sudo btrfs filesystem defragment -r -czstd /
```

> 注意：`defragment` 可能破坏 CoW 共享，导致快照占用空间增加。对快照较多的系统请谨慎。

设置子卷/文件压缩属性：

```bash
# 查看属性
sudo btrfs property get /path

# 设置压缩
sudo btrfs property set /path compression zstd
```

挂载选项压缩：

```fstab
compress=zstd:3
```

### 只读属性

```bash
# 设为只读
sudo btrfs property set /.snapshots/20260120-1427_root ro true

# 取消只读
sudo btrfs property set /.snapshots/20260120-1427_root ro false
```

### 设备管理简例

```bash
# 添加设备
sudo btrfs device add /dev/sdb1 /

# 移除设备
sudo btrfs device remove /dev/sdb1 /

# 调整文件系统大小
sudo btrfs filesystem resize max /
```

### 默认子卷

```bash
# 查看默认子卷
sudo btrfs subvolume get-default /

# 设置默认子卷，很少需要
sudo btrfs subvolume set-default <subvolid> /
```

---

## 常用命令速查

| 功能 | 命令 |
|---|---|
| 查看文件系统 | `sudo btrfs filesystem show` |
| 查看空间 | `sudo btrfs filesystem df /` |
| 查看使用率 | `sudo btrfs filesystem usage -h /` |
| 列出子卷 | `sudo btrfs subvolume list /` |
| 创建子卷 | `sudo btrfs subvolume create /mnt/@` |
| 创建只读快照 | `sudo btrfs subvolume snapshot -r / /.snapshots/日期_root` |
| 删除快照 | `sudo btrfs subvolume delete /.snapshots/日期_root` |
| 查看子卷详情 | `sudo btrfs subvolume show /.snapshots/日期_root` |
| 挂载快照 | `sudo mount -t btrfs -o subvol=.snapshots/日期_root,ro /dev/nvme0n1p2 /mnt/snap` |
| 完整发送 | `sudo btrfs send /.snapshots/snap1 \| sudo btrfs receive /mnt/backup` |
| 增量发送 | `sudo btrfs send -p /.snapshots/snap1 /.snapshots/snap2 \| sudo btrfs receive /mnt/backup` |
| scrub | `sudo btrfs scrub start -B /` |
| balance | `sudo btrfs balance start -dusage=50 /` |
| 压缩整理 | `sudo btrfs filesystem defragment -r -czstd /` |
| 查看属性 | `sudo btrfs property get /path` |
| 设置只读 | `sudo btrfs property set /path ro true` |
| 设置默认子卷 | `sudo btrfs subvolume set-default <subvolid> /` |

---

## 自动化工具推荐

| 工具 | 用途 |
|---|---|
| `snapper` | openSUSE 系快照管理，支持保留策略 |
| `snap-pac` | Arch 下配合 pacman，自动在安装/升级前后创建快照 |
| `btrbk` | 强大的 Btrfs 快照与 `send/receive` 备份工具 |
| `timeshift` | 图形化、简单易用，适合桌面用户 |
| `grub-btrfs` | 在 GRUB 菜单中直接启动到 Btrfs 快照 |
| `btrfs-assistant` | 图形化管理 Btrfs、snapper、grub-btrfs 等 |

建议：

- 初学者优先使用自动化工具，减少手误。
- 自动工具也要定期验证快照能否正常挂载和读取。
- 生产环境建议结合 `btrbk` 做异地增量备份。

---

## 最佳实践与注意事项

- 快照尽量使用只读：`btrfs subvolume snapshot -r`。
- 快照不是备份，重要数据遵循 3-2-1 原则：3 份副本、2 种介质、1 份异地。
- `.snapshots` 建议作为独立子卷挂载到 `/.snapshots`。
- 命名统一：`YYYYMMDD-HHMM_用途`，便于排序和清理。
- 系统升级、内核更新、重大配置变更前先创建快照。
- 定期 `scrub`，尤其是 RAID 环境。
- 监控空间：`btrfs filesystem usage -h /`。
- 谨慎使用 `defragment`，它可能破坏 CoW 共享，增加快照占用。
- `balance` 不要频繁全量运行，按需使用 `-dusage` / `-musage`。
- 定期测试恢复流程，确保快照和备份真的可用。
- 删除快照后空间未释放时，检查是否被其他快照引用。
- 跨机迁移时，注意 `send/receive` 的父快照依赖关系。
- 操作根子卷、删除 `@`、回滚系统前，务必确认路径和设备。

---

## 故障排查

### `btrfs send` 失败

可能原因：

- 快照不是只读。
- 增量发送的父快照在接收端不存在。
- 源和目标文件系统版本或特性不兼容。
- 路径写错。

处理：

```bash
sudo btrfs property get /.snapshots/快照名
sudo btrfs subvolume list /mnt/backup
```

### 删除快照后空间未释放

可能原因：

- 该快照仍被其他快照或子卷引用。
- 需要等待事务提交。
- 可尝试 `btrfs filesystem sync /`，或适度 `balance`。

```bash
sudo btrfs filesystem sync /
sudo btrfs filesystem usage -h /
```

### 挂载快照失败

检查：

- `subvol=` 路径是否正确。
- `.snapshots` 是否为独立子卷。
- 设备 UUID 是否正确。
- 是否只读挂载与当前状态冲突。

```bash
lsblk -f
sudo btrfs subvolume list /mnt
```

### 无法删除子卷

可能原因：

- 子卷内还有嵌套子卷。
- 路径错误。
- 文件系统只读。

处理：

```bash
sudo btrfs subvolume list /mnt
# 先删除嵌套子卷，再删除父级
```

### 空间满

```bash
sudo btrfs filesystem usage -h /
sudo btrfs filesystem df /
```

处理方向：

- 删除不需要的快照。
- 清理日志、缓存、包缓存。
- 适度 `balance`。
- 检查是否有异常大文件或快照保留过多。

### scrub 报错

- 检查硬件健康：`smartctl`。
- RAID1/RAID10 下可尝试修复。
- 单盘无冗余时，scrub 只能发现错误，不能修复。
- 重要数据立即从备份恢复。

---

## 附录：快照脚本示例

### 创建根与 home 只读快照

```bash
#!/usr/bin/env bash
set -euo pipefail

SNAP_DIR="/.snapshots"
TS="$(date +%Y%m%d-%H%M)"

sudo btrfs subvolume snapshot -r / "$SNAP_DIR/${TS}_root"
sudo btrfs subvolume snapshot -r /home "$SNAP_DIR/${TS}_home"

echo "Created snapshots:"
echo "  $SNAP_DIR/${TS}_root"
echo "  $SNAP_DIR/${TS}_home"
```

保存为 `/usr/local/bin/btrfs-snapshot.sh`：

```bash
sudo chmod +x /usr/local/bin/btrfs-snapshot.sh
```

### systemd timer 示例

`/etc/systemd/system/btrfs-snapshot.service`：

```ini
[Unit]
Description=Create Btrfs snapshots

[Service]
Type=oneshot
ExecStart=/usr/local/bin/btrfs-snapshot.sh
```

`/etc/systemd/system/btrfs-snapshot.timer`：

```ini
[Unit]
Description=Run Btrfs snapshot daily

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

启用：

```bash
sudo systemctl enable --now btrfs-snapshot.timer
```

### 简单保留策略思路

- 保留最近 7 天每日快照。
- 保留最近 4 周每周快照。
- 保留最近 6 个月每月快照。
- 使用 `btrbk`、`snapper` 等工具更可靠。

---

## 参考与延伸

- [《001_安装_Arch_Linux.md》 → “btrfs 文件系统” 章节](https://gitee.com/wudixianxin/open/blob/main/001_安装_Arch_Linux.md)
- Arch Wiki：Btrfs
- `man btrfs`、`man btrfs-subvolume`、`man btrfs-send`、`man btrfs-receive`
- `snapper`、`btrbk`、`timeshift`、`grub-btrfs` 官方文档

---

希望这份整理补充版对你和看到的人有帮助！欢迎补充/纠错～

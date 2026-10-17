# Btrfs 实用操作指南（CachyOS 实测修订版）

> 原作者：WuDiXianXin  
> 整理补充：附带目录、常用命令、维护与故障排查  
> 本版修订：基于 CachyOS / Arch Linux 实际部署（NVMe SSD + Btrfs + snapper + zram），
> 与一两年前的初版相比，主要在**子卷布局、压缩级别、nodatacow 使用、快照嵌套、tmpfs/zram、挂载选项**上做了对齐。  
> 适用环境：Linux + `btrfs-progs`，示例设备 `/dev/nvme0n1p3` 请替换为你的实际分区  
> 警告：涉及删除子卷、回滚系统、`send/receive`、`balance`、`defragment` 等操作前，请先确认路径与备份。

---

## 目录

- [简介](#简介)
- [核心概念](#核心概念)
- [安装与准备](#安装与准备)
- [子卷规划建议](#子卷规划建议)
- [创建与挂载子卷](#创建与挂载子卷)
- [fstab 完整示例](#fstab-完整示例)
- [zram 与 tmpfs 的配合](#zram-与-tmpfs-的配合)
- [快照管理](#快照管理)
- [系统回滚恢复](#系统回滚恢复)
- [挂载单个快照提取文件](#挂载单个快照提取文件)
- [备份与迁移](#备份与迁移)
- [维护、检查与优化](#维护-检查-优化)
- [常用命令速查](#常用命令速查)
- [自动化工具推荐](#自动化工具推荐)
- [最佳实践与注意事项](#最佳实践与注意事项)
- [故障排查](#故障排查)
- [附录-快照脚本示例](#附录-快照脚本示例)

---

## 简介

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
| nodatacow | 关闭数据 CoW，适合日志、缓存、虚拟机镜像等写多场景 |
| scrub | 校验数据与校验和，发现并尝试修复损坏块 |
| balance | 重新分布数据块，可回收空间、调整 RAID 布局 |
| qgroup | 配额组，用于统计子卷/快照空间占用 |

---

## 安装与准备

常见发行版安装 `btrfs-progs`：

```bash
# Arch Linux / CachyOS / Manjaro
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
lsblk -pf

# 查看 Btrfs 文件系统信息
sudo btrfs filesystem show

# 查看子卷列表（表格形式）
sudo btrfs subvolume list -t /
```

建议：

- 系统回滚、修复、迁移时，优先使用 Arch/CachyOS Live ISO，自带 `btrfs-progs`。
- 操作前确认设备名，例如 `/dev/nvme0n1p3`。
- 不要在已挂载为 `/` 且正在运行的系统上随意删除根子卷。

---

## 子卷规划建议

**本版布局对应实际部署**（NVMe 单盘 + snapper 自动快照），比初版的 `@ / @home / @log / @cache / .snapshots` 更细，主要新增：`@root`、`@srv`、`@tmp`、`@kvm`、`@git`，并把快照目录改为**嵌套子卷**（snapper 自动管理）。

| 子卷 | 挂载点 | 说明 |
|---|---|---|
| `@` | `/` | 根文件系统主体，含 `.snapshots` 嵌套子卷 |
| `@home` | `/home` | 用户数据，独立快照；含 `@home/.snapshots` 嵌套子卷 |
| `@root` | `/root` | root 用户家目录，独立子卷，避免混入根快照 |
| `@srv` | `/srv` | 服务数据 |
| `@log` | `/var/log` | 日志，变化频繁，**nodatacow + noexec/nodev/nosuid** |
| `@cache` | `/var/cache` | 缓存，**nodatacow + noexec/nodev/nosuid** |
| `@tmp` | `/var/tmp` | 临时数据，**nodatacow + noexec/nodev/nosuid** |
| `@kvm` | `/kvm` | 虚拟机镜像，**nodatacow**（避免 CoW 放大写入） |
| `@git` | `/git` | 代码仓库，保留 CoW + 压缩 |
| `.snapshots` | 嵌套在 `@` 下 | snapper 自动挂载，**不写入 fstab** |
| `@home/.snapshots` | 嵌套在 `@home` 下 | 同上 |

**与初版的关键差异：**

1. **快照目录不再作为顶层独立子卷挂到 `/.snapshots`**，而是作为 `@` 的嵌套子卷（`subvolid=5` 下的 `.snapshots`，top level 是 `@` 的 ID）。snapper 会在 `@` 下自动创建并管理，**fstab 中不写这一条**。
2. **`/var/log`、`/var/cache`、`/var/tmp`、`/kvm` 使用 `nodatacow`**：这些目录写入频繁、数据寿命短，CoW + 压缩反而增加写放大。
3. **压缩级别用 `zstd:1` 而不是 `zstd:3`**：NVMe SSD 上 zstd:1 已经能拿到大部分压缩收益，CPU 开销更低，配合 `discard=async` 体验更好。如果你更在意空间可以升到 `zstd:3`。
4. **`space_cache=v2` 已省略**：新内核（≥ 5.15）默认就是 v2，无需显式写。
5. **显式 `ssd,discard=async`**：`ssd` 让 btrfs 使用 SSD 优化路径，`discard=async` 由内核在后台异步 TRIM，比同步 discard 更温和。

---

## 创建与挂载子卷

以下示例适合在 Live ISO 或安装阶段操作。假设 Btrfs 分区为 `/dev/nvme0n1p3`。

### 创建子卷

```bash
# 挂载顶层子卷（subvolid=5）
sudo mount /dev/nvme0n1p3 /mnt

# 创建推荐子卷
sudo btrfs subvolume create /mnt/@
sudo btrfs subvolume create /mnt/@home
sudo btrfs subvolume create /mnt/@root
sudo btrfs subvolume create /mnt/@srv
sudo btrfs subvolume create /mnt/@log
sudo btrfs subvolume create /mnt/@cache
sudo btrfs subvolume create /mnt/@tmp
sudo btrfs subvolume create /mnt/@kvm
sudo btrfs subvolume create /mnt/@git

# 查看结果
sudo btrfs subvolume list /mnt

# 说明：.snapshots 与 @home/.snapshots 由 snapper 首次运行时自动创建，
# 也可以手动预建（推荐）：
sudo btrfs subvolume create /mnt/@/.snapshots
sudo btrfs subvolume create /mnt/@home/.snapshots
```

### 挂载子卷（安装阶段）

```bash
sudo umount /mnt

# 挂载根子卷 @
sudo mount -o subvol=@,compress=zstd:1,noatime,ssd,discard=async /dev/nvme0n1p3 /mnt

# 创建挂载点
sudo mkdir -p /mnt/{home,root,srv,var/log,var/cache,var/tmp,kvm,git}

# 挂载其余子卷
sudo mount -o subvol=@home,compress=zstd:1,noatime,ssd,discard=async   /dev/nvme0n1p3 /mnt/home
sudo mount -o subvol=@root,compress=zstd:1,noatime,ssd,discard=async   /dev/nvme0n1p3 /mnt/root
sudo mount -o subvol=@srv,compress=zstd:1,noatime,ssd,discard=async    /dev/nvme0n1p3 /mnt/srv
sudo mount -o subvol=@log,noatime,nodatacow,noexec,nodev,nosuid,ssd,discard=async   /dev/nvme0n1p3 /mnt/var/log
sudo mount -o subvol=@cache,noatime,nodatacow,noexec,nodev,nosuid,ssd,discard=async /dev/nvme0n1p3 /mnt/var/cache
sudo mount -o subvol=@tmp,noatime,nodatacow,noexec,nodev,nosuid,ssd,discard=async   /dev/nvme0n1p3 /mnt/var/tmp
sudo mount -o subvol=@kvm,noatime,nodatacow,noexec,nodev,nosuid,ssd,discard=async   /dev/nvme0n1p3 /mnt/kvm
sudo mount -o subvol=@git,compress=zstd:1,noatime,ssd,discard=async    /dev/nvme0n1p3 /mnt/git
```

### fstab-完整示例

**这是本次修订的核心**，与初版差异最大。所有 btrfs 子卷 `pass=0`（btrfs 不做传统 fsck）；`/boot` 是 vfat，`pass=2`。

```fstab
# <file system>             <mount point>  <type>  <options>  <dump>  <pass>
UUID=4BE6-F7C6                            /boot          vfat    rw,noatime,nodev,nosuid,noexec,fmask=0177,dmask=0077,shortname=mixed,utf8,errors=remount-ro 0 2
UUID=bd857a9e-d85f-479e-be0b-a97e782e9a7c swap           swap    defaults   0 0

UUID=266a2b91-1b69-46e4-a562-5177e3fb243a /              btrfs   rw,noatime,compress=zstd:1,ssd,discard=async,subvol=/@          0 0
UUID=266a2b91-1b69-46e4-a562-5177e3fb243a /home          btrfs   rw,noatime,compress=zstd:1,ssd,discard=async,subvol=/@home      0 0
UUID=266a2b91-1b69-46e4-a562-5177e3fb243a /root          btrfs   rw,noatime,compress=zstd:1,ssd,discard=async,subvol=/@root      0 0
UUID=266a2b91-1b69-46e4-a562-5177e3fb243a /srv           btrfs   rw,noatime,compress=zstd:1,ssd,discard=async,subvol=/@srv       0 0

UUID=266a2b91-1b69-46e4-a562-5177e3fb243a /var/log       btrfs   rw,noatime,nodatacow,noexec,nodev,nosuid,ssd,discard=async,commit=90,subvol=/@log    0 0
UUID=266a2b91-1b69-46e4-a562-5177e3fb243a /var/cache     btrfs   rw,noatime,nodatacow,noexec,nodev,nosuid,ssd,discard=async,commit=120,subvol=/@cache 0 0
UUID=266a2b91-1b69-46e4-a562-5177e3fb243a /var/tmp       btrfs   rw,noatime,nodatacow,noexec,nodev,nosuid,ssd,discard=async,commit=120,subvol=/@tmp   0 0

UUID=266a2b91-1b69-46e4-a562-5177e3fb243a /kvm           btrfs   rw,noatime,nodatacow,noexec,nodev,nosuid,ssd,discard=async,commit=60,subvol=/@kvm    0 0
UUID=266a2b91-1b69-46e4-a562-5177e3fb243a /git           btrfs   rw,noatime,compress=zstd:1,ssd,discard=async,subvol=/@git          0 0

tmpfs                                     /tmp           tmpfs   defaults,noatime,mode=1777 0 0
```

**关于挂载选项的补充说明：**

- `compress=zstd:1`：NVMe 上性价比最高，想更省空间可换 `zstd:3`。
- `nodatacow`：只对 `/var/log`、`/var/cache`、`/var/tmp`、`/kvm` 使用，其他子卷保留 CoW 以便快照。
- `noexec,nodev,nosuid`：对上述四个“写多且不需要执行”的子卷做安全加固。
- `commit=90/120/60`：拉长事务提交间隔，减少 btrfs 事务写入，代价是掉电时可能丢最近的几十秒日志/缓存。
- `/boot` 的 `fmask=0177,dmask=0077`：等价于文件 `0600`、目录 `0700`。若你用 GRUB 或非 root 工具改 /boot，建议改成 `fmask=0137,dmask=0077`（即文件 0640、目录 0700）。
- `.snapshots` 与 `@home/.snapshots` **不写 fstab**，由 snapper 自动挂载。
- `space_cache=v2` 已省略，新内核默认。

---

## zram-与-tmpfs-的配合

**这是初版完全没提的内容**，但在 CachyOS / 现代 Arch 上是标配。

### zram

CachyOS 默认安装 `zram-generator`（`/etc/systemd/zram-generator.conf`），开机自动创建 zram 块设备并作为 swap 使用。典型大小是物理内存的一半或全部。

```bash
# 查看 zram 状态
zramctl
cat /proc/swaps
swapon --show

# 查看 zram-generator 配置
cat /etc/systemd/zram-generator.conf
```

**优先级注意事项：**

- zram 的 swap 优先级应**高于**磁盘 swap，否则内核会优先写磁盘。
- `zram-generator` 默认给 zram `pri=100`；fstab 里的磁盘 swap 默认 `pri=-2`，一般已经正确。
- 用 `cat /proc/swaps` 一眼确认：

```text
Filename                Type            Size            Used    Priority
/dev/zram0              partition       16035712        0       100
/dev/nvme0n1p2          partition       16777212        0       -2
```

### tmpfs on /tmp

初版 fstab 示例里没有 `/tmp`，本版显式把 `/tmp` 挂到 tmpfs：

```fstab
tmpfs  /tmp  tmpfs  defaults,noatime,mode=1777  0 0
```

理由：

- `/tmp` 在 FHS 里语义是“可清空、不跨重启保留”，tmpfs 天然符合。
- 减少 SSD 写入。
- 大文件（如编译中间产物、下载缓存）会占用内存，注意内存足够。

**注意：** `/var/tmp` 是**持久化**的临时目录（跨重启保留），所以**不能**换成 tmpfs，仍然落在 btrfs `@tmp` 上。

---

## 快照管理

### 本版关键差异：快照由 snapper 自动管理

初版是手动 `btrfs subvolume snapshot -r`，本版实际用的是 **snapper + snap-pac**：

- `snapper` 在 `@` 下创建 `.snapshots` 嵌套子卷，并在 `@home` 下创建 `@home/.snapshots`。
- `snap-pac` 在每次 `pacman` 事务前后自动创建 pre/post 快照。
- `snapper` 通过 `/etc/snapper/configs/root`、`/etc/snapper/configs/home` 管理保留策略。

> [!TIP]
> 也可以使用 btrfs-assistant 进行图形化 使用 snapper 管理哟~

**查看当前快照：**

```bash
# 系统级（对应 @）
sudo snapper -c root list

# 用户数据级（对应 @home）
sudo snapper -c home list

# 底层 btrfs 视角
sudo btrfs subvolume list -t /

# 某个快照详情
sudo snapper -c root status 1..0
```

**手动创建快照：**

```bash
# 创建一对 pre/post 快照
sudo snapper -c root create --command "pacman -Syu"

# 只创建单个快照
sudo snapper -c root create --description "before upgrade"
sudo snapper -c home create --description "before big move"
```

**删除与清理：**

```bash
# 删除某个快照
sudo snapper -c root delete <编号>

# 按保留策略自动清理（snapper 会跑定时器）
sudo snapper -c root cleanup number
```

### 手动创建只读快照（仍可用）

如果你不想用 snapper，或想给非系统子卷（`@git`、`@srv`、`@kvm`）单独做快照，仍可手动：

```bash
# 注意：@git、@srv、@kvm 没有自己的 .snapshots 子卷，
# 需要先建目录（在 btrfs 里就是普通目录），或者放到 @ 的 .snapshots 下。
sudo mkdir -p /git/.snapshots
sudo btrfs subvolume snapshot -r /git /git/.snapshots/$(date +%Y%m%d-%H%M)_git
```

命名建议：

- `YYYYMMDD-HHMM_用途`
- `YYYYMMDD-序号_用途`

便于排序和识别。

### 查看快照与子卷

```bash
# 列出当前文件系统中的子卷
sudo btrfs subvolume list /

# 以表格形式查看（含 top level 层级）
sudo btrfs subvolume list -t /

# 查看某个子卷/快照详情
sudo btrfs subvolume show /.snapshots/1/snapshot
```

### 删除快照

```bash
# 手动删除（谨慎，snapper 管理的快照请用 snapper delete）
sudo btrfs subvolume delete /.snapshots/20260120-1427_root
```

批量清理建议：

- snapper 用户直接用 `snapper cleanup`。
- 非 snapper 用户写脚本按时间保留，例如保留最近 7 天、最近 4 周、最近 6 个月。
- 删除快照后空间不一定立即释放，可能仍被其他快照引用；可结合 `btrfs filesystem usage` 观察。

---

## 系统回滚恢复

最安全方式：使用 Arch/CachyOS Live ISO 启动，自带 `btrfs-progs`。

> 前提：`.snapshots` 最好是独立子卷或嵌套子卷。本版方案里 `.snapshots` 嵌套在 `@` 下，回滚 `@` 时要小心——**只读快照本身在 `@` 之外还是之内，决定了回滚步骤**。snapper 的方案是把 `.snapshots` 放在 `@` 内部，所以不能简单“删除 @ 再重建”，需要用 snapper 的 rollback 或者手动搬运。

### 方式一：使用 snapper rollback（推荐）

如果你用 snapper，直接：

```bash
# 从 Live ISO 或从运行系统（重启后生效）
sudo snapper -c root rollback <快照编号>
sudo reboot
```

snapper 会把该快照复制为新的 `@`，并把旧的 `@` 保留为 `@.old`（可后续删除）。

### 方式二：手动回滚（无 snapper 或想完全控制）

```bash
# 1. 启动 Live ISO，确认分区
lsblk -pf

# 2. 挂载顶层子卷
sudo mount /dev/nvme0n1p3 /mnt

# 3. 查看所有子卷/快照
sudo btrfs subvolume list -t /mnt

# 4. 假设要恢复的快照是 /mnt/@/.snapshots/1/snapshot
#    先把旧 @ 改名（不要直接删，留作保险）
sudo mv /mnt/@ /mnt/@.old

# 5. 从只读快照重建可写 @（-r 源是只读，重建的是可写副本）
sudo btrfs subvolume snapshot /mnt/@.old/.snapshots/1/snapshot /mnt/@

# 6. 把 .snapshots 搬回新 @ 内部（如果快照目录在 @ 里）
#    注意：如果 .snapshots 是嵌套在 @ 下的独立子卷，
#    上面的 snapshot 会把它一起带过来；如果是独立顶层子卷，则不用搬。

# 7. 卸载并重启
sudo umount /mnt
reboot
```

> 重要：只要子卷名字和原来一样，`/etc/fstab` 通常无需修改，因为 `subvol=@` 等选项不变。  
> 如果使用默认子卷启动，则可能需要 `btrfs subvolume set-default` 调整默认子卷。

### 同理处理 `@home`

如果 `/home` 也需要回滚（snapper 的 home 配置）：

```bash
sudo snapper -c home rollback <快照编号>
# 或手动
sudo mv /mnt/@home /mnt/@home.old
sudo btrfs subvolume snapshot /mnt/@home.old/.snapshots/1/snapshot /mnt/@home
```

---

## 挂载单个快照提取文件

只提取文件、不回滚整个系统时，可单独挂载某个快照。

```bash
sudo mkdir -p /mnt/snap

# 方式1：直接用 subvol= 选项挂载 snapper 快照
# snapper 快照路径通常是 @/.snapshots/<编号>/snapshot
sudo mount -t btrfs -o subvol=/@/.snapshots/1/snapshot,ro \
     /dev/nvme0n1p3 /mnt/snap

# 方式2：如果已挂载根文件系统，可用 bind 方式
sudo mount --bind /.snapshots/1/snapshot /mnt/snap
sudo mount -o remount,ro,bind /mnt/snap

# 用完记得卸载
sudo umount /mnt/snap
```

> 注意：`subvol=` 路径是相对于 Btrfs 顶层子卷的路径。  
> snapper 的快照路径带前缀 `/@/.snapshots/编号/snapshot`，与初版的手动快照路径不同。

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
sudo btrfs send /.snapshots/1/snapshot > /path/to/external/20260120_root.btrfs

# 或直接管道到 receive，节省本地空间
sudo btrfs send /.snapshots/1/snapshot | sudo btrfs receive /mnt/backup
```

### 增量备份（后续）

```bash
# snapper 会定期生成新快照，直接拿两个相邻的做增量
sudo btrfs send -p /.snapshots/1/snapshot \
     /.snapshots/2/snapshot \
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

# 重命名
sudo mv /mnt/20260120-1427_root /mnt/@
```

---

## 维护-检查-优化

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

> 单盘无冗余时，scrub 只能**发现**错误，不能修复。

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
> 本版使用 nodatacow 的子卷（`/var/log`、`/var/cache`、`/var/tmp`、`/kvm`）不要 defragment。

设置子卷/文件压缩属性：

```bash
# 查看属性
sudo btrfs property get /path

# 设置压缩
sudo btrfs property set /path compression zstd
```

挂载选项压缩：

```fstab
compress=zstd:1
```

### 只读属性

```bash
# 设为只读
sudo btrfs property set /.snapshots/1/snapshot ro true

# 取消只读
sudo btrfs property set /.snapshots/1/snapshot ro false
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
| 列出子卷（表格） | `sudo btrfs subvolume list -t /` |
| 创建子卷 | `sudo btrfs subvolume create /mnt/@` |
| 创建只读快照 | `sudo btrfs subvolume snapshot -r / /.snapshots/日期_root` |
| 删除快照 | `sudo btrfs subvolume delete /.snapshots/日期_root` |
| 查看子卷详情 | `sudo btrfs subvolume show /.snapshots/日期_root` |
| 挂载快照 | `sudo mount -t btrfs -o subvol=/@/.snapshots/1/snapshot,ro /dev/nvme0n1p3 /mnt/snap` |
| 完整发送 | `sudo btrfs send /.snapshots/snap1 \| sudo btrfs receive /mnt/backup` |
| 增量发送 | `sudo btrfs send -p /.snapshots/snap1 /.snapshots/snap2 \| sudo btrfs receive /mnt/backup` |
| scrub | `sudo btrfs scrub start -B /` |
| balance | `sudo btrfs balance start -dusage=50 /` |
| 压缩整理 | `sudo btrfs filesystem defragment -r -czstd /` |
| 查看属性 | `sudo btrfs property get /path` |
| 设置只读 | `sudo btrfs property set /path ro true` |
| 设置默认子卷 | `sudo btrfs subvolume set-default <subvolid> /` |
| **snapper 列表** | `sudo snapper -c root list` |
| **snapper 手动快照** | `sudo snapper -c root create --description "..."` |
| **snapper 回滚** | `sudo snapper -c root rollback <编号>` |
| **zram 状态** | `zramctl` / `cat /proc/swaps` |

---

## 自动化工具推荐

| 工具 | 用途 |
|---|---|
| `snapper` | openSUSE 系快照管理，支持保留策略；本版实际使用 |
| `snap-pac` | Arch/CachyOS 下配合 pacman，自动在安装/升级前后创建快照；本版实际使用 |
| `btrbk` | 强大的 Btrfs 快照与 `send/receive` 备份工具 |
| `timeshift` | 图形化、简单易用，适合桌面用户 |
| `grub-btrfs` | 在 GRUB 菜单中直接启动到 Btrfs 快照 |
| `btrfs-assistant` | 图形化管理 Btrfs、snapper、grub-btrfs 等 |
| `zram-generator` | 自动创建 zram 块设备，CachyOS 默认 |

建议：

- 初学者优先使用自动化工具，减少手误。
- 自动工具也要定期验证快照能否正常挂载和读取。
- 生产环境建议结合 `btrbk` 做异地增量备份。

---

## 最佳实践与注意事项

- 快照尽量使用只读：`btrfs subvolume snapshot -r` 或让 snapper 管理。
- 快照不是备份，重要数据遵循 3-2-1 原则：3 份副本、2 种介质、1 份异地。
- **`.snapshots` 由 snapper 作为嵌套子卷自动创建，不写 fstab**。
- **`/var/log`、`/var/cache`、`/var/tmp`、`/kvm` 使用 `nodatacow`**，避免 CoW 写放大。
- **`/tmp` 挂 tmpfs，`/var/tmp` 保持 btrfs**（跨重启保留语义）。
- **zram 优先级高于磁盘 swap**，用 `cat /proc/swaps` 确认。
- 命名统一：`YYYYMMDD-HHMM_用途`，便于排序和清理。
- 系统升级、内核更新、重大配置变更前先创建快照（snap-pac 自动做）。
- 定期 `scrub`，尤其是 RAID 环境。
- 监控空间：`btrfs filesystem usage -h /`。
- 谨慎使用 `defragment`，它可能破坏 CoW 共享，增加快照占用；nodatacow 子卷不 defrag。
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
sudo btrfs property get /.snapshots/1/snapshot
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

- `subvol=` 路径是否正确（snapper 是 `/@/.snapshots/N/snapshot`）。
- `.snapshots` 是否为独立或嵌套子卷。
- 设备 UUID 是否正确。
- 是否只读挂载与当前状态冲突。

```bash
lsblk -pf
sudo btrfs subvolume list -t /mnt
```

### 无法删除子卷

可能原因：

- 子卷内还有嵌套子卷（例如 `@/.snapshots`、`@home/.snapshots`）。
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

- 删除不需要的快照（`snapper cleanup`）。
- 清理日志、缓存、包缓存。
- 适度 `balance`。
- 检查是否有异常大文件或快照保留过多。

### scrub 报错

- 检查硬件健康：`smartctl`。
- RAID1/RAID10 下可尝试修复。
- 单盘无冗余时，scrub 只能发现错误，不能修复。
- 重要数据立即从备份恢复。

### zram 与磁盘 swap 优先级反了

症状：系统频繁写磁盘 swap，而不是先压缩进 zram。

```bash
cat /proc/swaps
# 确认 zram0 的 Priority 高于磁盘 swap
```

修复：调整 `zram-generator.conf` 里的 `zram-size` 或在 fstab 给磁盘 swap 加 `pri=` 选项。

---

## 附录-快照脚本示例

> 如果你用 snapper + snap-pac，本节可跳过；下面适合不想引入 snapper 的场景。

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
- Arch Wiki：Btrfs、Snapper、Zram
- CachyOS Wiki：Btrfs、snapper、zram-generator
- `man btrfs`、`man btrfs-subvolume`、`man btrfs-send`、`man btrfs-receive`
- `snapper`、`btrbk`、`timeshift`、`grub-btrfs` 官方文档

---

希望这份 CachyOS 实测修订版对你有帮助！主要改动集中在**子卷布局、fstab、snapper 嵌套快照、zram/tmpfs、nodatacow 策略**五块，其余命令和概念可以沿用初版。欢迎补充/纠错～

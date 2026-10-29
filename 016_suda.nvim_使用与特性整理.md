# suda.nvim 使用与特性整理

> [!NOTE]
> By WuDiXianXin

> [!IMPORTANT]
> 无需 sudo nvim 启动 Neovim，也能安全读写需要 root 权限的文件。

> [!TIP]
> **[WuDiXianXin/suda.nvim](https://github.com/WuDiXianXin/suda.nvim)** 是 [lambdalisue/vim-suda](https://github.com/lambdalisue/vim-suda) 的 **Neovim 0.12+ 纯 Lua 移植版**，
> 保留 :SudaRead 、 :SudaWrite 、 suda:// 协议，
> 同时做了更安全的实现：不走 shell、密码走 stdin、拒绝设备文件、自动处理换行符和编码，
> 并支持 sudo / sudo-rs / doas。
>
> 仅支持 `lua` 配置，不支持 `Vim` 配置。
> 支持 `Linux`、`MacOS`、`类 Unix` 系统，不支持 `Windows` 系统。

---

## 目录

- [1. 概述与核心对比](#1-概述与核心对比)
- [2. 主要改进点](#2-主要改进点)
- [3. 局限与取舍](#3-局限与取舍)
- [4. 总结建议](#4-总结建议)
- [5. 换行符处理](#5-换行符处理)
- [6. 特殊文件与不可读路径处理](#6-特殊文件与不可读路径处理)
- [7. 其他优点](#7-其他优点)
- [8. 平台支持：macOS 与 Windows](#8-平台支持macos-与-windows)
- [9. `++enc`、`++bin` 与 `++ff` 说明](#9-encbin-与-ff-说明)

---

## 1. 概述与核心对比

### 核心对比

| 方面 | 原版 **vim-suda** | **WuDiXianXin/suda.nvim** |
|------|-------------------|---------------------------|
| **语言 / 平台** | Vimscript，支持 Vim + Neovim | 纯 Lua，**仅 Neovim 0.12+** |
| **命令 / 协议** | `:SudaRead` / `:SudaWrite`、`suda://` | 完全兼容，用法基本一致 |
| **Smart edit** | `g:suda_smart_edit` | 支持（`smart_edit = true`），默认关闭；实现更保守，也会在 `BufAdd` 时检查 |
| **提权工具** | 主要 `sudo`（Windows 需额外工具） | `sudo` / `sudo-rs` / `doas` / 自定义 wrapper，自动探测 |
| **Shell 使用** | 通过 system / 相关机制执行 | **完全不经过 shell**（argv 列表），文件名中的空格、引号、`%`、`#`、`$()` 等更安全 |
| **密码处理** | 传统方式 | 通过 stdin（`-S`）传递，不进入 argv / 环境变量 |
| **写入实现** | 原有方式 | Unix 上优先用 `dd` 覆盖（保留 inode、权限、符号链接）；`:w >>` 支持 `dd oflag=append` 或回退到 `tee -a` |
| **配置方式** | `g:suda#*` 全局变量 | `require('suda').setup({...})` + 兼容旧全局变量 |
| **额外功能** | 基础 | `:checkhealth suda`、完整 Lua API、测试套件、中英文 README、更详细的编码/BOM/`++bin` 支持 |
| **维护状态** | 成熟，约 800+ stars，仍有更新 | 新仓库（提交数很少），文档很完善，但生态与使用量还远小于原版 |

---

## 2. 主要改进点

相对原版，suda.nvim 的主要改进包括：

- **安全性更高**：不走 shell，密码只走 stdin，临时文件用私有 `0700` 目录。
- **现代 Neovim 体验**：纯 Lua、`setup()` 配置、健康检查、Lua API（`suda.read` / `write` / `system` 等）。
- **更好的提权兼容**：原生支持 `sudo-rs`、`doas` 和自定义命令。
- **写入细节**：更注重保留原文件元数据；对不可搜索父目录（如 `/root`）的处理更谨慎。
- **Smart edit** 行为更保守，减少误判。

---

## 3. 局限与取舍

- **仅支持 Neovim 0.12+**（要求较新），原版兼容旧 Neovim / Vim。
- 写入**不是原子**的（中断可能留下不完整文件），原版也有类似特性。
- 项目较新，社区验证、issue 反馈和实际使用案例远少于原版（原版已稳定多年）。
- 没有 Windows 的完整原生支持（原版也主要依赖外部工具）。

---

## 4. 总结建议

- 如果你已经在用 **Neovim 0.12+**，想要纯 Lua、更安全的实现、更好的 `doas` / `sudo-rs` 支持，以及现代配置风格，**WuDiXianXin/suda.nvim** 是原版的良好现代化替代，功能对等且接口兼容，迁移成本低。
- 如果还需要支持 Vim、旧版 Neovim，或者更看重成熟稳定性和社区生态，继续用原版 **vim-suda** 更稳妥。

两者目标相同（方便编辑需要 sudo 的文件），新版在安全与 Neovim 专属体验上更进一步，但成熟度还处于早期阶段。安装后建议跑一下 `:checkhealth suda` 确认环境。

---

## 5. 换行符处理

**suda.nvim** 对换行符（`fileformat` / end-of-line）的处理相当完整，基本对齐 Neovim 原生行为，并针对 sudo 场景做了专门处理。

### 5.1 检测逻辑

读取文件时会自动检测三种风格：

| 检测结果 | 对应字符 | 含义 |
|---------|----------|------|
| `unix`  | LF (`\n`) | Unix/Linux 风格 |
| `dos`   | CRLF (`\r\n`) | Windows 风格 |
| `mac`   | CR (`\r`) | 旧 Macintosh 风格 |

具体实现见 `lua/suda/encoding.lua`：

- **普通编码（UTF-8 / Latin-1 等）**  
  扫描文件内容，优先找 LF；如果前面紧跟 CR 则判为 `dos`，否则为 `unix`。若只有 CR 而无 LF，则判为 `mac`。

- **宽字符编码（UTF-16 / UTF-32）**  
  按 code unit（2 或 4 字节）扫描，同样区分 `0x0A`（LF）和 `0x0D`（CR），逻辑与上面一致。

检测结果会写回 buffer 的 `'fileformat'` 选项，并保留 `'endofline'`（文件是否以换行结尾）。

### 5.2 读取时的特殊处理

1. **`BufReadCmd` 场景**（打开 `suda://` 文件）  
   - 先检测出 `fileformat`，然后**在执行 `:read` 之前**把 `vim.bo.fileformat` 设为检测结果。  
   - 这样 Neovim 的 `:read` 能正确按 CRLF / LF 拆行。

2. **旧 Mac（纯 CR）文件的特殊转换**  
   Neovim 的 `:read` 即使设置了 `fileformat=mac`，也**不会**正确拆分纯 CR 换行。  
   因此 suda 会：
   - 把整个文件读进内存；
   - 把所有 CR 替换成 LF（宽字符编码则按 code unit 替换）；
   - 写到临时文件后再用 `:read` 读入。  
   同时把原始的 `fileformat=mac` 和 `endofline` 信息保留到 buffer。

3. **二进制模式**  
   使用 `++bin` 时会跳过编码和换行符检测/转换，原样读写。

### 5.3 写入时的行为

写入时直接使用当前 buffer 的 `'fileformat'` 和 `'endofline'` 设置：

- Neovim 在把 buffer 内容写到临时文件时，会按当前 `fileformat` 生成对应的换行符（LF / CRLF / CR）。
- 然后 suda 用 `dd`（或 `tee`）把临时文件内容以提权方式写回目标路径。

因此：

- 你打开一个 CRLF 文件 → buffer 变成 `fileformat=dos` → 保存后仍然是 CRLF。
- 你手动改成 `:set fileformat=unix` 再保存 → 会写成 LF。
- 同样支持 `mac` 风格的写出。

### 5.4 与原版 vim-suda 的对比

原版 vim-suda 主要依赖 Vim 自身的读写机制，对换行符的处理相对“被动”。  
suda.nvim 则主动做了：

- 显式检测并设置 `fileformat`；
- 对旧 Mac CR 换行做预处理；
- 对 UTF-16/32 的换行检测单独实现（避免 NUL 字节问题）；
- 完整保留 `endofline` 状态。

整体上比原版更严谨，尤其是在处理跨平台或旧格式文件时更不容易出错。

---

## 6. 特殊文件与不可读路径处理

**suda.nvim** 对虚拟设备文件、特殊文件和不可读路径的处理比较谨慎，核心原则是：

> **只操作普通文件（regular file）及其符号链接，明确拒绝其他类型。**

### 6.1 虚拟设备 / 特殊文件：明确拒绝

官方文档和代码都写得很清楚：

> Only regular files and symlinks to them are read or written. FIFOs, sockets, and device files are rejected.

具体行为：

| 文件类型 | 处理结果 |
|---------|----------|
| 普通文件 (`file`) | 正常读写 |
| 指向普通文件的符号链接 | 跟随后按普通文件处理 |
| 目录 (`directory`) | 报错：`[suda] is a directory: ...` |
| 字符/块设备（`/dev/null`、`/dev/zero`、磁盘设备等） | 报错：`[suda] not a regular file: ...` |
| FIFO / 命名管道 | 同上，拒绝 |
| Socket | 同上，拒绝 |

检测流程（`path.lua`）：

1. 先用 `vim.uv.fs_lstat()` 看本地能看到的类型。
2. 如果当前用户因权限无法 `lstat`（例如在 `/root` 下），会尝试用**提权后的 `test -e` / `test -f`** 做探测（需要系统有 `test` 命令）。
3. 探测结果为 `other`（非普通文件）或明确是设备/FIFO 等，直接报错拒绝。

这样可以避免对 `/dev/sda`、`/dev/random`、管道等做危险的读写操作。

### 6.2 权限导致的不可读 / 不可搜索路径

#### 情况 A：文件本身存在，但当前用户不可读

- 用 `cat` 通过 sudo 读取内容到临时文件，再交给 Neovim。
- 正常检测编码、换行符等，体验接近普通文件。

#### 情况 B：父目录不可搜索（典型如 `/root/xxx`）

- 当前用户无法 `lstat`，也分不清“文件不存在”还是“没有权限”。
- 插件**不会**直接当成新文件打开空 buffer（避免误创建）。
- 会尝试用提权 `test` 探测：
  - 探测到是普通文件 → 允许继续读写；
  - 探测到不存在 → 报错 `file not found`；
  - 探测到是其他类型 → 报错 `not a regular file`；
  - 没有 `test` 或探测失败 → 保守处理，不擅自打开空 buffer。

文档明确说明：

> If an ancestor directory is not searchable (for example `/root`), suda cannot tell a missing file from an inaccessible one and does not silently open an empty buffer; use `:SudaWrite` to create a file in such a directory.

#### 情况 C：文件确实不存在

- 父目录可搜索时：打开空 buffer，触发 `BufNewFile`，第一次成功写入时创建文件。
- 父目录不可搜索时：不会静默打开空 buffer，需要显式用 `:SudaWrite` 创建。

### 6.3 写入时的额外保护

写入路径也会做同样的类型检查。如果探测到是设备/FIFO 等，会在真正执行 `dd`/`tee` 前就报错，避免把内容写进设备节点。

另外：

- 已存在文件用 `dd` 覆盖（保留 inode、权限、符号链接）。
- 新建文件由 root 创建，权限受 root 的 umask 影响。

### 6.4 已知局限

- **探测与实际操作是分开的**：`test` 探测和后续 `cat`/`dd` 之间有时间差，路径可能被其他人修改。文档建议对可被他人写入的目录设置 `timeout`。
- 系统没有 `test` 时，对不可搜索目录下的特殊文件检测能力会下降。
- 不支持把内容直接写到设备文件或管道（设计上就是拒绝）。

### 6.5 行为总结

| 场景 | 行为 |
|------|------|
| 普通 root 文件（如 `/etc/hosts`） | 正常 sudo 读写 |
| 设备文件 / FIFO / Socket | **明确拒绝**，报错 |
| `/root` 下不可见的普通文件 | 用提权 `test` 探测后允许操作 |
| `/root` 下不可见的特殊文件 | 探测到后拒绝 |
| 父目录不可搜索且无法判断是否存在 | 不静默开空 buffer，需手动 `:SudaWrite` |
| 权限不足的普通文件 | 通过 `cat`/`dd` + sudo 正常处理 |

整体策略是**保守且明确**：只碰普通文件，对虚拟设备和特殊文件直接拒绝，对权限模糊的路径尽量用提权探测而不是猜测。这比很多简单 sudo 插件更安全。

---

## 7. 其他优点

除了换行符处理和特殊文件防护，**suda.nvim** 还有不少值得一提的优点：

### 7.1 安全性设计更彻底

- **完全不走 shell**：所有命令用 argv 列表执行，文件名里的空格、引号、`%`、`#`、`$()`、管道符等都不会被解释。
- **密码只走 stdin**（`sudo -S`），不会出现在命令行参数、环境变量或日志里。
- 临时文件放在私有 `0700` 目录，用完即删。
- 对设备文件、FIFO、Socket 等明确拒绝，降低误操作风险。

### 7.2 现代 Neovim 体验

- **纯 Lua** 实现，结构清晰，和 folke 系插件风格接近。
- 提供完整 **Lua API**（`suda.read` / `write` / `append` / `system` / `systemlist` 等），方便脚本和插件集成。
- 支持 `:checkhealth suda`，能检查可执行文件、flavor、缓存凭证、`dd`/`cat`/`tee`/`test` 等环境。
- 配置用 `setup()`，同时兼容旧的 `g:suda#*` 变量，迁移成本低。

### 7.3 提权工具支持更广

- 自动识别并支持 **sudo**、**sudo-rs**、**doas**。
- 也可指定自定义 wrapper（`flavor = "custom"`）。
- 对 passwordless / 缓存凭证有明确开关（`noninteractive`）。

### 7.4 读写细节更到位

- 写入优先用 `dd` 覆盖**同一个 inode**，尽量保留原文件的 owner、权限和符号链接。
- 追加（`:w >>`）优先 `dd oflag=append`，不行再回退到 `tee -a`。
- 自动检测并保留 `'bomb'`、`'fileencoding'`、`'fileformat'`、`'endofline'`。
- 对 UTF-16/UTF-32 的 BOM 和换行有专门处理。
- 触发完整的 `BufReadPre/Post`、`BufWritePre/Post`、`FileAppendPre/Post` 等事件，和普通文件行为一致。

### 7.5 Smart edit 更保守、更可靠

- 默认关闭，开启后只在真正不可读/不可写时切换到 `suda://`。
- 不会误伤可写目录下的新文件，也不会动已经修改过的 buffer。
- 针对 Neovim 在打开不可读文件时不触发 `BufEnter` 的问题，额外用 `BufAdd` 做了兜底检查。

### 7.6 工程与可维护性

- 有独立的 **测试套件**（用假 sudo 模拟权限，可在普通用户下跑）。
- 文档齐全（英文 + 中文 README，完整 `:help suda`）。
- 代码按模块拆分（`encoding`、`path`、`io`、`runner`、`buffer` 等），可读性较好。
- 明确声明限制（非原子写入、大文件占内存、仅 Neovim 0.12+ 等），预期管理比较清晰。

### 7.7 与原版 vim-suda 的兼容性

- 命令（`:SudaRead` / `:SudaWrite`）、`suda://` 协议、大部分配置变量都保持兼容，从原版迁过来基本不用改习惯。

**简单总结**：它在「安全、现代 Neovim 体验、细节正确性」这三块比原版和多数同类插件更用心。适合已经用上 Neovim 0.12+、希望插件行为可预期、又在意安全细节的人。唯一需要注意的是项目还比较新，社区使用量和长期验证不如原版 vim-suda，实际踩坑反馈会相对少一些。

---

## 8. 平台支持：macOS 与 Windows

**结论先说：**

- **macOS**：可以正常使用，几乎和 Linux 一样。
- **Windows**：**不支持**，目前基本不可用（依赖 Unix 工具链）。

### 8.1 macOS 使用方法

macOS 是 Unix-like 系统，自带 `sudo`、`dd`、`cat`、`tee`、`test`，完全满足要求。

#### 1. 安装插件

用 `lazy.nvim`（推荐）：

```lua
-- ~/.config/nvim/lua/plugins/suda.lua
return {
  {
    'WuDiXianXin/suda.nvim',
    lazy = false,
    config = function()
      require('suda').setup({
        smart_edit = true,   -- 可选：自动处理无写文件
        -- executable = 'sudo',  -- 默认即可
      })
    end,
  },
}
```

或用 Neovim 0.12+ 自带的 `vim.pack`：

```lua
vim.pack.add({ 'https://github.com/WuDiXianXin/suda.nvim' })
require('suda').setup({ smart_edit = true })
```

#### 2. 验证环境

启动 Neovim 后执行：

```vim
:checkhealth suda
```

正常应看到：

- Neovim ≥ 0.12
- `sudo` / `dd` / `cat` 找到
- 可选的 `tee`、`test` 提示

#### 3. 日常用法（和 Linux 相同）

```vim
:SudaRead /etc/hosts
:SudaWrite
:e suda:///etc/sudoers
:w suda:///path/to/protected/file
```

启用 `smart_edit = true` 后，直接 `:e /etc/hosts` 也会在需要时自动切到 `suda://`。

#### 4. macOS 小提示

- 首次使用会弹密码框（`inputsecret`），和系统 `sudo` 一样。
- 若想减少反复输密码，可在 `/etc/sudoers`（用 `visudo`）配置较短的 `timestamp_timeout`，或（按需）加 `Defaults timestamp_type = global`。
- Apple Silicon / Intel 都适用，无额外依赖。

### 8.2 Windows 情况

**WuDiXianXin/suda.nvim 目前没有官方 Windows 支持。**

原因：

1. 读写依赖 Unix 工具：`dd`、`cat`、`tee`、`test`。
2. 健康检查会强制检查这些可执行文件。
3. 文档和实现都明确写的是 “Unix-like systems”。
4. 原版 `vim-suda` 在 Windows 上需要额外装 `mattn/sudo` 或 `gsudo`，而这个 Lua 移植版没有实现对应的 Windows 路径。

即使你装了 [gsudo](https://github.com/gerardog/gsudo) 或 [mattn/sudo](https://github.com/mattn/sudo)，并把 `executable` 设成它们，插件内部仍会调用 `dd`/`cat`，大概率直接失败。

#### Windows 上的替代方案

| 方案 | 说明 |
|------|------|
| **原版 vim-suda** | 仍支持有限 Windows（配合 gsudo / mattn/sudo） |
| **gnsfujiwara/suda.nvim** | 另一款 Lua 实现，文档明确提到 Windows + gsudo |
| **denialofsandwich/sudo.nvim** | 也有 Windows 相关实践 |
| 直接用管理员权限开 Neovim | 最简单粗暴，但不推荐日常使用 |

如果必须在 Windows 上用类似功能，更现实的选择是：

```lua
-- 示例：改用 gnsfujiwara/suda.nvim（需自行验证）
{
  "gnsfujiwara/suda.nvim",
  opts = {
    -- 按该项目文档配置 gsudo
  },
}
```

或继续用成熟的 `lambdalisue/vim-suda` + gsudo。

### 8.3 快速对照

| 平台 | 支持情况 | 额外依赖 | 推荐配置 |
|------|----------|----------|----------|
| **macOS** | ✅ 完整支持 | 系统自带即可 | `smart_edit = true` |
| **Linux** | ✅ 完整支持 | 系统自带即可 | 同上 |
| **Windows** | ❌ 不支持 | 即使装 gsudo 也难用 | 换其他插件或原版 vim-suda |

**总结**：在 Mac 上直接按 Linux 方式安装使用即可；在 Windows 上目前不建议用这个插件，换支持 Windows 的替代品更省事。

---

## 9. `++enc`、`++bin` 与 `++ff` 说明

在 Vim/Neovim 和 suda.nvim 里常见的是 **`++enc`**（encoding）和 **`++bin`**（binary）。下面分开说明。

### 9.1 `++enc`：指定文件编码

全称是 **`++enc={encoding}`**，用来告诉 Neovim：**按哪种字符编码去读/写这个文件**。

常见例子：

```vim
:edit ++enc=utf-8 suda:///path/to/file
:edit ++enc=latin1 /etc/some-old-file
:edit ++enc=utf-16le C:\path\file.txt
:edit ++enc=gbk some-chinese-file.txt
```

含义：

- 不写时：Neovim / suda 会自己猜（BOM、UTF-8、Latin-1 等）。
- 写了 `++enc=...`：以你指定的编码为准，覆盖自动检测。

在 suda 里也适用，例如：

```vim
:edit ++enc=latin1 suda:///path/to/file
```

### 9.2 `++bin`：二进制模式

全称是 **`++bin`**，表示 **按二进制方式打开**，尽量不做“文本处理”。

```vim
:edit ++bin suda:///path/to/file
:edit ++bin /usr/bin/some-program
```

开启后大致会：

| 行为 | 说明 |
|------|------|
| 不转换换行符 | 不把 CRLF/CR 当成行尾去“规范化” |
| 弱化/跳过编码转换 | 少做字符集转换，更接近原始字节 |
| 常配合 `binary` 选项 | 避免把文件当纯文本乱改 |

适合：可执行文件、图片、压缩包、带任意字节的数据文件等。  
suda 文档也写了：用 `++bin` 时会跳过编码和换行符的自动转换。

### 9.3 和 `++ff` 的关系

还有一个常一起出现的是 **`++ff`**（fileformat，换行风格）：

```vim
:edit ++ff=dos  file   " CRLF
:edit ++ff=unix file   " LF
:edit ++ff=mac  file   " 旧 Mac 的 CR
```

- `++enc` → 字符编码（UTF-8、GBK、Latin-1…）
- `++ff`  → 换行符风格（dos/unix/mac）
- `++bin` → 整文件当二进制，少做上述文本转换

### 9.4 小结

| 写法 | 含义 |
|------|------|
| `++enc=utf-8` | 按 UTF-8 读写 |
| `++enc=latin1` | 按 Latin-1 读写 |
| `++bin` | 二进制模式，不做编码/换行转换 |
| `++ff=dos` | 按 CRLF 处理换行 |

在 suda 里用法和普通 `:edit` / `:read` 一样，只是路径可以写成 `suda://...`。

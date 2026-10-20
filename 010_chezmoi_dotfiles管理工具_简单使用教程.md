# chezmoi 简单使用教程

> [!NOTE]
> By WuDiXianXin

`chezmoi` 是一个基于 Git 的跨平台 dotfiles 管理工具。它的核心设计是**复制而非符号链接**，并通过强大的模板功能处理不同机器之间的配置差异。

## 目录

- [核心概念](#核心概念)
- [安装与初始化](#安装与初始化)
- [日常核心工作流](#日常核心工作流)
- [文件命名与映射规则](#文件命名与映射规则)
- [模板-处理机器差异](#模板-处理机器差异)
- [安全管理机密信息](#安全管理机密信息)
- [脚本与自动化](#脚本与自动化)
- [常用命令速查](#常用命令速查)
- [进阶技巧与最佳实践](#进阶技巧与最佳实践)
- [常见问题](#常见问题)
- [总结](#总结)

---

## 核心概念

理解 chezmoi 的关键在于区分两个目录：

| 目录 | 默认位置 | 作用 |
| :--- | :--- | :--- |
| **源目录 Source Directory** | `~/.local/share/chezmoi` | dotfiles 的“真实来源”，所有配置文件以特殊命名形式存储在这里，并受 Git 版本控制 |
| **目标目录 Target Directory** | `~`，即 Home 目录 | 配置文件实际生效的地方 |

工作流程：

```text
在源目录中编辑
      ↓
chezmoi apply 同步到目标目录
      ↓
使用 Git 提交并推送到远程仓库
```

> [!TIP]
> 源目录可以自定义。

`~/.config/chezmoi/chezmoi.toml`

```toml
sourceDir = "~/git/xx/dotfiles"

[data]
    theme = "catppuccin"

[edit]
    command = "nvim"
    hardlink = false
```

注意：`sourceDir`、`encryption` 这类顶层配置项应放在所有 `[section]` 之前。

---

## 安装与初始化

### 安装 chezmoi

根据自己的操作系统选择安装方式：

- **macOS（Homebrew）**

  ```bash
  brew install chezmoi
  ```

- **Linux（通用脚本）**

  ```bash
  sh -c "$(curl -fsLS get.chezmoi.io)"
  ```

- **Linux（包管理器）**

  ```bash
  # Debian / Ubuntu
  apt-get install chezmoi

  # Fedora
  dnf install chezmoi

  # Arch
  # pacman -S chezmoi
  paru -S chezmoi
  ```

- **Windows（Chocolatey）**

  ```powershell
  choco install chezmoi
  ```

### 初始化

在**第一台机器**上，执行以下命令创建源目录并准备 Git 仓库：

```bash
chezmoi init
```

如果已经有一个远程 dotfiles 仓库，可以直接初始化并应用：

```bash
chezmoi init --apply 你的GitHub用户名
```

或者指定完整仓库地址：

```bash
chezmoi init --apply git@github.com:你的用户名/dotfiles.git
```

如果你需要应用非默认分支，可以使用 --branch 参数明确指定：

```bash
chezmoi init --apply --branch 你的分支名 你的GitHub用户名
```

例如，你想应用 dev 分支：

```bash
chezmoi init --apply --branch dev 你的GitHub用户名
```

---

## 日常核心工作流

以下是管理 dotfiles 最常用的命令。

### 1. 添加文件到管理

将 Home 目录下的现有配置文件纳入 chezmoi 管理：

```bash
chezmoi add ~/.bashrc
```

这会将 `~/.bashrc` 复制到源目录，并命名为 `dot_bashrc`。

### 2. 编辑配置文件

**永远不要直接编辑 Home 目录下的文件。**  
使用 `chezmoi edit` 编辑源目录中的副本：

```bash
chezmoi edit ~/.bashrc
```

编辑后自动应用变更：

```bash
chezmoi edit --apply ~/.bashrc
```

`--apply` 参数会在保存并退出编辑器后，自动将变更同步到 Home 目录。

### 3. 查看差异与应用变更

```bash
# 预览 chezmoi apply 将会做的所有更改，提交前必备
chezmoi diff

# 将源目录配置同步到 Home 目录
chezmoi apply
```

建议先 `diff`，确认无误后再 `apply`。

### 4. 与 Git 仓库同步

chezmoi 底层使用 Git，可以直接用 Git 命令管理版本：

```bash
chezmoi cd                 # 进入源目录的 shell

git add .
git commit -m "Update zsh config"

# 首次需要添加远程仓库
git remote add origin git@github.com:你的用户名/dotfiles.git
git push -u origin main
```

退出 shell 后，dotfiles 就备份到了远程仓库。

---

## 文件命名与映射规则

chezmoi 通过文件名前缀决定目标文件的位置和属性。

| 前缀 | 作用 | 示例 |
| :--- | :--- | :--- |
| `dot_` | 替换为 `.`，用于隐藏文件 | `dot_bashrc` → `~/.bashrc` |
| `private_` | 设置文件权限为 `0600`，仅所有者可读写 | `private_dot_ssh/config` |
| `executable_` | 设置可执行权限 | `executable_dot_local/bin/myscript` |
| `exact_` | 精确管理目录，删除目标目录中多余的文件 | `exact_dot_config/nvim` |
| `encrypted_` | 加密文件 | `encrypted_private_dot_ssh/config` |
| `run_` | 每次 `apply` 都执行的脚本 | `run_install-packages.sh` |
| `run_once_` | 只执行一次的脚本 | `run_once_install-packages.sh` |
| `run_onchange_` | 内容变化时执行的脚本 | `run_onchange_reload-service.sh` |
| `symlink_` | 创建符号链接 | `symlink_dot_vimrc` |
| `literal_` | 不解析模板，按字面量处理 | `literal_dot_file` |

---

## 模板-处理机器差异

模板是 chezmoi 的精华，它允许你根据不同的机器，如操作系统、主机名等，生成不同的配置内容。

### 1. 创建模板

在添加文件时使用 `--template` 标志：

```bash
chezmoi add --template ~/.gitconfig
```

这会将文件添加为 `dot_gitconfig.tmpl`。

### 2. 编辑模板

模板使用 Go 的 `text/template` 语法。例如，可以区分不同系统：

```gotemplate
{{ if eq .chezmoi.os "darwin" }}
# macOS 特定配置
{{ else if eq .chezmoi.os "linux" }}
# Linux 特定配置
{{ end }}
```

chezmoi 会自动提供以下常用变量：

| 变量 | 说明 |
| :--- | :--- |
| `.chezmoi.os` | 操作系统，如 `darwin`、`linux`、`windows` |
| `.chezmoi.arch` | 架构，如 `amd64`、`arm64` |
| `.chezmoi.hostname` | 主机名 |
| `.chezmoi.username` | 用户名 |
| `.chezmoi.homeDir` | Home 目录 |
| `.chezmoi.sourceDir` | 源目录 |

自定义数据可以在 `chezmoi.toml` 中定义：

```toml
[data]
    theme = "catppuccin"
```

模板中使用：

```gotemplate
theme = "{{ .theme }}"
```

### 3. 调试模板

使用 `chezmoi execute-template` 可以快速测试模板片段：

```bash
chezmoi execute-template '{{ .chezmoi.hostname }}'
```

查看所有可用模板数据：

```bash
chezmoi data
```

查看某个目标文件渲染后的内容：

```bash
chezmoi cat ~/.gitconfig
```

---

## 安全管理机密信息

对于包含密码、API Key 的配置文件，**绝对不要**以明文形式提交到 Git 仓库。chezmoi 提供了多种方案。

### 集成密码管理器

chezmoi 可以直接从 1Password、Bitwarden、KeePassXC 等密码管理器中拉取机密值。

例如，在模板中引用 1Password：

```gotemplate
export CF_API_TOKEN='{{ onepasswordRead "op://Personal/cloudflare-api-token/password" }}'
```

这样，仓库里只有引用，真正的机密保存在密码管理器中。

### 文件加密

chezmoi 内置对 `age` 和 `gpg` 加密的支持。

使用 `chezmoi add --encrypt ~/.ssh/config` 添加加密文件时，它会在源目录中存储加密后的版本，但在 `apply` 时自动解密到目标位置。

如果直接使用 `--encrypt` 报错，通常是因为还没有在 chezmoi 配置文件中明确指定加密方式，即 `age` 或 `gpg`。

---

### 方案一：使用 `age` 加密（推荐）

`age` 是一款现代、简洁的加密工具，chezmoi 内置了对它的支持，配置方便。

#### 1. 用 chezmoi 生成 age 密钥对（无需单独安装 age）

```bash
mkdir -p ~/.config/chezmoi
chezmoi age-keygen --output ~/.config/chezmoi/key.txt
chmod 600 ~/.config/chezmoi/key.txt
```

命令会输出一个公钥，例如：

```text
Public key: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
```

`~/.config/chezmoi/key.txt` 中也会存在这个公钥提示。

#### 2. 编辑 chezmoi 配置文件

创建或编辑 `~/.config/chezmoi/chezmoi.toml`：

```toml
# 注意：encryption 必须放在文件最顶部，在任何 [section] 之前
encryption = "age"

[age]
identity = "~/.config/chezmoi/key.txt"
recipient = "age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p"
```

请将 `recipient` 替换为你生成的公钥。

> **重要**：`encryption` 配置项必须位于文件顶部，在任何 `[section]` 定义之前，否则 chezmoi 可能无法正确读取。

#### 3. 使用加密文件

```bash
chezmoi add --encrypt ~/.ssh/config
chezmoi diff
chezmoi apply
```

#### 4. 验证加密是否生效

```bash
# 查看源目录中的对应文件路径
chezmoi source-path ~/.ssh/config

# 查看解密后的目标内容
chezmoi cat ~/.ssh/config
```

如果源目录中存储的是加密内容，而 `chezmoi cat` 能显示明文，说明配置生效。

---

### 方案二：使用 `gpg` 加密

如果你已经拥有 GPG 密钥对，并希望继续使用它，可以这样配置。

编辑 `~/.config/chezmoi/chezmoi.toml`：

```toml
# 同样，encryption 必须在最顶部
encryption = "gpg"

[gpg]
recipient = "your-gpg-key-id-or-email"
```

将 `recipient` 替换为你的 GPG 收件人标识，例如邮箱或 Key ID。

然后同样使用：

```bash
chezmoi add --encrypt ~/.ssh/config
```

---

## 脚本与自动化

对于安装软件包、配置系统等**非文件操作**，可以使用 `run_` 前缀的脚本。这些脚本会在 `chezmoi apply` 时自动执行。

常见脚本类型：

| 脚本前缀 | 执行时机 |
| :--- | :--- |
| `run_` | 每次 `chezmoi apply` 都执行 |
| `run_once_` | 只执行一次 |
| `run_onchange_` | 脚本内容变化时执行 |

例如，创建一个 `run_once_install-packages.sh`：

```bash
#!/bin/sh
set -e

if command -v brew >/dev/null 2>&1; then
    brew install ripgrep fd fzf
elif command -v apt-get >/dev/null 2>&1; then
    sudo apt-get update
    sudo apt-get install -y ripgrep fd-find fzf
fi
```

它只会在首次 `apply` 时运行一次，用于安装 Homebrew 或 apt 包。

建议脚本尽量幂等，避免重复执行时出错。

---

## 常用命令速查

| 命令 | 说明 |
| :--- | :--- |
| `chezmoi init` | 初始化源目录 |
| `chezmoi init --apply <repo>` | 克隆远程仓库并应用配置 |
| `chezmoi add <file>` | 添加文件到管理 |
| `chezmoi add --template <file>` | 添加为模板 |
| `chezmoi add --encrypt <file>` | 添加为加密文件 |
| `chezmoi edit <file>` | 编辑源目录中的文件 |
| `chezmoi edit --apply <file>` | 编辑后自动应用 |
| `chezmoi diff` | 预览变更 |
| `chezmoi apply` | 应用变更到 Home 目录 |
| `chezmoi apply --dry-run` | 只预览，不实际应用 |
| `chezmoi update` | 拉取远程变更并应用 |
| `chezmoi cd` | 进入源目录 shell |
| `chezmoi source-path <file>` | 查看目标文件对应的源文件路径 |
| `chezmoi cat <file>` | 查看目标文件渲染后的内容 |
| `chezmoi managed` | 列出被管理的文件 |
| `chezmoi status` | 查看状态 |
| `chezmoi verify` | 验证目标目录是否与期望一致 |
| `chezmoi forget <file>` | 取消管理，但不删除目标文件 |
| `chezmoi destroy <file>` | 删除源文件和目标文件，慎用 |
| `chezmoi data` | 查看模板数据 |
| `chezmoi execute-template` | 测试模板片段 |
| `chezmoi doctor` | 检查 chezmoi 环境是否正常 |

---

## 进阶技巧与最佳实践

- **在新机器上快速恢复**

  使用一条命令完成安装 chezmoi、克隆仓库并应用所有配置：

  ```bash
  sh -c "$(curl -fsLS get.chezmoi.io)" -- init --apply 你的GitHub用户名
  ```

  如果仓库不是 GitHub 用户名形式，可以直接指定仓库地址：

  ```bash
  chezmoi init --apply git@github.com:你的用户名/dotfiles.git
  ```

- **定期更新配置**

  ```bash
  chezmoi update
  ```

  它会拉取远程仓库的最新变更，并自动应用到本机。

- **先预览后应用**

  养成先 `chezmoi diff`，再 `chezmoi apply` 的习惯，避免意外覆盖。

- **善用 `chezmoi cd`**

  需要批量操作源目录时，用 `chezmoi cd` 进入该目录的 shell 会更高效。

- **排除文件**

  创建一个 `.chezmoiignore` 文件，在其中列出你希望 chezmoi 忽略管理的文件或目录模式，避免误操作。

- **多机器差异化**

  可以结合模板、`[data]` 和 `.chezmoi.toml.tmpl` 实现新机器初始化时交互式填写数据。

- **备份加密私钥**

  对于 `age`，私钥文件 `~/.config/chezmoi/key.txt` 极其重要，**切勿**提交到 Git 仓库。请使用密码管理器或加密后另行备份。

- **不要直接编辑 Home 目录文件**

  始终通过 `chezmoi edit` 或修改源目录来管理配置。

- **定期运行 `chezmoi doctor`**

  它可以帮你检查环境、配置、加密等是否存在问题。

---

## 常见问题

### 1. 使用 `chezmoi add --encrypt` 报错

报错通常是因为在使用 `--encrypt` 标志前，需要先在 chezmoi 配置文件中明确指定加密方式：`age` 或 `gpg`。

解决方法：

- 使用 `age`：配置 `encryption = "age"` 和 `[age]`。
- 使用 `gpg`：配置 `encryption = "gpg"` 和 `[gpg]`。

### 2. `encryption` 配置不生效

`encryption` 必须放在 `chezmoi.toml` 文件顶部，在任何 `[section]` 之前。

正确示例：

```toml
encryption = "age"

[age]
identity = "~/.config/chezmoi/key.txt"
recipient = "age1..."
```

错误示例：

```toml
[age]
identity = "~/.config/chezmoi/key.txt"
recipient = "age1..."

encryption = "age"
```

### 3. 新机器上无法解密

在新机器上初始化时，需要先安全地获取 `key.txt` 文件，并放置到相同路径：

```text
~/.config/chezmoi/key.txt
```

然后 chezmoi 才能正常解密文件。

### 4. `chezmoi apply` 覆盖了本地修改

这是 chezmoi 的正常行为。建议每次应用前先执行：

```bash
chezmoi diff
```

确认差异后再：

```bash
chezmoi apply
```

---

## 总结

chezmoi 通过源目录、模板和加密功能的组合，为你提供了一套强大且安全的 dotfiles 管理方案。

开始使用时，建议先添加一两个核心配置文件，例如 `.bashrc` 或 `.gitconfig`，熟悉基本流程后，再逐步引入模板、加密和脚本自动化功能。

推荐工作流：

```text
chezmoi add / chezmoi edit
        ↓
chezmoi diff
        ↓
chezmoi apply
        ↓
chezmoi cd
        ↓
git add / commit / push
```

长期维护时，使用：

```bash
chezmoi update
```

即可拉取远程最新配置并应用到本机。

---

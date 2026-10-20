# Rust 安装与镜像源配置

> [!NOTE]
> By WuDiXianXin

> [!TIP]
> 本文覆盖 Rust 安装、rustup 镜像源、Cargo 镜像源、核心组件、VS Code 配置与常用扩展。
> 构建加速、sccache、cargo-chef 等内容见 `009_Rust_构建加速与缓存配置.md`。

## 一、镜像源配置（国内推荐 RsProxy）

### 1. rustup 镜像环境变量

Linux / macOS（bash / zsh）：

```bash
export RUSTUP_DIST_SERVER="https://rsproxy.cn"
export RUSTUP_UPDATE_ROOT="https://rsproxy.cn/rustup"
```

fish：

```fish
set -gx RUSTUP_DIST_SERVER "https://rsproxy.cn"
set -gx RUSTUP_UPDATE_ROOT "https://rsproxy.cn/rustup"
```

Windows PowerShell（当前会话）：

```powershell
$env:RUSTUP_DIST_SERVER = "https://rsproxy.cn"
$env:RUSTUP_UPDATE_ROOT = "https://rsproxy.cn/rustup"
```

Windows PowerShell（永久写入用户环境变量）：

```powershell
[Environment]::SetEnvironmentVariable("RUSTUP_DIST_SERVER", "https://rsproxy.cn", "User")
[Environment]::SetEnvironmentVariable("RUSTUP_UPDATE_ROOT", "https://rsproxy.cn/rustup", "User")
```

建议将 Linux / macOS 的环境变量写入对应 shell 配置文件：

- bash：`~/.bashrc`
- zsh：`~/.zshrc`
- fish：`~/.config/fish/config.fish`

### 2. Cargo 镜像源

配置文件位置：

- Linux / macOS：`~/.cargo/config.toml`
- Windows：`%USERPROFILE%\.cargo\config.toml`

内容如下：

```toml
[source.crates-io]
replace-with = 'rsproxy-sparse'

[source.rsproxy]
registry = "https://rsproxy.cn/crates.io-index"

[source.rsproxy-sparse]
registry = "sparse+https://rsproxy.cn/index/"

[registries.rsproxy]
index = "https://rsproxy.cn/crates.io-index"

[net]
git-fetch-with-cli = true
```

注意：

- 如果 `config.toml` 已存在，请合并配置，不要直接覆盖。
- `sparse+` 稀疏索引需要较新的 Cargo，通常通过 rustup 安装的稳定版 Cargo 已支持。
- 修改后建议新开终端，或重新加载 shell 配置。

## 二、安装 rustup

`rustup` 是 Rust 官方工具链管理器，负责安装、切换、更新 Rust 编译器及相关组件，支持 Linux / macOS / Windows。

### 1. Linux / macOS

国内推荐使用 RsProxy 安装脚本：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://rsproxy.cn/rustup-init.sh | sh
```

也可以使用官方脚本：

```bash
# 方法 1：通过 curl
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 方法 2：通过 wget
wget https://sh.rustup.rs -O - | sh
```

如果已经设置了 `RUSTUP_DIST_SERVER` 和 `RUSTUP_UPDATE_ROOT`，再执行官方脚本也会走镜像源。

### 2. Windows

访问 [rustup 官网](https://rustup.rs/) 下载 `rustup-init.exe`，运行后按提示安装。  
建议安装时勾选“添加到 PATH”。

国内网络环境下，也可以先设置 `RUSTUP_DIST_SERVER` 和 `RUSTUP_UPDATE_ROOT`，再运行安装程序。

### 3. 环境变量生效

Linux / macOS：

安装完成后，重启终端，或手动执行：

```bash
source $HOME/.cargo/env
```

Windows：

安装时若已勾选“添加到 PATH”，重启终端即可。  
否则需要手动将 `%USERPROFILE%\.cargo\bin` 添加到系统环境变量 PATH。

## 三、核心工具链

`rustup` 默认安装最新稳定版工具链，包含以下核心工具，无需额外操作：

- `rustc`：Rust 编译器，将源代码编译为可执行文件或库。
- `cargo`：包管理器，负责项目创建、构建、测试、依赖管理等。
- `rustdoc`：文档生成工具，`cargo doc` 可生成 HTML 文档。
- `rustfmt` / `cargo-fmt`：代码格式化工具，`cargo fmt` 自动调整代码风格。
- `clippy` / `cargo-clippy`：代码检查工具，`cargo clippy` 检测潜在错误和风格问题。

## 四、工具链版本管理

Rust 工具链分三种渠道，可按需切换：

```bash
rustup install stable   # 稳定版，默认，推荐生产环境
rustup install beta     # 测试版，提前体验新特性
rustup install nightly  # nightly 版，包含最新实验特性，适合开发工具

# 切换默认工具链
rustup default stable
rustup default nightly
```

注意：部分工具如 `miri` 依赖 nightly 工具链，生产项目不建议默认使用 nightly。

## 五、额外组件

以下组件需手动安装，部分依赖特定工具链，例如 `miri` 需要 nightly。

| 组件用途 | 安装命令 | 说明 |
| --- | --- | --- |
| 现代语言服务器 | `rustup component add rust-analyzer` | 为 IDE 提供补全、跳转等功能，替代 RLS |
| Rust 标准库源代码 | `rustup component add rust-src` | 提升 Rust 开发体验，尤其是 IDE 集成 |
| GDB 调试适配 | `rustup component add rust-gdb` | 配合 GDB 调试 Rust 程序 |
| LLDB 调试适配 | `rustup component add rust-lldb` | 配合 LLDB 调试 Rust 程序，macOS 推荐 |
| 内存安全调试器 | `rustup component add miri` | 需切换到 nightly 工具链 |

注意：

- `clippy` 和 `rustfmt` 在 stable 工具链中默认已安装，无需重复执行 `rustup component add`。
- 若安装失败，可检查网络，或确认镜像源配置是否生效。

## 六、VS Code 配置

除基础扩展外，推荐安装以下扩展：

```bash
# 核心扩展：Rust 语言支持，基于 rust-analyzer
code --install-extension rust-lang.rust-analyzer

# 调试扩展：支持 LLDB 调试，Windows / Linux / macOS 通用
code --install-extension vadimcn.vscode-lldb

# 依赖管理：显示 Cargo 依赖版本信息
code --install-extension serayuzgur.crates

# 代码提示：增强类型 / 文档提示
code --install-extension belfz.search-crates-io
```

如果 `code` 命令不可用，可以在 VS Code 扩展面板中搜索扩展名安装。

## 七、Cargo 子命令扩展工具

通过 `cargo install` 安装扩展工具。安装前需确保系统具备基础编译依赖，例如 Linux 的 `build-essential`、`libssl-dev`，macOS 的 Xcode Command Line Tools。

| 工具用途 | 安装命令 | 常用场景 |
| --- | --- | --- |
| Docker 构建优化 | `cargo install cargo-chef` | 加速 Rust 项目的 Docker 镜像构建 |
| Tauri 桌面应用开发 | `cargo install create-tauri-app tauri-cli` | 创建跨平台桌面应用 |
| 依赖管理便捷工具 | `cargo install cargo-edit` | 使用 `cargo add` 直接修改 `Cargo.toml` |
| 检查依赖更新 | `cargo install cargo-outdated` | 查看可更新的依赖 |
| 批量更新子命令 | `cargo install cargo-update` | 一键更新所有已安装的子命令 |

注意：

- 部分工具如 `tauri-cli` 可能需要额外系统库，参考 [Tauri 环境要求](https://tauri.app/)。
- `cargo-chef` 的详细用法见 `009_Rust_构建加速与缓存配置.md`。

## 八、验证与更新

### 验证安装

```bash
rustup --version
rustc --version
cargo --version
rust-analyzer --version
cargo clippy --version
```

### 更新工具

```bash
rustup update

# 更新单个 cargo 子命令，将 cargo-chef 替换为目标工具名
cargo install --force cargo-chef

# 批量更新所有 cargo 子命令，需先安装 cargo-update
cargo install-update -a
```

## 九、常见问题

1. **安装失败或速度慢**  
   检查网络连接，优先配置 RsProxy 或清华镜像等国内镜像源。

2. **命令未找到**  
   确认 `~/.cargo/bin` 已加入 PATH，或重启终端。Windows 检查 `%USERPROFILE%\.cargo\bin`。

3. **miri 无法运行**  
   切换到 nightly 工具链：

   ```bash
   rustup default nightly
   ```

4. **VS Code 无提示**  
   确保 `rust-analyzer` 扩展已启用，且项目根目录存在 `Cargo.toml`。

5. **Cargo 镜像源不生效**  
   检查 `~/.cargo/config.toml` 路径和 TOML 格式，修改后重新打开终端。

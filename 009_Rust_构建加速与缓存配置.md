# Rust 构建加速与缓存配置

> 本文覆盖 Cargo profile、增量编译、sccache、cargo-chef 与 CI / Docker 构建缓存。
> Rust 安装与镜像源配置见 `008_Rust_安装与镜像源配置.md`。

## 一、加速手段概览

| 手段 | 作用 | 适用场景 |
| --- | --- | --- |
| Cargo profile | 控制优化级别、调试信息、增量编译等 | 所有项目 |
| 增量编译 | 只重新编译发生变化的部分 | 本地开发 |
| sccache | 缓存编译结果，跨项目 / 跨 CI 复用 | 本地 / CI |
| cargo-chef | 缓存 Docker / CI 中的依赖层 | Docker / CI |
| LTO / codegen-units | 控制发布构建优化 | release |

本地开发优先关注：

- `incremental = true`
- `opt-level = 0`
- 合理设置 `codegen-units`
- 使用 `sccache`

CI / Docker 优先关注：

- `cargo-chef`
- `sccache`
- release profile 优化
- 缓存目录持久化

## 二、Cargo profile 配置

profile 配置通常写在项目或工作区根目录的 `Cargo.toml` 中。

### 1. 开发 profile

```toml
[profile.dev]
debug = true                # 保留调试信息
incremental = true          # 启用增量编译
codegen-units = 16          # 多单元并行编译，加快编译
opt-level = 0               # 不优化，加速编译
```

### 2. 发布 profile

```toml
[profile.release]
debug = false               # 移除调试信息
incremental = false         # 禁用增量编译，节省磁盘空间
codegen-units = 1           # 最大优化
opt-level = 3               # 最高优化级别
strip = true                # 剥离符号表
lto = "thin"                # 链接时优化，平衡编译时间和优化效果
```

如果追求更极致的体积和性能，可以将 LTO 改为：

```toml
lto = "fat"
```

说明：

- `lto = "thin"`：编译时间相对较短，优化效果较好，适合大多数 release 场景。
- `lto = "fat"`：优化更激进，编译时间更长，适合最终发布或对性能 / 体积要求较高的场景。
- `strip = true`：减小二进制体积，但会丢失调试符号。
- `incremental = false`：release 构建通常不需要增量编译，关闭可节省磁盘空间。

## 三、sccache 配置

`sccache` 是编译缓存工具，可以缓存 Rust 编译结果，减少重复编译时间。

### 1. 安装

```bash
cargo install sccache
```

Arch Linux 也可以使用包管理器：

```bash
# paru -S sccache
```

### 2. 启用 sccache

关键环境变量是 `RUSTC_WRAPPER`，必须设置为 `sccache`，否则 sccache 不会生效。

bash / zsh：

```bash
export RUSTC_WRAPPER=sccache
export SCCACHE_DIR="$HOME/.cache/sccache"
export SCCACHE_CACHE_SIZE="20G"
export SCCACHE_COMPRESSION=zstd
export SCCACHE_IDLE_TIMEOUT=3600

# 可选：禁用分布式存储
export SCCACHE_NO_DISTRIBUTED=1

# 可选：限制并发存储操作数
export SCCACHE_MAX_STORES=100
```

fish：

```fish
set -gx RUSTC_WRAPPER sccache
set -gx SCCACHE_DIR "$HOME/.cache/sccache"
set -gx SCCACHE_CACHE_SIZE 20G
set -gx SCCACHE_COMPRESSION zstd
set -gx SCCACHE_IDLE_TIMEOUT 3600

# 可选
set -gx SCCACHE_NO_DISTRIBUTED 1
set -gx SCCACHE_MAX_STORES 100
```

持久化位置：

- bash：`~/.bashrc`
- zsh：`~/.zshrc`
- fish：`~/.config/fish/config.fish`

### 3. 使用与验证

```bash
# 清除现有缓存，可选
sccache --clear

# 首次编译，会生成缓存
cargo build --release

# 再次编译，应该命中缓存
cargo clean
cargo build --release

# 查看缓存统计
sccache --show-stats
```

CI 环境中也可以显式启动和停止服务：

```bash
sccache --start-server
cargo build --release
sccache --stop-server
```

### 4. GitHub Actions 缓存

在 GitHub Actions 中可启用 sccache 的 GHA 缓存后端：

```yaml
env:
  RUSTC_WRAPPER: sccache
  SCCACHE_GHA_ENABLED: "true"
```

注意：

- `SCCACHE_GHA_ENABLED=true` 仅适用于 GitHub Actions + sccache GHA 后端。
- `set -e SCCACHE_GHA_ENABLED` 是删除变量，不是启用变量，不要用它来开启缓存。
- 如果本地不需要 GHA 缓存，不要设置该变量。

### 5. 注意事项

- sccache 对过程宏、部分 crate 或特殊构建脚本可能无法有效缓存。
- 如果发现没有命中缓存，先检查 `RUSTC_WRAPPER` 是否设置为 `sccache`。
- 使用 `cargo chef cook` 时，如果也希望使用 sccache，需要确保 `RUSTC_WRAPPER` 在该阶段同样生效。
- `SCCACHE_CACHE_SIZE` 根据磁盘空间调整，缓存过大时可执行 `sccache --clear`。

## 四、cargo-chef 使用

`cargo-chef` 主要用于 Docker / CI 中缓存依赖层，不是本地日常开发加速的必需品。  
本地开发优先使用增量编译和 `sccache`。

### 1. 安装

```bash
cargo install cargo-chef
```

### 2. 工作流

普通构建：

```bash
# 生成食谱文件
cargo chef prepare --recipe-path recipe.json

# 预编译依赖
cargo chef cook --recipe-path recipe.json

# 编译项目
cargo build
```

Release 构建：

```bash
# 生成食谱文件
cargo chef prepare --recipe-path recipe.json

# 预编译依赖
cargo chef cook --release --recipe-path recipe.json

# 正常编译项目
cargo build --release
```

说明：

- `cargo chef prepare` 会根据项目生成 `recipe.json`。
- `cargo chef cook` 会根据 `recipe.json` 编译依赖。
- 当源码变化但依赖未变化时，Docker 或 CI 可以复用依赖层缓存。

### 3. Dockerfile 示例

```dockerfile
FROM rust:1-slim AS chef

RUN cargo install cargo-chef
WORKDIR /app

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
RUN cargo chef cook --release --recipe-path recipe.json
COPY . .
RUN cargo build --release
```

注意：

- 根据项目需要安装系统依赖，例如 `pkg-config`、`libssl-dev`、`build-essential` 等。
- 如果同时使用 `sccache`，需要在 Docker 构建阶段设置 `RUSTC_WRAPPER=sccache`，并挂载或持久化缓存目录。

## 五、推荐组合

| 场景 | 推荐配置 |
| --- | --- |
| 本地开发 | `profile.dev` + `incremental = true` + `sccache` |
| CI 构建 | `sccache` + 缓存目录 + `profile.release` |
| Docker 构建 | `cargo-chef` + `sccache` + `profile.release` |
| 最终发布 | `profile.release` + `lto = "thin"` 或 `lto = "fat"` + `strip = true` |

## 六、常见问题

1. **sccache 不生效**  
   检查是否设置：

   ```bash
   export RUSTC_WRAPPER=sccache
   ```

   然后执行：

   ```bash
   sccache --show-stats
   cargo clean
   cargo build --release
   ```

2. **缓存目录太大**  
   调整 `SCCACHE_CACHE_SIZE`，或执行：

   ```bash
   sccache --clear
   ```

3. **编译反而更慢**  
   首次编译没有缓存时会较慢；`lto = "fat"`、`codegen-units = 1` 也会明显增加 release 编译时间。

4. **cargo-chef 本地没有明显加速**  
   `cargo-chef` 主要面向 Docker / CI 的依赖层缓存，本地直接使用 `cargo build` 即可。

5. **GitHub Actions 缓存未命中**  
   确认设置了 `SCCACHE_GHA_ENABLED=true`，并确保使用支持的后端。不要使用 `set -e SCCACHE_GHA_ENABLED` 来“启用”缓存。

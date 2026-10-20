# linux-wallpaperengine 安装、手动构建与使用

将 Wallpaper Engine 风格的动态壁纸带到 Linux！这个项目允许你直接在桌面上运行 Steam 的 Wallpaper Engine 动画壁纸。

> [!WARNING]
> ⚠️ 这是一个从教育项目演变成的基于 OpenGL 的 Linux 壁纸引擎。可能会有一些限制和奇怪的行为！

---

## 系统要求与依赖

依赖包括：

- OpenGL 3.3 support
- CMake
- LZ4、Zlib
- SDL2
- FFmpeg
- X11 或 Wayland
- Xrandr（X11 需要）
- GLFW3、GLEW、GLUT、GLM
- MPV
- PulseAudio
- FFTW3

> 注：PulseAudio 依赖也可由 PipeWire 的 PulseAudio 兼容层提供。

---

## 发行版包管理器安装

以下安装方法来自 [linux-wallpaperengine readme 页面](https://github.com/Almamu/linux-wallpaperengine)。

### Arch Linux

使用 AUR helper 安装最新开发版：

```bash
yay -S linux-wallpaperengine-git
# 或
paru -S linux-wallpaperengine-git
```

> 安装后仍然需要 Steam 版 Wallpaper Engine 的资源文件，见下文“获取 Wallpaper Engine 资源”。

### Ubuntu 22.04

```bash
sudo apt-get update
sudo apt-get install build-essential cmake libxrandr-dev libxinerama-dev libxcursor-dev libxi-dev libgl-dev libglew-dev freeglut3-dev libsdl2-dev liblz4-dev libavcodec-dev libavformat-dev libavutil-dev libswscale-dev libxxf86vm-dev libglm-dev libglfw3-dev libmpv-dev mpv libmpv1 libpulse-dev libpulse0 libfftw3-dev libfreetype-dev
```

### Ubuntu 24.04

```bash
sudo apt-get update
sudo apt-get install build-essential cmake libxrandr-dev libxinerama-dev libxcursor-dev libxi-dev libgl-dev libglew-dev freeglut3-dev libsdl2-dev liblz4-dev libavcodec-dev libavformat-dev libavutil-dev libswscale-dev libxxf86vm-dev libglm-dev libglfw3-dev libmpv-dev mpv libmpv2 libpulse-dev libpulse0 libfftw3-dev libfreetype-dev
```

### Alt Linux

```bash
sudo epm update
sudo epm install gcc-c++ make cmake libXrandr-devel libXinerama-devel libXcursor-devel libXi-devel libGL-devel libGLEW-devel freeglut-devel libSDL2-devel liblz4-devel libavcodec-devel libavformat-devel libavutil-devel libswscale-devel libXxf86vm-devel libglm-devel libglfw3-devel libmpv-devel mpv libpulseaudio-devel libpulseaudio libfftw3-devel libpng-devel libffi-devel libswresample-devel libgmpxx-devel
```

### Fedora 42

```bash
sudo dnf update
sudo dnf install gcc g++ cmake libXrandr-devel libXinerama-devel libXcursor-devel libXi-devel mesa-libGL-devel glew-devel freeglut-devel SDL2-devel lz4-devel ffmpeg ffmpeg-free-devel libXxf86vm-devel glm-devel glfw-devel mpv mpv-devel pulseaudio-libs-devel fftw-devel gmp-devel
```

---

## 获取 Wallpaper Engine 资源

你**必须拥有并通过 Steam 安装 Wallpaper Engine**。许多背景需要官方 Wallpaper Engine 的 `assets` 资源。

程序会尝试从以下 Steam 安装路径自动检测资源：

```text
~/.steam/steam/steamapps/common
~/.local/share/Steam/steamapps/common
~/.var/app/com.valvesoftware.Steam/.local/share/Steam/steamapps/common
~/snap/steam/common/.local/share/Steam/steamapps/common
```

> ✅ 如果 Wallpaper Engine 安装在以上路径之一，资源通常会被自动检测。

### 如果资源没有被自动找到

你可能会看到类似提示：

```text
Cannot find a valid assets folder, resolved to 'assets'
```

此时可以手动复制 `assets` 文件夹：

1. 在 Steam 中右键 **Wallpaper Engine** → **管理** → **浏览本地文件**
2. 复制其中的 `assets` 文件夹
3. 将 `assets` 放到 `linux-wallpaperengine` 二进制文件所在目录中
   - 如果按源码构建，通常放在 `build/output` 中
   - 如果安装到 `/opt/linux-wallpaperengine`，则放到对应安装目录
4. 或者使用 `--assets-dir` 手动指定路径：

```bash
linux-wallpaperengine --assets-dir /path/to/assets
```

---

## 从源码构建（新 readme 推荐方式）

> ⚠️ 如果已经通过 AUR 安装，可跳过本节。

克隆仓库时一并拉取子模组：

```bash
git clone --recurse-submodules https://github.com/Almamu/linux-wallpaperengine.git
cd linux-wallpaperengine
```

构建：

```bash
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE='Release' ..
make
```

可选：使用多核编译加速：

```bash
make -j$(nproc)
```

构建完成后，会生成一个 `output` 文件夹，其中包含程序和运行所需文件。程序路径通常为：

```bash
./output/linux-wallpaperengine
```

---

## Arch Linux 手动构建与安装到 /opt（可选）

如果 AUR 安装或子模组拉取失败，可以按下面步骤手动构建。此方式也适合将程序安装到 `/opt/linux-wallpaperengine`。

### 步骤 1：安装编译/运行依赖

```bash
sudo pacman -S --needed lz4 ffmpeg mpv glfw glew freeglut libpulse git cmake sdl2 glm xorg-xrandr wayland-protocols
```

### 步骤 2：克隆主项目源码

```bash
mkdir -p ~/build
cd ~/build

# 推荐同时拉取子模组
git clone --recurse-submodules https://github.com/Almamu/linux-wallpaperengine.git linux-wallpaperengine-git
cd linux-wallpaperengine-git
```

如果子模组没有拉取成功，继续执行步骤 3。

### 步骤 3：手动补全所有子模组（子模组克隆失败时使用）

子模组位于 `src/External/`。手动下载每个子模组到对应路径：

```bash
cd src/External

# 删除空的/失败的子模组目录
rm -rf argparse Catch2 glslang-WallpaperEngine json kissfft MimeTypes SPIRV-Cross-WallpaperEngine stb

# 可加 --depth 1 加速下载
git clone https://github.com/catchorg/Catch2.git Catch2
git clone https://github.com/lasselukkari/MimeTypes.git MimeTypes
git clone https://github.com/Almamu/SPIRV-Cross-WallpaperEngine.git SPIRV-Cross-WallpaperEngine
git clone https://github.com/p-ranav/argparse.git argparse
git clone https://github.com/Almamu/glslang-WallpaperEngine.git glslang-WallpaperEngine
git clone https://github.com/nlohmann/json.git json
git clone https://github.com/mborgerding/kissfft.git kissfft
git clone https://github.com/nothings/stb.git stb

cd ../..
```

如果某个子模组 Git 克隆失败，例如 `MimeTypes`，可用压缩包替代：

```bash
cd src/External
rm -rf MimeTypes

wget https://github.com/lasselukkari/MimeTypes/archive/refs/heads/master.zip -O MimeTypes.zip
unzip MimeTypes.zip
mv MimeTypes-master MimeTypes
rm MimeTypes.zip

cd ../..
```

### 步骤 4：编译项目

```bash
mkdir -p build

cmake -B build -S . \
    -DCMAKE_BUILD_TYPE='Release' \
    -DCMAKE_INSTALL_PREFIX="/opt/linux-wallpaperengine" \
    -Wno-dev \
    -DCMAKE_CXX_FLAGS="-ffat-lto-objects -Wno-builtin-macro-redefined" \
    -DCMAKE_C_FLAGS="-Wno-builtin-macro-redefined"

cmake --build build -j$(nproc)
```

### 步骤 5：安装到系统

```bash
sudo cmake --install build
```

创建 `/usr/bin/linux-wallpaperengine` 启动脚本：

```bash
sudo tee /usr/bin/linux-wallpaperengine > /dev/null << EOF
#!/bin/bash
export LD_LIBRARY_PATH="/opt/linux-wallpaperengine/lib:\$LD_LIBRARY_PATH"
cd /opt/linux-wallpaperengine; ./linux-wallpaperengine "\$@"
EOF
```

赋予执行权限：

```bash
sudo chmod +x /usr/bin/linux-wallpaperengine
sudo chmod +x /opt/linux-wallpaperengine/linux-wallpaperengine
```

### 验证安装

```bash
linux-wallpaperengine --help
```

如果使用源码构建的 `output` 目录，可运行：

```bash
./output/linux-wallpaperengine --help
```

---

## 使用

### 基本语法

```bash
linux-wallpaperengine [options] <background_id or path>
```

背景可以使用以下任一方式指定：

- Steam 创意工坊 ID，例如 `1845706469`
- 背景文件夹路径，例如 `~/wallpaperengine/1845706469/`

### 常见选项

| 选项 | 描述 |
| --- | --- |
| `--silent` | 静音背景音频 |
| `--volume <val>` | 设置音频音量 |
| `--noautomute` | 其它应用程序播放音频时不要自动静音 |
| `--no-audio-processing` | 禁用音频响应功能 |
| `--fps <val>` | 限制帧率 |
| `--window <XxYxWxH>` | 以自定义大小/位置在窗口模式下运行 |
| `--screen-root <screen>` | 设置为特定屏幕的背景 |
| `--screen-span <screen-1>,<screen-2>,...` | 将单个壁纸拉伸到多个屏幕 |
| `--bg <id/path>` | 将背景分配给特定屏幕，需配合 `--screen-root` 或 `--screen-span` 使用 |
| `--scaling <mode>` | 壁纸缩放：`stretch`、`fit`、`fill` 或 `default` |
| `--clamping <mode>` | 设置纹理锁定：`clamp`、`border`、`repeat` |
| `--assets-dir <path>` | 设置 assets 的自定义路径 |
| `--screenshot <file>` | 保存屏幕截图（PNG、JPEG、BMP） |
| `--list-properties` | 显示壁纸的可定制属性 |
| `--set-property name=value` | 覆盖特定属性 |
| `--disable-mouse` | 禁用鼠标交互 |
| `--disable-parallax` | 禁用支持视差效果的背景 |
| `--no-fullscreen-pause` | 全屏应用运行时防止暂停 |
| `--fullscreen-pause-only-active` | 仅 Wayland：仅当有活动全屏窗口时暂停 |
| `--fullscreen-pause-ignore-appid <val>` | 仅 Wayland：忽略 `app_id` 包含 `<val>` 的全屏窗口，可重复使用 |

### 用法示例

通过 ID 运行背景：

```bash
linux-wallpaperengine 1845706469
```

从文件夹运行背景：

```bash
linux-wallpaperengine ~/wallpaperengine/1845706469/
```

为不同屏幕分配不同背景并设置缩放：

```bash
linux-wallpaperengine \
  --scaling stretch --screen-root eDP-1 --bg 2667198601 \
  --scaling fill --screen-root HDMI-1 --bg 2667198602
```

将单个壁纸跨多个显示器拉伸：

```bash
linux-wallpaperengine \
  --scaling fill --screen-span HDMI-A-1,DP-2,DP-3 --bg 1845706469
```

窗口模式运行：

```bash
linux-wallpaperengine --window 0x0x1280x720 1845706469
```

限制 FPS 以节省电量：

```bash
linux-wallpaperengine --fps 30 1845706469
```

保存截图：

```bash
linux-wallpaperengine --screenshot ~/wallpaper.png 1845706469
```

查看壁纸可定制属性：

```bash
linux-wallpaperengine --list-properties 2370927443
```

修改属性，例如开启 bloom：

```bash
linux-wallpaperengine --set-property bloom=1 2370927443
```

旧版示例也可用：

```bash
linux-wallpaperengine --screen-root eDP-1 --screen-root HDMI-A-1 --fps 15 --scaling fill ~/wallpaperengine/3625547979/
```

---

## 创建 Steam 创意工坊壁纸目录软链接

创建软链接后，可以直接用软链接路径指定壁纸，不用输入超长 Steam 路径。

### 步骤 1：确认目标目录存在

```bash
# `xx` 替换为你的用户名，431960 是 Wallpaper Engine 创意工坊目录
ls -d /home/xx/.steam/steam/steamapps/workshop/content/431960
```

如果输出目录路径且没有 `No such file or directory` 报错，说明目录存在。

### 步骤 2：创建软链接

```bash
ln -s /home/xx/.steam/steam/steamapps/workshop/content/431960 /home/xx/wallpaperengine
```

### 步骤 3：验证软链接

```bash
ls -l /home/xx/wallpaperengine
```

正常输出示例：

```text
lrwxrwxrwx 1 xx xx 70 Dec 18 19:00 /home/xx/wallpaperengine -> /home/xx/.steam/steam/steamapps/workshop/content/431960
```

### 删除软链接

```bash
# 注意不要加结尾的 /
rm /home/xx/wallpaperengine

# 如果软链接在 ~/.local/bin
rm ~/.local/bin/wallpaperengine
```

---

## GUI

目前项目本身不提供 GUI。以下是一些第三方 GUI / 控制工具：

- [simple-linux-wallpaperengine-gui](https://github.com/Maxnights/simple-linux-wallpaperengine-gui) by @Maxnights
- [linux-wallpaper-engine](https://github.com/jagrat7/linux-wallpaper-engine) by @jagrat7
- [wallpaperengine-gui](https://github.com/MikiDevLog/wallpaperengine-gui) by @MikiDevLog
- [linux-wallpaperengine-controller for Noctalia Shell](https://noctalia.dev/plugins/linux-wallpaperengine-controller/) by @PaloMiku
- [waypaper](https://github.com/anufrievroman/waypaper) by @anufrievroman

---

## Wayland 与 X11 支持

- **Wayland**：适用于支持 `wlr-layer-shell-unstable` 的合成器。使用 `xdg-output-unstable-v1` 进行准确的显示器定位，`--screen-span` 需要该支持。
- **X11**：需要 XRandr。使用 `--screen-root <screen_name>`，屏幕名称可通过 `xrandr` 查看。

> ⚠️ X11 用户注意：目前如果合成器或桌面环境（如 GNOME、KDE、Nautilus）正在绘制背景，程序可能无法正常工作。

---

## 示例背景

项目网站展示页：  
https://wpengine.alma.mu/#showcase

---

## 常见问题

### 设置为屏幕背景时出现黑屏

这可能由不同环境和设置导致。

#### X11

常见原因是合成器正在绘制背景，导致 Wallpaper Engine 无法正常显示。目前的解决方案是禁用合成器，让 Wallpaper Engine 能直接在屏幕上绘制。

#### NVIDIA

部分用户遇到 GLFW 初始化和其他 OpenGL 错误。可尝试运行时添加：

```bash
__GL_THREADED_OPTIMIZATIONS=0 linux-wallpaperengine
```

这是一个临时规避方案，未来项目可能会改进。

更多问题请参考 [linux-wallpaperengine readme 页面](https://github.com/Almamu/linux-wallpaperengine)。

---

## 特别感谢

- [RePKG](https://github.com/notscuffed/repkg) – texture flag insights
- [RenderDoc](https://github.com/baldurk/renderdoc) – the best OpenGL debugger out there!

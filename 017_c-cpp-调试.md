# c cpp 调试

> [!NOTE]
> By WuDiXianXin

**编译时带上调试信息，不要做优化**，给 codelldb 用的 debug 程序，重点是 `-g`（生成调试符号）、`-O0`（关闭优化）。

## GCC（C / C++）

```bash
# C
gcc main.c -o main_debug -g -O0

# C++
g++ main.cpp -o main_debug -g -O0
```

- `-g`：生成 DWARF 调试信息（codelldb 用的就是 DWARF）
- `-O0`：无优化，变量、代码行一一对应，断点最稳
- 可选 `-ggdb`：专门给 gdb/lldb 风格的调试信息，和 `-g` 大部分场景等价

> [!TIP]
> 如果你想保留少量优化但还能调试：`-Og`，适合后期调试，但是局部变量可能被优化掉，断点容易跳。

## Clang（C / C++，codelldb 底层就是 lldb，优先 clang）

```bash
clang main.c -o main_debug -g -O0
clang++ main.cpp -o main_debug -g -O0
```

clang 生成的调试信息对 lldb/codelldb 兼容性最好。

## CMake 项目（最常用）

```bash
# 构建时指定 Debug 类型
cmake -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build
```

CMake `Debug` 模式默认：

- `-g` 开启调试符号
- `-O0` 关闭优化

> [!TIP]
> RelWithDebInfo：带调试信息 + 优化（类似 `-Og`，性能接近release，能调试，但是断点容易异常）

```bash
cmake -B build -DCMAKE_BUILD_TYPE=RelWithDebInfo
```

## Makefile 写法示例

```makefile
CC = gcc
CXX = g++
CFLAGS += -g -O0
CXXFLAGS += -g -O0
```

## Meson

```bash
meson setup build --buildtype=debug
# 或者 debugoptimized（带调试信息+少量优化）
meson setup build --buildtype=debugoptimized
```

## 给 codelldb 的注意点（踩坑重点）

1. **不要 strip**：不能 `strip main_debug`，strip 会删掉调试符号，断点失效
2. 路径：源码位置要和编译时一致，不然 codelldb 找不到源码
3. 分离调试符号（可选，程序本体干净，符号单独放）

```bash
# 提取调试信息到单独文件，主程序strip
objcopy --only-keep-debug main_debug main.debug
objcopy --strip-debug main_debug
objcopy --add-gnu-debuglink=main.debug main_debug
```

## 简单 nvim-dap launch 示例（C/C++，直接用）

```lua
-- ==================== LLDB (codelldb) 适配器 ====================

local dap = require('dap')

local codelldb_pkg = require('mason-registry').get_package('codelldb')
local install_root = codelldb_pkg:get_install_path()
local codelldb_path = install_root .. '/extension/adapter/codelldb'

local this_os = vim.uv.os_uname().sysname
if this_os:find('Windows') then
    codelldb_path = install_root .. '\\extension\\adapter\\codelldb.exe'
end

dap.adapters.lldb = {
    type = 'server',
    port = '${port}',
    executable = {
        command = codelldb_path,
        args = { '--port', '${port}' },
    },
}

-- ==================== C / C++ ====================

local lldb_launch = {
    name = 'Launch',
    type = 'lldb',
    request = 'launch',
    program = function()
        local cwd = vim.fn.getcwd()
        local sep = vim.uv.os_uname().sysname:find('Windows') and '\\' or '/'
        local default = vim.fs.normalize(cwd) .. sep
        return vim.fn.input('可执行文件路径: ', default, 'file')
    end,
    cwd = vim.fn.getcwd(),
    stopOnEntry = false,
    args = {},
}
dap.configurations.c = { lldb_launch }
dap.configurations.cpp = { lldb_launch }
```

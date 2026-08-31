# Ch8. Miscellaneous Topics

> 对应 PDF 物理页 305-358（印刷页 291-344）。本章 54 页收尾章——C 标准库的工具箱：regex 正则 / assert 断言 / locale i18n 国际化 / **WebAssembly**（emcc 把 C 编成 wasm）/ signal 信号 / **静态/动态库构建 + C/Python 跨语言调用**。**AI 工具链中 wasm 沙箱 + Python ctypes 调用 C 库** 是这一章与前 7 章最不同的实战点。

## §1 章节概述

1. **POSIX `<regex.h>`** = `regcomp(&re, pattern, REG_EXTENDED)` 编译 + `regexec(&re, str, nmatch, pmatch, eflags)` 匹配 + `regfree(&re)` 释放。**标准 C regex 不支持 lookbehind/lookahead**——用 PCRE (`pcre.org`) 替代。
2. **`<assert.h>` 三种 assertion** = precondition（块前）/ invariant（块内）/ postcondition（块后）。`#define NDEBUG` **关闭所有 assert**——release build 必加。**C11 `_Static_assert` 编译期 assert**（比 `assert` 更严，编译就报错）。
3. **locale = 用户环境的"区域设置"**——`setlocale(LC_ALL, "")` 用环境变量 `LANG` 初始化；`localeconv()` 返回 `struct lconv*` 含小数点、千位分隔、货币符号等 18 字段。**i18n = internationalization**（i 18 字母 n）。
4. **WebAssembly (wasm)** = 浏览器/Node.js 里的二进制字节码格式（紧凑、可流式验证、近原生速度）。**emcc (Emscripten) 把 C 编成 wasm**，`--no-entry` 表无 main，`EMSCRIPTEN_KEEPALIVE` 让 C 函数暴露给 JS。**WASI = WebAssembly System Interface**——让 wasm 跑在 server 端（Deno / Wasmtime / Wasmer）。
5. **signal** = 进程间"低层通知"机制。`kill(pid, sig)` 发；`signal(SIGNO, handler)` 或 `sigaction(SIGNO, &sa, NULL)` 收。**`SIGKILL`/`SIGSTOP` 不能被捕获/忽略/屏蔽**——kernel 强杀。**handler 只该调 async-signal-safe 函数**（`write`/`_exit`/`sem_post`）。
6. **静态库 (`.a`)** = `ar rcs libfoo.a foo.o`；链接时整个塞进 binary；**binary 大、不能热更新**。**动态库 (`.so`)** = `gcc -shared -fPIC -Wl,-soname,libfoo.so.1 -o libfoo.so.1.0 foo.o` + `sudo ldconfig`；链接时只塞符号，运行时 dlopen 加载；**binary 小、可热更新、多个 binary 共享**。
7. **soname = dynamic library 的逻辑名** = `libshprimes.so`（不带版本号）；物理文件名 = `libshprimes.so.1`；链接器通过 `ld.so.conf` + `ldconfig` cache 找。**升级 lib 只需要改物理文件 + ldconfig**，client 不用重编译。
8. **Python `ctypes` 调 C 库** = `cdll.LoadLibrary("libfoo.so")` 加载 → `lib.foo(arg)` 调用 → `restype = None` 配置 void 返回 → `ctypes.c_int(42)` 配置入参类型。**NumPy / pandas** 大量用 ctypes + cffi。
9. **链接顺序** = `-lshprimes -lm` 而**不是** `-lm -lshprimes`——**GNU ld 是 lazy link**，从左到右解析未定义符号，**库靠右放**。`gcc -Wl,--as-needed` 可减少不必要的库链接。
10. **linker 是单趟扫描、lazy resolve**——`gcc main.c -L. -lfoo` 解析 `main.c` 里引用 → 找 `libfoo.so` → 把未解析的留到下次；左依赖右。

## §2 核心 Takeaways

### T1 — `<regex.h>` 是 POSIX 标准但功能有限
- **是什么**：`regcomp/regexec/regfree` 三件套 + `regex_t` (编译后的状态机) + `regmatch_t` (capture group 偏移)。
- **为什么重要**：**标准 C regex 够日常**（anchors、quantifiers、character classes、capture groups）。**不够**：lookbehind/lookahead、backreference、non-greedy（这些 PCRE 有）。
- **场景**：CLI 工具 (`grep`/`sed`)、用户输入验证、log 解析。**嵌入式**几乎不用（regex 引擎太大）。
- **陷阱**：`REG_NOMATCH` 是错误码（0/非 0），不是返回 -1。

### T2 — `assert` 是"开发期保险"，生产代码关掉
- **是什么**：`assert(cond)` 编译期由 `NDEBUG` 决定是否生效。**有 NDEBUG → assert 编译为空**（零开销）。
- **为什么重要**：release binary 里 `assert` 没了——所有"防御性检查"消失。**生产代码靠 errno + return code + log**。
- **C11 `_Static_assert(expr, "msg")`** 编译期检查（`sizeof` 验证 / ABI 兼容 / 常量表达式）。
- **场景**：单元测试 + CI 跑 debug build，release build 关掉。

### T3 — `setlocale(LC_ALL, "")` + `localeconv()` 是 i18n 标准路径
- **是什么**：`setlocale(category, name)` 改 locale；`name = ""` 读环境变量 `LANG`；`name = NULL` 查当前 locale。`localeconv()` 返回 `struct lconv*` 含数字/货币/时间格式。
- **为什么重要**：**i18n 第一步**——所有数字/货币/时间格式化函数（`printf %f`/`strftime`）都看当前 locale。
- **场景**：web 后端（`Accept-Language` 头 → `setlocale`）、财务系统（`LC_MONETARY`）、跨地区 CLI 工具。
- **限制**：`lconv` 不支持现代 i18n 概念（plural、gender）——用 ICU / gettext。

### T4 — WebAssembly 是 C 的"第二个生命周期"
- **是什么**：栈式 VM 字节码（~120KB 解释器），紧凑二进制格式（`.wasm`），JIT/AOT 编译到 native code。**emcc (Emscripten) = LLVM-to-wasm 编译器**，把 C 编成 wasm + 配 JS 胶水。
- **为什么重要**：
  - **浏览器** = JS + wasm = 浏览器跑 C/C++/Rust 高性能代码
  - **WASI** = 浏览器外跑 wasm = **Deno / Wasmtime / Wasmer** 替代 Docker 容器
  - **AI 工具链** = Claude Code / Codex 沙箱**用 wasm 跑用户 C 代码**（ch7 + ch8 实战）
- **场景**：浏览器内图像处理（Photoshop web）、游戏引擎、加密、跨语言复用、Serverless 函数。
- **限制**：**没有 raw socket / 文件系统**（必须 WASI 或 JS 桥）；**多线程依赖 SharedArrayBuffer + COOP/COEP 头**。

### T5 — emcc 把 C 编译成 wasm 的关键标志
- **是什么**：`emcc foo.c -o foo.js --no-entry -s EXPORTED_FUNCTIONS='["_hstone"]'` —— 输出 `.js` + `.wasm`。
- **关键 flags**：
  - `--no-entry` —— 没有 main（**纯函数库**）
  - `-s EXPORTED_FUNCTIONS='["_fn"]'` —— 暴露 C 函数给 JS（前缀 `_` 因为 C→wasm name mangling）
  - `-s EXPORTED_RUNTIME_METHODS='["ccall","cwrap"]'` —— 暴露 `Module.ccall`/`cwrap` JS API
  - `-O3` —— LLVM 优化（vs 默认 `-O0`）
  - `-s WASM=1` —— 纯 wasm 输出（不带 asm.js fallback）
- **`EMSCRIPTEN_KEEPALIVE`** = emcc 宏（`<emscripten/emscripten.h>`）= `__attribute__((used)) + visibility("default")`——让 dead-code elimination 保留函数。

### T6 — signal handler 是 async-signal-safe 的硬约束
- **是什么**：signal handler **在 signal 上下文**跑（**不是线程**），只能调 async-signal-safe 函数（POSIX 列了 117 个）。**不能调 `printf`/`malloc`/`pthread_mutex_lock`**——会死锁。
- **为什么重要**：典型错：handler 里 `printf("got SIGUSR1\n")` —— `printf` 用全局 lock，handler 打断正在 `printf` 的线程 → 死锁。
- **正确做法**：
  - **handler 只 set flag**（`volatile sig_atomic_t flag = 1;`）
  - **主线程/工作线程** poll flag 后做实际工作
  - 或用 `signalfd` / `eventfd` 把 signal 转成 fd（**与 epoll 整合**）
- **`sigaction` 比 `signal` 优先**——`signal` 在不同 Unix 行为不一（System V 重启 = 不重启；BSD 重启），`sigaction` 显式控制 `SA_RESTART`。

### T7 — 静态 vs 动态库：trade-off
- **静态库 `.a`** = 编译时塞进 binary；**binary 大**；**部署简单**（一个文件）；**库更新需重链接**所有 client。
- **动态库 `.so`** = 运行时 dlopen；**binary 小**；**库更新不用重链接**（soname 不变）；**多 binary 共享同一份代码**。
- **现代选择**：
  - **OS-level lib**（glibc、libm）= 动态
  - **应用层**（业务库）= 看情况——性能敏感用静态，模块化用动态
  - **容器化/嵌入式** = 几乎全静态（**distroless** 镜像 / musl libc）
- **Cargo/Rust** 全静态二进制（`cargo build --release`）成为现代趋势。

### T8 — soname + ldconfig 是动态库的灵魂
- **是什么**：动态库有 3 个名字：
  1. **logical name** = `libfoo.so`（编译时链接用，**永不变**）
  2. **soname** = `libfoo.so.1`（运行时 dlopen 用，major version 不变）
  3. **real name** = `libfoo.so.1.0.0`（物理文件，minor/patch 每次发布可改）
- **client 链接时**用 logical name（`libfoo.so`），**运行时**通过 `ld.so` 找 soname（`libfoo.so.1`）→ 找 real name（`libfoo.so.1.0.0`）。
- **`ldconfig`** 维护 `/etc/ld.so.cache` 索引；新装 lib 后必跑 `sudo ldconfig`。
- **ABI 兼容规则** = minor/patch 升级 = ABI 兼容（不需重链接）；major 升级 = ABI 不兼容（必须重链接）。

### T9 — `-fPIC` 是动态库的硬性要求
- **是什么**：Position-Independent Code 编译选项。**所有动态库的 `.o` 都必须 `-fPIC`**，否则 `gcc -shared` 链接时报 `relocation R_X86_64_PC32 against symbol ... can not be used when making a shared object`。
- **为什么需要**：动态库被加载到任意地址；指令里的绝对地址会让库崩溃。**`-fPIC` 用相对地址 + GOT/PLT** 间接寻址，**多一次内存访问**（性能损失 ~5%）。
- **x86-64 缓解**：现代 CPU 有 `RIP-relative addressing`（`-fpie` 充分利用），性能损失很小。

### T10 — Python `ctypes` 是最简单的跨语言 FFI
- **是什么**：`import ctypes; lib = ctypes.CDLL("libfoo.so")` 加载 → `lib.foo(42)` 调用。**完全在 Python 解释器内**无需编译。
- **代价**：**没有类型检查**——C 函数 `void foo(int)` Python 调 `lib.foo("hello")` 会在 C 层段错误；**类型必须显式 `argtypes = [ctypes.c_int]`**。
- **更强替代**：`cffi`（更 Pythonic）/ `pybind11` / `cppyy`（C++ 友好）/ `Rust pyo3`（Rust 友好）。
- **场景**：快速粘合现有 C 库；性能关键的 wrapper 用 C 扩展 / Cython。
- **NumPy 用 ctypes + 自己 C extension** 暴露 `ndarray`。

## §3 工程实践视角

### 3.1 Linux 系统开发视角

- **`/etc/ld.so.conf.d/*.conf` 配动态库搜索路径** = `echo "/usr/local/lib" > /etc/ld.so.conf.d/local.conf && sudo ldconfig`。
- **`LD_LIBRARY_PATH` 环境变量** = 临时改库搜索路径（**生产禁用**——library 注入攻击面）。
- **`RPATH` / `RUNPATH` ELF metadata** = 把库路径塞进 binary 自身，**更安全的部署**——`gcc -Wl,-rpath,$ORIGIN` 相对路径。
- **`patchelf --set-rpath`** = 改既有 binary 的 rpath（**Electron / AppImage 常用**）。
- **`ar -t libfoo.a`** 查 archive 内容；**`nm libfoo.so`** 查符号；**`objdump -T libfoo.so`** 查动态符号；**`ldd program`** 查运行时依赖。
- **`gcc -fvisibility=hidden -fvisibility-inlines-hidden`** 默认隐藏所有符号；只 `__attribute__((visibility("default")))` 的导出。**大幅减小 .so 体积 + 加速链接**。
- **`-Wl,--as-needed`** = 只链实际用到的库（**Docker 镜像瘦身的核心**）。
- **静态分析工具**：`ltrace program` 看所有动态库调用；`strace -e openat program` 看所有 dlopen。
- **ABI 兼容** = `gcc -fabi-version=N`（GCC 5+ 默认 v8）+ `gcc -fno-exceptions` 减少 ABI 表面。
- **C++ 静态库 ABI** = `_GLIBCXX_USE_CXX11_ABI`（libstdc++ 旧/新 string ABI 冲突）——**Python Cython / pybind11 经常踩**。
- **musl libc** = 静态链接首选；`apk add musl-dev` 配 `-static` 编译，**binary 单文件**跑 Alpine / distroless 容器。
- **Google `Abseil` / `Boost`** = C++ 准标准库；静态/动态都有；**现代 C++ 项目的底座**。
- **`-Wl,--gc-sections`** + `-ffunction-sections -fdata-sections` = **strip 未用代码**，**典型 30-50% binary 瘦身**。

### 3.2 机器人软件视角（ROS2 / 嵌入式控制）

- **ROS2 用 C/C++ 实现核心** = rclcpp/rclpy 底层都是 C 库 `rcl` + `rcpputils` + `rcutils`；**`rcl` 是动态库**（`/opt/ros/humble/lib/librcl.so`）。
- **`colcon build` = catkin_make 现代版** = 创建 `install/lib/pkgname/` 装 .so；`source install/setup.bash` 加 `LD_LIBRARY_PATH`。
- **ros2_control hardware interface 跨语言** = `rclcpp::Node` 是 C++，但通过 `micro-ros` 让 **嵌入式 MCU (FreeRTOS)** 用 C 调 ROS2 服务。
- **MoveIt 2 IK 求解器** = C++ + Python binding（pybind11）—— 性能关键算法 C++，调试/可视化 Python。
- **Gazebo 仿真器** = **纯 C++ 动态库架构** = `gz-physics` + `gz-rendering` + `gz-sensors` —— 插件机制基于 **动态库 dlopen**。
- **ROS2 message IDL → C 结构体** = 编译时生成；**Python ctypes 不能直接用**（layout 不同）——**用 `ros2 pkg create` 生成 Python binding**。
- **Robot Web Tools**（Foxglove / Webviz）= **浏览器调 ROS2 topic** = **WebSocket + protobuf + wasm 渲染**——**ch8 wasm + ch6 网络 + ch5 I/O** 三合一。
- **CUDA C 库 = 动态库** = `libcudart.so` / `libcublas.so`；**机器人加速 (Point Cloud → ICP SLAM)** = cuBLAS + cuDNN。

### 3.3 初级 vs 高级工程师对照

| 习惯 | 初级 | 高级 |
|---|---|---|
| 库选型 | 全静态 `.a`（binary 大） | 大型 OS lib 动态 / 应用 lib 按场景 |
| `.so` 命名 | `libfoo.so` | `libfoo.so.1.0.0` + `libfoo.so.1` (soname) + `libfoo.so` (logical) |
| 链接顺序 | `gcc main.c -lm -lfoo` (有 undefined reference) | `gcc main.c -lfoo -lm` (libfoo 用到 libm 时) |
| ldconfig | 装完不跑 | `sudo ldconfig` 后 `ldconfig -p \| grep foo` 验证 |
| signal handler | `printf` + `malloc` | `volatile sig_atomic_t flag = 1` + 主循环 poll |
| 国际化 | 不管 locale | `setlocale(LC_ALL, "")` + ICU for serious i18n |
| regex | 手写 state machine | `<regex.h>` 或 PCRE2 (现代) |
| assert | debug + release 都留着 | `#define NDEBUG` + `_Static_assert` 编译期 |
| WebAssembly | 不知道 | `emcc` + WASI + 浏览器沙箱 |
| Python 调 C | 不知道 ctypes | `ctypes` / `cffi` / `pybind11` 按场景 |

## §4 AI 时代视角

### 4.1 这些知识还重要吗？（2026 年视角）

**极重要——但呈现形式变了**：
- **动态库** = OS-level lib（glibc、libm、OpenSSL）；**应用层几乎全静态**（Docker distroless、Go/Rust binary）
- **WebAssembly** = **AI agent 沙箱运行时**——Claude Code / Codex 跑用户 C 代码 = wasm
- **Python ctypes** = AI 工具调 C 库的"胶水"——**LangChain tool calling 部分用 ctypes**
- **regex** = log 解析、数据提取的核心——AI 训练数据清洗大量靠 regex
- **assert** = **测试驱动开发**（TDD）的语法基础——AI 写代码必生成 assert 断言

**AI 写这些代码**：95% 错在**"看似能跑、实际有边界 bug"**——比如 signal handler 用 `printf`、动态库漏 `-fPIC`、soname 算错、locale 忘初始化。

### 4.2 AI 现在能做的

- ✅ 写 `<regex.h>` 三件套（编译、匹配、释放）
- ✅ 写 `assert` 模板 + `NDEBUG` 配置
- ✅ 解释 `setlocale` + `lconv` 18 字段
- ✅ 写 emcc 命令把 C 编成 wasm
- ✅ 写 `sigaction` 替代 `signal`
- ✅ 解释动态库构建 5 步（compile -fPIC / link -shared / cp to /usr/local/lib / ldconfig / symlink）
- ✅ 写 Python `ctypes` 调 C 库

### 4.3 AI 经常写错的地方（必看）

| 错误模式 | 例子 | 后果 |
|---|---|---|
| **1. signal handler 用 `printf`** | `void handler(int sig) { printf("got\n"); }` | printf 持锁，handler 打断正在 printf 的线程 → 死锁 |
| **2. 动态库忘 `-fPIC`** | `gcc -c foo.c` 然后 `gcc -shared foo.o -o libfoo.so` | 链接错 `relocation R_X86_64_PC32 ... can not be used` |
| **3. 链接顺序错** | `gcc main.c -lm -lfoo` 但 libfoo 用 sqrt | `undefined reference to 'sqrt'` |
| **4. `setlocale` 忘调** | 直接 `printf("%.2f", 1.23)` | 数字/货币格式按 C locale 输出，**不是用户期望** |
| **5. 装完 `.so` 不跑 `ldconfig`** | `sudo cp libfoo.so.1 /usr/local/lib` | `program: error while loading shared libraries: libfoo.so.1` |
| **6. `assert` 在 release 没关** | release binary 仍调 `__assert_fail` | 性能下降 + 异常路径不优雅 |
| **7. `regex` 释放漏** | `regcomp` 完没 `regfree` | 内存泄漏（ASan 抓） |
| **8. `setlocale(LC_ALL, "C")` 当作 default** | 强制 C locale 覆盖用户 `LANG=en_US.UTF-8` | **界面"突然变中文/英文"** 用户投诉 |
| **9. `<regex.h>` 假设支持 lookbehind** | `(?<=foo)bar` | **POSIX regex 不支持**——实际 match REG_NOMATCH，**AI 工具调试半天** |
| **10. WebAssembly 假设有文件系统** | `FILE* f = fopen("/etc/passwd", "r")` | wasm 跑在浏览器**没文件系统**——MEMFS 也没有这个路径 |
| **11. WebAssembly 假设有网络** | `socket(AF_INET, ...)` | wasm 跑在浏览器**没 raw socket**——只有 `fetch()` |
| **12. WebAssembly 假设线程** | `pthread_create(...)` | 浏览器 wasm 默认**单线程**——要 `SharedArrayBuffer` + `pthread` 支持 |
| **13. ctypes 调 C 库忘 `restype`** | `lib.void_fn()` | 返回**垃圾 int** —— Python 层 `None` 期望落空 |
| **14. ctypes 传字符串类型错** | `lib.fn("hello")` 实际 C 函数要 `char*` | 通常 OK（Python 3 str 自动转 c_char_p），但**C 函数释放参数**会段错误 |
| **15. 动态库**`SONAME` 跟 `filename` 错位 | `-o libfoo.so.1.0` 但 `-Wl,-soname,libfoo.so.2` | `ldconfig` 后 client 找 `libfoo.so.2` 找不到 |
| **16. `ar rcs` 漏 `s`** | `ar rc libfoo.a foo.o` | 没 `s` = **不创建索引**——链接时 `multiple definition` |
| **17. `nm` 看到 `U` 误判** | 函数没定义以为没事 | `U` = undefined = **链接会失败**（`T` 才 OK） |
| **18. `kill(0, SIGUSR1)` 当 ping** | `kill(0, sig)` 给**整个进程组**发信号 | 误杀兄弟进程 |

### 4.4 工程师必须保留的核心能力

- **能 5 秒内画出动态库 3 名字关系**（logical/soname/real name）。
- **能 30 秒内写出"safe signal handler"模板**（只 set flag + 主循环 poll）。
- **能用 `nm`/`ldd`/`objdump -T` 查库依赖**。
- **能写 `emcc` 命令把 C 编成 wasm**（即使不熟也能照模板改）。
- **能配 `setlocale(LC_ALL, "")`** + 用 `lconv` 字段做格式化。
- **能跟 AI 说"用 `<regex.h>` 标准库/signal handler 只 set flag/动态库必 `-fPIC`/`emcc --no-entry`"**——ch8 词汇是 prompt 的关键。

### 4.5 wasm 工具链延伸（用户已选需要）

- **WASI 0.2 (2024) 标准化** = wasm **正式成为** POSIX-like 系统接口。**Deno / Wasmtime / Wasmer** 都是 WASI runtime。
- **wasm Component Model** = **跨语言互操作标准**——Rust 写 wasm + Python 调，**无 FFI 代码**。
- **AI 工具链中 wasm 的 ch8 角色**：
  - **Claude Code / Codex 沙箱** 跑用户 C 代码 = **wasm + WASI**（ch7 pthread 模拟、ch8 assert/regex/locale stub）
  - **生成代码沙箱**（像 v0.dev / Bolt.new）= **wasm 跑用户写的 React/Vue 组件**
  - **MCP (Model Context Protocol) 工具** = 部分工具用 wasm 沙箱
- **pyodide** = CPython **编成 wasm**——浏览器跑 Python；**AI 工具的"嵌入式 Python"**；
- **FFmpeg.wasm / OpenCV.js** = 图像/视频处理 wasm 模块——**浏览器 Figma / Photopea / Photoshop web** 全用。

## §5 实践行动项

### A1 — 验证 regex 三件套（ch8.2 实战）
```bash
mkdir -p /tmp/modern-c/ch08 && cd /tmp/modern-c/ch08
cat > regex_emp.c <<'EOF'
#include <stdio.h>
#include <regex.h>
int main(void) {
    const char* pattern = "^[A-Z]{2}[1-9]{3}[a-k]{2}$";
    const char* inputs[] = {"AQ431af", "AQ431mf", "QQ444kk7", "AB123cd", NULL};
    regex_t re;
    char err[64];
    if (regcomp(&re, pattern, REG_EXTENDED) != 0) {
        fprintf(stderr, "regcomp failed\n");
        return 1;
    }
    for (int i = 0; inputs[i]; i++) {
        int m = regexec(&re, inputs[i], 0, NULL, 0);
        if (m == 0) {
            printf("  %s: VALID\n", inputs[i]);
        } else if (m == REG_NOMATCH) {
            printf("  %s: invalid\n", inputs[i]);
        } else {
            regerror(m, &re, err, sizeof(err));
            printf("  %s: error %s\n", inputs[i], err);
        }
    }
    regfree(&re);
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -o regex_emp regex_emp.c && ./regex_emp
```
**验收**：`AQ431af` VALID（k 之外 m 不行、`QQ444kk7` 末尾 7 不行、`AB123cd` 长度对 7）。

### A2 — 验证 assert + NDEBUG 关闭（ch8.3 实战）
```bash
cat > assert_test.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>
int main(int argc, char** argv) {
    assert(argc == 2);
    printf("arg: %s\n", argv[1]);
    int n = atoi(argv[1]);
    assert(n > 0 && n < 100);
    printf("n = %d OK\n", n);
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -g -O0 -o assert_debug assert_test.c
gcc -std=c11 -Wall -Wextra -O2 -DNDEBUG -o assert_release assert_test.c
echo "--- debug 跑 (无 arg 触发 assert) ---"
./assert_debug 2>&1 | head -3
echo "--- release 跑 (无 arg 静默通过) ---"
./assert_release 2>&1 | head -3
```
**验收**：debug 版本无 arg 报 `Assertion failed` 并 abort；release 版本无 arg 静默继续（**assert 编译成空**）。

### A3 — 验证 setlocale + localeconv（ch8.4 实战）
```bash
cat > locale_demo.c <<'EOF'
#include <stdio.h>
#include <locale.h>
int main(void) {
    setlocale(LC_ALL, "");
    struct lconv* lc = localeconv();
    printf("Locale: %s\n", setlocale(LC_ALL, NULL));
    printf("Decimal point: '%s'\n", lc->decimal_point);
    printf("Thousands sep: '%s'\n", lc->thousands_sep);
    printf("Currency sym:  '%s'\n", lc->currency_symbol);
    printf("Intl currency: '%s'\n", lc->int_curr_symbol);
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -o locale_demo locale_demo.c
echo "--- LANG=C ---"
LANG=C ./locale_demo
echo "--- LANG=en_US.UTF-8 ---"
LANG=en_US.UTF-8 ./locale_demo
echo "--- LANG=de_DE.UTF-8 (如果有) ---"
LANG=de_DE.UTF-8 ./locale_demo 2>&1 | head -3
```
**验收**：`LANG=C` 输出 `.` 为小数点、`""` 为千位分隔、`$` 为货币符号；`LANG=en_US.UTF-8` 类似；`LANG=de_DE` 输出 `,` 和 `EUR`（如果系统装了）。

### A4 — 静态库 + 动态库构建（ch8.7 实战）
```bash
cat > primes.h <<'EOF'
extern unsigned is_prime(unsigned n);
EOF
cat > primes.c <<'EOF'
#include "primes.h"
unsigned is_prime(unsigned n) {
    if (n < 2) return 0;
    if (n < 4) return 1;
    if (n % 2 == 0 || n % 3 == 0) return 0;
    for (unsigned i = 5; (i*i) <= n; i += 6) {
        if (n % i == 0 || n % (i+2) == 0) return 0;
    }
    return 1;
}
EOF

gcc -std=c11 -O2 -Wall -c primes.c -o primes.o
ar rcs libprimes.a primes.o
echo "--- nm libprimes.a ---"
nm libprimes.a

gcc -std=c11 -O2 -Wall -fPIC -c primes.c -o primes_pic.o
gcc -shared -Wl,-soname,libprimes.so.1 -o libprimes.so.1.0 primes_pic.o
ln -sf libprimes.so.1.0 libprimes.so.1
ln -sf libprimes.so.1.0 libprimes.so
ls -la libprimes*

cat > tester.c <<'EOF'
#include <stdio.h>
#include "primes.h"
int main(void) {
    for (unsigned i = 1; i <= 30; i++) {
        if (is_prime(i)) printf("%u ", i);
    }
    printf("\n");
    return 0;
}
EOF

echo "--- 链接静态库 ---"
gcc -std=c11 -Wall -o tester_static tester.c -L. -lprimes
ldd tester_static | grep primes

echo "--- 链接动态库 ---"
gcc -std=c11 -Wall -o tester_dynamic tester.c -L. -lprimes
LD_LIBRARY_PATH=. ldd tester_dynamic | grep primes
LD_LIBRARY_PATH=. ./tester_dynamic
```
**验收**：
- `nm libprimes.a` 显示 `T is_prime`
- `tester_static` 链接动态列表**没有** libprimes（已 baked in）
- `tester_dynamic` 链接列表**有** libprimes.so.1 → libprimes.so.1.0
- `./tester_dynamic` 输出 `2 3 5 7 11 13 17 19 23 29`

### A5 — Python ctypes 调 C 库（ch8.7.5 实战）
```bash
which python3
python3 -c "
import ctypes
lib = ctypes.CDLL('./libprimes.so')
print('is_prime(13):', lib.is_prime(13))
print('is_prime(12):', lib.is_prime(12))
print('is_prime(97):', lib.is_prime(97))
"
```
**验收**：`is_prime(13)=1, is_prime(12)=0, is_prime(97)=1`——Python 直接调 C 库，**零编译**。

## §6 值得深入思考的问题

1. **WASI 0.2 (2024) 标准化后，wasm 是不是替代 Docker 容器？** Wasmtime / Wasmer **启动时间 10ms**（Docker 100ms+）；**binary size 几 MB**（Docker image 几十 MB+）。**Serverless**（Lambda / Cloudflare Workers）已大量用 wasm。**Wasm + WASI = 下一代容器？** Cold start 优势压倒性。
2. **Python ctypes vs pybind11 vs cffi 选型？** **ctypes = 标准库，零依赖，简单但慢**（每次 call 有 type conversion）；**cffi = C-like syntax 写 wrapper，平衡**；**pybind11 = C++ 专属，最快，类型安全**；**PyO3 = Rust**。**NumPy / TensorFlow** 用 Cython + C extension 而不是 ctypes。
3. **assert vs `_Static_assert` vs error code 的选型？** `assert` = **开发期**（程序员/AI 自检）；`_Static_assert` = **编译期**（ABI 兼容、struct 布局）；`error code` = **运行期**（用户输入异常）。**生产** 必须 error code——`assert` 在 release 被关，依赖它就崩溃。
4. **soname 的语义化版本**（`libfoo.so.1.0.0`）怎么映射到实际兼容性？**Linux 标准**：major 变 = ABI 断；minor 变 = 加功能（ABI 兼容）；patch 变 = bug 修（ABI 兼容）。**Rust/Cargo** 全用 semver 强制。**Go modules** 没 major 概念，**靠 import path 强制**（`v2/module`）。**Python pip** 跟 semver 但允许 `^=`。
5. **`glibc` vs `musl` 的 ABI 兼容性**？**glibc = Linux 事实标准**（功能多，体积大）；**musl = 静态链接首选**（Alpine Linux 基础；体积小、启动快）。**同一个 C 程序**用 glibc 编译的 binary **不能**在 musl-only 系统跑。**Docker 镜像选择**（debian vs alpine）就是这问题。
6. **AI 写动态库经常漏的 `-fPIC`**——能否在 CI 强制？**Docker multi-stage build + 链接 `pic` flag 检查**；**`checksec --output=file,format=elf` 工具**（pwntools 项目）；**GitHub Action 跑 `lld` 严格检查**。
7. **WebAssembly Component Model (2024) 如何改变 C 库生态？** 设想：**C 库编成 wasm component**（带 ABI 描述）→ Rust/Python/JS 不用 ctypes 也能调。**wasm 1.0 = 字节码格式**；**wasm component = 自描述 + 跨语言**。**未来 5 年**：浏览器里跑 NumPy / OpenCV 不再是 demo。

---

## 🎉 全书完成

**8 章全部完成**。ch1-ch8 累计：约 **209 KB** markdown。

### 进度 8/8 ✅✅✅✅✅✅✅✅

| 章节 | 标题 | 状态 | KB |
|---|---|---|---|
| ch1 | Program Structure | ✅ | 20.1 |
| ch2 | Basic Data Types | ✅ | 22.3 |
| ch3 | Aggregates and Pointers | ✅ | 26.5 |
| ch4 | Storage Classes | ✅ | 24.8 |
| ch5 | Input and Output | ✅ | 26.3 |
| ch6 | Networking | ✅ | 31.7 |
| ch7 | Concurrency and Parallelism | ✅ | 29.6 |
| ch8 | Miscellaneous Topics | ✅ | 28.0 |

**Modern C: Up and Running (Kalin 2022) 精读完成**。所有笔记通过 `check_md_tables.py` + `check_code_blocks.py` 双自检，目录独立于 effective-c-2020/，README 索引、抽文缓存、`.cache/` 全齐。

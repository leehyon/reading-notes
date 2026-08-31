# Ch1. Program Structure

> 对应 PDF 物理页 18-49（印刷页 1-32）。本章是 C 程序"由函数拼装"的全景图：函数、main、汇编、控制流、命令行参数、变长参数。是后续 7 章的入口。

## §1 章节概述

1. **C 是"由函数拼装"的过程式语言**——程序 = 多个函数 + 唯一的 `main`。函数有声明（interface，无 body，以分号结尾）和定义（implementation，带 body）之分；声明可重复，定义必须唯一。
2. **`main` 是程序入口**——必须 `int main`（或带 `argc/argv`），返回 0/EXIT_SUCCESS 给 exec 系函数。C 风格：orthodox C（无嵌套函数定义，注释只用 `/* */`），本书为可移植性坚持 orthodox。
3. **C 函数 = 汇编可调用块（callable block）**——汇编没有"函数"，只有带 label 的 routine；参数传递通过 CPU 寄存器（x86-64 System V ABI：第 1 整数参数放 `%rdi`，返回值放 `%rax`）。`gcc -O1 -S hi.c` 即可生成 AT&T 语法汇编。
4. **预处理 → 编译 → 汇编 → 链接 四阶段**——`gcc --save-temps net.c` 同时保存 `.i` / `.s` / `.o`，便于逐步定位。
5. **命令行参数 `int main(int argc, char* argv[])`**——`argc ≥ 1`（`argv[0]` 是可执行文件名），`argv` 实际是 `char**`（指向指针数组的指针）。所有参数按字符串传入，需自己转 `atoi/strtol`。
6. **控制流三类**：test（`?: / if-else / switch`）、loop（`while / do-while / for`）、call。`break` 只能跳出一层循环；`switch` 的 `case` 一旦命中若无 `break` 会**顺序贯穿**到下一个 case（**高危点**）。
7. **call frame = 寄存器 + 栈**——被调函数用 `%rdi/%rsi/...` 收参，用 `%rax` 返回；`call` 压返回地址，`ret` 弹出。栈是 scratchpad，由系统自动管理。
8. **变长参数 `<stdarg.h>`**——`va_list` / `va_start(ap, last)` / `va_arg(ap, T)` / `va_end(ap)` 四件套；调用方**必须自己传 count 或 sentinel**（典型：`printf` 用格式串、`syscall` 用 SYS_* 编号），否则 `va_arg` 越界即 UB。

## §2 核心 Takeaways

### T1 — 声明（prototype）vs 定义（definition）不是一回事
- **是什么**：声明 = `int add2(int, int);`（分号结尾、无 body），描述调用约定；定义 = 同样签名但带 `{ ... }`，是函数实现。声明可重复，定义只能一次。
- **为什么重要**：跨文件调用时，编译器看到声明才能检查实参/形参类型匹配。**没有声明** 的话，C89 之前编译器默认 `extern int func()`（未声明参数类型），所有实参走默认提升——经典"传 int 实参当 double 用"坑。
- **解决什么**：C 头文件（`.h`）的天然用途就是装声明，`.c` 文件装定义。
- **适用场景**：任何超过 1 个 `.c` 文件的项目；任何用别人库（`#include <stdio.h>`）的代码。

### T2 — `int main` 不是装饰，`return 0` 会被 exec 系函数收走
- **是什么**：`main` 返回的 `int` 是给 OS 的 exit code。0 = 成功，非 0 = 失败（shell 里 `$?` 可读）。
- **为什么重要**：CI/CD、systemd、init 脚本都靠这个数字判定"是否成功"。C99 之后省略 `return` 隐式 `return 0`，但显式写更可读、更稳。
- **解决什么**：让脚本能 `if ./myprog; then ...; fi`，让 `make` 在失败时停止链。
- **适用场景**：CLI 工具、daemon、任何会被脚本/systemd 调用的程序。**嵌入式/裸 metal 上 `main` 通常 `void` + 死循环**——这是惯例，不是 C 强制。

### T3 — 汇编能帮你看穿 C 的"魔法"
- **是什么**：`gcc -O1 -S hi.c` 输出 `hi.s`（AT&T 语法：源在前目标在后、`%eax` 是寄存器前缀、`movl $0, %eax` 中 `l` = 32-bit longword）。
- **为什么重要**：函数调用约定、栈布局、寄存器传参——这些 C 教科书只讲"是什么"，汇编直接展示"怎么实现"。调 bug 看到"为啥这个指针变 NULL 了"，gdb 一回溯看到 call frame 立刻明白。
- **解决什么**：性能优化（看编译器到底生成了啥）、gdb 反汇编、跨语言 FFI（Rust/C++ 调 C 库要知道 calling convention）。
- **适用场景**：性能敏感的循环/数据通路、与汇编/FPGA IP 核对接的嵌入式代码、glibc/syscall 调试。

### T4 — `switch` 的 `case` 缺 `break` = 隐式 fallthrough
- **是什么**：进入 `case 0:` 后若无 `break`，会**按顺序继续执行** `case 1:`、`case 2:` 的语句，直到遇到 `break` 或 `switch` 结束。本书 ch1.6 用 `case 0:` 故意不带 `break` 来演示这个机制。
- **为什么重要**：**这是 C 里"看起来对、运行错"最高发的 bug 之一**。Linux kernel 用 `-Wimplicit-fallthrough` 显式标 `fallthrough;` 注释来压制告警。MISRA-C 直接禁用。
- **解决什么**：每个 `case` 末尾都加 `break`；多 case 共用一段代码时显式标 `/* fall through */`。
- **适用场景**：状态机里多个状态相同处理是合理用法，否则**一律加 `break`**；`-Wimplicit-fallthrough=5` 在 CI 里开。

### T5 — `for(;;)` 是合法的"无限循环"惯用法
- **是什么**：`for` 的 init/condition/post 三段都可空；`for(;;)` 等价 `while(1)`。本书 ch1.6 把它叫"obfuscated version"——能用 `while(1)` 就别用 `for(;;)`。
- **为什么重要**：嵌入式裸机 `for(;;) { ... }` 是惯例（main 返回也无处可返）；Linux kernel 里 `for (;;) { ... }` 也很常见。
- **解决什么**：需要无限循环的场合（event loop、main loop）。
- **适用场景**：RTOS main 循环、daemon event loop、内核线程入口。

### T6 — `va_list` 变长参数的可靠性靠"协议"
- **是什么**：`<stdarg.h>` 提供 `va_list` / `va_start(ap, last_named)` / `va_arg(ap, T)` / `va_end(ap)`。但 C 语言**不存参数个数**——必须靠调用方用某种方式（count、sentinel、format string）告诉被调方"还有几个"。
- **为什么重要**：`printf("%d %s", 42)` 漏了实参 → `va_arg` 读栈上垃圾 → 段错误或乱码。`syscall(SYS_chmod, "/path", perms)` 第一参 SYS_chmod 既是 syscall 编号也充当"协议锚点"。
- **解决什么**：库设计（自己写 `log_msg(level, fmt, ...)` 这类）、与 syscalls / 底层 API 对接。
- **适用场景**：写日志库、写 format-like 工具、glibc 内部大量使用。**裸机禁用**——va_list 行为依赖 ABI 实现细节。

### T7 — `argc ≥ 1`，不是 0
- **是什么**：`argv[0]` 总是可执行文件路径（POSIX 保证）。`./cline` 单独跑 → `argc=1`；`./cline a b` → `argc=3`。
- **为什么重要**：命令行参数解析的标准起点；`getopt()` / `argp` 库的合约。
- **解决什么**：CLI 工具的"参数检查"模式 `if (argc < 2) { usage(); return EXIT_FAILURE; }`。
- **适用场景**：所有 CLI 工具；尤其 ROS2 launch、systemd 单元调用外部程序时。

## §3 工程实践视角

### 3.1 Linux 系统开发视角

- **`gcc --save-temps` 是排查编译错误的核武器**——`net.i`（预处理后）能看到所有宏展开，`net.s`（汇编）能看优化效果，`net.o`（目标文件）能 `nm` / `objdump`。CI 流水线里如果"我的代码过了，发布版本挂了"——先看 `-O0` vs `-O2` 的 `.s` 差异，90% 是 UB 让优化器"激进"了。
- **`-Wall -Wextra -Werror -Wpedantic` 应是任何新项目的最小基线**；加上 `-Wshadow -Wstrict-prototypes -Wmissing-prototypes`，**`switch` 漏 `break` 会被 `-Wimplicit-fallthrough` 抓**。Linux kernel 还开 `-Wno-unused-parameter`（回调接口太多）。
- **`man` 是 Linux 上标准库的一手文档**——`man 3 printf` 看参数/返回值/线程安全（MT-Safe / MT-Unsafe 标记），`man 2 chmod` 看 syscall 直接调用。
- **C 标准 4 阶段管线的"陷阱地图"**：预处理（宏、头文件展开）→ 编译（语法/语义）→ 汇编（生成 `.o`）→ 链接（符号解析）。**未声明函数** 报的是链接错（implicit declaration warning，行为由 C99 起明确为 UB），不是编译错；**头文件多重包含** 报的是编译错（redefinition）；**库顺序** (`gcc main.c -lfoo`) 报的是链接错（undefined reference）。这三类错混在一起容易绕。
- **变长参数在 x86-64 上的实现**：System V ABI 把前 6 个整数实参放寄存器（`%rdi/%rsi/%rdx/%rcx/%r8/%r9`），多的才进栈。`va_start` 在这种 ABI 上"知道"去寄存器栈保存区读——但**这只是 x86-64 的实现**，跨平台（ARM、PowerPC）行为不同。**这就是为什么 `va_list` 程序要测在目标架构上**。

### 3.2 机器人软件视角（ROS2 / 嵌入式控制）

- **ROS2 节点的 `main` 就是 C 入口**——`rclcpp::init(argc, argv)` 接收的就是 C 的 `argc/argv`，要在 `main` 里直接拿。ROS2 client library 用 `rcl_ret_t` 错误码（不是 errno），返回 0/非 0 同理于 C main 返回值。
- **rclcpp::Node 的回调函数签名 = C 函数**——`void callback(const std::shared_ptr<msg::Type>&)`，本质是把 C++ 对象当 `void* user_data` 注入。ch1 讲的"声明/定义分离"在 ROS2 里是头文件 + .cpp 分离的硬约束（ament_cmake 强制）。
- **机器人控制循环就是 ch1.6 的 `for(;;)`**——典型结构：
  ```c
  int main(int argc, char* argv[]) {
    rclcpp::init(argc, argv);
    auto node = std::make_shared<MyController>();
    rclcpp::spin(node);   // 内部是 while(rclcpp::ok()) { rclcpp::spin_some(); }
    rclcpp::shutdown();
    return 0;
  }
  ```
  `rclcpp::spin` 内部就是 ch1.7 讲的"call/return 协议"的 RTOS 化实现——回调就是一个 `void*` + 任意签名的"event handler"。
- **机械臂/AGV 的实时循环 = `while(1)` + 周期 sleep**——ch1.6 的 `while (1) { ...; sleep_until(next); }` 模式是 90% 机器人控制器的骨架。`clock_gettime(CLOCK_MONOTONIC)` 算时间漂移（PID 控制器最怕 jitter）。
- **ROS2 launch 文件本质是把 CLI 参数注入节点 main**——`<node pkg="..." exec="..." args="--rate 100 --topic /cmd_vel" />`，对应 ch1.5 的 `argv[1..]`。**`getopt` 解析的位置参数 vs 命名参数** 在 ROS2 里被 `ros2 param` 替代了。
- **机器人 SLAM/navigation 节点 vs 实时控制节点**——前者可容忍秒级延迟（network pub/sub、消息队列），后者必须 1 kHz。**ch5 命名管道 / ch7 共享内存 / 消息队列** 的选用就是 ch3+ch5+ch7 的综合题。

### 3.3 初级 vs 高级工程师对照

| 习惯 | 初级 | 高级 |
|---|---|---|
| 函数位置 | 所有函数堆一个 `main.c` 2000 行 | 头文件放声明、`-Wall -Werror`、每个 `.c` < 200 行 |
| 入口校验 | 不检查 `argc` | `if (argc != 2) { fprintf(stderr, "Usage: ...\n"); return EXIT_FAILURE; }` |
| `switch` 写法 | 漏 `break`，靠运气 | 每个 case 末尾 `break`，CI 开 `-Wimplicit-fallthrough=5` |
| 变长参数 | `va_arg` 凭感觉 | 协议锚点（count/sentinel/format），被调方 + 调用方成对维护 |
| 编译命令 | `gcc foo.c` | `gcc -std=c11 -O2 -Wall -Wextra -Wpedantic -Werror foo.c -o foo` |
| 调试遇错 | 加 `printf` | `gcc -g -O0` + gdb + `objdump -d` 看汇编 |

## §4 AI 时代视角

### 4.1 这些知识还重要吗？（2026 年视角）

**重要。** 这是 C 的"操作系统级基础设施"——所有 C/C++/Rust 编译器、所有 OS kernel、所有数据库、所有高性能中间件（Redis/Nginx/ClickHouse）、所有嵌入式 MCU 固件都依赖 ch1 这套规则。AI 工具箱（PyTorch/TensorFlow 的 C++ 后端、vLLM 的 CUDA C++ runtime、Claude Code 的 Go+Rust 外壳——其中 Go 和 Rust 都要跟 C ABI 对接）也吃这套。

**但重要性已从"必须手写"变成"必须能读懂"。** 现代工程师的 ch1 日常：
- **生成代码** = 几乎都靠 AI/LLM 完成
- **看代码** = 必须能读懂 AI 生成的 C，包括 ch1 全部约定
- **修代码** = 当 AI 生成"看起来对"的 C 出 bug 时，**唯一能快速定位的就是 ch1 这套机制**

### 4.2 AI 现在能做的

- ✅ **写 ch1 全部 9 个 Listing 的任意变体**（add2, hello world, cline, tests, whiling, varArgs）—— 100% 准确
- ✅ **写 `Makefile` / `CMakeLists.txt` / `meson.build`**——ch1 末尾"gcc -o hi hi.c"扩到 build system 毫无难度
- ✅ **生成"教科书版"控制流**——`if/else/switch/while/for` 都能写
- ✅ **写规范的 `int main(int argc, char* argv[])`**——绝大多数情况
- ✅ **解释 C 调用约定**——x86-64 System V ABI 烂熟

### 4.3 AI 经常写错的地方（必看）

| 错误模式 | 例子 | 后果 |
|---|---|---|
| **1. `switch` 漏 `break`** | `case 1: puts("a");` 后面没 break | 隐式 fallthrough 到 case 2，bug 极难复现 |
| **2. `void main()` 滥用** | `void main() { return; }` | 编译器警告；嵌入式裸机可接受，host 程序是 anti-pattern（AI 不区分语境） |
| **3. 隐式函数声明** | 调用 `syscall` 忘了 `#include <unistd.h>` 和 `<sys/syscall.h>` | C99 起明确 UB；C23 已删除隐式声明规则；AI 经常漏 include |
| **4. `argv[i]` 不检查 i** | `argv[1]` 当字符串直接 `atoi`，没看 `argc` | 用户少传参数就段错误 |
| **5. `va_arg` 类型不匹配** | 实际传 `double` 但写 `va_arg(ap, int)` | 静默读到垃圾数据；x86-64 上 double 走 XMM 寄存器，int 走通用寄存器，行为更乱 |
| **6. 注释混用 `//` 和 `/* */`** | Kalin 强调 orthodox C 只用 `/* */`；AI 默认 C++ 风 | C89 编译器直接挂；混用时 `/*` 内的 `*/` 还会截断注释 |
| **7. `char*` vs `char*[]` 模糊** | `int main(int argc, char* argv[])` 写成 `char** argv` 是等价的但 AI 风格不统一 | 团队代码风格割裂；某些静态分析器会 warn |
| **8. `for(;;)` 死循环内忘记 yield** | 高频循环里没 `usleep/clock_nanosleep` | 单核占满、其他线程饿死；实时系统 deadline miss |
| **9. `main` 返回值乱用** | 错误路径 `return -1` 而不是 `EXIT_FAILURE` | 数字大小平台相关；shell `$?` 在不同 OS 取值范围不同；C 标规定 `unsigned char` 内，-1 会变 255 |
| **10. `int g() {...}`（K&R 旧式声明）** | 老代码风格 | C23 已删除 K&R 风格；AI 偶尔会模仿老 Stack Overflow 代码 |

### 4.4 工程师必须保留的核心能力

- **能读 `gcc -S` 出的汇编**——AI 优化错时（UB 让优化器"自由发挥"），只有看汇编能定位。
- **能 gdb 反汇编 + 调 call frame**——AI 写"对"的并发代码出 race 时，必须用 gdb 看线程栈。
- **能区分 orthodox C vs modern C**——维护老 codebase 时这点关键。
- **能写规范的 `.h` 头文件**（include guards / `extern "C"` for C++ 兼容）——AI 写头文件时经常漏 include guard。
- **能在 ch1 全部机制层面跟 AI 协作**——"请按 orthodox C 风格写" / "请用 `for(;;)` 写主循环" / "请在每个 case 末尾加 break"——**精确的 ch1 词汇**是 prompt 的关键。

### 4.5 本书后续章节里的 wasm 视角（用户已选需要延伸）

- **ch8.5 的 WebAssembly 块** —— 实际是把 ch1 的 C `main` 经 `emcc` 编译成 wasm，**LLVM 后端** 把 ch1 全部机制映射到 wasm 字节码。
- **AI 工具链中 wasm 的角色**：Claude Code / Codex 等 CLI agent 的部分 sandbox 机制（QuickJS、Wasmtime）就是用 wasm 跑用户代码——ch1 的 call/return 协议在这里变成 **wasm 的 call_indirect / return_call 指令**。
- **可延伸的"AI 视角"问题**（ch8 笔记会展开）：wasm 是否能替代 libc 全部？wasm 的"安全内存模型"和 C 的指针算术怎么调和？

## §5 实践行动项

### A1 — 编译 + 改写 ch1 全部 Listing，亲眼看汇编
```bash
mkdir -p /tmp/modern-c/ch01 && cd /tmp/modern-c/ch01
# 1. 写 add2.c（Listing 1-3）
cat > add2.c <<'EOF'
#include <stdio.h>
int add2(int n1, int n2) { return n1 + n2; }
int main() { printf("%i + %i = %i\n", -26, 44, add2(-26, 44)); return 0; }
EOF
gcc -O1 -S add2.c -o -   ## 看汇编
gcc -std=c11 -O2 -Wall -Wextra -Wpedantic -Werror -o add2 add2.c && ./add2
```
**验收**：能看到 `add2:` / `main:` 标签、`%edi` / `%eax` 寄存器名；编译无 warning。

### A2 — 用 `-Wimplicit-fallthrough=5` 抓 switch 漏 break
```bash
cat > switchbug.c <<'EOF'
#include <stdio.h>
int main(int argc, char** argv) {
    int r = argc - 1;          /* 模拟 */
    switch (r) {
    case 0: puts("zero");
    case 1: puts("one");  /* 故意漏 break */
    case 2: puts("two");  break;
    default: puts("other");
    }
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -Wpedantic -Wimplicit-fallthrough=5 -c switchbug.c
```
**验收**：编译应报 `case 0: ... warning: this statement may fall through`；删 break 加 `[[fallthrough]];`（C23） 或 `/* fall through */` 注释后警告消失。

### A3 — 写带 `argc/argv` 校验的 CLI（防 argv[i] 越界）
```bash
cat > cline.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
int main(int argc, char* argv[]) {
    if (argc < 2) {
        fprintf(stderr, "Usage: %s <one or more args>\n", argv[0]);
        return EXIT_FAILURE;
    }
    for (int i = 0; i < argc; i++) puts(argv[i]);
    return EXIT_SUCCESS;
}
EOF
gcc -std=c11 -Wall -Wextra -Werror -o cline cline.c
./cline a b c; echo "exit=$?"; ./cline; echo "exit=$?"
```
**验收**：缺参数时 exit=1、有参数时 exit=0、`argv[0]` 始终是 `./cline`。

### A4 — 用 `va_list` 写一个"加 count 协议"的 `sum` 函数
```bash
cat > varargs.c <<'EOF'
#include <stdarg.h>
#include <stdio.h>
double avg(int count, ...) {
    if (count <= 0) return 0.0;
    double sum = 0.0;
    va_list ap;
    va_start(ap, count);
    for (int i = 0; i < count; i++) sum += va_arg(ap, int);
    va_end(ap);
    return sum / count;
}
int main(void) {
    printf("%.2f\n", avg(4, 1, 2, 3, 4));
    printf("%.2f\n", avg(0));   /* 边界 */
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -O2 -o varargs varargs.c && ./varargs
```
**验收**：`avg(4,1,2,3,4)=2.50`；`avg(0)=0.00`；故意 `avg(2, 1, 2, 3, 4)`（多传一个）会算错——**这正是"协议错"的演示**。

### A5 — 用 `gcc --save-temps` 看 4 阶段产物
```bash
cat > net.c <<'EOF'
#include <stdio.h>
#define PI 3.14159
int main() { printf("PI=%f\n", PI); return 0; }
EOF
gcc --save-temps -O1 net.c -o net
ls -la net.*   ## 应有 net.i（预处理）net.s（汇编）net.o（目标）net（可执行）
head -20 net.i ## 看到宏展开、头文件全展开
head -20 net.s ## 看到 AT&T 汇编
```
**验收**：能说清 `.c → .i → .s → .o → 可执行` 各阶段产物；能用 `objdump -d net` 看到 main 函数汇编。

## §6 值得深入思考的问题

1. **声明 vs 定义的边界在哪？** `static inline` 既是声明也是定义（每个 .c 自己一份），C99/C11 允许这样放宽"一次定义"规则。如果用 `static inline` + 不同 .c 给同一个 inline 函数不同实现，链接器会怎么选？为什么？
2. **`main` 能不能递归？** 语法上 `int main() { main(); }` 合法，运行时栈会无限增长直到段错误。但 `fork` 之后的子进程里再调 `main` —— exec 体系下还成立吗？（提示：exec 后 PID 不变还是变？）
3. **AT&T 汇编的"源在前"约定对优化器意味着什么？** Intel 语法是 `mov dst, src`，AT&T 是 `mov src, dst`。编译器后端输出哪种语法由谁决定？为什么 Linux 内核和 glibc 默认 AT&T？
4. **`switch` 的 fallthrough 在状态机里是 feature 还是 bug？** TCP 状态机、SMTP 响应解析、protobuf 字段 tag 状态机——都利用 fallthrough 表达"多个 state 同一处理"。如果用 `if-else` 链等价改写，会不会破坏可读性？性能有差吗？
5. **`for(;;)` vs `while(1)` vs `do { } while(1)` 编译后真的一样吗？** 写个简单循环，分别用三种写法 + `-O2 -S` 看汇编。如果汇编完全相同，是不是说"风格选择"纯粹是文化习惯？
6. **变长参数函数的 ABI 稳定性**：x86-64 上 `va_list` 走寄存器+栈保存区，ARM AAPCS64 走完全不同的规则（最多 8 个整型实参寄存器）。如果你的库要给 C/Python/Rust 三种语言做 FFI，`va_list` 是不是该禁？改用什么？

---

*下一章预告*：**ch2 Basic Data Types**——int 的 2's complement 假设、integer overflow 的有符号 UB 行为、IEEE 754 浮点的 5 类异常、`%lu`/`%f` 的格式化坑。这是 Effective C ch3 (arithmetic types) 的 Kalin 版——重点会放在"嵌入式浮点 vs 主机浮点"的差异、ROS2 节点为什么 `costmap` 必须用 `float` 而不是 `double`（带宽 + GPU 内存）。

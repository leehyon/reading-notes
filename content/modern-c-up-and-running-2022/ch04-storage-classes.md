# Ch4. Storage Classes

> 对应 PDF 物理页 151-166（印刷页 135-150）。本章 16 页短章，**4 个存储类说明符 + 2 个类型限定符**：`extern`（跨文件）/ `static`（持久化/文件作用域）/ `auto`（默认栈局部）/ `register`（建议寄存器）+ `volatile`（防优化器消除读）+ `const`（顺带）。**`static` 函数 = C 的 "private"；`volatile` 是嵌入式 ISR、多线程共享变量、MMIO 的生命线**。

## §1 章节概述

1. **存储类 = 作用域 × 生命周期 × 链接性**——C 标准 4 个 specifier 决定变量"放在哪、活多久、谁能看见"。函数只有 `extern` / `static` 两选项；变量 4 个全可选。
2. **默认规则**：函数 = `extern`（除非标 `static`）；块外变量 = `extern`；块内变量 = `auto`。`register` 早已过时（C++17 deprecated，C 仍保留但**不优化**），现代编译器开 `-O1` 比 `register` 强。
3. **`auto` 与 `register` 已成历史**——Kalin 明说"the specifier is almost never used——except for demonstration purposes"。所有"局部变量"实际就是 auto（即使没写 `auto`）。
4. **`static` 的两副面孔**：
   - **块内 static** = 函数内"持久化局部变量"——值跨调用保留（**不重初始化**），但作用域仍是块（局部不可见）
   - **块外 static** = 文件作用域全局变量——**但**只在定义文件内可见（**内部链接**）
5. **`static` 函数 = C 的"private"**——文件外不可见，避免命名冲突、限制 API 表面。Linux kernel `static` 函数 + 头文件 = 模块封装。
6. **`extern` 的两步规则**：① 在某 .c 文件**定义**（定义时不写 `extern`，初始化与否均可）；② 在其他 .c 文件**声明**（必须 `extern` 关键字，**不能初始化**）。**"定义一次，声明多次"** 是 ODR（One Definition Rule）。
7. **`volatile` = 告诉编译器"别优化"**——每次访问都从内存读、每次写都立即写回。**典型场景**：ISR 与主程序共享变量、多线程共享变量、MMIO 硬件寄存器。**编译优化可能让 bug 静默发生**（"代码看上去对，运行错"）。
8. **`const` = 编译器静态承诺不变**——`const int* p` 是"指针可变、指向数据不可变"；`int* const p` 是"指针不可变、指向数据可变"；`const int* const p` 是"都不可变"。**`const` 修饰的是右边的"那个东西"**（从右往左读）。
9. **`const` 可被强制 cast 掉**——`int* p = (int*)&n;` 编译器**只 warning 不 error**；运行时能改值。**这是 C 的"君子协定"**——你违反 const 编译器拦不住。**C++ `const_cast` 行为更严**。
10. **`-pg + gprof` 是经典性能 profiling 工具**——ch4 末尾 sidebar 提到，但现代用 `perf` / `callgrind` / `perfetto` 替代。**AI 时代的"性能工程师"会读 perfetto 火焰图**。
11. **`-g` + `gdb` 是调试基线**——`-O0` 编译保留所有变量、便于单步。**生产二进制 strip 后 gdb 看不了符号**——`objcopy --only-keep-debug` 留 debug info。
12. **`volatile` 不是存储类**——C 标准明确说"qualifier, not a storage class"。**它不影响变量放在哪**，只影响编译器**对读写的优化策略**。**新坑**：`const volatile int* reg` 是合法组合（MMIO 寄存器：值会变但你不该主动写）。

## §2 核心 Takeaways

### T1 — `static` 函数 = 文件作用域；模块化的 C 手段
- **是什么**：`static int helper()` 定义的函数只在当前 `.c` 文件内可见。**链接器不会把它暴露给其他目标文件**。
- **为什么重要**：**避免命名冲突**、**隐藏实现细节**、**编译器可激进 inline 优化**。Linux kernel 的 `static` 函数 + 头文件 API 是 C 唯一可行的"模块化"模式（无 C++ namespace）。
- **解决什么**：库的内部实现封装、避免全局符号污染。
- **适用场景**：所有超过 1 个 .c 文件的项目；尤其**库代码**（`static` helper + 公开 API）。

### T2 — `static` 块内变量 = 函数级"单例"
- **是什么**：`void counter() { static int n = 0; n++; }` —— `n` 初始化**只发生一次**（首次调用），后续调用 `n` 保留值。
- **为什么重要**：**没有 GC 的 C 实现单例**的最简单方式。Kalin ch4.4 用它做"调用计数器"（`profile` 程序）。
- **解决什么**：状态机、缓存、计数器、延迟初始化。
- **适用场景**：logger、限流器、LRU 缓存。**线程安全版本** 要加 mutex（C11 `<threads.h>` `mtx_lock`）或 `pthread_mutex_t`。
- **坑**：`static` 变量**不是 `const`**——可改；**默认初始化为 0**（BSS 段）；**作用域仍是块内**——其他函数看不见。

### T3 — `extern` 是 ODR 的"链接"机制
- **是什么**：`extern` 声明 = "这个符号在别处定义"。C 链接器用**符号表**把声明和定义配对。
- **为什么重要**：多文件项目必备；理解它才能看懂 `gcc -c` 后的 `undefined reference to` 错误（声明了但没定义）。
- **解决什么**：跨文件全局变量共享、跨文件函数调用。
- **适用场景**：所有 C 项目；尤其**共享内存 IPC**（`extern volatile uint32_t* shm_ptr`）、**FFI 跨语言**。
- **规则**：定义一次（无 `extern` 或有 `extern` + 初始化），声明 N 次（`extern` + **不**初始化）。

### T4 — `volatile` 是嵌入式 / 多线程的"防优化器"保险
- **是什么**：`volatile int n` 告诉编译器：每次读 `n` 都从内存加载；每次写 `n` 都立即刷回。**编译器不能把 `n` 缓存到寄存器**。
- **为什么重要**：
  - **ISR 共享变量**：主循环 `while (!flag);` 若 `flag` 没标 volatile，**编译器会把 flag 缓存到寄存器**，主循环**永远看不到 ISR 改的 flag**。
  - **多线程共享变量**：thread T1 写、T2 读；T2 的读若不 volatile，**编译器把值缓存到寄存器，T2 永远看不到 T1 的写**。
  - **MMIO 硬件寄存器**：`*(volatile uint32_t*)0x40021000` 每次都触发总线访问；**没 volatile 优化掉 → 硬件不动**。
- **解决什么**：硬件寄存器访问、并发同步、信号 handler 共享数据。
- **适用场景**：所有嵌入式 MCU 代码、所有多线程共享变量、kernel driver。
- **误用**：普通局部变量**不要**乱加 `volatile`——会让所有优化失效，性能暴跌。

### T5 — `const` 是"右往左读"规则
- **是什么**：`const int* p` = 指向 const int 的指针（**数据不可变，指针可变**）；`int* const p` = const 指针指向 int（**指针不可变，数据可变**）；`const int* const p` = 都不可变。
- **为什么重要**：C 库 API 全用 `const` 标识"我不会改你"（`strlen(const char*)`、`printf(const char* fmt, ...)`）。**读错**就会写出"看似安全实则可改"或"看似可改实则改不了"的 bug。
- **解决什么**：API 文档、编译器辅助检查、const-correctness。
- **适用场景**：所有库函数参数；尤其 `qsort` 的 `comp(const void*, const void*)`。
- **高级**：`char* const *` (指针的指针，**第二级**指针不可变)；`const char* const * argv` (main 函数的 argv)。

### T6 — `register` 在 2026 年是反模式
- **是什么**：`register int n` 建议编译器"把 n 放 CPU 寄存器"。
- **为什么重要**：C89 时代有用（编译器不做优化），现在**编译器做的寄存器分配比程序员手写好**。**`register` 关键字还禁止取地址**（`&n` 编译错），进一步限制。
- **解决什么**：无（仅历史）。
- **适用场景**：**完全不要用**。开 `-O1` 或更高即可。`register` 在 C++17 已 deprecated。

### T7 — `auto` 也是反模式
- **是什么**：`auto int n` 与 `int n` 等价（块内变量默认 auto）。
- **为什么重要**：C++11 `auto` 是完全不同的语义（类型推导）。**C 用 `auto` 会让 C++ 程序员迷惑**。
- **解决什么**：**绝不要在 C 代码里写 `auto`**。C++ 项目里也别用 C 风格的 `auto int n`。

### T8 — `static` 全局变量 = 内部链接全局；C 版的"模块私有状态"
- **是什么**：`static int g_state;` 在 .c 文件顶层定义。**作用域**：本文件；**生命周期**：程序全程；**链接性**：内部（链接器不暴露）。
- **为什么重要**：比 `extern` 全局安全（不污染全局符号）；比函数内 `static` 可见（其他函数可访问）。
- **解决什么**：模块私有状态、单例延迟初始化。
- **适用场景**：库的全局 cache（如 Redis 的 dict 状态）、driver 状态机。
- **多文件注意**：每个 .c 文件可以有**同名**的 `static` 全局（互不干扰），但 `extern` 全局**不能同名**。

### T9 — 调试与 profiling 基线
- **是什么**：`-g` 给 gcc 加 DWARF 调试信息；`-O0` 关闭优化保留所有变量；`gdb` 设断点/单步/查看；`valgrind` 抓内存；`perf` 抓 CPU。
- **为什么重要**：CI 调试效率、生产环境 crash dump 解读、性能瓶颈定位。
- **解决什么**：bug 定位、性能优化、内存安全。
- **适用场景**：所有项目。

## §3 工程实践视角

### 3.1 Linux 系统开发视角

- **Linux kernel 全用 `static` 内部函数 + 头文件 API**：
  - `static int my_module_init(void)` 只在本 .c 可见
  - `EXPORT_SYMBOL(my_module_api)` 把符号注册到全局符号表
  - 头文件只放**声明**（`extern`）
- **`-fvisibility=hidden` 编译选项**——把**默认**链接性从 `extern` 改成 hidden；只有 `__attribute__((visibility("default")))` 的才暴露。比每个函数加 `static` 干净。
- **共享库版本控制** = `__attribute__((visibility("default"))) + 版本脚本`——`extern` 符号管理 ABI。
- **多线程共享变量的 `volatile` 是 2026 年的过时建议**——**正确做法是 `std::atomic` (C++) / `<stdatomic.h>` (C11)**。`volatile` 在 C++ 不保证原子性、不保证内存序、不能替代 mutex。
  ```c
  /* C11 正确做法 */
  #include <stdatomic.h>
  atomic_int flag = 0;  /* 取代 volatile int flag */
  /* 写: atomic_store(&flag, 1); */
  /* 读: int v = atomic_load(&flag); */
  ```
  C11 `atomic` 编译成 `lock cmpxchg` / `xchg` 等原子指令；`volatile` 只是"防优化"，不保证多核可见性（**需要 mfence 才行**）。
- **`-pg + gprof` 已被 `perf record` + `perf report` 替代**——后者能看 cache miss、branch mispredict、CPU 迁移。
- **gdb 多线程调试** = `gdb -p PID` attach 到进程；`info threads` 看所有线程；`thread N` 切换；`bt` 看栈；`watch var` 硬件断点。
- **`nm` 查符号表** = `nm prog | grep ' T '` 看 extern 函数；`nm prog | grep ' [bBdD] '` 看 static 数据。
- **`objcopy --only-keep-debug prog prog.debug`** 分离 debug info——生产 binary 体积减小，gdb 用 `file prog + add-symbol-file prog.debug` 仍能调试。

### 3.2 机器人软件视角（ROS2 / 嵌入式控制）

- **ROS2 node 全用 `static` 全局 = 单例管理器**——`rclcpp::Node::SharedPtr g_node` 不可能多实例（ROS2 一个进程一个 node 树）。
- **MCU 固件 100% 用 `volatile`**——STM32 HAL 库所有寄存器 `__IO` 宏 = `volatile`。`#define __IO volatile`。**没 `volatile` 的嵌入式 C = 几乎一定不工作**。
- **ISR 与主循环共享 flag**：
  ```c
  volatile uint8_t data_ready = 0;
  void EXTI0_IRQHandler(void) {  /* STM32 ISR */
      data_ready = 1;            /* 主循环看到 */
  }
  int main(void) {
      while (!data_ready);  /* 阻塞等待 */
      process_data();
  }
  ```
  **`volatile` 是必需的**；但**多字节数据还需关中断保护**——`volatile` 不保证多字节读写的原子性。
- **RTOS 共享内存 IPC**：`extern volatile uint32_t* shm_ptr` 在 shm_open + mmap 后跨进程访问。**正确做法**：C11 atomic + 内存屏障。`volatile` 只能防"被优化掉"，**不能防** CPU 流水线重排。
- **ROS2 message field 全用 primitive types**——`int32`、`float32`、`uint8[]` 显式宽度（ch2 重点）；**不是 `static`，是 ROS2 IDL 编译生成的 C 结构体字段**。
- **ros2_control hardware interface 状态机** = `static` 全局 FSM + mutex 保护：
  ```c
  static JointState g_joint_state;
  static pthread_mutex_t g_mtx = PTHREAD_MUTEX_INITIALIZER;
  void read_state(void) {
      pthread_mutex_lock(&g_mtx);
      /* 读传感器、更新 g_joint_state */
      pthread_mutex_unlock(&g_mtx);
  }
  ```
- **MoveIt 路径规划** = `static const` 大型规划器参数表（IK 求解器、避障阈值）——`static const` 放 .rodata 段，**不可改、cache 友好**。
- **`const char*` vs `char* const` 在 ROS2 logging 里** = ROS2 全部用 `const char*`（消息内容不可变）；**logger 的格式化模板都是 `const char*`**。

### 3.3 初级 vs 高级工程师对照

| 习惯 | 初级 | 高级 |
|---|---|---|
| 跨文件变量 | 直接在头文件定义（非 `extern`） | 头文件 `extern` 声明 + .c 文件定义 |
| 函数可见性 | 所有函数都是 `extern` | 模块内 helper 全 `static`；用 `-fvisibility=hidden` |
| 嵌入式共享变量 | `int flag;` | `volatile int flag;`（单字节）/`atomic_int flag;`（多线程） |
| const 位置 | `const int p` 写顺手 | "右往左读" + `int const * p`（数据不可变）vs `int * const p`（指针不可变） |
| `register` / `auto` | 偶尔写 | **永远不写**——开 `-O1` 就够 |
| `static` 局部变量 | 怕"用全局变量"而不敢用 | 当"函数级单例"用；清晰注释用途 |
| 调试 | printf 大法 | `-g` 编译 + gdb + perf + valgrind 组合 |
| profiling | `time ./prog` | `perf stat ./prog`（cache miss、IPC、branch） |

## §4 AI 时代视角

### 4.1 这些知识还重要吗？（2026 年视角）

**重要但弱化**。`static` / `extern` 是 C 的"基础工程语义"，AI 写 C 时代码 99% 会正确使用；**`volatile` 是 2026 年最容易被误用的关键字**——LLM 经常在不该用 volatile 的地方加（在已 atomic 的代码上再加 volatile），或在该用 `atomic` 的地方用 `volatile`（多线程共享 `int`）。

现代工程师的 ch4 日常：
- **review AI 生成的 C 代码** = 检查 `static` 边界、跨文件 extern 声明、`volatile` 是否滥用
- **嵌入式 MCU 代码** = 必查 ISR 共享变量 `volatile`（AI 经常漏）
- **多线程代码** = 必查是否用 `<stdatomic.h>`（AI 经常回退到 `volatile`）

### 4.2 AI 现在能做的

- ✅ 解释 4 个 storage class 的差异
- ✅ 写 `static` 函数、`extern` 跨文件
- ✅ 解释 volatile 适用场景
- ✅ 写 `-O1 -g` 编译命令
- ✅ 解释 const 三种位置

### 4.3 AI 经常写错的地方（必看）

| 错误模式 | 例子 | 后果 |
|---|---|---|
| **1. 多线程共享用 `volatile` 不够** | `volatile int counter; ++counter;` | volatile 不保证原子性，++ 不是 atomic；可能丢失更新；用 `atomic_int` |
| **2. 嵌入式 ISR 漏 `volatile`** | `int data_ready; void ISR() { data_ready=1; } while(!data_ready);` | 编译器把 data_ready 缓存到寄存器，主循环死循环 |
| **3. `const` 位置错** | `int* const p = ...; p = q;` 想"指针不可变"——对；但 `*p = 5;` **可以** | 想锁数据用 `const int* p`；想锁指针用 `int* const p` |
| **4. `static` 函数当 extern 用** | .c 文件 `static void helper();`，头文件也声明它 | 链接错 `undefined reference`；**static 函数不能跨文件** |
| **5. 头文件定义非 `extern` 变量** | `int g_count = 0;`（在 .h） | 每个 include 它的 .c 都有一份 g_count，链接多重定义错；**头文件只放 `extern` 声明** |
| **6. `register` + `&`** | `register int n; scanf("%i", &n);` | 编译错（C 标准禁止 `&register`） |
| **7. `volatile` 滥用** | 局部变量加 `volatile`，编译器无法优化 | 性能暴跌 10-100× |
| **8. `extern` 当函数定义** | `extern void f() { /* body */ }` | 合法但**强烈不推荐**——Kalin 明说"never use `extern` in definition"；函数定义默认就是 extern |
| **9. `static` 在头文件** | `static int g_helper;`（在 .h） | 每个 include 它的 .c 都有自己一份 g_helper；不报错但**逻辑错** |
| **10. 调试 -O0 -g 漏** | 生产 binary 没 `-g` | gdb 看不到局部变量、栈不符号化；线上 crash dump 难分析 |
| **11. `const_cast` 滥用** | `int* p = (int*)&n; *p = 99;` 改 const | 编译 warning 但能改；**UB if n 真的在 .rodata**；嵌入式 MCU 改 const 真值会导致 HardFault |
| **12. `volatile` 不加 barrier** | `volatile int ready; while(!ready);` 在 x86 OK，在 ARM/POWER 不行 | x86 是 TSO 内存模型；ARM 需要 `__sync_synchronize()` 或 `atomic_thread_fence` |

### 4.4 工程师必须保留的核心能力

- **能在 5 秒内判定"该用 `static` 还是 `extern`**——helper → static；跨文件 → extern。
- **能 30 秒内识别 volatile 滥用**——"多线程 + `int`" = 用 atomic；"ISR 共享 + `int`" = 用 volatile；"硬件寄存器" = `volatile` + `__atomic` 不需要。
- **能 gdb 调试复杂多线程程序**——`watch` / `info threads` / `bt full`。
- **能用 `nm` 查符号表**——`U`（undefined，缺定义）/ `T`（text，extern 函数）/ `t`（text，static 函数）/ `B`（BSS，未初始化全局）/ `D`（data，已初始化全局）。
- **能跟 AI 说"多线程用 `<stdatomic.h>`、嵌入式 ISR 用 `volatile`、`-fvisibility=hidden` 减少符号暴露"**——ch4 词汇是 prompt 的关键。

### 4.5 wasm 工具链延伸（用户已选需要）

- **wasm 几乎不需要 `volatile`**——wasm 规范保证**单线程**（除非用 SharedArrayBuffer + Atomics），且 wasm 编译器（LLVM/emscripten）不做 host 那种激进寄存器优化。**ch4.6 的 volatile 在 wasm 里意义有限**。
- **wasm 的 `static` 与 host 一样**——但 **wasm 里的 `static` 变量可能在 wasm linear memory 的 .data 段**——emcc 编译时处理。
- **wasm 的 `extern` 跨"模块"**——`emscripten` 生成的 wasm 模块之间通过 `EM_JS` / `EM_ASM` 宏做 JS ↔ C 互调；**C 内部跨 .c 文件仍用 `extern`**。
- **AI 工具链中 wasm 的 volatile 角色**：QuickJS + WASI 跑用户 C 代码时（Claude Code / Codex 沙箱）——**不需要 volatile**（单线程、线性内存模型简单）。**AI 经常误加 volatile 反而增加 wasm 字节码大小**。

## §5 实践行动项

### A1 — 验证 `static` 块内变量的"持久化"
```bash
mkdir -p /tmp/modern-c/ch04 && cd /tmp/modern-c/ch04
cat > static_local.c <<'EOF'
#include <stdio.h>
void counter(void) {
    static int n = 0;   /* 只初始化一次 */
    n++;
    printf("n = %d\n", n);
}
int main(void) {
    counter(); counter(); counter();   /* 输出 1, 2, 3 */
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -o static_local static_local.c && ./static_local
```
**验收**：输出 1, 2, 3；删 `static` 后输出全 1（重置）。

### A2 — `extern` 跨文件（ch4.5 实战）
```bash
cat > file1.c <<'EOF'
#include <stdio.h>
int g_count = 0;            /* 定义：不写 extern */
void bump(void) { g_count++; printf("file1: %d\n", g_count); }
EOF
cat > file2.c <<'EOF'
#include <stdio.h>
extern int g_count;         /* 声明：必须 extern，不初始化 */
void show(void) { printf("file2: %d\n", g_count); }
EOF
cat > main.c <<'EOF'
extern void bump(void);
extern void show(void);
int main(void) { bump(); bump(); show(); return 0; }
EOF
gcc -std=c11 -Wall -Wextra -o cross_extern file1.c file2.c main.c && ./cross_extern
```
**验收**：输出 `file1: 1 / file1: 2 / file2: 2`；`g_count` 在三处共享同一份。

### A3 — `volatile` 防优化器消除读（嵌入式模拟）
```bash
cat > volatile_needed.c <<'EOF'
#include <stdio.h>
#include <signal.h>
#include <unistd.h>
volatile int g_signal = 0;     /* volatile 必须 */
void handler(int sig) { g_signal = 1; }
int main(void) {
    signal(SIGUSR1, handler);
    raise(SIGUSR1);            /* 立即触发 */
    while (!g_signal);         /* 等 ISR 改 */
    printf("ok\n");
    return 0;
}
EOF
gcc -std=c11 -O2 -Wall -Wextra -o v_yes volatile_needed.c
sed 's/volatile //' volatile_needed.c > volatile_no.c
gcc -std=c11 -O2 -Wall -Wextra -o v_no volatile_no.c
timeout 2 ./v_yes; echo "v_yes exit=$?"
timeout 2 ./v_no; echo "v_no exit=$? (会被 SIGALRM 杀掉 = 死循环)"
```
**验收**：`v_yes` 立即输出 `ok`；`v_no` 2 秒后被 timeout 杀（**死循环** = 优化器把 g_signal 缓存到寄存器，main 看不到 ISR 改的值）。这是 `volatile` 必要性的活证据。

### A4 — `const` 三种位置的语义对比
```bash
cat > const_3ways.c <<'EOF'
#include <stdio.h>
int main(void) {
    int a = 1, b = 2;
    const int* p1 = &a;    /* 指向 const int 的指针：数据不可变 */
    int* const p2 = &a;    /* const 指针：指针本身不可变 */
    const int* const p3 = &a;  /* 都不可变 */
    (void)p3;               /* 避免 -Werror unused-variable */
    /* *p1 = 5; */        /* 编译错 */
    p1 = &b;               /* OK */
#if 0
    p2 = &b;               /* 编译错（放在 #if 0 仅为展示） */
#endif
    *p2 = 5;               /* OK */
    /* *p3 = 5; p3 = &b; */  /* 都错 */
    printf("a=%d *p2=%d *p1=%d\n", a, *p2, *p1);
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -Werror -o const3 const_3ways.c && ./const3
```
**验收**：编译过（注释掉的错行）；输出 `a=5 *p2=5 *p1=2`（a 被 *p2 改成 5，但 p1 仍指向旧 a=1，p1 现在指向 b=2）。

### A5 — 用 gdb 调试 + `nm` 查符号
```bash
cat > debugme.c <<'EOF'
#include <stdio.h>
static int helper(int n) { return n * 2; }   /* static, t 段 */
int g_count = 42;                            /* extern, D 段 */
int main(void) {
    int x = 5;
    printf("x=%d, helper(x)=%d, g_count=%d\n", x, helper(x), g_count);
    return 0;
}
EOF
gcc -std=c11 -g -O0 -o debugme debugme.c
nm debugme | grep -E ' (helper|g_count|main)$'   ## 查符号类型
echo "--- gdb ---"
gdb -batch -ex 'b main' -ex 'r' -ex 'p x' -ex 'p helper(10)' -ex 'q' ./debugme
```
**验收**：
- `nm` 输出 `t helper`（小写 t = static 函数）、`D g_count`（大写 D = 已初始化全局）、`T main`（大写 T = extern 函数）
- `gdb` 在 `main` 断点停下，`p x` 输出 `x = 5`，`p helper(10)` 输出 `20`

## §6 值得深入思考的问题

1. **`volatile` 在 C++ 为什么不保证线程安全？** C++ 标准明说"volatile is not a synchronization primitive"——它只是"对单线程 + 硬件 + signal handler 场景防优化"。**C11 `atomic` + memory order（relaxed/acquire/release/seq_cst）才是正解**。**为什么 x86 上 `volatile` 看起来"work"？** 因为 x86 的 TSO 内存模型本身就是顺序一致性；ARM/POWER 的弱内存模型下 `volatile` 失效。**嵌入式 ARM Cortex-M 该用哪个？**
2. **`-fvisibility=hidden` 怎么改默认？** 默认 gcc 给函数 `extern`（全局可见）。`-fvisibility=hidden` 让所有符号默认 hidden（仅本 .so 可见），需要 `__attribute__((visibility("default")))` 显式开。**Linux kernel 为何不用这个？**——kernel 是单一二进制，不需要 .so ABI 隔离。
3. **`static` 函数在头文件 = 每 .c 各一份，链接器不报多重定义？** 答：`static` 是内部链接，链接器根本不看 .o 之间的 static 符号——**所以不冲突但语义错**（你以为共享一个，实际每个 .c 自己的）。**Linux kernel 怎么避免这个？**——**kernel 禁止头文件放 `static` 函数**（checkpatch.pl 报 warning）。
4. **`-pg` 已被 `perf` 替代，但 `gprof` 还在哪些场景用？** 答：嵌入式无 `perf` 时（OProfile、SystemTap）；教学场景（gprof 简单）。**`perf` 怎么读火焰图？**——`perf script | FlameGraph/stackcollapse-perf.pl | flamegraph.pl > out.svg`。
5. **AI 写的 C 代码 80% 错在多线程共享 + `volatile` 误用**——你能写出一个 PR 规则，在 CI 抓"多线程共享但只用 volatile 不用 atomic"的模式吗？**clang-tidy 有 `bugprone-not-null-terminated-result` 等规则**；**`concurrency-mt-unsafe` 检 pthread 不安全调用**。**自定义规则怎么写？**
6. **`-O0` 调试 vs `-O2` 跑性能，怎么协调？** 答：CI 跑 ASan/UBSan 用 `-O1 -g`（少量优化 + 符号）；性能基线用 `-O2 -DNDEBUG`；release 用 `-O2 -DNDEBUG -fvisibility=hidden -fdata-sections -ffunction-sections -Wl,--gc-sections` 减体积。**Linux kernel 怎么平衡？**——`CONFIG_DEBUG_INFO=y` + `CONFIG_OPTIMIZE_FOR_SIZE`（默认 -Os）。
7. **`const` 在 C++ 怎么变强？** C++ `const` 成员函数保证"不修改对象"（除非 `mutable`）；C++17 `if constexpr` 配合 `const` 做编译期分支；C++20 `consteval` 强制编译期求值。**C 的 `const` 永远只是"运行时+约定"，不参与类型系统**——这是 C 的局限。

---

*下一章预告*：**ch5 Input and Output**——38 页中等章。从底层 `open/read/write/close` syscall（系统级 I/O）到高层 `fopen/fread/fwrite/fclose/printf/scanf`（缓冲 I/O），再到 `dup2` 重定向 stdin/stdout/stderr、`lseek` 随机访问、非阻塞 I/O（`O_NONBLOCK`）与 `select`/`poll`、命名管道（FIFO）。**这一章是 ch6 网络编程的预备**——所有 socket 调用都遵循与文件描述符一样的 `open/read/write/close` 模式。

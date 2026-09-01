# 第 2 章 · Objects, Functions, and Types

> 来源: *Effective C* (Seacord, 2020) — Chapter 2, pp. 13–34
> 笔记日期: 2026-08-27

---

# 一、章节概述

1. **每一个 C 类型，要么是 object type，要么是 function type**——这是 Seacord 说他"最后学到的东西之一"，是理解整个 C 类型系统的入口。
2. **object = 存储 + 类型**：object 是 ISO C 定义的"可持有值的存储区域"；变量是带名字的 object。同样的字节（如 IEEE 754 表示 1.0 的 `0x3f800000`）当 `int` 读是 1,065,353,216。
3. **call-by-value 是 C 的根本**：把"swap 失败"作为教学案例——函数收到的是参数的**副本**，修改副本不会传回去。指针只是"把副本换成地址副本"，仍是值传递（这是 C 唯一一种模拟 call-by-reference 的方式）。
4. **scope（作用域）vs lifetime（生存期）是两个独立概念**：scope 管"标识符在哪些源码位置可见"，lifetime 管"对象在运行期什么时候存在"。混用是初学者最常踩的坑。
5. **四种 scope**：file / block / function prototype / function（只有 label 用 function scope）。嵌套时内层标识符可隐藏外层；**好实践是大作用域用长名字，小作用域用短名字**。
6. **四种 storage duration**：automatic（块内）/ static（程序全程）/ thread（C11 起并发用，本书不覆盖）/ allocated（heap，第 6 章）。`static` 在块内也合法。
7. **static 变量必须用常量初始化**，不能用 const 变量（C 标准意义上的"常量"只包括字面量、enum、sizeof/alignof 结果等）。
8. **alignment（对齐）默认由编译器处理**，C11 起可用 `_Alignas` 显式声明。结构体里 double/pointer 跨越 cache line 会显著拖慢 ARM/x86。
9. **三大类型族**：object types（bool / char / int / float / void…）、function types（返回类型+参数列表）、derived types（pointer / array / struct / union / typedef）。所有"复杂类型"都是 derived。
10. **`_Bool` / `bool` / `true` / `false`** 是 C99 起的；`_Bool` 名字带下划线是为了避开历史代码里的 `bool`。**`_t` 后缀是标准/POSIX 保留**——你自己 typedef 别用 `_t` 结尾。
11. **char 不等于 signed char/unsigned char**，但与其中之一**对齐/大小/表示完全相同**。用 char 表示小整数是反模式；要做整数用 `signed char` 或 `unsigned char`。
12. **结构体 vs 联合体**：struct 成员独立分配、union 成员共享存储。union 用 `type` 字段做"判别联合"（tagged union），是变体类型的经典实现。
13. **tag 不是 type**：`struct s {…}` 里的 `s` 是 tag，要写 `struct s var` 不能写 `s var`。但 typedef 可以把 tag 包成别名，且 typedef 不创建新类型，只是别名。
14. **三大类型限定符**：`const`（禁止修改，编译器可放只读段）/ `volatile`（**禁止缓存**，用于 MMIO 和信号处理）/ `restrict`（承诺指针不别名，启用更激进优化）。
15. **`const` 不能用 cast 偷偷抹掉**（如果原对象本来就是 const）。**`volatile` 不是线程同步原语**——C 的 `volatile` 与 Java/C# 的语义完全不同，**不要用它做线程同步**。
16. **`restrict` 是契约**——你承诺两个指针不指向重叠区域；如果违反，UB。错误使用 `restrict` = 错误的安全感 = 错的优化。

---

# 二、核心 Takeaways

### Takeaway 1: C 的所有类型 = object type ∪ function type

- **是什么**：从类型论角度，C 只有两类东西：值（object）与动作（function）。指针、数组、结构体、typedef 都是从这两类"派生的"。
- **为什么重要**：这把所有奇怪的语法（`int (*fp)(void)`、`int (*arr)[5]`、`void (*signal(int, void(*)(int)))(int)`）都解释清楚——它们都是"在 base type 上加 declarator"。
- **解决什么问题**：读懂复杂声明的"螺旋规则"（clockwise/spiral rule），不再对 `void *(*(*fp)(int))[10]` 这种类型发怵。
- **适用场景**：所有 C 代码，特别是回调注册、信号处理、嵌入式驱动 API。

### Takeaway 2: C 是 call-by-value，连指针也是值传递

- **是什么**：函数参数永远是"传入值的副本"；指针也是"地址的副本"。
- **为什么重要**：很多人以为 C "有引用传递"——错。"传地址再 deref" 是 C 唯一一种模拟 call-by-reference 的方式，且本质仍是值传递。
- **解决什么问题**：解释"为什么 swap 函数需要 `int *pa, int *pb` 而不是 `int a, int b`"——不是 C 有引用，而是 C 让你手动拿到地址去改原始对象。
- **适用场景**：所有函数参数设计；写 API 时决定哪些参数是指针（要修改谁）、哪些是值（只读）。

### Takeaway 3: Scope ≠ Lifetime

- **是什么**：
  - **scope**：标识符在源码里**哪里可被引用**（编译期概念）。
  - **lifetime**：对象在运行期**什么时候存在**（运行期概念）。
- **为什么重要**：`static int counter = 0;` 在函数内——counter 的 scope 是块内，但 lifetime 是整个程序。"局部变量的值在函数返回后仍存在"是 scope/lifetime 分离的典型例子。
- **解决什么问题**：理解 static 在块内的含义；理解为什么函数返回局部指针通常 UB（对象已死）。
- **适用场景**：所有涉及 static、heap、回调闭包的设计。

### Takeaway 4: `_Alignas` 不是性能优化，是正确性

- **是什么**：C11 引入 `_Alignas(type)` 或 `_Alignas(N)`，强制对象按 N 字节对齐（N 必须是 2 的幂）。
- **为什么重要**：某些 CPU（ARMv5 之前的 ARM、older MIPS、Cortex-M0 部分指令）**不能解引用未对齐指针**——直接硬件异常。C 默认对齐对标准类型足够，但**做 type punning / 跨类型 reinterpret cast / 写 SIMD / 直接 cast 字节缓冲为结构体指针**时，需要 `_Alignas` 保证基地址对齐。
- **解决什么问题**：避免 SIGBUS / HardFault / 未定义硬件行为；让 SIMD（SSE/NEON）指令真正能跑。
- **适用场景**：网络协议解析（直接 cast packet buffer 为 struct）、驱动 MMIO、SIMD 内核、Cache-line 对齐的并发数据结构。

### Takeaway 5: `volatile` 不是线程同步原语

- **是什么**：`volatile` 只告诉编译器"每次访问必须真的读/写内存，不要优化掉"。它**不**保证原子性、不提供 memory barrier、不保证可见性。
- **为什么重要**：Java 的 `volatile` = C11 的 `_Atomic`；C 的 `volatile` = Java 的"基本没用"。Linux 内核里有大量"在多线程代码里错误使用 `volatile`"的历史代码被改成 `_Atomic` 或 `READ_ONCE/WRITE_ONCE`。
- **解决什么问题**：避免"`volatile int flag = 1;` 来同步线程" 这种**视觉上对、实际不保证可见性**的 bug。
- **适用场景**：**只**用在 MMIO 寄存器、单次中断标志、与硬件异步交互的内存；**不要**用在共享变量上。

### Takeaway 6: `restrict` 是给编译器的承诺

- **是什么**：`void *memcpy(void * restrict dst, const void * restrict src, size_t n)` —— 编译器可以假设 dst/src 不重叠。
- **为什么重要**：开启后编译器可向量化循环（`for(int i=0;i<n;i++) a[i]=b[i];` 变 SIMD）。**违反则 UB**——程序可能产生错误结果但不会报错。
- **解决什么问题**：标准库 memcpy/memmove 的语义分离：memcpy 假定不重叠（UB if overlap），memmove 保证处理 overlap。
- **适用场景**：写自己的 memcpy-like 函数时加 `restrict`；用 memmove 而不是 memcpy 当不确定是否重叠时。

### Takeaway 7: `const` cast-away = 隐藏的 UB

- **是什么**：`const int i = 1; int *ip = (int *)&i; *ip = 2;` —— 原对象是 const，写它就是 UB，可能段错误。
- **为什么重要**：编译器会把 const 全局/局部放到 `.rodata` / 栈只读页；运行时写会 segfault。这是"通过编译但运行崩溃"的经典案例。
- **解决什么问题**：避免为了"修改 const 入参"硬 cast——那应该一开始就不要传 const。
- **适用场景**：API 设计时决定"哪些参数 const"；code review 时盯住 cast-away-const。

---

# 三、工程实践视角

### 嵌入式开发

- **`_Alignas` 的真实价值**：在 Cortex-M 上做 cache line 对齐（`__ALIGNED(32)`），或者确保把外设寄存器结构体按 4 字节对齐（防 unaligned access 异常）。**TI C2000 / NXP S32K** 的硬件文档会单独强调 alignment 要求。
- **`static` 局部变量在中断上下文**：`static volatile uint32_t tick = 0;` 在 timer ISR 里递增——这是 `volatile` 的正确用法（不是线程同步，是与异步硬件事件通信）。**不要**用 `static volatile` 做主循环与 ISR 之间的共享变量同步——要用 `volatile sig_atomic_t` + 关中断。
- **裸机没有 heap 时**：`_Alignas` + 静态数组 = 手动控制布局。FreeRTOS 的 `xStaticCreate`、Zephyr 的 `K_HEAP_DEFINE` 都依赖 alignment 正确性。
- **`restrict` 在嵌入式里**几乎都是错的——同一 buffer 的两个视图（如 SPI RX/TX 双工）天然 overlap。**默认不加**。

### Linux 系统开发

- **内核代码明确规则**：Linux kernel coding style 第 19 章："不要用 `volatile`，它不是同步原语"。共享变量用 `READ_ONCE/WRITE_ONCE` 或 `smp_mb__after_atomic`。
- **POSIX 保留 `_t` 后缀**：你自己写 `mytime_t` 可能会和未来 POSIX 冲突。**Linux 内核代码里搜 `_t` 结尾的本地 typedef 是 code review 红线**。
- **`static` 局部 = 函数内全局**：kernel 里 `static int foo(void) { static int initialized; if (!initialized) { initialized = 1; ... } }` 是 idiom，比 global static 更 scope 友好。
- **`call-by-value` 的代价**：把大 struct 当参数 = 整块 memcpy 到栈。要么传指针，要么 C99 起用 `const struct T *` 避免修改。

### 机器人软件（ROS / ROS2）

- **ROS message 是结构体**——`sensor_msgs::msg::Imu` 就是 `struct Imu { Header header; ... };`。tagged union 用于 ROS 的 `time`（sec/nsec vs double）。
- **指针 vs 引用**：ROS2 C++ 客户端代码用引用传 callback 参数；C 层（micro-ROS）则要显式指针——**了解这层差异**才不会在跨语言桥接时困惑。
- **MMIO 风格寄存器访问**：电机控制器（CANopen / EtherCAT）的状态寄存器在用户态通过 `/dev/mem` 或 `mmap` 映射；这正是 `volatile` 的合法用法。

### 汽车电子软件（AUTOSAR / ISO 26262）

- **MISRA C 规则 11.8（"不应做 cast-away-const"）** = 本章 Takeaway 7 的工程版本。AUTOSAR 编译器配置里通常开 `-Wcast-qual` 警告。
- **struct 内存布局 = 决定 wire format**：CAN / FlexRay / Ethernet 信号打包到字节时，结构体字段顺序直接影响字节序、padding、对齐——MISRA 要求用 `#pragma pack` 或显式 `__attribute__((packed))` 时**必须文档化**。
- **ASIL-D 禁用 dynamic allocation** → 所有"对象"必须是 static duration，scope 可以是 file，但 lifetime = program。
- **`_Bool` 在汽车里有时被替换为 `uint8_t`**：为了让 wire format 明确，也避开 C99 引入的 `_Bool` 在某些 MCU 编译器的兼容性问题。
- **type qualifier 是合同**：`const` 标识"这个信号处理器只读输入"；`volatile` 标识"这个寄存器读侧硬件会改"；`restrict` 在汽车几乎不用。

### 初级 vs 高级工程师的关注差

| 维度 | 初级 | 高级 |
|---|---|---|
| 写 swap | `void swap(int a, int b)` 然后奇怪为啥不工作 | 一开始就写 `void swap(int *a, int *b)`，且能用三句话解释"call-by-value 是 C 的根本" |
| 看复杂声明 | 死记硬背、逐字读 | 套 clockwise/spiral rule：先找变量名、再 spiral |
| 用 `volatile` | 用来同步线程 | 用来访问 MMIO；线程同步用 `_Atomic` 或 mutex |
| 用 `const` | 加不加看心情 | API 边界一律 `const`；code review 盯 `cast-away-const` |
| 用 `restrict` | 看库里有就加上 | 严格评估 alias 关系；不确定时用 memmove 而不是 memcpy |

---

# 四、AI 时代视角

### 这个知识今天是否仍然重要？

**重要性级别：极高**（相比第 1 章更高）。

- **C 类型系统没变化**：这一章的概念在过去 20 年没动过，**也不会动**——C 标准委员会的核心约束是 ABI 稳定。
- **AI 仍经常写错**：尤其是复杂声明（`void (*(*fp)(int))[10]`）、restrict 误用、const cast-away。**这些恰恰是 AI 训练语料里"看起来对、实际 UB"最多的地方**。

### AI 能帮助完成什么

- ✅ 解释 `void (*signal(int, void (*)(int)))(int)` 这种 K&R 风格 signal 声明
- ✅ 用 clockwise rule 工具（cdecl.org）把 C 声明翻成英文
- ✅ 检查 typedef 是否违反 `_t` 后缀保留规则
- ✅ 建议 `_Atomic` 还是 `volatile`（基于场景）

### AI 无法替代什么

- ❌ **判断两个指针是否真正不重叠**——这是语义问题，要读懂代码上下文
- ❌ **决定 `static` 局部 vs 全局**——涉及可测试性、并发、可重入性 trade-off
- ❌ **MMIO 的对齐要求**——AI 没有你 MCU 的数据手册
- ❌ **struct wire format 的字节序**——属于协议设计
- ❌ **判断"我以为不 overlap 实际 overlap"**——要 trace 数据流

### 工程师必须掌握的核心能力

1. **读懂任何 C 声明**：熟练用 spiral rule（或 `cdecl` 工具）
2. **理解 storage duration 四种**：这是后续动态内存（ch6）、并发（ch11 之外）的基石
3. **区分 `volatile` / `_Atomic` / `mutex`**：三件事不要混
4. **理解 `restrict` 的"承诺即 UB"语义**：写代码时不该默认加
5. **理解 struct 内存布局**：padding、对齐、`#pragma pack` 的代价

---

# 五、实践行动项

### 行动 1: 验证你机器上整数类型的实际大小
```bash
cat > /tmp/types.c <<'EOF'
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include <limits.h>
int main(void) {
    printf("sizeof(_Bool)=%zu char=%d short=%d int=%d long=%d llong=%d\n",
        sizeof(_Bool), (int)sizeof(char), (int)sizeof(short),
        (int)sizeof(int), (int)sizeof(long), (int)sizeof(long long));
    printf("INT_MAX=%d UINT_MAX=%u\n", INT_MAX, UINT_MAX);
    printf("bool=true? %d\n", (1 == true));
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -Wpedantic -o /tmp/types /tmp/types.c && /tmp/types
```
**目的**：把"int 是 32 位"这种假设变可观测事实。`long` 在 LP64 是 8 字节、ILP32 是 4 字节——**可移植 C 代码不能假设**。

### 行动 2: 演示 `volatile` 在 MMIO 风格的必要性
```c
#include <stdio.h>
int main(void) {
    volatile int port = 0;
    port = port;   // 编译器必须真读真写，否则会被 -O2 优化掉
    printf("port = %d\n", port);
}
```
```bash
cc -std=c17 -O2 -S -o - /tmp/types.c | grep -E "volatile|port" | head
```
对比：不加 `volatile` 时，`port = port;` 在 `-O2` 下变 nop。

### 行动 3: 触发并观察 cast-away-const UB
```bash
cat > /tmp/castconst.c <<'EOF'
#include <stdio.h>
int main(void) {
    const int i = 1;
    int *ip = (int *)&i;
    *ip = 2;        // UB: 原对象是 const
    printf("i = %d (UB! 可能是 1 也可能是 2)\n", i);
    return 0;
}
EOF
cc -std=c17 -fsanitize=undefined -g -O1 -o /tmp/castconst /tmp/castconst.c
/tmp/castconst
```
**预期**：UBSan 报错；编译器在 `-O2` 下可能把 `i` 缓存到寄存器，所以打印仍是 1——**这是 UB 的具体表现**：编译器可以"完全忽略"。

### 行动 4: 演示 `restrict` 的契约
```bash
cat > /tmp/restrict_demo.c <<'EOF'
#include <string.h>
#include <stdio.h>
void fast_copy(int * restrict dst, const int * restrict src, size_t n) {
    for (size_t i = 0; i < n; i++) dst[i] = src[i];
}
int main(void) {
    int a[10] = {1,2,3,4,5,6,7,8,9,10};
    fast_copy(a + 1, a, 5);    /* overlap! UB because of restrict */
    for (int i = 0; i < 10; i++) printf("%d ", a[i]);
    printf("\n");
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -fsanitize=undefined -g -O2 -o /tmp/restrict_demo /tmp/restrict_demo.c && /tmp/restrict_demo
```
**预期**：结果可能是正确的、也可能是错的——因为 UB。**教训**：`restrict` 不是"免费加速"，是契约。

### 行动 5: 用 `_Alignas` 演示 cache-line 对齐
```bash
cat > /tmp/align.c <<'EOF'
#include <stdio.h>
#include <stdint.h>
#include <stdalign.h>
struct S { int i; double d; char c; };
_Alignas(64) char cache_aligned_buf[sizeof(struct S)];  // 64B cache line
char default_aligned_buf[sizeof(struct S)];
int main(void) {
    printf("default  buf: %p (mod 64 = %lu)\n",
           (void*)default_aligned_buf, (unsigned long)((uintptr_t)default_aligned_buf % 64));
    printf("cache    buf: %p (mod 64 = %lu)\n",
           (void*)cache_aligned_buf, (unsigned long)((uintptr_t)cache_aligned_buf % 64));
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o /tmp/align /tmp/align.c && /tmp/align
```
**目的**：可见地看到 `_Alignas` 让对象落到 64 字节边界——并发数据结构、SMP 缓存性能优化的基础。

---

# 六、值得深入思考的问题

### Q1: 如果 C 是"纯 call-by-value"，那么函数式编程里的"闭包"在 C 里怎么表达？

提示：自由变量只能塞到全局、`static` 局部、或 caller 分配的 `void *` context（POSIX `pthread_create` 第三参数、libevent 回调就是这种模式）。**问题**：为什么 C 不引入真正的 closure？这是设计取舍还是历史包袱？

### Q2: `_Bool` 引入 C99 时用下划线开头是为了避免冲突。如果当时叫 `bool`，今天的 C 标准库会怎样不同？

这涉及**关键字 vs 库标识符**的设计哲学——C 选择前者（关键字少 + 库扩展），C++/Rust 选择后者（更友好）。**问题**：当 AI 自动补全代码时，这两种选择哪个更鲁棒？

### Q3: Linux 内核明确禁止把 `volatile` 用于线程同步。**为什么 C 标准还保留 `volatile` 这个关键字让工程师踩坑？**

可能的解释：保留是为了 MMIO。**问题**：要不要引入 `_Atomic volatile`（"原子 + 禁止优化"）作为新关键字？`volatile` 是否应该 deprecate？

### Q4: `restrict` 是 C99 的伟大优化——但作者也警告"违反就是 UB"。**有没有一种方式，能让 `restrict` 变成"调试模式下 trap 但 release 模式下 unsafe"，把 UB 边界变可见？**

提示：可以参考 Rust 的 `unsafe` 块 + sanitizer。**问题**：C 的 UB 哲学和 Rust 的 unsafe 哲学的根本差异是什么？

### Q5: 假设你要写一个跨 MCU（ARM/RISC-V/x86）、跨 RTOS（FreeRTOS/Zephyr/ThreadX）的硬件抽象层（HAL）。**这一章的哪些概念（scope / lifetime / alignment / qualifier）会变成你的"必须"vs"可选"清单？**

约束思考：freestanding + MISRA + 0 dynamic allocation + 多 MCU 移植。**问题**：在这样的环境里，`const`、`volatile`、`restrict` 各自代表什么具体意义？

---

*下一章预告*: **Chapter 3 — Arithmetic Types** —— 整数与浮点的 representation、padding、precision、integer conversion rank、integer promotion、usual arithmetic conversion、safe conversion。这是后续表达式/类型转换的基石。
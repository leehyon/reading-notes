# 第 6 章 · Dynamically Allocated Memory

> 来源: *Effective C* (Seacord, 2020) — Chapter 6, pp. 99–118
> 笔记日期: 2026-08-27

---

# 一、章节概述

1. **第三种 storage duration = allocated（heap）**：从第 2 章的四种 storage duration 展开。heap 是"由 memory manager 管理的一块或多块可分割大内存"，申请方使用、归还由 free 负责。
2. **Memory manager 是用户态库**（glibc malloc / jemalloc / tcmalloc / dlmalloc），从 OS 申请大块、自己分割维护 boundary tags。**Knuth 经典算法**用前后 boundary tag 合并碎片。
3. **何时用动态内存**：编译期**不知道大小或数量**——运行时读文件、链表/哈希表/二叉树、用户输入。**编译期已知 = 不要 malloc**。
4. **`malloc(size)` 返回 `void *` 或 NULL**：① size 用 `sizeof(T)`、**不**用裸数字；② **永远检查 NULL**；③ **不初始化**——reading uninitialized = UB。
6. **`malloc` 返回值要不要 cast？**：C 里 `void *` 隐式转 `T *`（C++ 不行）——**cast 可加可不加**。CERT MEM02-C 推荐立即 cast，便于捕获类型/大小不匹配；但作者在书里"两种风格都接受"。
7. **calloc(nmemb, size)** 分配**并清零**——但清零 ≠ 浮点 0.0（IEEE 754 +0.0 是全 0，-0.0 也是全 0）；calloc 的 0 是全 0 字节。**calloc 内部检查 nmemb * size overflow**。
8. **realloc 的双重身份**：① ptr=NULL 时等价于 malloc(size)；② 成功时返回**新地址**（可能原地扩展、可能迁移），旧指针**立即失效**；③ 失败时返回 NULL，**旧指针仍有效**——必须先存新指针再覆盖旧的。
9. **realloc 的经典 leak bug**：`p = realloc(p, nsize)` —— 失败时 `p = NULL` 但旧内存仍占用 = leak。**正解**：`newp = realloc(p, nsize); if (!newp) { free(p); return NULL; } p = newp;`。
10. **realloc(p, 0)** 是 implementation-defined（C17 起 implicit UB / C2x 起显式 UB）——**不要传 0**。
11. **reallocarray 是 OpenBSD 引入的 realloc 安全替代**：`reallocarray(p, nmemb, size)` 内部检查 `nmemb * size` overflow。glibc 5.x+ 已采纳。
12. **free(p)** 释放 p；**free(NULL) 合法且 no-op**。**double-free = UB**——是安全漏洞来源（攻击者可借此执行任意代码）。**use-after-free = UB**——值看似能用实际已死。
13. **dangling pointer（悬空指针）** = 已 free 的指针值——**free 后立刻置 NULL** 是 mitigation（虽然 `free` 改不了原指针本身）。
14. **内存三态**：unallocated-and-uninitialized（属于 manager，read/write/free 都 UB）/ allocated-but-uninitialized（可 write、不能 read、可 free）/ allocated-and-initialized（都可）。
15. **柔性数组成员（C99）**：`struct { size_t n; int data[]; }` —— `sizeof` 不含 data，`malloc(sizeof(struct) + n * sizeof(int))`。替代老 "struct hack"（用 `data[1]` + `malloc(size + (n-1)*sizeof)`）。
16. **`alloca` 是 GCC/Clang 编译器内建**，栈上分配、自动随函数返回释放。**性能比 malloc 好**（一条指令调整栈指针）。**但**：无 NULL 返回（stack overflow 时静默踩坏）、不能 free、不在 C 标准库 / POSIX 里、**`-Walloca` 警告**。
17. **VLA（C99）** = `int vla[size];` —— 块作用域或函数原型里。**但 stack overflow 风险**、**size 必须先 check**、**`-Wvla` + `-Wvla-larger-than=`** 是防御。
18. **`sizeof` on VLA 在运行时求值**（不是编译期）：`sizeof(int[size++])` 会执行 `size++`！这是大陷阱。
19. **VLA 作函数参数**：`void *memset_vla(size_t n, char s[n], int c);` —— 不分配新存储，仅声明；调用方负责实际 storage。
20. **VLA 用于通用矩阵**：`int matrix_sum(size_t r, size_t c, int m[r][c])` —— 比固定 `int m[][4]` 通用。
21. **安全关键系统禁用 dynamic allocation**：因为 memory manager 行为不可预测。`alloca` / VLA / recursion 一起禁——**栈使用可静态证明上界**。
22. **dmalloc 是 dynamic analysis 工具**：替换 malloc/realloc/free/...，运行时检测 leak/double-free/UAF。每 N 次调用 check 一次。

---

# 二、核心 Takeaways

### Takeaway 1: `malloc` 失败必须检查，且返回的内存**未初始化**

- **是什么**：`void *p = malloc(n);` —— 返回 NULL 表失败；返回非 NULL 但**内容是垃圾**。
- **为什么重要**：① 不检查 NULL → 解引用 0x0 = SIGSEGV 或更糟；② 不初始化 → read UB（编译器可优化、值未定义、安全漏洞）。
- **解决什么问题**：所有动态内存路径必须先检查 NULL；用前必须先 memset/calloc 或显式赋值。
- **适用场景**：所有 `malloc/realloc/aligned_alloc` 调用。

### Takeaway 2: `realloc` 的陷阱在于"返回值覆盖原指针"导致 leak

- **是什么**：`p = realloc(p, n)` —— 失败时 `p = NULL`，旧内存仍占用（realloc 不 free 旧块）= **leak**。
- **为什么重要**：这是 CERT MEM34-C 的核心规则之一。leak 累积到 OOM。
- **解决什么问题**：始终用临时指针 `newp` 接收 realloc 返回值，确认成功再覆盖。
- **适用场景**：所有 realloc 调用；尤其动态数组扩容、字符串拼接缓冲区。

### Takeaway 3: `realloc(p, 0)` 是 UB（C2x）——不要传 0

- **是什么**：把 realloc 当 free 用 = 历史老 bug。C17 是 implementation-defined；C2x 起是显式 UB。
- **为什么重要**：某些 libc 实现会返回 NULL + free 旧块；某些返回小指针；某些崩溃——**不可预测**。
- **解决什么问题**：要释放就 `free(p)`；要扩展就 `realloc(p, n)` 且 `n > 0`。
- **适用场景**：所有 realloc 边界检查。

### Takeaway 4: free 后立即置 NULL——防止 double-free

- **是什么**：`free(p); p = NULL;` —— 后续 `if (p) free(p)` 是 no-op；double-free 不会触发。
- **为什么重要**：free 本身改不了原指针（传值），需要调用方手动置 NULL。这不能消除所有 UAF，但能阻止**一半的 double-free**。
- **解决什么问题**：降低 double-free 风险，让 ASan 之外的多一层防御。
- **适用场景**：所有 `free` 调用——尤其是错误处理路径、cleanup chain。

### Takeaway 5: 柔性数组 = 避免 "struct hack"

- **是什么**：`struct { size_t n; int data[]; } w;` —— 末尾数组无大小；`malloc(sizeof(w) + n*sizeof(int))`。
- **为什么重要**：老代码用 `data[1]` + `malloc(size + (n-1)*sizeof)`（CERT DCL38-C 警告）；柔性数组是 C99 标准方案。
- **解决什么问题**：可变长对象的结构化表达，编译器知道 `sizeof` 不含 data。
- **适用场景**：网络协议消息、变长记录、字符串 + 长度前缀结构。

### Takeaway 6: `alloca` 在 C 标准外、有 stack overflow 风险——不推荐

- **是什么**：栈分配，函数返回自动释放。GCC/Clang 用一条指令调整栈指针（极快）。
- **为什么重要**：无 NULL 返回（stack overflow 静默踩栈）、不能 free、不在 C 标准库 / POSIX、**`-Walloca` 警告**。
- **解决什么问题**：可用但要谨慎——只在分配大小明确受控（如 `strerrorlen_s` 返回值）且非递归函数里用。
- **适用场景**：基本不推荐；错误处理短字符串拼接偶尔用。

### Takeaway 7: VLA = 编译期灵活但 stack overflow 风险——安全关键禁用

- **是什么**：`int vla[size];` 在块作用域里声明。
- **为什么重要**：① stack overflow 无 portable 检测；② 递归函数里 VLA 可爆栈；③ **`sizeof(vla)` 运行时求值，副作用会被执行**——大陷阱。
- **解决什么问题**：在通用算法里做灵活 buffer；但生产安全关键代码（汽车/航空/医疗）禁用。
- **适用场景**：临时数组、矩阵函数参数、教学示例——**不用于 ASIL-C/D 等安全等级**。

### Takeaway 8: 安全关键系统（ASIL-D / 航空 / 医疗）禁用所有动态分配

- **是什么**：MISRA C / AUTOSAR / DO-178C 明确要求：禁用 malloc/calloc/realloc/alloca/VLA/recursion。
- **为什么重要**：所有内存使用必须**静态可证明**。**stack 上界**也能静态推导。
- **解决什么问题**：消除 memory manager 不确定性 + stack overflow 风险 → 安全认证可达。
- **适用场景**：任何 ISO 26262 ASIL-D、IEC 61508 SIL 3+、DO-178C 关键等级项目。

---

# 三、工程实践视角

### 嵌入式开发

- **MCU 上 malloc 通常不用**：裸机没有 OS，heap 来自 startup 里静态数组 → `malloc` 是用户态内存池。Bare metal 通常用**静态内存池**（FreeRTOS `xQueueCreate` / Zephyr `K_HEAP_DEFINE`）。
- **`realloc` 在嵌入式里是禁忌**：碎片化 + 不可预测。**预先分配最大可能 + 用 offset** 是常见模式。
- **柔性数组 = 协议包结构**：CAN / Ethernet packet = `{header; data[];}` —— 这就是柔性数组的典型用法。
- **`alloca` 在递归函数里 100% 是 bug**：裸机栈 4KB~64KB，递归里 alloca 几个 KB 直接 HardFault。
- **VLA 慎用**：嵌入式栈小（8KB 常见），`int vla[1024]`（4KB）= 栈爆一半。**`-Wvla-larger-than=64` 防爆**。

### Linux 系统开发

- **glibc malloc 是 ptmalloc2**：thread-safe（per-thread arena）、boundary tag。jemalloc/tcmalloc 在高竞争下更优。
- **`mmap` 阈值 (128KB)**：大分配绕过 heap 直接 mmap（避免污染 arena）。`mallopt(M_MMAP_THRESHOLD, ...)` 可调。
- **`mtrace` / `mtrace` 工具** 抓 leak；**valgrind --leak-check** 经典。
- **`realloc` 跨 arena 时不释放旧块**：多线程程序跨线程 realloc 可能 leak——`jemalloc` 改进了这点。
- **kernel 内存**：内核里 `kmalloc/vmalloc/kzalloc` 与用户态 malloc 完全不同——前者 slab allocator，后者 buddy system。
- **`reallocarray` 自 glibc 2.26+ 提供**——直接用，不要自己写 `realloc(p, n*s)` 然后检查 overflow。

### 机器人软件（ROS / ROS2）

- **ROS 消息用柔性数组模式**：`sensor_msgs::PointCloud2` = `{header; data[];}` —— C 客户端 micro-ROS 直接用柔性数组。
- **Real-time loop 禁 `malloc`**：控制循环里 `malloc` 引入不可预测延迟 → jitter。**预先分配 + reset**。
- **Point cloud 处理**：`reallocarray(p, n_new, sizeof(point_t))` —— 容量增长按需，比裸 realloc 安全。
- **`strdup` 在 ROS 里很少用**：因为 ROS 自定义 string 类型（`std::string` / `ros::String`）。
- **micro-ROS（嵌入式）** 静态内存池：禁止 malloc，用 `urosStaticAllocator`。

### 汽车电子软件（AUTOSAR / ISO 26262）

- **AUTOSAR 规范：所有内存使用必须静态声明** → 完全禁 malloc/realloc。
- **MISRA C 规则 21.3**："The dynamic memory allocation functions shall not be used"。
- **MISRA C 规则 21.6**："The Standard Library input/output functions shall not be used"（部分）。
- **ASIL-D 等价于 DO-178C Level A**：100% 静态分析覆盖、所有路径必须可证明。
- **内存池替代**：AUTOSAR 用 `MemPool` 模块——编译期固定大小，运行时只 allocate index。
- **VLA 禁**：MISRA 规则 6.2 / 18.8 —— 完全禁用；`alloca` 同理。
- **`free` 不置 NULL 的代价**：汽车代码 review 里常见——必须 `p = NULL` 同步。

### 初级 vs 高级工程师的关注差

| 维度 | 初级 | 高级 |
|---|---|---|
| `malloc` 后 | 直接用 | 检查 NULL + memset/calloc 初始化 |
| `realloc` 失败 | leak 不知道 | `newp = realloc(p, n); if (!newp) { free(p); return; } p = newp;` |
| `realloc(p, 0)` | 当 free 用 | **不传 0**；要释放就 `free(p)` |
| `free` 后 | 留着指针 | `p = NULL` |
| 柔性数组 | 用 `data[1]` | `int data[]` + 正确 `sizeof` |
| `alloca` | 栈快 | **避免**——除非明确受控 |
| VLA | 觉得灵活 | 知道安全关键禁用；非安全用 `-Wvla` 卡死 |
| 动态内存选择 | 不知道何时不用 | 编译期已知大小 → 永远用 static |

---

# 四、AI 时代视角

### 这个知识今天是否仍然重要？

**重要性级别：极高——内存 bug 是 CVE 的最大单一来源**。

- CERT C 编码规则中**所有 MEM/ARR 系列**（MEM30-C、MEM31-C、MEM34-C、ARR30-C 等）都是内存相关。
- **CVE 数据库统计**：buffer overflow + use-after-free + double-free = 每年数百个高危 CVE。
- **AI 生成的代码恰好高频踩这些坑**——这是"AI 写 C 代码"的最大风险面。

### AI 能帮助完成什么

- ✅ 生成 `realloc` 正确的临时指针模式
- ✅ 自动添加 `free` 后的 NULL 赋值
- ✅ 把 `int data[1]` 改写成 C99 柔性数组
- ✅ 写 wraparound-safe 的 size 乘法（用 `reallocarray`）
- ✅ 写 ASan/UBSan/MSan 触发的 minimal reproducer

### AI 无法替代什么

- ❌ **决定业务逻辑能否用静态内存** —— 涉及生命周期分析
- ❌ **评估 memory manager 的运行时性能** —— 需要 profiler
- ❌ **ASan 误报的根因排查** —— 需要读 sanitizer 源码
- ❌ **跨平台的 malloc 行为差异**（glibc vs musl vs jemalloc）—— 需要对比测试
- ❌ **决定 AUTOSAR / DO-178C 项目该不该用动态内存** —— 涉及合规

### 工程师必须掌握的核心能力

1. **malloc/calloc/realloc/free 的正确写法** —— 这是 C 工程师的"基本功"
2. **柔性数组 vs 老式 struct hack** —— 现代 C 必备
3. **`realloc` 临时指针 idiom** —— 防 leak 范式
4. **free 后置 NULL** —— 防 double-free 半层防御
5. **何时不用动态内存** —— 编译期已知 → static

---

# 五、实践行动项

### 行动 1: 演示 realloc 失败的 leak 范式
```bash
cat > /tmp/realloc_leak.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
// 错误版
void bad_realloc(void) {
    char *p = malloc(10);
    p = realloc(p, 1000000000UL);   // 可能失败，p 变 NULL，旧内存泄漏
    free(p);                          // free(NULL) no-op
}
// 正确版
void good_realloc(void) {
    char *p = malloc(10);
    char *newp = realloc(p, 1000000000UL);
    if (newp == NULL) { free(p); return; }   // 先 free 旧的
    p = newp;
    free(p);
}
int main(void) { bad_realloc(); good_realloc(); return 0; }
EOF
cc -std=c17 -Wall -Wextra -o /tmp/realloc /tmp/realloc_leak.c && /tmp/realloc
```
**预期**：bad 版 malloc 10 字节 + 不释放 = leak（valgrind 会报）；good 版无 leak。

### 行动 2: 用 ASan 演示 use-after-free
```bash
cat > /tmp/uaf.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
int main(void) {
    int *p = malloc(sizeof(int));
    *p = 42;
    free(p);
    printf("p = %d (UB!)\n", *p);   // use-after-free
    return 0;
}
EOF
cc -std=c17 -fsanitize=address -g -O1 -o /tmp/uaf /tmp/uaf.c 2>&1
/tmp/uaf 2>&1 | head -30
```
**预期**：ASan 报 `heap-use-after-free`，告诉你 free 后的指针读触发。

### 行动 3: 演示 free + 置 NULL 防 double-free
```bash
cat > /tmp/dfree.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
int main(void) {
    char *p = malloc(16);
    free(p);
    p = NULL;       // 防 double-free
    free(p);         // free(NULL) no-op, 合法
    puts("OK");
    return 0;
}
EOF
cc -std=c17 -fsanitize=address -g -O1 -o /tmp/dfree /tmp/dfree.c && /tmp/dfree
```
**预期**：不置 NULL 时 ASan 报 double-free；置 NULL 后无报错。

### 行动 4: 演示柔性数组成员
```bash
cat > /tmp/fam.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
typedef struct {
    size_t n;
    int data[];
} flex_t;
int main(void) {
    size_t n = 5;
    flex_t *p = malloc(sizeof(flex_t) + n * sizeof(int));
    p->n = n;
    for (size_t i = 0; i < n; i++) p->data[i] = i * 10;
    for (size_t i = 0; i < p->n; i++) printf("%d ", p->data[i]);
    printf("\nsizeof(flex_t) = %zu (data[] doesn't count)\n", sizeof(flex_t));
    free(p);
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o /tmp/fam /tmp/fam.c && /tmp/fam
```
**预期**：打印 0 10 20 30 40；`sizeof(flex_t) = 8`（size_t）—— data[] 不计入 sizeof。

### 行动 5: 演示 VLA 的 sizeof 运行时求值陷阱
```bash
cat > /tmp/vla_sizeof.c <<'EOF'
#include <stdio.h>
int main(void) {
    size_t s = 12;
    printf("before sizeof: s = %zu\n", s);
    (void)sizeof(int[s++]);   // sizeof VLA 会求值 s++
    printf("after  sizeof: s = %zu (increment took effect!)\n", s);
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o /tmp/vla_sizeof /tmp/vla_sizeof.c && /tmp/vla_sizeof
```
**预期**：s 从 12 → 13。因为 sizeof(VLA) 在运行时求值，`s++` 执行了。这与 `sizeof(int)`（编译期 4）截然不同。

---

# 六、值得深入思考的问题

### Q1: `realloc` 失败时为什么不自动 `free` 旧指针？**这违反了"原子性"直觉。**

提示：realloc 必须返回新指针才能让 caller 区分成功/失败；如果自动 free + 返回 NULL，caller 不知道"是否还有数据"。**问题**：有没有一种 API 设计能两全？比如 `realloc_or_free(p, n, &newp)` 返回 bool？glibc / jemalloc 为什么不这么做？

### Q2: `free(p)` 后 `p = NULL` 只是"半层防御"——如果多个指针指向同一块内存，置 NULL 只能改一个。**真正的根治方法是什么？**

提示：所有权语义（Rust 的 ownership）、引用计数、GC、escape analysis。**问题**：C 程序员如何在没有这些机制的情况下，写出 use-after-free free 的代码？有什么实用的 idiom？

### Q3: 安全关键系统禁 malloc/realloc/alloca/VLA/recursion——**这意味着栈/静态内存必须 100% 预测**。**这种严格限制是 bug 防御还是工程包袱？**

**正面**：消除 90% 内存漏洞；代码可静态验证。
**负面**：灵活性下降；某些算法（如动态链表）必须重写。
**问题**：现代 C++ 用 `std::vector` 在安全关键系统能用吗？或者安全关键系统只能 C 而禁 C++？

### Q4: `sizeof(VLA)` 在运行时求值——这意味着 `sizeof(int[size++])` 会执行 `size++`。**为什么 C 标准这样设计？**

提示：VLA 大小编译期未知，sizeof 也只能运行时求值。**问题**：有没有办法让 sizeof(VLA) **不**执行 side effect？这是 C 标准的设计失误还是有意为之？

### Q5: 柔性数组成员 `int data[]` 看起来像 0 长度数组——C99 之前用 `data[1]`（struct hack）、C99 起 `data[]`。**为什么不直接允许 `data[0]`？**

提示：`data[0]` 在某些 ABI 下与 `data[]` 行为不同；编译器无法确定是否对齐。**问题**：flexible array member 解决了哪些 struct hack 的具体问题？**在跨 ABI / 跨编译器的代码里**两者兼容吗？

---

*下一章预告*: **Chapter 7 — Characters and Strings** —— ASCII / Unicode / UTF-8 / wchar_t / char16_t / char32_t 的演化、Windows 的 UTF-16 vs Linux 的 UTF-8、`<uchar.h>`、字符常量前缀（`L`/`u`/`U`/`u8`）、escape sequence 的八进制陷阱（`\10` 是 backspace 还是 char(8)？）、C 标准库的多字节转换函数（mbtowc/wctomb/mbrtowc/mbstowcs）。这一章是 i18n / 跨平台 / Windows-Linux 编码不一致问题的根源。
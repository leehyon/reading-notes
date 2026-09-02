# Chapter 2 · Dynamic Memory Management in C

> 书目:Richard Reese,《Understanding and Using C Pointers》, O'Reilly 2013
> 本章范围:PDF p.33–56(全文行 1748–2692)
> 阅读日期:2026-09-01

---

## 一、第一性原理思考

**所有「动态内存」的语义都建立在三件事上**:堆管理器的实现、对齐保证、调用方对 free 责任的不变量。本章看似只是 `malloc` / `calloc` / `realloc` / `free` 的 API 手册，实际上在反复强调**生命周期契约**(alloc 一次必须 free 一次)与**地址生命周期**(malloc 返回的地址在 free 之后内容未定义)。

把这章的论断抽成最小公理:

1. **每一次成功的 malloc 必须匹配恰好一次 free**(否则泄漏或 double-free)。
2. **free 之后，任何访问(读或写)未定义**——所谓 `pi = NULL` 之后的保护只是「下一次可能崩」而不是「下一次一定崩」。
3. **堆内存分配的对齐由指针类型决定**(`double *` 8 字节对齐、`int *` 4 字节对齐)。
4. **malloc/calloc/realloc 返回的是首字节地址**,而不是整个块；块首部之后有 heap manager 的 metadata。

**对嵌入式工程师的现实意义**:
- AUTOSAR BSW 几乎全部用 static 内存 + 内存池，**禁用 heap**;但 MCU 上的协议栈仍有部分 `malloc`-style API(比如 lwIP 的 `pbuf_alloc`),要遵守与本章一致的语义。
- 实时系统中，**malloc 的耗时不确定**(碎片化),所以「为每个 PDU 分配 8 字节然后 free」的写法无法接受——通常一次性 pool + 自己实现 sub-allocator(对应 ch6 §「Avoiding malloc/free Overhead」)。
- **dangling pointer** 在 RTOS 任务切换时尤其危险，因为切换时另一个任务可能已经释放了共享内存。
- **指针转 int 取低 32 位再转回指针** 在 64-bit host 调试时直接丢高 32 位——这正是 ch8 §「Casting Pointers」的延续。

---

## 二、章节概述

1. **运行时系统与三种内存分区**:static/global、automatic(stack frame)、dynamic(heap);runtime 负责 zero-init、push/pop frame、alloc/free。
2. **基本动态分配步骤**:`malloc` → 使用 → `free`;漏 free 即 leak,double free 即 heap corruption。
3. **malloc 的语义**:返回 `void *`(C89 后无需 cast);失败返回 NULL;不初始化内容(garbage);size_t 参数。
4. **malloc 的常见错误**:`*pi = (int *)malloc(...)`(deref 未初始化指针);漏 sizeof;漏 NULL check;static/global 用函数调用做初值(illegal in C)。
5. **calloc(numElements, elementSize)**:分配并 zero-fill,失败时 `errno = ENOMEM`;可用 `malloc + memset` 替代，但 calloc 慢。
6. **realloc(ptr, size)**:四大行为矩阵(ptr=NULL 当 malloc;size=0 当 free;size 较小复用原块；size 较大可能搬移);失败保留原块。
7. **alloca 与 VLA**:stack 上分配，函数返回自动释放，不可 free;Microsoft `malloca`;C99 VLA `char *buffer[size]` 也是栈分配，**不能用 free**。
8. **free 的语义**:ptr 必须来自 malloc 系列；NULL 被忽略；非 malloc 地址 free 后果 UB。
9. **`pi = NULL` 之后的「保护」**:后续 deref 会段错误；但多副本情况下只改了原指针。
10. **Double Free**:同一块被 free 两次 → heap corruption;两指针 alias 同一块更容易触发。
11. **程序终止时是否 free**:OS 会回收，但 free 不为 0;诊断工具需要它；可在小系统上必需。
12. **Dangling Pointer**(未初始化副本):free 后 pi 仍持有原地址 → 后续访问未定义。
13. **三种处理 dangling pointer 的方法**:`pi = NULL`、自定义 `saferFree`、运行时填充 `0xDEADBEEF` / `0xCC` / `0xCD` 等 sentinel。
14. **Debug 内存管理**:MS `_CrtSetDbgFlag` 维护 filename/line;GCC Mudflap 通过 instrumentation 检测 deref 越界。
15. **替代技术**:Boehm GC、GNU `RAII_VARIABLE` 宏(用 `__attribute__((cleanup))`)、MS `__try/__finally`。

---

## 三、核心 Takeaways

| # | 是什么 | 为什么 | 解决了什么 | 适用场景 |
|---|---|---|---|---|
| **T1** | malloc/calloc/realloc/free 四件套语义 | heap manager 在用户数据前后塞 metadata,不能越过 | 任何需要可变大小数据的 C 程序 | 字符串/PDU payload/容器 |
| **T2** | malloc 返回 void\* | C89 前需要 cast;现在 void\* 可直接赋给任何 T\* | 简化代码，跨 C/C++ | 写库 |
| **T3** | calloc 把清零的责任写进 API | 防御「garbage 字段误用」;但慢 | 安全敏感的缓冲区 | 网络帧结构体、密钥 |
| **T4** | realloc 的搬移语义 | 原块后空间可能不够，必须搬；老指针失效 | 动态增长数组 | getLine 类输入缓冲 |
| **T5** | VLA 与 alloca 是栈内存 | 函数返回自动释放；但不可 free、不可跨函数 | 函数内临时缓冲；可避免 heap 碎片 | 嵌入式临时缓冲 |
| **T6** | dangling pointer 是「指针与生命周期的脱节」 | free 不改 pi 的内容，改的是 heap 的元数据 | 解释为什么「看起来没事」其实是 UB | 任何 shared ownership |
| **T7** | double free 是同一地址被多次 free | heap manager 信任 caller,不做去重检查 | 暴露 heap corruption 的常见路径 | 多线程 share pointer |
| **T8** | pi = NULL 只是「下次更可能崩」,不是「下次必崩」 | 多副本下只改了一个 | 防止 dangling 误读 | single-owner 场景 |
| **T9** | OS 终止时回收内存≠不需要 free | 内存检测工具会报 leak;小系统可能不回收 | 让 leak 工具输出干净 | CI 检测 |
| **T10** | 静态内存池替代 malloc | heap 实时性不可预测；对象池复用降低碎片 | RTOS / 嵌入式实时性 | AUTOSAR BSW 模式 |

---

## 四、工程实践视角(领域：嵌入式 / 汽车电子)

### 落地

- **静态内存池(Static memory pool)** 在 NeuSAR/Dcm 中是基本配置：每个 PDU buffer 在 `Dcm_Init` 阶段 pre-allocate,运行时只做「借/还」,绝不调用 malloc——这正是 ch6 §「Avoiding malloc/free Overhead」的预演。
- **`uint8 *payload` + `PduLengthType len` 形参的接受方**:接收方有责任在 `PduR/Rte` 出口把 payload 归还到原来的内存池——一旦忘了 `Dcm_<buf>Release`,就是 leak。
- **`PduInfoType *` 形参的 `const` 模式**(CanIf/Com 入口):ch1 的 `const T *` 矩阵已奠基；ch2 的 free 责任链进一步界定「谁拥有、何时归还」。
- **`memset(sensitive, 0, len)` 再 free**(参考 §「Clearing Sensitive Data」):UDS 安全访问(Security Access 0x27)种子/密钥释放前必须清零，避免冷启动 attack。
- **`getLine` 类缓冲增长模式**(参考 §「Using the realloc Function to Resize an Array」):可改造为「fixed pool + overflow guard」用于 LIN/UDS 长消息接收(>4095 byte)。

### 误区

- **M1** `int *pi; *pi = (int *)malloc(sizeof(int));`——ch1 已警示,ch2 §「A common error involving the dereference operator」原文再次强调。
- **M2** `static int *pi = malloc(sizeof(int));`——编译错误，全局同理；必须先声明再赋值。
- **M3** `free(pi); pi = NULL;` 后另存一个副本 `pi2 = pi`(free 之前)→ `pi2` 仍为 dangling。`pi = NULL` 救不了 alias。
- **M4** `realloc(p, 0)` 当 free 用——实现定义，可能返回 NULL 或有效指针；**只在你想 free 时才用 `realloc(p, 0)`**,跨平台代码最好显式 `free(p)`。
- **M5** `void free(void *ptr)` 之后**再读** ptr(`printf("%p", pi)`)——读旧地址本身合法，但任何 deref 即 UB;不要把「读打印」当成「已安全」。
- **M6** 在中断上下文里调 `malloc`/`free`——heap 不是 reentrant,会破坏 fastbin/tcache;MCU 上的协议栈都是 ISR-safe 自家实现。

### 初中高工程师视角

- **初中级**:记得 `malloc`→`free` 一一对应；`free(NULL)` 安全；每次 deref 怀疑 NULL。
- **中级**:能解释 realloc 的搬移语义与失败语义；会用 `saferFree(void **)` 宏；理解 pool 的本质是「自定义堆」。
- **高级**:在 review 时一眼识别「double free 经 alias 触发」;能说清为何 ASan/glibc tcache 的安全级别不同；能根据 RTOS 选 heap(tcmalloc / jemalloc / dlmalloc)。

---

## 五、AI 时代视角

- **LLM 写 C 代码的 leak 高频场景**:
  1. 多出口函数(每个 `return` 之前忘了 free)
  2. 异常路径(error 早返回忘 free 之前 alloc 的 buffer)
  3. `realloc` 后忘了用返回值，继续用旧指针
  4. 循环里 `arr[i] = malloc(...)` 之后某次失败，前面已分配的全 leak

  对应 Copilot 提示工程：**强制显式 `goto cleanup` 模式**(Linux 内核风格)。
- **`malloc` 失败处理被 LLM 忽略**:LLM 经常省略 `if (p == NULL) return -ENOMEM;`,需要 review 时用 `clang-tidy --checks=clang-analyzer-unix.Malloc`。
- **静态分析工具替代手工 leak 检查**:`-fsanitize=address`(gcc/clang ASan)、`valgrind --leak-check=full`、`coverity`、`cppcheck --enable=warning,style,performance,portability` ——都是 ch2 §「Debug Version Support for Detecting Memory Leaks」的现代版。
- **MISRA-C 规则强制**:MISRA-C:2012 Rule 22.1「All resources obtained by dynamic allocation shall be explicitly released」+ Rule 22.5「A pointer shall not be used to access an object whose lifetime has ended」直接对应 ch2 §「Dangling Pointer」。

---

## 六、实践行动项

1. **[必做]** 复现 ch2 的「losing address」场景(`while(1) chunk = malloc(1000000);`),用 `valgrind --tool=massif` 或 `time -v` 观察 RSS 增长，然后用 `ulimit -v` 限制虚拟内存，观察 OOM 行为。落档到附录「Action #2 复盘」。
2. **[必做]** 写 `saferFree(void **pp)` 宏并与 naive `free` 对比：重复 `safeFree(pi)` 两次，看是否触发崩溃。
3. **[推荐]** 跑一个 `realloc` 增/减的对比 demo:同一块连续 realloc,记录 `&ptr` 在不同尺寸下是否变化，贴到附录。
4. **[推荐]** 跑 `malloc(0)` 与 `realloc(p, 0)` 在 glibc 13、musl、FreeBSD libc 三家下的差异，贴表。

---

## 七、值得深入思考的问题

1. **Q1**:为什么 C 标准规定 `malloc(0)` 是 implementation-defined？这条空隙背后是「heap manager 不应被要求精确表达 0 长度」的设计哲学——如果你的代码依赖「`malloc(0)` 返回 NULL」,会跨平台崩；如果不是依赖，这条规则无害。这引出**API 设计中「明面无定义」的边界价值**。
2. **Q2**:`free(p); p = NULL;` 对所有副本无能为力——这暴露了 C **没有所有权语义**的根本限制。Rust 的 `Box<T>`/智能指针如何在不付出运行时代价下表达所有权？这是语言层而非库层的解决方案。
3. **Q3**:`realloc` 失败保留原块——这是「**in-place vs copy-and-update**」的设计折衷：返回 `T *` 而不是 `bool`,让 API 同时表达「成功 + 新指针」和「失败 + 旧指针」两件事。这与 ch3 的「`saferFree(void**)`」是同一思路的反面。
4. **Q4**:`alloca` 与 VLA 是栈分配，函数返回自动 free——这意味着**不能把它们的地址传出函数**。它与 heap 的根本差异是「栈帧随函数生命周期」,理解这点才能解释 ch3 的 dangling 问题。
5. **Q5**:GC(boehm)在嵌入式为什么至今小众？实时系统需要 deterministic free 时间，但 GC 是 stop-the-world 扫描；这与 AUTOSAR OS 的 deadline 监控直接冲突。**没有免费午餐**:dynamic memory 的「自动」必然付出实时性代价。

---

## 附录 · Action #2 复盘 · ch2 saferFree 与 realloc 增/减行为

### Case 1 · naive 重复 free(预期 abort,实际抓出)

```sh
$ ./ch02_saferfree 1
=== case1: naive double free ===
[case1] before free, pi=0x5e51a0e752b0 *pi=42
[case1] about to double-free...
free(): double free detected in tcache 2
```

**真实观察**:glibc 13 在 tcache 层就把 double-free 拦下，并 `abort()`。**`free()` 不再是「不可靠检测」的纯记录式 API**——这是 glibc 2.26+ 引入 tcache 后的安全强化。BSD/macOS 的 libc 行为可能不同(历史上是「静默 heap 损坏」)。

**对比 Reese 原书 p.48 的描述**「`free` does not detect double-free」:在 2013 年的 glibc 也是这样，但 2026 年的 glibc 已经收紧。这是个**「C 标准行为未变，但 runtime 默认收紧」**的活样本。

### Case 2 · safeFree 重复调用(预期 OK)

```sh
$ ./ch02_saferfree 2
=== case2: safeFree double free ===
[case2] before free, pi=0x5e51a0e752b0 *pi=42
[case2] after first safeFree, pi=(nil)
[case2] survived double safeFree
```

**真实观察**:`safeFree` 第一次调用后,`saferFree` 内部 `*pp = NULL` 把 `pi` 置 NULL;第二次调用 `*pp == NULL` 直接 return,**不触发 glibc 任何检测**。这正是 ch3 §「Writing your own free function」要的语义。

### Case 3 · realloc 行为矩阵

```sh
$ ./ch02_realloc_resize
=== realloc behavior matrix ===
[from_null] p=0x5bb16f5132b0, contents="0123456789AB"
[shrink]    p=0x5bb16f5132d0 q=0x5bb16f5132d0 same_block=YES
[grow]      p=0x5bb16f5132d0 q=0x5bb16f513320 moved=YES
[to_zero]   q=(nil)
```

四类行为全部命中:
- **`realloc(NULL, 16)`** → 与 `malloc(16)` 等价，得 `0x...b0`。
- **`realloc(p, 8)` shrink** → 复用 `0x...d0`,内容保留前 8 字符 `01234567`。
- **`realloc(p, 4096)` grow** → 搬移到 `0x...320`,旧指针 `p` 不再可用(书中反复警告)。
- **`realloc(p, 0)`** → glibc 返回 `(nil)`,等同于 `free(p)`。

**踩到的坑**:`-Wall -Wextra` 在两个文件都报出警告，因为程序中存在 `free(pi)` 后未用 `pi`,以及 `realloc(p, 4096)` 后未用 `p`。这是 **「现代编译器在你写错之前先标记你可能写错」** 的好习惯。

### 一行 repro(可在你机器上复现)

```sh
cd /home/liyahong/dev/read/notes/understanding-using-c-pointers/code-exercises
gcc -O0 -g -Wall -Wextra -o ch02_saferfree ch02_saferfree.c && ./ch02_saferfree 2
gcc -O0 -g -Wall -Wextra -o ch02_realloc_resize ch02_realloc_resize.c && ./ch02_realloc_resize
```

### 剩余未做的事

- 用 `valgrind --tool=helgrind` 跑多线程 alias 场景的 double-free,看 helgrind 报什么。
- 跑 `case_realloc_to_zero` 在 musl(Alpine)、FreeBSD libc 上的差异对照表。
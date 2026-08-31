# Ch3. Aggregates and Pointers

> 对应 PDF 物理页 83-150（印刷页 67-134）。本章是全书最长（68 页），覆盖 C 程序员必须掌握的"内存模型"：数组、指针算术、多维数组、`void*` 高阶回调、结构体（含排序指针数组）、`union`、`strtol` 字符串转换、堆存储（`malloc/calloc/realloc/free`）、嵌套堆、内存泄漏与 valgrind 诊断。**ch6 网络和 ch7 并发的所有代码都依赖本章**。

## §1 章节概述

1. **数组 = 同类型的固定大小集合**——`int arr[8]` 在栈上分配 8 个 int，**没有 bounds check**。索引 = 从 arr 起点的字节偏移除以元素大小；`arr[i]` 与 `*(arr+i)` 完全等价。
2. **指针算术按元素类型缩放**——`int* ptr = arr; ptr++` 跳过 4 字节（不是 1 字节）；`ptr + n` 实际地址 = `ptr_base + n * sizeof(*ptr)`。编译器帮程序员做这件事，代价是**指针必须有类型**。
3. **`i[arr]` 合法但丑**——`arr[i] == *(arr+i) == *(i+arr) == i[arr]`。Kalin 把它叫"obfuscated C"，但它是 C 标准规定的语法糖。
4. **多维数组在内存里是一维 row-major 布局**——`table[3][4]` 共 12 个 int，编译器**保证** `table[i][j]` 与 `table[0][0] + i*4 + j` 同地址。Fortran 是 column-major。
5. **C 永远是 call by value**——`func(arr, n)` 传的是 arr 的地址副本（**不是数组副本**），所以改 `arr[i]` 会改原数组。**结构体按值传**会复制整个结构体（Kalin 演示 2.8MB 结构体按值传 = 灾难）。
6. **`void*` = 通用指针**——可与任何指针类型相互隐式转换（C 允许的方向反了，C++ 要求显式 cast）。`NULL` = `((void*)0)`，三种"零"含义：boolean false / `\0` 字符串结尾 / NULL 指针。
7. **高阶函数 = 函数指针做参数**——`qsort(void* base, size_t nmemb, size_t size, int (*comp)(const void*, const void*))` 是模板。**比较函数**返回 0 / 负 / 正 → 等 / 小 / 大；程序员写 `comp` 控制升降序。
8. **`typedef` 简化函数指针**——`typedef int (*reducer)(int[], int);` 后 `reducer r = sum;` 干净。**K&R 风格的 `int(*comp)()` 难读**——这就是为什么 ROS2、libuv 等大库大量用 typedef。
9. **结构体** ——`struct` 是"不同类型字段的集合"；字段名访问用 `.`；指针访问用 `->`；结构体名**不是**指针（**与数组不同**）。`sizeof(struct)` 经常大于字段总和（alignment padding）。
10. **排序大结构体 = 排序指针**——`qsort` 移动 8KB 结构元素 vs 移动 8B 指针，差 1000×。**多索引数组**让你能按 ID、按 salary、按 years 排序，**不移动原数据**。
11. **`union` = 字段共享存储**——`sizeof(union) = max(字段大小)`；赋值会覆盖其他字段。**网络协议** / **类型擦除** / **variant 类型**都用它。
12. **`atoi` / `strtol` / `sscanf`** 三种字符串转换：atoi 简单但无错误信息（`"1z2q"` 解析为 1）；strtol 通过 `char** endptr` 指出剩余部分；sscanf 最通用但最慢。**`atoi("foo")` = 0**（不是错误），所以**生产代码**几乎都该用 `strtol`。
13. **堆是程序员的全部责任**——`malloc/calloc/realloc/free` 四件套；**`malloc` 不初始化**，`calloc` 初始化为 0；**`realloc` 失败返回 NULL 但原指针仍有效**（不释放原内存）；`free` 不设指针为 NULL（**use-after-free 经典源**）。
14. **嵌套堆要嵌套释放**——`HeapStruct` 含 `float* heap_nums`，`free_all` 必须**先 free 内层数组再 free 外层结构**，否则**内层字节泄漏**。**库函数应提供"二合一 free"**（如 `free_all`）。
15. **泄漏的根因 = 指针被覆盖**——`ptr = malloc(...); ptr = malloc(...);` 第一段就泄漏了。**`valgrind --leak-check=full`** 是诊断金标准；**`ASan`（Address Sanitizer）** 也抓且比 valgrind 快 10×。
16. **C 没有 GC**——Java/Python/Go 的"自动回收"在 C 里**全靠程序员**。这是 C 的"高性能合约"——也是它的最大坑。

## §2 核心 Takeaways

### T1 — `arr[i]` ≡ `*(arr+i)`：数组下标是"指针偏移"的语法糖
- **是什么**：`arr[i]` 在编译器视角就是 `*(arr+i)`，含义 = "从 arr 地址起 + i 个元素步长，再解引用"。`i[arr]` 同理（加法交换律）。
- **为什么重要**：理解 C 数组必须理解"数组名 = 首元素地址常量"；理解 a[i] 必须理解指针算术。**这是 C 内存模型的核心心智模型**。
- **解决什么**：所有多维数组、字符串、`qsort` 调用、回调函数都依赖这个心智模型。
- **适用场景**：所有 C 代码；尤其**调 bug** 时 `gdb` 打印 `arr+i` 看偏移。

### T2 — C 没有 bounds check，索引越界 = 静默 UB
- **是什么**：`int arr[4]; arr[-9876] = 27;` 编译无 warning，运行时**大概率段错误**，但**不保证**。**越界读** 可能读到"看起来对"的数据（栈邻居、堆碎片），让 bug 难复现。
- **为什么重要**：C 标准明确"未定义行为"——编译器**可以**假设不越界而激进优化。Linux kernel 用 `CONFIG_DEBUG_LIST` / `__list_add` 边界检查；glibc 用 `<sanitizer/asan_interface.h>`。
- **解决什么**：用 `BOUND_CHECK(arr, i, n)` 宏；用 `std::span` (C++20) 或 `bounds.h` 包装；编译开 `-fsanitize=address,undefined`。
- **适用场景**：所有 C 代码；尤其**网络协议解析**（用户输入长度可变）、**JSON/XML 解析**、**数据库**。

### T3 — 多维数组在内存里是 1D 连续的 row-major
- **是什么**：`int t[3][4]` 实际是 12 个 int 紧挨着：t[0][0], t[0][1], t[0][2], t[0][3], t[1][0], ...。`&t[i][j] = &t[0][0] + i*4 + j`（4 = 列数）。
- **为什么重要**：**指针偏移算术**能跨多维——`int* p = (int*)t; p[7] = t[1][3]`。C 性能优化**全部**靠这点（循环展开、SIMD 拉直、cache 友好）。
- **解决什么**：矩阵计算、图像处理（RGB 像素 = 1D 数组）、神经网络张量（**所有 tensor 在内存里都是 1D**）。
- **适用场景**：BLAS/LAPACK、CUDA tensor core、NumPy 内部布局（NumPy 默认 C-contiguous = row-major）、ROS2 costmap 2D 数组。

### T4 — `void*` = 通用指针；`qsort` 模式 = 模板的祖先
- **是什么**：`void*` 不指向任何特定类型；可与 `T*` 隐式互转（C 方向）。`qsort` 第一个参数是 `void*`，可排序**任何**类型数组——代价是 `comp` 回调需要自己 cast。
- **为什么重要**：C 标准库（`memcpy/memset/memcmp`、`qsort/bsearch`、`pthread_create`）几乎全用 `void*` 实现"泛型"。**所有 C 模板/泛型的 DNA**。
- **解决什么**：写通用库、回调机制、跨类型数据处理。
- **适用场景**：所有 C 库；尤其 `bsearch`、`qsort`、`qsort_r`（GNU 扩展，`comp` 多带 user_data 形参——POSIX 1.2008 标准化为 `qsort_r` 但签名不同）。

### T5 — `typedef` 函数指针是高阶函数可读性关键
- **是什么**：`typedef int (*reducer)(int[], int);` 后 `reducer r = sum;` 与 `int (*r)(int[], int) = sum;` 等价，但前者在函数签名里**极易读**——尤其当函数指针做参数时。
- **为什么重要**：ROS2、`libuv`、Linux kernel (`work_func_t`)、glib (`GCompareFunc`) 全是 typedef 函数指针。**没 typedef 的 C 代码 = 不可维护的 C 代码**。
- **解决什么**：API 文档、IDE 跳转、代码 review。
- **适用场景**：所有 C 库的回调 API；尤其**事件驱动编程**（epoll、kqueue、libev 的 watcher 回调）。

### T6 — `struct` 按值传 = 性能陷阱；按指针传 = 规则
- **是什么**：`void bad(BigNumsStruct arg)` 复制 2.8MB 到栈；`void good(BigNumsStruct* ptr)` 复制 8B 指针。Kalin ch3.11 用 2.8MB 演示差异。
- **为什么重要**：结构体 > ~32B 几乎必须按指针传；Linux kernel `container_of` 宏全靠结构体指针。**嵌入式**结构体可能更小（32-128B）——但仍按指针传以省栈空间。
- **解决什么**：性能、栈溢出防御。
- **适用场景**：所有 C 函数参数；尤其**库 API 设计**（`void*` + size vs `struct X` by value）。

### T7 — `union` = 类型擦除 + 节省空间
- **是什么**：`union { double d; long l; } v;` 大小 = 8B（不是 16B），任一时刻只能存其一。**赋值会覆盖其他字段**。
- **为什么重要**：网络协议（同一字段可能是 IPv4 地址或 IPv6）、变体类型（JSON value）、FFI 跨语言（OpenSSL、CUDA）、内存敏感场景。
- **解决什么**：多协议解析、tagged union 实现、union-find 数据结构。
- **适用场景**：网络协议栈（IP 头、TCP 选项）、解释器、数据库 BSON 解析。**MISRA-C 禁用 union**（汽车电子）——这是反例。

### T8 — `strtol` + `endptr` 是字符串转换的"专业版"
- **是什么**：`strtol(s, &endptr, base)` 返回数值，并把"剩余未解析部分"的指针写到 `endptr`。**可检查转换是否完全成功**：`if (*endptr != '\0') error;`。
- **为什么重要**：`atoi("foo") = 0`（不是错误码！），`atoi("999999999999")` 会溢出（UB）。`strtol` + `errno = ERANGE` 检查才能判断溢出。**生产代码必用**。
- **解决什么**：CLI 参数解析、配置文件读取、网络消息解析。
- **适用场景**：所有接受"用户输入字符串"转数字的代码。**ROS2 节点参数**、`getopt` 内部、systemd unit 解析都用。

### T9 — `malloc` 返回值**必须**检查 NULL
- **是什么**：`void* p = malloc(n);` 可能返回 NULL（内存不足 / 超过 `RLIMIT_AS`）。**现代系统几乎不会 OOM**，但嵌入式 / 容器内仍可能。
- **为什么重要**：`if (!p) { perror("malloc"); exit(1); }` 是基线。Linux kernel 用 `kmalloc` 返回值检查（GFP_ATOMIC 失败率高）。**ROS2 rcl_allocator.c** 也强制。
- **解决什么**：运行时崩溃防御、CI 在 stress 模式下复现 OOM。
- **适用场景**：所有堆分配；尤其**长跑 daemon**（systemd service、ROS2 node）、**内存敏感嵌入式**。

### T10 — `free` 不设指针为 NULL = use-after-free 温床
- **是什么**：`free(p);` 之后 `p` 仍持有原地址（**已失效**）。**再次 `free(p)` = double-free**（glibc 检测到会 abort）；`p[0] = 1;` = use-after-free（**ASan 抓，valgrind 抓**）。
- **为什么重要**：**60% 的 C 漏洞（CVE）都跟内存安全有关**。Linux kernel 用 `kfree(p); p = NULL;` 配合 `kfree(NULL)` 安全的特性。
- **解决什么**：所有 C 代码的硬规则。
- **适用场景**：所有 C 代码；尤其**多线程**（一个线程 free，另一个线程还在用——TOCTOU bug）。

### T11 — `valgrind` + `ASan` 是内存 bug 终极诊断器
- **是什么**：`valgrind --leak-check=full ./prog` 跑在 VM 里，能抓**泄漏、use-after-free、未初始化读**。`ASan`（`-fsanitize=address`）编译期插入检查，**比 valgrind 快 10×**——CI 必用。
- **为什么重要**：C 没有运行时安全网，必须靠工具。**Kalin 明确说"valgrind is my favorite"**——你读 Effective C 时应该已经熟。
- **解决什么**：CI 流水线抓内存 bug、压力测试、回归。
- **适用场景**：所有 C 项目 CI；尤其**库代码**（libuv、SQLite、glibc 都有 ASan nightly build）。

## §3 工程实践视角

### 3.1 Linux 系统开发视角

- **`<stdint.h>` + `qsort` 是数据处理组合拳**——ROS2 消息、`struct` 数组、动态数据全部走 `qsort` + `typedef` 比较函数。**`qsort_r`（POSIX 1.2008）**带 user_data，支持 closure——比 `qsort` 好用。
- **结构体对齐**（alignment）规则：字段按其大小对齐；`struct { int n; char c; double d; }` 大小 = 16（不是 13）。**`-Wpadded` warning** 帮你发现 padding。**`#pragma pack(1)`** 关对齐（用于网络协议、磁盘格式）——但性能下降。
- **`_Generic` (C11) + `void*` = 轻量泛型**——`#define PRINT(x) _Generic((x), int: print_int, double: print_double, default: print_ptr)(x)`。**Linux kernel 用 `_Generic` 做 `min/max` 的 type-safe 版本**。
- **`flexible array member` (C99)**：`struct Buf { size_t len; char data[]; }` 末尾不定长——`malloc(sizeof(Buf) + n)` 一次分配。**Redis、libuv** 全用这个模式。
- **零拷贝（zero-copy）** = `mmap` / `sendfile` / `splice`——ch6 网络服务器性能关键。`read()` + `malloc` + `write()` 是 2 次拷贝；`sendfile()` 是 0 次。
- **`valgrind --tool=memcheck` vs `--tool=helgrind`**：前者抓内存，后者抓**线程竞争**（ch7 用）。
- **Address Sanitizer (ASan) + UBSan 是 CI 双雄**：
  ```bash
  gcc -fsanitize=address,undefined -fno-omit-frame-pointer -g -O1 foo.c -o foo
  ```
  性能开销 ~2×；内存开销 ~3×；CI 必开。**Linux kernel 用 KASAN**（kernel ASan）。
- **多维数组性能 = 数据布局**：`int[N][M]` 顺行访问 (i 外 j 内) cache 友好；**反行访问** (i 内 j 外) cache miss ×N。**OpenCV / NumPy / Eigen 文档反复强调这点**。
- **`bsearch` vs 自定义二分**：`bsearch(key, base, n, size, comp)` 是 `<stdlib.h>` 标准函数，与 `qsort` 同模式。`tsearch`/`tdestroy` 是红黑树（GNU 扩展，glibc 里有）。

### 3.2 机器人软件视角（ROS2 / 嵌入式控制）

- **ROS2 IDL → C 结构体的对等**：`sensor_msgs/msg/PointCloud2` 生成 `struct PointCloud2 { uint32_t height; uint32_t width; ...; uint8_t data[]; }`。**末尾 `data[]` = flexible array member**。
- **点云处理 = 多维数组的活教材**——`PointCloud2::data` 是 1D `uint8_t[]`（`width * height * point_step`），但逻辑上是 `height × width × point_step` 3D。**ROS2 / PCL 用 `reinterpret_cast<>`** 跨类型访问——**Kalin ch3.5 的"二维当一维"** 就是这模式。
- **costmap 2D 数组** = `nav2_costmap_2d::Costmap2D` 内含 `std::vector<double>`，用 `(mx, my)` 索引映射到 `data[my * size_x + mx]`。**这正是 ch3.5 的 `table[i][j] == p[i*4 + j]` 实战版**。
- **TF 树** = `geometry_msgs/TransformStamped` 数组（`tf2_msgs/TFMessage.transforms`）——链表式存储。**结构体指针数组**就是 ROS2 消息总线。
- **消息序列化**（CDR / protobuf）= `memcpy` + 字节序转换。**字节序**用 `<endian.h>` 的 `htonl/htons`；**不要用** `((char*)&n)[0] = ...` 拼字节（strict aliasing UB）。
- **ros2_control 的 hardware interface** = `struct Controller` 含 `joint_handles_[]`（关节状态指针数组）——**ch3.8.1 排序大结构体** 的实操。**多个 controller 同时跑**需要共享内存（ch7）。
- **MoveIt 运动学** = `Eigen::Isometry3d`（C++）但底层是 `double[16]`（4×4 矩阵平铺）——ch3.5 1D 连续。
- **实时循环的"零分配"原则**——ch5/ch7 详解。简言之：ROS2 real-time node 不能在 `while(loop)` 里 `malloc`，必须 pre-allocate pool（ch3.10 的 `malloc/free` 成本 = 数百 ns，不确定）。**RT-Preempt 内核** + `mlockall(MCL_CURRENT)` 锁页。

### 3.3 初级 vs 高级工程师对照

| 习惯 | 初级 | 高级 |
|---|---|---|
| 数组初始化 | `int arr[1000];` 不初始化 | 静态数据用 `static const`；运行时用 `calloc` 或 `memset` |
| 数组访问 | `arr[i]` | 始终想 `*(arr+i)` 边界 |
| 多维数组 | 嵌套 for 循环 | 思考 row-major + cache 局部性；优先单循环拉直 |
| `malloc` 后 | 不用 | **必检查 NULL** + 记下谁 free |
| `free` 后 | 继续用 `p[i] = 1` | `free(p); p = NULL;` 杜绝 use-after-free |
| 字符串转数字 | `atoi(s)` | `strtol(s, &end, 10)` + `errno` 检查 |
| 结构体传参 | 按值传 | 按指针传；>32B 必传指针 |
| 排序大结构 | `qsort(arr, n, sizeof(S), comp)` | **先排指针数组**（ch3.8.1） |
| 函数指针 | 不写 | typedef 化，参考 `<signal.h>` `sighandler_t` |
| 内存诊断 | 出问题才跑 valgrind | CI 必跑 ASan/UBSan + 偶发 valgrind |

## §4 AI 时代视角

### 4.1 这些知识还重要吗？（2026 年视角）

**极重要。** 现代 C/C++ 工程（操作系统、数据库、编译器、机器学习框架、机器人栈）**80% 的代码是 ch3 范畴**。AI 生成的 C 代码在 ch3 的错误率**高于 ch1/ch2**——因为 ch3 涉及"内存模型"心智模型，LLM 训练数据里**指针算术、结构体布局、堆管理** 的"正确实现"密度比 Python 类低很多。

现代工程师的 ch3 日常：
- **看 AI 生成的 C 代码** = 反复检查 ch3 这套契约
- **修 AI 写的 bug** = 90% 是 use-after-free / double-free / 越界
- **性能调优** = 1D 拉直、cache 友好、零分配

### 4.2 AI 现在能做的

- ✅ 写 `qsort` + 比较函数的完整模板
- ✅ 写 `malloc/calloc/realloc/free` 的常规用法
- ✅ 解释 row-major vs column-major
- ✅ 写 `typedef` 函数指针
- ✅ 写 `strtol` + `endptr` 错误检查

### 4.3 AI 经常写错的地方（必看）

| 错误模式 | 例子 | 后果 |
|---|---|---|
| **1. `free` 后继续用** | `free(p); *p = 1;` | use-after-free；ASan 抓；LLVM/Chrome 多个 CVE |
| **2. 漏 `free`** | `malloc` 后没 free | 内存泄漏；长跑 daemon OOM；valgrind 抓 |
| **3. `realloc` 后指针丢失** | `p = realloc(p, 2n); if (!p) return;` 错误 | 原指针丢失（realloc 已 free）；正确：`new_p = realloc(p, 2n); if (!new_p) { free(p); return; } p = new_p;` |
| **4. 返回局部数组** | `int* f() { int a[10]; return a; }` | 栈上数据函数返回后失效；UB；gdb 看就是垃圾 |
| **5. 数组名当 `&` 错用** | `scanf("%i", arr)` 应为 `&arr[0]` | 数组名已是指针，**不需要 `&`**；但 `arr[i]` 要 `&arr[i]` |
| **6. `struct* p; p->field;` 不查 NULL** | 结构体指针直接解引用 | 段错误；必须 `if (p) p->field` |
| **7. `qsort` 比较函数返回值溢出** | `return n1 - n2;`（n1, n2 接近 INT_MAX/MIN） | 整数溢出 UB；用 `return (n1 > n2) - (n1 < n2);` |
| **8. `void*` 强转不检查** | `int* p = (int*) some_void_ptr;` 后直接用 | 类型不对 → 错位访问；用 `memcpy(&n, buf, sizeof(n))` |
| **9. `sizeof(struct)` 当作字段总和** | `struct S { int a; char b; };` 以为 sizeof = 5 | 实际 8（alignment）；用 `offsetof(struct, b)` 查 |
| **10. 嵌套结构体忘 free 内层** | `free(outer);` 完事 | 内层指针指向的字节泄漏；ASan 抓 |
| **11. `calloc(0, 1)` 行为未定义** | `calloc(0, n)` 或 `malloc(0)` | 实现自由：glibc 返回非 NULL 指针；musl 返回 NULL；**避免** |
| **12. `strtol` 不设 errno** | `long n = strtol(s, NULL, 10);` | 溢出无感知；必须 `errno = 0; strtol(...); if (errno == ERANGE) ...` |
| **13. `typedef` 函数指针签名错** | `typedef void (*cb)(int); cb = my_func;` 但 my_func 返回 int | 编译错（不易懂） |
| **14. 越界 strcpy** | `strcpy(buf, src);` buf 小于 src | **栈缓冲区溢出** = CVE 头号源；用 `strncpy` 或 `snprintf` |
| **15. `union` 字段 type-punning** | `union { int i; float f; }; u.i = 1; printf("%f", u.f);` | C 标准 UB（strict aliasing）；用 `memcpy(&f, &i, 4)` |

### 4.4 工程师必须保留的核心能力

- **能 5 秒内口算 `arr[i]` 的字节地址**——`base + i * sizeof(*ptr)`。
- **能 30 秒内写出"嵌套堆释放"模式**——`free(inner); free(outer); p = NULL;`。
- **能用 ASan 编译并解读报告**——CI 必备。
- **能区分 row-major / column-major**——与 Fortran/Numpy/Blas 互操作。
- **能跟 AI 说"请用 `strtol` + `errno` 检查 + ASan 编译"**——ch3 词汇是 prompt 的关键。

### 4.5 wasm 工具链延伸（用户已选需要）

- **wasm 的线性内存 = C 堆**——wasm 模块的整个地址空间是一个 `WebAssembly.Memory` 对象，C 的 `malloc` 在这之上实现。**ch3.10 的 `malloc`/`free` 在 wasm 里完全有效**。
- **wasm 没有"栈指针"概念**——但 wasm 编译器（emcc）会模拟 C 栈。**`alloca` / VLA 在 wasm 里开销大**（要 grow 内存）。
- **wasm 的 `sizeof(void*)` = 4**（wasm32）——比 host x86-64 的 8 小。**结构体含指针的 sizeof 在 wasm 里与 host 不同**——LLM 不会自动适配。
- **AI 工具链中 wasm 的 ch3 角色**：Claude Code / Codex 跑用户 C 代码（沙箱）时，**整个 ch3 的内存模型是 wasm 字节码层**。**ASan 在 wasm 里也工作**（emcc 支持）——CI 在 wasm 编译目标上跑 ASan 是双重保险。

## §5 实践行动项

### A1 — 验证 `arr[i]` 字节地址 = 指针算术
```bash
mkdir -p /tmp/modern-c/ch03 && cd /tmp/modern-c/ch03
cat > ptrarith.c <<'EOF'
#include <stdio.h>
int main(void) {
    int arr[5] = {10, 20, 30, 40, 50};
    int* p = arr;
    for (int i = 0; i < 5; i++) {
        printf("arr[%d]=%d  *(arr+%d)=%d  *(p+%d)=%d  p[%d]=%d  addr=%p  addr_off=%td\n",
               i, arr[i], i, *(arr+i), i, *(p+i), i, p[i], (void*)&arr[i], (void*)(p+i)-(void*)p);
    }
    printf("\ni[arr] trick: %d == %d (合法但丑)\n", 2[arr], arr[2]);
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -pedantic -o ptrarith ptrarith.c && ./ptrarith
```
**验收**：看到 `addr_off` 分别为 0,4,8,12,16（int 4B × 索引）；`2[arr] == 30`。

### A2 — 用 `qsort` + `typedef` 排序结构体指针数组
```bash
cat > qsortstruct.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
typedef struct { int id; double score; char name[32]; } Record;
typedef int (*cmp_fn)(const void*, const void*);
static int cmp_id(const void* a, const void* b) {
    const Record* ra = *(const Record* const*)a;  /* 双重解引：指针的指针 */
    const Record* rb = *(const Record* const*)b;
    return (ra->id > rb->id) - (ra->id < rb->id);  /* 防溢出 */
}
int main(void) {
    Record data[3] = {{2, 9.5, "Bob"}, {1, 8.0, "Alice"}, {3, 7.2, "Carol"}};
    Record* ptrs[3] = {&data[0], &data[1], &data[2]};
    qsort(ptrs, 3, sizeof(Record*), cmp_id);
    for (int i = 0; i < 3; i++) printf("%d %s %.1f\n", ptrs[i]->id, ptrs[i]->name, ptrs[i]->score);
    /* data 数组未被移动，仅指针数组排序 */
    printf("orig order: %d %d %d\n", data[0].id, data[1].id, data[2].id);
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -pedantic -o qsortstruct qsortstruct.c && ./qsortstruct
```
**验收**：输出按 id 升序的指针遍历结果，但原 `data` 数组顺序不变（ch3.8.1 "排序指针而非数据"）。

### A3 — 用 `strtol` 写带错误检查的字符串转 int
```bash
cat > str2int.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <limits.h>
int main(int argc, char* argv[]) {
    if (argc != 2) { fprintf(stderr, "Usage: %s <int>\n", argv[0]); return 1; }
    errno = 0;
    char* end = NULL;
    long n = strtol(argv[1], &end, 10);
    if (errno == ERANGE)            { fprintf(stderr, "overflow\n");  return 1; }
    if (end == argv[1] || *end != '\0') { fprintf(stderr, "bad input: %s\n", argv[1]); return 1; }
    if (n < INT_MIN || n > INT_MAX) { fprintf(stderr, "out of int range\n"); return 1; }
    printf("n = %ld\n", n);
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -Werror -o str2int str2int.c
./str2int 42; ./str2int 999999999999; ./str2int foo; echo "exit=$?"
```
**验收**：`42` OK；`999999999999` 报 overflow；`foo` 报 bad input；所有错误路径 exit=1。

### A4 — 用 ASan 抓 use-after-free
```bash
cat > uaf.c <<'EOF'
#include <stdlib.h>
int main(void) {
    int* p = malloc(sizeof(int));
    *p = 42;
    free(p);
    *p = 99;            /* use-after-free: ASan 抓 */
    /* double-free: 也抓 */
    free(p);
    return 0;
}
EOF
gcc -std=c11 -fsanitize=address -fno-omit-frame-pointer -g -O1 -o uaf uaf.c
./uaf
# 期望: AddressSanitizer: heap-use-after-free
```
**验收**：ASan 报 `heap-use-after-free` 并指出 `p` 来自 `malloc`，释放后再写的位置。删 `*p = 99;` 后 clean。

### A5 — 用 valgrind 抓内存泄漏（ch3.12.2 实战）
```bash
which valgrind || sudo apt install -y valgrind
cat > leak.c <<'EOF'
#include <stdlib.h>
int main(void) {
    int* p = malloc(100 * sizeof(int));
    p[0] = 1;
    return 0;  /* p never freed → leak */
}
EOF
gcc -std=c11 -Wall -o leak leak.c
valgrind --leak-check=full --error-exitcode=1 ./leak
# 期望: "definitely lost: 400 bytes in 1 blocks"
# 改 free(p) 后: "All heap blocks were freed -- no leaks are possible"
```
**验收**：valgrind 报 400 bytes 泄漏；加 `free(p);` 后变 0。

## §6 值得深入思考的问题

1. **`qsort_r` 签名为什么 POSIX 和 glibc 不同？** POSIX 定义 `qsort_r(base, nmemb, size, comp, arg)`，glibc 旧版用 `qsort_r(base, nmemb, size, comp, arg)` ——实际签名是 `(base, nmemb, size, comp)` 中 comp 接受 `void* arg` 作为**第三参**。**为什么**？历史包袱：BSD vs System V。
2. **多维数组的"维度参数"为什么只能是第一个之外的全部？** C 函数参数里 `void f(int arr[][4][5])` 合法（最右两维必须写），但 `void f(int arr[][][5])` 不合法（中间维度必须指定大小）。**为什么 row-major 让"右维度"必须固定？** 这跟 CPU 寻址计算怎么从 `arr[i][j][k]` 算出字节偏移有关。
3. **`flexible array member` (`T data[]`) 为什么 C99 才进标准？** C89 的"动态结构体"需要"hack"——先 `malloc(sizeof(S) + n)`，再把 `data` 强转为 `(char*)(s+1)`。**C99 怎么解决了类型安全问题？** 为什么 Linux kernel `commit 4a8d4d0` 全面切换？
4. **结构体 padding 是怎么决定的？** `struct { int n; char c; }` 大小 8B 不是 5B。**`#pragma pack(1)` 关 padding** 后大小 5B，但访问效率下降（未对齐访问）。**x86 容忍未对齐，ARM strict alignment 时会 fault。** 嵌入式 / 网络协议怎么平衡？
5. **GC 在 C 里可行吗？** 历史上 `Boehm-Demers-Weiser GC` 是 C/C++ 的 conservative GC 库；`libgc` 是其开源版。**为什么工业界几乎不采纳？** 实时性不可控（GC pause 几百 ms）；内存开销 2-3×。**Rust 也没用 GC**——用 ownership/borrow checker 静态保证。
6. **ASan vs valgrind 怎么选？** ASan 编译期插入检查，2× 慢；valgrind VM 模拟，10× 慢。**为什么 Linux kernel 用 KASAN 而非 valgrind**？**ROS2 rcl 和 rclcpp 的 CI 用 ASan 还是 valgrind？**（提示：CI 资源敏感时 ASan 优先；深度 bug 排查时 valgrind 优先）
7. **`qsort` 排序指针数组 vs 直接 `qsort_r` 用 user_data**——前者**全局共享 comp**（无闭包），后者**每实例不同 comp**。**STL 的 `std::sort` 用 `Compare comp` 对象**——是 C++ closure 的同款思路。C 没有 closure，怎么模拟？`qsort_r` 是答案吗？

---

*下一章预告*：**ch4 Storage Classes**——16 页短章，但**是 ch3 内存模型的"持久化层"**。覆盖 `auto`（默认栈局部）/ `register`（请求放寄存器，**C++17 deprecated，C23 仍保留**）/ `static`（函数内静态变量=持久存储；文件内静态=内部链接）/ `extern`（跨文件引用）/ `volatile`（防优化器消除读——嵌入式寄存器、信号 handler 共享变量、MMIO 必备）。ch7 共享内存 + ch8 信号 + 后续嵌入式 MCU 编程都靠这章的 `static`/`volatile` 关键字。

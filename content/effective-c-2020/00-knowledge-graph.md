# Effective C 知识体系总图

> 来源: *Effective C* (Seacord, 2020) 11 章六段式精读
> 落盘日期: 2026-08-31
> 用途: **工作中随时反查**——按"工程能力维度"重组，不是按章节复述
> 阅读量: 全文 ~3500 行章节笔记的索引层

## 总图导览

本书 11 章内容**横向切**成 5 个工程能力维度：

| 维度 | 章节范围 | 核心问题 |
|---|---|---|
| **A. C 语言哲学与标准** | ch1, ch11 | "为什么 C 这样设计？" "什么时候信编译器？" |
| **B. 类型与表示** | ch2, ch3, ch7 | "这个值在内存里长什么样？" |
| **C. 表达式与控制流** | ch4, ch5 | "这句话执行顺序如何？" |
| **D. 资源管理** | ch6, ch8, ch9 | "谁拥有这块内存？这块数据哪儿来？" |
| **E. 工程结构与质量** | ch10, ch11 | "代码怎么组织？怎么怎么保证质量？" |

每个维度下面，按"**知识卡片**"组织——每张卡片都是工作中**立即可用**的"该做什么/不该做什么"。

---

# A. C 语言哲学与标准

## A1. The Spirit of C (ch1)

五条信念（设计哲学）：

1. **Trust the programmer** — 但 50 年历史证明不可信
2. **Don't prevent** — 给你 raw pointer、manual memory
3. **Keep small and simple** — 语言层面只有 32 个关键字（C99）
4. **One way to do it** — 但实际不严格（`int i;` vs `signed int i` 都合法）
5. **Make it fast** — 可移植性是你（程序员）的责任

**工程含义**：每个"自由"对应一个"陷阱"。专业 = 用工具 + 流程把"trust"变"verify"。

## A2. C 标准的权威性 (ch1)

C 标准 (ISO/IEC 9899) **是行为的最终裁判**，但给实现留了大 latitude：

- **C89/C90** → C99 → **C11** → **C17/C18** → **C2x (= C23)**
- 每个编译选项组合（compiler + flags + lib）= 一个独立 implementation
- GCC 默认 `gnu17`，**生产代码显式 `-std=c17`**

**反模式**："写简单测试看行为"——同一代码不同 implementation 不同行为。**C 标准是唯一权威**。

## A3. Undefined / Unspecified / Implementation-defined 三态 (ch1)

| 类型 | 含义 | 例子 |
|---|---|---|
| **Undefined (UB)** | 标准不规定，编译器可做任何事 | signed overflow、`*(NULL)`、`i++*i++` |
| **Unspecified** | 标准给 ≥2 选项，每次可选不同 | 函数参数求值顺序 |
| **Implementation-defined** | 编译器必须文档化结果 | `char` 是 signed 还是 unsigned |
| **Locale-specific** | 依赖 locale 设置 | `strerror` 返回的本地化字符串 |

**核心原则**：**UB 是优化器燃料**——`x+1 > x` 在 INT_MAX 处恒真，编译器可删条件。**UBSan 是必备防御**。

## A4. Hosted vs Freestanding (ch1, ch6)

- **Hosted**：有 OS，main 固定，完整标准库
- **Freestanding**：裸机/嵌入式，main 由实现定义，只有 <float.h> <iso646.h> <limits.h> <stdarg.h> <stdbool.h> <stddef.h> <stdint.h> 子集

**工程含义**：嵌入式代码不能用 `puts` / `EXIT_SUCCESS` / heap / recursion——本书例子全 hosted。

---

# B. 类型与表示

## B1. Object / Function / Derived 三类类型 (ch2)

- **Object type**：值存在内存里（int、char、struct）
- **Function type**：`int (int, char *)` 是 type（不是签名）
- **Derived type**：pointer / array / struct / union / typedef

**读懂复杂声明**：用 **spiral rule**（顺时针螺旋）——找变量名 → spiral → 解读。

## B2. Scope ≠ Lifetime (ch2)

| 概念 | 管什么 | 例 |
|---|---|---|
| **Scope** | 标识符在源码哪里可见（编译期） | `int i` 在块内可见 |
| **Lifetime** | 对象在运行期什么时候存在 | `static int i` 程序启动到结束 |

**混淆陷阱**：`static int counter` 在函数内 → scope 是块内，lifetime 是程序级。函数返回后值仍存在。

## B3. 三种 Storage Duration (ch2, ch6)

| Duration | 关键字 | Lifetime |
|---|---|---|
| **automatic** | 块内、参数 | 块执行期间 |
| **static** | `static` / file scope | 整个程序 |
| **thread** | `_Thread_local` (C11) | 线程生命周期 |
| **allocated** | `malloc` family | 申请到 free |

**static 初始化必须用常量**（字面量、enum、sizeof/alignof）——**不能用 const 变量**。

## B4. 类型限定符三大金刚 (ch2)

| 限定符 | 含义 | 常见误用 |
| | | |
| **`const`** | 不可修改；编译器可放只读段 | `cast-away const` = UB（cast 改原 const 对象） |
| **`volatile`** | 禁止优化缓存；只用于 MMIO / 信号处理 | **不是线程同步原语**——Java/C# 语义 ≠ C 语义 |
| **`restrict`** | 承诺指针不别名——**违反即 UB** | 默认不加；memmove（保证 overlap）vs memcpy（不保证） |

**线程同步该用 `_Atomic`** 或 mutex，不是 volatile。Linux kernel 明确禁止 `volatile` 当同步。

## B5. `_Bool` / `char` / `wchar_t` / `char16_t` / `char32_t` (ch2, ch7)

- `_Bool`：C99 起的 0/1 类型，`<stdbool.h>` 暴露 `bool` `true` `false`
- `char`：implementation-defined signed/unsigned；**永远用 char 表字符**，`signed/unsigned char` 表整数
- `wchar_t`：Linux 32-bit / Windows 16-bit——**跨平台不要用**
- `char16_t` / `char32_t`：C11 起 `<uchar.h>`；UTF-16 / UTF-32 显式

**陷阱**：`char c = 'ÿ'; if (c == EOF)` — `ÿ` (0xFF) sign-extend 到 -1 == EOF。**`<ctype.h>` 调用必须 cast `(unsigned char)c`**。

## B6. 整数表示与算术 (ch3)

| 类型 | 范围 (x86) | 注意 |
|---|---|---|
| `int8_t` | -128..127 | 最负值绝对值不可表示 |
| `uint32_t` | 0..2^32-1 | wraparound well-defined |
| `int32_t` | -2^31..2^31-1 | overflow = **UB** |

**B-787 教训**：发电机控制单元 32-bit counter 248 天 wraparound → fail-safe。**写永远循环用减法**：`for(unsigned i=n; i>0; --i)`——**不是 `i>=0`**。

## B7. 整数运算四陷阱 (ch3)

1. **unsigned wraparound** = well-defined，但循环条件、范围检查常踩
2. **signed overflow** = UB，编译器可可优化掉比较
3. **mixed-sign 比较** = 隐隐提升 + 转换（`c == ui` 可能为真）
4. **`Abs(INT_MIN)`** = UB——`glibc abs` 用 cast-to-unsigned 规避

**wraparound 检查范式**：
```c
// 错误：wrap 已在测试里发生
if (a + b > UINT_MAX) overflow();
// 正确：减法不可能 wrap
if (a > UINT_MAX - b) overflow();
```

## B8. 浮点 = 不是实数 (ch3)

- **不结合**、**不分配**、`0.1` 不不可精确表示
- **永远不做循环计数器**（CERT FLP30-C）
- **NaN 传染**：`NaN == NaN` 为 false；`NaN !=` = true
- 浮点有三种 zero（+0/-0），三种"长度"（bytes/code units/code points/extended grapheme clusters）

## B9. 字符集与字符串 (ch7)

| 编码 | 字节 | 平台 |
|---|---|---|
| ASCII | 7-bit | 基础 |
| **UTF-8** | 变长 1~4 | **POSIX / Linux / macOS 默认** |
| UTF-16 | 变长 1~2 个 16-bit code unit | Windows 默认 |
| UTF-32 | 定长 4 字节 | 索引 O(1)，空间 4× |

**跨平台字符串约定**：永远用 `char` + UTF-8。**不要用 `wchar_t`**。

**escape sequence 陷阱**：`'\10'` 是 octal 8 (backspace)，**不是 '1'+'0'**。同理 `'\x8'` = backspace。

## B10. 字符串函数四件套 (ch7)

| 函数 | 风险 |
|---|---|
| `gets` | C99 deprecated、C11 删除——**永远不要用** |
| `strcpy` | 无 size 参数 = buffer overflow |
| `strncpy` | **不保证 null 终止**（CERT STR32-C）——手动 `buf[n-1]='\0'` |
| `strlen(s)` | 遍历到 NUL；传未 null-terminated = buffer overrun |

**现代替代**：`fgets(buf, sizeof(buf), stdin)` —— 永远传 sizeof。

## B11. Annex K  失败案例 (ch7)

C11 Annex K 的 `strcpy_s` / `strcat_s` 等安全函数：
- **Microsoft 发起**（90 年代安全事件推动）
- **Linux/macOS 默认不实现**
- **MS 自己也不完全合规**（用旧 `_set_invalid_parameter_handler`）
- **C23 起 deprecated**

**教训**：标准化 vs 实际采用是两码事。

---

# C. 表达式与控制流

## C1. lvalue vs rvalue (ch2, ch4)

- **lvalue** = locator value = 指向可寻址对象（可放赋值左侧）
- **rvalue** = 一个值（不可放赋值左侧）
- `i + 12` 不是 lvalue——`i` 是，`i+12` 不是

## C2. Sequence Point 与 UB (ch4)

**跨 sequence point 改同一 scalar = UB**。Sequence point 在：
- `&&` / `||` / `?:` / `,` 操作数之间
- 完整表达式结尾
- 函数调用入口/出口

**`i++ * i++`** 是 UB——两个 side effect unsequenced。**正解**：拆完整表达式。

## C3. Order of Evaluation unspecified (ch4)

除 `&&` `||` `?:` `,` 外，**操作数求值顺序 unspecified**。`max(f(), g())` 中 f 和 g 哪个先调**没规定**。

**规范范式**：拆临时变量 —— caller 一行，callee 一行。

## C4. `a < b < c` 不是数学链式 (ch4)

C 解释为 `(a < b) < c`。`a < b` 返回 0 或 1，再与 c 比。

**正确写法**：`(a < b) && (b < c)`；CI 加 `-Wparentheses`。

## C5. 前置/后置自增 (ch4)

| 表达式 | 行为 |
|---|---|
| `i++` | 返回旧值，副作用递增 |
| `++i` | 返回新值（已是新值） |
| `++*p` vs `*p++` | 前者先 deref 后 inc；后者先 inc ptr 后 deref **旧**ptr |

## C6. Operator Precedence 15 层 (ch4)

postfix `()` `[]` `->` `.` > unary `++` `--` `*` `&` `sizeof` > `*` `/` `%` > `+` `-` > `<<` `>>` > `<` `<=` `>` `>=` > `==` `!=` > `&` > `^` > `|` > `&&` > `||` > `?:` > `=` > `,`

**`==` 和 `!=` 优先级低于 `<` `>`** —— `a < b == c < d` 是 `(a<b) == (c<d)`。

## C7. 位运算三原则 (ch4)

1. **永远用 unsigned** —— signed `>>` 是 implementation-defined（arithmetic vs logical）
2. **shift count 必检查**：`shift >= width` = UB；`shift < 0` = UB；signed left shift 溢出 = UB
3. **`~` 在小类型会 sign-extend** —— `~0xFF` 是 0xFFFFFF00（负），不是 0x000000FF

## C8. Pointer Arithmetic + Too-far pointer (ch4)

- `pi < &arr[N]` 的 `&arr[N]` 是"too-far pointer"——**合法**（只要不解引用）
- `pi + n` 越界 = UB；同数组内减法 well-defined
- `p + 1` 按 `sizeof(*p)` 缩放（不是字节）

## C9. 强制转型三大坑 (ch4)

1. **reinterpret bits**（`(intptr_t)ptr`）——不修改 bits
2. **change bits**（`(int)float`）——重新编码
3. **disable warning**——`(char)fgetc(in)` 静音 C4244 但**没修问题**

**`getchar`/`fgetc` 必须用 `int` 接**——强转 char 永远不可能等于 EOF。

## C10. 控制流五大陷阱 (ch5)

| 陷阱 | 后果 |
|---|---|
| **`if` 单行 + 后面语句** | indent 错觉；后面永远执行 |
| **`switch` 漏 `break`** | fall-through 到下一 case |
| **`switch` 无 `default`** | 新增 enum 值编译不报错 |
| **`for` expression3 位置** | `free(p); p = p->next` 中 p 已 free——UAF |
| **缺 `return` 路径** | 编译器不报，运行期 UB |

## C11. goto cleanup chain 是 idiomatic (ch5)

Dijkstra 1968 "goto harmful" 是反对"随意跳转"。**结构化 goto cleanup** 是合法的：

```c
if ((f1 = fopen(...)) == NULL) goto FAIL_F1;
if ((f2 = fopen(...)) == NULL) goto FAIL_F2;
...
FAIL_F2: fclose(f1);
FAIL_F1: return ret_val;
```

Linux kernel `copy_process` 用 **17 个 labels** 做 cleanup。

## C12. `do-while` 适合"先做再查" (ch5)

`do { fscanf(...); } while (!feof(stdin));` —— I/O 循环必须先读再判 EOF。

---

# D. 资源管理

## D1. 内存三态 (ch6)

| un | 合法操作 |
|---|---|
| **unallocated + uninitialized** |（属于 manager）） read/write/w write/free 都 UB |
| **allocated + uninitialized** | 可 write、可 free，**不能 read** |
| **allocated + initialized** | 都合法 |

## D2. malloc 三原则 (ch6)

1. **永远检查 NULL**——不查 = SIGSEGV
2. **永远初始化**——malloc 不清零，reading = UB
3. **size 用 `sizeof(T)`**——不要裸数字

## D3. realloc 三范式 (ch6)

```c
// 范式 1：失败保留旧指针
errno_t safe_realloc(void **p, size_t n) {
    void *newp = realloc(*p, n);
    if (!newp) return -1;
    *p = newp;
    return 0;
}

// 反范式：覆盖 = leak
p = realloc(p, n);   // 失败时 p=NULL，旧内存泄漏
```

**`realloc(p, 0)`** = C2x 起显式 UB——**不要传 0**。

## D4. free 三件事 (ch6)

1. **`free(NULL)`** = no-op，合法
2. **free 后置 NULL**——防 double-free 半层防御
3. **`free` 不能 cast 内存**（`free` 不知道原 malloc 函数）

**根治 use-after-free 需要 Rust 的 ownership 语义**——C 没有。

## D5. 柔性数组 vs struct hack (ch6)

```c
// 现代（C99）
typedef struct { size_t n; int data[]; } flex_t;
flex_t *p = malloc(sizeof(flex_t) + n * sizeof(int));

// 老式（struct hack）
typedef struct { size_t n; int data[1]; } hack_t;
hack_t *p = malloc(sizeof(hack_t) + (n-1) * sizeof(int));
```

柔性数组 `sizeof` 不算 `data[]`；CERT DCL38-C 推荐前者。

## D6. 安全关键系统禁用所有动态分配 (ch6)

- AUTOSAR / ISO 26262 ASIL-D
- MISRA C 21.3：禁 malloc/realloc
- MISRA 18.8 / 6.2：禁 VLA、alloca
- MISRA 17.2：禁 recursion（让 stack 可静态证明）

**static memory pool + 启动期一次性分配** = 唯一合规路径。

## D7. `volatile` 不是线程同步 (ch2, ch11)

**`volatile int flag = 1;`** 不保证跨线程可见性。需要：
- `_Atomic int flag`（C11）
- mutex / semaphore / condition variable
- `__atomic_store()` / `__atomic_load()`

## D8. I/O Buffering 三模式 (ch8)

| 模式 | 用途 | flush 时机 |
|---|---|---|
| **Unbuffered** | stderr（错误日志） | 立即 |
| **Fully buffered** | 文件 I/O | buffer 满 |
| **Line buffered** | 终端 | 遇 `\n` |

**`fflush` 在 last op 是 input 时 UB**。Write + read 中间必须 `fflush` / `fseek`。

## D9. I/O `fread`/`fwrite` 返回元素数 (ch8)

```c
if (fwrite(buf, 16, 4, fp) != 4) { /* partial write! */ }
```

**不是字节数**——partial read/write 看起来对但实际丢数据。

## D10. Big-endian vs Little-endian (ch8)

- x86 / AMD：little-endian
- 网络协议（IP/TCP/UDP）：big-endian
- ARM / POWER：可切换

**跨平台 binary I/O** 用 `texttext/ntotxt` 或固定 big-endian + 标识符。

## D11. Preprocessor 七武器 (ch9)

| 武器 | 用途 |
|---|---|
| `#include "x.h"` / `#include <x.h>` | 文件包含（用户头文件用引号，系统用用引号） |
| `#if` / `#elif` / `#else` / `#endif` | 条件编译 |
| `#ifdef` / `#ifndef` | `#if defined` 简写 |
| `#define` / `#undef` | 宏定义 / 取消 |
| `#error "msg"` | 编译期错误 |
| `#` / `##` | stringify / token paste（**只在函数宏里**） |
| `__`ST`` | C11 编译期类型分派 |

## D12. Macro 五陷阱 (ch9)

| 陷阱 | 后果 |
|---|---|
| **参数多次求值** | `bad_abs(i++)` 让 i 递增 2 次 (CERT PRE31-C) |
| **不 full par** | `SQUARE(1+2)` = 5 错；必须 `((x)*(x))` |
| **名字后空格** | `#define MAX (a,b)` 让 MAX 变 object-like，编译错 |
| **comma in arg** | `ATOMIC_VAR_INIT({1,2})` 错，逗号当分隔符 |
| **不 `#undef` 重定义** | 必须先 `#undef` 才能重定义 |

## D13. Header guard 模式 (ch9)

```c
#ifndef FOO_H          // 或 #ifndef FOO_BAR_BAZ_H
#define FOO_H
/* declarations */
#endif
```

**不要用 `_FOO_H`**——`_` + 大写是 reserved。**永远全大写但无下划线开头**。

## D14. 预定义宏 (ch9)

| 宏 | 内容 |
|---|---|
| `__FILE__` / `__LINE__` / `__func__` | 调试信息 |
| `__DATE__` / `__TIME__` | 编译时间 |
| `__STDC_VERSION__` | C 标准版本（`201710L` = C17） |
| `__STDC_NO_VLA__` / `__STDC_NO_ATOMICS__` / ... | 特性探测 |

---

# E. 工程结构与质量

## E1. Cohesion + Coupling 两轴 (ch10)

- **Cohesion（内聚）**：组件内元素相关度。`<string.h>` 高内聚；混 strlen+tan+thread 低内聚。
- **Coupling（耦合）**：组件间依赖度。**低耦合** = 头文件可单独 include。

好设计 = **高内聚 + 低耦合**。

## E2. Opaque Type 双 header (ch10)

```c
// 外部 header（用户可见）
typedef struct foo foo;
extern int create_foo(foo **out);
extern void destroy_foo(foo *f);

// 内部 header（实现可见）
struct foo { int x; int y; };
```

用户只见指针不见实现——真正的封装。换换数据结构零成本。

## E3. Self-contained Header (ch10)

你的 header 用了 `size_t` → `#include <stddef.h>`。**永远自包含**——不要依赖用户先 include 别的东西。IWYU。

## E4. 三种 Linkage (ch10)

| Linkage | 含义 | 用 |
|---|---|---|
| **External** | 跨 TU 同一 | 函数、全默认 |
| **Internal** | 仅本 TU  | 模块文件、私私有 |
| **none** | 参数、块变量、enum 常量 |

## E5. `static` 多义性 (ch10)

- `static` file scope = **internal linkage**
- `static` block scope = **static storage duration** + no linkage

## E6. Library 二选一 (ch10)

| 类型 | 优点 | 缺点 |
|---|---|---|
| **Static** | 启动快、部署简单、deterministic | 不能独立升级 |
| **Dynamic** | 升级灵活、节省内存、多进程共享 | 版本兼容风险、启动延迟 |

**AUTOSAR / 嵌入式全用 static**；**OS / 库全用 dynamic**。

## E7. Build 三步流程 (ch10)

```bash
cc -std=c17 -Wall -Wextra -Wpedantic -Werror -c foo.c -o foo.o
ar rcs libfoo.a foo.o              # r=replace, c=create, s=index
cc bar.o -L. -lfoo -o app           # -L=搜索路径, -l=库名（去 lib 前缀）
```

## E8. `static_assert` vs `assert` (ch11)

| | `static_assert` | `assert` |
|---|---|---|
| **触发时机** | 编译期 | 运行期 |
| **可表达** | 常量表达式 | 任何 scalar expression |
| **release 还在？** | 是 | 否（NDEBUG 关掉） |
| **用于** | 假设 / 契约 | precondition/postcondition/invariant |

**`assert(p != NULL)`** 是 OK（API misuse）。**`assert(fopen(...) != NULL)`** 是**反模式**——运行时错误应该走 error code。

## E9. Compiler Flag 五件套 (ch11)

**CI 标配**：
```bash
cc -std=c17 -O2 \
   -Wall -Wextra -Wpedantic -Werror \
   -D_FORTIFY_SOURCE=2 \
   -g3 \
   -fpie -pie \
   source.c -o app
```

| Flag | 作用 |
|---|---|
| `-std=c17` | 显式标准 |
| `-O2` | 优化（`_FORTIFY_SOURCE` 必需） |
| `-Wall -Wextra -Wpedantic` | 全警告 |
| `-Werror` | 警告当错 |
| `-D_FORTIFY_SOURCE=2` | glibc 轻量级 buffer 检查 |
| `-g3` | 最丰富 debug 信息 |
| `-fpie -pie` | 启用 ASLR |

## E10. Sanitizer 四件套 (ch11)

| Sanitizer | 检测 | 性能开销 |
|---|---|---|
| **ASan** | UAF / buffer overflow / leak | ~2× |
| **UBSan** | 各种 UB | ~30~200% |
| **MSan** | 未初始化内存读 | ~3~5× |
| **TSan** | data race | ~5~15× |

**开发期 + CI 必开 ASan + UBSan**。

## E11. 静态分析 ≠ 完整 (ch11)

**Halting problem** → 任何"程序会出错吗"的问题不可完全静态回答。

- **Sound**：不漏报（无 false negative）
- **Complete**：不误报（无 false positive）
- **大多数工具 incomplete + sound**（保守报告）= 有 false positive

**多工具组合**：Clang Static Analyzer + cppcheck + Coverity 等。

## E12. Unit Test 选型 (ch11)

| Framework | 语言 | 备注 |
|---|---|---|
| **Google Test** | C++ | 推荐、生态最好 |
| Unity | C | 嵌入式友好 |
| CUnit | C | 老牌 |
| Check | C | Linux 友好 |
| CppUnit | C++ | JUnit 风格 |

---

# 跨章节高频反模式清单

> 这部分**价值极高**——记录**AI 写 C 代码最常见的错**。

| 反模式 | 章节 | 正确做法 |
|---|---|---|
| `gets(buf)` | ch7, ch8 | `fgets(buf, sizeof(buf), stdin)` |
| `printf(user_input)` | ch1 | `printf("%s", user_input)` |
| `p = realloc(p, n)` | ch6 | 临时指针 + free 旧 |
| `free(p); use(p)` | ch6 | `free(p); p = NULL;` |
| `Abs(INT_MIN)` | ch3 | 先 check `i == INT_MIN` |
| `if (a + b > UINT_MAX)` | ch3 | `if (a > UINT_MAX - b)` |
| `volatile` 当线程同步 | ch2 | `_Atomic` 或 mutex |
| `(unsigned char)getchar()` | ch7, ch8 | `int c; while ((c = getchar()) != EOF)` |
| `strncpy` 当"安全 strcpy" | ch7 | 手动补 null 或用 `_s` |
| `MAX(a++, b++)` | ch9 | 用 inline 函数 |
| `#define SQUARE(x) (x*x)` | ch9 | `((x)*(x))` 双层括号 |
| `for(p=h; p; p=p->next) free(p)` | ch5 | 先存 `q = p->next` |
| `if (cond) do_thing(); bar();` | ch5 | 永远加花括号 |
| `fputs(*p++, fp)` | ch8 | `fputc(*p++, fp)`（putc 是 macro） |
| `fseek(fp, 0, SEEK_CUR)` | ch8 | 大文件用 `fgetpos`/`fsetpos` |
| `tmpnam()` | ch8 | `mkstemp()` 或 socket |
| `assert(fopen(...) != NULL)` | ch11 | 走 error code |
| `char c = 'ÿ'; if (c == EOF)` | ch7 | 用 `int c = getchar()` |
| `i++ * i++` | ch4 | 拆完整表达式 |
| `a < b < c` | ch4 | `(a < b) && (b < c)` |
| `#define MAX (a,b) ...` | ch9 | 必须 `MAX(a,b)` 紧贴 |
| Header guard 用 `_FOO_H` | ch9 | 用 `FOO_H`（reserved） |
| `sprintf(buf, fmt, args)` | | ch) 7, ch8 | `snprintf(buf, sizeof(buf), fmt, ...)` |
| `void *` cast 一切 | ch6 | C 里让隐式转换 |
| 不用 `-Wall -Wextra` | ch11 | CI 标配 |

---

# 跨章节核心原则（5 条）

> 这是整本书"**最压缩**的提取"——如果你只能记住 5 件事：

### 1. **UB 是优化器燃料**

任何 UB（signed overflow、UAF、null deref、`i++*i++`）编译器可做任何事——包括按"UB 不发生"做激进优化删代码。**永远假定 UB 不可预测**。

### 2. **函数宏 ≠ 函数**

函数宏参数多次求值 + 不 full parens + name with space + comma in args 四类陷阱。**能用 inline 函数就不用宏**；用宏必 double parens。

### 3. **类型系统不替你查类型**

C 是弱类型 + 隐式转换。`c == ui` 为真、`(char)fgetc` 静音 warning、`void *` 隐式 cast —— 都是隐藏炸弹。**所有 narrowing cast 显式声明意图**。

### 4. **资源管理有边界**

`malloc` → `free` 一一对应；`realloc` 用临时指针；`free` 后置 NULL；`fopen` 检查 NULL；`fclose` 检查返回值。**漏一个 = leak / UAF / double-free**。

### 5. **质量是 process 不是 patch**

`-Wall -Wextra -Wpedantic -Werror` + `-D_FORTIFY_SOURCE=2` + ASan + UBSan + static_assert + unit test + static analysis + code review = 八层防御。**只靠调试 = 永远漏**。

---

# 何时回查哪一章（速查表）

| 我在写... | 翻到... |
|---|---|
| `int` / `unsigned` / `printf` 格式 | **ch3** |
| `malloc` / `free` / `realloc` | **ch6** |
| `*p` 解引用、`->`、指针运算 | **ch4** |
| 字符/字符串/UTF-8/宽字符 | **ch7** |
| `fopen` / `fread` / `fclose` / `tmpfile` | **ch8** |
| `#define` / `#include` / macro | **ch9** |
| 多文件项目 / `.a` / `.so` | **ch10** |
| `assert` / sanitizer 选型 | **ch11** |
| UB 哲学 / 标准版本 / SPIRIT | **ch1** |
| struct / typedef / opaque type | **ch2** |
| if/switch/while/for/goto | **ch5** |

---

# 总结一句话

> **Effective C 的核心 = 学会识别"AI/初学者最容易写错的 25 个反模式"，并用**编译器、Sanitizer、静态分析、单元测试**四层防御系统化拦截。**

下一步：把这 25 个反模式做成 `code-review-checklist.md` 或者 `ai-code-review-prompt.md`——**直接喂给 AI 做代码审查**。要不要做这个？
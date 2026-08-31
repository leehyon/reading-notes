# 第 9 章 · Preprocessor

> 来源: *Effective C* (Seacord, 2020) — Chapter 9, pp. 169–183
> 笔记日期: 2026-08-27

---

# 一、章节概述

1. **编译的 8 个 translation phases**：character mapping → line splicing（`\` + newline 拼接）→ tokenization → preprocessing → character-set mapping → string concatenation → translation → linkage。**预处理在第 4 步**——它没有类型系统、没有语义，只懂 tokens。
2. **预处理器的无知**：不知道函数、变量、类型——只能看到 identifier、literal、运算符。**所以 macro 不能做类型检查**。
3. **`# preprocessing directive`**：`#include` / `#define` / `#if` / `#ifdef` / `#ifndef` / `#else` / `#elif` / `#endif` / `#undef` / `#error` / `#pragma`。预处理指令以 newline 结尾。
4. **看预处理结果**：Clang/GCC `cc -E source.c -o out.i`；MSVC `cl /P /Fiout.i source.c`。`.i` 是 preprocessed source。**debug macro 必看 `-E` 输出**。
5. **`#include` 有传递性**：foo.c include bar.h，bar.h 又 include baz.h → translation unit 包含 baz.h + bar.h + foo.c。**循环 include 会导致无限递归**。
6. **`#include "foo.h"` vs `#include <foo.h>`**：差异是 implementation-defined。**惯例**：尖括号找系统路径（`-isystem`），引号找项目路径（`-iquote`）。**用户头文件**用 `"`，**系统/库头文件**用 `<>`。
7. **`#if` / `#elif` / `#else` / `#endif`** 是预处理条件包含——**没有花括号**！从 directive 到下一个 balanced `#elif`/`#else`/`#endif` 之间的所有 tokens 都包含/排除。**可以嵌套**。
8. **`defined` 运算符**：`defined(IDENT)` 是 1 if macro 定义，否则 0。**`#ifdef X`** 是 `#if defined X` 的简写。
9. **`#error "message"`**：编译期强制错误 + 自定义消息。**用途**：强制 caller 必须 `#define CONFIG_FOO` 后才能编译。
10. **`int compile_error[-1];` 诱导编译错误**：负长度数组 = compile error。比 `#error` 信息少。
11. **Header guard 模式**：```c
    #ifndef BAR_H
    #define BAR_H
    /* declarations */
    #endif
    ```
    防止同一 header 在同一 translation unit 里被多次 include。
12. **Header guard 命名规则**：用文件路径相关的大写标识符（`foo/bar/baz.h` → `FOO_BAR_BAZ_H`）。**永远不要用下划线开头的大写**（`_FOO_H` 是 reserved）——会与编译器/标准库的 macro 冲突。
13. **`#define` 两种 macro**：object-like（`#define PI 3.14`）和 function-like（`#define MAX(a,b) ((a)>(b)?(a):(b))`）。**函数宏必须有名字后紧跟 `(`，没有空格**——`#define MAX (a,b)` 不是函数宏。
14. **函数宏参数多次求值 = side effect 陷阱**：
    ```c
    #define bad_abs(x) (x >= 0 ? x : -x)
    bad_abs(i++)   // 展开: (i++ >= 0 ? i++ : -i++)   i 被递增两次
    ```
    **CERT PRE31-C**：永远不在 unsafe macro 里传带 side effect 的参数。
15. **Macro 参数必须 fully parenthesize**：`#define SQUARE(x) ((x) * (x))` ——外层 + 内层都得加，否则 `SQUARE(1+2)` = `((1+2) * 1+2)` = `6`，错。
16. **Comma in macro args 是分隔符**：`ATOMIC_VAR_INIT({1, 2})` 里的 comma 被当 argument分隔——传两个参数 `{1` 和 `2}`。**这就是 ATOMIC_VAR_INIT 在 C17 被 deprecated 的原因之一**。
17. **Stringify 运算符 `#`**：`#define STRINGIZE(x) #x` → `STRINGIZE(hello)` = `"hello"`。**只能用在 function-like macro 的 replacement list 里**。
18. **Token paste 运算符 `##`**：`#define PASTE(a,b) a ## _ ## b` → `PASTE(foo, bar)` = `foo_bar`。**生成新 identifier**。**`##` 的结果不再次被 macro 扫描**——避免无限递归。
19. **Macro rescan 规则**：展开后 rescanning，**包括正在展开的 macro 不会再次展开**（避免递归）。但**如果展开结果恰好是 `#include` 等预处理 directive，不作为指令处理**。
20. **Redefining macro**：必须先 `#undef`。**安全 idiom**：
    ```c
    #undef NAME
    #define NAME(x) ...
    ```
21. **宏的 scope** = 从 `#define` 到 `#undef` 或 translation unit 结束。**不依赖 block 结构**。
22. **macro 名字惯例**：全大写 + 前缀。`#define foo (1 + 1); void foo(int);` → preprocessor 展开为 `void (1+1)(int)` = 编译错。**用户 macro 永远全大写**避免冲突。
23. **`_Generic` (C11) 是类型泛型宏**：
    ```c
    #define sin(X) _Generic((X), \
        float: sinf, \
        double: sin, \
        long double: sinl)(X)
    ```
    **编译期根据类型选函数**。`<tgmath.h>` 用这个机制实现 `sin` 自动选 sinf/sin/sinl。
24. **`_Generic` 的 controlling expression 不求值**——只查类型。**没有匹配 + 没 default = 编译错**。
25. **`_Generic` 的 default 匹配一切未列类型**——**包括指针/struct，可能误匹配**。
26. **Predefined macros（编译器自动定义）**：
    - `__DATE__` / `__TIME__`：编译时间
    - `__FILE__` / `__LINE__`：当前文件/行号
    - `__func__`：当前函数名（C99）
    - `__STDC__` / `__STDC_VERSION__` / `__STDC_HOSTED__`：标准版本
    - `__STDC_ISO_10646__` / `__STDC_UTF_16__` / `__STDC_UTF_32__`：字符集
    - `__STDC_NO_ATOMICS__` / `__STDC_NO_COMPLEX__` / `__STDC_NO_THREADS__` / `__STDC_NO_VLA__`：可选特性开关
27. **`__STDC_VERSION__` 值**：`201710L` = C17 / C18，`201112L` = C11，`199901L` = C99。**查 GCC/Clang 默认**：`cc -dM -E - < /dev/null | grep __STDC_VERSION__`。

---

# 二、核心 Takeaways

### Takeaway 1: function-like macro 参数多次求值是隐藏的 UB 工厂

- **是什么**：`#define bad_abs(x) (x >= 0 ? x : -x)` 调用 `bad_abs(i++)` 展开为 `(i++ >= 0 ? i++ : -i++)`——i 递增 2 次。
- **为什么重要**：人类看源码觉得是一次 i++；编译器展开后是多次。**CERT PRE31-C**。
- **解决什么问题**：① 用 inline 函数代替；② macro 内只引用参数一次（用临时变量）；③ 永远不在 macro 参数里传 side effect。
- **适用场景**：所有 macro 调用；尤其 `MAX/MIN/SQUARE/abs` 类。

### Takeaway 2: macro 参数必须 fully parenthesize——两层

- **是什么**：`#define SQUARE(x) (x * x)` 看起来 OK；`SQUARE(1+2)` 展开 `(1+2 * 1+2)` = `5`（错，应为 9）。
- **为什么重要**：**operator precedence** 静默改变计算顺序。
- **解决什么问题**：永远写 `#define SQUARE(x) ((x) * (x))`——外层 `(...)` 防止外层组合错误，内层 `(x)` 防止参数内运算符冲突。
- **适用场景**：所有函数宏。

### Takeaway 3: macro 函数名后必须紧跟 `(`——空格会变成 object-like

- **是什么**：`#define MAX (a,b) ((a)>(b)?(a):(b))` —— 空格使 `MAX` 变成 object-like 宏，replacement list = `(a,b) ((a)>(b)?(a):(b))`。`int x = MAX(1,2);` 展开为 `(a,b) ((a)>(b)?(a):(b))(1,2);` = 编译错。
- **为什么重要**：常见 typo bug——`#define MAX (...)` vs `#define MAX(...)`。
- **解决什么问题**：所有函数宏必须 `NAME(` 紧贴。
- **适用场景**：code review 必查点；CI grep 检查。

### Takeaway 4: `_Generic` 是 C11 给的"伪重载"——比函数重载弱

- **是什么**：`_Generic(expr, type1: e1, type2: e2, ...)` 编译期选 e_i。**只查类型，不查参数个数**。
- **为什么重要**：`<tgmath.h>` 的 `sin(x)` 自动选 sinf/sin/sinl 就是这个机制。**C 没有 Java/C++ 那种函数重载**——`_Generic` 是变通。
- **解决什么问题**：类型相关的 generic 算法（cbrt/sin/log 等）。
- **适用场景**：math 库、通用 max/min、类型相关 dispatch。

### Takeaway 5: `#error` 比 `int x[-1];` 强——错误信息清晰

- **是什么**：```c
    #ifndef CONFIG_FOO
    #error "Must define CONFIG_FOO before including this header"
    #endif
    ```
- **为什么重要**：portable 代码常需要强制 caller 提供 config；`#error` 给出可读消息。
- **解决什么问题**：silent compilation failure → actionable error。
- **适用场景**：所有条件性 header（依赖 platform-specific macro）。

### Takeaway 6: `_FOO_H` 是 reserved header guard——会和编译器冲突

- **是什么**：以 `_` + 大写字母开头，或 `__` 开头的 identifier 是**reserved for implementation**——编译器/标准库可能定义。
- **为什么重要**：header guard 用 `_FOO_H` 可能与 `<features.h>` / `<sys/cdefs.h>` 冲突。
- **解决什么问题**：永远用 `FOO_H` 风格（大写但无下划线开头）；或 `PROJECT_FOO_H`（前缀）。
- **适用场景**：所有 header guard；code review 红线。

### Takeaway 7: macro 名字全大写是约定——不是编译要求

- **是什么**：`#define foo (1+1); void foo(int);` 编译错。**如果改 `void foo`，函数名是小写，编译器不报错**——因为 macro 总是展开。
- **为什么重要**：约定 = code review 检查。**不遵守 = silent 改名**。
- **解决什么问题**：所有用户 macro 必须全大写 + 至少一个下划线分隔（如 `MY_MAX`）。
- **适用场景**：所有 `#define`。

### Takeaway 8: `__STDC_NO_*` 宏是 C11 给的"特性探测"开关

- **是什么**：`__STDC_NO_ATOMICS__`、`__STDC_NO_THREADS__`、`__STDC_NO_VLA__` 等是 C 标准定义的"如果不支持这些特性就定义"宏。
- **为什么重要**：**freestanding / 嵌入式 / 老编译器**用这些宏知道哪些特性可用。
- **解决什么问题**：写 portable 代码——`#ifdef __STDC_NO_ATOMICS__` 决定是否用 atomics。
- **适用场景**：freestanding code、跨编译器/跨平台代码。

---

# 三、工程实践视角

### 嵌入式开发

- **Header guard + `__STDC_NO_*` 探测**：freestanding 环境用 `#ifndef __STDC_NO_THREADS__` 等决定能否用 threads.h。
- **不用 function-like macro 实现硬件寄存器访问**——`#define SET_BIT(reg, bit) ((reg) |= (1U << (bit)))` 因 side effect 危险。**用 inline 函数**或 static helper。
- **`#pragma once` vs include guard**：嵌入式 GCC 支持 `#pragma once`（**非标准**）——更简洁，但**不是所有编译器都支持**（IAR、Ke老编译器仍只信 header guard）。
- **MCU 寄存器映射**：用 macro 生成 register struct 定义——但**永远加 `_()` 函数化防多次求值**。
- **VLAs 在 MCU 禁用**：检查 `__STDC_NO_VLA__` 决定 fallback。

### Linux 系统开发

- **Linux kernel 编码规范第 11 章**："用宏来隐藏 magic number 是 OK，但宏实现函数 = 反模式"——**永远优先 inline 函数**。
- **`#error` 在内核极少用**——内核模块用 `#include <linux/module.h>` 触发依赖。
- **kernel `container_of` 宏**：用 `typeof` (GCC ext) 和 statement expression `({...})`——**Linux-specific**。可移植代码不要用。
- **`_Generic` 在 Linux 偶尔用**：`<sys/cdefs.h>` 提供 `__builtin_choose_expr` 兜底。
- **`-D_GNU_SOURCE` 是 Linux 用户态开发必加**——启用 GNU extensions。

### 机器人软件（ROS / ROS2）

- **ROS message 用 `_Generic` dispatch**：不同 message type 的统一处理函数（`pointcloud_to_laserscan` 等）。
- **ROS 跨平台宏**：`#ifdef __ANDROID__` / `__APPLE__` / `_WIN32` 分别处理 Android/iOS/Windows。
- **micro-ROS 嵌入式**：`#ifdef __STDC_NO_THREADS__` → 用 RTOS 原生 thread API 替代 `<threads.h>`。
- **ROS log 宏**：`ROS_INFO(...)` 是 function-like macro（不是 printf 安全）——**user data 不能做 format**。

### 汽车电子软件（AUTOSAR / ISO 26262）

- **AUTOSAR 编码规范**：
  - **禁止 function-like macro**（除编译器内置如 `_Alignas`）
  - 允许 object-like macro（常量）
  - 所有 macro 必须大写 + 项目前缀
- **MISRA C 规则 20.x 系列**：
  - 20.1：禁止 `#include` 绝对路径
  - 20.4：禁止 macro 重定义（必须先 `#undef`）
  - 20.5：禁止 `#undef` 已定义的 macro（要靠 `#include` 顺序保证）
  - 20.7：禁止 function-like macro 表达式参数有 side effect
  - 20.13：禁止 `#error` 出现
- **header guard 必须有**（AUTOSAR + MISRA 都强制）。
- **`__STDC_VERSION__` 检查**：MISRA 要求显式声明目标标准。

### 初级 vs 高级工程师的关注差

| 维度 | 初级 | 高级 |
|---|---|---|
| `MAX(a,b)` 宏 | 觉得 OK | 知道 side effect 危险，用 inline 函数 |
| `SQUARE(x)` | `(x*x)` | `((x)*(x))` 双层括号 |
| `#define MAX (a,b)` | 编译错不知 | 知道 macro 名后空格错误 |
| Header guard `_FOO_H` | 用 | 知道 reserved → `FOO_H` |
| `#error` 不用 | 不知道有这工具 | 用于"必须 CONFIG_X" 的硬性要求 |
| `__STDC_NO_VLA__` | 不知道 | 跨平台代码 `ifdef` 探测 |
| `_Generic` | 当作函数重载 | 知道是 type dispatch，写对 default |
| Macro 名字小写 | 编译能过就过 | 全大写 + 项目前缀 |

---

# 四、AI 时代视角

### 这个知识今天是否仍然重要？

**重要性级别：极高——macro 是 C 的"元编程"，AI 写错率最高**。

- **AI 经常写错**：
  - 函数宏 side effect 陷阱（`MAX(i++, j++)` 错）
  - Macro 参数不全加括号
  - Comma in macro args 被当分隔符
  - `_Generic` default 误匹配指针/struct
  - Header guard 用 `_FOO_H` reserved
  - Macro 名字小写导致意外展开

### AI 能帮助完成什么

- ✅ 把 `MAX` 函数宏改 inline 函数
- ✅ 给 macro 加 full parenthesize
- ✅ 生成正确的 header guard
- ✅ 解释 `#error` / `_Generic` / `defined` 用法
- ✅ 解释预处理输出 `cc -E`

### AI 无法替代什么

- ❌ **决定哪些代码应该 macro 化**——涉及 API 设计
- ❌ **处理 side effect 副作用**——需要理解业务流
- ❌ **跨编译器 macro 兼容性**——需要在目标编译器验证
- ❌ **`_Generic` 的 type association 设计**——需要类型学知识

### 工程师必须掌握的核心能力

1. **macro 全加括号**（两层）
2. **理解 macro 多次求值 side effect**
3. **`cc -E` 看预处理输出** —— debug macro 必备
4. **Header guard 不用 reserved 名字**
5. **理解 `_Generic` 的编译期类型选择**
6. **能用 `#error` 做硬性配置检查**

---

# 五、实践行动项

### 行动 1: 演示 function-like macro 参数多次求值陷阱
```bash
cat > /tmp/macro_sideeffect.c <<'EOF'
#include <stdio.h>
#define BAD_ABS(x) (x >= 0 ? x : -x)
int main(void) {
    int i = -5;
    // BAD: i 被递增两次
    int r = BAD_ABS(i++);
    printf("r = %d, i = %d (i should be -4, got %d - UB)\n", r, i, i);
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o /tmp/macro /tmp/macro_sideeffect.c && /tmp/macro
```
**预期**：i 实际被递增了 2 次。**对比 inline 函数**就只会递增一次。

### 行动 2: 演示 macro 必须 double-parenthesize
```bash
cat > /tmp/square.c <<'EOF'
#include <stdio.h>
// 错误版：没有外层括号
#define BAD_SQUARE(x) (x * x)
// 正确版：两层括号
#define GOOD_SQUARE(x) ((x) * (x))
int main(void) {
    printf("BAD_SQUARE(1+2) = %d (should be 9, got ...)\n", BAD_SQUARE(1+2));
    printf("GOOD_SQUARE(1+2) = %d\n", GOOD_SQUARE(1+2));
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o /tmp/square /tmp/square.c && /tmp/square
```
**预期**：BAD_SQUARE(1+2) = 5（错）；GOOD_SQUARE(1+2) = 9。

### 行动 3: 看预处理输出（debug macro 必会）
```bash
cat > /tmp/demo.c <<'EOF'
#include <stdio.h>
#define SQUARE(x) ((x) * (x))
#define MAX(a,b) ((a) > (b) ? (a) : (b))
int main(void) {
    int x = MAX(3, SQUARE(2));
    printf("x = %d\n", x);
    return 0;
}
EOF
cc -std=c17 -E /tmp/demo.c -o /tmp/demo.i
echo "=== expanded main ==="
grep -A 4 'int main' /tmp/demo.i | head -10
```
**预期**：所有 macro 展开——这是理解 macro 行为的必备工具。

### 行动 4: 演示 header guard 与 `#pragma once`
```bash
mkdir -p /tmp/header_test && cd /tmp/header_test
cat > foo.h <<'EOF'
#ifndef FOO_H
#define FOO_H
int get(void);
#endif
EOF
cat > bar.c <<'EOF'
#include "foo.h"
#include "foo.h"
int main(void) { return get(); }
EOF
cc -std=c17 -I. bar.c -o /tmp/header_test_bar 2>&1 | head -5
echo "Compiled successfully (header guard worked)"
```
**预期**：编译成功——第二次 include 被 guard 跳过，函数只声明一次。

### 行动 5: 演示 `_Generic` 类型分派
```bash
cat > /tmp/generic.c <<'EOF'
#include <stdio.h>
#define TYPE_NAME(X) _Generic((X), \
    int: "int", \
    double: "double", \
    char *: "char*", \
    default: "other" \
)
int main(void) {
    int i = 0;
    double d = 0.0;
    char *s = "hi;
    printf("i is %s\n", TYPE_NAME(i));
    printf("d is %s\n", TYPE_NAME(d));
    printf("s is %s\n", TYPE_NAME(s));
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o /tmp/generic /tmp/generic.c && /tmp/generic
```
**预期**：分别打印 `int`/`double`/`char*`——编译期类型选择。

---

# 六、值得深入思考的问题

### Q1: Macro 多参数多次求值是 C 的根本缺陷——为什么 50 年不引入 inline 函数作为替代？

C99 引入了 `inline`，C 编译器 GCC/Clang 长期支持 `static inline` 当 inline 函数用。**问题**：为什么 macro 还这么流行？**是不是因为 macro 提供 inline 函数给不了的能力**（stringify/token paste、`#pragma`、variadic）？

### Q2: `_Generic` 在 C11 引入——为什么 C89/C99 不引入？

**提示**：C89/C99 没有 type-of 表达式，`typeof` 是 GCC ext。**问题**：C 标准 30 年后才引入 type-based dispatch 是不是太晚？**为什么不一开始就用 `_Generic`**——是不是和 C++ function overloading 路线冲突？

### Q3: `_FOO_H` reserved 命名冲突——C 标准为什么不让 header guard 用 `_` 开头？

**提示**：保留 `_` 开头的 identifier 是为了未来扩展。**问题**：为什么 header guard 推荐 `FOO_H`（无下划线）而不是 `_FOO_H`（有下划线）？**这反映了标准对未来扩展空间的什么考量？**

### Q4: `ATOMIC_VAR_INIT({1,2})` 因 comma 被当分隔符导致 deprecated——C 标准为什么不修 macro 接受 brace-enclosed init？

**提示**：comma 是 macro argument separator，无法绕过。**问题**：为什么 C 标准不允许 macro 调用时用 `{...}` 整体当参数？**是不是 preprocessor 的根本限制**？**有没有 syntax extension 空间？**

### Q5: `__STDC_NO_*` 宏是 C11 加的——**为什么 C 标准不直接强制特性存在而是允许"不存在"**？

**观点 A**：freestanding 必须允许特性缺失 → `__STDC_NO_*` 必要。
**观点 B**：标准应统一所有实现都支持 → 不应有 NO 宏。
**问题**：C 标准的"可选特性"模型是灵活性还是分裂？**有没有可能 Rust/Go 风格的"必须有"标准化路径？**

---

*下一章预告*: **Chapter 10 — Program Structure** —— Principles of componentization（cohesion / coupling / code reuse / data abstraction / opaque types）、**header file 设计**（什么放 header / 什么放 .c）、linkage（external / internal / none）、构建流程（preprocess → compile → assemble → link）、make / CMake。这一章是把前面所有知识组装成"可维护项目"的工程桥梁。
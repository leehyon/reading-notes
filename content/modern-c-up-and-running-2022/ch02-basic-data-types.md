# Ch2. Basic Data Types

> 对应 PDF 物理页 50-82（印刷页 33-65）。本章是 C 数据类型的"内幕"：整数的有符号 vs 无符号、2's complement 的 INT_MIN 坑、IEEE 754 三类浮点（normalized/denormalized/special）、运算符优先级与位运算陷阱。与 Effective C ch3 高度重叠但视角更工程实操。

## §1 章节概述

1. **C 的内置类型 = 机器原生类型的镜像**——`char` 1B、`short` 2B、`int` 4B、`long` 8B、`float` 4B、`double` 8B、`long double` 16B（x86-64 Linux 典型值；`sizeof` 必须可查）。这让 C 既"可移植"又"必须按平台验证"。
2. **`sizeof(char) == 1` 是 C 标准唯一保证的大小**，其他类型"至少 `sizeof(char)`"但实际多大由 ABI 决定。**`int` 32-bit 几乎成事实标准，但不是 C 标准要求**。
3. **整数三组对照**：`signed/unsigned`、`short/int/long/long long`、`char/wchar_t`。规则：**`unsigned` 必须显式写**（不像 `signed` 可省略）；`char` 等价 `signed char` 或 `unsigned char` 由实现决定。
4. **2's complement 核心坑**：`INT_MIN == -INT_MIN`（因为 0x80000000 取负 = 0x80000000，溢出）。C 编译器**会**对 `-INT_MIN` 给 warning，但**不会**对一般 `n * n`（n 接近 INT_MAX）给任何 warning——**靠程序员防 overflow**。
5. **混合比较隐式转换**：`signed` 比 `unsigned` → signed 转 unsigned（`-1` 变 UINT_MAX）；`short` 比 `int` → short 提升到 int。结果：`int small = -1; unsigned big = 100; if (small > big)` 永远 true。
6. **IEEE 754 三类浮点**（32-bit `float` 为例）：
   - **normalized**（指数位非全 0 也非全 1）：隐式 leading 1，写入指数 = 实际指数 + 127 偏置
   - **denormalized**（指数位全 0）：无 leading 1，固定实际指数 -126，**用于表示 ±0 和极小值**
   - **special**（指数位全 1）：mantissa 全 0 = ±∞，否则 = NaN（`sqrt(-1)` 触发）
7. **浮点等号是 bit 相等**——`(1.0/3.0)*2.5 != 5.0/6.0`（最后 1 bit 不同）。要"近似等"用 `fabs(a-b) < FLT_EPSILON`（`<float.h>` 里有 `FLT_EPSILON` ≈ 1.19e-7）。
8. **算符优先级**：`*/%` 高于 `+-`，左结合。**永远加括号**——Kalin 明确说"括号比记优先级容易读"。
9. **经典 bug：`if (n = 1)`**——赋值 = 表达式有值。**惯用法：常量放左边** `if (1 == n)` 让编译器抓 typo。
10. **位运算只对整数合法**——`float` 直接 `<<` 编译错；`signed <<` 可能改变符号位；`>>` 对 signed 是逻辑移位（补 0）还是算术移位（补符号位）**由实现决定**——`unsigned` 才是可移植。`endian.h` 提供 `htonl/htons/ntohl/ntohs`。
11. **`-lm` 是数学库链标志**——`sqrt`/`pow` 等在 `libm.so`，默认**不自动链**。`-l` 后面跟 `lib` 前缀和扩展名都去掉的名字。

## §2 核心 Takeaways

### T1 — `sizeof` 是编译期运算符，类型信息是"如何分配/对齐/对齐到哪"的合约
- **是什么**：`sizeof(T)` 在编译时求出，不在运行时执行；返回 `size_t`（= `unsigned long` 在 64-bit Linux）。
- **为什么重要**：数组大小、`malloc` 长度、ABI 兼容判定都靠它。**`sizeof` 不会对空类型数组求值**——`int a[printf("hello")] = {0};` 里的 `printf` 不执行（C 标准允许的实现）。
- **解决什么**：可移植代码、动态内存分配、跨平台二进制协议。
- **适用场景**：所有 C 代码——尤其跨平台库、网络协议打包、共享内存 IPC。

### T2 — `unsigned` 必显式；隐式比较规则要熟
- **是什么**：`unsigned` 不能省（不像 `signed`）；`int n = -1; unsigned u = n;` → u 变 UINT_MAX（赋值时是"模 2^N 转换"）；`if (-1 > 1u)` 永远 true（比较时 signed 转 unsigned）。
- **为什么重要**：C 标准库的 `size_t` 是 `unsigned`，所有 `<string.h>`/`<stdlib.h>` 长度参数都是 `size_t`。`int i; for (i = 0; i < strlen(s); i++)` 中 `i` 是 signed，比较时**会有符号/无符号警告**。`while (n-- > 0)` 安全，`while (n--)` 当 n 是 unsigned 时陷阱。
- **解决什么**：循环计数器、缓冲区长度、API 边界。
- **适用场景**：所有 C 代码；**尤其嵌入式**：寄存器位宽天然 unsigned。

### T3 — `INT_MIN == -INT_MIN` 是 2's complement 的结构性 bug
- **是什么**：取负算法 = "取反 + 1"，`1000...0`（INT_MIN）取反 = `0111...1`，+1 = `1000...0` 自身。所以 `-INT_MIN` = `INT_MIN`，需要"放大一位"才能装，结果溢出。
- **为什么重要**：Kalin ch2.2.1 把它叫"surprising but well-publicized peculiarity"。C 编译器**会**对 `-INT_MIN` 文本 warning（C11 起 `-Woverflow`），但**不会**对 `int n = INT_MIN; n = -n;` warning。**`abs(INT_MIN)` 仍是 UB**（C11 §7.22.6.1）——POSIX 才把 `abs`/`labs` 定义为安全。
- **解决什么**：写 `labs`/`llabs` 处理长整型绝对值；用 `<inttypes.h>` 的 `imaxabs` 处理最大整数。
- **适用场景**：图像处理（负坐标转正）、金融计算（绝对值）、位图位运算。

### T4 — IEEE 754 的"三个零"与 NaN/Inf
- **是什么**：32-bit float = 1 sign + 8 exponent + 23 mantissa。三类：
  - 指数 0..254：normalized（隐式 leading 1）
  - 指数 0：denormalized（±0 + 极小值）
  - 指数 255：special（mantissa 0 = ±∞，否则 = NaN）
- **为什么重要**：`0.0F == -0.0F` 为 true（C 保证），但 `1.0F / 0.0F` 和 `1.0F / -0.0F` 不同（`+inf` vs `-inf`）。**NaN 永不等于自己**（`NaN == NaN` 为 false）；**NaN 传染**：`NaN + 1 = NaN`。
- **解决什么**：除零保护、信号处理、数值稳定性、机器学习推理（`log(0) = -inf`）。
- **适用场景**：科学计算、SIMD 数据通路、CUDA kernel、ROS2 costmap 距离场（inf 表示"未知"）。

### T5 — 浮点等号要用 epsilon，不能直接 `==`
- **是什么**：`0.1 + 0.2 != 0.3`（0.30000000000000004）；`5.0/6.0 != (1.0/3.0)*2.5`（最后 1 bit 不同）。正确做法：`fabs(a - b) < FLT_EPSILON`（float）或 `< DBL_EPSILON`（double）。
- **为什么重要**：所有浮点判等都需要 epsilon；`==` 用在 double 上几乎一定是 bug。
- **解决什么**：测试断言、状态机转移条件、控制回路阈值判定。
- **适用场景**：PID 控制器（`fabs(error) < threshold`）、数值算法收敛判定、单元测试（`ASSERT_DOUBLE_EQ` in gtest 是带 4 ULP 容差）。

### T6 — 算符优先级 = bug 温床
- **是什么**：`8 + 2 * 3 = 14` 而非 30；`n1 + n2 * n3` 与 `(n1+n2)*n3` 完全不同；`&` 比 `==` 优先级低（`if (x & 1 == 0)` 永远 false，**经典 MISRA 违规**）。
- **为什么重要**：Linux kernel 用 `grep '[^&]&[^&]' | grep '='` 抓 `&` 与 `==` 混用。MISRA-C 2012 Rule 12.1 强制优先级显式加括号。
- **解决什么**：可读性 + 跨工程师协作 + 静态分析器（clang-tidy 有 `bugprone-misplaced-operator`）。
- **适用场景**：所有 C 代码；尤其嵌入式（位运算 + 比较混用高频）。

### T7 — `if (n = 1)` 是赋值不是比较；常量放左边是 1970s 的 Yoda 条件
- **是什么**：`=` 是赋值、表达式值 = 右值；`==` 是比较。`if (n = 1)` 编译能过（n 变 1，表达式值 1 = true），是 bug 温床。Yoda 条件：`if (1 == n)` 让 `if (1 = n)` 编译错。
- **为什么重要**：现代编译器（gcc `-Wall`、clang `-Wparentheses`）会 warn `suggest parentheses around assignment used as truth value`，但**默认不开** `[-Werror]`。Linux kernel 开 `-Wno-parentheses`（注释风格兼容）；嵌入式 MISRA 强制 Yoda。
- **解决什么**：消除一类 whole class of bugs。
- **适用场景**：所有 C 代码；尤其老 codebase 维护。

### T8 — `>>` 对 signed 是 logical 还是 arithmetic 由实现决定
- **是什么**：C11 §6.5.7 仅规定 `E1 >> E2` 对 `E1` 是 signed 时"通常算术移位"，但**实现自由**。`int n = -1; n >> 1` 在 x86 上 = `-1`（算术移位，gcc），在某些 DSP 上可能 = `0x7FFFFFFF`（逻辑移位，编译器有 `#pragma` 控制）。
- **为什么重要**：可移植代码必须**只用 `unsigned` 做位运算**；编译器（gcc）实际行为"通常算术移位"，但标准不保证。
- **解决什么**：跨平台 bit-manipulation、网络字节序、图像像素处理、加密算法。
- **适用场景**：任何位运算、SIMD intrinsic、固件寄存器读写。

## §3 工程实践视角

### 3.1 Linux 系统开发视角

- **`<stdint.h>` 才是真"可移植 C"**——`int` 不可移植，但 `int32_t` / `uint64_t` 保证 32/64 bit。Linux kernel 内部代码几乎全用 `u32/u64/u8`（`asm/types.h`），不用裸 `int`。**Kalin ch2 只讲了内置类型，但工程上你必须用 `<stdint.h>`**。
- **`-fsanitize=integer / -fsanitize=undefined`**：UBSan（Undefined Behavior Sanitizer）能抓**有符号溢出**（`int n = INT_MAX; n++`）、左移负数、`-INT_MIN`。**CI 里**给 lib 用，给 bin 用也行（性能损耗 1-2x）。
- **`-ftrapv`**：GCC 把有符号溢出变成 `SIGABRT`（不推荐生产用，但 debug 极好）。
- **`-Wsign-compare -Wsign-conversion`**：抓**所有**有符号/无符号混合——CI 必须开。Linux kernel 默认关（太多 false positive）；嵌入式/RTOS 项目**必开**。
- **`-Wfloat-equal`**：抓所有浮点 `==` 误用。
- **`<endian.h>` 是网络协议标准库**——`htonl/htons/ntohl/ntohs` 在 `<arpa/inet.h>`。网络协议一定要用，**不能自己 `<< 24` 拼**，因为 `endian.h` 在大端机器上就是 no-op，在小端机器上才做 swap。
- **`-lm` 不自动链是历史包袱**——glibc 把 libm 单独成库是为了让非数学程序不付 libm 体积。现代 musl 链 glibc 时会把 libm 静态并入，但 `-lm` 仍是便携写法。
- **整数溢出的安全加法**（防 UB 触发优化器发疯）：
  ```c
  /* 来自 SafeInt / Chromium 风格 */
  if (a > 0 && b > INT_MAX - a) goto overflow;
  int c = a + b;   /* 不会 UB */
  ```
  或用 gcc 7+ 的 `__builtin_add_overflow(a, b, &result)`。

### 3.2 机器人软件视角（ROS2 / 嵌入式控制）

- **ROS2 消息字段全部 `<stdint.h>` 风格**——`int32`, `uint8`, `float32`, `float64`（不是 `double`，是显式宽度浮点）。**`builtin_interfaces/Time` 用 `int32 sec + uint32 nanosec` 分开存**——**因为 ROS2 要兼容 32/64-bit 平台**。
- **`sensor_msgs/PointCloud2` 的 `data` 是 `uint8[]`**——用 `float32` 还是 `float64` 取决于 sensor。Velodyne 激光雷达 = `float32`（省带宽），科研高精度 LiDAR = `float64`。**带宽 = 1.5MB / 帧 × 10Hz = 15MB/s**，从 `float64` 降到 `float32` 直接砍一半。
- **机器人控制循环的浮点选择**：
  - **动力学 / 运动学正解**：用 `double`（关节多、累积误差大）
  - **PID 控制器**：`float` 够用（x86+NEON/ARM+Cortex-M4F 都有 FPU）
  - **图像处理**：`float` 强制（CUDA tensor core 也偏 float）
  - **底层寄存器 / 通信协议**：`int32_t` / `uint16_t` 显式宽度
- **NaN 在 ROS2 costmap 里是"未知栅格"**——`nav2_costmap_2d` 用 `std::numeric_limits<double>::infinity()` 标记未观测区域，`-1.0` 标记障碍，`0.0` 标记空闲。**`inf > 任何值` 永远 true**，但 `inf == inf` 永远 false——机器人决策用 `> threshold` 而非 `== inf`。
- **机械臂奇异点附近会出现 NaN 传播**——IK 求解器返回 NaN 关节角 → 驱动器收到 NaN → 电机扭矩 NaN → 安全保护触发。**驱动器软件必须有 NaN 检测并进入安全模式**。
- **时间戳精度**——ROS2 默认 `nanosec`，单数（不是 `double sec`），因为 `double` 只能精确表示 15 位十进制，1 秒 + 1 纳秒 = 1.000000001 秒，单数精度更高。**`builtin_interfaces/Time` 的两个字段都是单数**。
- **位运算在机器人协议里无处不在**——CAN bus 帧解析、Modbus 寄存器读写、EtherCAT 状态机、ROS2 message header 的 `seq` 字段。**这些都必须用 `uint32_t`/`uint16_t` 显式宽度 + `htonl/ntohl`**。

### 3.3 初级 vs 高级工程师对照

| 习惯 | 初级 | 高级 |
|---|---|---|
| 整数类型 | `int` / `long` 满天飞 | `<stdint.h>` 的 `int32_t` / `uint16_t` |
| 浮点等号 | `if (a == b)` | `if (fabsf(a-b) < 1e-6f)` |
| 位运算 | signed int 上 `>>` 和 `<<` | 只用 `uint32_t`，位操作前想 endian |
| 优先级 | 死记硬背 | 永远加括号 + clang-tidy `bugprone-misplaced-operator` |
| `if (n = 1)` 防御 | 没意识到 | 开 `-Wparentheses` / Yoda 条件 / 拆分 if 赋值 |
| 浮点 `==` 防御 | 偶尔想起 | 开 `-Wfloat-equal` + 永远写 epsilon 比较 |
| 溢出检查 | 假设编译器帮查 | `__builtin_add_overflow` 或 SafeInt 库 |
| 编译命令 | `gcc foo.c` | `gcc -std=c11 -O2 -Wall -Wextra -Wpedantic -Werror -Wsign-compare -Wfloat-equal -fsanitize=undefined foo.c` |
| 调试浮点 | 盯着输出 | `printf("%.20f\n", x)` 看 20 位精度 |

## §4 AI 时代视角

### 4.1 这些知识还重要吗？（2026 年视角）

**极重要。** 这是 C 的"数据契约"层。LLM 生成的 C 代码 80% 的 bug 出在这一层（整数溢出、浮点等号、混合符号比较）。AI 训练数据里 C 代码密度远低于 Python/JS，**LLM 在 C 数据类型上的错误率是 Python 同类错误的 2-3 倍**。

现代工程师的 ch2 日常：
- **生成代码** = 100% 靠 LLM
- **修代码** = 80% 时间在追这一章的 bug
- **性能调优** = 看懂 SIMD intrinsic 的浮点 bit layout（vllm、CUTLASS）

### 4.2 AI 现在能做的

- ✅ 解释 IEEE 754 三类、normalized/denormalized 区别
- ✅ 写 2's complement 转换代码
- ✅ 生成 `<stdint.h>` 风格的可移植代码
- ✅ 写浮点 epsilon 比较模板
- ✅ 用 `-Wall -Wextra -Werror` 配置 Makefile
- ✅ 解释 5 个 NaN/Inf 的特殊行为

### 4.3 AI 经常写错的地方（必看）

| 错误模式 | 例子 | 后果 |
|---|---|---|
| **1. 混合符号比较** | `for (int i = 0; i < strlen(s); i++)` | `int vs size_t` 警告；负值循环直接退出；用 `ptrdiff_t` 或 `size_t i` |
| **2. 浮点 `==`** | `if (sum == 0.0)` | 累加误差让 sum = 1e-16 → 永远 != 0；用 `fabs(sum) < DBL_EPSILON` |
| **3. 整数溢出** | `int n = strlen(s) + 1; malloc(n);` | 巨型字符串时溢出；用 `calloc(slen+1, 1)` |
| **4. `unsigned` wraparound 当循环** | `for (unsigned i = 10; i >= 0; i--)` | i 变 0 后 i-- 变 UINT_MAX → 死循环 |
| **5. `abs(INT_MIN)`** | `int x = abs(INT_MIN);` | UB（C 标准未定义；POSIX 安全）；用 `llabs` + 显式检查 |
| **6. signed left shift 改符号** | `int n = 1 << 31;` | UB（C 标准 signed 左移结果未定义）；用 `int32_t n = 1U << 31;` 拿 INT_MIN |
| **7. `~0` 当 max** | `unsigned mask = ~0;` | 实际是 0xFFFFFFFF；可移植版本 `UINT_MAX` from `<limits.h>` |
| **8. `&` vs `==` 优先级** | `if (x & 1 == 0)` | 解析为 `x & (1 == 0)` = `x & 0` = 0，**永远 false**；必须 `if ((x & 1) == 0)` |
| **9. NaN 比较** | `if (x == NAN)` | NaN != NaN，永远 false；用 `isnan(x)` from `<math.h>` |
| **10. `printf` 格式不匹配** | `printf("%d", sizeof(int));` | `%d` 要 `int`，`sizeof` 是 `size_t`；用 `printf("%zu", sizeof(int))` |
| **11. 隐式 `int` 返回** | `f() { return 1.5; }` | 老 C 风格；C99 起必须显式返回类型 |
| **12. `-lm` 漏链** | `gcc foo.c -o foo; ./foo` (用了 sqrt) | 链接错 `undefined reference to sqrt`；必须 `gcc foo.c -o foo -lm` |
| **13. `htonl` 顺序** | `uint32_t n = htonl(*(uint32_t*)buf);` | 违反 strict aliasing → UB；必须 `memcpy(&n, buf, 4); n = htonl(n);` |
| **14. `1.0/0.0` 在整数上** | `int n = 1/0;` | 编译期常量除零 = 编译错（除 `0` 不能编译过） |

### 4.4 工程师必须保留的核心能力

- **能 5 秒内口算 `INT_MAX + 1` = `INT_MIN`**（wraparound）——嵌入式调试 / overflow bug 定位。
- **能 30 秒内写 `fabsf(a-b) < FLT_EPSILON` 模板**——所有浮点判等。
- **能区分 normalized / denormalized / NaN 的 bit pattern**——看 gdb `print/x f` 时立刻知道是哪类。
- **能用 `-fsanitize=undefined` 配置 CI**——这一项就能挡掉 50% 整数 bug。
- **能跟 AI 说"请用 `<stdint.h>` 风格 + 开 -Wsign-compare"**——ch2 词汇是 prompt 的关键。

### 4.5 wasm 工具链延伸（用户已选需要）

- **IEEE 754 在 wasm 里是强制的**——wasm 规范 §4.2.4 要求所有 FPU 操作符合 IEEE 754。**这意味着 ch2.3 的所有规则在 wasm 里 100% 适用**——编译器不能优化掉。
- **wasm 没用 `<stdint.h>` 问题**：wasm32 的 `int` 是 32-bit（OK），但 `long` 是 32-bit（不是 64-bit！），`long long` 是 64-bit。**这跟 host Linux x86-64 不一样**——LLM 经常写"long = 8B"假设，在 wasm 里会截断。
- **AI 工具链中 wasm 的 C 数据契约**：QuickJS + WASI 跑用户 C 代码时，**整型宽度契约必须严格**——`int32_t` 显式宽度才是安全；`int`/`long` 是定时炸弹。

## §5 实践行动项

### A1 — 验证 `sizeof` 实际值，看 `<stdint.h>` 的价值
```bash
mkdir -p /tmp/modern-c/ch02 && cd /tmp/modern-c/ch02
cat > sizes.c <<'EOF'
#include <stdio.h>
#include <stdint.h>
#include <float.h>
int main(void) {
    printf("char=%zu short=%zu int=%zu long=%zu long long=%zu\n",
           sizeof(char), sizeof(short), sizeof(int), sizeof(long), sizeof(long long));
    printf("int8=%zu int16=%zu int32=%zu int64=%zu\n",
           sizeof(int8_t), sizeof(int16_t), sizeof(int32_t), sizeof(int64_t));
    printf("float=%zu double=%zu long double=%zu\n",
           sizeof(float), sizeof(double), sizeof(long long double));
    printf("FLT_EPSILON=%.10e DBL_EPSILON=%.10e\n", FLT_EPSILON, DBL_EPSILON);
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -pedantic -o sizes sizes.c && ./sizes
```
**验收**：记住你的平台 `long` 是 8B（x86-64 Linux）还是 4B（x86-64 Windows / wasm32）；`FLT_EPSILON ≈ 1.19e-7`、`DBL_EPSILON ≈ 2.22e-16`。

### A2 — 复现 2's complement 的 `INT_MIN == -INT_MIN` 陷阱
```bash
cat > intmin.c <<'EOF'
#include <stdio.h>
#include <limits.h>
int main(void) {
    int a = INT_MIN;
    int b = -a;          /* 触发 -Woverflow warning */
    printf("INT_MIN  = %d\n", a);
    printf("-INT_MIN = %d\n", b);
    printf("a == b ? %s\n", a == b ? "yes (溢出陷阱)" : "no");
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -Woverflow -Werror -o intmin intmin.c 2>&1
./intmin
```
**验收**：看到 `-Woverflow` warning；`-INT_MIN = -2147483648`（不是 `+2147483648`）；用 `int64_t` + `llabs` 改写后消除 UB。

### A3 — 复现混合符号比较的"反直觉 true"
```bash
cat > mixcmp.c <<'EOF'
#include <stdio.h>
int main(void) {
    int small = -1;
    unsigned big = 100U;
    if (small > big) printf("yep, small > big (反直觉)\n");
    /* 安全改写：把 int 转 unsigned 后比，或把 unsigned 转 long long 后比 */
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -Wsign-compare -o mixcmp mixcmp.c && ./mixcmp
```
**验收**：输出 `yep, small > big`；加 `-Werror` 后编译失败；改 `if ((unsigned)small > big)` 后逻辑正确但仍要 cast。

### A4 — 验证浮点 epsilon 比较 vs `==`
```bash
cat > fpeq.c <<'EOF'
#include <stdio.h>
#include <math.h>
#include <float.h>
int main(void) {
    double a = (1.0/3.0) * 2.5;
    double b = 5.0 / 6.0;
    printf("a = %.20f\n", a);
    printf("b = %.20f\n", b);
    if (a == b) printf("a == b (几乎不可能)\n");
    else        printf("a != b (bit 不同)\n");
    if (fabs(a-b) < DBL_EPSILON) printf("|a-b| < DBL_EPSILON → 近似等\n");
    /* NaN 测试 */
    double nan_val = sqrt(-1.0);
    if (nan_val == nan_val) printf("NaN == NaN (永远不会)\n");
    else                    printf("NaN != NaN (标准行为)\n");
    if (isnan(nan_val))     printf("isnan(NaN) = true (正确方法)\n");
    return 0;
}
EOF
gcc -std=c11 -Wall -Wextra -o fpeq fpeq.c -lm && ./fpeq
```
**验收**：看到 `(1.0/3.0)*2.5` 和 `5.0/6.0` 最后 1 bit 不同；`NaN == NaN` 为 false；`isnan` 为 true。

### A5 — 用 UBSan 抓有符号溢出
```bash
cat > ub.c <<'EOF'
#include <stdio.h>
#include <limits.h>
int main(void) {
    int n = INT_MAX;
    n = n + 1;            /* UB: 有符号溢出 */
    printf("n = %d\n", n);
    return 0;
}
EOF
gcc -std=c11 -fsanitize=undefined -g -O0 -o ub ub.c
./ub
# 期望: runtime error: signed integer overflow: 2147483647 + 1 cannot be represented in type 'int'
```
**验收**：UBSan 报"signed integer overflow"；改用 `__builtin_add_overflow` 后不再报。

## §6 值得深入思考的问题

1. **`sizeof(int) == 4` 是 C 标准吗？** C 标准只说"int 至少 16 bit"，实际 32/64 bit 都是"巧合"。如果你要为 RISC-V 32-bit 写代码，**`int` 还是 32 bit**；`int64_t` 来自 `<stdint.h>`，**必须包含这个头**才保证可移植。为什么 `<stdint.h>` 是 C99 才进入标准？C89 时代怎么写可移植整型？
2. **`abs(INT_MIN)` 为什么是 UB 而 `abs(-5)` 安全？** C 标准为什么没规定"abs 应当处理边界"？这跟谁负责"溢出语义"有关——编译器还是程序员？Linux kernel 怎么处理 `abs(INT_MIN)`？（提示：看你 git log `kstrtol`/`kstrtoll`）
3. **NaN 应该怎么处理？** ROS2 costmap、数据库 NULL、JSON `null`、Python `float('nan')`——四者表示"无值"的语义有何不同？机器人控制循环里出现 NaN 是该清零、抛错、还是进入安全模式？
4. **`fabs(a-b) < epsilon` 是好的浮点比较吗？** 对极小值 `a ≈ b ≈ 1e-30`，`fabs(a-b) < 1e-7` 永远 true（数值淹没在 epsilon 里）。正确写法是相对 epsilon：`fabs(a-b) < max(EPS, EPS * max(fabs(a), fabs(b)))`。**为什么 Effective C 强调"绝对 epsilon"是反模式？**
5. **为什么 `1.0/0.0 == inf` 而 `1/0` 编译错？** C 标准怎么区分"整数除零"和"浮点除零"？这是 IEEE 754 的要求还是 C 自己的要求？**嵌入式无 FPU 时 `1.0f/0.0f` 行为又是什么？**（提示：Cortex-M0 没有 FPU）
6. **`<stdint.h>` 的 `int_fast32_t` vs `int32_t` vs `int_least32_t` 区别？** 三个"32 位整型"哪个最快？哪个最小？AI 写代码用哪个？为什么 ROS2 几乎全用 `int32_t` 而非 `int_fast32_t`？

---

*下一章预告*：**ch3 Aggregates and Pointers**——Kalin 的"指针大章"（68 页！全书最长）。覆盖数组 vs 指针算术、多维数组、`void*` 与 NULL（高阶回调函数如 `qsort`）、结构体（含 `qsort` 排序指针数组）、联合体、字符串转换（`strtol` + `char** endptr`）、堆存储（`malloc`/`free` 配对）、嵌套堆结构（树/链表）、**内存泄漏与堆碎片**及诊断工具（`valgrind` / `ASan`）。这是与 ch6 (Networking) / ch7 (Concurrency) 直接相关的"内存管理基础章"。

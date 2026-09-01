# 第 3 章 · Arithmetic Types

> 来源: *Effective C* (Seacord, 2020) — Chapter 3, pp. 35–55
> 笔记日期: 2026-08-27

---

# 一、章节概述

1. **整数类型都有有限范围**：representation 是位编码方式，value 是数学意义的数值。**width** 是含符号位总位数，**precision** 是不含符号位和 padding 的可用位数。
2. **`<limits.h>` 提供可移植的边界常量**：写 `UINT_MAX`、`INT_MIN` 而非字面量 `2147483647` —— 跨平台时自动适配。
3. **unsigned wraparound 是 well-defined**：算术结果对 `(MAX+1)` 取模；可预测，但**仍是常见 bug 来源**（循环计数、宽度溢出检测错误）。
4. **B-787 真实事故**：发电机控制单元的 248 天 wraparound 导致所有六台发电机进入 fail-safe。**`unsigned int i = n; i >= 0; --i` 永真无限循环**。
6. **signed 表示将统一为 two's complement**：C2x 起**只支持 two's complement**（C23 已正式发布）。这是标准第一次在表示上"收敛"。
7. **most negative value 的绝对值不可表示**：8-bit signed char 的 `-128` → `+128` 是 UB（溢出）。`Abs(INT_MIN)` 是经典例子——`glibc abs` 用 cast-to-unsigned 巧妙规避，但仍是 implementation-defined。
8. **signed integer overflow = UB**：编译器可 silently wrap / trap / 优化掉。**"先 wrap 再说"是反模式**——因为 UB。
9. **整数常数的三种进制**：decimal（71）、octal（0777 = 511，注意前导 0）、hex（0xDEADBEEF）。**octal 陷阱**：误写 `int perm = 0777` 看着像十进制但实际是 511；电话号前导 0 经常踩坑。
10. **浮点 = `(sign, exponent, significand)` 三段**：float (1+8+23) 32 位 / double (1+11+52) 64 位。**bias 处理**：float 指数 bias 127、double 指数 bias 1023。
11. **浮点不是数学实数**：**不结合**、**不分配**、不能表示 `0.1`、**不能作循环计数器**（CERT FLP30-C）。
12. **subnormal / NaN / ±0 / ±∞**：浮点比你想得"复杂得多"——subnormal 处理 0 附近的下溢、NaN 传播、IEEE 754 静默异常。`fpclassify()` 是分类入口。
13. **三种算术转换**：integer conversion rank（类型等级）、integer promotion（小类型升 int/unsigned int）、usual arithmetic conversions（双目运算找公共类型）。
14. **integer promotion 的设计动机**：避免 `signed char * signed char` 溢出；让 CPU 用自然字长算。**value-preserving** 规则（C89 起）：优先保持符号。
15. **`c == ui` 的真实陷阱**：`signed char c = -1` 与 `unsigned int ui = UINT_MAX` 比较相等——`c` 先 sign-extension 到 -1 (int)，再转 unsigned int = UINT_MAX，相等。这**经常出 bug**。
16. **safe conversion 必须做范围检查**：窄化转换 (`long → char`) 必然丢失值；要做 `if (value < SCHAR_MIN || value > SCHAR_MAX) return ERANGE;`。**Int-to-FP 越界也是 UB**。

---

# 二、核心 Takeaways

### Takeaway 1: unsigned wraparound 是 well-defined，但仍是 bug 的高发地

- **是什么**：unsigned 算术运算超出范围 = 模 `MAX+1`。
- **为什么重要**：因为是 well-defined，**初学者以为"和真值算一样"**。但循环条件 `for (unsigned i = n; i >= 0; --i)` 永真、`sum + ui > UINT_MAX` 永远 false（wrap 之后）。
- **解决什么问题**：教你写正确的**宽度溢出检查**——把溢出比较转换成"减法 + 比较"，消除 wrap 在测试里出现的可能。
- **适用场景**：任何 unsigned 计数 / 哈希 / 时间戳 / 协议字段。

### Takeaway 2: signed overflow 是 UB，编译器可以"做任何事"

- **是什么**：signed 算术结果超出范围 = UB。`INT_MIN * -1` 是经典案例。
- **为什么重要**：现代 GCC/Clang 在 `-O2` 下会**基于 UB 不发生**做激进优化（常量传播）。`-fwrapv` / `-fno-strict-overflow` 是给非优化场景的兜底，**但不能依赖**。
- **解决什么问题**：避免"看似工作、移植或升级编译器就崩"的代码。
- **适用场景**：所有 signed 算术；尤其是金融、传感器数据、控制算法。

### Takeaway 3: integer promotion 不是 bug，是 C 的有意设计

- **是什么**：`char` / `short` 在运算前**先升到 int / unsigned int**。
- **为什么重要**：避免中间结果溢出（`signed char c = 100 * 3` 在 promote 后算 300 在 int 范围内）。**同时是 UB 检测的混淆源**——你以为在算 char，编译器在算 int。
- **解决什么问题**：解释 "为什么 `c1 * c2 / c3` 不会溢出"。
- **适用场景**：写位运算 / 协议解析 / 哈希时要知道"你的 char 早被升级了"。

### Takeaway 4: `c == ui` 这种比较是隐藏的语义炸弹

- **是什么**：`signed char c = -1; unsigned int ui = UINT_MAX; c == ui` 为真。
- **为什么重要**：人类读 `-1 vs 4,294,967,295` 觉得明显不等；C 算下来却相等。**signed 升 unsigned = sign-extend 后转无符号**。
- **解决什么问题**：避免在 mixed-signed 比较里"看到代码以为懂了，实际编译器干了别的事"。
- **适用场景**：所有 mixed-sign 比较——通常是协议字段、长度计算、checksum 验证。

### Takeaway 5: 浮点不是数学实数，循环计数器是重灾区

- **是什么**：`for (float i = 0; i < 1.0; i += 0.1) ...` —— `0.1` 二进制无法精确表示，**i 可能永远到不了 1.0**。
- **为什么重要**：CERT FLP30-C 明确禁止。能避就避；非要浮点循环，用 `int` 计数 + 内部乘以 `0.1`。
- **解决什么问题**：避免"看起来能跑、跑出莫名死循环或差一"的 bug。
- **适用场景**：所有浮点循环；DSP / 图形 / 物理仿真尤其常见。

### Takeaway 6: NaN 是不"等于任何东西"的特殊值

- **是什么**：`NaN == NaN` 是 false；`NaN != NaN` 是 true。任何涉及 NaN 的比较都用 `isnan()`。
- **为什么重要**：sort / minmax / dedup 在 NaN 存在时几乎必然出错。**关键**：信号处理、控制算法里"传感器瞬时返回 NaN"会导致整个回路 fail。
- **解决什么问题**：避免"NaN 静默通过导致下游数据污染"。
- **适用场景**：传感器融合、机器学习、绘图、物理仿真。

### Takeaway 7: int → FP 越界是 UB；FP → int 越界也是 UB

- **是什么**：把 `long long`（64-bit）的值转 `float`（24-bit significand）= 越界 = UB；把 `1e30` 转 `int` = 越界 = UB。
- **为什么重要**：这是最容易被忽略的 UB 之一——看似"自然转换"。
- **解决什么问题**：避免"大整数塞给小浮点"或"大浮点塞给小整数"的 silent corruption。
- **适用场景**：数值库、单位转换、网络协议字段解码（big-endian uint64 → double）。

---

# 三、工程实践视角

### 嵌入式开发

- **固定宽度类型是嵌入式铁律**：用 `<stdint.h>` 的 `uint32_t` / `int32_t` 代替 `int` / `unsigned`。**寄存器映射、网络协议字段、硬件计数**一律明确宽度。
- **`UINT32_MAX` vs `0xFFFFFFFF`**：永远用宏；避免 `1UL << 32` 在 32 位 MCU 上是 UB（移位溢出）。
- **wraparound 陷阱在 MCU 更阴险**：硬件计数器（timer tick、CAN ID）通常 32-bit wraparound。你的"超时检测"如果写成 `if (now - start > TIMEOUT)` 在 wrap 时**反向**，必须用 `(int32_t)(now - start) > TIMEOUT` 这种显式 signed 差值（Linux kernel `time_after` 也是这个思路）。
- **禁用浮点**在低端 MCU 上：M0 没有 FPU，`float` 用 soft-float 是 100x 性能损失。**汽车 ECU、医疗设备**常用 fixed-point。
- **NaN 处理**：传感器返回 NaN = 通常是 I2C 失败 / ADC 异常。**必须显式检查 `isnan()`**——MCU 上位机看不到 log，悄悄 crash。

### Linux 系统开发

- **`-Wsign-compare` 是 CI 红线**：GCC/Clang 都有这条警告，CI 配 `-Werror` 直接 fail。
- **`UINT_MAX` 检测 wrap 的范式**：
  ```c
  if (a > UINT_MAX - b) { /* 溢出 */ } else { c = a + b; }
  ```
  这是整本书的核心 idiom。**不要写** `if (a + b > UINT_MAX)` —— wrap 已经发生在测试里。
- **`-ftrapv`**：GCC 选项让 signed overflow 触发 trap；CI 开 ASan + UBSan 后**signed overflow 自动 trap**。
- **NaN 传播是真实世界的 bug**：`std::sort` / `qsort` 遇到 NaN 会出错；用 `qsort_r` 自己写比较函数前先 `isnan()` 检查。
- **大整数 → 浮点必丢精度**：`uint64_t → double` 会丢 11 位精度。**金融系统**永远不要 `double` 表示钱；用 `int64_t cents`。

### 机器人软件

- **关节角度 wrap**：舵机/电机编码器通常 `int32_t` 圈数计数。在跨圈路径规划时**必须显式处理 wrap**——ROS `sensor_msgs/JointState` 不做 wrap 处理，靠上层 TF 库。
- **IMU 积分的浮点漂移**：6 自由度 IMU 数据积分几小时就漂出几十米；用 double + 周期性重置。
- **点云 float → depth image int16**：LiDAR 返回 float 米数，要 cast 到 depth image 像素值（mm / cm）；**float → int 越界 = UB**。必须先 `isnormal()` + 范围检查。
- **NaN 在 ROS2 节点间传播**：一个节点输出 NaN，整个 graph 烂掉——用 `guard_condition` 或显式 validation 过滤。

### 汽车电子软件

- **AUTOSAR 禁用浮点 + 禁用动态内存** → 所有算术都用 `sint32` / `uint32` + fixed-point。
- **wraparound 检查是 MISRA Required**：MISRA C 2012 规则 12.x 系列明确要求溢出检测；`Abs()` 类函数必须处理 `INT_MIN`（MISHA 推荐用 `INT_MAX + 1` 类型技巧）。
- **C99 `<stdint.h>` 在 AUTOSAR 里是 mandatory**：编译器必须支持 fixed-width 类型。**某些老旧 MCU 编译器不支持 C99**——这就是为什么汽车里"嵌入式 C"经常卡在 C90。
- **NaN 不应进入控制环**：AUTOSAR SWS 里规定所有传感器数据需经 range check；NaN 必须被拒。

### 初级 vs 高级工程师的关注差

| 维度 | 初级 | 高级 |
|---|---|---|
| unsigned 边界 | `0xFFFFFFFF` 硬编码 | `UINT32_MAX` 宏 |
| 检查 wraparound | `if (a + b > UINT_MAX)` | `if (a > UINT_MAX - b)`（先消除 wrap） |
| `Abs(INT_MIN)` | 直接 `-x` | `if (x == INT_MIN) special; else if (x < 0) -x;` |
| 浮点循环 | `for (float i = 0; i < 1; i += 0.1)` | `for (int i = 0; i < 10; i++) ...; i*0.1f` |
| mixed sign 比较 | 不在意 | 强制 cast 到同 signedness |
| NaN | `NaN == NaN` 是 false → 困惑 | 用 `isnan()` 早过滤 |
| `<limits.h>` vs 字面量 | 看着写 `2147483647` | 永远写 `INT_MAX` |

---

# 四、AI 时代视角

### 这个知识今天是否仍然重要？

**重要性级别：极高（且与硬件紧耦合）**。

- 这一章是 C 与硬件的**最直接接触面**——wraparound、浮点表示、整数提升都直接对应 CPU 行为。
- **AI 经常写错的**：
  - 漏写 wraparound 检查
  - 浮点 == 浮点比较（应该用 epsilon）
  - 把 `-1U` 当 -1 用（= UINT_MAX）
  - 在跨平台代码假设 `sizeof(int) == 4`

### AI 能帮助完成什么

- ✅ 写 wraparound 检查的"减法范式"
- ✅ 把 `if (sum + ui > UINT_MAX)` 改写为 `if (sum > UINT_MAX - ui)` 并解释为什么
- ✅ 用 `<limits.h>` 替换硬编码
- ✅ 解释 `c == ui` 这种 mixed-sign 比较的真实路径

### AI 无法替代什么

- ❌ **决定业务上"这个数值会不会 wrap"** —— 需要理解数据流
- ❌ **选择 fixed-point vs floating-point** —— 涉及性能、精度、硬件
- ❌ **判别"这 NaN 是传感器异常还是数学自然产生"** —— 需要领域知识
- ❌ **写跨 MCU 的 fixed-width 类型映射表** —— 取决于具体 MCU

### 工程师必须掌握的核心能力

1. **二进制 / 十六进制 / 十进制心算能力**：signed 的补码表示、浮点 biased exponent
2. **在 mental model 里跑 wraparound**：知道 32-bit counter 在 49.7 天后 wrap
3. **能识别 AI 生成的 wrap / overflow bug**：因为 LLM 训练语料里"看起来对"的代码很多
4. **掌握 `<limits.h>` / `<stdint.h>` / `<float.h>`**：替代所有硬编码
5. **理解 float == 比较的危险**：几乎永远不应该 `a == b`

---

# 五、实践行动项

### 行动 1: 验证你机器的整数类型与 wraparound 行为
```bash
cat > /tmp/wrap.c <<'EOF'
#include <stdio.h>
#include <stdint.h>
#include <limits.h>
int main(void) {
    printf("INT_MIN = %d, INT_MAX = %d\n", INT_MIN, INT_MAX);
    printf("UINT_MAX = %u\n", UINT_MAX);
    uint32_t u = UINT32_MAX;
    printf("UINT32_MAX + 1 = %u (wraparound to 0)\n", u + 1);
    int32_t s = INT32_MIN;
    printf("-INT32_MIN = %d (UB; likely wraps)\n", -s);
    return 0;
}
EOF
cc -std=c17 -fsanitize=undefined -g -O1 -o /tmp/wrap /tmp/wrap.c && /tmp/wrap
```
**预期**：UBSan 报告 `-INT32_MIN` 是 UB；`UINT32_MAX+1` = 0 是 well-defined。

### 行动 2: 演示错误的 wrap 检查 vs 正确的
```bash
cat > /tmp/wrapcheck.c <<'EOF'
#include <stdio.h>
#include <stdint.h>
// 错误版本
int bad_check(uint32_t a, uint32_t b) {
    return (a + b > UINT32_MAX);   // wrap 已经在测试里发生
}
// 正确版本
int good_check(uint32_t a, uint32_t b) {
    return (a > UINT32_MAX - b);   // 减法不可能 wrap
}
int main(void) {
    printf("bad: %d\n", bad_check(UINT32_MAX, 1));
    printf("good: %d\n", good_check(UINT32_MAX, 1));
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -O2 -o /tmp/wrapcheck /tmp/wrapcheck.c && /tmp/wrapcheck
```
**预期**：bad 永远返回 0（因为已经 wrap）；good 正确返回 1。

### 行动 3: 演示混合符号比较的"惊喜"
```bash
cat > /tmp/mixsign.c <<'EOF'
#include <stdio.h>
#include <stdint.h>
int main(void) {
    int8_t c = -1;
    uint32_t ui = UINT32_MAX;
    printf("c == ui ? %d\n", c == ui);
    printf("-1 == UINT_MAX in C's eyes.\n");
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o /tmp/mixsign /tmp/mixsign.c && /tmp/mixsign
```
**预期**：打印 1。**目的**：把"看起来明显不等"的代码变成可观测事实。

### 行动 4: 演示浮点不能做循环计数器
```bash
cat > /tmp/floatcount.c <<'EOF'
#include <stdio.h>
int main(void) {
    int count = 0;
    for (float f = 0.0f; f < 1.0f; f += 0.1f) count++;
    printf("Iterations (should be 10): %d\n", count);
    count = 0;
    for (int i = 0; i < 10; i++) { (void)(i * 0.1f); count++; }
    printf("Iterations (integer loop): %d\n", count);
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o /tmp/floatcount /tmp/floatcount.c && /tmp/floatcount
```
**预期**：第一次打印可能是 9、10 或 11（看编译器实现）；第二次精确是 10。

### 行动 5: 触发 NaN 传播，演示其传染性
```bash
cat > /tmp/nanprop.c <<'EOF'
#include <math.h>
#include <stdio.h>
int main(void) {
    double x = 0.0 / 0.0;
    double y = x + 1.0;
    double z = y * 3.0;
    printf("NaN == NaN ? %d\n", x == x);
    printf("isnan(NaN) ? %d\n", isnan(x));
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o /tmp/nanprop /tmp/nanprop.c -lm && /tmp/nanprop
```
**目的**：让 NaN 这个"看不见的传染源"变得具体可观察。

---

# 六、值得深入思考的问题

### Q1: 作者说"不要写 `for (unsigned i = n; i >= 0; --i)`"，但实际上 GLib / glibc / Linux kernel 内部大量用 `unsigned` 做计数。**它们怎么避坑的？**

提示：通常改用 `while (i-- > 0)` 或显式 `n != 0`。**问题**：API 设计时，何时该选 `unsigned`，何时该选 `signed`？有没有可量化的决策表？

### Q2: C2x 起只支持 two's complement，但 ARM / x86 历史上有过 ones' complement 机器。**这种"统一 representation"是进步还是限制？**

**进步**：跨平台行为可预测；标准章节减少。
**限制**：嵌入式 / 老 MCU 兼容性受影响。
**问题**：如果当年 C89 就强制 two's complement，今天的 cert / MISRA / sanitizers 会不会更简单？

### Q3: NaN != NaN 是 IEEE 754 设计的，但 IEEE 754 后续版本（2008 / 2019）有 `minNum` / `maxNum` 来定义 NaN 在比较里的语义。**C 标准应该跟随这些进展吗？**

提示：C 标准目前没有 `minNum`；用 `fmin` 处理 NaN 时 NaN 被忽略。**问题**：C 的 `<math.h>` 与 IEEE 754 的对应关系是滞后、领先还是并行？这反映了标准委员会的什么价值取向？

### Q4: `Abs(INT_MIN) = UB` 是一个 1970 年代就存在的 bug 模式。**glibc 用 `return (i >= 0) ? i : -(unsigned)i;` 规避——但这只是 implementation-defined，不是 C 标准保证的。**

**问题**：为什么 C 标准不直接给 `abs()` 一个定义良好的 "INT_MIN 返回 INT_MIN" 语义？这反映了什么设计哲学的取舍？

### Q5: 假设你设计的嵌入式系统每秒产生 10000 个传感器数据点，要存 24 小时。如果用 `uint32_t` 时间戳，会在 4.97 天后 wrap。**如果业务要求"每条数据可追溯 30 天"，你怎么设计时间戳？**

提示：扩展到 `uint64_t`（wrap 周期 ~5 亿年）；或用 epoch + offset；或用 `int64_t` + UTC 秒级偏移（2038 年问题）。**问题**：**为什么嵌入式里仍大量保留 `uint32_t` 时间戳**？是历史包袱还是 trade-off？

---

*下一章预告*: **Chapter 4 — Expressions and Operators** —— lvalue/rvalue / 运算符优先级（precedence table）/ order of evaluation（重要 UB 来源）/ sequence points / `i++ * i++` 是 UB / `++*p` vs `*p++` 的实际差异 / pointer arithmetic。这是写出"看起来对、实际 UB"代码的高发地带。
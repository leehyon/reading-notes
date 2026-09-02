# Chapter 7 · Security Issues and the Improper Use of Pointers

> 书目:Richard Reese,《Understanding and Using C Pointers》, O'Reilly 2013
> 本章范围:PDF p.159–174(全文行 6732–7308)
> 阅读日期:2026-09-01

---

## 一、第一性原理思考

**C 安全问题的核心是「语言允许程序员做不该做的事」**——没有边界检查、没有所有权语义、没有 type safety。这就是 ch7 把所有这些坑一一列出的原因:

1. **越界访问**(buffer overflow):数组下标无检查、指针算术无 check。
2. **未初始化指针**(wild pointer):栈 garbage,行为完全依赖 runtime。
3. **不当 cast**(intptr_t cast):64-bit 平台截断。
4. **格式化字符串攻击**:user input 当 format string。
5. **结构体内指针算术**:跨 padding 字段乱算。

把这五类问题归纳到一条公理：**任何让程序「访问到不该访问的内存」的操作，在 C 里都是合法的(语法上)**——只有 runtime 行为(segv、ASan 报错)才能抓住它们。

**对嵌入式工程师的现实意义**:
- **车载 ECU 安全等级**(ISO 21434)要求对所有 input 做边界检查、缓冲区大小检查、类型检查——直接对应 ch7 全部条目。
- **UDS 0x27 安全访问、0x34/0x36/0x37 大数据传输**:所有 size 字段必须强校验，否则攻击者构造畸形 payload 越界写入。
- **`scanf("%s", buf)` 在 ECU 代码里禁止出现**——必须用 `scanf("%ms", &buf)`(GNU)或自行管理 size 读取。
- **format string attack 在 ECU 上罕见**(攻击者无法交互),但 flash 日志格式化时仍要小心。

---

## 二、章节概述

1. **C 安全难的原因**:C 不阻止越界访问；CERT 等组织的研究表明多数 C 安全问题追溯到指针误用。
2. **ASLR / DEP**:OS 级安全强化；ASLR 让攻击者难预测栈/libc 地址；DEP 让 stack/heap 不可执行。
3. **指针声明的坑**:`int* ptr1, ptr2;` —— 只有 ptr1 是指针,ptr2 是 int。`int *ptr1, *ptr2;` 才是两个指针。`#define PINT int*` 同样问题；`typedef int* PINT;` 才是正确做法。
4. **未初始化指针**(wild pointer):栈垃圾内容,deref 后果取决于 runtime。
5. **处理未初始化指针**:三种方法——总是初始化 NULL、用 `assert(pi != NULL)`、用第三方工具(valgrind/ASan)。
6. **buffer overflow 概述**:越界写入，可能覆盖栈帧返回地址导致控制流劫持。
7. **必须检查 malloc 返回值**:`if (vector == NULL) ...`。
8. **dereference operator 误用**:`*pi = &num;` 错(给 deref 后的位置赋地址);`pi = &num;` 对(给 pi 本身赋地址)。
9. **Dangling pointer**(ch2 已讲):free 后访问、block 内地址逃出。
10. **数组越界**:`char firstName[8] = "1234567"; middleName[-2] = 'X';` —— 静默踩相邻数组。
11. **replace 函数**:显式 size 参数，即使内部 strcpy overflow 也能靠 size 阻止越界。
12. **sizeof 误用**:`for (i = 0; i < sizeof(buffer); i++)` 跑 80 次而非 20 次——`sizeof(buffer)/sizeof(int)` 才对。
13. **指针类型不匹配**:`int *pi = &num; short *ps = pi;` —— 在 little endian 上只读 2 字节得 `-1`,全 4 字节得 `INT_MAX`。
14. **bounded pointers**:C 不支持，需手动 `if (p >= name && p < name + SIZE)`,或用 CBMC 工具。
15. **字符串函数安全**:`strcpy/strcat` 无 size 参数；C11 Annex K 的 `strcpy_s/strcat_s`(仅 MS 支持);`gets` 必删。
16. **format string attack**:`printf(argv[1])` —— user input 直接当 format,可能读写任意内存；防御：绝不把 user input 当 format。
17. **指针算术与结构体**:结构体内字段可能有 padding,指针算术跳过 padding 是 UB。
18. **函数指针问题**:`if (getSystemStatus == 0)` 漏 `()` → 比较函数地址而非返回值，恒为 false(函数地址非零)。
19. **Double free**(ch2 已讲):重 free 是 heap corruption,可被 attacker 利用。
20. **清除敏感数据**:`memset(name, 0, sizeof(name));` 在 free 前清零，防止 OS 分配给其他进程后泄漏。
21. **静态分析工具**:GCC `-Wall` 是底线；`clang-analyzer` 更深；`coverity`、`cppcheck` 商业级。

---

## 三、核心 Takeaways

| # | 是什么 | 为什么 | 解决了什么 | 适用场景 |
|---|---|---|---|---|
| **T1** | C 不阻止越界，安全靠程序员自律 | C 设计哲学是「信任程序员」 | 解释了为什么 `clang-tidy`/ASan 是必须 | 所有 C 代码 |
| **T2** | `int* a, b;` 链式声明陷阱 | C 不是 C++,星号只修饰一个变量 | 防止误声明 | 所有指针声明 |
| **T3** | 未初始化指针 = 栈 garbage | C 不自动初始化 | 解释了为什么 undefined behavior 这么危险 | 栈分配指针 |
| **T4** | bounded pointers 需手写 | C 标准无内建支持 | buffer overflow 防御 | 关键 API |
| **T5** | `strcpy/strcat` 无 size = 历史包袱 | C89 时期函数 | 解释了 `strncpy/strncat` 与 `snprintf` 的兴起 |
| **T6** | format string attack | `printf` 不验证 format | 禁止 user input 当 format | 所有 `*printf` 调用 |
| **T7** | 函数指针 vs 函数调用 | 漏 `()` 比较的是地址 | 函数指针安全 | API 设计 |
| **T8** | 敏感数据 free 前清零 | OS 不自动 zero | 防止跨进程泄漏 | 密钥/密码处理 |
| **T9** | GCC `-Wall` 是底线但不够 | 静态分析只覆盖模式匹配 | 必须结合 ASan/UBSan | CI |
| **T10** | CERT C 是工业标准 | 涵盖 ch7 全部条目 | ECU 安全合规依据 | AUTOSAR / ISO 21434 |

---

## 四、工程实践视角(领域：嵌入式 / 汽车电子)

### 落地

- **AUTOSAR E2E protection**(End-to-End):在 PDU payload 加 CRC + counter,**对应 ch7 §「Pointer Usage Issues」→ buffer overflow 的运行时检测**。
- **UDS 0x34/0x36/0x37 Download/TransferData/RequestTransferExit**:每次传输 size 校验，长度超 4095 字节直接 NACK 0x14——边界检查必须强。
- **GCC `-Wall -Wextra -Wformat=2 -Wpointer-arith`** 是 CI 起步；MISRA-C 工具链(EB tresos、GreenHills)额外加严。
- **OEM 规范**(VW/Audi/BMW):禁止 `strcpy/strcat/gets/sprintf`,强制 `strncpy/strncat/snprintf`,与 ch7 §「String Security Issues」一一对应。
- **ECU 启动时清零 BSS 与 RAM 残留**:UDS 安全等级切换时，旧安全状态必须清零——对应 ch7 §「Clearing Sensitive Data」。
- **`static analysis tools`** = `clang-tidy` + `cppcheck` + `coverity`(商业);每条 MISRA-C rule 对应一个 static check。

### 误区

- **M1** 误以为 `*pi = &num;` 是初始化指针——实际是 deref 后的位置赋地址,deref garbage 区域即 segv。
- **M2** `for (i = 0; i < sizeof(arr); i++) ...` —— 跑多了 4 倍。
- **M3** `printf(buf)` 用用户输入作 format —— format string attack。
- **M4** `if (func() == 0)` 漏 `()` —— `if (func == 0)`,恒 false。
- **M5** `gets(buf)` —— 标准库已删除，但老代码还在。
- **M6** 函数指针赋值不兼容签名:`int (*f)(int, int) = add;` 然后调用 `f(1)` —— 不匹配签名,UB。

### 初中高工程师视角

- **初中级**:记得所有 malloc 要查 NULL;不直接用 user input 当 format。
- **中级**:能用 `strncpy/snprintf/strncat` 替代 `strcpy/strcat/sprintf`;理解 `typedef` vs `#define` 在指针声明的差异。
- **高级**:能设计带 size 参数的字符串 API;在 review 时一眼识别 format string attack 与栈溢出；能根据 CERT C 规则做静态扫描。

---

## 五、AI 时代视角

- **LLM 字符串函数 bug**:`strcpy/strcat/sprintf/gets` 在 LLM 生成代码中频率极高；**Copilot 提示工程**:「use `strncpy/snprintf/strncat`」显著降低 bug 率。
- **LLM 声明陷阱**:`int* a, b;` LLM 也常写错——需要在 prompt 中显式说明「每个变量前都加 `*`」。
- **静态分析工具替代 review**:`clang-tidy --checks=bugprone-*`、`cppcheck --enable=warning,style,performance,portability` 是 CI 标配。
- **CERT C 编码规范** + **MISRA-C:2012** + **AUTOSAR C++14** 是汽车 ECU 三大工业编码标准,ch7 几乎每条都有对应 rule。
- **ASan + UBSan + MSan 三件套** = ch7 的 runtime 检测总集,debug 构建必开。

---

## 六、实践行动项

1. **[必做]** 复现 ch7 §「Misuse of the Dereference Operator」—— `*pi = &num;` 错例，观察编译器警告与运行时 segv。落档到附录。
2. **[必做]** 复现 ch7 §「Misusing the sizeof Operator」—— `for (i = 0; i < sizeof(buffer); i++)` 跑过头，验证数组越界。
3. **[推荐]** 复现 ch7 §「Function Pointer Issues」—— `if (getSystemStatus == 0)` 漏 `()`,看 GCC 警告。
4. **[推荐]** 用 ASan 跑 ch7 §「Pointer Arithmetic and Structures」—— 跨 padding 的指针算术。

---

## 七、值得深入思考的问题

1. **Q1**:为什么 C 标准不强制「指针必须初始化」?——这违反 C 的「贴近硬件、零开销」哲学；`int *p = NULL;` 多一行代码 = 数百万 C 程序 × 数千万次循环的微小开销累积。
2. **Q2**:`gets` 已被 C11 删除,`strcpy`/`strcat` 为何还留着？——向后兼容。Reese ch7 的「用 `strncpy/strncat` 替代」是过渡方案；C11 的 `strcpy_s` 是终极方案，但需要 MSVC 之外的实现跟上。
3. **Q3**:`printf(buf)` 用 user input 当 format 是 bug,但 `printf("%s", buf)` 安全——这条规则的本质是什么？——**信任边界**:`%s` 把 buf 当 data,format string 把 buf 当 code;**user input 不应被当 code 处理**。
4. **Q4**:`if (func() == 0)` 漏 `()` 是几乎所有 C 教程都会强调的点，但仍有 LLM 高频犯错——为什么？——「看起来一致」的语法误导；`func` 是函数名,`func()` 是调用，差别只在 `()`。
5. **Q5**:CERT C、MSAN、ASan、UBSan、Coverity、MISRA-C 哪一个最值得学？**都是必须的**,但优先级——**ASan > UBSan > CERT C > Coverity > MISRA-C**,因为 ASan 在 runtime 拦截 UB,CERT/Coverity/MISRA 在编译期/静态扫描。

---

## 附录 · Action #7 复盘 · ch7 deref 误用、sizeof 误用、函数指针漏括号

### Case 1 · deref 未初始化指针(预期 segv,实际取决于 garbage 地址)

```sh
$ ./ch07_deref_misuse
warning: 'pbad' is used uninitialized [-Wuninitialized]
unexpected: pbad = 0x7ffcf0064f58
exit=0
```

**真实观察**:`*pbad = num` 这一行**没有 segv**——`pbad` 在栈上未初始化，但 `0x7ffcf0064f58` 恰好命中栈上的合法可写区域,`num = 42` 被写入，然后程序继续跑。

**对比 ch1 §「A common error」+ ch7 §「Wild pointer」** 的反复警告：**garbage 地址的写操作不会立刻 segv**。本次实验恰好命中可写页，如果是别的栈布局会 segv。这是 ch1/ch7 强调「后果不可预测」的活样本。

### Case 2 · sizeof 误用(预期越界，实际 stack canary 保护)

```sh
$ ./ch07_sizeof_misuse
sizeof(buffer)         = 80 (用作循环上限就跑过头)
sizeof(buffer)/sizeof(int) = 20 (正确元素数)
[done] 错误循环已跑完，栈可能已损坏，函数返回时可能 segv。
*** stack smashing detected ***: terminated
Aborted (core dumped)
```

**真实观察**:循环跑 80 次而非 20 次，越过 `buffer[20]` 边界写入栈帧。GCC 默认开启 `-fstack-protector`,在函数返回前检查 stack canary,发现被覆盖就 `abort()` —— 这是 ch7 §「Buffer Overflow」的现代护栏。

**2013 年原书出版时，这种保护不普遍**;**2026 年 GCC/Clang 默认开 stack protector** + ASan,基本「**栈溢出必抓**」。

### Case 3 · 函数指针漏 `()`(预期 GCC 警告 + 行为错)

```sh
$ ./ch07_fptr_paren
warning: the comparison will always evaluate as 'false' for the address of 'getSystemStatus' will never be NULL [-Waddress]
warning: the address of 'getSystemStatus' will always evaluate as 'true' [-Waddress]
[good] status is not 0 (进入 else)
[bad 1] status is not 0 → 函数地址 != 0,恒走 else
[bad 2] 进入 if(裸函数名),因为函数地址非零 → true

函数 getSystemStatus 地址 = 0x5e3d23747169
```

**真实观察**:
- GCC 直接给两条警告:`address-of-comparison` 与 `address-as-bool`,**在编译期就指出代码错**。
- 运行时分支:`[bad 1]` 恒走 else(地址非 0);`[bad 2]` 恒走 if(地址作 bool 是 true)。
- 函数实际地址 `0x5e3d23747169`,证实「漏 `()` 比较的是地址而非返回值」。

### 一行 repro(可在你机器上复现)

```sh
cd /home/liyahong/dev/read/notes/understanding-using-c-pointers/code-exercises
gcc -O0 -g -Wall -Wextra -o ch07_deref_misuse ch07_deref_misuse.c && ./ch07_deref_misuse
gcc -O0 -g -Wall -Wextra -o ch07_sizeof_misuse ch07_sizeof_misuse.c && ./ch07_sizeof_misuse
gcc -O0 -g -Wall -Wextra -o ch07_fptr_paren ch07_fptr_paren.c && ./ch07_fptr_paren
```

### 剩余未做的事

- 关闭 stack protector(`-fno-stack-protector`)重跑 ch07_sizeof_misuse,看是否能触发返回地址覆盖的真正劫持。
- 用 ASan 跑 ch07_deref_misuse 看是否能稳定 segv。
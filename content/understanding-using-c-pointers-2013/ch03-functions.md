# Chapter 3 · Pointers and Functions

> 书目:Richard Reese,《Understanding and Using C Pointers》, O'Reilly 2013
> 本章范围:PDF p.57–78(全文行 2810–3558)
> 阅读日期:2026-09-01

---

## 一、第一性原理思考

**C 函数调用的本质是「在调用者的栈帧上方 push 一个新栈帧」**。这个栈帧里装着:

1. **参数**(按声明的逆序压栈——C 调用约定，非全部 ABI 但 x86-64 System V 是这样)
2. **返回地址**(由 `call` 指令压入)
3. **局部变量**(编译器决定顺序)
4. **保存的 callee-saved 寄存器**(rbx、r12–r15 等)

这意味着**所有 C 形参都是「值传递」**——即便是指针，也只复制一份指针变量本身(8 字节),而不是复制整个对象。这条事实同时是 ch1「指针 = 类型化地址」的延伸，也是 ch3 全部设计的根基:

- **传 `int`** → 拷贝值；修改不影响 caller。
- **传 `int *`** → 拷贝地址；通过 `*p` 修改会影响 caller(因为 deref 跳回原对象)。
- **传 `int **`** → 拷贝指向指针的指针；通过 `*pp` 修改 caller 的指针变量本身。

把这三种参数化「所有权/可改性」的语义区分用一张表记清，后面所有代码风格选择都从这推出。

**对嵌入式工程师的现实意义**:
- `CanIf_Transmit(PduIdType, const PduInfoType *)` 形参里 `*PduInfoPtr` 的 `SduDataPtr` 是「**调用者拥有**」的指针——形参声明 `const` 只是保护「本函数不改」,并不转移所有权。**释放仍然归 caller(Com)**。
- 函数指针注册表(CanIf/Rte):回调函数被注册到模块内部 table,模块持有 function pointer,callback 上下文所有权归注册者——这条契约在 AUTOSAR SWS 里隐含但很少写明。
- ISR/Task context switch 时，栈帧是切换的载体——所以 ISR 入口函数禁止持有跨 ISR 的栈地址(对应 §「Pointers to Local Data」)。

---

## 二、章节概述

1. **栈帧布局**:参数逆序压栈 → 返回地址 → 局部变量；block statement 算「mini」函数，有自己的栈帧。
2. **传指针 vs 传值**:`swapWithPointers` 改原值 vs `swap` 不改；后者只是局部 swap。
3. **传指针到常量(`const T *`)**:保护入参不被改；`passingAddressOfConstants(&limit, &limit)` 编译错——指针类型与常量保护矛盾。
4. **返回指针的两种模式**:malloc 出内部 + caller 释放；或 caller 传入 buffer + 函数填充。
5. **返回局部变量的地址 → dangling**:`int arr[size]; return arr;`——函数返回后栈帧被覆盖。
6. **返回 static 数组 → 共享但每次都覆盖**:`static int arr[5]; return arr;`——多次调用共享同一块，旧值会被冲掉。
7. **传 NULL 指针防御**:`if (arr != NULL) {...}`,进函数就检查。
8. **传指针的指针(`T **arr`)**:让函数能改 caller 的指针变量本身。
9. **写自己的 free 函数**:`saferFree(void **)` 把指针置 NULL;`safeFree(p)` 宏包一层避免调用方手写 `&`。
10. **函数指针**:`void (*foo)()`、`int (*f1)(double)` 等；倒读法；与「返回指针的函数」区分。
11. **函数指针 typedef**:简化形参与赋值；`typedef int (*funcptr)(int)`。
12. **传递函数指针**:`compute(add, 5, 6)`;`compute` 内部调用 `operation(num1, num2)`。
13. **返回函数指针**:`select(opcode)` 返回 `add` 或 `subtract` 的指针；`evaluate` 再调用它。
14. **函数指针数组**:`operation ops[128] = {NULL};` 用 ASCII 字符做索引查表。
15. **比较函数指针**:`fptr1 == add`;可识别表项是否被设置。
16. **cast 函数指针**:`fptrFirst = (fptrToTwoInts)fptrSecond;`——不检查参数列表，需小心。
17. **void\* 与函数指针**:POSIX 不保证 `void *` 可与函数指针互通；常见做法是 base 类型 `void (*fptrBase)()` 占位。
18. **分支预测与函数指针**:间接调用打断流水线；但函数指针表(查表)反而比 switch 链快。

---

## 三、核心 Takeaways

| # | 是什么 | 为什么 | 解决了什么 | 适用场景 |
|---|---|---|---|---|
| **T1** | C 形参一律「按值拷贝」 | 编译期 ABI 决定，无引用语义 | 解释为什么改形参不影响 caller | 所有 C 函数 |
| **T2** | `T *` 形参 = 「传地址」,`T **` = 「传地址的地址」 | 形参只是 8 字节拷贝，改它要看 deref 层级 | 区分「改对象」与「改指针本身」 | 初始化分配 |
| **T3** | `const T *` 形参 = 声明「我不改」 | 编译器能查；但 caller 仍可改原对象 | API 文档级承诺 | 函数形参保护 |
| **T4** | 函数返回指针的两种所有权模式 | 调用方决定「何时释放」与「谁释放」 | 明确 ownership contract | 库 API 设计 |
| **T5** | 永远不返回局部变量的地址 | 栈帧 pop 后原地址内容被下一个函数覆盖 | 避免 dangling | 一切返回指针的函数 |
| **T6** | `static` 局部数组=函数间共享 | static 变量不在栈帧上，生命周期是进程级 | 状态缓存，但要小心覆盖 | 小型固定状态 |
| **T7** | 函数指针 = 「地址 + 签名」 | 签名是 ABI 一部分,cast 后不验证 | 多态/回调实现 | 排序、回调表 |
| **T8** | 函数指针数组 = 表驱动 | `ops[opcode]` 一次寻址，无 if-else 链 | DSL、VM、协议解析 | 命令分派 |
| **T9** | 函数指针类型转换 = 不安全 | POSIX 不保证互通 | 跨语言/跨平台需小心 | 写 dynamic loader |
| **T10** | 传 `T **` 让函数能改 caller 的指针 | 比返回值多输出更可控 | `saferFree` 与容器初始化 | 任何要「输出指针」的函数 |

---

## 四、工程实践视角(领域：嵌入式 / 汽车电子)

### 落地

- **AUTOSAR 函数指针注册**:CanIf 的 `CanIf_TransmitConfirmation`、`CanIf_RxIndication` 都是函数指针,BSW 通过 Rte/RTE 间接调用。这是 ch3 函数指针的「跨模块回调」场景——`void (*RxIndication)(PduIdType, const PduInfoType *)` 就是函数指针数组的 real-world 实例。
- **CanIf 模块的 `const PduInfoType *` 形参**:ch1 `const T *` 矩阵 + ch3 「Passing a Pointer to a Constant」联合解释：**CanIf 承诺不修改 payload**,但 caller(CanDrv/Com)仍保留所有权。
- **`PduR` 路由表**用函数指针数组(`BufCfg[]`)实现发往不同 buffer 的分发——对应 ch3 §「Using an Array of Function Pointers」。
- **`static` 局部数组做协议状态缓存**:UDS 0x27 安全访问的「失败计数器」常驻在 `static uint8 failCount;`,跨函数调用保持；这正是 ch3 §「An alternative approach is to declare the arr variable as static」的实践。
- **写一个 `saferFree` 的变种 `bsw_SafeFree(void **pp)`** 作为 BSW 内部工具函数，统一所有 BSW 模块的 free 模式，防止 double free。
- **`compute(fptrOperation, a, b)` 模式**:CanNm 的状态机转换可用此模式，不同状态由不同 operation 处理,`compute(current_state_op, frame_data)`。

### 误区

- **M1** 函数指针形参忘加括号:`void foo(int *fp())` 是「返回 `int *` 的函数声明」,不是「`int *` 的函数指针形参」。正确写法 `void foo(int (*fp)())`。
- **M2** 函数指针 cast 到不兼容签名:`fptrCompute(int,int)` cast 成 `fptrCompute(void)`,运行时 UB。POSIX `dlsym` 返回的 `void *` 必须 cast 回原签名。
- **M3** `if (getSystemStatus == 0)` 漏了 `()` → 比较的是函数地址而非返回值，恒为 false(函数地址非 NULL)。
- **M4** 用函数指针数组时不查 NULL 索引 → 对未设置的 slot 调用 → segv。需先 `if (ops[i]) ops[i](...)`。
- **M5** 写 `int *ptr1, ptr2;` 链式声明 → `ptr2` 是 `int`。正确 `int *ptr1, *ptr2;`。
- **M6** 函数指针比较 `fp == NULL` 漏写「fp 是否合法」语义——fp 是某个特定函数地址，正常情况非 NULL,但「从未初始化」fp 是 dangling。

### 初中高工程师视角

- **初中级**:能识别 `void (*foo)()` 是函数指针；能区分「`T *p`」与「`T **pp`」的不同含义。
- **中级**:能写「`compute(operation, ...)`」式的回调函数；理解 `const T *` 与 `T *const` 矩阵在形参中的语义。
- **高级**:能设计「`T **` 形参 + 内部 malloc」的初始化 API;在 review 时一眼识别「cast 函数指针绕开类型检查」的 UB 风险。

---

## 五、AI 时代视角

- **LLM 函数指针声明高频 bug**:LLM 经常写错 `void (*signal(int, void (*)(int)))(int)` 之类的复杂声明，推荐用 `typedef void (*SigHandler)(int); signal(SIGINT, handler);` 降低认知负担。
- **LLM 误把函数当成「first-class value」**:LLM 经常 `return NULL` 当作「函数指针的 none」,但应明确写「`if (!fp) return ERR;`」并加 NULL check。
- **Copilot 提示工程**:API 形参加 `const`(尤其 `const uint8 *payload, uint16_t len`)显著降低 Copilot 在函数内误改 payload 的概率。
- **MISRA-C 强制**:MISRA-C:2012 Rule 11.1「Conversions shall not be performed between a pointer to a function and any other type」直接对应 ch3 §「Casting Function Pointers」。

---

## 六、实践行动项

1. **[必做]** 复现 ch3 §「Pointers to Local Data」——返回局部数组地址 + 立刻 `printf` 验证内容被覆盖。落档到附录。
2. **[必做]** 复现 ch3 §「Passing a Pointer to a Pointer」——同一逻辑 `T *arr` 与 `T **arr` 的对比，演示前者改不了 caller、后者可以。
3. **[推荐]** 写一个 `ops[128]` 函数指针表做「字符 → 操作」分派(可模拟简易 calculator),打印所有支持的 op 的结果。
4. **[推荐]** 跑 ch3 §「Passing Data by Value vs Pointer」——同一 `swap` 行为对比，打印 swap 前后 caller 的变量值。

---

## 七、值得深入思考的问题

1. **Q1**:C 既然是「按值传递」,为什么 `scanf("%d", &n)` 用 `&n` 而不是 `n`？这暴露 C 的输入参数语义本质上**必须通过指针跨越函数边界**——「无引用」是 C 的设计选择。
2. **Q2**:`T **` 与「返回值」各自解决什么问题？`int *vector = allocateArray(5, 45);` 把指针放在返回值里,`allocateArray(&vector, 5, 45);` 通过输出形参放——前者简洁，后者兼容「**多个输出值**」(`init(a, b, c, &out1, &out2, &out3)`)。Rte 接口因此常用后者。
3. **Q3**:`static` 局部数组在多线程下安全吗？**默认不安全**——它是进程级共享状态，需 mutex。BSW 模块里 `static` 变量如果被并发访问，必须显式说明 critical section。
4. **Q4**:函数指针 cast 在 C 标准里是 UB 吗？POSIX `dlsym` 要求 cast 回原签名，但「cast 错签名调用」是 UB。这与「`void *` 互通数据指针」的标准允许完全不同。
5. **Q5**:为什么 x86-64 System V ABI 把参数**逆序**压栈？从 caller 视角看是「最后 push 的最先被 callee 拿到」——但 ABI 其实是用寄存器传前 6 个参数，栈只在寄存器用完才用。逆序是历史兼容，新 ABI 已经不再严格要求。

---

## 附录 · Action #3 复盘 · ch3 pointer-to-pointer 与 dangling local

### Case 1 · 返回局部数组地址(预期 UB,实际 segv)

```sh
$ ./ch03_dangling_local
=== case1: bad_allocateArray (returning local VLA) ===
Segmentation fault (core dumped)
```

**真实观察**:编译器在编译期就给了警告:
```text
warning: function returns address of local variable [-Wreturn-local-addr]
```

运行时，`int *bad = bad_allocateArray(...)` 拿到一个**栈帧地址**，紧接着 `printf("bad[0..4] = %d ...", bad[0]...)` 的 deref 直接 segv —— **不是「读到 stale 值」而是「访问已被覆盖/无效的栈帧」**。

**对比 Reese 原书 p.66** 说的「Each array element may still contain a 45」:那是 1990 年代没有 ASLR/栈保护的情况；**2026 年的 glibc + GCC 在 -O0 下也会立刻 segv**。这是「UB 的具体表现随 runtime 变化」的活样本。

### Case 2 · `T *` 形参 vs `T **` 形参(预期 NULL vs 45)

```sh
$ ./ch03_ptr_to_ptr
[case A: bad]   vector_a: (null)
[case B: good]  vector_b: 45 45 45 45 45 (addr 0x5a4cb016e2d0)
```

**真实观察**:
- `case A`:`vector_a` 仍是 `NULL`——`allocateArray_bad` 内的 `arr = malloc(...)` 只改了形参副本,caller 的 `vector_a` 不知道堆地址。
- `case B`:`vector_b` 已被 `allocateArray_good(&vector_b, ...)` 通过 `T **` 改写，内容 `45 45 45 45 45`。

**这正好是 AUTOSAR BSW 模块的「初始化分配」契约范本**:`init(SomeHandle **out, size_t size)` 通过 `T **` 输出指针，调用方拿到所有权后负责 free。

### 一行 repro(可在你机器上复现)

```sh
cd /home/liyahong/dev/read/notes/understanding-using-c-pointers/code-exercises
gcc -O0 -g -Wall -Wextra -o ch03_dangling_local ch03_dangling_local.c
./ch03_dangling_local        # 预期 segv
gcc -O0 -g -Wall -Wextra -o ch03_ptr_to_ptr ch03_ptr_to_ptr.c
./ch03_ptr_to_ptr            # 预期 vector_a=(null), vector_b=45 45 45 45 45
```

### 剩余未做的事

- 在 `-O2 -fomit-frame-pointer` 下重跑 case1,看是否同样 segv(优化关闭 frame pointer 让 stale 数据 更难复现，但 dangling 仍 UB)。
- 用 `valgrind --tool=memcheck` 重跑，看 valgrind 给出的具体诊断。
# Chapter 4 · Pointers and Arrays

> 书目:Richard Reese,《Understanding and Using C Pointers》, O'Reilly 2013
> 本章范围:PDF p.79–105(全文行 3670–4606)
> 阅读日期:2026-09-01

---

## 一、第一性原理思考

**数组在 C 里既是一个「地址」又是一个「不可赋值的符号」**。这个二象性是本章几乎所有讨论的源头:

- `int vector[5] = {1,2,3,4,5};` → `vector` 是「第一个元素的地址」,但 `vector = vector + 1;` 编译错(数组名不可作为 lvalue)。
- `int *pv = vector;` → `pv` 是「独立的指针变量」,可被赋值、被算术运算、被 deref。
- `vector[i]` 与 `*(vector + i)` 等价，但生成的机器码不同:`vector[i]` 是一次 deref,`vector+i` 是先算地址再 deref。

把这层「符号 vs 变量」的区分立住，后面所有「`sizeof(vector) = 20` vs `sizeof(pv) = 8`」、「`vector = vector + 1` 编译错 vs `pv = pv + 1` OK」、「多维数组参数必须给出除第一维外的所有维度」都能从同一个根推出。

**对嵌入式工程师的现实意义**:
- **CAN 帧 / UDS 响应通常是固定大小数组**(`uint8 canPayload[8]`、`uint8 udsResp[4095]`)。这些数组传给低层函数时，**地址是值传递**,底层必须知道长度——这是 CANoe/Vector 工具栈里「payload + DLC」成对传递的根因。
- **多维数组对应协议字段表**:`Dcm_ReturnParameterType Dcm_<func>(PduIdType, const uint8 *payload, uint16_t len)` —— `payload` 一维数组 + 长度，而不是二维表；这与 ch4 §「Passing a One-Dimensional Array」一致。
- **jagged array** 在嵌入式不常见，但 UDS 服务子函数(0x22 DIDRead)的多 DID 响应可以用「指针数组」模拟「变长子响应」。

---

## 二、章节概述

1. **数组快速回顾**:一维初始化、sizeof 算元素数、二维(行主序 row-major)、三维 rank 维。
2. **数组与指针**:数组名 = 首元素地址；`&array` = 「指向整个数组的指针」(`int (*)[N]`)。
3. **`pv[i]` 等价 `*(pv+i)`**:stride 由 `*pv` 类型决定。
4. **数组 vs 指针的差异**:`sizeof` 结果不同；数组名不可作为 lvalue;`vector+1` 合法但 `vector=vector+1` 非法。
5. **`malloc` 当一维数组**:`int *pv = malloc(5 * sizeof(int)); pv[i] = ...;` 用法与数组同。
6. **`realloc` 重定大小**:`getLine` 函数按 10 字节为增量；`trim` 函数原地缩。
7. **传一维数组**:形参 `int arr[]` 与 `int *arr` 等价；必须**额外传 size**。
8. **`sizeof(arr)` 在函数内返回指针大小**,不是数组大小——常见 bug。
9. **数组指针数组**:`int *arr[5]; for (...) arr[i] = malloc(sizeof(int)); *arr[i] = i;` 两级 deref。
10. **二维数组与多维数组**:行主序；`int (*pmatrix)[5] = matrix;` 指针 stride 是行大小。
11. **传多维数组**:必须指定**除第一维外**的所有维度；`int arr[][5]` 与 `int (*arr)[5]` 等价。
13. **动态分配二维数组**:三种方式
    - **非连续**(每行单独 malloc)
    - **伪连续**(外层指针数组 + 一整块行 body)
    - **完全连续**(一次 malloc + 手工算 `(i*cols + j)`)
14. **compound literals 创建 jagged**:`int (*(arr1[])) = { (int[]){0,1,2}, ... };`
15. **jagged array 的代价**:三行长度不同，需每行单独记录 size;访问比 rect 数组麻烦。

---

## 三、核心 Takeaways

| # | 是什么 | 为什么 | 解决了什么 | 适用场景 |
|---|---|---|---|---|
| **T1** | 数组名 = 首元素地址(非 lvalue) | C 标准规定的语义，符号到内存映射 | 解释 `arr[i]` 与 `*(arr+i)` 等价 | 所有数组 |
| **T2** | `sizeof(arr)` 在数组上下文中是数组大小，在指针上下文中是指针大小 | 编译期决定，取决于「声明」 | 区分 `int arr[5]` 与 `int *arr` | 函数形参 |
| **T3** | `vector+i` 按 `sizeof(T)` 缩放 | stride 由类型决定 | 让 `*(vector+i)` 与 `vector[i]` 等价 | 指针遍历 |
| **T4** | 多维数组必须指定除第一维外所有维度 | 编译器需要 stride 算下标 | 函数形参 `int arr[][5]` | 协议响应表 |
| **T5** | `malloc(N*sizeof(T))` 创建「伪数组」 | 编译器不知大小，需运行时记 length | 动态大小数组 | 输入缓冲 |
| **T6** | `realloc` 增/减可能搬移，必须用返回值 | heap 连续性不可保证 | 动态数组扩容 | getLine 类 |
| **T7** | jagged array = 「指针数组 + 多个小数组」 | 各行长度独立 | 不规则表格 | 命令/参数表 |
| **T8** | compound literal 可用作数组初始值 | C99 语法，栈上分配 | 写不规则表 | 协议定长响应 |
| **T9** | 二维数组完整连续分配 vs 指针数组分配 | 性能(cache locality)+ 释放复杂度 | 选择合适方式 | 图像/矩阵 |
| **T10** | `int (*p)[N]` 是「指向 N 个 int 数组」的指针 | `(*p)` 括号决定绑定 | 步进 = 一整行 | 多维遍历 |

---

## 四、工程实践视角(领域：嵌入式 / 汽车电子)

### 落地

- **CAN 帧 payload `[8]`**:可声明 `uint8 canData[8];`,传给 `Can_Write(uint32 canId, const uint8 *data, uint8 len)` —— 必须显式传 `len=8`,因为形参 `data` 已退化为 `uint8 *`。
- **多维数组服务表**:`Dcm_<service>_Cfg[NUM_PDUS][MAX_PARAMS]` 形参进函数时要 `Dcm_<service>_Cfg[][MAX_PARAMS]` —— 第二个维度必须保留。
- **jagged DID 表**:`Dcm_DidConfig[DID_0xF186] = { .Length = 17, .Data = &Dcm_DefaultData_DID_0xF186 };` —— 长度可变的 DID 信息用「指针 + size」而不是「固定数组」。
- **CanTp 分帧**:多帧首帧(FF)+ 连续帧(CF)用「固定 buffer + index」实现，等价于「`uint8 buffer[MAX_LEN]; uint16 index;`」+ 不用 realloc(避免 heap)。
- **`realloc` 在 ECU 上慎用**:`getLine` 用 realloc 增长，但 ECU 上 realloc 会触发碎片整理——必须 pre-allocate 整个 buffer。
- **jagged 不适用 ECU**:因为 `free` 责任多(`free(list[i]); free(list);`),易漏。

### 误区

- **M1** 函数内 `sizeof(arr)` 以为是数组大小——实为指针大小。bug 经典。
- **M2** `display2DArray(int *arr[5], int rows)` 误以为是二维数组形参——实是「指针数组」,永远按行 stride 1。
- **M3** 写 `int *p = matrix; p += 1; printf("%d\n", *p);` —— `p` 现在指向 `matrix[0][1]`?**不**,指向「5 个 int 的整行」(`int (*)[5]` vs `int *` 步长差 20 vs 4)。
- **M4** jagged 的每行忘了 `free`,只 `free(list);` —— 内存泄漏。
- **M5** 多维数组 `int arr[][5][10]` 形参漏写维度 —— 编译器拒绝。
- **M6** `realloc(p, 0)` 当 free 用 —— 跨平台不一致(glibc/musl/BSD)。

### 初中高工程师视角

- **初中级**:能用 `int arr[N]`、`int *p = arr;` 进行基本遍历；理解 `sizeof` 在不同上下文的大小。
- **中级**:能写动态分配的多维数组(三种模式之一);理解 jagged array 的应用场景。
- **高级**:能在 review 时识别「函数内 `sizeof` 误用」,要求显式传 size;能根据 cache locality 选二维数组的分配模式。

---

## 五、AI 时代视角

- **LLM 多维数组声明 bug**:LLM 经常 `void foo(int arr[][])` 漏维度，或 `void foo(int *arr[5])` 与「二维数组」混淆。
- **LLM `sizeof(arr)` 误用**:LLM 经常在函数内写 `for (i = 0; i < sizeof(arr)/sizeof(int); i++)`,应当 `n` 显式传。
- **Copilot 提示工程**:形参写 `(const uint8 *data, size_t len)` 比写 `(uint8 *data)` 更不容易让 LLM 漏 len。
- **MISRA-C 强制**:MISRA-C:2012 Rule 18.5「Declarations should contain no more than two levels of pointer nesting」与 ch4 间接级数上限呼应。

---

## 六、实践行动项

1. **[必做]** 复现 ch4 §「Differences Between Arrays and Pointers」——演示 `sizeof` 在不同上下文的大小、`vector = vector + 1` 编译错、`pv = pv + 1` 合法。落档到附录。
2. **[必做]** 复现 ch4 §「Passing a Multidimensional Array」——三种声明 `int arr[][5]`、`int (*arr)[5]`、`int *arr` 的 stride 差异。
3. **[推荐]** 写一个 `getLine` 风格的可变长 string 收集器(用 `realloc` 增长),打印每次扩容的「原地址 / 新地址 / 新容量」。
4. **[推荐]** 跑 jagged array 的三种分配模式(noncontiguous / contiguous 双 malloc / contiguous 单 malloc),打印每种模式下 `&arr[i][0]` 的实际地址间隔，验证 contiguous 模式地址相邻。

---

## 七、值得深入思考的问题

1. **Q1**:为什么 C 不允许 `vector = vector + 1`？数组名「不可作为 lvalue」的本质是什么——它表达「这块内存的起始地址由编译器固定，运行时不能改」,这与「指针是变量」的根本对比。
2. **Q2**:`sizeof` 在数组上下文中是数组大小，在指针上下文中是指针大小——为什么不能「在编译期追踪」？答:C 标准允许函数形参 `int arr[]` 等价 `int *arr`,编译器在函数体内无法区分。
3. **Q3**:多维数组为何只「必须指定除第一维外的维度」？因为**下标算式 `arr[i][j]` 需要 `i * (sizeof inner) + j`**只需知道 inner 的 stride，而 outer 的 stride 可由数组总长推断。
5. **Q4**:jagged array 在嵌入式几乎不用——为什么？答：每行单独 `malloc` + `free` 的双重责任，加 cache locality 差(行之间不连续),在内存受限的 MCU 上是反模式。
6. **Q5**:`realloc(p, 0)` 当 free 用安全吗？POSIX 规定「may free and return NULL」,glibc/musl/BSD 实现各异——写跨平台代码时显式 `free(p); p = NULL;` 更稳。

---

## 附录 · Action #4 复盘 · ch4 数组/指针语义与多维 stride

### Case 1 · sizeof 在不同上下文的差异

```sh
$ ./ch04_array_vs_ptr
[main] sizeof(vector) = 20 (数组 5 个 int = 20)
[main] sizeof(pv)     = 8 (指针 8)
[main] pv+1 后 *pv = 2
[in_function] sizeof(arr) = 8 (指针大小)
[in_function] arr[0]=1 (注:arr 已退化为指针)
```

**真实观察**:
- `main` 内 `sizeof(vector) = 20` (数组上下文)
- `main` 内 `sizeof(pv) = 8` (指针上下文)
- `in_function` 内 `sizeof(arr) = 8` —— **关键**:函数形参 `int arr[]` 已退化为 `int *arr`,所以 `sizeof` 看不到原数组大小。

**踩到的坑**:编译器直接警告:
```text
warning: 'sizeof' on array function parameter 'arr' will return size of 'int *' [-Wsizeof-array-argument]
```
这意味着 ch4 §「Using Array Notation」的 sidebar 「A common mistake is to use the sizeof operator with the array」**有现代编译器兜底**——你写出来会被警告。

### Case 2 · 二维数组的 stride

```sh
$ ./ch04_2d_stride
matrix       = 0x7fff165124f0
matrix + 1   = 0x7fff16512504   （跳过 20 字节，等于 sizeof(int[5])）
&matrix[0][0]= 0x7fff165124f0
&matrix[0][0]+1 = 0x7fff165124f4 (跳过 4 字节)
matrix[0][0] = 1, matrix[1][0] = 6

pmatrix  = 0x7fff165124f0 (= matrix)
pmatrix+1= 0x7fff16512504 (行 stride = 20)
*pmatrix = 0x7fff165124f0 (= &matrix[0][0])
```

**真实观察**:
- `matrix` 与 `&matrix[0][0]` 同地址 → 都是首元素地址。
- `matrix + 1` 与 `&matrix[0][0] + 1` 差 16 字节(20 - 4),**前者跳一行，后者跳一个 int**。
- `*pmatrix == &matrix[0][0]`,确认 `int (*)[5]` deref 后回到 `int *`。

这正是 ch4 §「Pointers and Multidimensional Arrays」的核心：**`int (*p)[5]` 与 `int *p` 是完全不同的指针类型**,步进单位是「一行」vs「一个元素」。

### 一行 repro(可在你机器上复现)

```sh
cd /home/liyahong/dev/read/notes/understanding-using-c-pointers/code-exercises
gcc -O0 -g -Wall -Wextra -o ch04_array_vs_ptr ch04_array_vs_ptr.c && ./ch04_array_vs_ptr
gcc -O0 -g -Wall -Wextra -o ch04_2d_stride ch04_2d_stride.c && ./ch04_2d_stride
```

### 剩余未做的事

- 加 `-O2` 重跑 ch04_2d_stride,看优化是否改变 stride(不会，但可以观察编译器是否把 `matrix + 1` 折叠成 `matrix[1]`)。
- 写 jagged 版本(`int **(arr2[])` 三种长度),打印每行首地址，验证 noncontiguous 模式确实「行不连续」。
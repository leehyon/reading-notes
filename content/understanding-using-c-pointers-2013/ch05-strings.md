# Chapter 5 · Pointers and Strings

> 书目:Richard Reese,《Understanding and Using C Pointers》, O'Reilly 2013
> 本章范围:PDF p.107–132(全文行 4718–5588)
> 阅读日期:2026-09-01

---

## 一、第一性原理思考

**C 字符串的本质是「以 `'\0'` 结尾的 char 数组」**。这条规则决定了所有字符串操作必须遵守的两条铁律:

1. **`strlen(s) < 实际分配字节数`**(`'\0'` 必须预留位置,`malloc(strlen(s)+1)`)。
2. **`strcpy/strcat/strcmp` 操作到 `'\0'` 截止**(所有 <string.h> 函数族假设这一定)。

字符串字面量(「...」)的存储位置是另一个独立维度：**literal pool**——一块只读内存，内容由编译器合并去重；同一字符串多次出现可能复用同一地址。这条规则决定了 ch5 §「Standard String Operations」的多个陷阱:

- `char *p = "hello"; *p = 'H';` —— 修改只读段,UB。
- `strcat(s1, s2)` 直接拼接 s2 到 s1 —— 若 s1 是字面量，**写入只读段**即段错误。
- `strcat(error, errorMessage)` 当 errorMessage 紧邻 error 时 —— 字面量相邻导致覆盖错误。

**对嵌入式工程师的现实意义**:
- UDS 服务响应(`PositiveResponse` 0x50+) 的 payload 用 `uint8[4095]` 而非 `char[]` —— UDS 是字节流而非 ASCII,但**相同的陷阱存在**(CAN 帧 vs UDS payload 拼接)。
- **字符串字面量进入 ROM** 在 MCU 上是优化机会(Const 段 `rodata`),但 `char *p = "x"; *p = 'y';` 会写 flash —— **bootloader OTA 升级尤其要警惕**,改 flash 会触发 fault。
- **`snprintf(buf, size, ...)` 比 `sprintf(buf, ...)` 更安全**——后者没有 size,buffer overflow 高发。

---

## 二、章节概述

1. **字符串基础**:`'\0'` 终止的 char 序列；不一定所有 char[] 都是字符串；字节 vs 宽字符串(`wchar_t`)。
2. **字符串字面量**:进 literal pool(可去重),默认只读；GCC `-fwritable-strings` 可关闭只读(不推荐);MS `/GF` 显式开启去重。
3. **字面量不是常量时的修改**:`*tabHeader = 'L';` 在 GCC 可改写但 UB,改用 `const char *` 强类型保护。
4. **字符串初始化三种形式**:`char header[] = "..."`(拷贝到栈数组);`char *header = malloc(strlen+1); strcpy(header, "...");`(动态);`char *header = "..."`(只指向字面量，不可写)。
5. **`sizeof` vs `strlen`**:`sizeof(arr)` 是数组大小,`strlen(s)` 是字符串长度(不含 `'\0'`)。
6. **字符串内存位置**:全局 / static / local / heap —— 四种 lifetime 对应不同所有权语义。
7. **`strcmp(s1, s2)`**:`< 0` / `== 0` / `> 0`,按字典序；**不能**用 `==` 比较字符串(那是指针地址比较)。
8. **`strcpy(dst, src)`**:dst 需够大(包含 `'\0'`);返回 dst。
9. **`strcat(dst, src)`**:dst 必须够大，会把 src 拼接到 dst 末尾；若 dst 是字面量则写只读段,segv。
10. **`snprintf(buf, size, ...)`**:格式化到 buf,不会越界 size。
11. **传字符串形参**:三种调用方式 `func(arr)` / `func(&arr)` / `func(&arr[0])`,前两种等价。
12. **形参 `const char *`**:保护入参不被改；对应 ch1 `const T *`。
13. **`format(buf, size, name, qty, weight)` 模式**:caller 提供 buf + size,函数填；**buf 分配责任归 caller**。
14. **`format(NULL, 0, ...)` 模式**:传 NULL 触发函数内部 malloc,返回后 caller free。
15. **`argc` / `argv`**:`int main(int argc, char *argv[])` 等价 `char **argv`;`argv[0]` 是程序名。
16. **返回字面量指针**:返回 literal pool 地址，只读，安全。
17. **返回 static 数组指针**:多次调用共享同一块，旧值会被覆盖。
18. **返回动态分配字符串指针**:caller 必须 free;`printf("%s", blanks(5))` 是常见泄漏。
19. **返回局部数组指针**:dangling,UB。
20. **`qsort` / 自定义 sort 函数指针**:`typedef int (*fptrOperation)(const char *, const char *);` —— 注入比较函数，实现可复用 sort。

---

## 三、核心 Takeaways

| # | 是什么 | 为什么 | 解决了什么 | 适用场景 |
|---|---|---|---|---|
| **T1** | C 字符串 = `'\0'` 终止的 char 数组 | <string.h> 函数族假设这一定 | 表达文本 | 所有字符串 |
| **T2** | literal pool 去重 + 只读 | 编译器优化 + 类型保护 | 节省内存 + 防误写 | 全局字符串 |
| **T3** | `sizeof(arr)` vs `strlen(s)` | 前者是容量，后者是长度 | 决定 malloc 参数 | 字符串初始化 |
| **T4** | `strcpy/strcat` 无 size 参数 | 历史包袱,C89 时期函数 | 老代码常见 | 谨慎使用 |
| **T5** | `snprintf` 永远比 `sprintf` 安全 | 知道 size | 防止 buffer overflow | 所有 sprintf 调用 |
| **T6** | `strcmp` vs `==` 比较字符串 | `==` 是指针地址 | 字符串比较正确写法 | 字符串比较 |
| **T7** | 函数返回字符串的四种所有权模式 | caller vs callee 责任不同 | 明确 ownership | 库 API 设计 |
| **T8** | `argc/argv` 是 `char **` | main 函数特例 | 命令行参数 | 工具程序 |
| **T9** | 函数指针注入排序 | 比较策略外部化 | sort 函数可复用 | 排序、过滤 |
| **T10** | 字面量 vs 字符串拷贝 | 前者不可写，后者可写 | API 设计权衡 | 配置项 |

---

## 四、工程实践视角(领域：嵌入式 / 汽车电子)

### 落地

- **CAN/UDS payload 是字节流而非 ASCII**,但「长度 + 缓冲」的模式完全相同:`PduInfoType.SduDataPtr` (uint8*) + `SduLengthType`。
- **`snprintf` 在 BSW 中常用**:格式化诊断响应(`Dcm_<Service>_FormatResponse`),显式 size 防 overflow。
- **字符串字面量进 Const 段**:`const char *Dcm_ServiceName = "ReadDataByIdentifier";` —— 链接器放 `.rodata`,无 RAM 开销。
- **`static char buffer[64]` 模式**(Reese `staticFormat`):某些 ECU 用 static buffer 做格式化输出，**但**跨任务访问需 mutex。
- **命令行参数 argc/argv** 在 ECU 启动脚本中常见(从命令行读配置),但**生产 ECU 通常无 shell**,bootloader 才会用。

### 误区

- **M1** `char *p = "hello"; *p = 'H';` —— 字面量只读,segv 或 write fault。
- **M2** `strcat(literal, other)` —— literal 不可写。
- **M3** `if (s1 == s2) ...` —— 这是指针地址比较，不是字符串内容。
- **M4** `sprintf(buf, ...)` 不传 size —— buffer overflow。
- **M5** `char *header = malloc(strlen("..."));` —— 忘了 `+1` 给 `'\0'`。
- **M6** `printf("%s", blanks(5));` —— `blanks` 返回的指针没保存就丢，泄漏。

### 初中高工程师视角

- **初中级**:会用 `strcpy/strcat/strlen/strcmp`;理解字面量只读。
- **中级**:能用 `snprintf` 替代 `sprintf`;理解字面量 vs `char[]` vs `char *` 三种初始化差异。
- **高级**:设计字符串 API 时明确 ownership;能用函数指针注入比较函数实现可复用 sort。

---

## 五、AI 时代视角

- **LLM 字符串 bug 高频**:`strcpy/strcat` 直接出，忽略 size;`char *p = "..."` 然后修改；`strcmp` vs `==` 混用。
- **Copilot 提示工程**:API 形参加 `const`(尤其 `const char *`)显著降低误改。
- **MISRA-C 强制**:MISRA-C:2012 Rule 7.4「String literals shall not be assigned to objects unless the objects are of pointer to const type」直接对应 ch5 §「When a string literal is not a constant」。
- **MISRA-C Rule 21.6**:`<stdio.h>` 的 `sprintf/snprintf/printf` 等函数使用受规则约束，鼓励 `snprintf` 替代 `sprintf`。

---

## 六、实践行动项

1. **[必做]** 复现 ch5 §「The String Literal Pool」—— `printf("%p\n", "hello"); printf("%p\n", "hello");` 看是否同地址；再演示 GCC `-fwritable-strings` 下能否 `*p='H'`(本机不开)。落档到附录。
2. **[必做]** 复现 ch5 §「Comparing Strings」—— `if (s1 == "literal")` 的错误 vs `if (strcmp(s1, "literal") == 0)` 的正确。
3. **[推荐]** 写 `snprintf` vs `sprintf` 在 overflow 时的行为对比:`char buf[5]; sprintf(buf, "%s", "verylongstring");` 看实际写到 buf 后面的内存是否被破坏。
4. **[推荐]** 写 `sort` + 函数指针比较的 demo:case-sensitive 与 case-insensitive 两套排序，验证可复用。

---

## 七、值得深入思考的问题

1. **Q1**:为什么 C 把字符串当作「带 `'\0'` 终止的数组」而不是「带 length 字段的结构」？历史包袱:UNIX 早期 `bcopy` 用 `'\0'` 作哨兵比 `len` 更省 4 字节；但这导致 NUL 字符无法在字符串中出现——这是 PASCAL 风格字符串的代价。
2. **Q2**:`char header[] = "...";` 与 `char *header = "...";` 在 ROM/RAM 上有何不同？前者把字面量拷贝到栈数组(占 RAM),后者只复制指针到字面量(字面量在 ROM/rodata)。**MCU 上后者更省 RAM**。
3. **Q3**:`strcat(error, errorMessage)` 当二者相邻时为何会错？literal pool 在某些布局下确实相邻,`strcat` 把 errorMessage 内容写到 error 位置，**起始地址被覆盖**,后续读 errorMessage 时已是混合后的字符串。
4. **Q4**:返回 `static` 数组的指针为何会「两次调用互相覆盖」?static 变量不在栈帧，生命周期是程序级；多次调用**复用同一块**，所以后一次写入覆盖前一次。
5. **Q5**:函数指针比较策略注入的 sort,为什么比 `qsort` 简单？`qsort` 用 `void *` + 比较函数签名固定；自己写的 sort 可以有更丰富的 metadata(如 `compareIgnoreCase` 临时分配 tolower 副本并 free)。

---

## 附录 · Action #5 复盘 · ch5 literal pool、strcmp vs ==、snprintf 越界

### Case 1 · Literal pool 合并 + 字面量只读

```sh
$ ./ch05_literal_pool
a = 0x6499bfa2e035
b = 0x6499bfa2e035
same address? YES (literal pool 合并)
literal in printf = 0x6499bfa2e035
literal again     = 0x6499bfa2e035

尝试 *p='L' (modifying literal)...

  >> SIGSEGV: 试图写只读 literal pool
```

**真实观察**:
- 三个 `"hello world"` 字面量**全部同地址** → GCC 默认合并到 literal pool。
- `*p = 'L'` 改字面量 → **glibc 直接 SIGSEGV**(rodata 段在现代 ELF 中是 read-only mapped)。

**对比 Reese 原书 p.110**「some compilers may permit modification」—— GCC 2013 年默认允许(可执行 `-fwritable-strings`),**2026 年 GCC 13 默认禁止**(已合并到 `-fno-rodata` 关闭机制)。这是 C 标准与编译器默认行为**都在收紧**的样本。

### Case 2 · `==` vs `strcmp`

```sh
$ ./ch05_strcmp_eq
warning: comparison with string literal results in unspecified behavior [-Waddress]
command     = 0x7ffc1b828dc0, content = "Quit"
"Quit" lit  = 0x64ddb7534008
[bad if] command == "Quit"  → FALSE (因为 stack address != literal pool address)
[good] strcmp(command, "Quit") == 0 → TRUE (内容匹配)
[note] p1 == p2 → TRUE (字面量 pool 合并)
```

**真实观察**:
- `command` 在 `0x7ffc...`(栈),`"Quit"` 字面量在 `0x64ddb7534008`(rodata)—— 地址完全不同。
- `strcmp == 0` 走内容比较，正确返回 TRUE。
- 两个指针都指向 `"Quit"` 字面量时,`==` 是 TRUE(literal pool 合并)。

**踩到的坑**:GCC `-Waddress` 直接警告 `if (command == "Quit")` 是 unspecified behavior。

### Case 3 · `snprintf` / `strncpy` 安全截断

```sh
$ ./ch05_snprintf_overflow
warning: '%s' directive output truncated writing 19 bytes into a region of size 8 [-Wformat-truncation=]
[snprintf] buf2 = "verylon"  return = 19 (表示「需要写 19 字节」)
[snprintf] 后 8 字节 hex: 76 65 72 79 6c 6f 6e 00
[strncpy]  buf3 = "verylon"
```

**真实观察**:
- `snprintf(buf2, 8, "%s", "verylongstringover8")` 截断成 `"verylon"`(7 字符 + `'\0'`),返回 19 表示「需要 19 字节」—— 调用方可据此检测 truncation。
- `strncpy + 手动加 '\0'` 是 Reese p.124 的等价安全模式。
- GCC `-Wformat-truncation=` 在编译期就把 truncation 标出，**这是 ch7 §「String Security Issues」的现代编译器护栏**。

**额外观察**:原本想写 `sprintf(buf, "%s", "verylongstringover8")` 演示不安全版本，**gcc -Wall 直接拒绝编译**:
```text
warning: '%s' directive writing 19 bytes into a region of size 8 [-Wformat-overflow=]
```
现代编译器已经**主动把可检测的 overflow 在编译期拦下**——这是 ch7 §「Using Static Analysis Tools」的极致体现。

### 一行 repro(可在你机器上复现)

```sh
cd /home/liyahong/dev/read/notes/understanding-using-c-pointers/code-exercises
gcc -O0 -g -Wall -Wextra -o ch05_literal_pool ch05_literal_pool.c && ./ch05_literal_pool
gcc -O0 -g -Wall -Wextra -o ch05_strcmp_eq ch05_strcmp_eq.c && ./ch05_strcmp_eq
gcc -O0 -g -Wall -Wextra -o ch05_snprintf_overflow ch05_snprintf_overflow.c && ./ch05_snprintf_overflow
```

### 剩余未做的事

- 在 `ch05_snprintf_overflow.c` 重新加 `sprintf` 但禁用 `-Wformat-overflow`,看实际溢出是否覆盖返回地址(stack canary 在 GCC 默认会拦截，可能拿不到预期结果)。
- 在 `ch05_literal_pool.c` 试 `-z norelro` / `objcopy --writable-text` 等关闭只读映射的实验。
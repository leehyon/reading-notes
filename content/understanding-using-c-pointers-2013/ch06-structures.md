# Chapter 6 · Pointers and Structures

> 书目:Richard Reese,《Understanding and Using C Pointers》, O'Reilly 2013
> 本章范围:PDF p.133–158(全文行 5700–6614)
> 阅读日期:2026-09-01

---

## 一、第一性原理思考

**结构体在 C 里是「字段在内存中按声明顺序排列，但可能有 padding」**。这条规则带出三条核心约束:

1. **`sizeof(struct)` ≥ 各字段 sizeof 之和**——因对齐 padding,常大于。
2. **`->` 与 `.` 的等价性**:`p->field` ≡ `(*p).field`(括号强制 deref 先于 `.`)。
3. **结构体内的指针字段不会自动管理内存**——这是 ch6 §「Structure Deallocation Issues」的核心:alloc 一个 struct 时，只 alloc struct 本身，**struct 里指针字段若要指向数据，要另行 malloc**。

把这三点立住，后续所有「链表/队列/栈/树」都能从同一份内存图推出 —— 每个节点是一个 struct,节点之间用 struct 里的指针字段串联，**每个节点的 data 字段又指向另一块 alloc 的内存**。

**对嵌入式工程师的现实意义**:
- AUTOSAR Dcm 的 `Dcm_<Session>_CfgType` 常包含 `PduIdType` + `uint16` + `const uint8 *`(指针)——配置项在 ROM,但数据指针指向 RAM,生命周期解耦。
- **结构体内指针字段的 deallocation** 是 review 重灾区：**先 free 每个指针字段，再 free struct 本身**。漏一个就泄漏。
- **链表/队列/栈/树**:这章其实是「OS 数据结构」的基础——FreeRTOS 的 `List_t`、Linux `list_head` 都是结构体 + 指针的极致运用。

---

## 二、章节概述

1. **struct 声明**:`struct _person { ... };` 与 `typedef struct _person { ... } Person;` 后者更常用。
2. **访问字段**:`person.field`(对象)/ `ptrPerson->field`(指针) / `(*ptrPerson).field`。
3. **结构体内存分配**:padding 由对齐规则决定；`Person` 4 字段各 4 字节 = 16 字节，但 `AlternatePerson` 用 `short age` 仍占 16 字节(尾部 2 字节 padding)。
4. **结构体数组的 padding**:每元素占 sizeof(Struct),即使内部有 padding。
5. **结构体 deallocation**:alloc Person 后，**Person.firstName 等指针字段需要单独 free**;只 `free(person)` 不够。
6. **`initializePerson` / `deallocatePerson` 函数**:手动管理 + 配对调用，易漏。
7. **`Person *ptrPerson; ptrPerson = malloc(sizeof(Person));` 与栈分配的区别**:栈上变量不需要 free,但其指针字段仍需要单独 free。
8. **避免 malloc/free 开销**:对象池(`Person *list[LIST_SIZE];` + `getPerson`/`returnPerson`);只 malloc 一次，用 list 复用。
9. **数据结构四件套**:Linked List、Queue、Stack、Tree——都基于指针 + 结构体。
10. **单链表**:`Node { void *data; Node *next; }`,head/tail/current;addHead、addTail、delete、getNode、displayLinkedList。
11. **比较函数与显示函数指针**:`typedef int(*COMPARE)(void*, void*); typedef void(*DISPLAY)(void*);` —— 注入运行时行为。
12. **队列**:基于链表；`enqueue` 头插、`dequeue` 尾删；`typedef LinkedList Queue;` 复用。
13. **栈**:基于链表；`push` 头插、`pop` 头删；`typedef LinkedList Stack;`。
14. **二叉搜索树**:Node 有 left/right;`insertNode(TreeNode **root, COMPARE, void *)` —— 用 `TreeNode **` 形参，因为函数内部要改 root 本身。
15. **遍历**:`pre-order` / `in-order` / `post-order`,递归，三个不同顺序调用 display。
16. **BST 性质**:左子树 < 父节点，右子树 > 父节点；in-order 遍历得升序序列。

---

## 三、核心 Takeaways

| # | 是什么 | 为什么 | 解决了什么 | 适用场景 |
|---|---|---|---|---|
| **T1** | `->` 与 `.` 语法糖 | deref 优先级保证 | 访问结构体字段 | struct 操作 |
| **T2** | 结构体内指针字段不会自动管理 | C 没有 RAII | 显式 init/free | 任何含指针字段的 struct |
| **T3** | sizeof 含 padding | 字段对齐 | 与编译器 ABI 兼容 | 跨平台 ABI |
| **T4** | 对象池替代 malloc/free | 实时性 + 减少碎片 | RTOS / 高频分配 | ECU 协议栈 |
| **T5** | 函数指针注入比较/显示 | 算法与数据解耦 | 通用容器 | sort / qsort / map |
| **T6** | 链表 + 指针 = 通用容器 | 任意大小数据可挂在 `void *data` | 任何动态集合 | 任务队列、对象池 |
| **T7** | 二叉搜索树靠 `TreeNode **` 形参 | 函数内部要改 root | 可变根节点 | BST 插入 |
| **T8** | 队列/栈都是单链表的特化 | 都是头插/头删或头插/尾删 | 复用同一份代码 | RTOS 任务调度 |
| **T9** | in-order 遍历 BST 得升序序列 | BST 性质 | 排序输出 | 数据持久化 |
| **T10** | 递归遍历 = 栈帧展开 | C 默认无 trampoline | 实现简洁但有栈深度上限 | 任意深度遍历 |

---

## 四、工程实践视角(领域：嵌入式 / 汽车电子)

### 落地

- **AUTOSAR 配置结构**:`Dcm_<Service>_CfgType { uint16 DidCount; const Dcm_DidConfigType *DidList; }` —— 指针字段指向 ROM 中的常量表，**不需 free**。
- **运行时数据缓冲**:`CanTp_TxBufferType { PduIdType PduId; uint8 *DataPtr; uint16 Length; CanTp_StateType State; }` —— DataPtr 指向外部 buffer,**TxBuffer 自身在 pool**,DataPtr 由外部管理。
- **对象池模式** = AUTOSAR BSW 的核心:`Com_TxPduCfgType[]` 是 static 数组,`Com_TxStateType` 实例从 `Com_PduPool[]` 取 —— 完全对应 ch6 §「Avoiding malloc/free Overhead」。
- **FreeRTOS `xList` / `List_t`**:用 `MiniListItem_t` 与 `List_t` 实现双向链表 + 链表头；直接对应 ch6 单链表的扩展。
- **遍历时调用 DISPLAY 函数指针**:这是「回调」的具体形态,UDS 服务子函数里经常用一个 `Dcm_<Service>_DidHandler` 函数指针数组。

### 误区

- **M1** `free(person)` 但漏 `free(person->firstName)` —— **最常见的结构体泄漏**。
- **M2** 链表 addHead 不更新 `list->tail` —— 队列 dequeue 找不到尾。
- **M3** 二叉树 insertNode 用 `TreeNode *root`(单指针)而非 `TreeNode **` —— 第一次插入时空 root,无法赋值。
- **M4** `void *data` 取出时忘了 cast:`data->name` 编译错。
- **M5** 递归遍历深度过大 —— MCU 默认栈只有几 KB,深链表会 stack overflow。
- **M6** 函数指针比较/显示签名不匹配 —— `COMPARE` 类型声明 `int(*)(void*, void*)`,传入签名不一致会 undefined behavior。

### 初中高工程师视角

- **初中级**:能写 struct + malloc + free;理解 `->` 与 `.` 区别。
- **中级**:能写链表/队列/栈，知道函数指针注入比较/显示。
- **高级**:能设计对象池替代 malloc;在 review 时识别「struct 内部指针字段漏 free」的常见 bug。

---

## 五、AI 时代视角

- **LLM 链表 bug 高频**:LLM 经常写 addHead 时漏更新 tail,或在 delete 时漏处理 head/tail 边界。
- **LLM 二叉树 insertNode 经常用 `TreeNode *`** 而非 `TreeNode **` —— 这是 ch6 的关键坑，需要在提示中显式说明「需要能改 root」。
- **Copilot 提示工程**:「实现一个 BST insert,使用 `TreeNode **` 形参」明确提示比「实现 BST insert」更不容易出错。
- **MISRA-C 强制**:MISRA-C:2012 Rule 9.1「Object identification» 与 ch6 函数指针注入的标识符管理强相关。

---

## 六、实践行动项

1. **[必做]** 复现 ch6 §「Structure Deallocation Issues」—— `initializePerson(&p, ...)` 后只 free(p) 而不 free(p->firstName),用 valgrind 看 leak 数量。落档到附录。
2. **[必做]** 跑 `insertNode(&root, ...)` 与 `insertNode(root, ...)` 两个版本的对比，验证后者无法在空树上插入第一个节点。
3. **[推荐]** 实现一个完整 LinkedList(addHead/addTail/delete/getNode/displayLinkedList),用 void* data + 注入的 DISPLAY/COMPARE 函数指针。
4. **[推荐]** 用 BST in-order 遍历打印一棵树，验证输出是升序。

---

## 七、值得深入思考的问题

1. **Q1**:为什么 `struct _person { char* a; char* b; char* c; uint age; };` 占 16 字节，而 `AlternatePerson` 用 `short age` 仍 16 字节？padding 是按「结构体对齐 = 最大字段对齐」—— 三个指针都是 8 字节对齐，故 short 也按 8 字节对齐排，尾部加 padding。
2. **Q2**:`TreeNode **` 形参必须，而 `Person *` 单指针也常见 —— **统一规则**:若函数需要修改 caller 持有的指针变量，就用 `T **`;否则 `T *`。
3. **Q3**:`typedef LinkedList Queue` / `typedef LinkedList Stack` —— 同一份代码，不同语义。这暴露了 C 的「类型即文档」特性 —— typedef 比注释更可靠。
4. **Q4**:BST insertNode 用了「`while(1)` + break」而不是「`do...while`」—— 为什么？因为初始化 root 检查需要先做，然后再进入循环；do-while 至少执行一次，与「首次检查 root」不匹配。
5. **Q5**:递归遍历与迭代遍历(BFS/DFS 用栈)在嵌入式哪个优先？答：**迭代优先**。MCU 栈深度受限；迭代遍历可手动维护栈结构(数组)而不消耗 call stack。

---

## 附录 · Action #6 复盘 · ch6 结构体字段泄漏与 TreeNode** 必要性

### Case 1 · struct 字段泄漏(ASan 捕获)

本机无 valgrind,改用 **AddressSanitizer 的 LeakSanitizer**(gcc 13 内置):

```sh
$ gcc -O0 -g -Wall -Wextra -fsanitize=address -o ch06_struct_leak_asan ch06_struct_leak.c
$ ASAN_OPTIONS=detect_leaks=1 ./ch06_struct_leak_asan
...
==50207==ERROR: LeakSanitizer: detected memory leaks

Direct leak of 10 byte(s) in 1 object(s) allocated from:
    #0 malloc
    #1 init_person  ch06_struct_leak.c:26   ← p->lastName = malloc(...)
    #2 case_bad_free_struct_only  ch06_struct_leak.c:35  ← free(p)
    ...
Direct leak of 8 byte(s) in 1 object(s) allocated from:
    #1 init_person  ch06_struct_leak.c:28   ← p->title = malloc(...)
Direct leak of 6 byte(s) in 1 object(s) allocated from:
    #1 init_person  ch06_struct_leak.c:24   ← p->firstName = malloc(...)

SUMMARY: AddressSanitizer: 24 byte(s) leaked in 3 allocation(s).
```

**真实观察**:`case_bad_free_struct_only` 只 `free(p)` 但漏了三个内字段,ASan 把每个泄漏的字节数与**调用栈行号**精确标出:

- 6 字节 = `"Peter\0"`(firstName)
- 10 字节 = `"Underwood\0"`(lastName)
- 8 字节 = `"Manager\0"`(title)

**对比 Reese 原书 p.138**:Reese 说「you must remember to call the initialize and deallocate functions」——这是手动管理的痛点。**现代工程的回应**:RAII(C++)、Rust 的 `Drop` trait、Go 的 `defer`,都把这种责任交给语言。

### Case 2 · `TreeNode *` vs `TreeNode **` 形参

```sh
$ ./ch06_tree_root
[bad]  tree_bad = (nil) (仍是 NULL,因为第一次插入失败)
[good] tree_good = 0x64c0627a2310 (成功)
[good] in-order traversal:
  Sally
  Samuel
  Susan
```

**真实观察**:
- **bad 版**:`insertNode_bad(tree_bad, ...)` 第一次调用时，函数内 `root = node` 改了形参副本，**caller 的 `tree_bad` 永远是 NULL**;第二次、第三次插入时仍走 `if (root == NULL)` 分支，循环分配三个孤儿节点。
- **good 版**:`insertNode_good(&tree_good, ...)` 通过 `TreeNode **` 改 caller 的 root,三次插入成功；in-order 遍历输出 **Sally / Samuel / Susan**(BST 升序)。

**踩到的坑**:BST 形参 `TreeNode *` vs `TreeNode **` 是 ch6 整章最反直觉的设计，也是 LLM 写 BST 时最高频的 bug。

### Case 3 · 完整单链表 addHead + delete

```sh
$ ./ch06_linkedlist
Linked List:
  Susan	45
  Sally	28
  Samuel	32

[after delete sally]:

Linked List:
  Susan	45
  Samuel	32
```

**真实观察**:
- 头插顺序 `addHead(samuel) → addHead(sally) → addHead(susan)`,实际序列倒过来 `susan → sally → samuel` —— 这就是 addHead 的特性。
- `delete(sally)` 后剩 `susan → samuel`,符合 `head/tail + getNode + delete` 链路的预期。

**额外观察**:`Employee` 用栈分配(`Employee samuel = {...}`),所以无需 free data —— 这是 ch6 §「Structure Deallocation Issues」的简化场景。生产代码会用 malloc + 单独的 deallocateEmployee。

### 一行 repro(可在你机器上复现)

```sh
cd /home/liyahong/dev/read/notes/understanding-using-c-pointers/code-exercises
gcc -O0 -g -Wall -Wextra -fsanitize=address -o ch06_struct_leak_asan ch06_struct_leak.c && \
  ASAN_OPTIONS=detect_leaks=1 ./ch06_struct_leak_asan
gcc -O0 -g -Wall -Wextra -o ch06_tree_root ch06_tree_root.c && ./ch06_tree_root
gcc -O0 -g -Wall -Wextra -o ch06_linkedlist ch06_linkedlist.c && ./ch06_linkedlist
```

### 剩余未做的事

- 扩展 BST insertNode 为「支持重复 key」版本，验证行为。
- 在 ch06_linkedlist 加上 deallocateEmployee,把 data 字段也动态化，跑 ASan 验证 clean exit。
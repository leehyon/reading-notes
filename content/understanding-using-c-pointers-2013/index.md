---
title: Understanding and Using C Pointers
---

> Richard Reese, O'Reilly 2013, ISBN 978-1-449-34418-4

## 节奏

| 段 | 章 | 主题 | 优先级 | 状态 |
| --- | --- | --- | :---: | :---: |
| **A. 概念地基** | ch1 Introduction | 三种内存、声明、`const` 四种组合、指针算术、`size_t`、`intptr_t`、内存模型 | ⭐⭐⭐⭐⭐ | ✅ 2026-09-01 |
| **B. 动态分配** | ch2 Dynamic Memory Management | malloc、calloc、realloc、free、alloca、VLA、memory leak、dangling pointer、RAII、GC | ⭐⭐⭐⭐⭐ | ✅ 2026-09-01 |
| **C. 与语言构件结合** | ch3 Functions / ch4 Arrays / ch5 Strings | stack frame、传址对比传值、函数指针、jagged array、literal pool | ⭐⭐⭐⭐ | ✅ 2026-09-01 |
| **D. 高级与安全** | ch6 Structures / ch7 Security / ch8 Odds and Ends | 链表、队列、栈、树、缓冲区溢出、port、DMA、endianness、strict aliasing、线程、不透明指针、多态 | ⭐⭐⭐⭐ | ✅ 2026-09-01 |

## 跨章核心问题

1. **地址层对比权限层**：指针行为是否可以拆成「地址 stride + 读写权限」两条独立维度？——ch1 const 矩阵 → ch3 形参保护 → ch7 越界检测。
2. **生命周期与内存分区**：Static、Automatic、Dynamic 三种分区在每一章以什么新形态出现？——ch2 heap → ch3 stack frame → ch6 结构体成员。
3. **实现定义对比未定义行为**：ch1 §「Pointer Behavior」铺的术语，后续每一章都在踩哪条边界？——ch4 数组退化、ch7 double free、ch8 strict aliasing。
4. **回调与多态的同构性**：ch3 函数指针对比 ch8 不透明指针 + `void *` 多态——同一种「间接」机制的不同语言级包装。
5. **C 与 C++ 的接口断层**：ch5 函数指针与字符串、ch8 严格别名对比 C++ 的 `std::function`、`std::string_view`——什么时候该用 C 写，什么时候该用 C++。

## 落档清单

| 笔记 | 路径 | 完成日 |
| --- | --- | :---: |
| **概念图谱** | `knowledge-graph.md` | 2026-09-02 |
| ch1 Introduction | `ch01-introduction.md` | 2026-09-01 |
| ch2 Dynamic Memory Management | `ch02-dynamic-memory.md` | 2026-09-01 |
| ch3 Pointers and Functions | `ch03-functions.md` | 2026-09-01 |
| ch4 Pointers and Arrays | `ch04-arrays.md` | 2026-09-01 |
| ch5 Pointers and Strings | `ch05-strings.md` | 2026-09-01 |
| ch6 Pointers and Structures | `ch06-structures.md` | 2026-09-01 |
| ch7 Security Issues | `ch07-security.md` | 2026-09-01 |
| ch8 Odds and Ends | `ch08-odds-and-ends.md` | 2026-09-01 |

## Action 代码归档目录

`code-exercises/`

| 文件 | ch | 关键结论 |
| --- | --- | --- |
| `ch01_misuse.c` | 1 | case1 SIGSEGV@139,case2 size_t=%zu 在 LP64 下输出 `18446744073709551611`,case3 glibc 13 `malloc(0)` 静默 |
| `ch02_saferfree.c` | 2 | glibc 13 tcache 检测到 double-free 抓出；`safeFree` 第二次调用 OK |
| `ch02_realloc_resize.c` | 2 | shrink 复用,grow 搬移,`realloc(p, 0)` → NULL |
| `ch03_dangling_local.c` | 3 | 返回局部 VLA 地址 → GCC 警告 + segv |
| `ch03_ptr_to_ptr.c` | 3 | `T *` 改不了 caller,`T **` 改得了 |
| `ch04_array_vs_ptr.c` | 4 | `sizeof(vector)=20`、`sizeof(pv)=8`、`sizeof(arr in func)=8` |
| `ch04_2d_stride.c` | 4 | `matrix+1` 跳 20 字节,`&matrix[0][0]+1` 跳 4 字节 |
| `ch05_literal_pool.c` | 5 | 三个 `"hello world"` 同址,`*p='L'` 改字面量 → SIGSEGV |
| `ch05_strcmp_eq.c` | 5 | `command == "Quit"` 永为 false,`strcmp == 0` 正确 |
| `ch05_snprintf_overflow.c` | 5 | GCC `-Wformat-overflow=` 编译期直接拦截 `sprintf` 越界；`snprintf` 安全截断 |
| `ch06_struct_leak.c` | 6 | ASan LeakSanitizer 抓 24 字节 3 个泄漏，行号精确 |
| `ch06_tree_root.c` | 6 | `TreeNode*` 失败,`TreeNode**` 成功；BST in-order = 升序 |
| `ch06_linkedlist.c` | 6 | addHead + delete 完整 demo,函数指针注入 |
| `ch07_deref_misuse.c` | 7 | `*pbad = num` 在本机栈布局下未 segv(garbage 碰巧可写) |
| `ch07_sizeof_misuse.c` | 7 | 循环跑过头 → GCC stack protector `*** stack smashing detected ***` |
| `ch07_fptr_paren.c` | 7 | GCC `-Waddress` 两条警告,if/else 行为错 |
| `ch08_endian.c` | 8 | x86-64 little-endian 实测 |
| `ch08_union_pun.c` | 8 | `3.14f` = `0x4048f5c3`,union vs 指针 cast strict-aliasing |
| `ch08_opaque.c` | 8 | 不透明指针封装 LinkedList |

## 一键复现

```sh
cd /home/liyahong/dev/read/notes/understanding-using-c-pointers/code-exercises
for c in ch01_misuse ch02_saferfree ch02_realloc_resize ch03_dangling_local ch03_ptr_to_ptr \
         ch04_array_vs_ptr ch04_2d_stride ch05_literal_pool ch05_strcmp_eq ch05_snprintf_overflow \
         ch06_tree_root ch06_linkedlist ch07_deref_misuse ch07_sizeof_misuse ch07_fptr_paren \
         ch08_endian ch08_union_pun ch08_opaque; do
    gcc -O0 -g -Wall -Wextra -o $c $c.c && echo "=== $c ===" && timeout 5 ./$c 2>&1 || true
done
```
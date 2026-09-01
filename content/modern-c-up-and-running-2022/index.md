---
title: Modern C
---

**书名**: Modern C: Up and Running — A Programmer's Guide to Finding Fluency and Bypassing the Quirks
**作者**: Martin Kalin (DePaul University, Ph.D. Northwestern)
**出版社**: Apress, 2022
**ISBN**: 978-1-4842-8675-3 (pbk) / 978-1-4842-8676-0 (electronic)
**原文路径**: `dev/books/Modern_C_Up_and_Running_2022.pdf`
**技术审校**: Germán González-Morris

## 阅读约定

每章按六段式模板输出:
1. 章节概述 (5~10 条要点)
2. 核心 Takeaways (是什么 / 为什么 / 解决什么 / 适用场景)
3. 工程实践视角 (**重点 Linux 系统 + 机器人软件** —— 与 Effective C 错位侧重)
4. AI 时代视角 (知识时效 / AI 能/不能做什么 / wasm 工具链延伸)
5. 实践行动项 (3~5 个可执行行动)
6. 值得深入思考的问题 (3~5 个高质量问题)

## 全书结构 (8 章 / 341 页正文 / 371 页 PDF)

- ch1 **Program Structure** —— 函数/声明定义/main/汇编/控制流/变长参数
- ch2 **Basic Data Types** —— int/float/IEEE 754/运算符
- ch3 **Aggregates and Pointers** —— 数组/void\*/结构体/堆存储/内存泄漏
- ch4 **Storage Classes** —— auto/register/static/extern/volatile
- ch5 **Input and Output** —— 系统级 I/O/缓冲/非阻塞/命名管道
- ch6 **Networking** —— HTTP client/event-driven server/OpenSSL TLS
- ch7 **Concurrency and Parallelism** —— fork/exec/共享内存/文件锁/消息队列/pthread/SIMD
- ch8 **Miscellaneous Topics** —— regex/assert/i18n/WebAssembly/信号/动态库

## 与 Effective C (Seacord 2020) 的关系

- 相同：8 章主题（ch1 vs ch1-2 程序结构 + 类型 + 表达式）几乎一一对应
- 差异：Kalin 偏 Apress 工程实操 + 网络/并发/wasm；Seacord 偏 CERT 安全 + 算术/控制流陷阱
- 本笔记 `modern-c-2022/` 与 `effective-c-2020/` 各自独立，不做交叉章

## 章节索引

| 章节 | 标题 | PDF 页 | 笔记 |
|---|---|---|---|
| 0 | **整书知识体系总图** | — | ⭐ [00-knowledge-graph.md](00-knowledge-graph.md) |
| 1 | Program Structure | 18-49 (32p) | ✅ [ch01-program-structure.md](ch01-program-structure.md) |
| 2 | Basic Data Types | 50-82 (33p) | ✅ [ch02-basic-data-types.md](ch02-basic-data-types.md) |
| 3 | Aggregates and Pointers | 83-150 (68p) | ✅ [ch03-aggregates-pointers.md](ch03-aggregates-pointers.md) |
| 4 | Storage Classes | 151-166 (16p) | ✅ [ch04-storage-classes.md](ch04-storage-classes.md) |
| 5 | Input and Output | 167-204 (38p) | ✅ [ch05-input-output.md](ch05-input-output.md) |
| 6 | Networking | 205-245 (41p) | ✅ [ch06-networking.md](ch06-networking.md) |
| 7 | Concurrency and Parallelism | 246-304 (59p) | ✅ [ch07-concurrency-parallelism.md](ch07-concurrency-parallelism.md) |
| 8 | Miscellaneous Topics | 305-358 (54p) | ✅ [ch08-miscellaneous.md](ch08-miscellaneous.md) |


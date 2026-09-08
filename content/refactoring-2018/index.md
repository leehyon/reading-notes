---
title: Refactoring
---

> Martin Fowler, *Refactoring: Improving the Design of Existing Code*, 2/e (2018), Addison-Wesley。

---

## 节奏
| 段 | 章 | 主题 | 优先级 |
|---|---|---|---|
| A. 入门 & 原理 | 1–2 | First Example + Principles | ⭐⭐⭐⭐⭐ |
| B. Smell & Catalog 入口 | 3–5 | Bad Smells + Tests + Catalog 索引 | ⭐⭐⭐⭐⭐ |
| C. Catalog 第一线(常用 refactoring) | 6–11 | First Set / Encapsulation / Moving / Data / Conditional / API | ⭐⭐⭐⭐⭐ |
| D. Inheritance 进阶 | 12 | Dealing with Inheritance | ⭐⭐⭐⭐ |

---

## 12 章 × 文件对照表

| 章 | 标题 | 文件 | 字节 |
|---|---|---|---|
| 1 | Refactoring: A First Example | `ch01-refactoring-first-example.md` | 7.2 KB |
| 2 | Principles in Refactoring | `ch02-principles-in-refactoring.md` | 7.1 KB |
| 3 | Bad Smells in Code | `ch03-bad-smells-in-code.md` | 8.7 KB |
| 4 | Building Tests | `ch04-building-tests.md` | 5.4 KB |
| 5 | Introducing the Catalog | `ch05-introducing-the-catalog.md` | 5.0 KB |
| 6 | A First Set of Refactorings | `ch06-first-set-of-refactorings.md` | 8.0 KB |
| 7 | Encapsulation | `ch07-encapsulation.md` | 7.8 KB |
| 8 | Moving Features | `ch08-moving-features.md` | 7.8 KB |
| 9 | Organizing Data | `ch09-organizing-data.md` | 7.6 KB |
| 10 | Simplifying Conditional Logic | `ch10-simplifying-conditional-logic.md` | 8.9 KB |
| 11 | Refactoring APIs | `ch11-refactoring-apis.md` | 9.0 KB |
| 12 | Dealing with Inheritance | `ch12-dealing-with-inheritance.md` | 9.4 KB |
| — | 索引 | `refactoring-2018-index.md`(本文) | — |

**总统计**: 12 章 / 12 文件 / ~92 KB / ~2700 行（章节笔记 + 索引）。

---

## 跨章核心问题

1. **「refactoring 的 purpose 是 behavior preservation」**(chapter 2 公理) — refactor 与 bug fix / feature add 严格分离，这条不变量贯穿全书。
2. **「smell 是诊断,refactor 是处方」**(chapter 3) — 24 个 smell × catalog 重构条目是全书的工作流闭环。
3. **「小步 + 测试永远绿 = refactoring 的 rhythm」**(chapter 1) — 整个 catalog 的 mechanics 都建立在 atomic commit + atomic test 上。
4. **「Extract / Inline / Move / Rename = 80% 日常 refactor」**(chapter 6) — Fowler 的入门组。
5. **「encapsulation 是 access path 设计，不只是 private」**(chapter 7) — record / collection / primitive / temp 都要各自的封装策略。
6. **「code ownership 是 design 的根本」**(chapter 8) — Move / Pull Up / Push Down 改的是设计，不是表面重构。
7. **「data structure 形态本身是 refactoring 的对象」**(chapter 9) — primitive ↔ class, mutable ↔ immutable, reference ↔ value 都是 transform。
8. **「conditional logic 是程序复杂度的主要来源」**(chapter 10) — Decompose Conditional、Guard Clauses、Polymorphism、Special Case 四件套消灭 conditional。
9. **「API 是 contract,refactoring API 必须 migrate」**(chapter 11) — Query/Command Separation / Remove Flag Argument / Preserve Whole Object。
10. **「inheritance 是 design 中成本最高的工具」**(chapter 12) — composition / delegate / strategy 经常比 inheritance 更优。
11. **「rule of three 决定抽象时机」**(chapter 2 + 6) — 1~2 次重复容忍,3 次抽 — XP 社区共识。
12. **「comments 通常是 smell」**(chapter 3) — 注释经常意味着「这段 code 没抽干净」。
13. **「测试是 refactoring 的前置保险丝」**(chapter 4) — 没有测试 = refactor 是「闭眼跳悬崖」。
14. **「Speculative Generality 是 over-engineering 的根源」**(chapter 3 + 12) — 为「将来」预留的 hook 是 smell。
15. **「Replace Conditional with Polymorphism vs Replace Type Code with Subclasses」** — 两者等价，前者 design pattern,后者 type system。
16. **「Favor composition over inheritance」**(chapter 12) — GoF / Effective Java 共识。
17. **「Preparatory Refactoring 让 refactor 与 feature 同步」**(chapter 2) — refactor 不另占 sprint velocity。
18. **「when NOT to refactor」** — Fowler 强调的负向 — change probability = 0 的代码不 refactor。

---

## 历史主线

| 年 | 事件 | 章节定位 |
|---|---|---|
| 1990 | Smalltalk 社区开始 refactoring 实践 | chapter 2 |
| 1992 | Bill Opdyke 博士论文 — refactoring frameworks | chapter 2 |
| 1999 | Fowler 第一版 *Refactoring* | chapter 2 |
| 2002 | Kent Beck *Test-Driven Development by Example* | chapter 4 |
| 2004 | Michael Feathers *Working Effectively with Legacy Code* | chapter 4 |
| 2008 | Fowler 在 OOPSLA talk 重申 refactoring 价值 | chapter 2 |
| 2010 | Functional Programming 复兴(immutable value 重回主流) | chapter 9 |
| 2014 | Liskov 可替换性的现代解读(Erik Ernst 等) | chapter 12 |
| 2018 | Fowler 第二版，新增 30+ refactoring 条目 + JS examples | 全书 |

---

## Smell × Refactoring 矩阵

| Smell | 推荐 Refactoring |
|---|---|
| Mysterious Name | Change Function Declaration / Rename Variable / Rename Field |
| Duplicated Code | Extract Function / Slide Statements / Pull Up Method |
| Long Function | Extract Function (99% of the time) / Replace Temp with Query / Introduce Parameter Object |
| Long Parameter List | Replace Parameter with Query / Preserve Whole Object / Introduce Parameter Object |
| Global Data | Encapsulate Variable |
| Mutable Data | Encapsulate Variable / Split Variable / Change Reference to Value |
| Divergent Change | Split Phase / Extract Class |
| Shotgun Surgery | Move Function / Move Field / Combine Functions into Class |
| Feature Envy | Move Function |
| Data Clumps | Extract Class / Introduce Parameter Object / Preserve Whole Object |
| Primitive Obsession | Replace Primitive with Object / Replace Type Code with Subclasses |
| Repeated Switch | Replace Conditional with Polymorphism |
| Lazy Element | Inline Function / Inline Class / Collapse Hierarchy |
| Speculative Generality | Collapse Hierarchy / Inline Function / Inline Class / Remove Dead Code |
| Comments | Extract Function / Rename Variable / Introduce Assertion |
| Long Conditional | Decompose Conditional / Replace Nested Conditional with Guard Clauses / Replace Conditional with Polymorphism |
| Refused Bequest | Push Down Method / Push Down Field / Replace Subclass with Delegate |
| Message Chains | Hide Delegate |
| Middle Man | Remove Middle Man / Inline Class |
| Data Class | Encapsulate Record / Remove Setting Method |

---

## 工程实践视角 — NeuSAR cCore V3.0 落档

每章笔记都在 §四「工程实践视角」落地到 NeuSAR cCore V3.0 的对应模块。**全书的 actionable patterns**:

1. **Cantp**: Extract Function 重构 state machine、Encapsulate Variable 封装 RxBufferPool、Introduce Parameter Object 把流控参数(N_TA, N_BS, N_CR, N_AS)打包成 `CanTpTiming`。
2. **Dcm**: Split Phase 把 service handler 拆「解析 request → 调 callback → 编码 response」、Replace Type Code with Subclasses 让 `Dcm_ServiceHandler` 成为 base class + 各 service subclass。
3. **CanIf**: Encapsulate Variable 隐藏 ChannelConfig 内部 mutable state、Move Function 让 channel-specific 行为归属 Channel。
4. **PduR**: Replace Conditional with Polymorphism 处理 routing type 决策(Direct / TP / Gateway)、Move Function 把 routing-specific 行为归属 RoutingPath。
5. **ComM / CanSm**: Encapsulate Variable 封装 network state、Introduce Special Case 处理「未连接」特殊情况。
6. **RTE**: Extract Class 拆出 Rte_PortGroup、Preserve Whole Object 在 RTE API 替代解构参数。

---

## AI 时代视角——跨章汇总

每章 §五「AI 时代视角」已分析。本书内容在 AI 时代的整体定位:

- **变**：mechanics 部分(具体 refactor 步骤 / boilerplate)被 AI 自动化 — Extract Function / Rename / Encapsulate Variable AI 都能做。
- **不变**：判断框架(why + when + 是否 refactor) + smell × refactoring 矩阵选择 + Liskov / design 决策——这些是 AI 的瓶颈。
- **新机会**：
  - 大仓 smell 扫描(SonarQube + LLM)。
  - mechanics 自动生成 + 本项目 context 化。
  - migration 路径规划(API refactor / Replace Subclass with Delegate)。
  - Liskov violation 检测 + 测试生成。

# Refactoring 2/e · Chapter 2 — Principles in Refactoring

> 来源:Martin Fowler, *Refactoring: Improving the Design of Existing Code*, 2/e (2018), Chapter 2.
> 章节定位:chapter 1 用 example 建立体感，本章抽象出原理 — refactoring 的定义(noun/verb)、why、when、how(与性能/架构的关系)。
> 模板裁剪：技术书，全 7 节保留。

---

## 一、第一性原理思考

Fowler 这一章在做一件事:**把 refactoring 这个被滥用的词，重新 fix 到「noun=不改行为的内部结构改动、verb=做这种改动」上**。然后他做了一件反直觉的事:**先讲为什么不要 refactor**。

**公理 1(Refactoring 定义)**:noun:不改变软件可观察行为的内部结构调整;verb:做这个动作。**两边都不是「rebuild」「rewrite」「port」**,任何包含 bug fix 或新 feature 的都不算 refactoring。

**公理 2(Why 与 The Two Hats)**:Kent Beck 的「Two Hats」— programmer 一会儿戴 refactoring hat,一会儿戴 feature hat。**戴上 refactoring hat 的标志是「既不增加功能也不修复 bug」**。

**公理 3(why refactor)**:
- 改进软件设计(防止 rigidity / fragility / immobility — 是 Robert Martin 的 3 大 design smell)
- 让软件更易理解
- 帮我找 bug
- 帮我写得更快

**公理 4(when not to refactor)**:**如果代码永远不需要改，就别动它**。refactor 本身有 cost(时间 + 引入新 bug 的可能性),**只在「change 概率 > 0」的代码上才有 ROI**。

**假设 vs 结论**:
- 假设：读者已经写过一段时间代码，对「代码必须好看」有朴素直觉，但没有 quantifier
- 结论:refactor 的 ROI 用「未来 change 概率」计,**beauty 不是 value, changeability 才是**

---

## 二、章节概述

1. **Defining Refactoring**:noun/verb 双定义,excludes「fix bug」「add feature」。
2. **Why Should You Refactor?**:4 大动机——改进设计 / 易理解 / 找 bug / 更快编程。**第 4 个最反直觉**。
3. **When Should You Refactor?**:三次法则(rule of three)、preparatory refactoring、comprehension refactoring、litter-pickup refactoring、planned vs opportunistic。
4. **Problems with Refactoring**:refactor 不变 behavior 这件事,**没有编译器能 enforce**(虽然类型系统近似)。**它要求大量 testing 或 类型系统的纪律**。
5. **Refactoring and Design**:与「先 design 后 code」流派对冲 — Fowler 主张**渐进 design,through refactoring**。
6. **Refactoring and Performance**:与「performance first」流派对冲——先易读再调性能(profile first, refactor second)。
7. **Where Did This Come From?**:历史 — 1990s Smalltalk 社区,Bill Opdyke、John Brant、Don Roberts 等。
8. **Further Reading**:推荐书目 — FOWLER 第一版、Feathers Legacy Code、Tufte 排版、Ruby Refactoring 等。

---

## 三、核心 Takeaways

### Takeaway 1 — 「Refactoring ≠ rewriting + 修 bug + 加 feature」
- **是什么**:严格的「internal structure change only, behavior unchanged」。
- **为什么重要**:**行为不变**这个约束让 refactoring 可以小步、可以分 commit、可以 git revert。**混入修 bug 之后，你无法区分「行为改变是 refactor 引入的 bug」还是「修掉的 bug」**。
- **解决了什么问题**:让 refactor 与 feature 工作流彻底分离，在 git log 里两类 commit 泾渭分明。
- **适用场景**:code review 时判断 — 「这个 PR 里有没有偷偷 fix bug?」

### Takeaway 2 — 「why refactor = 写代码更快」
- **是什么**:Fowler 第 4 个动机「Help Me Program Faster」最反直觉——不是「让代码好看」,而是「让你更快写下一个 feature」。
- **为什么重要**:它把 refactor 从「审美的洁癖」变成「工程师的工程纪律」。**refactor 的 ROI 是「下一波 feature 节省的时间 - refactor 本身的时间」**。
- **解决了什么问题**:反驳「我们没时间 refactor」的常见 managers 反论。**答：你没时间不 refactor,导致下一波 feature 慢 2 倍**。
- **适用场景**:任何 sprint planning 讨论。

### Takeaway 3 — 「Rule of Three」
- **是什么**:**第一次写代码做一件事；第二次做相似的事时容忍 duplication;第三次出现相似的事时必须 refactor**。
- **为什么重要**:**前两次抽象是猜测(可能抽象错了),第三次有了足够 sample 才能看到真正的共同点**。这是 XP 社区的经典 heuristic。
- **解决了什么问题**:「过早抽象」造成的 over-engineering。
- **适用场景**:代码 review 时 — 「这是第几次? 1 次就别抽 helper,3 次再抽」。

### Takeaway 4 — 「Preparatory Refactoring」
- **是什么**:加 feature 之前先 refactor — 让新 feature 有一个「合适的着陆点」。**用新 feature 本身的需求作为 refactor 的 driver**。
- **为什么重要**:它把 refactor 的 cost 算到「加 feature」头上，而不是「重构专项时间」头上——经理最爱听。
- **解决了什么问题**:「refactor 不算 sprint velocity」的政治问题。
- **适用场景**:BSW 新增 UDS service 时，先把现有 Dcm 状态机分 phase 再加新 service。

### Takeaway 5 — 「Refactoring and Performance = 先易读再 perf」
- **是什么**:**性能优化的第一刀应该是 profile,不是猜测**。代码先写得易读,profile 后再针对 hotspot 改——此时通常会发现 90% 时间在 10% 代码,**这 10% 之外的 readability 比 perf 重要**。
- **为什么重要**:反驳「必须先写高效代码」的低层教条。Fowler 引用 Knuth 的「premature optimization is the root of all evil」作为盟友。
- **解决了什么问题**:避免嵌入式 / 协议栈代码「为了快 5% 牺牲可读性」的常见误判。
- **适用场景**:Cantp 分帧逻辑、CAN 收发中断、Dcm 0x27 安全访问时序。

---

## 四、工程实践视角

### 如何落地
- **commit message 模板**:`[refactor] Extract Function amountFor from statement (#123)` — `[refactor]` tag 让 git log 自动分类。
- **CI 加 behavior-snapshot test**:不靠 100% 覆盖,**靠关键路径的 snapshot test**(golden file)检测「refactor 偷偷改了行为」。
- **sprint velocity = feature + refactor 各占一部分**(Fowler 引用 Microsoft Research 的实证——持续 refactor 的团队 feature velocity 高 2x)。

### 常见误区(初级工程师)
- **「refactor 就是加注释」**:注释不解决长函数问题,**Extract Function 才是答案**。注释只能解释 why,不能解决 what 太长。
- **「refactor = 改完可以 working 即可」**:**「working」不等于「behavior unchanged」**。snap test 是唯一可信判据。
- **「重构时机 = 看不顺眼就改」**:**Fowler 强调「when not to refactor」** — 如果代码已 stable 且不会改，留它。

### 高级工程师更关注
- **Refactor vs Rewrite 的判据**:**看测试覆盖率** — Feathers 在《Working Effectively with Legacy Code》(本书 further reading)中给 hard rule,**测试覆盖 < 80% 的 legacy code 一律先 refactor 加测试，不要直接 rewrite**(因为 rewrite 没测试保护，容易从「还能跑」变成「跑不起来」)。
- **类型系统的支持**:TypeScript / Rust / Kotlin 的强类型在 refactoring 时能 capture 大量「改错了」的错误,**这是强类型给工程师的最大红利**(初学者常低估)。
- **Preparatory Refactoring 的政治化**:把它包在「加 feature 的 story」里，而不是「重构专项」。

### 与 NeuSAR cCore V3.0 的潜在连接
- **AUTOSAR 模块的 refactor 边界**:BSW 每个 module 有生成代码 + 手写代码两层,**refactor 手写代码时不能让 generated code 漂移**。这就是 refactor 的「behavior unchanged」约束在 AUTOSAR 语境的具体含义 — arxml 是 single source of truth。
- **Preparatory Refactoring 在 BSW**:加 UDS 0x34/0x36/0x37(数据传输)service 时,**先重构 Dcm 状态机把「download 阶段」独立成 sub-state machine**,再加 service。比直接加 switch case 干净 10 倍。
- **Rule of Three 在 RTE**:RTE 的 port 数量、interface 数量,**3 个相似 port 出现时立刻考虑 extract common interface**,但 1~2 个 port 时不要强行抽象(因为 AUTOSAR 工具链对 abstraction 有开销)。

---

## 五、AI 时代视角

- **本章内容今天仍然重要吗**:**why + when + how 不变**。AI 改的是 mechanics(章节 3-12 的具体 refactoring 步骤),不变的是这套判断框架。
- **AI 能够帮助什么**:
  - **判断「要不要 refactor」**:可以基于 git history + test coverage + change frequency 给出 heuristic 报告。
  - **Preparatory Refactoring 路径规划**:给定「要加 feature X」,AI 能预先给出 refactor 顺序建议。
- **AI 无法替代什么**:
  - **when NOT to refactor** — 这需要「未来 6 个月的产品 roadmap」,AI 无法判断。
  - **「行为不变」的最终判据** — golden file test 必须是人类写,AI 写的 snap test 可能 miss 边界 case。
- **工程师必须掌握的核心能力**:
  - **判断 refactor ROI 的能力** — 在 sprint planning 中能给出数字(refactor 4 小时，省未来 3 个 feature 每个 2 小时 = ROI 50%)。
  - **识别「过度 refactor」的能力** — 「rule of three 还没触发」就别抽象。

---

## 六、实践行动项

1. **为最近一次 refactor 标 `[refactor]` tag + 写 1 段 commit message 解释「为什么 refactor」(引用 chapter 2 的 4 大动机之一)**。
2. **找一处 legacy code 应用 Preparatory Refactoring**:加一个新 feature 前，先加一行 Extract Function 让 add feature 有 clean 落点。
3. **跑 chapter 1 的 statement 例子的 performance profile,refactor 完后看 90/10 法则是否真的成立**。

---

## 七、值得深入思考的问题

1. **「refactoring 不改变行为」与「连续小步 + 测试永远绿」组合下，单步可以被验证；但 50 步累加如何保证没漂移?** Fowler 没说，但实际答案是 — **每步 git revert**, 留 git log 永久可查。
2. **「rule of three」与「DRY(Don't Repeat Yourself)」冲突吗?** DRY 是「永远不重复」;rule of three 是「前两次容忍」。**问自己：你认为 DRY 是不是 over-engineering 的根源之一?**
3. **「refactor 不算 sprint velocity」** — 这个组织病应该怎么治?Microsoft Research 给的实证 vs 多数 manager 的直觉,**哪边赢?**
4. **Preparatory Refactoring 与「先 design 后 code」流派谁更优?** — Fowler 与 TDD 同一阵营，主打 emergent design。**问自己：嵌入式固件这种「design 一次,10 年不改」的场景,emergent design 还有意义吗?**

---

## 交叉引用

- 第 1 章 First Example → 本章抽象出 chapter 1 的具象
- 第 3 章 Bad Smells → 何时 refactor 的具体 signal
- 第 4 章 Building Tests → refactoring 的前置保险丝
- 第 5 章 Catalog → 怎么读后面的 catalog 条目
- Feathers *Working Effectively with Legacy Code* → 本章 further reading 列出,Refactor + Legacy Code 双生书

---

## 附录 · Action #N 复盘

> 留待用户在本地执行时补充。

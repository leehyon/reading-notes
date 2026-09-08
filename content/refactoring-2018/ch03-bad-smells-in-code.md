# Refactoring 2/e · Chapter 3 — Bad Smells in Code

> 来源:Martin Fowler, *Refactoring 2/e* (2018), Chapter 3(与 Kent Beck 合著)。
> 章节定位：全书最经典的目录之一——把「代码哪里坏」命名为 24 个 bad smell,每个 smell 指向后面的 catalog 重构条目。**smell 是诊断,refactoring 是处方**。
> 模板裁剪：技术书，全 7 节保留。

---

## 一、第一性原理思考

**Fowler & Beck 的核心论证**:不是「代码丑所以要 refactor」,而是「某种 code structure 模式反复出现，意味着行为和结构在错位」 — 这些结构是「诊断指标」。**没有精确 trigger,只有 informed intuition**(Fowler 原话),所以这一章是启发式的，不是教条。

**公理**:**smell 是一种 code pattern,出现概率越高，意味着越需要 refactor**。但**不必然** — 也有「smell 但不该 refactor」(代码不再被改)。**判断 ROI 的是 change frequency × smell severity**。

**假设 vs 结论**:
- 假设:refactoring 是品味问题，无法工程化
- 结论:**24 个 smell 名字 + 后面的 catalog 条目 = 把品味变成可教学的 heuristics**

---

## 二、章节概述

1. **Mysterious Name** — 命名不清。重命名三件套:Change Function Declaration (124) / Rename Variable (137) / Rename Field (244)。
2. **Duplicated Code** — 重复。Extract Function / Slide Statements / Pull Up Method。
3. **Long Function** — 长函数。**99% 的情况下 Extract Function (106)** 解决。Replace Temp with Query (178) + Introduce Parameter Object (140) 处理 temp 和 param。Replace Function with Command (337) 是重型武器。
4. **Long Parameter List** — 长参数列表。Replace Parameter with Query (324) / Preserve Whole Object (319) / Introduce Parameter Object (140) / Combine Functions into Class (144)。
5. **Global Data** — 全局可变数据。**Encapsulate Variable (132) 第一反应**。
6. **Mutable Data** — 可变数据。Encapsulate Variable + 把变量封装在函数里。
7. **Divergent Change** — 一个模块因不同原因被改。**Split Phase (154) / Split Module / Extract Function / Extract Class**。
8. **Shotgun Surgery** — 改一个需求要改 N 个类。Move Function (198) / Move Field (207) / Combine Functions into Class (144)。
9. **Feature Envy** — 函数对别的类的数据比对本类还亲。Move Function (198)。
10. **Data Clumps** — 几个数据总是成群出现。Extract Class (182) / Introduce Parameter Object (140) / Preserve Whole Object (319)。
11. **Primitive Obsession** — 用 primitive type 表示领域概念。Replace Primitive with Object (174) / Replace Type Code with Subclasses (362) / Replace Conditional with Polymorphism (272)。
12. **Repeated Switch** — 多处 switch 在同一 type code 上。Replace Conditional with Polymorphism (272)。
13. **Loops** — 不用管道。Replace Loop with Pipeline (231)。
14. **Lazy Element** — 没什么用的小结构。Inline Function (115) / Inline Class (186) / Collapse Hierarchy (380)。
15. **Speculative Generality** — 为「将来可能」而写的 hook。Collapse Hierarchy / Inline Function / Inline Class / Remove Dead Code (237)。
16. **Temporary Field** — 只在某阶段被赋值的字段。Extract Class (182) / Move Function (198) / Introduce Special Case (289)。
17. **Message Chains** — 一串 a.b().c().d()。Hide Delegate (189) / Extract Function / Move Function。
18. **Middle Man** — 类大半方法都是委托给别的类。Remove Middle Man (192) / Inline Class (186) / Replace Delegation with Inheritance (351)。
19. **Insider Trading** — 模块互相摸太多内部。Move Function (198) / Move Field (207) / Hide Delegate。
20. **Large Class** — 类太大。Extract Class (182) / Extract Superclass (375) / Replace Type Code with Subclasses (362)。
21. **Alternative Classes with Different Interfaces** — 两个类做相似的事。Rename Method / Move Function / Extract Superclass。
22. **Data Class** — 只有字段没有行为的类。Encapsulate Record / Remove Setting Method / Move Function / Extract Function。
23. **Refused Bequest** — 子类不用父类的遗产。Push Down Method (359) / Push Down Field (361) / Replace Subclass with Delegate (381)。
24. **Comments** — 注释通常是 smell。**因为注释通常在说「这段代码可以更清楚」**。Extract Function / Rename Variable / Introduce Assertion (302)。

---

## 三、核心 Takeaways

### Takeaway 1 — 「smell 是诊断,refactoring 是处方」
- **是什么**:每个 smell 名字对应一个或多个 catalog 重构条目。**目录最后一页有完整 smell × refactoring 矩阵**。
- **为什么重要**:**脱离 smell 谈 refactoring 是空中楼阁**。refactor 的入口永远是「这里有问题」,smell 是把问题结构化的方法。
- **解决了什么问题**:refactor 「why」与「how」脱钩的常见误区。
- **适用场景**:code review checklist 用 smell 名分类 — 「这处有 Long Function smell,需要 Extract Function」。

### Takeaway 2 — 「Comments 通常是 smell」
- **是什么**:**Fowler 反直觉指出注释经常意味着「这段 code 没抽干净」**。注释的本质 = Extract Function 的 description,直接抽出 function 命名即可。
- **为什么重要**:它是 chapter 3 的反转——大部分工程师以为注释 = good practice,实际上**注释是 refactor 没做完的残骸**。
- **解决了什么问题**:「为啥要 refactor 已经清楚的代码」 — 因为 comment 本身在喊「我应该被抽走」。
- **适用场景**:code review 中看到大段解释 what 的注释，立刻嗅 Long Function smell。

### Takeaway 3 — 「Global Data 是最毒的 smell 之一」
- **是什么**:可被任意修改的全局可变数据——没人知道谁动了它,bug 出在 spooky action at a distance。**Encapsulate Variable (132) 是 first move**。
- **为什么重要**:现代 OOP / 模块化语言给了我们工具,**小剂量 global data 可以容忍，但大剂量一定毒死**。
- **解决了什么问题**:嵌入式 firmware 里常见的「全局 flag 控制一切」的 anti-pattern。
- **适用场景**:任何 singleton / 全局 state / 配置文件裸用。

### Takeaway 4 — 「Feature Envy vs Divergent Change vs Shotgun Surgery 三件套」
- **是什么**:
  - **Feature Envy**:一个 function 喜欢别的 class 的数据(应该 Move Function)。
  - **Divergent Change**:一个 class 因为不同原因被改(应该 Split Phase / Split Module)。
  - **Shotgun Surgery**:改一个 feature 要改 N 个 class(应该 Move Function / Combine Functions into Class)。
- **为什么重要**:**三者是「职责放错位置」的三种表现**,混淆就开错药。
- **解决了什么问题**:BSW 模块拆分时常见的判断 —「Dcm 这一坨该移到 PduR 还是 Cantp?」
- **适用场景**:模块边界重构时按三件套诊断。

---

## 四、工程实践视角

### 如何落地
- **smell × refactoring 矩阵印在桌上** — chapter 12 末页有完整表格。**code review 时：闻 smell → 查表 → 选 refactoring → 给同事一句话解释**。
- **CI 集成 smell 工具** — SonarQube / CodeClimate / ESLint(`complexity`、`max-lines-per-function`、`no-duplicate-imports`)。**但不要 100% 信任** — SonarQube 的 threshold 是 noise 偏高的启发式。
- **「Comments 通常是 smell」的反用** — 自己写代码时,**先写注释，再把注释替换成 Extract Function**。这是 TDD-style 的 refactoring 习惯。

### 常见误区(初级工程师)
- **「smell 出现就 refactor」**:**Fowler 强调「when not to refactor」**。smell 是必要条件不是充分条件。
- **「Long Function 就是行数多」**:**关键是「function 名 vs body 的 semantic distance」**,10 行也可能太长(全是 magic number),5 行也可能合适(名字完全说明意图)。
- **「Polymorphism 解决所有 switch」**:**Repeated Switch smell 是 trigger,但 2 处 switch 用 polymorphism 过度，直接 Extract Function 更轻**(Fowler 的 pragmatism)。

### 高级工程师更关注
- **smell × refactoring 矩阵的优先级**:团队应该挑 3~5 个 smell 重点治理(根据 code review 历史 top smell),而不是追求覆盖全部 24 个。
- **「Speculative Generality」**反着读 — **YAGNI 是 smell 的防御**,**不要为了「将来」写 hook**。
- **「Temporary Field」在 distributed system 里特别毒** — 异步上下文中「只在某个阶段有效」的字段是 race condition 温床,**Introduce Special Case + Encapsulate** 才能根治。

### 与 NeuSAR cCore V3.0 的潜在连接
- **BSW 的 Global Data smell**:`Dem_*` 诊断事件管理模块、`NvM_*` 非易失存储管理大量全局数组。**Encapsulate Variable 在 MCAL 层最难做**,因为中断上下文无法 OOP 化,**但 ComM / CanSm 这种 high-level module 应该做**。
- **Feature Envy / Shotgun Surgery 在 RTE/BSW 集成** — 当改一个 UDS service 改动散布在 Dcm + PduR + CanTp + CanIf 4 个 module,典型 shotgun surgery,需要 Move Function 或 Extract Interface 把相关逻辑集中到一个 RTE port。
- **Primitive Obsession in BSW** — PDU ID / CAN ID / Priority 在配置工具里就是 number,**应该 Replace Primitive with Object**(在工具链里表现为 enum + validator,而非裸 uint16)。

---

## 五、AI 时代视角

- **本章内容今天仍然重要吗**:**100% 重要**。AI 时代的 smell detection 工具都基于这个 list,没有新 smell。
- **AI 能够帮助什么**:
  - **大仓 smell 扫描** — 传统 SonarQube 跑一周,LLM 加 AST embedding 1 小时出 top smell list。
  - **smell 解释** — 给 smell name 配上本模块/本项目的具体解释(LLM 比抽象描述更有效)。
  - **book 内 smell × refactoring 矩阵自动生成** — 给定 10k LOC,LLM 生成 (file, smell, suggested refactoring) 三元组。
- **AI 无法替代什么**:
  - **「smell 是否值得 refactor」的判断** — 取决于该模块未来 change probability,这是产品决策。
  - **「smell 优先级排序」** — 跨模块对比 ROI 是组织决策。
- **工程师必须掌握的核心能力**:
  - **smell 命名作为团队 vocabulary** — 团队会议中说「这处有 Message Chains smell」比「这个函数链太多 dot」准确 10 倍。
  - **判断 smell × refactoring 矩阵中「过度 refactoring」陷阱** — 不是所有 smell 都需要 fix,有时候 smell 是最优解(legacy code 不动原则)。

---

## 六、实践行动项

1. **跑本项目 SonarQube / CodeClimate,top 3 smell 各选 1 处应用 chapter 对应 refactoring**。
2. **最近一次 code review,把所有「注释」摘出来——是不是都可以拆成 Extract Function?**
3. **「Comments 通常是 smell」行动** — 找一个 50+ 行函数，把所有解释 what 的注释**直接替换为 Extract Function 调用**,注释消失。

---

## 七、值得深入思考的问题

1. **「24 个 smell」是完备的吗?** Fowler & Beck 强调是启发式,**你能否在自己项目里发现 chapter 3 没列的 smell?** 例如 distributed system 的「Pervasive Timestamp Drift」「Hidden Coupling via Logs」?
2. **「Comments 通常是 smell」** — 是不是所有注释都是 smell?**API doc、protocol spec 注释、license header 显然不是**。这条 rule 的边界在哪?
3. **「Smell × Refactoring 矩阵」是 1-to-many 还是 many-to-1?** — Long Function 既能 Extract Function 又能 Replace Function with Command,选哪个?**为什么 Fowler 不给决策树?**
4. **Speculative Generality vs Preparatory Refactoring** — 两者都是「为未来做准备」,但前者是 smell,后者是 strategy。**边界在哪?**

---

## 交叉引用

- 第 2 章 Principles → 给出 refactor 的 ROI 判断(本 chapter 的 smell × refactor 矩阵是入口)
- 第 6 章 A First Set → 本章最常用的 smell 在 catalog 的对应
- 第 7 章 Encapsulation → Encapsulate Variable (132) / Encapsulate Record / Encapsulate Collection
- 第 8 章 Moving Features → Move Function (198) / Move Field (207) 是 shotgun surgery / feature envy 的标准处方
- 第 9 章 Organizing Data → Primitive Obsession / Data Clumps 全文
- 第 10 章 Simplifying Conditional Logic → Repeated Switch / Conditional Complexity
- 第 12 章 Dealing with Inheritance → Refused Bequest / Speculative Generality

---

## 附录 · Action #N 复盘

> 留待用户在本地执行时补充。

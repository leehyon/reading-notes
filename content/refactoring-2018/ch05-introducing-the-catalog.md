# Refactoring 2/e · Chapter 5 — Introducing the Catalog

> 来源:Martin Fowler, *Refactoring 2/e* (2018), Chapter 5。
> 章节定位:catalog 入口——介绍后面 6 章(chapter 6~12)怎么读每个 refactoring 条目。每条目五段:name / motivation / mechanics / example / ...
> 模板裁剪：技术书，全 7 节保留。

---

## 一、第一性原理思考

**Fowler 的方法论**:**把 refactoring 从「叙述」变成「可索引的目录」**。每个条目是一份小手术说明书——名字(契约) + 动机(why) + 做法(step-by-step) + 范例。

**公理 1(Catalog 是一本手术手册)**:refactoring 像外科手术,**每个 catalog 条目 = 一个手术**(Extract Function = 切开取出组织;Move Function = 移植器官)。

**公理 2(Pattern-style format)**:**name → sketch → motivation → mechanics → example**。这是 GoF 设计模式的形式,**Fowler 把 refactoring 也按 pattern 形式 catalogize**。

**假设 vs 结论**:
- 假设:refactoring 只能口耳相传，每对师徒重新发明
- 结论:**catalog 让 refactoring 变成可学习的 vocabulary** — 团队会议说「这处用 Extract Function」比「这处把这段抽成函数」精确 10 倍

---

## 二、章节概述

1. **Format of the Refactorings**:每个 catalog 条目五段:
   - **Name**(索引 + 通用契约)
   - **Sketch**(简单示意代码前后对比)
   - **Motivation**(why 用这个 refactoring,什么场景)
   - **Mechanics**(step-by-step 流程)
   - **Example**(完整 before/after 代码)
2. **Choice of Refactorings**:**本书不穷举所有 refactoring** — 只挑常用的 + 关键的。catalog 是「worklist」,不是「comprehensive dictionary」。
3. **How to Read the Catalog**:**读者使用方法 = (1) 闻到 smell → (2) 查 smell × refactoring 矩阵 → (3) 读对应 catalog 条目 → (4) 实施**。
4. **The Catalog**(简单介绍后面章节分组的 6 类:Chapter 6 first set, Ch7 Encapsulation, Ch8 Moving, Ch9 Data, Ch10 Conditional, Ch11 API, Ch12 Inheritance)。

---

## 三、核心 Takeaways

### Takeaway 1 — 「每个 catalog 条目 = 5 段结构」
- **是什么**:**Name + Sketch + Motivation + Mechanics + Example** 五段统一格式。
- **为什么重要**:**统一格式让翻阅和记忆成为可能**。每条 entry 都能用同样的 mental scan 读。
- **解决了什么问题**:每条 entry 写成长篇散文会让 catalog 无法作为工具书。
- **适用场景**:团队内部写「重构 checklist」时采用同样的 5 段格式。

### Takeaway 2 — 「smell × refactoring 矩阵是 catalog 索引」
- **是什么**:chapter 12 末页(内封)有完整矩阵,**每个 smell 列对应建议的 refactorings**。
- **为什么重要**:**从诊断(smell)直接跳到处方(refactor)**。不需要从头读 catalog。
- **解决了什么问题**:**工程师不知道「这个 smell 该用哪个 refactor」** — 矩阵直接回答。
- **适用场景**:code review 中识别 smell → 查表 → 给同事引用具体 refactor。

### Takeaway 3 — 「Mechanics 是 step-by-step,不是 narrative」
- **是什么**:Mechanics 段是 numbered list,**每一步具体到「copy this code」「run this test」「verify」**。**不是描述，而是 procedure**。
- **为什么重要**:**refactor 步骤的离散化**让你可以做 partial commit — 每步 commit + 测试 + 下一 步。
- **解决了什么问题**:「我做了 Extract Function 但中间状态不对」 — mechanics 不让你 skip 步骤。
- **适用场景**:CI 强制 atomic commit,每个 commit 应该是 mechanics 中的 1 个 step。

### Takeaway 4 — 「catalog 不教你新 refactorings,只 catalogize 已存在的」
- **是什么**:Fowler 明示 catalog 是现有知识的整理,**不是「创新」**。**读 catalog 不能让你成为 refactoring 专家，只是加速手册**。
- **为什么重要**:把本书定位清楚——它是 reference,不是 textbook。**真功夫在 smell 识别 + 设计判断，不在记忆 mechanics**。
- **解决了什么问题**:**「读完本书 = 学会 refactoring」的错觉**。
- **适用场景**:把这本书放桌上作 reference,不要试图背完。

---

## 四、工程实践视角

### 如何落地
- **团队 wiki 复刻 5 段格式** — 给每个内部 pattern / refactor 写 Name / Sketch / Motivation / Mechanics / Example 5 段。
- **smell × refactoring 矩阵贴 IDE 边栏** — 把 chapter 12 末页 PDF 截图打印，贴显示器旁。
- **Mechanics 的 git commit 对应** — 每个 step 一个 commit,`git log --grep=Extract` 能搜出所有该 refactor 历史。

### 常见误区(初级工程师)
- **「读完 catalog 就懂 refactor」**:**catalog 是参考不是教科书**。要先有「识别 smell」的能力，再用 catalog 加速。
- **「逐条读 catalog」**:**目录式阅读** — 闻到 smell 才查对应条目，不要试图背完。

### 高级工程师更关注
- **catalog 内部的依赖关系**:**Move Function 依赖 Extract Function(先抽出函数才能移动),Encapsulate Variable 是 Rename Variable 的前提**。**理解依赖图比记忆 mechanics 更重要**。
- **Mechanics 步骤是否 atomic** — Fowler 的每个 step 都能独立 commit,**团队应该照做**。

### 与 NeuSAR cCore V3.0 的潜在连接
- **AUTOSAR 模块的 refactoring 命名** — RTE / BSW 模块内部 refactor 可以用 Fowler 命名,**团队 vocabulary 统一**(口头「在 Cantp 里做 Extract Function」比「把那段拆出来」明确)。
- **Mechanics 节奏映射到 git workflow** — 每个 step 一个 PR,PR title 写 `[refactor] Extract Function amountFor from Cantp_ProcessTx`。
- **内部 wiki 写 5 段条目** — Vector 内部的 RTE refactor 模式可以 catalogize,**方便 onboarding**。

---

## 五、AI 时代视角

- **本章内容今天仍然重要吗**:**format 仍然重要，具体 mechanics 可以 LLM 补充**。
- **AI 能够帮助什么**:
  - **自动补全 mechanics** — 给定 refactoring 名字，生成具体 step-by-step。
  - **本项目 context 化** — 给定 code smell,生成「针对本项目的具体 mechanics」(本项目的 test 框架 / 编译命令 / CI 步骤)。
- **AI 无法替代什么**:
  - **「该用哪个 refactor」的判断** — chapter 3 的 smell × refactoring 矩阵 + 团队上下文。
  - **「这个 refactor 是否值得做」的 ROI** — 取决于未来 change 概率。
- **工程师必须掌握的核心能力**:
  - **快速翻 catalog 找 refactor 的能力**(5 秒内定位)。
  - **把 AI 给的 mechanics 适配到本项目的能力**(不是机械照抄)。

---

## 六、实践行动项

1. **打印 chapter 12 末页 smell × refactoring 矩阵，贴在显示器旁 1 周，每天 code review 试着用 1 次**。
2. **把本项目最近 5 次 refactor PR 重新写「Mechanics」,对比 Fowler 的格式，看看哪一步可改进**。
3. **写一份「本团队的 internal refactoring wiki」草稿，采用 5 段格式**。

---

## 七、值得深入思考的问题

1. **「catalog 是整理现有知识」** — 这意味着 Fowler 1st edition 的 catalog 与 2nd edition 不一样(2nd 多 30+ 条),**新 smell 是否必然伴随新 refactor?**
2. **Mechanics 必须 step-by-step** — 但实际中我们经常「我直接 git stash,改，再 stash pop」。**这是不是 violation?**
3. **「Mechanics 是给新手用的」** — 高级工程师不读 mechanics,直接 inline 改。**是不是意味着 mechanics 只对初学者有价值?**

---

## 交叉引用

- 第 3 章 Bad Smells → catalog 索引入口
- 第 6~12 章 → 每个 chapter 是一组 catalog 条目

---

## 附录 · Action #N 复盘

> 留待用户在本地执行时补充。

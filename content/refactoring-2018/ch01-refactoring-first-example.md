# Refactoring 2/e · Chapter 1 — Refactoring: A First Example

> 来源:Martin Fowler, *Refactoring: Improving the Design of Existing Code*, 2/e (2018), Chapter 1.
> 章节定位：全书开篇，用 JS 重写的「剧团账单」(statement)例子走完一整轮 refactoring,把本书后续要讲的 catalog 先用一遍。
> 模板裁剪：技术书，全 7 节保留。

---

## 一、第一性原理思考

**Fowler 的核心公理**:代码的「好坏」不是审美问题，而是「未来改起来有多快」。编译器不在乎丑不丑，但人会，而人会修改代码。**the true test of good code is how easy it is to change it.**

作者把整个 book 的方法论先压在一个 example 上：从 `statement()` 一坨大函数，经过 Extract Function → Move Function → Inline Variable → Replace Temp with Query → Split Phase → Replace Conditional with Polymorphism 一连串重构，最终得到一个「计算多态 + 渲染线性」的双层结构。

**假设 vs 结论**:
- 假设：小步 + 测试永远绿 + 即时编译 = 「refactoring 比 write-from-scratch 快」
- 结论:**refactoring 的本质不是改变行为，而是把程序员的认知来回搬运——先 read code → 获得 insight → 把 insight 用 refactoring 钉进代码 → 更清晰的代码再激出更深 insight**(正反馈环)

**可迁移到**:任何领域(嵌入式 firmware / AUTOSAR 配置 / Linux driver)凡是「认知先于代码」的工作流都成立。

---

## 二、章节概述

1. **起点代码**(statement 函数)55 行，塞了 switch 计算 + volume credit 计算 + 字符串拼接三类职责。
2. **Comments on the Starting Program** 提出两个变更需求:HTML 输出(横向变体)、新增 play 类型(纵向变体)。两个变体都打到同一坨函数上——这是 refactor 的契机。
3. **The First Step in Refactoring**:先写测试，后改代码。给出 statement 用 JSON fixtures 的 test sample。
4. **Decomposing the statement Function**:连续 Extract Function 抽出 `amountFor` / `volumeCreditsFor` / `playFor` / `format` 等。
5. **Status: Lots of nested functions**:批评者会说「都是 delegation,没逻辑」。Fowler 反驳 — 「小函数的 payoff 是 explanation + sharing + choosing」。
6. **Split Phase (154)** 把 statement 切为 `renderPlainText` + `createStatementData`,后者返回 immutable data,前者只做渲染。
7. **A First Set of Refactorings**:Move Function 把 amountFor/volumeCreditsFor 移到 play 上 → 让 play 拥有自己的计算逻辑。
8. **Replacing the Conditional with Polymorphism (272)**:抽出 `PerformanceCalculator` 抽象基类 + `TragedyCalculator` / `ComedyCalculator` 子类,createPerformanceCalculator 是工厂。
9. **Final Thoughts**:三阶段(decompose → split phase → polymorphic calculator),总结「read → insight → refactor → deeper insight」正反馈环。

---

## 三、核心 Takeaways

### Takeaway 1 — 「the rhythm of refactoring = 小步 + 测试永远绿」
- **是什么**:每一步 Extract/Move 之后必须编译通过 + 测试通过,**永远不在 broken code 上停留**。
- **为什么重要**:把「改 30 处」的风险切成「改 1 处的可观察错误」。Kent Beck 在 Detroit 旅馆教 Fowler 这一招，从此重塑他的工作流。
- **解决了什么问题**:大爆炸式重写「一次性写 200 行新代码」所引发的 debug 灾难。新代码出错时不知道是 200 行哪一行坏的。
- **适用场景**:所有有时间压力的工程改动 — firmware bring-up、tracing 重写、CI 脚本迭代。

### Takeaway 2 — 「refactoring 的目的 = 把 insight 从你的脑子搬进代码」
- **是什么**:读代码 → 局部 insight(例如「tragedy 和 comedy 的 amount 计算应该归各自类」) → 用 refactoring 钉死 → 别人读代码时不必重新推导这个 insight。
- **为什么重要**:团队 cognitive load 分摊 — insight 沉在脑子里没价值，沉在代码里才有复利。
- **解决了什么问题**:「同组里只有 senior 看得懂这段代码」的状态。
- **适用场景**:onboarding 慢、维护成本高的代码区。

### Takeaway 3 — 「Split Phase = 把 calculate 和 render 切开」
- **是什么**:任何「先计算再展示」的函数，先抽成一个返回中间数据结构(createStatementData)的 pure 计算，再抽一个只渲染的 renderText。中间数据**不可变**。
- **为什么重要**:renderText 的多形态版本(HTML / PDF / 邮件)只共享一个数据源，不会有「两份逻辑漂移」的风险。
- **解决了什么问题**:customer 要 HTML,你 copy-paste 出一份 htmlStatement → charging 改了，两份都得改 → bug。
- **适用场景**:报表生成、协议 serialization、firmware telemetry output。

### Takeaway 4 — 「Replace Conditional with Polymorphism 不必类爆炸」
- **是什么**:switch 在 2 个地方(charge + volume credit)都基于 play.type → 抽基类 `PerformanceCalculator`,各子类各自实现 amount / volumeCredits getter。**JS 类的 getter 让外部调用方感觉像读字段**。
- **为什么重要**:新增 play 类型(读者就明确说要加 pastoral / historical)只需新增一个子类 + 在 createPerformanceCalculator 的 switch 加一行,**不要碰 renderText**。
- **解决了什么问题**:「open for extension, closed for modification」在真实例子里的具体落地。
- **适用场景**:状态机类型(type code)、按类型分派的计算、协议版本族。

---

## 四、工程实践视角

### 如何落地
- **先写 fixture-based unit test**,把 statement 当前输出钉死成「reference snapshot」(chapter 给出 `expect(statement(invoice, plays)).toBe(...)`)。任何 refactor 后再跑，只要 green 就行为不变。
- **键盘级别的 Extract Function hotkey**:VSCode `Ctrl+Shift+R` / IntelliJ `Ctrl+Alt+M`。**每 3 分钟一次 Extract Function 比一次写 30 行函数要慢?** 实测相反，因为测试编译时间已经吃掉 80% 等待。
- **commits 频率** = 每个 Extract Function / Move Function 一次 commit,commit message 写「Extract amountFor from statement」。**一周后 git blame 一目了然**。

### 常见误区(初级工程师)
- **「这个函数已经很短了，不用拆」**:Fowler 第 1 段就反驳,**关键不是行数而是「function 名 vs body 的语义距离」**。一行也行，只要 name 比 line 更说明意图。
- **「先写完 feature 再回头重构」**:对「永远不变」的代码行得通，对「下周就要改 6 次」的代码等于挖坟。Fowler 第 6 页明示 — change requests come not as single spies but in **battalions**。
- **「测试可以加在最后」**:refactoring 中加测试 = 中途无法判断「这步是不是 broke 行为」。第 4 章会展开，但本章已暗示:**测试是 refactoring 的前置保险丝**。

### 高级工程师更关注
- **何时停** — 当下一波 change 已经清晰可见，就别再动；当下一波 change 模糊,**继续 refactor 直到 insight 沉下来**。
- **重构 vs 重写** — chapter 的例子原本是 1000+ 行的 C++(Fowler 之前两版选过的),改写到 JS 重新教学是因为 C++ 一旦语法 noise 太大，读者看不到节奏。**真实 production 中,「refactor 老语言 codebase」 vs「rewrite」的选择要看测试覆盖率**(chapter 4 展开)。

### 与 NeuSAR cCore V3.0 的潜在连接
- **BSW 模块拆分**:Dcm/Cantp/CanIf/PduR 的代码表面看起来「一个 module 一个文件」很整齐，但**很多 module 内部仍然有大函数**。把 `Dcm_ProcessPage*` 等函数按 phase(诊断会话层 / 请求处理层 / 应答构造层)做 Split Phase,能直接降低 reviewer 脑负载。
- **测试金字塔** — NeuSAR 配的 CMocka / Vector VT System 测试,**应作为 refactoring 准入门槛**而不是「release 前补」。chapter 4 给出这套哲学。
- **polymorphism 在 RTE**:BSW 的 callback 体系其实已经是「按 service type 分派」,但**配置层**仍大量用 type code + switch。**按行为分组抽接口**可对应 Replace Conditional with Polymorphism。

---

## 五、AI 时代视角

- **本章内容今天仍然重要吗**:100% 重要。**AI 不改变 refactoring 的「为什么」,只改变「找到 smell + 应用 refactoring」的效率**。
- **AI 能够帮助什么**:
  - 自动检测 Duplicated Code、Long Function、Long Parameter List(chapter 3 的 smell 都有现成 LLM 检测器)。
  - 给定 before 代码，生成完整 Extract Function diff + 测试。
  - 重命名一组相关变量时，跨文件保持一致的 semantic naming。
- **AI 无法替代什么**:
  - 「**何时 refactor**」的判断——取决于未来 change 概率，这是不可观测的。
  - 「**为什么这是 smell**」的论证——必须有领域上下文(本团队未来 6 个月加什么 feature)。
  - 测试用例的**价值选择**(哪些行为钉死 vs 哪些允许漂移) — 是产品决策。
- **工程师必须掌握的核心能力**:
  - **阅读 diff 的能力**(用 AI 改完 30 行，你必须能 5 分钟内 review 一遍，挑出 1 处 hidden regression)。
  - **判断「refactor 后是否真的等价」的能力** — 状态机/异步代码里 AI 经常「逻辑等价但语义不等」。
  - **风险评估 + 回滚策略** — AI 生成的 refactoring 必须配合小步 git revert 习惯。

---

## 六、实践行动项

1. **重构自己最近写过的一个 50+ 行函数**:用 `eslint-plugin-sonarjs` + IDE metric,先测「Long Function」warning,**然后用 chapter 1 的节奏,3 分钟一次 Extract Function,commit 一次**。
2. **写 statement 的 TypeScript 版并 100% 测试覆盖**:把 chapter 1 的 JS 例子挪到本地 `~/dev/read/notes/refactoring-2018/code-exercises/ch01-statement/`。
3. **把项目里至少一个模块做成「Split Phase」**:拆出 pure calculation 层，看 render 层是否真的能多形态复用(写一份 JSON / 一份 HTML)。

---

## 七、值得深入思考的问题

1. **为什么 Fowler 选择 JS 而不是 Java 来重写 chapter 1 example?** 答案藏在 chapter 1 末段——因为 JS 的 getter 让 polymorphism「看起来像数据访问」,**新读者不被 class 噪音带跑**。**问自己——教学材料应该选最简的语言还是最普的语言?**
2. **「refactoring 的目标是让 insight 沉到代码」** — 但 AI 时代 insight 本身可以由 LLM 即时生成,**那「沉到代码」这个动作还重要吗?**
3. **statement 函数里有 `format` 是用 `Intl.NumberFormat` 注入的——为什么不把 format 也抽进 PerformanceCalculator?** Fowler 没抽，因为金额只在 render 用，不在 calculate 用。**这条边界告诉你:「应该跟数据走」还是「应该跟渲染走」?**

---

## 交叉引用

- 第 2 章 Principles → 「the rhythm of refactoring」的「why」(本章只给「what」)
- 第 3 章 Bad Smells → 本章的 Long Function / Duplicated Code / Conditional 都是 chapter 3 的 named smell
- 第 4 章 Building Tests → 本章「先写测试」的详细论述
- 第 6 章 A First Set of Refactorings → 本章用到的 Extract Function / Move Function / Split Phase 都有 catalog 条目
- 第 10 章 Simplifying Conditional Logic → Replace Conditional with Polymorphism (272) 全文
- 第 12 章 Dealing with Inheritance → PerformanceCalculator 子类树就是 inheritance 重构的迷你例子

---

## 附录 · Action #N 复盘(执行后填)

> 留待用户在本地执行时补充。

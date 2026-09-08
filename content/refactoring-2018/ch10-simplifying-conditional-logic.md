# Refactoring 2/e · Chapter 10 — Simplifying Conditional Logic

> 来源:Martin Fowler, *Refactoring 2/e* (2018), Chapter 10。
> 章节定位:Conditional logic 是「程序复杂度的主要来源」 — Decompose Conditional、Consolidate Conditional、Replace Nested Conditional with Guard Clauses、Introduce Special Case、Introduce Assertion。
> 模板裁剪：技术书，全 7 节保留。

---

## 一、第一性原理思考

**Fowler 的核心洞察**:**Conditional logic 是程序复杂度的主要来源** — 一段程序如果都是顺序执行，易读；一旦出现 if/else / switch / nested conditional,**每条路径都需要 mental simulation**。**Refactoring conditional logic = 让 control flow 线性化 + 显化 intent**。

**公理 1(Decompose Conditional = 提取分支的意图)**:**复杂的 if/else 让 condition + then + else 都自己命名**。读者不用 trace 全分支。

**公理 2(Guard Clauses = early return)**:**嵌套 conditional 是「正常路径被埋在底部」**。Guard clauses 把异常 / 边界条件前置 + early return。

**公理 3(Replace Conditional with Polymorphism = 用 OO 消灭 switch)**:**switch(type) 的每个 case 应该是 subclass 的 method override**。polymorphism 让 conditional 变成 virtual dispatch。

**公理 4(Introduce Special Case = null object pattern)**:**重复检查 null / sentinel value** → 抽 special case class,默认行为放在里面。

**公理 5(Introduce Assertion = 把假设显化)**:**invariant 检查从 comment 升级为 assert**(运行时验证)。

**假设 vs 结论**:
- 假设:conditional logic 是程序结构的必要部分，无法消灭
- 结论:**conditional 是 smell 时，可以 polymorphism、guard clause、special case 三种方式消灭**

---

## 二、章节概述

包含的 catalog 条目(8 条):

1. **Decompose Conditional (260)** — 复杂 if/else 拆出独立函数。
2. **Consolidate Conditional Expression (263)** — 一连串返回相同结果的 if 合并。
3. **Consolidate Duplicate Conditional Fragments (267)** — if/else 都有同一行 → 提到外面。
4. **Remove Control Flag (270)** — 控制 flag 退出循环 → break / return。
5. **Replace Nested Conditional with Guard Clauses (272)** — 嵌套 if → early return。
6. **Replace Conditional with Polymorphism (272)** — type code switch → 子类 override。
7. **Introduce Special Case (289)** — null / sentinel → special case class。
8. **Introduce Assertion (302)** — invariant 用 assert 显化。

---

## 三、核心 Takeaways

### Takeaway 1 — 「Decompose Conditional 让分支意图显式」
- **是什么**:`if (date.before(SUMMER_START) || date.after(SUMMER_END)) charge = quantity * winterRate + winterServiceCharge; else charge = quantity * summerRate;` → 抽出 `isSummer(date)` / `winterCharge(quantity)` / `summerCharge(quantity)` / `calculateCharge(date, quantity)`。
- **为什么重要**:**分支条件 + then + else 全部命名后，读者不必 trace mental flow**。
- **解决了什么问题**:Long Function smell;Complex Conditional smell;Repeated Switch smell。
- **适用场景**:Dcm 服务权限检查 `if (session == PROGRAMMING && securityLevel >= 1 && notInDefaultSession) ...` → 拆出 `canProgram()` 函数。
- **mechanics**:先抽 condition → 抽 then branch → 抽 else branch → 用 Extract Function 整合 → 替换。

### Takeaway 2 — 「Guard Clauses 把异常路径前置」
- **是什么**:**嵌套 if 让「正常路径」藏在最深处** → Guard Clauses 把每个「异常条件」前置 + early return,正常路径在最外层。
- **为什么重要**:**reader 顺着读下去就是正常路径，异常路径都已处理**。
- **解决了什么问题**:Deeply Nested Conditional smell;Cognitive Load smell。
- **适用场景**:Dcm 0x27 安全访问 seed 生成:
  ```js
  function getSeed(level) {
    if (level === LOCKED) return null;       // guard
    if (!hasAccess(level)) return NO_ACCESS;  // guard
    if (pendingSeed) return pendingSeed;      // guard
    return generateSeed(level);               // happy path
  }
  ```

### Takeaway 3 — 「Replace Conditional with Polymorphism 是 conditional 终极解」
- **是什么**:**switch(type) 散落 N 处 → 抽基类 + 各 case 子类 + virtual method override**。
- **为什么重要**:**加新 case 只需加一个子类，不动现有代码**。**Open-Closed 原则的真实例子**。
- **解决了什么问题**:Repeated Switch smell;Conditional Complexity smell;Speculative Generality smell(预先 switch 留扩展)。
- **适用场景**:chapter 1 的 PerformanceCalculator 就是这模式 — tragedy/comedy 各子类 override `amount` getter。
- **mechanics**:**switch 在 2 处以上才考虑 polymorphism**(rule of three);抽基类(abstract method);子类各自实现;factory 取代 switch。

### Takeaway 4 — 「Introduce Special Case = null object pattern 的 Fowler 化」
- **是什么**:**重复检查 null / "UNKNOWN" / sentinel value → 抽 special case class**(例如 NullCustomer extends Customer),默认行为在 special case 内。
- **为什么重要**:**消灭重复 null check**,每个 caller 不必先判 null。
- **解决了什么问题**:Null Check smell;Repeated Conditional smell。
- **适用场景**:User 没设置 billing address → `customer.getBillingAddress()` 返回 `NULL_ADDRESS`(special case),不返回 null。

### Takeaway 5 — 「Consolidate Conditional Expression = 多条件合一**
- **是什么**:**一连串 if 都返回同一结果** → 合并成一个 if(用 || 或 && 合并 condition)。
- **为什么重要**:**合并表达「这一组条件都视为同一情况」**。
- **解决了什么问题**:Conditional Complexity smell。
- **适用场景**:`if (isDisabled) return 0; if (isRetired) return 0; if (isStudent) return 0;` → `if (isDisabled || isRetired || isStudent) return 0;`。

### Takeaway 6 — 「Introduce Assertion = 把假设从注释升级为运行时检查」
- **是什么**:**invariant 不再是注释「this should never happen」** → 写成 `assert(...)`,失败即 crash + 显化。
- **为什么重要**:**注释是「我希望永远 happen」,assert 是「如果不 happen 我立刻知道」**。
- **解决了什么问题**:Implicit Assumption smell;Bug-prone path 暴露。
- **适用场景**:Dcm session 切换时 `assert(newSession !== currentSession)`(同一 session 不能切换)。

---

## 四、工程实践视角

### 如何落地
- **Decompose Conditional + Extract Function = 经典组合** — 先抽 condition + then + else 三个独立函数，再整合。
- **Guard Clauses 的 firmware 适配** — C 语言没有 early return 时,**用 goto cleanup**(Linux kernel 风格)。
- **Introduce Special Case 用 Null Object** — 设计模式已成熟,**实现成本低**。
- **Replace Conditional with Polymorphism 的 TypeScript 实践** — 用 abstract class 或 union types + discriminated union。

### 常见误区(初级工程师)
- **「switch 比 polymorphism 快」** — **V8 / JVM 已对 virtual call 做 inline cache**,switch 不一定更快。
- **「assert 是 debug 模式用的」** — **release 也开 assert**(NDEBUG 在 C 关闭 assert 的传统已被现代实践否决 — assert 是 invariant enforcement)。
- **「guard clause 浪费一行」** — **多一层缩进浪费 cognitive load 更严重**。

### 高级工程师更关注
- **Replace Conditional with Polymorphism 的过度使用** — 2 case 的 switch 用 polymorphism 是 over-engineering(rule of three)。
- **Introduce Special Case 的成本** — 每个 special case 都要 override 所有 abstract method,**有时 shallow null check 更便宜**。
- **Decompose Conditional 与 Extract Function 的 trade-off** — 拆太细，函数散落难追；拆太粗,function 还是长。**sweet spot = 函数名能完整表达意图**。

### 与 NeuSAR cCore V3.0 的潜在连接
- **Decompose Conditional 在 Dcm**:`Dcm_ProcessRequest` 中的 session check + security check + service dispatch 都应该各自 Extract。
- **Guard Clauses 在 Cantp**:`CanTp_ProcessTx` 的 case 分支前置 timeout / N_Bs > N_Cr 等异常处理,**normal flow 在最后**。
- **Replace Conditional with Polymorphism in BSW**:
  - **Cantp 子协议**(ISO 15765-2 / ISO 15765-4 / CAN-FD TP):抽 `CanTp_Protocol` 基类，各子类 override。
  - **Dcm UDS service handler**:抽 `Dcm_ServiceHandler` 基类，各 service subclass override `handleRequest()` / `buildResponse()`。
  - **PduR routing path**:抽 `PduR_RoutingPath` 基类，各 path 类型 override `route()`。
- **Introduce Special Case in BSW**:`CanTp_NoConnection` 作为 Null Object 处理「没 connection」的特殊 case,而不是到处检查 NULL pointer。
- **Introduce Assertion in Dcm**:Dcm session transition 时 assert 新旧 session 不一致 + assert session 范围合法。

---

## 五、AI 时代视角

- **本章内容今天仍然重要吗**:**100% 重要**。Conditional logic 是软件复杂度核心,AI 改变不了这个事实。
- **AI 能够帮助什么**:
  - **「Decompose Conditional」自动候选** — 识别可拆出的 condition + branch + default。
  - **「Replace Conditional with Polymorphism」的 inheritance 树设计** — 给定 switch + cases,AI 建议 base class + subclasses。
  - **「Introduce Special Case」的 null check 检测** — 批量找出重复 null check,建议 special case class。
- **AI 无法替代什么**:
  - **「polymorphism vs switch」的 ROI 判断** — 2 case vs 3+ case 的 sweet spot。
  - **「null check vs special case class」的 trade-off** — over-engineering 的判断。
  - **「assert 该断言什么」的 invariant 识别** — 业务 invariant 只能由 domain engineer 提取。
- **工程师必须掌握的核心能力**:
  - **Decompose Conditional 的能力**(识别可拆的 condition + branch)。
  - **Replace Conditional with Polymorphism 的设计能力**(abstract class + subclass 设计)。
  - **Introduce Special Case 的成本估算**(每个 special case override N 个 method 的成本)。

---

## 六、实践行动项

1. **一段嵌套 if/else,先用 Decompose Conditional 拆出独立函数，再用 Guard Clauses 重写**。
2. **3+ case 的 switch + 散落多处,Replace Conditional with Polymorphism**(TS abstract class / union types)。
3. **一段重复 null check,Introduce Special Case(Null Object Pattern)**。
4. **一段 invariant 在注释里，改成 assert**(C assert / TS invariant / Python assert / Rust debug_assert)。

---

## 七、值得深入思考的问题

1. **「Replace Conditional with Polymorphism」是 OO 党派的胜利吗?** — **Rust 的 enum match + functional programming 也解决同一问题** — match 是更轻量的 polymorphism?
2. **Guard Clauses vs Decompose Conditional** — 何时该用哪个?Guard Clauses 用于「多异常条件 + happy path」,Decompose Conditional 用于「单一复杂 if-else」。
3. **Introduce Special Case 在 firmware 的成本** — Null Object 在 C 语言要手动写空函数,**值不值?**
4. **assert 在 release 模式该开吗?** — C 的 NDEBUG 关闭的传统被现代 practice 否决，但**embedded system 受限于 flash 空间**,**取舍?**

---

## 交叉引用

- 第 1 章 First Example → PerformanceCalculator 是 Replace Conditional with Polymorphism 的范例
- 第 3 章 Bad Smells → Repeated Switch / Conditional Complexity
- 第 6 章 A First Set → Extract Function 是 Decompose Conditional 的子集
- 第 9 章 Organizing Data → Replace Type Code with Subclasses 是 polymorphism 的入口
- 第 12 章 Dealing with Inheritance → Replace Conditional with Polymorphism 在 inheritance 上下文

---

## 附录 · Action #N 复盘

> 留待用户在本地执行时补充。

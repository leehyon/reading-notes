# Refactoring 2/e · Chapter 12 — Dealing with Inheritance

> 来源:Martin Fowler, *Refactoring 2/e* (2018), Chapter 12。
> 章节定位:inheritance 是 OO 中最强但最容易被滥用的特性 — Pull Up / Push Down / Extract Superclass / Replace Type Code with Subclasses / Replace Subclass with Delegate / Replace Superclass with Delegate 等。
> 模板裁剪：技术书，全 7 节保留。

---

## 一、第一性原理思考

**Fowler 的核心洞察**:**Inheritance = 两个承诺** —— (1) 子类是父类的「is-a」,Liskov 可替换;(2) 子类继承父类的所有可见字段与方法。**两个承诺必须同时成立才能用 inheritance**,否则用 composition(delegation)或组合。

**公理 1(Pull Up / Push Down 改 inheritance 树)**:Pull Up = 把子类共性抽到父类;Push Down = 把父类特性下沉到子类。**判据是「共性 vs 差异」**。

**公理 2(Extract Superclass / Extract Subclass)**:**当 2 个类共享部分字段 + 方法 → 抽 superclass**;**当一个类的某些字段/方法只被部分 instance 用 → 抽 subclass**。

**公理 3(Collapse Hierarchy = inheritance 树过深)**:**subclass 没增加任何实质行为 → 合并回父类**。

**公理 4(Replace Subclass with Delegate / Replace Superclass with Delegate)**:**inheritance 不合适时(违反 Liskov 或增加不必要的耦合)→ 改用 composition + delegation**。

**公理 5(Replace Type Code with Subclasses / Strategy)**:**type code + switch → polymorphism** — chapter 1 PerformanceCalculator 模式。

**假设 vs 结论**:
- 假设:inheritance 是 OO 的核心，越多越好
- 结论:**inheritance 是 design 中成本最高的工具之一——慎重使用，宁可 composition**

---

## 二、章节概述

包含的 catalog 条目(10+ 条):

1. **Pull Up Method (350)** — 子类同名实现 → 抽到父类。
2. **Pull Up Field (353)** — 子类同名字段 → 抽到父类。
3. **Pull Up Constructor Body (355)** — 子类构造共性 → 父类构造。
4. **Push Down Method (359)** — 父类方法只有个别子类用 → 下沉。
5. **Push Down Field (361)** — 父类字段只有个别子类用 → 下沉。
6. **Replace Type Code with Subclasses (362)** — type code + switch → 子类。
7. **Remove Subclass (369)** — 子类没特别行为 → 合并。
8. **Extract Superclass (375)** — 2 个类共享部分字段 + 方法 → 抽 superclass。
9. **Collapse Hierarchy (380)** — subclass 没新增实质 → 合并。
10. **Replace Subclass with Delegate (381)** — inheritance 不合适 → composition + delegation。
11. **Replace Superclass with Delegate (399)** — 反向。
12. **Replace Inheritance with Delegation (399)** — 同 Replace Superclass with Delegate。

---

## 三、核心 Takeaways

### Takeaway 1 — 「inheritance 是 design 中成本最高的工具」
- **是什么**:**inheritance 一旦定下来，很难改** — 它影响所有 caller 对该类型的理解。
- **为什么重要**:**「应该 inherit 还是 delegate」是 OO 设计的核心抉择**。**判错代价巨大**。
- **解决了什么问题**:Speculative Generality smell(为「将来」准备的继承树);Refused Bequest smell(子类不用父类)。
- **适用场景**:新增 play 类型——是 inherit PerformanceCalculator 还是 hold 一个 calculator field?rule:**多变类型用 inherit,稳定类型用 delegate**。

### Takeaway 2 — 「Replace Type Code with Subclasses = type code 的 polymorphism 化」
- **是什么**:`type: 'tragedy'` / `'comedy'` + switch(type) → TragedyCalculator / ComedyCalculator 子类 + factory method。
- **为什么重要**:**加新 type 只需加新 subclass + factory 一行**;**不动 call site**。
- **解决了什么问题**:Repeated Switch smell;Conditional Complexity smell。
- **适用场景**:chapter 1 的 PerformanceCalculator;Cantp 协议子类(ISO 15765-2 / CAN-FD TP);Dcm session 子类。
- **mechanics**:**先 encapsulate 自定义 + factory + 多处 switch 用 polymorphism**。

### Takeaway 3 — 「Pull Up Method / Field = 共性抽取」
- **是什么**:**多个子类有完全相同的实现 → 抽到父类**(abstract method 让各自的 subclass 实现差异部分)。
- **为什么重要**:**Pull Up 是 inheritance tree 的「去重」** — 让共性集中管理。
- **解决了什么问题**:Duplicated Code smell;Divergent Change smell。
- **适用场景**:TragedyCalculator / ComedyCalculator 都有 `performance` / `play` 字段 + `volumeCredits` getter → Pull Up 到 PerformanceCalculator 父类。

### Takeaway 4 — 「Push Down = 拒绝父类的遗产」
- **是什么**:**父类的方法 / 字段只有个别子类用 → 下沉**。
- **为什么重要**:**父类不应该「为 1 个子类预留」 — 那是 Speculative Generality**。
- **解决了什么问题**:Speculative Generality smell;Refused Bequest smell。
- **适用场景**:PerformanceCalculator 父类有 `volumeCredits` getter,只有 ComedyCalculator 特殊逻辑 → **不能下沉**(因为 base volumeCredits 是「共性 + 1 个子类 override」);**实际是 Pull Up + Override 模式**。

### Takeaway 5 — 「Replace Subclass with Delegate = inheritance 转 composition」
- **是什么**:**subclass 共享父类字段但没有真正「is-a」关系 → 改为 hold 一个父类实例,delegate 调用**。
- **为什么重要**:**inheritance 是 compile-time + 静态绑定**;**delegate 是 runtime + 可替换**。
- **解决了什么问题**:Refused Bequest smell;over-engineered inheritance smell。
- **适用场景**:chapter 12 范例:Booking 的 PremiumBooking subclass 应该改为 Booking hold 一个 PremiumBehavior delegate。
- **mechanics**:**先 delegate field + 把 inherited 行为 delegate 调用 + 移除 inheritance + 客户端从「create subclass」改为「create object + inject behavior」**。

### Takeaway 6 — 「Extract Superclass = 现有 class 的共性抽取」
- **是什么**:**多个独立类共享部分字段 + 方法 → 抽 superclass**。
- **为什么重要**:**Extract Superclass 是 inheritance 树的「构造」** — 从 flat class 抽取共性。
- **解决了什么问题**:Duplicated Code smell;类似 class 重复。
- **适用场景**:Cantp ISO 15765-2 / ISO 15765-4 共享 timing 参数 → 抽 CanTp_ProtocolBase superclass。

### Takeaway 7 — 「Collapse Hierarchy = inheritance tree 过浅的精简」
- **是什么**:**subclass 没新增任何实质行为 → 合并到父类**。
- **为什么重要**:**Lazy Element smell 的 inheritance 体现**。
- **解决了什么问题**:Lazy Element smell;useless abstraction。
- **适用场景**:PerformanceCalculator 直接是 ComedyCalculator(没有 TragedyCalculator 用父类逻辑)→ 合并。

### Takeaway 8 — 「Replace Superclass with Delegate = 反向 delegate」
- **是什么**:**父类存在但 client 不该有「is-a」语义 → 改为 hold 父类实例 + delegation**。
- **为什么重要**:**避免 inheritance 强制暴露父类 API**。
- **解决了什么问题**:Interface Segregation smell;Speculative Generality smell。
- **适用场景**:List 继承自 Collection,但 client 用 List 时不应该知道 Collection 的所有 API → List 改为 hold Collection delegate。

---

## 四、工程实践视角

### 如何落地
- **Replace Type Code with Subclasses 用 Strategy pattern** — GoF Strategy 是 Polymorphism 的经典实现。
- **Replace Subclass with Delegate 用 Strategy pattern** — **Strategy 同时是 polymorphism + delegation 的工具**。
- **Pull Up 时机**:**2 个子类开始有重复实现**(rule of two,不是 three,因为 inheritance tree 一次性抽取成本高)。
- **Extract Superclass 在 BSW 子协议** — Cantp 多版本协议(CAN-FD TP / FlexRay TP / Ethernet DoIP)的共性抽取。
- **Liskov 可替换性的判据**:子类是否能在所有父类出现的地方替代父类?**如果不能，应该 delegate**。

### 常见误区(初级工程师)
- **「inheritance 比 composition 高级」** — **composition 比 inheritance 更灵活,GoF 与 Effective Java 都明确建议「favor composition」**。
- **「Pull Up / Push Down 是单向的」** — inheritance 树会演化,**Pull Up 后的共性可能再分裂 → Push Down**。
- **「Replace Subclass with Delegate 是降级」** — **delegate 是更灵活的 design,inheritance 是更紧的 coupling**。

### 高级工程师更关注
- **Liskov 可替换性的实际判据** — **子类 override 不能 narrower preconditions / wider postconditions**(否则父类 caller 契约破)。
- **Inheritance vs Composition vs Mixin** — **Mixin(trait)是第三选项**,介于两者之间。Rust trait / Scala trait 是 mixin 的现代体现。
- **Type Code 用 Subclass 还是 Enum** — **Subclass 是 polymorphism,Enum 是 data + dispatch**。**Java enum 支持 abstract method,所以 enum + abstract method = polymorphism without inheritance tree**。

### 与 NeuSAR cCore V3.0 的潜在连接
- **Pull Up / Push Down in Cantp 子协议**:
  - **ISO 15765-2 / CAN-FD TP 共有 timing**:Pull Up 到 `CanTp_ProtocolBase`。
  - **CAN-FD TP 独有 FD-specific 字段**:Push Down 到 `CanFdTp_Protocol`。
- **Extract Superclass in BSW**:
  - **`Dcm_ServiceHandler` 基类**(所有 UDS service 抽共性):extract `handleRequest` / `buildResponse` abstract method。
  - **`CanTp_Protocol` 基类**(所有 TP 协议):抽 `splitFrame` / `waitForFlowControl` abstract。
- **Replace Type Code with Subclasses in BSW**:
  - **`PlayType`(chapter 1 模式)**。
  - **`PduR routing type`**:Direct / TP / Gateway routing subclass。
  - **`Dcm session type`**:Default / Programming / Extended subclass。
- **Replace Subclass with Delegate in Dcm security level**:Dcm 的 security level 0x01 / 0x03 / 0x05 不需要真正的 is-a,**delegate 给 SecurityLevelStrategy**。
- **Collapse Hierarchy in BSW**:BSW 模块的抽象 class 偶尔只 1 个子类,**合并**。
- **Liskov 可替换性的实际检验** — AUTOSAR 接口的「所有 port group 必须有相同 signature」,**违反就要 replace with delegate**。

---

## 五、AI 时代视角

- **本章内容今天仍然重要吗**:**100% 重要**。inheritance 是 OO 的核心,AI 改变不了这个事实。
- **AI 能够帮助什么**:
  - **「inheritance vs delegate」建议** — 给定 type code + cases,AI 建议 inheritance tree vs composition。
  - **Pull Up / Push Down 候选识别** — 扫描 inheritance tree,识别 Pull Up / Push Down 候选。
  - **Liskov 可替换性检测** — override 方法的 precondition / postcondition 对比父类，识别违反。
- **AI 无法替代什么**:
  - **「inheritance tree 该多深」的 architecture 决策** — depends on future evolution。
  - **Liskov 违反的语义判断** — 业务语义决策,AI 不懂。
  - **「delegate 还是 inherit」的未来 trade-off** — change pattern 决定。
- **工程师必须掌握的核心能力**:
  - **inheritance vs composition 的判断能力** — 何时该用哪个。
  - **Liskov 可替换性的判别能力** — 违反就改 composition。
  - **Pull Up / Push Down 的时机判断** — smell × refactoring 矩阵应用。

---

## 六、实践行动项

1. **本项目 type code + switch,Replace Type Code with Subclasses**(Subclass or Strategy pattern)。
2. **inheritance tree 中 subclass 没新增实质行为,Collapse Hierarchy 合并**。
3. **inheritance tree 中父类为「将来」预留空方法 / 字段(Speculative Generality),Push Down 或 Remove**。
4. **一段 Liskov 可替换性违反,Replace Subclass with Delegate**。

---

## 七、值得深入思考的问题

1. **「favor composition over inheritance」在 Rust / Scala 仍适用吗?** — Rust 的 trait / Scala 的 trait 是 mixin,**是第三选项**。
2. **Replace Type Code with Subclasses vs Replace Conditional with Polymorphism** — 两者几乎等价,**差别在哪?** — Subclasses 是 type system enforcement,Polymorphism 是 design pattern。
3. **Liskov 可替换性是 inheritance 的「道德底线」吗?** — 违反 Liskov 的 inheritance 是不是 anti-pattern?
4. **inheritance tree 的「适当深度」是几层?** — 3 层以上是否 over-engineering?**rule of three 适用吗?**

---

## 交叉引用

- 第 1 章 First Example → PerformanceCalculator 是 Replace Type Code with Subclasses + Replace Conditional with Polymorphism 的完整范例
- 第 7 章 Encapsulation → Encapsulate 与 inheritance 的边界
- 第 9 章 Organizing Data → Replace Type Code with Subclasses / Replace Type Code with State-Strategy
- 第 10 章 Simplifying Conditional Logic → Replace Conditional with Polymorphism 是 inheritance 的常见落地
- 第 11 章 Refactoring APIs → Replace Constructor with Factory 是 polymorphism 的入门

---

## 附录 · Action #N 复盘

> 留待用户在本地执行时补充。

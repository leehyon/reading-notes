# Refactoring 2/e · Chapter 9 — Organizing Data

> 来源:Martin Fowler, *Refactoring 2/e* (2018), Chapter 9。
> 章节定位：数据形态本身就是 refactoring 的对象 — primitive / record / collection / type code / 双向引用 / 单向引用,**「用什么结构表达数据」是 design 决策**。
> 模板裁剪：技术书，全 7 节保留。

---

## 一、第一性原理思考

**Fowler 的核心洞察**:**「数据形态」与「行为责任」是两个独立的设计维度**。同一个 domain 概念可以用 primitive / class / record / reference / value 表达,**每种表达带来不同的 invariant 和演化能力**。

**公理 1(Replace Magic Literal = 让字面量表达意图)**:`1000 * (perf.audience - 30)` → `extraAudienceFee(audience)` / `MAX_BASE_AUDIENCE = 30`。

**公理 2(Replace Type Code with Subclasses)**:**type code + switch → polymorphism** — chapter 1 的 PerformanceCalculator 就是这模式。

**公理 3(Reference vs Value)**:**reference = 对象身份**(共享);**value = 不可变副本**。**何时该 immutable?** 当 equality 是 by-value 且不需身份。

**公理 4(双向引用 vs 单向 + lookup)**:双向 → 死锁风险、序列化麻烦;**单向 + lookup table = 更易演化**。

**假设 vs 结论**:
- 假设：数据结构是「客观的」,选错了改也难
- 结论:**数据形态也是 refactoring 的对象 — primitive → class, mutable → immutable, reference → value,双向 → 单向，每一种都有明确 transform**

---

## 二、章节概述

包含的 catalog 条目(10+ 条):

1. **Split Variable (240)** — 一个变量被多次赋值承担不同职责 → 拆成多个变量。
2. **Rename Field (244)** — 字段改名。
3. **Encapsulate Field (248)** — 字段 public → private + getter/setter。
4. **Encapsulate Collection (同 chapter 7,但 organizational)** — collection getter 隐藏内部 + add/remove 显式。
5. **Replace Magic Literal with Constant (252)** — `40000` / `"tragedy"` 提取为命名常量。
7. **Replace Type Code with Subclasses (362)** — type code + switch → polymorphism(同 chapter 12)。
8. **Change Reference to Value (252)** — 共享 mutable 对象 → 不可变 value。
9. **Change Value to Reference (256)** — 反向。
10. **Replace Array with Object (261)** — `[x, y, z]` 数组 → `{x, y, z}` 对象。

---

## 三、核心 Takeaways

### Takeaway 1 — 「Replace Magic Literal with Constant = 自描述基本功」
- **是什么**:`40000` / `0.1` / `"tragedy"` → `TWO_HOURS_BASE_FEE` / `STANDARD_DISCOUNT_RATE` / `PlayType.TRAGEDY`。
- **为什么重要**:**magic literal 让代码读起来像密码本** — reader 必须去查 spec 才懂含义。
- **解决了什么问题**:Mysterious Name smell;primitive obsession smell。
- **适用场景**:Cantp 的 `0xF1` 流控状态 / `0xCD` 等待状态 → 命名常量。
- **mechanics**:**先声明常量 + 替换所有使用 + 测试 + 跑 cover**。命名要从 domain 而非 implementation 取。

### Takeaway 2 — 「Split Variable 是 Long Function 中隐秘的 smell」
- **是什么**:**一个变量被多次赋值(用同一个名字做不同的事)** → 拆成不同变量 + 加描述性名字。
- **为什么重要**:**变量复用 = reader 必须猜「这个变量现在表达什么」**。
- **解决了什么问题**:Long Function smell;implicit temporal coupling smell。
- **适用场景**:`let result = ...; ...; result = result * 2;` → 改成独立命名变量 + 每次新意图。

### Takeaway 3 — 「Change Reference to Value = 不可变的胜利」
- **是什么**:**共享 mutable 对象(身份重要)→ 不可变 value(每次修改返回新对象)**。
- **为什么重要**:**immutable object 不需要 defensive copy,不会 race condition,可安全 share**。
- **解决了什么问题**:Mutable Data smell;race condition smell;分布式系统的「不一致更新」bug。
- **适用场景**:Dcm 的 `Seed` 应该是 value(`randomByte = 0x42`),不应是 reference(**seed 不应该被持有方修改**)。

### Takeaway 4 — 「Change Value to Reference = 共享身份的代价**
- **是什么**:**每个 instance 独立 value → 共享 identity(同 instance 引用)**。
- **为什么重要**:**当「identity」是 domain 概念**(例如 user / order / config),需要 reference。
- **解决了什么问题**:Data Clumps smell;重复对象导致的「同一概念多个实例」bug。
- **适用场景**:Cantp 的 RxConnection 应该是 reference,**多个 PduR 调用共享同一 instance**(保持 connection state)。

### Takeaway 5 — 「Replace Array with Object = 自描述**
- **是什么**:`[x, y, z]` → `{x, y, z}`。**数组下标是 anonymous,对象字段是 named**。
- **为什么重要**:**下标是 reader 的负担**(`arr[2]` 是「第三个？第五个?」);**named field 是 self-documenting**。
- **解决了什么问题**:Mysterious Name smell;Data Clumps smell。
- **适用场景**:Dcm `[serviceId, subFunction, dataRecord...]` → `{serviceId, subFunction, dataRecord}`。

### Takeaway 6 — 「Encapsulate Field 是数据封装的最小单位」
- **是什么**:public field → private + getter/setter。
- **为什么重要**:**public field 等于放弃 invariant 控制** — 任何代码都能 `obj.field = invalid`。
- **解决了什么问题**:Mutable Data smell;Data Class smell。
- **适用场景**:任何 struct/class 的字段都应该 private + access function。**getter 也不一定要有 — derived data 应该是 query**。

---

## 四、工程实践视角

### 如何落地
- **Magic Literal 自动化检测**:ESLint `no-magic-numbers` / SonarQube / IDE warning。
- **Immutable value 用 TS readonly / Rust mut** — 编译期 enforce。
- **Replace Array with Object 用 TS interface**:`interface Config { width: number; height: number; }` 替换 `[number, number]`。
- **Reference vs Value 在 distributed system** — **CRDT / Event sourcing 的核心就是 Change Reference to Value**(每次状态变化生成 immutable event)。

### 常见误区(初级工程师)
- **「Replace Magic Literal with Constant = 加 const 关键字」** — 这是 TS 的强制,**但 const 不解决「constant 名不描述意图」**。
- **「immutable 一律更好」** — **identity 重要时用 reference**(user / order / session),**equality 重要时用 value**(coordinate / timestamp / money)。
- **「array 比 object 快」** — V8 / JVM 都做了优化,**可读性 > 微秒级 perf**。

### 高级工程师更关注
- **Replace Magic Literal 时机** — 在 smell 出现时,**不要预先全部提取**(rule of three)。
- **Change Reference to Value 的迁移** — 大 codebase 改造 reference → value 是 breaking change,**要有 git migrate + 兼容期**。
- **type code vs polymorphism** — **2 个 case 用 switch,3 个 case 用 polymorphism**? — Fowler 不给 hard rule,**team 决定 sweet spot**。

### 与 NeuSAR cCore V3.0 的潜在连接
- **Replace Magic Literal in BSW config**:`0x00` / `0x01` / `0xFF` 在 BSW 满天飞,**应 Replace Magic Literal with Constant**(或 enum class)。
- **Replace Array with Object in PDU info**:`PduInfoType { SduDataPtr, SduLength }` 已经 object,**比 `[uint8* length]` 好**。
- **Change Reference to Value in Dcm session / security level** — **session state 应该是 value**,每次 transition 返回新 session 对象;**security level 应该 reference**(同一个 level 跨多个 session 共享身份)。
- **Change Value to Reference in BSW module instance** — 每个 ECU 的 CanDrv / CanIf 应该是 reference(单例),不是每个 service 独立 value。
- **Replace Type Code with Subclasses in PlayType / DcmServiceType / PduR routing type** — `type = {TP, CAN, FlexRay}` 等,**用 polymorphism 替代 switch**。

---

## 五、AI 时代视角

- **本章内容今天仍然重要吗**:**100% 重要**。数据形态是 design 的核心,AI 改变不了这个事实。
- **AI 能够帮助什么**:
  - **Magic Literal 批量检测 + 命名建议** — 给定 codebase,AI 找出字面量 + 建议常量名。
  - **Reference vs Value 决策建议** — 给定 domain 概念,LLM 判断该用 reference 还是 value。
  - **data structure 自动重构** — `[a, b, c]` → `{a, b, c}` 的机械迁移。
- **AI 无法替代什么**:
  - **「该 immutable 还是 mutable」的领域判断** — 业务语义决策。
  - **「magic literal 命名」的语义决策** — 名字要反映 domain,不是 implementation。
  - **「reference 共享范围」的 architecture 决策** — singleton vs scoped vs transient。
- **工程师必须掌握的核心能力**:
  - **Magic Literal 自描述能力**(命名清晰度)。
  - **判断 Reference vs Value** — identity 重要 vs equality 重要。
  - **Type Code vs Polymorphism 的 sweet spot** — 2 case 用 switch 还是 polymorphism。

---

## 六、实践行动项

1. **本项目找 5 处 magic number / string,Replace Magic Literal with Constant**。
2. **一段函数有 split variable smell(一个变量被多次赋值),拆成独立命名变量**。
3. **一段 `[a, b, c]` 数组传递，改成 `{a, b, c}` 对象**(或 TS interface)。
4. **一个 mutable value,Change Reference to Value**(TS readonly / Rust impl)。
5. **2+ case 的 type code + switch,改用 polymorphism 或策略模式**。

---

## 七、值得深入思考的问题

1. **「Replace Magic Literal with Constant」的 ROI** — 一处 magic literal 加常量,vs reader 多查 1 次 spec,**哪个更便宜?** 在不同场景答不同。
2. **Reference vs Value 的判据是否普适?** — 在 single-process 是对的,**distributed system 里 Event Sourcing 总是 value-based**,还有别的判据吗?
3. **「Replace Array with Object」** — TS tuple `[number, number, number]` 也可以命名,object 真的必胜吗?
4. **Type Code vs Polymorphism 在 firmware** — 嵌入式 C 经常用 type code(switch 编译期 case table 效率高),**polymorphism 在 C 里代价大**,**取舍?**

---

## 交叉引用

- 第 7 章 Encapsulation → Encapsulate Variable / Encapsulate Record / Encapsulate Collection
- 第 8 章 Moving Features → Move Field 是数据迁移的工具
- 第 10 章 Simplifying Conditional Logic → Replace Conditional with Polymorphism 全文
- 第 12 章 Dealing with Inheritance → Replace Type Code with Subclasses 全文

---

## 附录 · Action #N 复盘

> 留待用户在本地执行时补充。

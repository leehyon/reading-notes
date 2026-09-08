# Refactoring 2/e · Chapter 7 — Encapsulation

> 来源:Martin Fowler, *Refactoring 2/e* (2018), Chapter 7。
> 章节定位：封装是 OO 的核心——但**封装不仅是 private,而是「谁能看到 / 能改 / 怎么改」的设计决策**。
> 模板裁剪：技术书，全 7 节保留。

---

## 一、第一性原理思考

**Fowler 的核心洞察**:**封装不是「隐藏」,而是「受控的访问」**。**Encapsulate Variable (132)** / **Encapsulate Record** / **Encapsulate Collection** — 三件套把不同形态的数据都关进笼子。

**公理 1(Record 是 first-class)**:**JS / Python 的 record(literal object)很容易被任意改字段**。**Encapsulate Record = 把 record 抽成 class / getter 函数**。

**公理 2(Collection 是「指针」)**:返回内部 collection 的函数等于把内部数据泄漏出去。**Encapsulate Collection = 返回 read-only view + 加 add/remove 方法**。

**公理 3(Primitive 不表达 domain)**:**用 string/int 表示领域概念是 Primitive Obsession smell** — **Replace Primitive with Object (174)** 让领域概念升格为 first-class。

**公理 4(Temp Variable 是 smell)**:**temp 变量通常是 Replace Temp with Query (178)** — 把临时变量改成 query function,**避免 query 之间数据不一致**。

**假设 vs 结论**:
- 假设：封装就是加 private 关键字
- 结论:**封装是访问路径的设计** — record / collection / primitive / temp 都要各自的封装策略

---

## 二、章节概述

包含的 catalog 条目(8 条):

1. **Encapsulate Record (162)** — 把 naked record 改成 class。提供 getter,有 mutable 字段提供 setter。
2. **Encapsulate Collection (170)** — 返回内部 collection 的 getter + 不暴露内部 mutator。
3. **Replace Primitive with Object (174)** — 把 magic number / string(如 phone number)抽成 value class。
4. **Replace Temp with Query (178)** — 把局部 temp 改成 query function。
5. **Extract Class (182)** — 类职责过多 → 拆成多个类，数据 + 行为一起搬。
6. **Inline Class (186)** — 类只有一两个方法 → inline 回去。
7. **Hide Delegate (189)** — `manager.department.manager` 链太长 → 暴露 delegate 方法。
8. **Remove Middle Man (192)** — 反向操作：委托过度时让 caller 直接访问。

---

## 三、核心 Takeaways

### Takeaway 1 — 「Encapsulate Record 是 JS 数据结构改造第一反应」
- **是什么**:naked object literal → class with private fields + public getters。
- **为什么重要**:**record 的字段可以被任意赋值** — `customer.address = "evil"` 是 runtime error 的源头。
- **解决了什么问题**:Mutable Data smell;Data Class smell。
- **适用场景**:JSON 配置文件解析后的 object 转成 typed class。

### Takeaway 2 — 「Encapsulate Collection 不只是返回 read-only」
- **是什么**:返回 collection 时,**提供 read-only view + 显式 add/remove 方法 + setter 通常返回 void 或新 collection**。
- **为什么重要**:**返回内部 array = 把内部状态暴露出去** — caller 一改,class invariant 就破了。
- **解决了什么问题**:Mutable Data smell;Feature Envy smell(caller 知道太多 collection 内部)。
- **适用场景**:Cantp 的 RxBufferPool getter 不应该返回 buffer array,**应该返回 bufferCount + getBuffer(i) + addBuffer**。

### Takeaway 3 — 「Replace Primitive with Object = 领域语言升级」
- **是什么**:`phoneNumber: "555-1234"` → `phoneNumber: new PhoneNumber("555-1234")`;PhoneNumber 内部验证格式 + 提供 `getAreaCode()` / `getNumber()`。
- **为什么重要**:**领域概念是 first-class 后,business logic 自然靠拢**(例如 PhoneNumber 可以 `isMobile()` / `isInternational()`)。
- **解决了什么问题**:Primitive Obsession smell;Repeated Switch smell(`switch(areaCode)` 散落各处)。
- **适用场景**:CAN ID 应该是 `CanId` 类，有 `isExtended()` / `getPriority()` / `getPduId()`。

### Takeaway 4 — 「Replace Temp with Query = 数据一致性保险」
- **是什么**:**一段循环或计算里把中间结果存到 temp**,后面再用——把 temp 改成函数,**每次 call 算一次**。
- **为什么重要**:**temp 在多次 query 之间可能 stale**(尤其当 query 内部依赖会被修改的字段)。
- **解决了什么问题**:Long Function smell(配合 Extract Function);Stale Data smell(隐式)。
- **适用场景**:Dcm service 计算费用时 `let basePrice = quantity * unitPrice` → 改成 `function basePrice() { return quantity * unitPrice }`。

### Takeaway 5 — 「Extract Class 的 feature envy 检测」
- **是什么**:**类一半字段被某些方法频繁使用，另一半被另一些** → 拆成两个类，各管自己数据 + 行为。
- **为什么重要**:**职责聚合 → 高内聚低耦合**。Feature Envy smell 的标准解药。
- **解决了什么问题**:Large Class smell;Data Clumps smell(成组字段一起搬)。
- **适用场景**:Cantp 类既有 TxState 又有 RxState → 拆 CantpTx + CantpRx。

### Takeaway 6 — 「Hide Delegate vs Remove Middle Man = 中间层的两难」
- **是什么**:
  - **Hide Delegate**:`manager.department.manager` 太长 → 给 manager 加 `getDepartmentManager()`。
  - **Remove Middle Man**:manager 类大量方法都是委托给 department → 删 manager 的中间层，让 caller 直接调 department。
- **为什么重要**:**delegate 深度 vs 委托广度** — 太深用 Hide Delegate,**太广(过半方法是委托)用 Remove Middle Man**。
- **解决了什么问题**:Message Chains smell;Middle Man smell。
- **适用场景**:RTE 的 port chain `Rte_PpPort_Service_Signal` 太长,**抽 helper**;但 helper 只调一个函数,**直接 inline**。

---

## 四、工程实践视角

### 如何落地
- **TypeScript / Java 的 record** — `readonly` + `as const` + private setter 是 Encapsulate Record 的类型系统强制。
- **Encapsulate Collection 在 C** — 返回数组时**返回 const + 提供 access function** + 永远不在 getter 里返回内部 mutable state。
- **Encapsulate Record 在 TS** — 优先用 `private` field + getter,**避免 `Object.freeze`(运行时成本高)**。

### 常见误区(初级工程师)
- **「封装 = 加 private」** — private 只是 access modifier,**封装是 access path 设计**。
- **「getter / setter 一定要有」** — **read-only 数据就不需要 setter**;**derived data 不需要 getter**(它该是 query function)。
- **「primitive 都是 primitive obsession」** — 有些 primitive 真的就是 primitive(loop index、temporary counter),**不要全部升格**。

### 高级工程师更关注
- **Encapsulate Record 的迁移成本** — 大 codebase 的 record 改造要渐进,**从核心 domain 开始，边角可后置**。
- **Replace Temp with Query 的性能** — query 多次调用 = 多次计算;**对于 hot path,要保留 temp + 加 assertion 验证等价性**。
- **Hide Delegate 的「封装边界」决策** — 不是所有 delegate 都 hide,**只有「client 应该知道的 delegate」才暴露**。

### 与 NeuSAR cCore V3.0 的潜在连接
- **Encapsulate Record 在 arxml 解析**:Vector DaVinci / EB tresos 解析后的 `EcuC` 配置对象通常裸用,**应该 Encapsulate 成 typed wrapper**(提供 getter + setter with validation)。
- **Encapsulate Collection 在 Cantp buffer pool**:`CanTp_GetBufferPool()` 应该返回 buffer count + 单独 access function,**不暴露 array**。
- **Replace Primitive with Object 在 BSW**:
  - **CAN ID**:从 `uint32` 升级成 `CanId` 类，有 `isExtended()` / `getPriority()`。
  - **PDU ID**:升级成 `PduId` 类，有 `getServiceType()` / `getNetworkType()`。
  - **Dcm session**:从 `uint8` session level 升级成 `DcmSession` enum class。
- **Extract Class 在 BSW**:
  - **Dcm 拆 Dcm_RequestHandler + Dcm_ResponseBuilder**。
  - **ComM 拆 ComM_StateManager + ComM_NetworkManager**。
- **Hide Delegate 在 RTE**:`Rte_Ipport_App_Service_Component_Signal_Data` 太长,**抽 helper**。

---

## 五、AI 时代视角

- **本章内容今天仍然重要吗**:**100% 重要**。封装是 OOP 的核心抽象，不会过时。
- **AI 能够帮助什么**:
  - **Encapsulate Record 的 typed wrapper 自动生成** — 给定 JSON schema,生成 typed class + validation。
  - **Primitive Obsession 检测 + 建议升格** — LLM 扫描 codebase,识别频繁出现的 magic number / string。
  - **Replace Temp with Query 的等价性证明** — 给定 temp + query,LLM 论证两者等价。
- **AI 无法替代什么**:
  - **「哪个 primitive 该升格」的领域判断** — 业务概念 vs 工具 primitive 的区别。
  - **Encapsulation 边界的 architecture 决策** — 哪些数据该公开，哪些该隐藏。
  - **Hide Delegate 的「client 应该知道什么」判断** — API design 是产品决策。
- **工程师必须掌握的核心能力**:
  - **识别哪些 primitive 是 domain concept**(phone, email, ID)vs 哪些是 implementation detail(counter, loop index)。
  - **判断 Encapsulation 的边界** — 不 over-engineer 也不 under-encapsulate。
  - **Hide Delegate / Remove Middle Man 的 trade-off** — delegate 链深度 vs 中间层广度。

---

## 六、实践行动项

1. **本项目找 1 个 magic number / string,Replace Primitive with Object 成 typed class**(如 CAN ID、PDU ID、session level)。
2. **一段循环 + temp,改用 Replace Temp with Query**(加 unit test 验证等价性)。
3. **找一处 Message Chains smell(`a.b().c().d()`),Hide Delegate + 链上每个 helper**。
4. **Large Class 拆 Extract Class**:选一个 500+ 行类，按职责拆成 2~3 个。

---

## 七、值得深入思考的问题

1. **Encapsulate Record 在 JSON-heavy codebase 里成本巨大** — 所有反序列化对象都 typed,**值不值?** TS / Java 已经回答了(值),但 Python / JS 还在挣扎。
2. **「read-only getter + mutable setter」的语义边界** — Rust 的 `mut` reference 提供语言级保证,**其他语言怎么 enforce?**
3. **Replace Primitive with Object 的 ROI 曲线** — 1 个升格值,5 个 marginal,100 个 over-engineering。**team 怎么找到 sweet spot?**
4. **Hide Delegate vs Remove Middle Man 的判据** — Fowler 说「看比例」,**有没有更精确的 metric?**

---

## 交叉引用

- 第 6 章 A First Set → Encapsulate Variable (132) 是入门
- 第 8 章 Moving Features → Move Field / Move Function 与 Extract Class 紧密相关
- 第 9 章 Organizing Data → Replace Type Code with Subclasses / Replace Type Code with State / Replace Array with Object
- 第 12 章 Dealing with Inheritance → Replace Subclass with Delegate 的另一面是 Hide Delegate

---

## 附录 · Action #N 复盘

> 留待用户在本地执行时补充。

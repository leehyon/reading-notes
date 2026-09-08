# Refactoring 2/e · Chapter 11 — Refactoring APIs

> 来源:Martin Fowler, *Refactoring 2/e* (2018), Chapter 11。
> 章节定位:API 是 contract — refactoring API 要兼顾「API 调用方不破」「语义不变」「对未来友好」三大约束。
> 模板裁剪：技术书，全 7 节保留。

---

## 一、第一性原理思考

**Fowler 的核心洞察**:**API 是程序与外界(internal / external)的 contract** — 修改 API 比修改内部实现贵得多，因为它影响所有 caller。**API refactoring 必须 migrate 模式，不能大爆炸改签名**。

**公理 1(Query / Command Separation)**:**Query = 只读不副作用**;**Command = 修改状态**。**两者应该分开** — 命令不应返回数据，查询不应修改状态。

**公理 2(Parameterize Function = 消灭 duplicated 行为)**:**多个函数做相似的事(只是参数不同)→ 合并成一个带参数的函数**。

**公判 3(Remove Flag Argument = flag 不应是 dispatch 机制)**:**flag 参数 = switch(caller 决定)**。**改用 Replace Parameter with Explicit Methods**(各 flag 各函数)或 Replace Conditional with Polymorphism。

**公理 4(Preserve Whole Object = 传递完整对象而不是解构参数)**:**解构 = caller 知道太多对象内部**。**传递对象让 receiver 自己取需要的字段**。

**假设 vs 结论**:
- 假设:API 设计是 architect 决定,refactoring 只是「实现细节」
- 结论:**API 是 refactor 中最敏感的部分——它影响所有 caller,改它必须 migrate**

---

## 二、章节概述

包含的 catalog 条目(13 条):

1. **Separate Query from Modifier (305)** — query 不应有副作用,modifier 不应返回数据。
3. **Parameterize Function (310)** — 多个相似函数合并为参数化函数。
4. **Remove Flag Argument (314)** — flag 参数 → 拆成多个显式函数。
5. **Replace Query with Parameter (319)** — 函数查别的类的数据 → 让 caller 传进来。
6. **Replace Parameter with Query (324)** — 反向：不再需要的参数改为函数内部 query。
7. **Remove Setting Method (331)** — field 创建后不应 set → 删 setter。
8. **Replace Constructor with Factory Function (334)** — 构造函数不能表达 factory 语义时用 static factory。
9. **Replace Function with Command (337)** — 函数太复杂(很多参数 + dispatch) → 抽成 command object。
10. **Return Modified Value (346)** — 修改值要 return。
11. **Replace Error Code with Exception (349)** — 错误码 → exception(同 chapter 12 的 Replace Subclass with Delegate)。

### 实际条目清单

1. Separate Query from Modifier
2. Parameterize Function
3. Remove Flag Argument
4. Replace Query with Parameter
5. Replace Parameter with Query
6. Remove Setting Method
7. Replace Constructor with Factory Function
8. Replace Function with Command
9. Preserve Whole Object (319 — 实际在 Long Parameter List 段)
10. Introduce Parameter Object (140 — chapter 6)

---

## 三、核心 Takeaways

### Takeaway 1 — 「Query / Command Separation 是 API 设计的核心原则」
- **是什么**:**函数要么是 query(纯读)要么是 command(纯写)**。**绝不混合**。
- **为什么重要**:**query 可以任意缓存 + 并行 + 重试**;**command 必须顺序**。**混合导致「为什么这次没生效」的玄学 bug**。
- **解决了什么问题**:API 设计 smell;caching 困难;replay 困难。
- **适用场景**:Cantp `CanTp_GetStatus()` 应该是 pure query(返回 status);`CanTp_Transmit()` 应该是 command(不返回结果,error 通过 callback)。

### Takeaway 2 — 「Remove Flag Argument = API 不该有 dispatch 参数」
- **是什么**:`setMode(DEBUG)` 与 `setMode(PROD)` 与 `setMode(TRACE)` 三个函数 → 改用 `enableDebug()` / `enableProduction()` / `enableTracing()`。
- **为什么重要**:**flag 参数让 caller 必须读 doc 才知道 flag 含义**;**显式函数名是 self-documentation**。
- **解决了什么问题**:Speculative Generality smell(flag 留扩展);Lazy Element smell(flag 多半无效值)。
- **适用场景**:Cantp 流控 `CanTp_ProcessFrame(frame, IS_FIRST)` → 改成 `CanTp_ProcessFirstFrame()` + `CanTp_ProcessConsecutiveFrame()`。

### Takeaway 3 — 「Preserve Whole Object = 解构参数是 API smell」
- **是什么**:`planTrip(source, destination, weather)` 解构 caller 手中的 `trip` 对象 → `planTrip(trip)` 直接传对象。
- **为什么重要**:**解构 = caller 知道 trip 对象内部结构**;**传对象 = caller 只知道 trip 概念**。
- **解决了什么问题**:Long Parameter List smell;Feature Envy smell;Insider Trading smell。
- **适用场景**:Dcm `Dcm_ProcessRequest(serviceId, subFunction, dataRecord)` → `Dcm_ProcessRequest(request)`。

### Takeaway 4 — 「Replace Function with Command = API 太复杂时的退路」
- **是什么**:**函数参数过多 + 函数内做 conditional dispatch + 函数成为类的方法** → 抽成 command object。
- **为什么重要**:**command object 可以 hold state(避免参数),可以 subclass override(实现 polymorphism),可以 undo / replay**。
- **解决了什么问题**:Long Function smell;Long Parameter List smell;Feature Envy smell。
- **适用场景**:Dcm `Dcm_ProcessRequest(request)` 如果内部要做 service dispatch,**抽成 `Dcm_Command` 基类 + 各 service subclass**。

### Takeaway 5 — 「Replace Query with Parameter = 解耦隐藏依赖」
- **是什么**:**函数内部 query 别的类，但调用方已经知道那个数据 → 把 query 改成参数，让 caller 传**。
- **为什么重要**:**query 隐藏了依赖**;**参数显化依赖** — 让函数更容易测试(不必 mock 那个类)。
- **解决了什么问题**:Feature Envy smell;Insider Trading smell;test 难以 mock。
- **适用场景**:Cantp `CanTp_HandleFrame(frame)` 内部 `queryConnectionState()`,如果 caller 已经知道 state,**改成参数**。

### Takeaway 6 — 「Parameterize Function = 消灭 duplicated 相似函数」
- **是什么**:`fivePercentRaise()` / `tenPercentRaise()` → `raise(percentage)`。
- **为什么重要**:**两个函数做相同的事只是参数不同 → 合并**。
- **解决了什么问题**:Duplicated Code smell;Speculative Generality smell。
- **适用场景**:Cantp 的 `CanTp_SetSTmin(100ms)` + `CanTp_SetSTmin(50ms)` + ... → `CanTp_SetSTmin(value)`。
- **mechanics 边界**:不要为了参数化把语义不同的函数合并 — **Parameterize 只在「行为完全一致只是数值不同」时**。

### Takeaway 7 — 「Replace Constructor with Factory Function = 命名构造」
- **是什么**:**static factory function 比 constructor 强** — 名字表达意图 + 可以缓存 + 可以返回子类。
- **为什么重要**:**constructor 受限于类名，不能表意**;**factory function 名字完整自描述**。
- **解决了什么问题**:Mysterious Name smell;constructor 限制(constructor 不能有名字 + 不能返回子类)。
- **适用场景**:`new DcmService(...)` → `DcmService.createReadDataByIdentifier(did)`、`DcmService.createDiagnosticSessionControl(level)`。

### Takeaway 8 — 「Remove Setting Method = field 不该可改」
- **是什么**:**field 创建后只能 read,不该可写 → 删 setter**。
- **为什么重要**:**immutable 字段 invariant 更简单**。
- **解决了什么问题**:Mutable Data smell;Long Function smell(无 setter 强制创建时初始化)。
- **适用场景**:Dcm 的 `seed` 是 random 生成,**不该有 setter**。

---

## 四、工程实践视角

### 如何落地
- **API migration 用 deprecation warning** — 旧 API 标 `@deprecated`,console.warn;**不立即删**,给 caller 迁移时间。
- **Query / Command Separation 的 Enforce** — TypeScript 用 readonly return types / Rust 用 `&self` 与 `&mut self` 区分。
- **Parameterize Function 的 IDE 重构** — `Change Function Declaration` 自动管理多个 caller 同步迁移。
- **Factory Function 在 TS** — `static create*()` 命名清晰;Java/Scala 用 `companion object`;Rust 用 `From` trait。

### 常见误区(初级工程师)
- **「flag 参数更紧凑」** — **flag 让 caller 不知道 dispatch 含义**,显式多个函数更好。
- **「constructor 足以」** — **constructor 受限**,factory 才能命名 + 返回子类 + cache。
- **「Parameterize Function 总是好事」** — 行为差异性大的函数,**Parameterize 反而失去语义**。

### 高级工程师更关注
- **API versioning**:**migrate 模式 vs versioned API**(v1 / v2)vs feature flag。
- **Backward compatibility**:**API refactor 永远要保持向后兼容至少一版**。
- **Remove Flag Argument 配合 polymorphism** — flag 通常是 conditional dispatch,**抽多个函数 → Replace Conditional with Polymorphism 更彻底**。

### 与 NeuSAR cCore V3.0 的潜在连接
- **Query / Command Separation in BSW**:RTE 的 `Rte_IRead_*()` (query) 与 `Rte_IWrite_*()` (command) 已分离——这是 RTE API 的核心设计。
- **Remove Flag Argument in CanTp**:`CanTp_Transmit(pduId, pduInfo, mode)` 中 mode flag → 拆成 `CanTp_TransmitStrict()` / `CanTp_TransmitBestEffort()`。
- **Preserve Whole Object in Dcm**:Dcm service handler 不应解构 `Dcm_MsgContext`,**应传整个 msgContext**。
- **Replace Function with Command in Dcm service**:UDS service handler 抽成 Command Pattern(`Dcm_Command` base + 各 service subclass),**为 0x22/0x23/0x27/0x34/0x36 各自 override**。
- **Factory Function in PduR routing**:`PduR_CreateRoutingPath(canIfId, canTpId, ...)` 替代 `new PduR_Routing(...)`。
- **Parameterize Function in Cantp timing**:`CanTp_SetSTmin(100ms)` vs `CanTp_SetBS(50ms)` 行为不同(BS 是 block size,STmin 是 separation time),**不 Parameterize**,**保留独立**。

---

## 五、AI 时代视角

- **本章内容今天仍然重要吗**:**100% 重要**。API 是 contract,AI 不能改变这个事实。
- **AI 能够帮助什么**:
  - **flag argument 检测** — 识别函数内根据 boolean/int 参数 dispatch 的地方，建议 Replace Flag Argument。
  - **Parameterize 候选识别** — 找出结构相似的多个函数，建议 Parameterize。
  - **API deprecation 路径规划** — 给定 API refactor 计划,AI 自动生成 deprecation warning 注释 + migration guide。
- **AI 无法替代什么**:
  - **「API breaking vs backward compat」的策略选择** — 这是产品决策(支持多版本 vs 单版本)。
  - **「Factory Function 命名」** — 业务意图决策。
  - **「Replace Function with Command 是否值得」** — command 模式增加 surface,**case-by-case**。
- **工程师必须掌握的核心能力**:
  - **Query / Command 严格分离的能力)。
  - **flag argument 检测能力**。
  - **API migration 路径设计能力**(deprecation 周期 + migration guide)。

---

## 六、实践行动项

1. **本项目找 1 个 flag argument,Remove Flag Argument 成多个显式函数**。
2. **2 个相似函数(只是参数不同),Parameterize Function 合并**。
3. **一段解构参数(多个对象字段),Preserve Whole Object 改成传整个对象**。
4. **一段 Query / Command 混杂,Separate Query from Modifier**。
5. **一段 constructor 用法，改 factory function 看是否更清晰**。

---

## 七、值得深入思考的问题

1. **API refactor 永远是 breaking change**?**semver 怎么用?** — Major 版本升级 vs continuous migration。
2. **Query / Command Separation 在分布式系统里失效了吗?** — **CQRS 把 query 端与 command 端拆成不同模型**,**是 separation 的极致**,也是 over-engineering 的代表。
3. **「Replace Function with Command」是 OO 时代的产物吗?** — **functional programming 倾向组合 + closure**,command pattern 在 FP 里有何意义?
4. **Factory Function 的「强制命名」是好是坏?** — Java/Kotlin 的 `companion object` 强制 static factory 命名,**过度设计还是好习惯?**

---

## 交叉引用

- 第 6 章 A First Set → Change Function Declaration / Introduce Parameter Object 与本章互为镜像
- 第 7 章 Encapsulation → Encapsulate Collection 的 API 视图
- 第 9 章 Organizing Data → Preserve Whole Object 的数据形态选择
- 第 12 章 Dealing with Inheritance → Replace Constructor with Factory 是 polymorphism 的入门

---

## 附录 · Action #N 复盘

> 留待用户在本地执行时补充。

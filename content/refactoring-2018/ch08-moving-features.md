# Refactoring 2/e · Chapter 8 — Moving Features

> 来源:Martin Fowler, *Refactoring 2/e* (2018), Chapter 8。
> 章节定位:refactoring 的「move」族——改「谁拥有这段代码」的归属。Shotgun Surgery 和 Feature Envy 的标准解药。
> 模板裁剪：技术书，全 7 节保留。

---

## 一、第一性原理思考

**Fowler 的核心洞察**:**「代码归属」是 design 的第一维度** — 一个 function 应该在 A 类还是 B 类，决定耦合方向。把代码从错的位置搬到对的位置，是 design 改写成本最低的工具。

**公理 1(Move 改 ownership)**:Function / Field / Statement 的所有权 = 它归属哪个类 / 模块。**当函数喜欢别的类的数据(Feature Envy)→ 搬过去**;**当多个 class 因同一原因被改(Shotgun Surgery)→ 把代码聚合**。

**公理 2(Pull Up / Push Down 改继承结构)**:**Pull Up = 把子类的相似代码抽到父类**;**Push Down = 把父类代码下沉到具体子类**。**判据是「这是共性还是差异」**。

**公理 3(Inline 是 Move 的反向)**:**当一个类只剩薄薄一层方法,inline 它**;**当一个函数被反复 Move 又 Move 不动,inline 它**。

**假设 vs 结论**:
- 假设:refactoring 是「改代码」,归属问题不能动
- 结论:**归属是 design 的根本——改归属 = 改设计，且成本远低于 rewrite**

---

## 二、章节概述

包含的 catalog 条目(10+ 条):

1. **Move Function (198)** — function 改归属。**Feature Envy smell 解药**。
2. **Move Field (207)** — field 改归属。
3. **Move Statements into Function (213)** — 把 caller 的语句下沉到 callee。
4. **Move Statements to Callers (217)** — 反向：把 callee 的语句上提到 caller。
5. **Replace Inline Code with Function Call (222)** — 把内联表达式换成函数调用。
6. **Slide Statements (223)** — 把相关语句聚拢。
7. **Split Loop (227)** — 一个循环干多事 → 拆成多个。
8. **Replace Loop with Pipeline (231)** — for 循环 → filter/map/reduce。
9. **Remove Dead Code (237)** — 删没用的代码。
10. **Pull Up Field / Pull Up Method (350/353)** — 抽到父类。
11. **Pull Up Constructor Body (355)** — 构造逻辑抽到父类。
12. **Push Down Method / Push Down Field (359/361)** — 父类下沉到子类。

---

## 三、核心 Takeaways

### Takeaway 1 — 「Move Function 是 Feature Envy 标准解」
- **是什么**:**函数大部分时间在摸别的 class 的字段 → 把函数搬到那个 class**。
- **为什么重要**:**让数据 + 行为靠拢** — OOP 的「data with behavior」原则。
- **解决了什么问题**:Feature Envy smell;Shotgun Surgery smell(把分散逻辑聚拢)。
- **适用场景**:Cantp 里有个 `calculateCanIfPduLength(ifConfig, perf)` → 改 `ifConfig.calculatePduLength(perf)`。

### Takeaway 2 — 「Move Field 比 Move Function 更影响 invariant」
- **是什么**:**一个 field 被另一个 class 的方法频繁读写 → 把 field 搬到那个 class**。
- **为什么重要**:**field 是 invariant 的载体** — 搬动 field 等于改 invariant 的归属。
- **解决了什么问题**:Data Clumps smell;Feature Envy smell;Shotgun Surgery smell。
- **适用场景**:CanIf 里 `CanChannelCfg` 的 channel 字段——如果 Tx 处理函数读 channel 多于 CanIf 自己读,**搬到 TxHandler**。

### Takeaway 3 — 「Move Statements into Function vs to Callers = decision tree」
- **是什么**:
  - **Move Statements into Function**:caller 端的初始化语句应该进 callee 内部(避免重复)。
  - **Move Statements to Callers**:callee 的逻辑只有部分 caller 用的前置条件,**上提到 caller**。
- **为什么重要**:**callee 的「common logic」 vs caller 的「specific logic」分离** — 让 callee 更通用。
- **解决了什么问题**:Lazy Element smell;Speculative Generality smell(把 caller-specific 的代码误抽)。
- **适用场景**:Dcm 0x27 安全访问的 seed 生成，先放到 service 函数内；如果是不同 service 都要的 seed,**提到 Dcm 顶层**。

### Takeaway 4 — 「Slide Statements 是 Extract Function 的前奏」
- **是什么**:**把相关语句聚拢到一起(同 locality、相同 precondition、相同 intent),为后续 Extract Function 做准备**。
- **为什么重要**:**Extract Function 抽不出散落的语句** — Slide 是 Extract 的前置条件。
- **解决了什么问题**:Long Function smell;Duplicated Code smell(把相关语句聚拢，跨函数重复更易发现)。
- **适用场景**:Cantp 状态机的 case 分支里散落的「update buffer」「set state」「trigger callback」三件事 → Slide 到一起再 Extract。

### Takeaway 5 — 「Split Loop = 一个 loop 干多事的反模式」
- **是什么**:**一个 for 循环里算 A 又算 B → 拆成两个独立循环**。
- **为什么重要**:**loop 的语义距离** — 「for each item do X and Y」比「for each item do X」+「for each item do Y」更难理解。
- **解决了什么问题**:Long Function smell;Lazy Element smell(复杂 loop 难以命名)。
- **适用场景**:Dcm loop over performances 里既算 amount 又算 volume credit → 拆成两个 loop。

### Takeaway 6 — 「Replace Loop with Pipeline = 函数式化」
- **是什么**:for 循环 + 临时变量 → `data.filter().map().reduce()`。
- **为什么重要**:**pipeline 让每一步 intent 显式,data 不可变，易于并行化**。
- **解决了什么问题**:Long Function smell;Mutable Data smell。
- **适用场景**:Cantp 处理 N_PDU 列表 → filter invalid → map to CanTp_PduInfo → reduce to total size。

### Takeaway 7 — 「Pull Up / Push Down 是 inheritance 的对齐工具」
- **是什么**:
  - **Pull Up**:子类共用代码 → 抽到父类。
  - **Push Down**:父类只有某个子类用 → 下沉到该子类。
- **为什么重要**:**继承结构反映共性 / 差异** — Pull Up / Push Down 让结构对齐真实共性。
- **解决了什么问题**:Duplicated Code smell(子类重复);Speculative Generality smell(父类为「将来」留空)。
- **适用场景**:PerformanceCalculator 子类共有的 volumeCredits 计算 → Pull Up 到父类。

### Takeaway 8 — 「Remove Dead Code 是最被低估的 refactoring」
- **是什么**:**删任何不被引用的代码——包括 comment-out 块、unreachable branch、unused import**。
- **为什么重要**:**dead code = reader 的认知负担** + git 历史已留,**不要保留「以防万一」**。
- **解决了什么问题**:Speculative Generality smell;General Hygiene。
- **适用场景**:BSW 模块升级后，旧 API 没人调 → 删。

---

## 四、工程实践视角

### 如何落地
- **IDE Move Function / Move Field 重构** — IntelliJ / VSCode / CLion 都支持。
- **Pull Up / Push Down 在 BSW** — Cantp 子协议(ISO 15765-2 / ISO 15765-4 / CAN-FD TP)共有的 transport 层 → Pull Up 到父类 TP。
- **Remove Dead Code 配合 git blame** — 删之前 `git log -- <file>` 看历史,**重要功能可能被注释掉但实际还有 fallback**。

### 常见误区(初级工程师)
- **「Move Function 改 ABI / 接口」** — Move Function 改变类归属,**调用方要相应迁移**。不是无成本操作。
- **「Slide Statements 不重要」** — 实战 Slide Statements 是 Extract Function 的前 5 分钟动作,**跳过它 Extract 失败率高**。
- **「dead code 留着以防万一」** — git log 永久可查,**留 dead code 是认知负担**。

### 高级工程师更关注
- **Move Function 的 trade-off**:**搬过去增加耦合**;**搬过去降低 feature envy**。判据:**搬过去后 caller 还能不能单元测试**(可以，搬；不行，考虑 Extract Class 而不是 Move)。
- **Pull Up 时机**:**先有 3 个子类共有的实现差异，再做 Pull Up** — rule of three。
- **Split Loop vs Loop Aggregation** — Split Loop 让每 loop 简单,**代价是多次遍历**。**对于 hot path,有时不拆**。

### 与 NeuSAR cCore V3.0 的潜在连接
- **Move Function in BSW** — Cantp / CanIf / PduR 之间的 boundary 经常因功能演进错位，定期审视「这个函数真的属于这个 module 吗?」。
- **Move Field in RTE** — RTE port group 中共享的 meta 数据，如果只有部分 callback 用,**下推到 callback**。
- **Pull Up / Push Down in BSW**:
  - **Cantp ISO 15765-2 子协议**:把共有 transport 逻辑 Pull Up 到 base TP 类。
  - **Dcm 子 service handler**:把共有 session check + P3Server 计时逻辑 Pull Up。
- **Slide Statements + Extract Function 在 Cantp state machine** — case 分支的「update state」语句散落,**Slide 到一起**再 Extract 成 `transitionTo(state)`。
- **Remove Dead Code in legacy BSW**:旧 ECUC 配置生成的代码常残留,**定期 `git grep` dead function**。

---

## 五、AI 时代视角

- **本章内容今天仍然重要吗**:**100% 重要**。Move 是 design 工具,AI 不改变这个事实。
- **AI 能够帮助什么**:
  - **Feature Envy 检测** — LLM 分析 function × field 矩阵，识别 function 用别 class 的 field 多于本 class。
  - **Move / Pull Up / Push Down 的候选建议** — 给定 inheritance tree,AI 建议 Pull Up / Push Down 候选。
  - **dead code detection 升级** — git history + static analysis + LLM 推断「真没用」 vs 「特殊场景下用了」。
- **AI 无法替代什么**:
  - **Move 后是否真的更好的判断** — 取决于未来 change pattern,AI 看不到未来。
  - **Pull Up / Push Down 的 ROI** — abstraction 有成本(更多 layer),过度 Pull Up = Speculative Generality。
  - **dead code 的「真 dead」判断** — 偶尔被引用的 fallback 代码删了会出 bug,**只有 domain engineer 能判**。
- **工程师必须掌握的核心能力**:
  - **识别 Feature Envy 的能力**(哪个 function 摸别 class 多)。
  - **判断 Move / Pull Up / Push Down 时机** — rule of three + smell 矩阵。
  - **dead code 删除的勇气 + git blame 验证习惯**。

---

## 六、实践行动项

1. **找一处 Feature Envy smell,Move Function 到目标 class**。
2. **找 1 个 loop 干多事,Split Loop + Replace Loop with Pipeline**。
3. **一段 inheritance 树,2 个子类有相似实现 → Pull Up Method / Field 到父类**。
4. **跑 dead code 工具(deadcode / vulture / IDE warning),删 5 处死代码**。

---

## 七、值得深入思考的问题

1. **Move Function 的 cost** — 改类归属 + 改 caller + 改 ABI,**真的比「保留 Feature Envy」便宜吗?** 答：在长期 evolution 中是的，但单次看可能更贵。
2. **Pull Up / Push Down 的边界** — 何时该 inherit / 何时该 delegate / 何时该拆 module?**chapter 12 给出更多继承 vs delegate 判据。**
3. **Slide Statements 是不是 Extract Function 的必备前置?** — 实战中不一定(简单 Extract 也行),**什么时候 Slide 是强必要?**
4. **「dead code 留着以防万一」** — 这个常见借口怎么反驳?**用 git history 作为反驳的武器**。

---

## 交叉引用

- 第 6 章 A First Set → Extract Function 是 Move Statements into Function 的特例
- 第 7 章 Encapsulation → Move Field 与 Encapsulate Variable 紧密相关
- 第 9 章 Organizing Data → Move Field 后续数据迁移
- 第 10 章 Simplifying Conditional Logic → Slide Statements 为 Decompose Conditional 做准备
- 第 12 章 Dealing with Inheritance → Pull Up / Push Down 全文

---

## 附录 · Action #N 复盘

> 留待用户在本地执行时补充。

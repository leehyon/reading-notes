# Refactoring 2/e · Chapter 6 — A First Set of Refactorings

> 来源:Martin Fowler, *Refactoring 2/e* (2018), Chapter 6。
> 章节定位:catalog 第一组——最常用的入门 12 条 refactoring。Extract / Inline / Move / Rename / Introduce Parameter Object 等。
> 模板裁剪：技术书，全 7 节保留。

---

## 一、第一性原理思考

**Fowler 的策略**:**第一批 refactoring 是「机械的、最常用的、新手也能上手的」**。它们不改 architecture,只改 local 结构 — **安全的入门组**。

**公理 1(Extract vs Inline)**:extract 与 inline 是对偶 — **extract 抽走代码,inline 反向合并**。**什么时候 inline?** 当函数 / 变量被抽出后,call site 比函数体还清楚时。

**公理 2(Move 改职责归属)**:Function / Field 的位置 = 谁拥有它的数据 / 谁最常调用它。**如果某个 function 一直摸别的 class 的数据,Move Function 把它搬过去**。

**公理 3(命名即文档)**:Change Function Declaration、Rename Variable、Rename Field 是 **最频繁的 refactoring** — 因为命名是认知的核心。

**假设 vs 结论**:
- 假设:refactoring 都复杂
- 结论:**80% 的日常 refactor 就是 Extract / Inline / Move / Rename 4 件套**

---

## 二、章节概述

包含 12 条 catalog 条目，每条按 5 段格式:

1. **Extract Function (106)** — 把一段代码抽成函数。**99% 的 Long Function smell 解药**。
2. **Inline Function (115)** — 反向操作：把函数体 inline 回去。
3. **Extract Variable (119)** — 把表达式抽成局部变量，常用于复杂条件或魔术数。
4. **Inline Variable (123)** — 反向操作。
5. **Change Function Declaration (124)** — 改名 / 改参数 / 改返回类型。**用 migrate 步骤逐步 atomic 替换**。
6. **Rename Variable (137)** — 局部变量改名。
7. **Encapsulate Variable (132)** — 把全局 / 类变量包成 getter/setter。**Mutable Data + Global Data smell 第一反应**。
8. **Introduce Parameter Object (140)** — 把总是一起出现的参数打包成对象。
9. **Combine Functions into Class (144)** — 把共享参数的多个函数聚合成类。
11. **Split Phase (154)** — 把一个函数切成两个 phase(计算 + 渲染),中间用 immutable 数据结构传递。
12. **Combine Functions into Transform (149)** — 输入 → 累积 transform → 输出，适合报表生成。

---

## 三、核心 Takeaways

### Takeaway 1 — 「Extract Function = 99% 的 Long Function 答案」
- **是什么**:把一段代码 + 它依赖的 locals 抽成新函数,**调用方传最少参数**。前提:**测试覆盖到位**。
- **为什么重要**:**降低 cognitive load + 拆出可独立命名的小函数 = 复用 + 共享 + 选择 三大 payoff**。
- **解决了什么问题**:Long Function smell;Nested Conditional smell(配合 Decompose Conditional);Shotgun Surgery smell(把重复逻辑聚合)。
- **适用场景**:Cantp 里把 FC frame 处理抽成 `processFlowControl()`。
- **mechanics 关键**:先声明新函数 + 复制代码 + 替换原代码为调用;**而不是先写新函数再删旧的**。

### Takeaway 2 — 「Inline Function 在 Over-abstraction 时反向用」
- **是什么**:**当函数 / 变量 body 比调用点还短时，把函数体 inline 回去**。
- **为什么重要**:避免「过度 delegation」 — 一层无意义的 wrapper 让 reader 多跳一次但没获得 information。
- **解决了什么问题**:Lazy Element smell(类 / 函数没什么用);Speculative Generality smell(为「将来」抽的 hook)。
- **适用场景**:重构中途发现某个 helper 函数就一行 — inline 它。

### Takeaway 3 — 「Change Function Declaration 的 migrate 步骤」
- **是什么**:**改名函数时不要直接 replace_all**,而是 (1) 用新名字加新函数 → (2) 调用方迁移到新函数 → (3) 旧函数 inline 或 delegate 到新函数 → (4) 删旧函数。
- **为什么重要**:**atomic commit + 兼容旧调用** — 在大型 codebase 渐进重构时必备。
- **解决了什么问题**:**「我在 branch A 改函数签名,branch B 同时改,merge 冲突」**。
- **适用场景**:RTE port 改名时 BSW + ASW 同时迁移。

### Takeaway 4 — 「Encapsulate Variable 是 Mutable Data smell 的 first move」
- **是什么**:**把变量包成 getter/setter 函数**,所有访问通过函数，赋值受控。
- **为什么重要**:**Mutable Data 是「spooky action at a distance」** — 任何一处可改任何一处可读,bug 不可追踪。
- **解决了什么问题**:Mutable Data smell;Global Data smell;Long Function smell(参数过多时配合 Preserve Whole Object)。
- **适用场景**:BSW 模块的全局 state flag — 应该 Encapsulate 成 setter。

### Takeaway 5 — 「Introduce Parameter Object = Long Parameter List smell 解药」
- **是什么**:**把总是成群出现的参数打包成对象**(如 `startDate`、`endDate` → `DateRange`)。
- **为什么重要**:**减少参数数量 + 增加 parameter 的语义 + 让一组参数可以在对象上加 method**。
- **解决了什么问题**:Long Parameter List smell;Data Clumps smell。
- **适用场景**:Cantp 流控参数(N_TA, N_SA, N_BS, N_CR, N_AS)打包成 `CanTpTiming`。

### Takeaway 6 — 「Split Phase = 报表类函数的通用解」
- **是什么**:任何「先计算再展示」的函数 → 拆成 pure 计算 phase + render phase。**中间用 immutable data structure 传递**。
- **为什么重要**:render 多形态复用同一份数据;calculate 单元测试无依赖。
- **解决了什么问题**:Duplicated Code smell(HTML 报表 vs Text 报表共享 calculate)。
- **适用场景**:Dcm 0x22 ReadDataByIdentifier 把 data flow 和 protocol encoding 分 phase。

### Takeaway 7 — 「Combine Functions into Transform = 流水线化」
- **是什么**:把「读数据 → 算 a → 算 b → 算 c → 写结果」改成「读数据 → 多次 transform → 写」,**transform 累积成 pipeline**。
- **为什么重要**:**每个 transform 单元测试 + 复用 + 独立命名**。
- **解决了什么问题**:Long Function smell;Feature Envy smell(transform 函数对 raw 数据改写)。
- **适用场景**:Dcm 0x34/0x36 download 数据按 chunk 计算 CRC + 解密 + 校验。

### Takeaway 8 — 「Combine Functions into Class = 把共享参数的函数聚合」
- **是什么**:**一组共享参数的函数 → 聚合成一个 class,参数变成 fields**。
- **为什么重要**:**类能 hold state + 函数能 partial apply** — 把「函数 + 数据」封装起来。
- **解决了什么问题**:Long Parameter List smell;Feature Envy smell(数据属于谁);Data Clumps smell。
- **适用场景**:RTE 的 port group 共享同一 PDU — 抽成 Rte_PduGroup 类。

---

## 四、工程实践视角

### 如何落地
- **IDE hotkey 配对** — Extract Function(IDE 自带)、Rename Variable(`Shift+F6` in IntelliJ)、Encapsulate Variable(自动 getter/setter)。
- **Change Function Declaration 用 git migrate** — 不依赖 IDE 重构能力,**git commit + 团队通知 + 兼容期**。
- **Encapsulate Variable 在 firmware 上下文** — C 语言没 class,**Encapsulate Variable = 把全局变量藏到 .c 里只暴露 setter/getter 函数**,这是 C 的「封装」。

### 常见误区(初级工程师)
- **「参数多就抽 Parameter Object」** — rule of three:1~2 次出现的参数组合,**不一定抽**。
- **「函数短就好」** — inline function 的反例：过度 extract 比原版更难读。
- **「Encapsulate Variable 就是加 private」** — Encapsulate Variable 强调 **「访问路径受控」**,不只是 access modifier。

### 高级工程师更关注
- **Extract Function + Slide Statements 的协同** — 抽函数前先用 Slide Statements 让相关代码聚拢,**否则抽出来的函数参数过多**。
- **Split Phase 的中间数据结构** — **不可变 + 自描述** 是关键，不只是「返回 object」。
- **Encapsulate Variable 的 setter 校验** — Encapsulate 不是简单 setter,**setter 里加 invariant check**(setter 拒绝非法值)。

### 与 NeuSAR cCore V3.0 的潜在连接
- **Extract Function 在 Cantp / CanIf** — Cantp 的 `CanTp_ProcessTx` 通常是 80+ 行,**应该 Extract 出 `processFlowControl`、`processSingleFrame`、`processFirstFrame` 等**。
- **Encapsulate Variable 在 BSW 全局**:`CanIf_ChannelConfig` / `PduR_RoutingTable` 应该 Encapsulate 起来，避免全局裸访问。
- **Introduce Parameter Object 在 RTE**:Rte_Port 数量爆炸时,**group 同类 port 到 Rte_PortGroup 类**(类似 AUTOSAR 的 SenderReceiverInterface 概念)。
- **Split Phase 在 Dcm**:Dcm service handler 的「解析 request → 调 ASW callback → 编码 response」天然三 phase,**应该分别抽函数 + 用 immutable data 结构传递**。

---

## 五、AI 时代视角

- **本章内容今天仍然重要吗**:**mechanics 部分机械化，可被 AI 自动化；但 smell 识别 + 是否该 refactor 仍靠人**。
- **AI 能够帮助什么**:
  - **批量 Extract Function** — 给定大函数,AI 找到可抽取的子逻辑，生成 candidate function。
  - **Encapsulate Variable 的 getter/setter 自动生成** — 静态语言 IDE 已做,JS/动态语言 AI 更有效。
  - **Parameter Object 自动打包** — 找出「总是一起出现」的参数，建议打包。
- **AI 无法替代什么**:
  - **「该 inline 还是 extract」的判断** — 看 semantic distance,不是机械规则。
  - **「Change Function Declaration 的迁移路径规划」** — 跨模块 / 跨 repo 的渐进迁移 AI 难规划。
  - **「Encapsulate Variable 的 setter invariant 决定」** — 这是领域决策,AI 不懂业务约束。
- **工程师必须掌握的核心能力**:
  - **Extract Function 时机判断**(别在写 code 时就抽完所有可能性——留到 smell 出现时)。
  - **Encapsulate Variable 的边界判断**(不是所有变量都要 Encapsulate,**临时 / 局部 / 函数内** 都不必)。
  - **理解 refactor 之间的依赖**(Change Function Declaration 需 Encapsulate Variable 配合)。

---

## 六、实践行动项

1. **最近一个 50+ 行函数，每 5 分钟一次 Extract Function + commit**,10 个 commit 后看 diff 是否 atomic。
2. **找一处全局变量，用 Encapsulate Variable 改成 setter/getter**,看 setter 是否加 invariant check。
3. **本项目 3+ 个函数共享同样 2+ 参数，用 Introduce Parameter Object 打包成 struct/class**。
4. **一段 Long Function 兼 Long Parameter List,用 Slide Statements + Extract Function + Introduce Parameter Object 三连做掉**。

---

## 七、值得深入思考的问题

1. **Extract Function 的「参数最少化」和「可读性」权衡** — 抽出的函数如果参数过多,**是不是该抽 Parameter Object?** — 是，但 Parameter Object 本身又增加 surface,**什么时候是 over-engineering?**
2. **Change Function Declaration 的 migrate 模式 vs 大爆炸改签名** — 在大型 codebase,migrate 模式有时会拖 2 周,**有没有更快的方式?**(Bazel / monorepo 的 impact analysis)
3. **Encapsulate Variable 在 C 语言怎么办?** — `static` + getter/setter 是 C 的最佳实践,**还是有更地道的 C 写法?**
4. **Split Phase 的中间数据结构** — 不可变 + 自描述是好的，但**有时候纯函数返回不可变 struct 在嵌入式栈上太贵**。**取舍怎么定?**

---

## 交叉引用

- 第 1 章 First Example → Extract Function / Move Function / Split Phase / Replace Conditional with Polymorphism 全用到了
- 第 3 章 Bad Smells → Extract Function 解 Long Function / Duplicated Code
- 第 5 章 Introducing the Catalog → 5 段格式定义
- 第 7 章 Encapsulation → Encapsulate Variable / Encapsulate Record / Encapsulate Collection
- 第 10 章 Simplifying Conditional Logic → Extract Function 配合 Decompose Conditional

---

## 附录 · Action #N 复盘

> 留待用户在本地执行时补充。

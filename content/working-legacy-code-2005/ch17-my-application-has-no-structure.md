# Chapter 17 — My Application Has No Structure

> **PDF**: p.237-248（12 页）
> **定位**: ch16 给你 *单文件* 的理解技法（sketching / listing markup / scratch refactor / delete unused）；ch17 推到 *整个应用架构* —— 当你不只是不懂一段代码，而是 *没人懂整个应用的形状*。Feathers 给 3 个低技术、高传染性技法：**Telling the Story of the System**（强制 2-3 句话讲架构） + **Naked CRC**（index cards 摆对象实例） + **Conversation Scrutiny**（听对话，看代码里的概念是否和对话对齐）。整章隐线：*架构不能只属于 architect，必须属于团队每个人*。

## 〇、第一性原理思考

**这章做了什么**: 针对"没人懂整个应用架构"的问题, 给 3 个团队级动作 —— Telling the Story of the System(用 2-3 句话强迫简化讲架构)、Naked CRC(index cards 摆对象实例表达运行时关系)、Conversation Scrutiny(听对话里冒出的新词, 看代码里有没有对应类)。

**为什么这样拆**: Feathers 把 ch16 的单文件动作推到了团队规模, 因为没结构是组织问题而非个人能力 —— 强制"架构属于每个写代码的人"通过对话和卡片这些低技术形式落地, 而不是文档/UML, 因为后者会随项目长大而 drift。

**最值钱的洞见**: 实际上 Conversation Scrutiny 揭示的是 —— 当对话里出现 locking policy 但代码里没有 LockingPolicy 类, 这不是文档缺失, 是抽象滞后, 锁定 policy 这种命名机会通常就藏在工程师嘴边的口头词里。
## 一、章节概述

- **应用的"没结构"是历史形成的**。早期架构清晰 → 项目长大 + 工期压力 → 加 hack → hack 越来越多 → 没人记得原架构 → 新人找 hack 点加 feature → hack 更密。**没人故意这样**，但发生得不知不觉。
- **没结构的 3 大原因**：(1) 系统太复杂，"big picture" 要花很久才看得见；(2) 系统太复杂，*根本没有* big picture；(3) 团队一直在 emergency 模式，看不到 big picture。
- **Architect 角色的局限**：可以有 architect，但 architect 必须 *每天在团队里*；否则两种 drift：(a) 团队做 architect 不知道的事；(b) architect 的图和现实差太远。
- **关键断言：架构不能只属于 architect**。每个写代码的人都要知道架构；任何学到东西的人都要把知识广播给团队。**当 20 人团队只有 3 人懂架构时，剩下 17 人必然犯错**。
- **3 个技法**：
  1. **Telling the Story of the System**：两人对话。一个人问"系统架构是什么"，另一个 *只用 2-3 个概念* 解释。强迫简化 = 强迫抽象。
  2. **Naked CRC**：用 index cards 在桌上摆对象实例。两规则：(a) cards 代表 *实例* 不是 class；(b) cards 重叠表示 collection。这是 Ward Cunningham + Kent Beck 1980s 的 CRC 卡片的 *简化版*。
  3. **Conversation Scrutiny**：听 pair 工作时的对话，看 *对话里用的概念* 和 *代码里的概念* 是否一致。如果不一致，要么代码没跟上团队理解，要么团队该重新理解。
- **JUnit 例子贯穿 Telling the Story**：Feathers 解释 JUnit 架构用 1 句话"有两个主类 Test 和 TestResult"。然后逐步细化：当他说 "TestCase" / "TestSuite" / "reflection" 时，他主动暴露 *简化是撒谎*——并以此为基础问"这是合理的简化吗？"
- **改 feature 时，用 *故事* 判断方案好坏**。Feathers 演示：JUnit 加 "report tests with no asserts"——两个方案：(a) 加 `TestCase.buildUsageReport()`；(b) 改 TestResult 记 assertion count。**(b) 让故事更接近真**，因为不增加 TestCase 的责任。
- **Conversation Scrutiny 例子**：一个团队要把 single-threaded 代码改 multi-threaded；他们讨论 *locking policy + count array*。Feathers 说 "等等，你说的是 locking policy 啊，**为什么不让 LockingPolicy 类承载这个？**"——团队 *对话里* 用了 "locking policy" 这个词，但 *代码里* 没有这个类。这是 conversation scrutiny 的胜利。
- **架构在脑子里死掉的最坏情况**：团队觉得 "design 结束了"。**之后还有改动** = 新代码乱塞 + 类膨胀 + 没有新抽象 = legacy 越变越糟。
- **chapter 17 是 ch16 的"系统级版本"**。ch16 是单文件动作；ch17 是团队级别动作。两者同精神（低技术 + 高传染 + 动作化）。

## 二、核心 Takeaways

### Takeaway 1: 架构不属于 architect —— 属于团队每个人

- **是什么**：Feathers 直言 *"The brutal truth is that architecture is too important to be left exclusively to a few people."* 即使有 architect，*每个写代码的人* 都要知道架构；任何学到东西的人要把知识广播给团队。
- **为什么重要**：当 20 人团队只有 3 人懂架构时，剩下 17 人必然犯错——而且是 *系统级* 的错（hack 点错 / 抽象不到位 / 类膨胀）。
- **解决什么问题**：让架构 knowledge 在团队里 *分布式* 存在——任何单点离开（人员变动 / 紧急任务）不影响整体。
- **适用场景**：长期项目；团队扩大；onboarding 体系设计。

### Takeaway 2: Telling the Story —— 强迫 1-2 句话讲架构

- **是什么**：两人对话。Q: "系统架构是什么？" A: 用 2-3 个概念解释。然后逐层细化。**关键**：简化是 *有意的撒谎*，但 "故意撒谎" 比 "真话太多" 更有用——它强迫抽象、强迫识别最重要的事。
- **为什么重要**：写文档常失败因为"试图讲全"——Telling the Story 反过来：先讲 *最少*，逐步补。每次 *简化* 都挑战"这是合理的简化吗？"
- **解决什么问题**：让团队对架构有 *共同语言*——所有 pair 工作都基于同一故事。
- **适用场景**：新人 onboarding；架构变更 review；feature 设计 early stage。

### Takeaway 3: Naked CRC —— index cards 摆对象实例

- **是什么**：用 index cards 在桌上摆对象实例。两条规则：(a) cards 代表 *实例* 不是 class；(b) cards 重叠表示 collection。Ron Jeffries 1980s 给 Feathers 演示的形式——用 cards + 位置 + 移动表达对象关系。
- **为什么重要**：类图（UML）静态；cards + 位置表达 *运行时关系*——比如 "session 启动时在 server 侧创建"，可以 "把 server session card 放在 session card 旁边"——这个空间关系是 UML 难表达的。
- **解决什么问题**：让"对象怎么交互" *可演示* —— 比 UML 更接近真实运行模型。
- **适用场景**：多对象系统设计；onboarding；分布式系统调试。

### Takeaway 4: Conversation Scrutiny —— 对话里用的词 = 代码该有的类

- **是什么**：听 pair 工作的对话。如果对话里出现 *新概念*（如 "locking policy"），但代码里没有 *对应类*——这是个 *类提取机会*。**LockingPolicy 的诞生** 就是这个观察的产物。
- **为什么重要**：对话是 *自然语言*——它表达团队当前 *理解*。如果对话里有概念但代码没有 = 抽象滞后。
- **解决什么问题**：让代码和团队理解保持同步——避免"代码结构 vs 团队心智模型" drift。
- **适用场景**：复杂功能设计；长期维护的类膨胀治理。

### Takeaway 5: "简化是有意的撒谎" 是 Telling the Story 的关键 insight

- **是什么**：JUnit 故事 "有两个主类 Test 和 TestResult" 显然是 *简化*——JUnit 有 50+ 类。但 *简化* 强迫 *"为什么不是真的这样？"* 这个问题。如果发现"应该有但没有" = 设计机会。
- **为什么重要**：把"故事" 当 *设计 oracle* —— 故事越接近真，结构越好。
- **解决什么问题**：让"我们是不是该加一个新类" 变成可回答（看故事是否要改）。
- **适用场景**：feature 加法 review；class 命名争议。

### Take Takeaway 6: "大块 procedural code 像黑洞" —— 它吸引更多 procedural

- **是什么**：Feathers 引用 LockingPolicy 例子的 *"There is something mesmerizing about large chunks of procedural code: They seem to beg for more."* —— 团队没有 inexperienced，但 *procedural style 一旦确立* 就自我强化。
- **为什么重要**：legacy 的腐烂有 *传染性* —— 一处坏代码邀请更多坏代码。
- **解决什么问题**：让"为什么我们要 refactor" 不只 *效率* 论证，还有 *传染性* 论证。
- **适用场景**：code review；refactor 优先级讨论。

### Takeaway 7: "design is over" 是最坏的态度

- **是什么**：Feathers 末段 *"One of the worst mistakes a team can make is to feel that design is over at some point in development."* —— 如果团队觉得 design 结束，但改动继续 = 新代码乱塞 + 类膨胀 + 没有新抽象 = legacy 越糟。
- **为什么重要**：design 是 continuous activity，不是 phase。
- **解决什么问题**：让"design review" 成为持续实践，不是 milestone。
- **适用场景**：流程设计；Sprint 规划。

### Takeaway 8: Ch17 与 ch16 的接力

- **是什么**：ch16 = 单文件动作（sketching / markup / scratch refactor / delete unused）。ch17 = 团队级别动作（story / CRC / scrutiny）。两者同精神：低技术 + 高传染 + 动作化。
- **为什么重要**：从"我不懂这段代码" 到 "没人懂这个系统" 的递进。
- **解决什么问题**：避免团队把 ch16 用作单点技巧而忽略系统层。
- **适用场景**：长期项目；trail-and-error 的复盘。

### Takeaway 9: JUnit 故事 = Telling the Story 的工业 demo

- **是什么**：Feathers 用 JUnit 做 *对话演练*——
  1. "JUnit 有两个主类，Test 和 TestResult。Users 跑 tests，passing TestResult。当 test fail，告诉 TestResult。人们向 TestResult 问 failures。"
  2. 简化声明：TestResult 是 interface、reflection 创建 test 实例、errors 和 failures 区别、TestSuite grouping。
  3. "这些简化给我们什么 insight？"
- **为什么重要**：把 *抽象级别* 的判断变成 *对话*——两人可以一起做。
- **解决什么问题**：让"我们的架构" 也能像 JUnit 一样 *讲个故事*。
- **适用场景**：实际 onboarding；架构 review 模板。

## 三、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **kernel 维护者的 *telling the story***。Linus 在邮件列表讲 kernel 架构时，常用 1-2 句话："kernel = scheduler + memory + VFS + network + ..."。新人读邮件列表就能抓 big picture。
- **systemd 文档的 story-driven design**。systemd 的 `systemd(1)` manpage 第一段就讲 "systemd is a system and service manager for Linux"——一行的故事。后续细化。**这正是 ch17 的工业版**。
- **dbus 的 architect + maintainer drift 风险**。dbus 早期有明确 architect；后期 maintainer 分散 = 文档 vs 实际 drift。dbus 现在的 *specification* 和 *implementation* 不对齐 = ch17 警告的现实。
- **kernel scratch refactor 与 system-wide Telling the Story**：当一个 subsystem refactor 大改（如 io_uring 引入），maintainer 通常在 LKML 写 *"the story is..."* 邮件讲清新架构——ch17 在邮件列表的映射。
- **Naked CRC 在 kernel patch review 里 = call chain diagram**。Reviewer 经常画 "A calls B which calls C in context X" 的 ASCII 图——位置关系表达系统级交互。
- **Conversation Scrutiny 在 Linux 系统侧 = review 时 catch 新概念**。kernel patch review 时常见"wait, you're talking about X, shouldn't there be a helper for X?"——与 LockingPolicy 类比完全一致。
- **LockingPolicy 类比 = kernel 里的 locking primitives**。kernel 把 lock 设计成独立子系统 (`kernel/locking/`)——而不是内联到每个 driver。这就是 *对话里有 locking policy* → *代码里有独立子系统* 的工业实例。

#### Linux 系统 — ch17 技法映射表

| Feathers 技法             | Linux 系统等价做法                          | 工具 / 输出                          |
| ------------------------- | ------------------------------------------- | ------------------------------------ |
| Telling the Story         | manpage 第 1 段 / LKML "the story is..."    | 1-2 句话                              |
| Naked CRC                 | ASCII call chain + 位置关系图                | mermaid / 邮件草图                    |
| Conversation Scrutiny     | patch review 里 "wait, shouldn't there be..." | LKML / GitHub review comment        |
| "LockingPolicy" 类比      | kernel/locking/ 子系统化                    | `lockdep` / `rt_mutex`               |

### 3.2 机器人软件视角

- **ROS/ROS2 launch 文件 = Naked CRC 的工业版**。一个 launch 文件列了所有 nodes + topics + params——这是 *系统的形状*。新人看 launch 文件 = 看 CRC 卡片。
- **Nav2 的 architecture story = "planner + controller + recovery + BT"**。Nav2 文档第一句就是 4 个 server 的名字——ch17 的工业映射。
- **MoveIt2 的 pipeline = Naked CRC 的代码层映射**。MoveIt2 拆 `PlanningPipeline` / `PlanningScene` / `PlanningRequestAdapter` —— 每个 card 摆在 pipeline 链上 = ch17 cards 摆位置。
- **机器人 multi-node 系统的 *Telling the Story* 提前**。机器人系统天然多进程；新人第一天就要 *讲 launch 文件的故事*——ch17 在 ROS 上的天然落地。
- **ros2_control 的 hardware interface = Conversation Scrutiny 的工业实例**。最初机器人项目把硬件调用直接散落在 nodes；后来团队对话里反复出现 "hardware interface" = 新类被提出来。**这就是 ch17 LockingPolicy 在 ROS 上的版本**。
- **MoveIt! IK solver plugin system = Naked CRC + Telling the Story**。MoveIt 把 IK solver 拆 plugin——一张卡片代表一个 solver 实例；卡片位置 = 调用的链上环节。
- **Nav2 的 BT XML = Naked CRC 的 XML 化**。XML 节点摆出行为树——卡片叠起来表示子节点，位置关系 = BT 节点的 parent/child。

#### 机器人软件 — ch17 技法映射表

| Feathers 技法             | 机器人软件等价做法                          | 工具 / 输出                          |
| ------------------------- | ------------------------------------------- | ------------------------------------ |
| Telling the Story         | Nav2 文档第一句 / ROS launch 文件概览       | "planner + controller + recovery + BT" |
| Naked CRC                 | launch 文件 / BT XML / plugin 列表          | mermaid / rviz graph                  |
| Conversation Scrutiny     | 节点 review "wait, 这个不是 hardware interface 吗?" | ros2_control 设计史                 |
| "LockingPolicy" 类比      | ros2_control 子系统化                        | hardware_interface::SystemInterface  |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                   | 高级工程师                                                   |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 接到陌生系统         | 反复读所有文档                                                | 第一动作 = "讲一遍系统故事"（2-3 句话）                     |
| 看到大块 procedural | "这写得太烂，重写吧"                                          | "它吸引更多 procedural——必须先抽出 LockingPolicy 之类"      |
| 设计新 feature       | 加 method 到现有类                                            | "这让故事更复杂还是更清晰？"——选让故事更接近真的方案       |
| Pair 工作时          | 不画图                                                        | 随手画 cards / 关系图                                         |
| 对话里出现新概念     | 没注意到                                                      | "等等，代码里没有这个类，我们应该抽出来"                   |
| Architect role       | "architect 说了算"                                            | "每个写代码的人都要懂架构，architect 是 facilitator"        |
| Design 时机          | "Sprint 开始时 design 完"                                     | "design 是 continuous activity"                              |
| Refactor 优先级      | 按 LOC 估                                                     | 按"传染性"估——大块 procedural 优先于单点 hack              |
| 架构变更             | 写文档                                                        | 改故事 + 改 launch 文件 + 改文档                            |
| 系统知识沉淀         | wiki / doc                                                    | "told story recently" — 反复口头讲                          |

> **关键差异**：高级工程师把架构当 *团队共同语言*（不断讲故事）；初级把它当 *文档 phase*（写完即结束）。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

仍然极重要。原因：
- **AI 不能"讲故事"**——故事是 *对话*，需要两人都懂。AI 给架构总结 ≠ 团队共同故事。
- **AI 让"design 结束" 的诱惑更大**——AI 给 architecture doc 自动生成，团队觉得"已经写完了"。ch17 警告 *"design is over" 是最坏*——AI 时代这个风险被加剧。
- **Naked CRC 在 AI 时代被异化**——AI 给 mermaid 图代替 cards。mermaid 是 UML 风格静态，丢失 cards 的 *运行时位置* + *可移动* 的对话性。
- **Conversation Scrutiny 在 AI 时代更有价值**——AI 给完整文档，团队更倾向"读 AI 文档"，不再对话。ch17 的核心反而是被边缘化。
- **Telling the Story 的"撒谎"是 AI 给不了的**——AI 给"完整描述"不擅长"故意简化"。ch17 的核心精神（强迫简化）= AI 反模式（AI 默认详细）。

### 4.2 AI 已经能做的（具体到 ch17 主题）

- **生成 architecture summary**：给仓库生成 *1-2 句话* 的故事。**风险**：故事可能不是"团队当前共同的故事"，而是 AI 总结。
- **生成 Naked CRC cards 的 mermaid 版本**：把代码对象画成 mermaid graph。
- **识别对话里的"missing class"**：给 pair 录音 + 转录 + AI 标新概念（但代码没类）。**限制**：transcript 不一定准确。
- **生成 "the story" 不同长度的版本**：1 句话 / 1 段 / 1 页。
- **Telling the Story 的对话模拟**：用 AI 模拟 "你问我答" 的 pair，Telling the Story 演练。

### 4.3 AI 不能替代的（具体到 ch17 主题）

- **替代不了"对话"本身**：Telling the Story 是 *两人对话*——AI 给 story ≠ 团队的故事。
- **替代不了 cards 的物理感**：cards 在桌上 *可移动*——mermaid 是静态的，丢失对话维度。
- **替代不了 conversation scrutiny 的 *现场感***：听 pair 现场对话的能力是 *出席*，不是文本。
- **替代不了"故意简化"的判断**：AI 给 *真描述* 反而多；Telling the Story 要 *故意简化*。
- **替代不了团队共同语言的形成**：故事要被团队 *重复讲*——AI 给一次没用。

### 4.4 AI 经常写错的地方

针对 ch17 "Telling the Story / Naked CRC / Conversation Scrutiny" 主题：

| 错误模式                                       | 例子                                                                 | 后果 |
| ---------------------------------------------- | -------------------------------------------------------------------- | ---- |
| **AI 给 architecture doc 当"团队故事"**         | AI 自动生成 architecture.md，团队觉得"已经讲过架构了"               | Telling the Story 没发生；doc ≠ shared story |
| **AI mermaid 替代 cards 时丢失移动性**           | AI 给出静态 graph，新人不能移动 cards 看不同视角                    | Naked CRC 的 *运行时空间关系* 丢失           |
| **AI 总结里没有"故意撒谎"**                    | AI 输出"完整描述"包括 TestResult interface / reflection 等          | Telling the Story 的核心（强迫简化）反掉   |
| **AI 拒绝抽取新类**                              | AI 看到 "locking policy" 对话，建议 *inline* 进现有函数              | Conversation Scrutiny 反掉                  |
| **AI "识别 missing class" 假阳性**              | AI 把对话里的 *临时口语* 标为 missing class                          | 抽错类 = 设计污染                            |
| **AI 把 design 当 phase**                       | AI 生成 "Sprint planning 后架构完成" 模板                           | 强化"design is over" 反模式                |
| **AI 生成的 architecture summary 不一致**        | AI 给不同人不同 architecture summary                                  | 团队没有共同故事                             |
| **AI 用 doc 替代对话**                          | AI 建议 "用 doc 描述架构比对话高效"                                  | ch17 警告：doc ≠ shared story                |
| **AI 不识别大块 procedural 的"传染性"**         | AI 看 legacy 代码不主动 recommend 抽独立子系统                      | "大块 procedural 黑洞" 没被对抗             |
| **AI 自动抽 class 但缺 conversation 验证**      | AI 抽 `LockingPolicy` 但团队对话里没这词                             | 抽象过度 / 错位                              |

### 4.5 子段: AI 辅助遗留代码理解

适用本会话锁定视角（Linux 系统 + 机器人）。

- **Linux kernel / 系统 daemon 的 ch17 AI 辅助**：
  - AI 给 LKML 邮件 "the story is..." 总结。**风险**：邮件是 *对话*，AI 总结丢掉了对话性。
  - AI 给 kernel 子系统的 mermaid call graph——好用于 onboarding；但 *运行时位置* 仍要人来用 bpftrace 现场画。
- **ROS/ROS2 系统的 ch17 AI 辅助**：
  - AI 给 ROS launch 文件的故事版本：1-2 句话讲整个 launch 启动什么。**好** 用于 onboarding。
  - AI 给 Nav2 architecture summary —— 但 *故意简化* 要人工做（AI 给完整描述多）。
  - AI 给 MoveIt2 pipeline 的 mermaid —— 但 cards 摆位置（运行时）仍要人来 pair work。

### 4.6 工程师必须保留的核心能力

- **Telling the Story 的"故意简化"**：AI 给完整描述容易，强迫 1-2 句话难——这是 ch17 核心技能。
- **Naked CRC 的 *移动性***：cards 在桌上 = 表达"如果这样改，A 的位置变"——UML/mermaid 缺这个维度。
- **Conversation Scrutiny 的 *现场感*：听 pair 现场对话 = 识别 missing class；AI 转录丢掉现场感。
- **拒绝"design is over"**：AI 给 architecture doc 后仍坚持 *反复讲* 故事。
- **识别"传染性"**：大块 procedural 代码是黑洞，需要主动对抗。

## 五、实践行动项

> ch17 是 12 页大章。4 个行动把 3 个技法 + 1 个应用（"design is over" 检测）全做成可跑工具。

### A1 — Telling the Story CLI: 强迫 1-2 句话讲架构

> 复刻 ch17 Telling the Story 的核心动作：给一个项目目录，输出 *1-2 句话* 的架构故事 + 完整描述（让团队选择"先讲哪个版本"）。

```bash
mkdir -p /tmp/ch17-structure && cd /tmp/ch17-structure

cat > story.py <<'PY'
#!/usr/bin/env python3
"""story.py — Telling the Story CLI.
给一个项目目录, 输出:
  - 1-sentence story (Telling the Story 目标)
  - 1-paragraph story (中间粒度)
  - full description (AI 总结风格 — 不推荐)
让团队对比: 1-sentence 版本能否 capture 核心架构.
"""
import os, re, sys, pathlib

def list_main_classes(root):
    """简易: 找 main 函数 / 入口文件 + 顶层类/函数."""
    mains = []
    for f in pathlib.Path(root).rglob("*.c"):
        text = f.read_text()
        for m in re.finditer(r"^\s*int\s+(main_\w+|main)\s*\(", text, re.M):
            mains.append((f.name, m.group(1)))
    for f in pathlib.Path(root).rglob("*.py"):
        text = f.read_text()
        for m in re.finditer(r"^def\s+(main_\w+|main)\s*\(", text, re.M):
            mains.append((f.name, m.group(1)))
    return mains

def list_modules(root):
    mods = []
    for ext in ("*.c", "*.h", "*.py"):
        for f in pathlib.Path(root).rglob(ext):
            mods.append(f.stem)
    return sorted(set(mods))

def list_keyword_hits(root, patterns):
    hits = {p: 0 for p in patterns}
    for ext in ("*.c", "*.h", "*.py"):
        for f in pathlib.Path(root).rglob(ext):
            text = f.read_text()
            for p in patterns:
                hits[p] += len(re.findall(p, text, re.I))
    return hits

def main():
    if len(sys.argv) < 2:
        print("usage: story.py <project_dir>")
        sys.exit(1)
    root = sys.argv[1]
    mains = list_main_classes(root)
    mods = list_modules(root)
    print(f"== 1-SENTENCE STORY (Telling the Story 目标) ==")
    if mains:
        main_names = ", ".join(name for _, name in mains[:3])
        print(f"   '{root}' has these primary entry points: {main_names}.")
        print(f"   (this is intentionally a 'lie of omission' — challenges:")
        print(f"    is the real architecture actually like this?)")
    else:
        print(f"   '{root}' has no obvious main entry — design is opaque.")

    print()
    print("== 1-PARAGRAPH STORY (中间粒度) ==")
    print(f"   modules: {', '.join(mods[:8])}...")
    print(f"   module count: {len(mods)}")

    print()
    print("== FULL DESCRIPTION (AI style — 不推荐作 story) ==")
    patterns = ["vendor_", "fake_", "test_", "init", "process",
                "handle", "manager", "service", "adapter"]
    hits = list_keyword_hits(root, patterns)
    for p, n in sorted(hits.items(), key=lambda x: -x[1])[:8]:
        if n > 0:
            print(f"   {p:15s}: {n} occurrences")
    print()
    print("== RECOMMENDATION ==")
    print("   用 1-sentence version 在每个 Sprint planning 讲一遍.")
    print("   如果 1-sentence 不能 capture 真实架构 = 系统太复杂.")
    print("   ch17: 'If a system isn't as simple as the simplest story")
    print("         we can tell about it, does that mean it's bad? No.'")
    print("         Invariably, as systems grow, they get more complicated.")
    print("         The story gives us guidance.")

if __name__ == "__main__":
    main()
PY
chmod +x story.py

# 用 ch15 的 mail_forwarder.c + ch14 的 vendor_math.c 等试一下
mkdir -p ./demo
# 复制前面 ch14/ch15 sample 到 demo
cp /tmp/ch15-api/mail_forwarder.c ./demo/ 2>/dev/null || echo '// stub' > demo/mail_forwarder.c
cp /tmp/ch14-lib/vendor_math.c ./demo/ 2>/dev/null || echo '// stub' > demo/vendor_math.c
cp /tmp/ch14-lib/calc.c ./demo/ 2>/dev/null || echo '// stub' > demo/calc.c

./story.py ./demo
echo "rc=$?"
```

**验收**：
- 输出 3 个 section: 1-sentence / 1-paragraph / full description
- 1-sentence 版本应该是 *故意简化*——挑战团队"这是真实架构吗？"
- `rc=0`

### A2 — Naked CRC: index cards 在 ASCII 桌上摆对象实例

> 复刻 ch17 Naked CRC 的两条规则：(a) cards 代表 *实例* 不是 class；(b) cards 重叠表示 collection。用 ASCII 输出桌上位置。

```bash
mkdir -p /tmp/ch17-structure && cd /tmp/ch17-structure

cat > crc_cards.sh <<'EOF'
#!/bin/bash
# crc_cards.sh — Naked CRC 的 ASCII 实现.
# 输入: "class_name:instance_count [x y]" (x y = 桌上位置)
# 规则:
#   - 每个 entry 是一个 instance (1 个 class card = 1 个 instance)
#   - 多个 cards 重叠 = collection
# 输出: 桌上的 ASCII 布局

set -e
WIDTH=60
HEIGHT=15
# init board with dots
BOARD=()
for ((i=0; i<HEIGHT; i++)); do
    ROW=""
    for ((j=0; j<WIDTH; j++)); do ROW+="."; done
    BOARD+=("$ROW")
done

place_card() {
    local label="$1" x="$2" y="$3"
    # label 太长截断
    local L=${#label}
    [[ $L -gt 8 ]] && label="${label:0:8}"
    # 把 label 摆到 (x, y)
    for ((i=0; i<${#label}; i++)); do
        local cx=$((x + i))
        local cy=$y
        if [[ $cy -ge 0 && $cy -lt $HEIGHT && $cx -ge 0 && $cx -lt $WIDTH ]]; then
            local row="${BOARD[$cy]}"
            BOARD[$cy]="${row:0:$cx}${label:$i:1}${row:$((cx+1))}"
        fi
    done
}

# 自动计算 x 位置以避免重叠 (cards 不重叠 = Naked CRC 规则 1 的 ASCII 化)
# 用 ASSOCIATIVE array 跟踪每行当前最大 x
declare -A MAX_X

# Parse args: "label:x:y" (多个)
for ARG in "$@"; do
    LABEL=$(echo "$ARG" | cut -d: -f1)
    X=$(echo "$ARG" | cut -d: -f2)
    Y=$(echo "$ARG" | cut -d: -f3)
    # 如果 X < 该行 max_x, 把 X 推到 max_x 之后 (留 1 格间距)
    CUR=${MAX_X[$Y]:-0}
    if [[ $X -lt $((CUR + 1)) ]]; then
        X=$((CUR + 1))
    fi
    place_card "$LABEL" "$X" "$Y"
    NEW_MAX=$((X + ${#LABEL}))
    if [[ $NEW_MAX -gt ${MAX_X[$Y]:-0} ]]; then
        MAX_X[$Y]=$NEW_MAX
    fi
done

# print board
echo "=== Naked CRC table (WIDTH=$WIDTH HEIGHT=$HEIGHT) ==="
for ((i=0; i<HEIGHT; i++)); do
    echo "${BOARD[$i]}"
done
echo
echo "rules:"
echo "  1. cards represent INSTANCES, not classes"
echo "  2. overlapping cards = collection (e.g. multiple sessions)"
EOF
chmod +x crc_cards.sh

# 摆一个实时投票系统 (Feathers ch17 在线投票例子)
# "client session 有 incoming + outgoing 两个 connections"
# 卡片摆出这个空间关系:
./crc_cards.sh \
    "clientA:5:3" "inComA:8:3" "outComA:8:4" \
    "clientB:5:7" "inComB:8:7" "outComB:8:8" \
    "server:25:5" \
    "svSess1:30:3" "inSvS1:33:3" "outSvS1:33:4" \
    "svSess2:30:7" "inSvS2:33:7" "outSvS2:33:8" \
    "voteMgr:42:5"
echo "rc=$?"
```

**验收**：
- 输出 ASCII 桌上布局，cards 在指定 (x, y) 位置
- `clientA` / `clientB` 卡片 + 各自的 in/out connections；`server` 在中间；`svSess1` / `svSess2` 在右侧；`voteMgr` 在最右
- 这就是 ch17 在线投票例子的 ASCII 版本

### A3 — Conversation Scrutiny: 从对话录音转录识别 missing class

> 复刻 ch17 LockingPolicy 例子——对话里出现 "locking policy"，代码里没有，AI 应推荐抽 `LockingPolicy` 类。

```bash
mkdir -p /tmp/ch17-structure && cd /tmp/ch17-structure

cat > scrutiny.py <<'PY'
#!/usr/bin/env python3
"""scrutiny.py — Conversation Scrutiny CLI.
输入: 一段对话文本 (transcript) + 项目目录.
输出: 对话里提到的"概念词" vs 代码里的 class 名对比.
推荐: missing class candidates.
"""
import re, sys, pathlib

# 简易: 名词短语 = CamelCase 1-3 词 OR 少量专门术语 (locking policy / vote manager / session / connection)
CONCEPT_RX = re.compile(
    r"\b([A-Z][a-z]+(?:\s+[A-Z][a-z]+){0,2})\b"
    r"|\b(locking policy|vote manager|session|connection|"
    r"lockingpolicy|testcase|testresult|testsuite)\b",
    re.I)

CLASS_RX = re.compile(
    r"\b(class|struct|typedef\s+struct)\s+([A-Z]\w+)\b")

def concepts(text):
    seen = set()
    for m in CONCEPT_RX.finditer(text):
        # group(1) = CamelCase, group(2) = 专门术语
        word = (m.group(1) or m.group(2) or "").lower().strip()
        if 3 < len(word) < 40:           # 过滤太短/太长的
            seen.add(word)
    return seen

def classes_in_code(root):
    found = set()
    for ext in ("*.c", "*.h", "*.cpp", "*.java", "*.py"):
        for f in pathlib.Path(root).rglob(ext):
            text = f.read_text()
            for m in CLASS_RX.finditer(text):
                found.add(m.group(2).lower())
    return found

def main():
    if len(sys.argv) < 3:
        print("usage: scrutiny.py <transcript_file> <project_dir>")
        sys.exit(1)
    transcript = pathlib.Path(sys.argv[1]).read_text()
    root = sys.argv[2]
    convo = concepts(transcript)
    code = classes_in_code(root)
    print("=== Concepts in conversation ===")
    for c in sorted(convo): print(f"  {c}")
    print()
    print("=== Classes in code ===")
    for c in sorted(code): print(f"  {c}")
    print()
    print("=== Missing class candidates (in convo, not in code) ===")
    missing = convo - code
    for c in sorted(missing): print(f"  {c}")
    if not missing:
        print("  (none — 代码已覆盖对话中的概念)")
    print()
    print("=== Ch17 advice ===")
    if missing:
        print(f"  对话里有 {len(missing)} 个概念在代码里没对应 class.")
        print(f"  推荐: 至少抽 1 个 missing concept 为 class.")
        print(f"  (LockingPolicy 的诞生就是这一观察的产物)")
        return 0  # 有 missing = 好发现, 不是错误
    else:
        print(f"  对话中的概念都已映射到代码 — 没有 obvious missing class.")
        return 0

if __name__ == "__main__":
    main()
PY
chmod +x scrutiny.py

# 准备一段对话 — ch17 LockingPolicy 例子的 transcript
cat > transcript.txt <<'EOF'
Team member 1: So we're going to modify the code to enable a specific locking policy.
Team member 2: We need to make sure resources are locked and unlocked in a particular order.
Team member 1: We can avoid deadlock if we guarantee the order.
Team member 2: I was thinking we maintain counts in arrays to enable this.
Team member 1: Wait, we're talking about a locking policy, right?
Team member 1: Why don't we create a class called LockingPolicy and maintain the counts in there?
Team member 2: We can use method names that really describe what we are trying to do.
EOF

# 准备一个项目目录 — 不含 LockingPolicy (演示 missing class)
mkdir -p ./code
cat > ./code/lock.c <<'EOF'
/* legacy 状态: 没有 LockingPolicy 类 */
#include <pthread.h>
static pthread_mutex_t locks[10];
static int counts[10];
void lock_resource(int id) {
    pthread_mutex_lock(&locks[id]);
    counts[id]++;
}
void unlock_resource(int id) {
    counts[id]--;
    pthread_mutex_unlock(&locks[id]);
}
EOF

./scrutiny.py transcript.txt ./code
echo "rc=$?"
```

**验收**：
- 输出对话里的概念（locking policy / vote manager / session / connection）
- 对比代码里的 class（这里只有 `pthread_mutex_t` 之类，没有 LockingPolicy）
- 推荐 missing class candidates
- `rc=0`

### A4 — "Design is over" 检测: 架构变 stale 的工程信号

> 复刻 ch17 末段警告 —— 团队觉得 design 结束时最危险。给项目目录，输出"design 处于 active 还是 stale"。

```bash
mkdir -p /tmp/ch17-structure && cd /tmp/ch17-structure

cat > design_alive.sh <<'EOF'
#!/bin/bash
# design_alive.sh — "design is over?" 检测.
# 启发式: 看 git history, 文档 age, 新增 abstraction 频率.
# 输出: stale (设计已死) / active (设计仍在演化)

set -e
DIR="${1:-.}"
if [[ ! -d "$DIR/.git" ]]; then
    echo "WARN: $DIR is not a git repo — can't infer design activity"
    exit 1
fi

cd "$DIR"

# 1. 文档 age: 找 ARCHITECTURE.md / DESIGN.md / README.md
echo "=== 1. Architecture doc age ==="
for F in ARCHITECTURE.md DESIGN.md docs/architecture.md README.md; do
    if [[ -f "$F" ]]; then
        LAST_MOD=$(git log -1 --format="%ai" -- "$F" 2>/dev/null || echo "never")
        echo "  $F last modified: $LAST_MOD"
    fi
done

# 2. 是否有 "design doc" 提交? 找 commit msg 含 "design" / "architect" / "refactor"
echo
echo "=== 2. Recent design-related commits (last 6 months) ==="
CUTOFF=$(date -d '6 months ago' +%Y-%m-%d 2>/dev/null || date -v-6m +%Y-%m-%d)
# 防止 head 关闭管道导致 grep SIGPIPE + set -e 终止脚本, 加 || true
git log --since="$CUTOFF" --oneline \
    | grep -iE "(design|architect|refactor|abstraction)" \
    | head -10 \
    | while read L; do echo "  $L"; done || true
DESIGN_COMMITS=$(git log --since="$CUTOFF" --oneline \
                 | grep -ciE "(design|architect|refactor|abstraction)" || echo 0)
echo "  total design-related commits (last 6 months): $DESIGN_COMMITS"

# 3. 代码增长率 — 文件数 / LOC 增长趋势
echo
echo "=== 3. Code growth (last 30 days) ==="
# 同样防 SIGPIPE: 用 || echo 0 而不是直接 wc
COMMITS_30D=$(git log --since="30 days ago" --oneline 2>/dev/null | wc -l || echo 0)
echo "  commits in last 30 days: $COMMITS_30D"

# 4. 诊断
echo
echo "=== 4. Diagnosis ==="
if [[ $DESIGN_COMMITS -eq 0 && $COMMITS_30D -gt 20 ]]; then
    echo "  WARN: 30+ commits/month but 0 design-related — design is stale!"
    echo "  (ch17: 'One of the worst mistakes a team can make is to feel that"
    echo "   design is over at some point in development.')"
    exit 2
elif [[ $DESIGN_COMMITS -gt 0 ]]; then
    echo "  OK: design-related commits exist — design is active"
    exit 0
else
    echo "  INCONCLUSIVE: not enough signal"
    exit 1
fi
EOF
chmod +x design_alive.sh

# 准备一个 "design is over" demo 仓库
rm -rf demo-repo
mkdir demo-repo && cd demo-repo
git init -q
git config user.email "you@x" && git config user.name "you"
echo "# Demo project" > README.md
git add . && git commit -q -m "initial"

# 加 50 个 commit 但都不提 design/architect
for i in $(seq 1 50); do
    echo "// feature $i" >> main.c
    git add main.c && git commit -q -m "feat: add feature $i" > /dev/null
done

cd ..
./design_alive.sh ./demo-repo
echo "rc=$?"
echo
echo "(rc=2 = WARN, design is stale — ch17 警告)"
```

**验收**：
- demo-repo 触发 `rc=2`（30+ commits 但 0 design-related）
- 输出 ch17 引用的"design is over"警告
- 这就是 ch17 末段警告的工程检测版

## 六、值得深入思考的问题

### Q1: Telling the Story 是不是过度简化到失真？

Feathers 承认故事是"故意撒谎"。**问**：当 1-2 句话讲不清架构时，团队应不应该 *拒绝* 简化、坚持更多细节？还是这种"讲不清" 反而是 *好信号*（说明架构确实复杂）？ch17 的"撒谎有价值"是不是太乐观？

### Q2: Naked CRC 在远程团队里怎么 work？

cards 在桌上 = 物理感。**问**：远程团队用 Excalidraw / FigJam 替代时，是不是丢失了 *可移动性* 反而退化成 mermaid 图？有什么工作流能保留 cards 的"移动"动作？

### Q3: Conversation Scrutiny 是不是依赖 *高水平对话*？

ch17 LockingPolicy 例子里团队的对话用了"locking policy" 这个 *工程语汇*。**问**：如果团队对话用的是 *业务语汇*（"那个上锁的东西"）而不是 *工程语汇*（"LockingPolicy"），scrutiny 还能识别 missing class 吗？是不是要先 *教育团队* 用工程语汇对话？

### Q4: 当 *Architect 角色缺失* 时 Telling the Story 谁讲？

ch17 警告 architect 不能独占架构。但 *实际上* 大多数团队没有足够 senior 的人能讲 1-2 句话的架构。**问**：这是不是说明 *大部分团队的架构其实不可讲*？怎么办？找 consultant？还是先承认 *不可讲* 然后逐步建立 *可讲* 的能力？

### Q5: AI 时代 Telling the Story 的"故意撒谎"是不是更难？

AI 给 architecture doc 倾向于"完整"而非"故意简化"。**问**：团队用 AI 写 architecture doc 时，是不是应该 *强制 prompt* "用 1-2 句话讲"？这种 prompt 规约能形成团队文化吗？

### Q6: Naked CRC 是不是有 *时代局限*？

1980s CRC 是为了解 *对象思维* 给 OO 新人。**问**：2025 年工程师已经默认 OO 思维，CRC 还有什么独特价值？是不是 CRC 的 *空间感* 已经被 *时序图 / 序列图* 取代？还是说 cards 的"可移动"维度仍有不可替代之处？

### Q7: "Design is over" 在 *敏捷/Scrum* 文化里是不是必然发生？

Scrum 把工作切成 Sprint，每个 Sprint 强调 *完成*。**问**：Scrum 文化是不是强化了"design 结束 = 进入 Sprint" 的反模式？Agile community 有没有反制？这是不是 Lean Software Development 强调 *"decide as late as possible"* 的另一面？

### Q8: Telling the Story 与 *Architectural Decision Records* (ADR) 的关系

ADR 是写下来的设计决策。**问**：Telling the Story 是 *口头的* 共同语言，ADR 是 *书面的* 决策记录。两者互补还是冲突？是不是该 *每次 Telling the Story 都对应一份 ADR*？或者 ADR 反而让 Telling the Story 失去 *对话* 维度？

### Q9: Conversation Scrutiny 在 *异步工作* 时代怎么 work？

Scrum / GitHub 时代很多 pair 工作是异步（PR comment / async standup）。**问**：异步对话里 *新概念* 怎么识别？是不是 async scrutiny 需要 *周会回顾* 而不是 *现场听*？这种异步版 scrutiny 效果会减弱吗？

### Q10: 当系统 *真的没有故事可讲* 时怎么办？

ch17 给的技法都假设 *有故事可讲*。**问**：如果系统实在太混乱、连 1-2 句话都讲不清，团队应该 *停止 adding features* 专做架构探索吗？还是边 add features 边 *逐步构建* 可讲的故事？这是不是 *重写 vs 演进* 的工程取舍？

---

*下一章预告*: **Chapter 18 — My Test Code Is in the Way** — ch14-17 是 *理解 + 拆依赖 + 隔离 vendor + 写测试网* 的 4 章铺垫；ch18 转到一个具体痛点 —— *测试代码本身成了障碍*。当测试 fixture 越来越复杂、test helper 越来越厚、test setup 越来越长，测试代码开始 *拖累 production code* 而不是保护它。ch18 是 ch2 Software Vise 的"测试自身的 maintenance cost" 那一面。

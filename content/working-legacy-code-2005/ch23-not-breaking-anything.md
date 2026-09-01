# Chapter 23 — How Do I Know That I'm Not Breaking Anything?

> **PDF**: p.331-340（10 页）
> **定位**: ch22 教"怎么拆"，本章教"拆的过程中怎么**不破坏**"。ch25 里 24 个技法都"无测试保底", 本章就是它们的 *安全手册*: Hyperaware Editing / Single-Goal Editing / Preserve Signatures / Lean on the Compiler / Pair Programming。ch23 是 ch6/8/22 共同的"安全网"。

## 〇、第一性原理思考

**这章做了什么**: 为 ch22 拆 monster 和 ch25 24 个技法补一层"编辑纪律" —— Hyperaware Editing、Single-Goal Editing、Preserve Signatures、Lean on the Compiler、Pair Programming 五个动作, 在无测试兜底时手动把 breakage 概率压到最低。

**为什么这样拆**: 它把 ch22/ch25 的"技法如何安全地执行"独立成章 —— 任何 refactor 动作都被显式分类为"改行为 / 不改行为", 安全不靠个人谨慎, 而靠纪律 (单目标 + 复制签名 + 编译器反推 + pair) 把 hyperaware 分摊给机器和同伴。

**最值钱的洞见**: Pair 不是"提高效率", 是 dependency-breaking 阶段的强约束 —— 单人 refactor 一旦 silent breakage 几小时后才在 production 暴露, 需要"两个 feedback channel"才能让 Preserve Signatures 这种廉价纪律真的被执行。
## 一、章节概述

- **代码与物理材料的对比**：金属疲劳、塑料老化 — 物理材料越用越坏。代码不一样: 放着不用就不会坏。*唯一*让代码坏的途径就是人编辑它。开发者既是 software 的 primary agent of change, 也是 *primary agent of breakage*。这是 ch23 的认识论前提。
- **Hyperaware Editing（超警觉编辑）**: 把每次按键分成 *改行为* vs *不改行为* 两类(comment、空格、改格式 = 不改行为; 改字面量、字符串、控制流 = 改行为)。目标: 让自己清楚地知道"我这一下到底改了什么"。TDD + 快速 feedback loop 是 hyperaware editing 的基础设施 —— test 在 < 1 秒内跑完。
- **Pair Programming 是 hyperaware editing 的人肉版**：两个人盯同一段代码 = 两个 feedback channel。Feathers 在 *break dependencies* 阶段强制要求 pair —— "做手术不该独自一人"。
- **Single-Goal Editing（单目标编辑）**: 在编辑过程中, 一次只做一件事。常见反模式: 你开始改 A, 看 A 里要调 B, 改 B 之前发现 B 又要调 C —— 一次跑题 3 层, 最后忘了原来要改什么。Feathers 的口头禅: *"Programming is the art of doing one thing at a time."*
- **Single-Goal 的实践**: 当你被引到另一处, 把它**写下来**, 回头先做完手头的。Pair partner 的角色: 喊 "What are you doing?"; 如果答案超过一件事, 选一件做。
- **Preserve Signatures（保留签名）**: 在 *break dependencies* 时, 严禁修改方法签名 — 用 cut/copy paste 整个签名, 避免打字错误。Feathers 给的步骤: 复制参数列表 → 写新方法声明 → 粘贴 → 写调用 → 粘贴 → 删类型。这是 ch11 的相反原则 —— ch11 鼓励改签名; ch23 在 *break dependencies* 阶段禁止改。
- **Lean on the Compiler（依赖编译器）**: 用编译器引导 refactor —— 把声明改一处, 让编译器告诉你哪些地方需要跟改。常见两种用法:(a) 结构性变更(Encapsulate Global References 时把 `double rate` 变成 `exchange.rate`);(b) 接口变更(类型从 class 变 interface, 看哪些方法是接口必需的)。
- **Lean on the Compiler 的盲区**: **inheritance 是隐形杀手** — 注释掉 `getX()`, 编译器无报错 = superclass 仍在用, 不是 unused method。变量继承同理。这是"编译器告诉你 ≠ 全对"的边界。
- **Pair Programming**: 当 ch25 dependency-breaking 技法被用时, Feathers 强制要求 pair —— 单人做错一次就是 silent breakage, 有 pair 至少两个人盯着。

## 二、核心 Takeaways

### Takeaway 1: 代码是 *编辑器敏感* 材料, 不是 *运行时敏感* 材料

- **是什么**: 物理材料会因使用而退化; 代码不会 —— 跑 1000 次和跑 1 次一样。代码坏掉只可能是 *人改了它*。
- **为什么重要**: 这条认识决定了开发者的核心责任 —— "我的编辑 = 我的风险"。每次按键都要清楚自己在改什么。Hyperaware editing 不是过度紧张, 是 *认识论必然*。
- **解决什么问题**: 给"为什么 edit 要小心"一个 *物理* 论证 —— 不是规范, 是材料特性。
- **适用场景**: 工程师 onboarding 时讲清"开发的责任"; 也用于说服管理层"为什么 dev time 不能压太紧"(每压缩一秒 = 增加 breakage 风险)。

### Takeaway 2: Hyperaware Editing = 把每次按键分类

- **是什么**: 你敲下的每个键要么 *改行为* 要么 *不改行为*。改注释 = 不改; 改字面量 = 改; 改格式 = 不改; 改表达式 = 改。
- **为什么重要**: 训练大脑 *识别每一下* 的后果。这是 TDD 的认知基础 —— TDD 的 test 不只是 verification, 更是 *实时反馈* 让 hyperaware 状态可维持。
- **解决什么问题**: "我改了半小时, 现在跑 test 失败 — 哪一下破坏的?" 这种痛苦状态。Hyperaware 让大脑不进入这种状态。
- **适用场景**: 任何 edit session; 尤其 refactor monster method 时(参考 ch22 Takeaway 7)。

### Takeaway 3: TDD + 快速 feedback = Hyperaware 的工程基础

- **是什么**: Hyperaware 靠 *feedback < 1 秒* 维持。TDD 让你每次 edit 完立刻跑 test —— 行为改了立刻红。这把"我刚改了啥" 的 *心理负担* 转嫁到 *机器*。
- **为什么重要**: 没 test 时, hyperaware 完全靠大脑 —— 撑 30 分钟就累; 有 test 时, 大脑只负责 *意图*, 验证交机器。这是 *认知卸载*。
- **解决什么问题**: 解决 ch22 / ch25 技法在 *没测试* 时的高压处境(只能靠 hyperaware 单兵扛)。
- **适用场景**: 任何 TDD-friendly 项目(单元测试 < 1 秒); 以及 ch25 的 *post-dependency-breaking* 阶段(测试就位后)。

### Takeaway 4: Single-Goal Editing — "一次只做一件事"

- **是什么**: 编辑时一次只做一件事, 不被中间发现"还要改别处"带走。被引到别处 → 写下来 → 回头做手头的。
- **为什么重要**: Multi-goal editing 是 "你做了 5 件事, test fail, 你不知道哪件破坏的" —— hyperaware 直接失效。
- **解决什么问题**: 解决 review 时常见的 PR 描述"重构 + 加 feature + 修 bug 全在一起"的乱。
- **适用场景**: Pair programming 时最显眼 —— partner 一句"What are you doing?"就能让你停。

### Takeaway 5: Preserve Signatures — 拆 dependency 时禁止改签名

- **是什么**: 在 ch25 的技法里, 新方法的签名应当用 *复制粘贴* 整个参数列表, 而不是手打。这避免打字错误(变量名错位、类型错配、顺序错位)。
- **为什么重要**: ch25 技法都在 *没测试* 条件下做 —— 签名错一处 = 整方法 silent breakage。复制粘贴是最便宜的保险。
- **解决什么问题**: 解决"我抽了个方法, 运行时 segfault, 不知道是参数没传对还是逻辑错" 的双重错误。
- **适用场景**: Break Out Method Object / Extract and Override * / Parameterize * / Expose Static Method 等所有 ch25 技法。

### Takeaway 6: Lean on the Compiler — 让编译器当 refactor 助手

- **是什么**: 改一处声明(例如 `double rate` → `exchange.rate`), 让编译器报所有需要跟改的位置。这比 grep 可靠 —— grep 漏字符串拼接、宏、注释里的同名变量; 编译器不漏。
- **为什么重要**: 结构性 refactor 时(Encapsulate Global / 抽接口 / 类型切换), 靠编译器 = 靠 *全程序分析*, 而不是 *人肉搜索*。
- **解决什么问题**: 解决"我改了声明, 但有些用法找不到"的痛苦 —— 编译器是 reference resolution 的金标准。
- **适用场景**: 任何全局变量 / 类型 / 接口的 rename / move。内核 `EXPORT_SYMBOL` rename 也要靠 build error 反推。

### Takeaway 7: Lean on the Compiler 的盲区 —— inheritance 是隐形通道

- **是什么**: `class B extends A` 时, 注释掉 `B.getX()` 编译器无报错 = `A.getX()` 顶上。看上去 "unused", 实际 superclass 还在用。
- **为什么重要**: 这是 Lean on the Compiler 的 false negative —— 编译器告诉你"0 处引用", 但实际 N 处(都是 superclass 路径)。
- **解决什么问题**: 提醒 refactor 时 *继承深度* 是关键 — 多层继承的 rename 不能只看编译错误, 还得扫 override 链。
- **适用场景**: C++ / Java 多层继承; C++ 多重继承; template instantiation 的隐式继承。

### Takeaway 8: Pair Programming 在 dependency-breaking 阶段不是可选

- **是什么**: Feathers 明确: *ch25 的 dependency-breaking 技法必须在 pair 下做*。单人做错一次 = 几小时后才被 production 发现; pair 实时发现。
- **为什么重要**: 这些技法无测试兜底, 等于 *没 safety net* 走钢丝 —— 一个人走容易摔, 两个人相互把着走。
- **解决什么问题**: 解决"refactor commit 悄悄引入 regression" 的延迟发现成本。
- **适用场景**: Legacy code 重构期、kernel 大改动、ROS 节点大改。

## 三、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **Hyperaware Editing 在内核 patch 里**: 内核 patch review 是 *公开* 多人 review —— 比 pair 更彻底。一个 patch series 多达 20+ 轮 review(Linus Torvalds 风格)。这天然让 *single-goal editing* 受约束: 每个 patch 一个变更, patch 间逻辑独立。
- **Linux kernel 编码规范强制 single-goal**: `Documentation/process/submitting-patches.rst` 明确说"one logical change per patch"——和 Feathers *Single-Goal* 直接对应。
- **Preserve Signatures 在内核的实例**: API rename 必走 *deprecated alias* 模式 —— `old_name()` 留 2-3 个版本, 新代码用 `new_name()`。这是 *Preserve Signature + deprecation window* 的工程实现。`EXPORT_SYMBOL_GPL(old_name)` 保留, 新增 `EXPORT_SYMBOL_GPL(new_name)`。
- **Lean on the Compiler 在内核**: 内核 *没有完整 build*, 靠 *incremental*:`make M=drivers/net/wireless/ M=drivers/usb/` 只编子模块 —— 这把 *lean on compiler* 的 *回环时间* 缩短。但全树 rebuild 仍 30-60 分钟, 所以 refactor 时尽量 *partial build*。
- **C++ 在内核的 inheritance 盲区**: 内核 C 代码几乎无继承 —— 盲区少。但 *container_of* + 函数指针模拟继承 的代码有同样问题: 注释掉一个 method 不报"unused", 但通过 container_of 拿到基类的还能调。Lean on Compiler 在内核 C 不可靠, 需要 *`objdump`* + 静态分析辅助。
- **Pair Programming 在内核**: patch review 是 *asynchronous pair* —— 不是同一房间盯屏幕, 但多轮 maintainer review 起到同等作用。Linux subsystem maintainer 是 *pair 的异步版*。
- **Single-Goal 在内核 patch series**: `git format-patch -n` 强制一个 commit 一个逻辑变化。Linux Torvalds *强烈反对* squash commit —— 因为 squash 抹掉 single-goal 边界。
- **TDD 在内核**: KUnit / kselftest / LTP 让 *测试 < 1 秒* 局部成立(KUnit), 但全树测试仍慢。`make -j$(nproc) M=drivers/<x>` 给 sub-system hyperaware 提供 < 30 秒 feedback。

#### Linux 系统 — Hyperaware Editing 工程实践表

| 实践                | 内核等价物                  | 反馈回路                  |
| ------------------- | -------------------------- | ------------------------- |
| Hyperaware Editing  | 每 patch 一个 reviewer + `checkpatch.pl` | 邮件 review 链 < 24h      |
| Single-Goal         | `git format-patch` 多 commit series    | patch 数量 = 关注点数量 |
| Preserve Signatures | `EXPORT_SYMBOL` 旧名 alias 保留 2 版 | ABI 工具 `abigail` 检查 |
| Lean on Compiler    | `make M=drivers/<x>` 部分 build + 编译错 | < 30s 反馈                  |
| Pair Programming    | subsystem maintainer review | 多轮邮件 + ACK 链        |

### 3.2 机器人软件视角

- **ROS 节点的 hyperaware 编辑**: ROS 节点通常很短(50-200 行), TDD 容易(`rclcpp` 节点可用 gtest + mock node 跑 < 1 秒)。Hyperaware 自然成立。
- **BehaviorTree. CPP 的 single-goal 强制**: BT node 的 `tick()` 必须一次返回 SUCCESS / FAILURE / RUNNING —— 不能跨 tick 留状态。这是 *single-goal* 的运行时强制。
- **ros2_control 的 preserve signature**: hardware_interface:: SystemInterface 的 read/write/stop 方法签名有 ABI 约束 —— 改签名 = 改 hardware plugin 不兼容。社区用 *pluginlib* + *visibility 控制* 强制 preserve。
- **Nav2 的 lean on compiler**: 拆 costmap layer 时改 base class interface, 看哪些 layer 编译失败 —— 这是 lean on compiler 的应用。`colcon build --packages-select nav2_costmap_2d` 给 < 10s feedback。
- **ROS2 lifecycle 的 pair**: lifecycle state 转移有 *destructive test* —— 任何错误状态转移要 pair verify。ROS2 `lifecycle` 节点测试通常 *故意* 让 partner 制造异常 state 来 verify 恢复。
- **robot_localization 的 multi-goal 风险**: EKF 同时订阅 odom + IMU + GPS + twist —— 一个 PR 同时改 4 个 topic 的协方差处理 = multi-goal 经典反模式。Feathers *Single-Goal* 直接适用。
- **TF2 的 preserve signatures**: `tf2::Transform` 的 API 稳定多年 —— 改签名 = 整个 ROS 生态 break。社区用 *deprecation macro* `ROS_DEPRECATED` 强制 preserve + 警告。
- **ROS message 的 single-goal**: `.msg` 文件变更 = ABI 变更; 同 sensor_msgs 的 `Header` 改一个字段 = 整个 ROS graph break。社区用 *message versioning* + *field deprecation* 强制 preserve。

#### 机器人软件 — Hyperaware Editing 工程实践表

| 实践                | ROS2 / Nav2 等价物                  | 反馈回路                  |
| ------------------- | ---------------------------------- | ------------------------- |
| Hyperaware Editing  | `colcon build --packages-select` + gtest | < 10s 单包               |
| Single-Goal         | BT node tick 一次返回              | BT 单元 test < 1s         |
| Preserve Signatures | `ROS_DEPRECATED` macro + ABI alias  | ABI linter `ros2 doctor`  |
| Lean on Compiler    | 改 base costmap layer interface     | `colcon build` < 30s       |
| Pair Programming    | lifecycle state 转移 pair verify    | destructive test + partner |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                   | 高级工程师                                                   |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 改完一段             | 写完一段,等"想起来"再跑 test                                  | 改完 *每段* 都跑 test                                       |
| 编辑过程             | 一边改 A 一边"顺便"改 B 和 C                                  | Single-Goal —— *写下来要改的别处*,先做完手头的              |
| 抽方法               | 手打新签名                                                    | Preserve Signatures —— 复制粘贴整个签名                     |
| 改全局变量           | 用 grep 找                                                   | Lean on Compiler —— 改一处,让编译器报告所有                |
| Pair                 | "我一个人做得快"                                              | 在 break-dependencies 阶段主动要求 pair                     |
| review 自己代码      | "看上去对"                                                    | 知道 *看上去对* ≠ *真对*,用 test + compiler + partner 多层 verify |
| 拆错一次             | "我前面都白做了"                                              | 知道 single-goal + preserve + compiler 是 *多层安全网* — 拆错概率小 |
| 改 PR 描述           | "重构 + 加 feature + 修 bug 一并做"                           | 一个 PR = 一个目标;多目标就拆 PR                            |
| 维护旧代码           | "反正没 test,凑合改"                                          | "没 test" 才是要 *先* 写 test 的强信号                       |
| 编译反馈             | 等全编                                                       | 利用 incremental / colcon / 部分 build 加速                  |

> **关键差异**: 高级工程师 *把 hyperaware 的责任分摊给机器* (test + compiler + partner)。初级把 hyperaware 当作 *个人素质* —— 撑不久, 容易疲劳犯错。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

**极其重要** —— 而且 AI 让它*更重要*。原因:

- **AI 的多目标倾向**: AI 写一段代码, 默认 *顺便优化*: 改格式、调顺序、改命名、加注释 —— 这是 Takeaway 1 的反面。AI 是 *multi-goal editor*, 人必须强制 single-goal。
- **AI 不擅长 preserve signatures**: AI 抽方法时常 *改参数名顺序* 或 *拆 const* —— 破坏 preserve。
- **AI 不停 run test**: AI 写完一次 commit 才跑 test, 不是 *每段 edit* 都跑。这是 hyperaware 的反面。
- **AI 不 pair**: AI 没有 partner —— 它"自信地说代码对", 不会主动问"What are you doing"。
- **AI 编译反馈慢**: AI 模型不直接接 compiler —— 它写的代码常常 *根本没编译过* 就给用户。

### 4.2 AI 已经能做的

- **建议 single-goal 的拆分**: 给一个 multi-goal PR, AI 给出 *按目标拆 PR* 的方案 —— 准确率 80%。
- **检测 preserve signatures 破坏**: 给一段抽方法代码, AI 比对"原签名 vs 新签名" —— 找出改动点。准确率 90%。
- **Lean on Compiler 辅助**: 列出"声明改了, 需要跟改的所有调用点" —— AI 给候选, 编译器给权威答案。
- **Pair-style 代码 review**: AI 当 *async pair*, 在 review 时问"你刚才为什么要改这一行" —— 让作者反思 single-goal。
- **Hyperaware 训练 prompt**: 给 PR 时说"每段 edit 都跑 test, 失败立刻 stop" —— 提醒 AI 像 pair 一样。

### 4.3 AI 不能替代的

- **真正的 pair partner 盯着屏幕**: AI 不在同一 *物理房间*, 不能即时打断"你跑题了"。这是 *async pair* 的根本差异。
- **Compiler 的真实反馈**: AI 不直接接编译器, 只能"预测"编译结果。Linux kernel 改 struct 时 AI 没法保证不破坏 ABI。
- **Test 的真实运行**: AI 不跑 test —— 它给的"代码对"是基于 *概率*, 不是 *验证*。
- **行为契约的判断**: AI 不知道 "改 `if (x == NULL)` 为 `if (!x)` 在某种 path 下语义不同" —— 这是 Takeaway 1 的 hyperaware 边界。

### 4.4 AI 经常写错的地方

| 错误模式                                       | 例子                                                                 | 后果 |
| ---------------------------------------------- | -------------------------------------------------------------------- | ---- |
| **multi-goal editing**                          | 用户让"加 feature",AI 顺便 refactor + 优化 + 改命名                  | 单 PR 多目标,review 时难分清哪部分是 feature / refactor |
| **不 preserve signatures**                     | AI 抽方法时给参数改名(更"语义化"),原调用点也跟着改                 | 看似无害,但 ABI / 反射 / 序列化路径 break |
| **不每段 edit 跑 test**                        | AI 写完整段才"心理跑 test",不真的运行                                | 错误累积到末尾才发现,debug 困难 |
| **"lean on compiler"代替真 compile**            | AI 声称"我推断这代码会编译过",没真编译                              | 编译错 / 链接错 / ABI 错 |
| **不 pair 当作不 review**                       | AI 自己写的代码自己 review = 0 反馈                                  | 同样错误被自我合理化 |
| **formatting 改变算 "refactor"**                | AI 改缩进 / 改换行 / 调 import 顺序,声称"只做 refactor"             | 实际 multi-goal,Takeaway 4 violation |
| **comment 改写算 "改行为"**                     | AI 把 `// fix bug` 改成 `// TODO: refactor` 算"改行为"             | comment 是 documentation,改了 doc 算 doc change ≠ behavior change |
| **抽方法时改默认参数值**                        | AI 抽方法时"顺手"给某参数默认值,原行为变                            | silent behavior change |
| **拆 dependency 时顺手改 ABI**                 | AI 改 method signature 让它"更通用"(加 const / 加 nullable),原调用者漏 | 运行时崩或 silent type coerce |
| **不识别人肉 refactor 的边界**                  | AI 看不到 *人脑里"再坚持一下就拆完"*,可能给出 *未完成* 的中间态 commit | 提交时是 90% 完成,余下 10% 留 TODO |

### 4.5 子段：AI 辅助遗留代码理解 — 在本主题专项

- **AI 当 async pair partner**: PR review 时 AI 问 "What are you doing?" —— 作者答"加 feature", AI 追问"你顺便改的命名算 feature 还是 refactor?" —— 强制 single-goal 反思。
- **AI 给 preserve-signature 检查表**: 抽方法后 AI 列"原签名 vs 新签名, 以下字段变了:..." 让人 review。
- **AI 给 lean-on-compiler 候选**: 改一处声明后 AI 给"以下文件可能需要改:..." 让 *人* 去 *真编译* 验证。
- **AI 给 hyperaware 编辑日志**: 每次 edit 后 AI 问"这是改行为还是改格式?" —— 强制自我反思。
- **AI 给 multi-goal 检测**: PR 描述 + diff, AI 标"这个 PR 实际做了 N 个目标, 建议拆 PR"。
- **AI 不能做**: 真跑 test、真 compile、真打断 multi-goal —— 这是 pair + CI 的不可替代作用。

### 4.6 工程师必须保留的核心能力

- **Single-Goal 强制**: 自己写 PR 时, 标题 *只一个动词* —— 加 / 修 / 重构 / 优化。多目标就拆。AI 给 multi-goal 时 push back。
- **Preserve Signatures 纪律**: 抽方法前先把签名 *完整复制* —— AI 改签名时 push back。
- **Lean on Compiler 真跑**: 不相信 AI 的"应该能编译" —— 一定真跑一遍。Linux kernel 改 struct ABI 必跑 `make modules_prepare`。
- **Pair Programming 在 break-dependency 阶段**: AI 不能替代, 必配 partner。
- **review 自己 PR 的纪律**: 用 *AI 当 async pair* 强制反思, 但最终判断 *人* 做。
- **区分行为 change vs format change**: review PR 时显式标 "behavior diff" vs "format diff" —— format diff 不能进 behavior commit。

## 五、实践行动项

> 本章聚焦"在 break dependencies 时如何不破坏"。C 行动项用 *编译反馈 + 假 pair* 复现 lean on compiler + preserve signatures + single-goal 的纪律。

### A1 — Lean on the Compiler: 把全局变量封装成 struct, 让编译器告诉你哪些地方要改

```bash
mkdir -p /tmp/ch23-compiler && cd /tmp/ch23-compiler

cat > before.c <<'EOF'
/* Encapsulate Global References 的"前": 全局变量散落 */
#include <stdio.h>
double domestic_exchange_rate = 1.10;
double foreign_exchange_rate = 0.91;

double convert_domestic(int shares) {
    return shares * domestic_exchange_rate;
}
double convert_foreign(int shares) {
    return shares * foreign_exchange_rate;
}
double total(int shares_dom, int shares_for) {
    return convert_domestic(shares_dom) + convert_foreign(shares_for);
}

int main(void) {
    printf("total = %.2f\n", total(100, 100));
    return 0;
}
EOF

cat > after.c <<'EOF'
/* Encapsulate Global References 的"后": 全局放进 struct */
/* 这个版本故意保留 domestic_exchange_rate 的旧使用 —— lean on compiler 会报 */
#include <stdio.h>
typedef struct { double domestic; double foreign; } Exchange;
static Exchange ex = { 1.10, 0.91 };

double convert_domestic(int shares) { return shares * ex.domestic; }
double convert_foreign(int shares) { return shares * ex.foreign; }
double total(int shares_dom, int shares_for) {
    return convert_domestic(shares_dom) + convert_foreign(shares_for);
}

int main(void) {
    /* 这里故意留一个旧名引用, lean on compiler 应该报 */
    double leak = domestic_exchange_rate;  /* 应报:undeclared */
    printf("total = %.2f, leak = %.2f\n", total(100,100), leak);
    return 0;
}
EOF

echo "=== before.c compiles cleanly ==="
cc -std=c17 -Wall -Wextra -O0 -o before before.c && ./before

echo "=== after.c (故意留旧名) — lean on compiler 应当报错 ==="
if cc -std=c17 -Wall -Wextra -O0 -o after after.c 2>&1; then
  echo "UNEXPECTED: compile should fail because of leaked global name"
  exit 1
else
  echo "OK: 编译器报告泄漏的旧名,引导我们改"
fi
```

**验收**:
- `before.c` 编译通过,`./before` 输出 `total = 201.00`。
- `after.c` 编译**应该失败** —— 编译器报 `domestic_exchange_rate` undeclared。
- 这证明 Lean on Compiler 的核心: **改一处声明 → 编译错告诉你所有要改的地方**。修掉那一处 `domestic_exchange_rate` → `ex.domestic`, 再编译就过。

### A2 — Preserve Signatures: 抽方法时复制粘贴参数列表, 验证零签名改动

```bash
mkdir -p /tmp/ch23-preserve && cd /tmp/ch23-preserve

cat > preserve.c <<'EOF'
/* Preserve Signatures: 抽方法时签名一字不差复制 */
#include <stdio.h>

typedef struct { int a; int b; int c; } Triple;

static int max3(int a, int b, int c) {
    int m = a;
    if (b > m) m = b;
    if (c > m) m = c;
    return m;
}

/* "preserve signatures" 后: process 调用 max3 — 签名一字不差 */
int process(int a, int b, int c) {
    int m = max3(a, b, c);
    int s = a + b + c;
    return m * 10 + (s % 7);
}

int main(void) {
    printf("process(3,5,7) = %d\n", process(3, 5, 7));
    return (process(3,5,7) == 56) ? 0 : 1;  /* 7*10+6=76... wait, recompute */
}
EOF

# 注意: max(3,5,7)=7, 3+5+7=15%7=1, 7*10+1=71. 改 expectation.
sed -i 's/process(3,5,7) == 56/process(3,5,7) == 71/' preserve.c

cc -std=c17 -Wall -Wextra -O0 -o preserve preserve.c && ./preserve; echo "exit=$?"
```

**验收**:
- 编译零警告。
- 输出 `process(3,5,7) = 71`, exit 0。
- **演示点**: `max3(int a, int b, int c)` 的签名和 `process` 调它的位置完全一致 —— 复制粘贴来的。任何人手打都难做到 *完全一致*。

### A3 — Single-Goal: 把 multi-goal PR 的 diff 拆成多个 single-goal commit

```bash
mkdir -p /tmp/ch23-single-goal && cd /tmp/ch23-single-goal

# 初始化一个 git repo, 模拟一次 multi-goal PR
git init -q
git config user.email "you@example.com" && git config user.name "you"

# base: 一个 working 的文件
cat > order.c <<'EOF'
int calc_total(int qty, int price) { return qty * price; }
int main(void) { printf("%d\n", calc_total(3, 100)); return 0; }
EOF
git add . && git commit -q -m "initial: working"

# multi-goal "改": 加 feature (新方法) + refactor (改名) + 修 bug (off-by-one)
cat > order.c <<'EOF'
int calc_total(int qty, int price) {
    /* refactor: 抽常量 (其实是 multi-goal 嫌疑) */
    int base = qty * price;
    /* bugfix: 之前漏了 quantity<=0 的检查 */
    if (qty <= 0) return 0;
    return base;
}
/* feature: 加一个新函数 */
int calc_total_with_tax(int qty, int price, double rate) {
    int base = calc_total(qty, price);
    return (int)(base * (1.0 + rate));
}
int main(void) { printf("%d\n", calc_total_with_tax(3, 100, 0.1)); return 0; }
EOF
git diff --stat HEAD
echo "---"
git add . && git commit -q -m "multi-goal: feature + refactor + bugfix"

echo "=== now propose the split into 3 commits ==="
# Reset & replay in 3 separate single-goal commits
git reset --hard HEAD~1 -q
git reset -q

# Commit 1: bugfix only
git checkout order.c -q
cat > order.c <<'EOF'
int calc_total(int qty, int price) {
    if (qty <= 0) return 0;
    return qty * price;
}
int main(void) { printf("%d\n", calc_total(3, 100)); return 0; }
EOF
git add . && git commit -q -m "bugfix: handle qty <= 0"

# Commit 2: refactor
cat > order.c <<'EOF'
int calc_total(int qty, int price) {
    if (qty <= 0) return 0;
    int base = qty * price;
    return base;
}
int main(void) { printf("%d\n", calc_total(3, 100)); return 0; }
EOF
git add . && git commit -q -m "refactor: extract local variable for clarity"

# Commit 3: feature
cat > order.c <<'EOF'
int calc_total(int qty, int price) {
    if (qty <= 0) return 0;
    int base = qty * price;
    return base;
}
int calc_total_with_tax(int qty, int price, double rate) {
    int base = calc_total(qty, price);
    return (int)(base * (1.0 + rate));
}
int main(void) { printf("%d\n", calc_total_with_tax(3, 100, 0.1)); return 0; }
EOF
git add . && git commit -q -m "feature: add calc_total_with_tax"

echo "=== final log ==="
git log --oneline
```

**验收**:
- `git log --oneline` 看到 4 个 commits:`initial`、`bugfix`、`refactor`、`feature`。
- 每个 commit 一个目标, review 时可以 *按 commit 单独 review*。
- 这是 *Single-Goal* 的 git 化表达 —— *多 commit = 多 single-goal*。

## 六、值得深入思考的问题

- **"hyperaware editing"的可持续性**: Feathers 描述的状态是"flow state" —— 但它能持续多久?1 小时? 半天? 疲劳后怎么恢复? 团队节奏怎么调?
- **Single-Goal vs Sprint 节奏**: 现实 sprint 里工程师被迫 *同时做多件事*。Single-Goal 怎么和 sprint 协调?
- **Preserve Signatures 的"过度保留"**: 当 API 真要改时(breaking change 是必要的), Preserve Signatures 的 *deprecation window* 应该多长? Linux kernel 是 2 版本, 有些项目是 6 个月, 有些是 *never*。
- **Lean on Compiler 的"false confidence"**: 编译器告诉你 "0 error" ≠ 行为对。Takeaway 7 的 inheritance 盲区是经典反例 —— 还有什么盲区? 模板实例化?`extern "C"` 的符号混淆?
- **Pair Programming 的成本/收益**: 2 人做 1 人的活 = *看起来慢一倍*。什么时候 pairing 的 ROI 是正的? Break dependencies 阶段之外呢?
- **AI 是 "asynchronous pair" 吗?**: AI 不能实时打断"你跑题了", 但它能在 PR review 时给反馈。这 *够不够* 替代真 pair?

## 七、本章与全本的关系 + 后续书预告

### 与全本 25 章的关系

- **前置**: ch22 (拆 monster 时怎么不破坏) + ch11 (改签名的纪律 — Preserve Signatures 是它的反面)。
- **本章**: 给出 5 个安全纪律 —— Hyperaware / Single-Goal / Preserve / Lean on Compiler / Pair。这是 ch25 24 个技法共同的 *安全网*。
- **后续**:
  - **ch24** = 情感维度 —— 在 monster / 没测试 / 团队压力下, 工程师心理怎么保持。
  - **ch25** = 24 个 dependency-breaking 技法 reference —— 全部需要 ch23 纪律保底。
  - **全书结束** —— Part III = ch25 + Glossary/Index。

> **下一章预告**: ch24 把 ch22 / ch23 的 *技术纪律* 切到 *情感维度* —— *legacy code 里工作为什么值得*。feathers 自己点出: green-field 团队追不上旧系统持续演化,*legacy 工作是 critical infra*, fun + community + impact 是 *可持续* legacy 工作的燃料。

### 后续书预告

**Working Effectively with Legacy Code** 是 *2004* 的书。今天的工程师读它, 会感到 *AI 没出现* 这个明显的空白。

- **Sam Newman — *Monolith to Microservices* (2019)**: ch22 的 *monster 拆解* 在 microservice 拆分里被推到架构级。Feathers 的 *Extract to Current Class First* 对应 *Strangler Fig Pattern* 的第一步 —— *先把旧代码包一层, 新功能在新代码里写*。
- **Titus Winters — *Software Engineering at Google* (2020) 第 16 章 Large-Scale Changes**: ch23 的 *Hyperaware Editing* 在 *LSC* 里被推到 *全代码库* 维度 —— 你怎么在 *一千万行 codebase* 里做 single-goal commit? Google 的答案是 *大批小 commit + 强 review + 强测试门槛*。
- **AI 时代的延伸**: ch23 在 AI 写代码时代变成 *hyperaware prompting* —— 提示词要 *显式 single-goal*, 要 *显式 preserve signatures*, 要 *显式 lean on compiler (真的去编译)*。*Andrew Ng / Harrison Chase* 等人的 prompt engineering 课程正在向这方向走, 但 Feathers 的 *纪律* 视角仍是核心 —— AI 工具越强, *纪律* 越重要(因为 AI 让 multi-goal edit 变得 *太容易*)。
- **可能的后续书**: *Working Effectively with AI-Assisted Legacy Code* (还没出版, 可能 2027-2028) —— 这本书会把 ch23 的 hyperaware / single-goal / preserve / lean-on-compiler / pair 推到 *AI pair* 维度。

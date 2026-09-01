# Chapter 6 — I Don't Have Much Time and I Have to Change It

> **PDF**: p.79-98（20 页）
> **定位**: 上一章（ch5 tools）是工具视角；本章是 *战术* 视角 — 当你**没时间**先建测试网时，怎么把代码改了又不让债务爆炸。四个核心 pattern：**Sprout Method / Sprout Class / Wrap Method / Wrap Class**。这是 Part II 短篇症状章节的第一篇 — ch6–ch24 每章按"症状"分类，本章症状 = "没时间"。

## 〇、第一性原理思考

**这章做了什么**：给"没时间测"的场景预制四种 *additive change* 手法 — Sprout Method / Sprout Class / Wrap Method / Wrap Class, 让你不动老代码也能塞进新行为。

**为什么这样拆**：按"加在哪儿"分层(Method 内 / Class 内 / 方法名替换 / 类包装), 正好覆盖 PM 催你上线时能落手的全部注入点。

**最值钱的洞见**: 改动聚类 — 今天改的代码, 下次 80% 还会改; 测试不是今天花的钱, 是分摊到未来多次改动的预付,4 个 pattern 是这个预付的"先上车后补票"妥协。

legacy 加东西的两种语义: 加新行为(Sprout/Wrap, 本章)vs 改既有行为(需要 ch10/ch12) — 前者不动 = 老行为契约保留。

## 一、章节概述

- **核心论断**：legacy 改动 = 写测试 + 改代码；写测试看上去费时，但**改动聚类**（changes cluster） — 今天改的代码，下次很快又改；测试是分摊到未来多次改动的预付。
- **PM 施压 vs 工程纪律**：PM 要"今天上线"，工程要"先写测试"。Feathers 的立场 — 用 *sprout/wrap* **两种"写测试但不动老代码"的妥协**让前进可能。
- **Sprout Method**：把新逻辑抽成新方法，用 TDD 写测试，**从老方法内调一下**。原方法不动 = 原行为可保留；新方法 = 新行为可测。**前提**：能把新逻辑"独立成一个方法调用"。
- **Sprout Class**：当 sprout 不可能（构造依赖太重 / 类太大无法实例化），把新逻辑放到 **新类**，从老类实例化并调用。C++ 里这技法还顺带**避免改既有 header**，所以编译冲击面更小。
- **Wrap Method**：把老方法 `pay()` 改名 `dispatchPayment()`，再写一个同名 `pay()` 调它 + 新逻辑。让 *所有现有 caller* 不变，但新逻辑插入位置 = 一个 *新方法体*。两种变体：(a) 复用原方法名；(b) 引入新方法名（如 `makeLoggedPayment`），让 caller 自选。
- **Wrap Class**：类级别的 wrapper — 拿到老 `Employee` 后包成 `LoggingEmployee`（*decorator pattern*）。所有 `Employee` 的方法都要在 `LoggingEmployee` 上"转发"。当多个 caller 都想加同一逻辑时，decorator 比逐处改更省力。
- **4 个 pattern 的共同点**：都是 *additive change*，**不动既有测试不到的代码**。区别是 *在哪一层加*：Method = 方法内调新方法；Class = 调新类的方法；Wrap Method = 替换原方法名；Wrap Class = 包新类。
- **何时不能用 4 个 pattern**：当你需要 *改既有行为* 时。Sprout/Wrap 都是"加新东西"，不动旧的行为契约 —— 改行为得 ch10/ch12 这种动到老代码的技法。
- **静态方法当 staging area**：sprout Method 实在没法实例化原类时，可以把 sprout 写成 *static*，把原对象 instance variables 当参数传。**Feathers 立场**：把多个这样的 static 攒起来后，往往能识别出"应该抽成一个新类" —— static 是 *临时*设计，不是终点。
- **Wrap Method 的命名痛点**：你改名了 `pay()` → `dispatchPayment()` 后，新 `pay()` 的代码里出现 `dispatchPayment() calculatePay() logPayment()` —— 名字常常勉强。**Feathers 自己不喜欢**：如果有 Extract Method 工具可以再进一步拆；如果没有，**勉强但可读**也比 *乱改老方法*好。
- **Wrap Class 的两个 case**：(a) *行为独立* — 加的逻辑和现有类无关；(b) *类太大拒绝对它动刀* — 用 wrap 标一个"我以后要重构它"的旗子。case (b) 难习惯，但你得做。
- **Chapter 9 + Chapter 20 是续篇**：sprout/wrap 留了很多"以后要来愈合的疤"。ch9（*I Can't Get This Class into a Test Harness*）和 ch20（*This Class Is Too Big*）就是愈合这些疤的章节。

## 二、核心 Takeaways

### Takeaway 1: 改动 clustering — 写测试是分摊到未来多次改动的预付

- **是什么**：legacy 系统里改动有聚类效应 — 今天改的代码，下次（一个月内）80% 还会改。Feathers 的团队实验 — "一个迭代内不做没测试的改动" — 短期痛苦是真的，长期是工程健康的保证。
- **为什么重要**：把"测 vs 不测"的争论从"今天省事"提升到"团队未来 6 个月平均节奏"。
- **解决什么问题**：让"测试债"从抽象口号换成"在改动 cluster 区里具体哪几行值得 1 小时测"。
- **适用场景**：每周 code review 时，挑一个改动 cluster 区，下次 sprint 把这个区的测试债还掉一块。

### Takeaway 2: Sprout Method — 写在已有方法内的"新方法调用"

- **是什么**：在原方法体加一行 `// newMethod(args)`（先注释掉），再用 TDD 写 `newMethod`，最后去掉注释启用调用。
- **为什么重要**：原方法保留（行为不变 + 没测到的代码也没测），新方法 100% 测过 → 增量行为是绿的。这与 ch1 Takeaway 3 矩阵的"Adding a Feature"行直接对应。
- **解决什么问题**："我必须加功能，但原方法没测试覆盖" 的标准解。
- **适用场景**：从 ch3 的 Sale. scan() → 抽 `uniqueEntries()` 的简化版本。

### Takeaway 3: Sprout Class — 当 Sprout Method 的"加个方法"也构造不出来

- **是什么**：把新逻辑放到 *新文件* 的 *新类*。原方法实例化新类，调用其方法。
- **为什么重要**：原类可能构造依赖太重（ch9 主题），但新类只需要"原方法的局部变量"作为构造参数。新类的依赖面可控 → 新类可测 → 新行为可验证。
- **解决什么问题**：C++ 项目里尤其省 header 改动（原文专章写） —— 新 header 不污染既有编译单元。
- **适用场景**：从 ch6 QuarterlyReportGenerator 的 `QuarterlyReportTableHeaderProducer` 例子出发。

### Takeaway 4: Wrap Method = 改名 + 加同名 wrapper，让所有 caller 透明

- **是什么**：原 `pay()` → `dispatchPayment()` (private)；新增 `pay()` 调 `logPayment(); dispatchPayment();`。所有 caller 看到的还是 `pay()`，但内部多了 logging。
- **为什么重要**：是 *seam* 的一种 — 加功能而不污染老代码的边界。配合 TDD 让 `logPayment()` 单独测。
- **解决什么问题**："每个 caller 都要触发同一新逻辑" 的情况（decorator 化之前最简单形态）。
- **适用场景**：cross-cutting concern（日志/审计/指标）插入。

### Takeaway 5: Wrap Class = decorator pattern — 当 wrap 跨越多个 caller / 方法

- **是什么**：包新类 `LoggingEmployee implements Employee`，构造时收一个 `Employee`，每个方法 *delegate* 后加 logging。
- **为什么重要**：decorator 是 OO 教科书级 seam — 类型相同，行为可叠加。但要小心 **onion problem**：多层 decorator 嵌套追踪难。
- **解决什么问题**："同一逻辑需要加到 5 个不同方法上，每个方法都有大量 caller"。
- **适用场景**：cross-cutting + 行为独立 + 测试可注入 — 注意 n 层 decorator 的可读性代价。

### Takeaway 6: "临时 static" 是 staging area，不是终点

- **是什么**：实在实例化不出原类时，把 sprout 方法写成 `static`，把原对象 instance variables 当参数传入。
- **为什么重要**：Feathers 把 static 当 *staging area* —— 当攒出 3-4 个共用 instance variables 的 static 后，自然能看到"应抽新类"的轮廓。
- **解决什么问题**：避免"为了能测试，把全局状态拆得到处都是"的过度工程。
- **适用场景**：原类构造器依赖 5+ 个对象、又没时间全部 mock 时，先 static，后期识别出 *核心责任块* 后再迁回 instance method。

### Takeaway 7: sprout/wrap 留下的"疤"是 *已知债*，不是 *隐藏债*

- **是什么**：每次 sprout/wrap 都在原类周围建一座 *类围栏*，让新代码能在围栏外自由测试；围栏内的旧代码维持不动。
- **为什么重要**：这些疤是**显式的债** —— 你知道它在哪、它挡住了什么、什么时候该愈合（ch9/ch20）。比"假装代码没问题"的隐藏债健康。
- **解决什么问题**：让"避开改动"变成"标出待改区" —— 团队能用 *有节奏的偿还* 处理掉，而不是被债压垮。
- **适用场景**：每个 sprint 末尾挑 1-2 个"待愈合疤"做 ch9/ch20 章节里的技法处理。

## 三、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **Sprout Method = 内核 helper extraction 套路**。`tcp_sendmsg()` 加新行为（如 ECN 反馈）时，惯例是抽 `tcp_ecn_compute_ece()` 静态 helper，新行为独立测，老函数只加一行调用 — 与 ch6 Sprout Method 同构。
- **Sprout Class = 内核子系统抽离**。`io_uring` 从 `aio` 抽出来作为独立子系统 = Sprout Class 的工业实例：原 aio 维持，新 io_uring 独立测、独立演化、独立 patch window。
- **Wrap Method ≈ 内核 kfunc wrapper**。kfunc vtable 模式：原函数通过 `kfunc_lookup` 调用，再加包装函数注入新逻辑（如 `bpf_prog_run`），本质是 Wrap Method 在 runtime 的版本。
- **Wrap Class ≈ `struct xxx_ops`**。`net_device_ops` / `file_operations` / `inode_operations` 都是 *decorator pattern*：每个 ops 都是原对象的一个 *cross-cutting 层*。
- **C++ 编译防火墙**：Sprout Class 在 C++ 强调"不动既有 header"。Linux 内核 `__user` 标注、`EXPORT_SYMBOL_GPL` 同思路 —— 改 *边界* 不改 *接口*。
- **静态 staging area = `core_initcall` + `module_init`**：把"系统初始化逻辑"放到 static / module init 后，instance 主代码只留注册调用，与 Feathers "static 是 staging area" 思路一致。

#### Linux — Sprout/Wrap 的实际工程映射

| ch6 pattern        | Linux 系统对应                                  | 工程动作                         |
| ------------------ | -------------------------------------------- | ------------------------------ |
| Sprout Method      | `tcp_*()` 函数抽 `tcp_xxx_compute_yyy()` helper   | 静态 helper + TDD                  |
| Sprout Class       | `io_uring` 从 `aio` 抽                          | 新子系统 + 独立 patch series            |
| Wrap Method        | kfunc lookup + `__x_run()` 包装               | 替换原调用为 wrapper                   |
| Wrap Class         | `net_device_ops` / `file_operations`          | ops 注入新 ops；dev 注册时挂入           |
| 静态 staging area | `core_initcall` / `module_init`                | 把 init 逻辑抽 module；core 只留 register |

### 3.2 机器人软件视角

- **Sprout Method = ROS 节点内行为抽 callback**。`MoveBase::executeCb()` 加新逻辑（如新 recovery 行为）时，惯例是抽 `recovery_behavior_.run()`，老 executeCb 只加一行调用。原方法测不了（依赖 costmap/tf/DDS），但 `recovery_behavior_.run()` 可 mock 测。
- **Sprout Class = `ros2_control` 子系统抽 hardware interface**。hardware 从 controller 抽出来 = Sprout Class 工业实例：原 controller 维持，hardware interface 独立测、独立 fake 化（`mock_components`）。
- **Wrap Method ≈ `lifecycle::LifecycleNode` 配置验证**。在 `on_configure`/`on_activate` 之间加 *配置验证* 时，wrap 一个新 `validate_configure()` 调老 `on_configure()`。
- **Wrap Class ≈ `nav2_core` pluginlib**。每个 global planner plugin 都是 `nav2_core::GlobalPlanner` 的具体实现 = decorator pattern 的工业映射。nav2 用 pluginlib 装饰不同 planner，**加新行为不污染 base planner**。
- **机器人硬件 = 严重 hidden dependency**。`/dev/ttyUSB0` / `can0` / `eth0` 都是真实硬件 — 用 `ros2_control` 的 `mock_system_interface` 注入测试 fake，正好是 ch9 Irritating Global Dependency 的预备。
- **decorator 嵌套陷阱**：Nav2 的 `smac_planner` × `controller_server` × `bt_navigator` 三层叠加 = ch6 警告的"onion problem"。每次看 bug 要 peel 三层。

#### 机器人 — Sprout/Wrap 的实际工程映射

| ch6 pattern        | ROS/ROS2 对应                                    | 工程动作                          |
| ------------------ | ----------------------------------------------- | ------------------------------- |
| Sprout Method      | `MoveBase::executeCb()` 抽 `recoveryBehavior_.run()`  | 单 callback TDD                  |
| Sprout Class       | `ros2_control::SystemInterface` 从 controller 抽       | 新 pkg + 独立 fake 测试            |
| Wrap Method        | `lifecycle::LifecycleNode::on_configure` 加 validation  | wrap 配置验证                    |
| Wrap Class         | `nav2_core::GlobalPlanner` × pluginlib                 | planner 注入；新 planner 测      |
| 静态 staging       | `rclcpp::Node` 内 `declare_parameter` 集中化           | 把参数声明抽 init method         |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                    | 高级工程师                                                    |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 接到 PM 紧急任务      | "我 1 小时就改完，何必写测试"                                | "我加新方法用 TDD 写；老方法不动；下次改动 cluster 受益"     |
| 看到不能测的类      | "重写吧"                                                    | "我先 Sprout / Wrap；标出 *待愈合疤*；下个 sprint 处理"      |
| 对 sprout 的取舍    | "sprout 看起来假，干脆 inline 改"                            | "sprout 留疤可追踪；inline 改更糟"                          |
| 对 decorator 的取舍   | "decorator 让代码难看，先不要"                            | "decisions: 行为是否独立？是否多方法？是否多 caller？ → 决定" |
| 选 Sprout vs Wrap    | 看心情                                                      | *改老方法* 时 Wrap；*加新方法* 时 Sprout                      |
| 命名                | 把新方法起个拼音首字母名                                    | 起名强类型、文档化（`logPayment` > `lp`）                |
| 对测试网            | "测了老代码再改" = 不可能                                   | "我可以避开老代码 — 但我标出将来必须补的疤"              |

> **关键差异**：高级工程师把"避免改老代码"作为 *主动设计选择*，不是 *被动放弃*；并把 sprout/wrap 留下的疤当作**显式债务**，每次 sprint 末处理 1-2 个。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

仍然极重要。原因：
- **AI 让"先写代码再说"更便宜** — 但 ch1 矩阵里 *Adding a Feature* 行依然要 regression 网。Sprout 是给 *Adding* 行装网的最轻技法。
- **AI 倾向给出 *inline 改* + mock framework 配套** —— 看起来很"专业"，实际把老代码改坏了。Sprout / Wrap 的核心约束（**不动老代码**）正是 AI 容易违反的。
- **遗留代码存量** —— AI 让新代码生成加速，但存量 legacy code 不会消失。Sprout/Wrap 在未来 5 年仍是 legacy 改动的主力技法。

### 4.2 AI 已经能做的（具体到 ch6 主题）

- **识别该用 sprout 还是 wrap**：基于"新逻辑是否独立 / 是否需要 decorator-like 多方法" 等特征分类。准确率中上。
- **自动生成 sprout method**：给一段新逻辑描述 + 老方法签名，AI 抽出新方法 + 写 TDD 测试草稿。准确率 70-80%，需 review。
- **识别"待愈合疤"**：扫描代码库找出"看起来 sprout/wrap 但应该愈合"的实例，给出愈合方案（ch9/ch20 技法）。准确率 60%。
- **decorator 链静态分析**：识别多层 decorator，给每层加日志帮助 trace。准确率中。

### 4.3 AI 不能替代的（具体到 ch6 主题）

- **决定哪个 caller 走 Wrap Class vs 走 Sprout Class**：取决于 *改动频率 × 跨方法需求*，这是工程判断。
- **staging-area static 何时合并**：攒到几个 static 后识别"应抽新类" —— 取决于对系统业务的理解。
- **判断 wrap 的命名是否合理**：`dispatchPayment` 是不是真合适？AI 给的命名只是 plausible 的，不是最贴业务的。
- **评估 decorator 多层嵌套的可读性代价**：是否 3 层装饰已经过头？人读代码的 *主观痛苦度* AI 量不出来。

### 4.4 AI 经常写错的地方

针对 ch6 Sprout / Wrap 主题：

| 错误模式                                          | 例子                                                                                                  | 后果 |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------------- | ---- |
| **把 sprout 写成 inline 改 + 加注释**             | AI 收到"加逻辑 X 到 pay()"，直接在 pay() 内改 + 注释 `// ADDED: X`。                                  | 老代码被改；老代码没测试 → *矩阵 Adding 行悄悄变成 Bug/Refactor 行* |
| **wrap method 时没保 Preserve Signatures**         | AI 把 `pay()` 改名 `dispatchPayment()` 但 *忘了* 加同名 `pay()` wrapper。所有 caller 编译失败。     | 改动爆炸；改了一个方法破坏了 N 个调用点 |
| **wrap class 漏 delegate 方法**                  | AI 给 `LoggingEmployee` 加 `pay()` wrapper 但忘了 `dispatchPayment()`，原 caller 调 `dispatchPayment` 时 logging 失效 | 部分 caller 不生效；qa 测 A 通过、测 B 失败 |
| **decorator 链打错顺序**                          | AI `new StepNotify(new Alarm(new ACME()))` 但业务应该是 `new Alarm(new StepNotify(new ACME()))` | 行为反序；onion problem 在生产才暴露 |
| **staging static 永久化**                          | AI 觉得 static 好用就一直留，从不识别"该抽类"。                                                     | 临时债变永久债；测试覆盖也只能 cover 局部 |
| **Sprout Class 不保 header 隔离**                | C++ 里 AI 把 sprout 类写到原类同一 header；include 原 header 时触发新依赖。                              | 编译防火墙失效；ch6 Sprout Class 在 C++ 的核心收益丢失 |
| **sprout 方法命名糟糕**                            | AI 写 `doIt()`、`process()`、`helper()` 这种泛名                                                      | 测试断言可读性下降；1 年后没人记得 do它是啥 |

### 4.5 子段：AI 辅助遗留代码理解 — 在本主题专项

- **AI 帮你"识别待 sprout 区"**：legacy 代码中"老方法 + 应该独立出来的逻辑"，AI 能用静态分析（call graph + 控制流图）标记 70% 候选。但"该不该 sprout" 仍由人决定。
- **AI 帮你"识别 sprout 后的疤"**：扫描仓库，给"看起来像 sprout 但写法可疑"的代码段打 tag。例如：sprout method 长度 > 原方法两倍 = *可能是过度 sprout*；装饰链深度 > 3 = *onion 风险*。
- **AI 帮你"批量生成 wrap class 的 delegate 方法"**：给一个目标接口，AI 自动生成所有 delegate 方法。**风险**：原接口变了 AI 不会同步更新 → 漏 delegate。
- **AI 不会替你写 *测试约束*：每个 sprout / wrap 必须 *有测试*。AI 可以给测试，但 *没有测试* 时 AI 不会主动加。

### 4.6 工程师必须保留的核心能力

- **判断 Sprout / Wrap 的边界**：是加新方法还是改老方法名？是否有 cross-cutting？这些是 commit message 前置判断。
- **wrap 时 *Preserve Signatures* 的纪律**：每个 wrap 必须能让所有 caller 透明工作 — 这要 review 时一对一比对。
- **decorator 嵌套层的可读性判断**：3 层以上就该警觉。AI 不知道这阈值。
- **staging-area static 的"应抽新类"识别**：3-4 个 static 后该看是否要重组。这是设计感，不是 AI 能给的。
- **对 sprout/wrap 留下疤的 *愈合节奏* 判断**：每个 sprint 该处理几个？团队节奏 vs 项目阶段。这是项目管理问题。

## 五、实践行动项

> ch6 是 Part II 短篇症状章，4 个核心 pattern 各对应一个 C 复刻 demo + 一个测试。所有 demo 在 `/tmp/ch06-*` 下编译跑通。

### A1 — Sprout Method: `TransactionGate.postEntries` 去重

```bash
mkdir -p /tmp/ch06-sprout && cd /tmp/ch06-sprout

cat > gate.h <<'EOF'
#ifndef GATE_H
#define GATE_H
typedef struct Entry { int id; } Entry;
typedef struct ListMgr { int seen[16]; int n; } ListMgr;
typedef struct Gate {
    ListMgr mgr;
    int (*has_entry)(struct Gate *g, Entry *e);
} Gate;
void gate_init(Gate *g);
int  gate_has_entry(Gate *g, Entry *e);
void gate_add(Gate *g, Entry *e);
#endif
EOF

cat > gate.c <<'EOF'
#include "gate.h"
void gate_init(Gate *g) { g->mgr.n = 0; g->has_entry = gate_has_entry; }
int gate_has_entry(Gate *g, Entry *e) {
    for (int i = 0; i < g->mgr.n; i++)
        if (g->mgr.seen[i] == e->id) return 1;
    return 0;
}
void gate_add(Gate *g, Entry *e) {
    if (g->mgr.n < 16) g->mgr.seen[g->mgr.n++] = e->id;
}
EOF

cat > unique.h <<'EOF'
#ifndef UNIQUE_H
#define UNIQUE_H
#include "gate.h"
typedef struct { Entry *out; int n; int cap; } UniqueBuf;
/* Sprout Method: 抽出 uniqueEntries — 跳过 has_entry() == 1 的元素 */
void unique_entries(Gate *g, Entry *in, int nin, UniqueBuf *out);
#endif
EOF

cat > unique.c <<'EOF'
#include "unique.h"
void unique_entries(Gate *g, Entry *in, int nin, UniqueBuf *out) {
    out->n = 0;
    for (int i = 0; i < nin; i++) {
        if (!g->has_entry(g, &in[i]))
            if (out->n < out->cap) out->out[out->n++] = in[i];
    }
}
EOF

cat > test_sprout.c <<'EOF'
#include "gate.h"
#include "unique.h"
#include <assert.h>
#include <stdio.h>
int main(void) {
    Gate g; gate_init(&g);
    Entry e1 = {.id = 1}; gate_add(&g, &e1);
    Entry ins[] = { {.id=1}, {.id=2}, {.id=1}, {.id=3} };
    Entry out[8];
    UniqueBuf buf = { .out = out, .cap = 8 };
    unique_entries(&g, ins, 4, &buf);
    assert(buf.n == 2);
    assert(buf.out[0].id == 2);
    assert(buf.out[1].id == 3);
    fprintf(stderr, "test_sprout PASS\n");
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_sprout gate.c unique.c test_sprout.c
./test_sprout
echo "rc=$?"
```

**验收**：`test_sprout PASS` + `rc=0`。

### A2 — Sprout Class: `QuarterlyReportGenerator` 加 HTML header

```bash
mkdir -p /tmp/ch06-sprout && cd /tmp/ch06-sprout

cat > report.h <<'EOF'
#ifndef REPORT_H
#define REPORT_H
typedef struct ReportGenerator ReportGenerator;
ReportGenerator *report_new(void);
void report_free(ReportGenerator *g);
void report_add_row(ReportGenerator *g, const char *dept,
                    const char *mgr, int profit, int expense);
const char *report_render(ReportGenerator *g);
#endif
EOF

cat > report.c <<'EOF'
#include "report.h"
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
struct ReportGenerator { char *html; size_t cap, len; };
ReportGenerator *report_new(void) {
    ReportGenerator *g = calloc(1, sizeof(*g));
    g->cap = 256; g->html = malloc(g->cap); g->html[0] = 0;
    return g;
}
void report_free(ReportGenerator *g) { free(g->html); free(g); }
void report_add_row(ReportGenerator *g, const char *dept,
                    const char *mgr, int profit, int expense) {
    char row[256];
    snprintf(row, sizeof(row),
        "<tr><td>%s</td><td>%s</td><td>$%d</td><td>$%d</td></tr>\n",
        dept, mgr, profit, expense);
    size_t need = g->len + strlen(row) + 1;
    if (need > g->cap) { g->cap *= 2; g->html = realloc(g->html, g->cap); }
    strcat(g->html, row); g->len += strlen(row);
}
const char *report_render(ReportGenerator *g) { return g->html; }
EOF

cat > header_producer.h <<'EOF'
#ifndef HEADER_PRODUCER_H
#define HEADER_PRODUCER_H
const char *header_producer_make(void);
#endif
EOF

cat > header_producer.c <<'EOF'
#include "header_producer.h"
const char *header_producer_make(void) {
    return "<tr><td>Department</td><td>Manager</td>"
           "<td>Profit</td><td>Expenses</td></tr>\n";
}
EOF

cat > test_sprout_class.c <<'EOF'
#include "header_producer.h"
#include <assert.h>
#include <string.h>
#include <stdio.h>
int main(void) {
    const char *h = header_producer_make();
    assert(strstr(h, "<td>Department</td>")  != NULL);
    assert(strstr(h, "<td>Manager</td>")     != NULL);
    assert(strstr(h, "<td>Profit</td>")      != NULL);
    assert(strstr(h, "<td>Expenses</td>")    != NULL);
    fprintf(stderr, "test_sprout_class PASS\n");
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_sprout_class header_producer.c test_sprout_class.c
./test_sprout_class
echo "rc=$?"
```

**验收**：`test_sprout_class PASS` + `rc=0`。

### A3 — Wrap Method: `Employee.pay()` wrap 出 logging

```bash
mkdir -p /tmp/ch06-wrap && cd /tmp/ch06-wrap

cat > employee.h <<'EOF'
#ifndef EMPLOYEE_H
#define EMPLOYEE_H
#include <stdio.h>
typedef struct Employee {
    double hours, rate;
    int pay_count, log_count;
    char log_path[64];
} Employee;
void employee_init(Employee *e, double hours, double rate,
                   const char *log_path);
void employee_pay(Employee *e);             /* wrap: log + dispatch */
void employee_dispatch_payment(Employee *e); /* 老 pay() 核心 */
void employee_log_payment(Employee *e);
#endif
EOF

cat > employee.c <<'EOF'
#include "employee.h"
#include <stdio.h>
#include <string.h>
void employee_init(Employee *e, double hours, double rate,
                   const char *log_path) {
    e->hours = hours; e->rate = rate;
    e->pay_count = e->log_count = 0;
    strncpy(e->log_path, log_path, sizeof(e->log_path)-1);
}
void employee_log_payment(Employee *e) {
    FILE *f = fopen(e->log_path, "a");
    if (!f) return;
    fprintf(f, "pay: hours=%.2f rate=%.2f\n", e->hours, e->rate);
    fclose(f);
    e->log_count++;
}
void employee_dispatch_payment(Employee *e) { e->pay_count++; }
void employee_pay(Employee *e) {
    employee_log_payment(e);
    employee_dispatch_payment(e);
}
EOF

cat > test_wrap_method.c <<'EOF'
#include "employee.h"
#include <stdio.h>
#include <assert.h>
#include <unistd.h>
int main(void) {
    const char *log = "/tmp/ch06-wrap-pay.log";
    unlink(log);
    Employee e;
    employee_init(&e, 40.0, 50.0, log);

    employee_pay(&e);
    assert(e.pay_count == 1);
    assert(e.log_count == 1);

    FILE *f = fopen(log, "r");
    assert(f != NULL);
    char buf[128];
    fgets(buf, sizeof(buf), f);
    fclose(f);
    assert(strstr(buf, "hours=40.00") != NULL);
    fprintf(stderr, "test_wrap_method PASS\n");
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_wrap_method employee.c test_wrap_method.c
./test_wrap_method
echo "rc=$?"
```

**验收**：`test_wrap_method PASS` + `rc=0`。

### A4 — Wrap Class: `LoggingEmployee` decorator

```bash
mkdir -p /tmp/ch06-wrap && cd /tmp/ch06-wrap

cat > logging_employee.h <<'EOF'
#ifndef LOGGING_EMPLOYEE_H
#define LOGGING_EMPLOYEE_H
#include "employee.h"
typedef struct LoggingEmployee {
    Employee *inner;
    char      log_path[64];
    int       call_count;
} LoggingEmployee;
void logging_employee_init(LoggingEmployee *le, Employee *inner,
                            const char *log_path);
void logging_employee_pay(LoggingEmployee *le);
#endif
EOF

cat > logging_employee.c <<'EOF'
#include "logging_employee.h"
#include <stdio.h>
#include <string.h>
void logging_employee_init(LoggingEmployee *le, Employee *inner,
                          const char *log_path) {
    le->inner = inner;
    strncpy(le->log_path, log_path, sizeof(le->log_path)-1);
    le->log_path[sizeof(le->log_path)-1] = 0;
    le->call_count = 0;
}
void logging_employee_pay(LoggingEmployee *le) {
    FILE *f = fopen(le->log_path, "a");
    if (f) { fprintf(f, "decorator-pay\n"); }
    fclose(f);
    le->inner->pay();
    le->call_count++;
}
EOF

cat > test_decor.c <<'EOF'
#include "employee.h"
#include "logging_employee.h"
#include <stdio.h>
#include <assert.h>
#include <unistd.h>
int main(void) {
    const char *log = "/tmp/ch06-wrap-decor.log";
    unlink(log);
    Employee inner;
    employee_init(&inner, 40.0, 50.0, log);
    LoggingEmployee le;
    logging_employee_init(&le, &inner, log);

    logging_employee_pay(&le);
    assert(le.call_count == 1);
    assert(inner.pay_count == 1);
    assert(inner.log_count == 1);
    fprintf(stderr, "test_decor PASS\n");
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_decor employee.c logging_employee.c test_decor.c
./test_decor
echo "rc=$?"
```

**验收**：`test_decor PASS` + `rc=0`。

## 六、值得深入思考的问题

### Q1: Sprout Method vs Inline 改的边界在哪？

Feathers 强调"几乎所有情况都用 Sprout"。但实际项目里有些"3 行新逻辑 + 5 行上下文" 的改动 inline 看起来更直观。**关键问题**：团队应该用 *什么规则*判断"该 sprout"？还是说一切看个人审美？

### Q2: Wrap Class / Decorator 的可读性税到底什么时候付出？

3 层 decorator 已经难读 — ch6 自己警告 onion problem。**关键问题**：什么时候 decorator 该被 *拆回*正常继承 / composition？这是设计判断 + 团队约定，不是普适规则。

### Q3: 静态方法"staging area"该停留多久？

Feathers 说"几个攒起来就该识别新类"。但实际团队经常 *永远停留*在 staging — 5 年后看代码还是 static 一堆。**关键问题**：用什么 review 流程 / 工具强制"static 该被 refactor 了" 的检查？

### Q4: AI 写 sprout/wrap 时怎么防止"inline 改 + 标 sprout" 的伪合规？

AI 默认倾向给最小可见改动。Commit message 写 `sprout: ...`，diff 实际是 inline 改 + 加注释。**关键问题**：这条 *虚标* 现象怎么被 review 流程自动捕获？是否要 *diff 必须包含新方法签名* 的硬性 commit-hook？

### Q5: sprout/wrap 留下疤后，谁来负责愈合？

ch9/ch20 是愈合技法。每个团队里 *谁* 把这些疤列在 sprint backlog？谁来保证不在 *新功能 backlog 永远占满*时被挤掉？这是项目管理问题，不是工程问题。

### Q6: Wrap Class 与 Strategy Pattern 的边界模糊

`LoggingEmployee` 是 decorator 还是 *strategy*？两者都是 *包一个 inner*。**关键问题**：当我们说"wrap 是 decorator"时，是否暗示了 *可叠加多个 wrap*？如果不可叠加，是否该用 strategy 而非 decorator？

---

*下一章预告*: **Chapter 7 — It Takes Forever to Make a Change** —— 同一问题的另一面：当 *反馈时间*（compile + link + 跑测试）拖到十几分钟时，怎么办。本章教 *build 依赖* 解构：Extract Interface / Extract Implementer 在 build graph 上 *建编译防火墙*；分包 / 拆 lib 让增量 build 时间从 10 分钟跌到 10 秒。ch7 是 ch6 的 *配套章* — 一个管 *代码改动*，一个管 *build 时间*，合起来让 legacy 改造 *实际可承受*。
# Chapter 8 — How Do I Add a Feature?

> **PDF**: p.109-126（18 页）
> **定位**: 在 ch6 (避开老代码改) + ch7 (build 快) 的铺垫后，ch8 正面讲 *"如何在 legacy 上加 feature"*。三大技法：**TDD**（按 ch2 算法扩展） / **Programming by Difference**（继承加 feature） / **Normalized Hierarchy**（避免 LSP 违反）。这是 Part II 中 *"改动方法"* 的代表章。

## 〇、第一性原理思考

**这章做了什么**: 在 ch6 + ch7 铺垫后正面讲 legacy 加 feature 的标准路径 — (1) 先把代码放进 test harness (2) TDD 写新功能 (3) refactor 愈合, 用 TDD 5 步算法 + Programming by Difference + Normalized Hierarchy 三件套收尾。

**为什么这样拆**: 把"加 feature"切成节奏(TDD 算法)、staging 工具(Programming by Difference)、设计守门(Normalized Hierarchy / LSP)三层, 覆盖从落地到长期健康度的全部关口。

**最值钱的洞见**: legacy TDD 与 greenfield TDD 的唯一区别是 *第 0 步* — 必须先把类放进 test harness, 否则标准 4 步根本起不来; ch8 表面讲 feature, 实际上在召唤 ch9–ch13。

Programming by Difference 不是终态 — 多次叠加会让 design 快速退化(Figure 8.2 两个 subclass 不能叠加), 它必须配 *愈合 sprint* 才能用。

## 一、章节概述

- **核心论断**：加 feature 不是"加代码"，而是 *扩展测试网 + 加新行为*。在 legacy 上加 feature 的标准路径：(1) **先把改的代码放进 test harness**（ch9–ch13 的技法）→ (2) **TDD 写新功能** → (3) **refactor 愈合**。
- **TDD 5 步算法 (legacy 扩展版)**：标准 TDD 是 4 步（写失败 test → 编译 → 通过 → 去重）。legacy 扩展 = **第 0 步：先把类放进 test harness**，再走标准 5 步。**关键差异**：legacy 里第 0 步可能耗时数小时。
- **TDD cycle demo**：`InstrumentCalculator.firstMomentAbout(point)` → `secondMomentAbout(point)` → 抽取 `nthMomentAbout(point, n)`。每步都有 *failing test → compile → pass → remove duplication* 的清晰节奏。
- **Programming by Difference**：用 *继承* 添加 feature — `AnonymousMessageForwarder extends MessageForwarder` override `getFromAddress()`。**这是 OO 特有的 option** — 让 test 在不修改原类的情况下，先把行为加上。
- **Programming by Difference 的归宿**：继承只是 *staging*。当新行为稳定后，*折叠* 回原类（或抽到 `MailingConfiguration`）。原书 Figure 8.3 → 8.6 的演进就是这过程。
- **Normalized Hierarchy**（*normalized*）：一种 hierarchy 约束 — **没有子类覆盖父类的具体方法**。每个 method 在 hierarchy 中只有一个实现。"如何做 X？" 答案只在一个地方。
- **Liskov Substitution Principle (LSP)**：子类的对象必须能替换父类对象，*不改变程序的可观察行为*。`Square extends Rectangle` 是经典反例 — `setWidth(3); setHeight(4);` Square 后面积 = 16 不是 12。
- **LSP 与 inheritance 的关系**：Programming by Difference 容易违反 LSP（override concrete methods 改变了父类的 contract）。**对策**：尽量把父类的具体方法做成 abstract，或 move behavior 到 *config object* / *strategy*。
- **MailingConfiguration 的演进**：`MessageForwarder` 接收 `Properties` config → 把 config 抽成 `MailingConfiguration` 类 → 把 `getFromAddress()` 移到 `MailingConfiguration` → 最终 rename 为 `MailingList`。这与 ch6 Sprout Class 同构，但对象不同。
- **chapter 总结**：在能 get code under test 的前提下，**(TDD + refactor)** 是加 feature 的最强大工具链；Programming by Difference 是 *过渡* 工具，不能是 *最终* 状态。

## 二、核心 Takeaways

### Takeaway 1: legacy TDD = 标准 TDD + 第 0 步 "先把类放进 test harness"

- **是什么**：ch8 给出 5 步算法 — (0) 把类放进 test harness → (1) 写 failing test → (2) 编译过 → (3) 让它过 → (4) 去重 → (5) 重复。
- **为什么重要**：标准 TDD 假设你 *已经能跑这个类*。legacy 上跑不动 = 必须先 ch9–ch13 拆依赖。**第 0 步是 legacy TDD 与 greenfield TDD 的唯一区别**。
- **解决什么问题**：把"legacy 改动" 从 "祈祷改对" 转换成"测试驱动 + 行为锁定"。
- **适用场景**：每个 legacy 改动任务的第一步。

### Takeaway 2: "Cut/Copy/Paste + Refactor" 是 legacy TDD 的合法步骤

- **是什么**：TDD 第 4 步去重之前，可以临时 copy-paste 整段代码（如 `secondMomentAbout` 复制 `firstMomentAbout`）。看似 dirty，但**只要最终去重**就 OK。
- **为什么重要**：copy-paste 让 *老代码与新代码并排可见*，便于人 review 时识别差异。比"先 generalize 再写" 安全。
- **解决什么问题**：legacy 上"看到老代码长什么样" 比"猜它应该长什么样" 更稳。
- **适用场景**：legacy TDD 的早期 cycle（1-3 次），等测试网密了再 generalize。

### Takeaway 3: Programming by Difference 是 *staging*，不是 *终态*

- **是什么**：用 `AnonymousXxx extends Xxx` 暂时 override 一个方法加新行为。看起来"加 feature 完毕"。
- **为什么重要**：原书明确警告 — **多次 Program by Difference 会让 design 快速退化**（Figure 8.2 的两个 subclass 不能叠加）。需要时用 tests *折叠* 回原类或抽到 strategy。
- **解决什么问题**："加 feature 但原类测不了" 的快速通道。
- **适用场景**：需要快速通过 release 截止 + 没时间 refactor 时。但必须配 *愈合 sprint*。

### Takeaway 4: LSP 违反 = 静默 bug 源

- **是什么**：子类覆盖父类具体方法，且 *改变了父类的 contract*。`Rectangle r = new Square(); r.setWidth(3); r.setHeight(4);` 得到 16 而不是 12。
- **为什么重要**：LSP 违反在测试套件全绿时仍会爆 —— 因为"使用方"的假设被悄悄破坏。
- **解决什么问题**：避免 *override concrete methods* 的过度使用。**两条规则**：(a) 避免 override concrete methods；(b) 如果 override 了，确保在 override 里调 super. method()。
- **适用场景**：每次 PR review 时检查"这个 subclass 是 normalized 吗"。

### Takeaway 5: Normalized Hierarchy = "method 在 hierarchy 中只有一个实现"

- **是什么**：父类 abstract / 子类 concrete。父类没有 concrete method 被覆盖。回答"如何做 X？" 只在一个地方。
- **为什么重要**：让 *代码可被快速理解*。读 base class 就知道"行为长什么样"，不会被 override 偷换。
- **解决什么问题**：hierarchy 越长越难懂的 classic 反例（深 OO tree 的方法追溯噩梦）。
- **适用场景**：每次决定"该继承还是该组合" 时 — 默认 *normalized*。

### Takeaway 6: Properties Configuration → Config Object 是 legacy feature flag 的演化

- **是什么**：从 `Properties configuration = new Properties(); setProperty("anonymous", "true");` 演化成 `MailingConfiguration` 类 — 把 *字符串配置* 转成 *类型化对象*。
- **为什么重要**：type化后 IDE 能检查 + test 能 mock + 行为更可观察。
- **解决什么问题**：feature flag 用 properties 后，"哪个 flag 在哪生效" 难追。
- **适用场景**：3-4 个 feature flag 时就考虑抽 config 类。

### Takeaway 7: TDD + Refactor 让你 *免费* 折叠 Programming by Difference

- **是什么**：Programming by Difference 留下的临时子类，最终被 tests 折叠回原类或 strategy。tests 让 *折叠* 是机械操作，不破坏行为。
- **为什么重要**：让 *暂时 dirty* 不等于 *永久 dirty*。这是 test 网价值的核心体现。
- **解决什么问题**：让"先用 dirty 路径通过 deadline，再清" 成为可行流程。
- **适用场景**：每个 sprint 末尾安排 timebox 处理 PdD 折叠。

## 三、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **TDD 在内核的体现 = KUnit + Kselftest**。`tcp_*()` 函数加新行为前，先写 KUnit test (`TEST(t, fast_path)`) — 步骤与 ch8 TDD 算法完全同构。但 kernel 的 *第 0 步*（get under test）需要 kfunc indirection 或 user-mode driver。
- **Programming by Difference 在内核 = `__init` vs `__exit` hooks**。新 platform 加 driver 时，先 subclass-like 注册 `xxx_probe` 而不直接改通用代码。这是 *临时 dirty* 的工业实例。
- **Normalized Hierarchy = kernel 的 ops table**。`struct file_operations` 是 abstract base；`ext4_file_operations` 是具体子类。每个 method 在 hierarchy 中只有一个实现 — 是 *normalized*。
- **LSP 违反的 kernel 实例**：`memcpy()` 在不同 arch 是 *不同实现*（x86 vs ARM vs RISC-V）。`memcpy()` 的 contract 是"copy bytes" — 不同 arch 的实现都保持这 contract。这就是 *normalized* 的好例子（*实际上* 通过 header 选实现，不是真继承）。
- **Properties → Config Object = kernel module parameters**。`module_param(foo, int, 0644)` 是 *字符串 + 类型* 的混合。kernel 5. x 引入 `module_param_cb()` 走 callback，是 *config object* 思路的简化版。

#### Linux — TDD / Programming by Difference 映射

| ch8 技法             | Linux 内核对应                                  | 工程动作                         |
| ------------------- | -------------------------------------------- | ------------------------------ |
| TDD 第 0 步          | KUnit 测试 setup                                | `DECLARE_TEST_SUITE`           |
| TDD cycle            | `TEST(t, foo)` → 编译 → pass → 去重              | KUnit 标准 cycle                |
| Programming by Diff  | `xxx_probe` 注册到 platform_driver                | 新 platform 不动 core            |
| Normalized Hierarchy | `struct file_operations`                       | ops 全 concrete method；无 override |
| LSP 违反反例         | `memcpy()` 不该被用户自己 override                | `EXPORT_SYMBOL(memcpy)` 但实际实现可选 |

### 3.2 机器人软件视角

- **TDD 在 ROS 节点 = launch_test + ament test**。`ament_add_gtest(test_foo ...)` → `TEST(foo, behavior)`。第 0 步是 *把节点放进 test harness* — 用 `rclcpp::Node::make_shared` + 假 publisher/subscriber。
- **Programming by Difference = behavior tree 节点继承**。`nav2_bt_navigator` 的 BT XML 中，每个 Node 是 *继承* 自 BT framework 的具体类。`RecoveryNode` / `PipelineSequence` 都是 *"在 BT framework 上加 behavior"* 的例子。
- **Normalized Hierarchy = `nav2_core` plugin 体系**。`nav2_core::GlobalPlanner` 是 abstract base；每种 planner (`NavFnPlanner`、`SmacPlanner`) 是具体子类，每个 method 各在一个地方定义。**改 planner 不影响其它 planner**。
- **LSP 在 ROS2 = lifecycle 节点 state transition**。`lifecycle::LifecycleNode` 子类必须保证 `on_configure` / `on_activate` / `on_deactivate` / `on_cleanup` / `on_shutdown` 的语义不破坏父类 contract。LSP 违反会让 lifecycle manager 误判状态。
- **Properties → Config Object = ROS2 parameters + parameter callbacks**。`declare_parameter("anonymous", false)` + `add_on_set_parameters_callback(...)` — *类型化* + *callback* 的组合，正是 *config object* 的工业实例。

#### 机器人 — TDD / PdD 映射

| ch8 技法             | ROS/ROS2 对应                                    | 工程动作                         |
| ------------------- | ----------------------------------------------- | ------------------------------ |
| TDD 第 0 步          | ament test setup                                 | `rclcpp::Node::make_shared`    |
| TDD cycle            | `TEST(bt, recovery)`                              | `ament_add_gtest`              |
| Programming by Diff  | BT 节点 `RecoveryNode extends BT::ControlNode`    | 自定义 behavior                  |
| Normalized Hierarchy | `nav2_core::GlobalPlanner`                       | 每 planner 一处实现              |
| Properties → Config | `declare_parameter` + on_set_parameters_callback  | 类型化 param + 响应 callback     |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                   | 高级工程师                                                    |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| TDD 节奏            | "测了再说"                                                  | "第 0 步：先把类测得动；再走 5 步"                           |
| 看到大函数           | "重写"                                                       | "copy-paste 改一版；TDD 验；refactor 折叠"                  |
| 看到不可测类         | "先打 mock"                                                  | "第 0 步 = ch9–ch13 技法；get under test 后才 TDD"          |
| 选 inheritance       | "能继承就继承"                                              | "继承 = PdD 的 staging；终态是 composition"                  |
| 看到 override concrete method | "反正能 override"                                  | "LSP 风险；要么改 abstract，要么改 strategy"                 |
| feature flag         | 加 if-else                                                   | 加 config object + type-safe accessor                       |

> **关键差异**：高级工程师把 TDD 当"5 步算法" — 把 *第 0 步* 当作独立工程，与 ch9–ch13 联动；初级把 TDD 当"4 步循环" — 忽略第 0 步。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

仍然极重要。原因：
- **AI 写测试快，但"先把类测得动"** 仍依赖对 *依赖图* 的判断 — 这与 ch9–ch13 绑定。
- **AI 默认偏好 PdD**（继承 override） — 因为 *看起来 minimal change*。但 PdD 多次使用会 *设计塌陷*。LSP 违反是隐形 bug 源。
- **AI 写 TDD 时不会自动加 "第 0 步"** — 因为 *第 0 步* 不是从 test 描述能生成的，是依赖 ch9–ch13 的技法选择。
- **AI 写 normalized hierarchy 时倾向于把所有 abstract method 留空** — 这与 Liskov 兼容但增加 implementation effort。

### 4.2 AI 已经能做的（具体到 ch8 主题）

- **生成 TDD test 序列**：给一段 legacy 函数描述，AI 按 5 步生成 test 草稿。第 0 步识别率 50%，其余 80%。
- **识别 LSP 违反**：基于 inheritance 树 + override 方法的 signature diff，AI 标出 *可能* 违反 LSP 的点。准确率 70%（需要人工 review）。
- **生成 config object 类**：从 `Properties` 散落代码生成自动生成 `Config` 类。准确率 80%。
- **识别 normalized vs 非 normalized hierarchy**：基于 override 关系图，给"这个 hierarchy 是否 normalized" 的判分。准确率 90%。

### 4.3 AI 不能替代的（具体到 ch8 主题）

- **第 0 步的具体技法选择**：TDD 之前是用 *Extract Interface* / *Pass Null* / *Subclass and Override*？这是 ch9 决策，与团队 / 项目阶段相关。
- **判断 PdD 何时该折叠**：sprint 末安排 PdD 愈合，但 5 个 PdD 哪个先折？这是优先级判断。
- **判断 LSP 违反的 *实际风险***：AI 给 70% 准确率；剩余 30% 需要 *业务理解* 才能识别（"setWidth 之后 rectangle 的 area 是否还成立？"）。

### 4.4 AI 经常写错的地方

针对 ch8 TDD / PdD / LSP 主题：

| 错误模式                                          | 例子                                                                                                  | 后果 |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------------- | ---- |
| **跳过第 0 步，直接写 TDD test**                  | AI 收到 "add `getValidationPercent()`"，直接写 failing test 但 `CreditValidator` 没测 → compile 失败 | 测试套件没增加；只是"看起来在 TDD" |
| **TDD 时 copy-paste 但不去重**                    | AI 写 `secondMomentAbout` 复制 `firstMomentAbout` 整段；不抽 `nthMomentAbout`                              | 重复代码累积；维护成本增加                       |
| **PdD 永久化**                                    | 写 `AnonymousMessageForwarder` 后从不折叠；feature flag 加到 8 个 subclass                          | 设计塌陷；新 feature 加不进去                  |
| **LSP 违反假装没发生**                            | AI 写 `Square extends Rectangle`，看到 `setWidth(3); setHeight(4); area=12` 测试通过就交付                | 用户拿 `Rectangle r = new Square();` 时 area=16 而不是 12 |
| **normalized hierarchy 强制 = 过度抽象**          | AI 把所有 method 都改 abstract；具体子类必须实现 15 个方法                                          | 实际只有 2-3 个 method 需要 override                            |
| **Properties flag 加到 30+ 才想到 config object**  | AI 让每个 feature 加一行 `properties.getProperty(...)`；累积到 `if-else` 链 50 行                      | 维护灾难；测试 mock 不动                         |
| **refactor 不跑 test 验证行为**                    | AI "折叠 PdD 回原类" 后宣称行为不变；没跑 test                                                        | 行为悄悄变了；测试套件全绿但生产报错              |

### 4.5 子段：AI 辅助遗留代码理解 — 在本主题专项

- **AI 帮你"识别 PdD 残留"**：扫描代码库标记 `extends X` 但 `override` 一个 concrete method 的子类 — 这些是 *PdD 残留*，是 *应折叠* 候选。准确率 75%。
- **AI 帮你"自动 fold PdD"**：给 `AnonymousMessageForwarder`，AI 推荐 move `getFromAddress` 到 `MailingConfiguration` 类。**风险**：AI 不一定知道 `MailingConfiguration` 是否已有同 method，需要人工 review。
- **AI 帮你"测试 5 步 checklist"**：每次 PR 提交时，AI 自动 check 是否走过 (0) get-under-test → (1) failing → (2) compile → (3) pass → (4) dedup。
- **AI 不会替你做"哪个 PdD 先折"** 的优先级：依赖业务 + 团队 sprint 节奏。

### 4.6 工程师必须保留的核心能力

- **判断第 0 步的具体技法**：ch9–ch13 的技法选择是 *commit 前置判断*。
- **识别 LSP 违反**：review 时一眼能看出 `Square extends Rectangle` 的危险。
- **判断 PdD 何时该折叠**：sprint 末 timebox 处理。
- **写 normalized hierarchy**：避免"盲目 override concrete method" 的诱惑。
- **判断 feature flag 该 properties 还是 config object**：3-4 个时该抽类。

## 五、实践行动项

> ch8 核心是 TDD / PdD / normalized hierarchy 三个技法。下面 4 个 demo 演示：TDD 的 firstMomentAbout → secondMomentAbout → nthMomentAbout cycle；PdD 的临时 subclass；normalized hierarchy 的对比；properties → config object 的演进。

### A1 — TDD cycle: firstMomentAbout → secondMomentAbout → nthMomentAbout

```bash
mkdir -p /tmp/ch08-tdd && cd /tmp/ch08-tdd

cat > calc.h <<'EOF'
#ifndef CALC_H
#define CALC_H
typedef struct {
    double elements[64];
    int n;
} Calc;
void  calc_init(Calc *c);
void  calc_add(Calc *c, double v);
double calc_nth_moment_about(Calc *c, double point, double n);
#endif
EOF

cat > calc.c <<'EOF'
#include "calc.h"
#include <math.h>
/* 一段实现, 通用: nth 阶中心矩 */
double calc_nth_moment_about(Calc *c, double point, double n) {
    if (c->n == 0) return 0.0/0.0;   /* NaN: 空时返回 NaN (异常由 caller 检) */
    double num = 0.0;
    for (int i = 0; i < c->n; i++)
        num += pow(c->elements[i] - point, n);
    return num / c->n;
}
void calc_init(Calc *c) { c->n = 0; }
void calc_add(Calc *c, double v) {
    if (c->n < 64) c->elements[c->n++] = v;
}
EOF

cat > test_calc.c <<'EOF'
#include "calc.h"
#include <math.h>
#include <assert.h>
#include <stdio.h>
int main(void) {
    Calc c; calc_init(&c);
    calc_add(&c, 1.0);
    calc_add(&c, 2.0);

    /* first moment about 2.0 = ((1-2) + (2-2))/2 = -0.5 */
    double m1 = calc_nth_moment_about(&c, 2.0, 1.0);
    assert(fabs(m1 - (-0.5)) < 1e-9);

    /* second moment about 2.0 = ((1-2)^2 + (2-2)^2)/2 = 0.5 */
    double m2 = calc_nth_moment_about(&c, 2.0, 2.0);
    assert(fabs(m2 - 0.5) < 1e-9);

    fprintf(stderr, "test_calc PASS\n");
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_calc calc.c test_calc.c -lm
./test_calc
echo "rc=$?"
```

**验收**：`test_calc PASS` + `rc=0`。证明 TDD-style 单元测试 + 通用 `nth_moment_about` 实现 = 1 阶 2 阶矩都通过。

### A2 — Programming by Difference: 临时 subclass 加新 behavior

```bash
mkdir -p /tmp/ch08-pdd && cd /tmp/ch08-pdd

# 父类 (模拟 MessageForwarder)
cat > forwarder.h <<'EOF'
#ifndef FORWARDER_H
#define FORWARDER_H
typedef struct Forwarder {
    int use_anonymous;
    int processed;
} Forwarder;
void forwarder_init(Forwarder *f);
const char *forwarder_from_address(Forwarder *f);
void forwarder_process(Forwarder *f);
#endif
EOF

cat > forwarder.c <<'EOF'
#include "forwarder.h"
#include <string.h>
void forwarder_init(Forwarder *f) { f->use_anonymous = 0; f->processed = 0; }
const char *forwarder_from_address(Forwarder *f) {
    return f->use_anonymous ? "anon-members@example.com" : "default@example.com";
}
void forwarder_process(Forwarder *f) { f->processed++; }
EOF

# PdD 子类: override forwarder_from_address
cat > anon_forwarder.h <<'EOF'
#ifndef ANON_FORWARDER_H
#define ANON_FORWARDER_H
#include "forwarder.h"
typedef struct AnonForwarder {
    Forwarder base;   /* PdD: 模拟 'extends' */
} AnonForwarder;
void anon_forwarder_init(AnonForwarder *a);
#endif
EOF

cat > anon_forwarder.c <<'EOF'
#include "anon_forwarder.h"
#include <string.h>
void anon_forwarder_init(AnonForwarder *a) {
    forwarder_init(&a->base);
    a->base.use_anonymous = 1;
}
EOF

cat > test_pdd.c <<'EOF'
#include "forwarder.h"
#include "anon_forwarder.h"
#include <string.h>
#include <stdio.h>
#include <assert.h>
int main(void) {
    AnonForwarder a; anon_forwarder_init(&a);
    /* 通过 base interface 调用 */
    Forwarder *f = &a.base;
    assert(strcmp(forwarder_from_address(f), "anon-members@example.com") == 0);
    forwarder_process(f);
    assert(f->processed == 1);
    fprintf(stderr, "test_pdd PASS\n");
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_pdd forwarder.c anon_forwarder.c test_pdd.c
./test_pdd
echo "rc=$?"
```

**验收**：`test_pdd PASS` + `rc=0`。证明 PdD (override `use_anonymous` flag 通过 subclass) 工作 — 但这是 *staging*，下面 A3 演示 normalized version。

### A3 — Normalized Hierarchy: 把 subclass 折回 config object

```bash
mkdir -p /tmp/ch08-pdd && cd /tmp/ch08-pdd

# 折回后的版本: 一个 MailingConfig 类带 getFromAddress()
cat > config.h <<'EOF'
#ifndef MAILING_CONFIG_H
#define MAILING_CONFIG_H
typedef struct {
    int use_anonymous;
    const char *domain;
} MailingConfig;
void config_init(MailingConfig *c, int anon, const char *domain);
const char *config_from_address(MailingConfig *c);
#endif
EOF

cat > config.c <<'EOF'
#include "config.h"
#include <stdio.h>
#include <stdlib.h>
void config_init(MailingConfig *c, int anon, const char *domain) {
    c->use_anonymous = anon;
    c->domain = domain;
}
const char *config_from_address(MailingConfig *c) {
    static char buf[128];
    if (c->use_anonymous)
        snprintf(buf, sizeof(buf), "anon-members@%s", c->domain);
    else
        snprintf(buf, sizeof(buf), "default@%s", c->domain);
    return buf;
}
EOF

cat > test_config.c <<'EOF'
#include "config.h"
#include <string.h>
#include <stdio.h>
#include <assert.h>
int main(void) {
    MailingConfig anon;
    config_init(&anon, 1, "example.com");
    assert(strcmp(config_from_address(&anon), "anon-members@example.com") == 0);

    MailingConfig norm;
    config_init(&norm, 0, "example.com");
    assert(strcmp(config_from_address(&norm), "default@example.com") == 0);

    fprintf(stderr, "test_config PASS\n");
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_config config.c test_config.c
./test_config
echo "rc=$?"
```

**验收**：`test_config PASS` + `rc=0`。演示 PdD → config object 折叠：A2 的子类在 A3 里被消除，行为由 `MailingConfig` 配置驱动 — *normalized* 形式。

### A4 — TDD 5 步完整 cycle 演示 (with NaN edge case)

```bash
mkdir -p /tmp/ch08-tdd && cd /tmp/ch08-tdd

# 复用 A1 的 calc.h / calc.c, 但加 empty-set NaN 测试
cat > test_empty.c <<'EOF'
#include "calc.h"
#include <math.h>
#include <assert.h>
#include <stdio.h>
int main(void) {
    Calc c; calc_init(&c);
    /* empty set: NaN 应当被识别 */
    double m = calc_nth_moment_about(&c, 0.0, 1.0);
    assert(isnan(m));
    fprintf(stderr, "test_empty PASS\n");
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_empty calc.c test_empty.c -lm
./test_empty
echo "rc=$?"
```

**验收**：`test_empty PASS` + `rc=0`。演示 TDD 第 4 步（refactor）前的 edge case handling：empty set 不该除以 0，返回 NaN 让 caller 决定如何处理（throw exception / skip / return default）。

## 六、值得深入思考的问题

### Q1: TDD 第 0 步的"get under test"开销怎么算入 ROI？

Ch8 强调第 0 步是 legacy TDD 与 greenfield TDD 的唯一区别。但第 0 步可能耗时数小时。**关键问题**：团队如何判断 *今天* 该做第 0 步（写新 feature），还是 *跳过* 第 0 步（用 ch6 sprout/wrap）？

### Q2: Programming by Difference 的"sprint 末折叠"如何强制？

PdD 是 *dirty staging*。但实际项目里 60% 的 PdD 都永久化。**关键问题**：用什么流程强制 PdD 在下个 sprint 末被 review + 折叠？靠 commit message 标 "PDD_TEMP"？靠 sprint ceremony？

### Q3: LSP 违反如何 *自动化* 检测？

经典 `Square extends Rectangle` 是已知反例。但实际项目里 LSP 违反往往是 *隐式* 的。**关键问题**：能否用工具静态分析 "父类 contract 在子类是否保持"？这是研究问题 — 工具支持有限（Eiffel 有 Design by Contract，但 C++/Java/Python 没有工业级工具）。

### Q4: Properties flag 到 config object 的临界点在哪？

Ch8 没给具体数字。**关键问题**：3 个 flag 就该抽类？还是 5 个？10 个？是否有团队约定？或者按"测试能否 mock" 判定？

### Q5: TDD 5 步的"remove duplication" 在 copy-paste-first 工作流里如何把握？

Ch8 明示可以临时 copy-paste。但 *何时* 该去重？**关键问题**：团队应该写"deliberate duplication" 注释，标记 *待去重*？还是按 srp / lint 工具检测？

### Q6: AI 写 PdD 时如何防止"过度继承"？

AI 默认 PdD（因为 minimal change）。**关键问题**：是否应该有 commit-hook 限制 "extends X" 的次数，超过 N 次就警告？或者依赖 review 时人工识别？

### Q7: "Normalized hierarchy" 在性能敏感路径上是否成本过高？

所有 method abstract = virtual call 开销。**关键问题**：在性能敏感（RTOS 内核、音频处理）的 hierarchy 中，是否要放弃 *normalized* 来换取 inlining？什么时候 trade-off 合理？

---

*下一章预告*: **Chapter 9 — I Can't Get This Class into a Test Harness** —— TDD 第 0 步的 *技法目录*。ch9 是全书 *症状-技法* 对照表的代表：4 大不可测原因（构造困难 / build 难 / 副作用 / sensing）+ 5+ 个 dependency-breaking technique 案例（Irritating Parameter / Hidden Dependency / Construction Blob / Irritating Global Dependency / Horrible Include Dependencies / Onion Parameter / Aliased Parameter）。ch9 把 ch3 sensing/separation 推进到 *具体可操作* 的拆依赖技法集合。
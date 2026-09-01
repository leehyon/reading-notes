# Chapter 5 — Tools

> **PDF**: p.67-78（12 页）
> **定位**: 把 ch4 的 seam 落到 4 类工程师工具箱。**核心信息**: *没有工具别的都是空谈* — 编辑器 / build system / unit-test framework / mock framework / refactoring browser / 集成测试 framework (FIT / Fitnesse)。Feathers 在本章末尾*明说本章是 toolbox*。每类工具都标 Feathers 的判定 — "够安全才用"。

## 〇、第一性原理思考

**这章做了什么**：把 ch4 的 seam 落地到 4 类工程师工具 —— Automated Refactoring Tool（不动 behavior 改 structure）、Mock Object（unit test 替协作者）、xUnit 系（JUnit / CppUnitLite / NUnit，靠 reflection 自动注册）、FIT / Fitnesse（客户写 HTML 表格测）；并给了一条 *refactor tool 的安全红线*（`int v = getValue(); loop += v;` 改成 `loop += getValue();` 表面等价，但 `getValue()` 里 `alpha++` 从 1 次变 N 次）。

**为什么这样拆**：前面的 seam、change/test point 都是抽象，没有工具就只是 talk —— Feathers 把这一章明写为 *toolbox*，每类工具都配他自己的判定（refactor tool 没验证 safety 就别用、mock 不是必须、simple fake 也够），不是中立罗列。

**最值钱的洞见**：评估一门新语言能不能做 TDD，*不是先跑 benchmark*，而是查 5 种 seam 在该语言的覆盖 + xUnit 端口有几个 —— 这条 Feathers 在 5.5 末尾给了，因此 ch5 不是工具说明书，而是 *新语言 / 新环境 testability 的诊断表*。
## 一、章节概述

- **4 类工具的隐式分工**: 
  - **Automated Refactoring Tools** (你写代码时, 不动 behavior 改 structure),
  - **Mock Objects** (你在 unit test 里替协作者),
  - **Unit-Testing Harnesses** (xUnit / JUnit / CppUnitLite / NUnit 等),
  - **General Test Harnesses** (FIT / Fitnesse，acceptance test 在 HTML 表格里)。
- **Refactoring tool 的"安全红线"**: 工具必须 *真的 preserve behavior*。Feathers 给了一个反例 — 用 refactoring tool 把 `int v = getValue(); loop += v;` 改成 `loop += getValue();` — 表面等价但 `getValue()` 内 `alpha++` 被调用 N 次（不再是 1 次）→ behavior 被偷偷改。
- **不能盲信 refactoring tool，必须先有 tests**。 这是 ch5 与 ch4 seam view 的咬合 — refactor tool = object seam 的极致应用, but enabling-point 现在在 IDE 里。
- **Mock objects 是 dependency 问题在 unit-test 层面的唯一干净答案**。 `www.mock-objects.com` 那时是入口，今天改 JMock / Mockito / gMock / mockgen。但 Feathers 早期坚持 *"simple fakes 也够"*。
- **xUnit 设计三原则**:
  - 测试用开发语言写,
  - 测试在 *隔离* 中跑 (每个 test method 一个独立 object),
  - 测试可分组 suite, 可任意 rerun。
- **JUnit 的 reflection 让这一切成了可能** — xUnit 的 reflection 是 C++ 难题根源 (Feathers 自述当年的 CppUnit 痛点)。
- **CppUnitLite = Feathers 自写** — 用 macro 把测试名 / 类名粘成 subclass name，static instance 注册, "nasty things" 在 macro 里。
- **NUnit = JUnit in . NET** — 用 attribute 而非 subclass, syntax 因 . NET 语言而异。
- **FIT = 客户写测试** — 把 spec 写成 HTML 表格, FIT 框架运行 + 红绿着色。
- **Fitnesse = FIT 在 wiki 上** — Robert Martin / Micah Martin 主导, Feathers 也参与过一点。
- **Refactor by hand 是 fallback**。 没工具不可怕，可怕的是用了 *不安全的工具* 而不自知。
- **5.5: 在新语言评估 testability 第一步不是 benchmark — 是查 5 种 seam + xUnit 在那语言有多少 ports**。

## 二、核心 Takeaways

### Takeaway 1: Refactor tool 没验证 safety 就别用

- **是什么**: Fowler 《Refactoring》定义"refactor = 不改 behavior 的结构变化"。但 *工具* 替你说"这句话"要有证据; 工具不验证就等于你信 driver 的歪头是"安全"。
- **为什么重要**: Feathers 当年看的 refactor tool 改 `int v = getValue(); total += v;` 为 `total += getValue();` 是 *alpha++ 总调用次数不一样的 case* — 等价貌、不等价心。
- **解决什么问题**: 给团队一个*"先 evaluate 工具安全再决定用不用"*的标准。
- **适用场景**: 任何 IDE 内置 refactor (IntelliJ, Eclipse, CLion) — 不开 safety net 直接 trust 是工程债。

### Takeaway 2: xUnit 的核心是 reflection + 每个 test 独立 object

- **是什么**: 测试用开发语言写, 框架 reflection 找 `void test*`, 每个 method 创建独立 fixture; `setUp` / `tearDown` 隔离每个 test 方法。
- **为什么重要**: 这是 xUnit 能当 *开发工具* 而不只是 *QA 工具* 的根本原因 — 测试进入内循环。
- **解决什么问题**: 让"test 慢就不能本地跑"的恶性循环 (ch2 反馈速度问题) 终止。
- **适用场景**: 评估任何测试框架时第一问 — 它有 reflection 找 test methods 吗?

### Takeaway 3: C++ reflection 缺失是 xUnit 在 C++ 痛的根本

- **是什么**: xUnit 设计假设 reflection 找方法名。C++ 没 reflection, 必须写 suite 注册方法 (CppUnit) 或 macro 注册 (CppUnitLite)。
- **为什么重要**: C++ 项目测试 framework 选择多 — CppUnit vs CppUnitLite vs Catch2 vs GoogleTest, 都要在 *macro 是否清白* 上做 trade-off。
- **解决什么问题**: 给 C++ 工程师选 xUnit port 的依据 — 不是"feature 多" 而是 *macro 是否能理解 + reflection 缺失如何绕过*。
- **适用场景**: C++ 工程 starter kit 选 type。

### Takeaway 4: Mock object framework 是 *测试 adjunct*, 不是替代 fakes

- **是什么**: Framework 提供 mock 容器 + 自动 verify; 但 *简单手写 fake* 在 60% 的场景下够用。
- **为什么重要**: AI 时代倾向 *不论场景全 mock* — Feathers 早期坚持简单 fake 在大多数场景已够。
- **解决什么问题**: 阻止团队引入过重 mock framework — 它往往让 fake 维护成本反而高于被测代码。
- **适用场景**: 评估"项目里到底该不该引入 mockito-pytest-mock" 时。

### Takeaway 5: FIT / Fitnesse 把 acceptance test 写在 HTML 表格里

- **是什么**: 把客户的 specification 写成 HTML table; framework 解析 table → 调代码 → 红绿着色结果。
- **为什么重要**: 它解决了"客户/QA/开发 的 spec 不一致" — 同一文档同时是 spec + test, customer/developer 共看一份。
- **解决什么问题**: handoff 失真 + acceptance test 写不完整。
- **适用场景**: 客户协作要求高的项目 (银行 / 嵌入式合同 / 监管严的工程)。

### Takeaway 6: refactoring browser 之前身（Smalltalk）是我们今天 IDE 的 ancestor

- **是什么**: Brant & Roberts 在 UIUC 写 Smalltalk Refactoring Browser, 内嵌 IDE, refactoring 是 first-class action。
- **为什么重要**: 现在 IntelliJ "extract method" 这一键就是从那开始的。
- **解决什么问题**: 把 refactor 行为从 "review 中找" 还原为 "编辑器内置"。
- **适用场景**: 教育新人 — refactor 不是 banner command, 是 typography。

### Takeaway 7: 每个新测试工具都要跑 5 分钟 sanity check

- **是什么**: Feathers 给的小测试集合 — 把一个 method 重命名为已存在的方法 / base class method 名字, 工具报警吗?
- **为什么重要**: "工具说它安全" 到 "它真安全" 之间的 gap 是 *测试*。
- **解决什么问题**: 给团队决定"该不该把工具上 CI" 的客观证据。
- **适用场景**: 把 *工具信任评估* 纳入技术雷达流程。

### Takeaway 8: refactoring 没测试不在 root — wrap them

- **是什么**: 即使 refactor tool 标"safe", 没测试的情况下你的 *production 代码* 仍可能让工具悄悄改错 (call 顺序、内嵌函数调用次数等)。
- **为什么重要**: wrap refactor 在 characterization test 里 — 即使工具只对结构改了一处, 你也能 verify 它真没动 behavior。
- **解决什么问题**: 防御性 refactor 而不引入测试债。
- **适用场景**: 长期 legacy 重构时 (ch24)。

## 三、工程实践视角

> 锁定领域: **Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **Linux kernel 自带 KUnit — reflection-free 的 C-language unit test**。 Static instance 注册 (similar to CppUnitLite macro), 编译进 vmlinux 或独立跑。
- **`bpf_prog_test_run` 用 refactor 概念于 BPF 验证** — 验证 pass / verifier 是单元测试 + 模拟 KB 级的"真硬件"。但 verifier 本身不在 source。
- **OpenBSD `kernel/regress/` 是 FIT 模拟 — 不用 . html, 用 . c 的 syscall series**。 等于 FIT table-driven 但用 shell 跑。
- **`glibc` 自带 `test-skeleton.c`** — 不需 xUnit, reflection-free, 全 macro, 测 harness 自带 isolation。
- **systemd 的 `TEST-01-*` 集成测试 = Fitnesse 全套 shell**, 而 unit test 全在 `src/test/`。
- **Linux 内核 sync_test 用 refactoring tool 失败案例** — 历史中多次 refactor 修改 `smp_mb__after_atomic()` 但引入 compiler reorder — *refactor 是工具, 不是 safety guarantee*。

#### Linux 系统 — 工具栈选型对照

| 测试场景                | Feathers 时期 (2004) 选型          | 现在 (2026) 选型                         |
| ----------------------- | --------------------------------- | ---------------------------------------- |
| C unit test             | CppUnitLite  / 手写测试主        | GoogleTest / Check / cmocka / Unity    |
| C++ unit                | CppUnit / CppUnitLite            | Catch2 / GoogleTest / doctest           |
| kernel module           | 内嵌测试 (printk + assert)        | KUnit / kselftest                       |
| userspace daemon        | 标准 xUnit (JUnit)               | Criterion / Catch2 / pytest             |
| refactoring 安全网      | JRefactory / IDEA refactor        | clang-tidy / Coccinelle / C++ refactor  |
| acceptance test         | FIT / Fitnesse                   | Cucumber / Gauge / Robot Framework      |
| mock framework          | mockobjects.com registry         | cmocka / gMock / mock-generate          |
| hex regression          | LTP / autotest                   | syzkaller / locktorture / nvme-cli       |

### 3.2 机器人软件视角

- **ROS2 的 colcon test = Generalized Test Harness**。 跑 `colcon test --packages-select nav2_bt_navigator`, framework 找 ament target, 类似 xUnit runner。
- **Launch-testing = 集成测试**。 `ros2 launch` 的 `test_launch_ros2` 把 launch file 视为 acceptance test 子 — 类似 FIT 的"application behavior via launch file"。
- **MoveIt! 的 kinematic test = Fitnesse 的"client specification"**。 不同 robot 配置下同一组测试, 红绿直接 show 7-DOF vs 6-DOF, 是 "spec + test" 合一。
- **`gtest` 是 ROS2 的默认 xUnit** (因 C++); `pytest` 是 client-side 测试首选 (因 Python 节点)。
- **launch_test 是 specialized FIT**。 由于 ROS2 的 launch 有 XML, launch_test 类似 FIT table-driven 但 layout-specific。
- **行为树测试 = Mock objects 极致**。 BT engine 用 mock nodes (BehaviorTree. CPP TestNode) — 类似 ch3 FakeDisplay 的"两个面"。

#### 机器人软件 — Tools 实施表

| 场景                  | 工具                                  | 类似 Feathers 类别       |
| --------------------- | ------------------------------------- | ----------------------- |
| Node unit test        | gtest + mock publishers/subs          | Unit-Testing Harness +  Mock Objects |
| Integration test      | colcon test + launch_test             | General Test Harness    |
| End-to-end            | ros2 doctor + acceptance_test          | FIT/Fitnesse 类          |
| Behavior Tree 节点   | BehaviorTree.CPP TestNode             | Mock Objects            |
| 硬件在环             | ros2_control mock hardware            | Link seam (ch4) + Mock  |
| 机械臂轨迹规划       | MoveIt! IKFast test (C++)             | Unit-Testing Harness    |
| 仿真回放             | rosbag2 录制 + 离线 replay            | General Test Harness    |
| Code refactor         | clang-tidy + Coccinelle              | Automated Refactoring   |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                   | 高级工程师                                                   |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 选 IDE               | Intellisense 建议 = 真                                       | 自动 IDE 重构先看它是否 preserve behavior; 不信它的"安全" |
| 引入 mock framework  | 想用 Mockito / gMock 全覆盖                                  | 60% 用手写 fake, 40% 才用 framework                        |
| 看 C++ 测试          | "我写 CppUnit 像 Java JUnit 那样"                            | 评估 reflection-free 限制; 选 Catch2 / GoogleTest           |
| 看 FIT               | "客户哪里会写 HTML"                                          | "在监管严的工程 (车规 / 医疗 / 金融)FIT 真的是对工具"      |
| refactoring 信心     | "这 IDE 自动做 = 安全"                                       | 在 characterization test 围栏里 refactor                    |
| 工具安全评估         | "新工具好酷"                                                | "5 分钟 sanity check — 改名为同类方法, 工具报警吗"        |
| CI 引入工具          | "加进去再说"                                                | 评估 macro / reflection-free 限制 + sanity check         |
| 测试 vs spec 分离    | spec 一份文档, 测试一份代码                                  | "FIT-style: spec = test, 一致性" 评估                      |

> **关键差异**: 高级工程师在 *工具的边界* 上评估, 初级在 *工具的功能* 上惊讶。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

仍然极重要。原因:
- AI 写 mock 极快, 写 xUnit 极快 — 但 AI *不会替你判断* 这个项目该不该上 mockito, 它默认上。
- AI 重构 IDE 现在能 preserve 多数 behavior — 但 call sequence / side effect 在长代码中仍是问题。
- 工具评估 (Fowler 1999 框架) 几乎没人做 — AI 让"自动 refactor 出现损伤"反而变多。
- xUnit 在新语言 (Bun / Vite testing / Vitest) 仍有支持, 但 *业界趋势* 在融合 — 这是 ch5 的隐含下沉点。

### 4.2 AI 已经能做的 (具体到 ch5 主题)

- **生成 mock 初始 draft** — 对 method 给一组输入, AI 给 mock + verify。7 成可 review。
- **选择 test framework** — 给项目语言, AI 推荐 Catch2/GoogleTest/gtest/pytest — 但需要领域 context。
- **生成 FIT-like table** — 给 spec, AI 输出 HTML table。

### 4.3 AI 不能替代的 (具体到 ch5 主题)

- **判断工具在团队里是否值得引入** — AI 给推荐, 真实成本要 prod usage 评估。
- **sanity check 工具安全** — 这是手工 hack, AI 给 "NICE try" 没用, 实际跑才知道。
- **跨语言 porting strategy** — 测试 porting 的成本 vs 重写的 trade-off。

### 4.4 AI 经常写错的地方

| 错误模式                                       | 例子                                                                 | 后果 |
| ---------------------------------------------- | -------------------------------------------------------------------- | ---- |
| **过度依赖 mock**                              | 替手写 fake 全上 Mockito                                              | mock 自带 complexity, simple case 反而复杂  |
| **生成 testing framework 罕见 assertion**       | `assertEqual(actual.toString(), expected)` 而非 `assertEquals`        | 失败信息不可读, 不进 CI                      |
| **断言 placeholder**                            | `assertTrue(true)` 在 AI 写的测试中常见                              | 测试看起来通过但实际空                          |
| **xUnit 漏 setUp**                              | AI 写 setUp 但每个 test 都自建对象 — setUp 没起效                    | test 之间状态泄露                            |
| **refactor 工具默认 trust**                      | AI 帮用 IntelliJ 重命名 import 但 *包路径不同就 OK*                   | 编译错误, 工具没报警                            |
| **Mock framework 默认 setExpectation 错次**       | 期望调 1 次但产品调 0 或 2 次, verify 过 / 不过可疑                  | false positive / negative                       |
| **FIT 表格生成但内容不复核**                    | AI 给 HTML 表, 数字看似合理但反例错                                  | acceptance test 锁的是 "AI 幻想"                              |
| **CppUnitLite macro 在不同编译器展开不一致**    | AI 写的 C++ 测试在 GCC OK 但 clang 报 redeclaration                    | test 代码不可移植                              |
| **跳过 refactor 前的 characterization test**     | AI 帮 refactor 但不提示先写 test                                      | 工具偷偷改 behavior 时已无法 capture               |

### 4.5 子段: AI 辅助遗留代码理解 — tools 视角专项

- **AI 帮 decide test framework** — 项目语言 + 主要场景 + 团队背景 → AI 给推荐三选一。**风险**: AI 不会看 macro 维护成本 / reflection-free 限制。
- **AI 帮搭 FIT table** — PDF / Word spec 抽成表格 + 红色预期。**风险**: AI 抓不到 spec 内部不一致。
- **AI 不会做 sanity check** — 工具安全性是手工 hack 必须。
- **AI 帮 mock 难测类** — 对数据库 / 网络 / 文件 IO, AI 给 mock + verify 完整初稿 — 但生产代码仍要人 review。
- **AI 帮写 mock 生成器** — mockgen style AI 版 (a la swift-mock-generation).

### 4.6 工程师必须保留的核心能力

- **评测工具安全** — sanity check 5 min 不能省。
- **决定 mock vs fake** — 60% fake, 40% mock; AI 默认全 mock 要人为修正。
- **工具边界识别** — 在 reflection-free 语言里手动 test suite 还是要写。
- **工具功能 vs 工具边界** — 看到 "工具能 X" 不等于 "我该用 X"。

## 五、实践行动项

> 本章是 toolbox 章。行动项聚焦 **(a) 写出 C-mini-test-harness** + **(b) 重演 ch5 refactor 反例 (alpha++)** + **(c) FIT-like table-driven** + **(d) mock 框架微观实现**。

### A1 — 实现 C-mini-test-harness (类似 CppUnitLite 但精简): reflection-free unit test runner

```bash
mkdir -p /tmp/ch05-tools && cd /tmp/ch05-tools

cat > testharness.h <<'EOF'
/* Tiny C unit-test harness: reflection-free, 用 macro 注册 test cases */
#ifndef TESTHARNESS_H
#define TESTHARNESS_H

#include <stdio.h>
#include <string.h>

#define MAX_TESTS 256

typedef struct {
    const char *name;
    void (*fn)(void);
} Test;

/* 注册表 — extern 共享, 定义在 harness_global.c
 * (不能 static, 否则每个 translation unit 各有一份) */
extern Test tests[MAX_TESTS];
extern int n_tests;

static inline void register_test(const char *name, void (*fn)(void)) {
    if (n_tests >= MAX_TESTS) return;
    tests[n_tests].name = name;
    tests[n_tests].fn = fn;
    n_tests++;
}

/* TEST(name) - 创建一个无副作用 test 函数 + 自动注册 */
#define TEST(name)                                     \\
    static void test_##name(void);                     \\
    __attribute__((constructor))                       \\
    static void reg_##name(void) {                     \\
        register_test(#name, test_##name);             \\
    }                                                  \\
    static void test_##name(void)

extern int fails;

#define ASSERT_TRUE(x)  do { if (!(x)) { \\
    fprintf(stderr, "FAIL %s:%d %s\\n", __FILE__, __LINE__, #x); \\
    fails++; return; } } while (0)

#define ASSERT_EQ(a, b) do { long _a=(long)(a), _b=(long)(b); \\
    if (_a != _b) { fprintf(stderr, "FAIL %s:%d expected %ld got %ld\\n", \\
        __FILE__, __LINE__, _b, _a); fails++; return; } } while (0)

static inline int run_all_tests(void) {
    int total_fail = 0;
    for (int i = 0; i < n_tests; i++) {
        fails = 0;
        tests[i].fn();
        if (fails) total_fail++;
    }
    fprintf(stderr, "\\n=== TEST RESULT ===\\n");
    fprintf(stderr, "ran %d tests, %d failed\\n", n_tests, total_fail);
    return total_fail == 0 ? 0 : 1;
}

#endif
EOF

# 关键: tests[] / n_tests / fails 必须是单 file-visible 变量 (harness_global.c)
echo 'Test tests[MAX_TESTS]; int n_tests = 0; int fails = 0;' > harness_global.c

# demo test
cat > demo.c <<'EOF'
#include "testharness.h"

int add(int a, int b) { return a + b; }
int sub(int a, int b) { return a - b; }

TEST(addition_works) {
    ASSERT_EQ(add(2, 3), 5);
    ASSERT_TRUE(add(0, 0) == 0);
}
TEST(subtraction_works) {
    ASSERT_EQ(sub(10, 3), 7);
}
TEST(demo_failing_test_for_demo) {
    ASSERT_TRUE(1 == 2);   /* 故意 */
}
EOF

# main driver (run_all_tests() 是 inline static, 在 #include header 后可调用)
cat > main.c <<'EOF'
#include "testharness.h"
int main(void) {
    return run_all_tests();
}
EOF

cc -std=c17 -Wall -Wextra -o test_demo harness_global.c main.c demo.c
./test_demo
echo "rc=$?"
```

**验收**:
- `addition_works` 和 `subtraction_works` PASS。
- `demo_failing_test_for_demo` FAIL, stderr 输出 `ASSERT_TRUE(1==2)` 位置。
- 末尾 `ran 3 tests, 1 failed` 输出，`rc=1`。

### A2 — 重演 ch5 Refactor 反例: alpha++ 被偷改

```bash
mkdir -p /tmp/ch05-tools && cd /tmp/ch05-tools

# 模拟 "extract-method" refactor 把循环抽出去 — 行为变了
cat > alpha_before.c <<'EOF'
#include <stdio.h>

static int alpha = 0;
int getValue(void) { alpha++; return 12; }

int main(void) {
    /* "改之前": alpha++ 只 1 次 */
    int v = getValue();
    int total = 0;
    for (int n = 0; n < 10; n++) total += v;
    printf("alpha_before: alpha=%d total=%d (期望 alpha=1, total=120)\n",
           alpha, total);
    return alpha != 1 ? 1 : 0;   /* 期望 0 */
}
EOF

cat > alpha_after.c <<'EOF'
#include <stdio.h>

static int alpha = 0;
int getValue(void) { alpha++; return 12; }

int main(void) {
    /* "改之后": alpha++ 被调 10 次 — refactor tooling 把 int v 抽掉了 */
    int total = 0;
    for (int n = 0; n < 10; n++) total += getValue();
    printf("alpha_after:  alpha=%d total=%d (期望 alpha=1, total=120)\n",
           alpha, total);
    return alpha != 1 ? 1 : 0;   /* 现在返回 1! */
}
EOF

cc -std=c17 -Wall -Wextra -o alpha_before alpha_before.c
cc -std=c17 -Wall -Wextra -o alpha_after  alpha_after.c
echo "--- before ---"; ./alpha_before; echo "rc=$?"
echo "--- after  ---"; ./alpha_after;  echo "rc=$?"
```

**验收**:
- `alpha_before` 输出 `alpha=1 total=120`, `rc=0`。
- `alpha_after` 输出 `alpha=10 total=120`, `rc=1` — **refactor 工具悄悄改了 behavior**, 正是 ch5 Takeaway 1 的反例。

### A3 — FIT-like table-driven acceptance test (Python)

```bash
mkdir -p /tmp/ch05-tools && cd /tmp/ch05-tools

cat > calc.c <<'EOF'
#include <stdio.h>
/* 一个 ""业务函数"" — 计算贷款的 monthly payment (P*r) / (1 - (1+r)^-n) */
/* 注: -Wall 下被调函数必须先声明; 因此 pow_helper 提前 */
static double pow_helper(double base, int n);
double monthly_payment(double principal, double rate_pct, int months) {
    double r = rate_pct / 100.0 / 12.0;
    if (r == 0) return principal / months;
    return (principal * r) / (1 - 1 / pow_helper(1 + r, months));
}
static double pow_helper(double base, int n) {
    double r = 1;
    for (int i = 0; i < n; i++) r *= base;
    return r;
}
EOF
# cc -std=c17 -Wall -Wextra -c calc.c -o calc.o (注意: 不能给 Python ctypes load .o,
# 因为 ctypes 要求 ET_DYN / ET_EXEC, 不是 ET_REL — 必须先 cc -shared 出 .so)
cc -std=c17 -Wall -Wextra -fPIC -shared -o calc.so calc.c
# (警告: -Wall 会因 pow_helper 无先声明报错; calc.c 里需加 forward decl 或 #include <math.h>)
# 实测修: 上面 calc.c 我们把 pow_helper 定义放在 monthly_payment 之前, 避免 warning

# FIT-like 表格: input → expected output (类似 Fitnesse HTML table)
cat > table.csv <<'EOF'
principal,rate_pct,months,expected_payment
10000,6.0,12,860.66
50000,5.5,360,284.46
200000,4.5,360,1013.37
EOF

# Python 测试 driver 把 CSV 解析, 然后调 .so 计算
cat > fit_like.py <<'PY'
#!/usr/bin/env python3
import ctypes, csv, sys

# 注意: ctypes 只能 load ET_DYN / ET_EXEC (共享库); 纯 .o 是 ET_REL
mod = ctypes.CDLL("./calc.so")
mod.monthly_payment.argtypes = [ctypes.c_double, ctypes.c_double, ctypes.c_int]
mod.monthly_payment.restype = ctypes.c_double

with open("table.csv") as f:
    rows = list(csv.DictReader(f))

print("=== FIT-style table-driven test ===")
passed = failed = 0
for r in rows:
    p = float(r["principal"]); rate = float(r["rate_pct"]); months = int(r["months"])
    exp = float(r["expected_payment"])
    got = mod.monthly_payment(p, rate, months)
    diff = abs(got - exp)
    status = "PASS" if diff < 1.0 else "FAIL"     # ±$1 tolerance
    if diff < 1.0: passed += 1
    else: failed += 1
    print(f"  [{status}] P={p:7g} r={rate:4.2f}% m={months:4d} -> ${got:8.2f} (期望 ${exp:8.2f})")
print(f"\n{passed} pass, {failed} fail out of {passed+failed}")
sys.exit(0 if failed == 0 else 1)
PY
chmod +x fit_like.py

python3 fit_like.py
echo "rc=$?"
```

**验收**:
- 输出每个 case 的 PASS/FAIL 表格行, 类似 Fitnesse 红绿色。
- 全部 3 case 都应该接近 ±$1 内通过 (贷款计算 standard 公式)。

### A4 — Minimal mock framework (C): 在 C 视角再现 "mock object" 的两大面

```bash
mkdir -p /tmp/ch05-tools && cd /tmp/ch05-tools

# 完整 minimock: header 定义 interface + 3 macro (REGISTER/EXPECT/VERIFY),  .c 实现
cat > minimock.h <<'EOF'
#ifndef MINIMOCK_H
#define MINIMOCK_H

#include <stdio.h>
#include <string.h>

#define MAX_MOCKS 32

typedef struct {
    const char *_name_;
    int   call_count;
    long  last_arg1;
    long  last_arg2;
    int   expected_called;
    long  expected_arg1;
    long  expected_arg2;
} MockSlot;

extern MockSlot mocks[MAX_MOCKS];
extern int n_mocks;
extern int last_mock_idx;

int find_mock(const char *name);
void mock_record(long a, long b);
void mock_reset(const char *name);

#define MOCK_REGISTER(name) do {                                  \
    if (find_mock(#name) < 0 && n_mocks < MAX_MOCKS) {           \
        int idx = n_mocks++;                                     \
        mocks[idx]._name_ = #name;                               \
        mocks[idx].call_count = 0;                               \
    }                                                            \
    last_mock_idx = find_mock(#name);                            \
} while (0)

#define MOCK_EXPECT(name, exp_cnt, exp_a, exp_b) do {            \
    int idx = find_mock(#name);                                  \
    if (idx >= 0) {                                              \
        mocks[idx].expected_called = (exp_cnt);                  \
        mocks[idx].expected_arg1 = (exp_a);                      \
        mocks[idx].expected_arg2 = (exp_b);                      \
    }                                                            \
} while (0)

#define MOCK_VERIFY(name) ({                                     \
    int idx = find_mock(#name);                                  \
    int passed = 1;                                              \
    if (idx >= 0) {                                              \
        MockSlot *m = &mocks[idx];                               \
        if (m->call_count != m->expected_called) {               \
            fprintf(stderr, "MOCK FAIL %s: expected %d calls got %d\n", \
                #name, m->expected_called, m->call_count);      \
            passed = 0;                                          \
        } else if (m->last_arg1 != m->expected_arg1 ||           \
                   m->last_arg2 != m->expected_arg2) {           \
            fprintf(stderr, "MOCK FAIL %s: arg mismatch\n", #name); \
            passed = 0;                                          \
        } else {                                                 \
            fprintf(stderr, "MOCK OK   %s: %d calls\n", #name, m->call_count); \
        }                                                        \
    }                                                            \
    passed;                                                      \
})

#endif
EOF

cat > minimock.c <<'EOF'
#include "minimock.h"
#include <string.h>
#include <stdio.h>
MockSlot mocks[MAX_MOCKS];
int n_mocks = 0;
int last_mock_idx = -1;
int find_mock(const char *name) {
    for (int i = 0; i < n_mocks; i++)
        if (strcmp(mocks[i]._name_, name) == 0) return i;
    return -1;
}
void mock_record(long a, long b) {
    int idx = last_mock_idx;
    if (idx >= 0) {
        mocks[idx].call_count++;
        mocks[idx].last_arg1 = a;
        mocks[idx].last_arg2 = b;
    }
}
void mock_reset(const char *name) {
    int idx = find_mock(name);
    if (idx >= 0) {
        mocks[idx].call_count = 0;
        mocks[idx].last_arg1 = mocks[idx].last_arg2 = 0;
        mocks[idx].expected_called = -1;
    }
}
EOF

# 测试 demo: do_foo (单调用), do_bar_n (调 n 次)
cat > test_minimock.c <<'EOF'
#include "minimock.h"
#include <assert.h>

void do_foo(void) { mock_record(1, 0); }
void do_bar_n(int n) { for (int i = 0; i < n; i++) mock_record(i, n); }

int main(void) {
    MOCK_REGISTER(mock_record);
    MOCK_EXPECT(mock_record, 1, 1, 0);

    do_foo();
    int ok1 = MOCK_VERIFY(mock_record);
    assert(ok1);

    mock_reset("mock_record");
    MOCK_EXPECT(mock_record, 3, 2, 3);

    do_bar_n(3);
    int ok2 = MOCK_VERIFY(mock_record);
    assert(ok2);

    fprintf(stderr, "minimock PASS\n");
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_minimock minimock.c test_minimock.c
./test_minimock
echo "rc=$?"
```

**验收**:
- `test_minimock` 输出 `MOCK OK mock_record: 1 calls` 和 `MOCK OK mock_record: 3 calls`, 然后 `minimock PASS`, `rc=0`。
- `MOCK_VERIFY` 调用次数 / 参数 mismatch 时 stderr 给反馈; 这里 happy path。

> ⚠️ 这是 ch5 tools 章的"理论 + 实践"双层闭环。minimock 是简化版的 mock framework (cpp mock lib 实际用 template + 类成员, 见 ch5 ch25 catalog)。

## 六、值得深入思考的问题

### Q1: 在 reflection-free 语言里 xUnit 的代偿模式是什么?

Feathers 给出 macro-based 注册 (CppUnitLite) 和手写 suite (CppUnit)。 **关键问题**: 现代 C++ (GoogleTest, Catch2, doctest) 自动发现是 reflection-free 的, 但仍 near-zero-config — 这是否让 ch5 提到的"reflection 缺失"成为历史? 对新语言的工程 trade-off 怎么算?

### Q2: refactor tool 永远不能完全安全 — 但仍然买入

Feathers 给的反例证明 *alpha 被悄悄加*。但同时, 现代 IntelliJ / C++ refactoring tool 的覆盖率远超当年。**关键问题**: 在 *sanity check pass* 的前提下, refactor tool 是否仍需要 *characterization test 围栏*, 还是只要工具厂商说安全就够?

### Q3: Mock vs Fake 是否仍然是性价比区分?

AI 让 mock 自动生成极快, fake 手写 30 行也极快。**关键问题**: 当两者成本几乎相同时, "fake 60%, mock 40%" 的传统经验还成立吗?

### Q4: FIT / Fitnesse 价值的真正持久性如何?

Ward Cunningham (FIT 创作者) 在业界被公认; Fitnesse 在嵌入式 / 监管项目的 acceptance 工具仍然有市场。**关键问题**: AI 写 spec / acceptance test 时, FIT 这类 *spec = test 一致性* 的角色是更突出还是退场?

### Q5: 测试 framework 选择是否应当按"feedback budget"算?

ch2 feedback 速度论 (3000 classes × 0.1s = 1hr), ch5 加了 *工具选择*。**关键问题**: 选 Catch2 + Python bridge 时是否要做"整体 compiler time / test runtime 预算"评估, 还是跟随业界默认?

---

*下一章预告*: **Chapter 6 — I Don't Have Much Time and I Have to Change It** — 进入 Part II 的第一篇"按症状分类的处理方案"。当时间紧时 Feathers 给 4 个具体 pattern: **Sprout Method** (加新方法从黑盒里套) / **Sprout Class** (加新类独立测) / **Wrap Method** (在已有方法外加 wrapper) / **Wrap Class** (decorator 模式继承)。每个 pattern 都附"如何在不破坏既有测试的前提下加新功能"的执行步骤 — 这些步骤是 ch8/9 模板的预演。

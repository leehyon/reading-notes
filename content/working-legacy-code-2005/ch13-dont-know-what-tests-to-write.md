# Chapter 13 — I Need to Make a Change, but I Don't Know What Tests to Write

> **PDF**: p.207-218（12 页）
> **定位**: ch11/ch12 解决了"在哪写测试"；ch13 解决"**写什么样的测试**"。核心武器是 **characterization test** —— 一类专门"捕捉"既有行为的测试：写必然失败的断言，让失败信息告诉你真实行为，把断言改成真实期望。ch13 与 ch8 形成对照 —— ch8 是 "Add a Feature" 时写新测试，ch13 是 "Change / Refactor legacy" 时写既有行为快照。

## 〇、第一性原理思考

**这章做了什么**：给出 characterization test 的 5 步算法（写必然失败的断言，让失败信息告诉你真实行为，把断言改成真实期望），并配套 4 条 charactering classes 的启发式和 targeted testing 注意事项。

**为什么这样拆**：ch11/ch12 解决了"在哪写测试"，但 legacy code 没有 spec 可以"验证正确性"；所以测试目的需要从"判断正确"翻转为"记录现状" —— 用测试当 spec 的替代品。

**最值钱的洞见**：Characterization test 不是"断言应该是什么"，是"断言现在是什么"；关键是 inversion —— 故意写错的断言，让失败信息本身成为 spec 的来源，所以实际上测试是 spec 的勘探工具，不是 spec 的裁判。
## 一、章节概述

- **Characterization Test 的定义**：测试目的不是判断"行为是否正确"，而是"行为是什么"。把当前返回值 / 副作用当作"权威"，让未来改动触发差异告警。
- **5 步算法（Feathers 给的）**：(1) 在 test harness 里用一段代码；(2) 写一个必然失败的断言；(3) 让失败告诉你行为；(4) 把断言改成期望真实值；(5) 重复。
- **道德权威的放弃**：characterization test 不"应该"是什么 —— 它就是"它是什么"。把软件的实际行为当作 spec，让改动后的差异自然浮现。
- **Characterizing Classes 4 条启发式**：(a) 看 tangled 逻辑，引入 sensing variable；(b) 列"会出错的点"并写 trigger；(c) 输入边界值；(d) 写 invariant 测试（class lifetime 内永远成立的 condition）。
- **Method Use Rule**：使用 legacy 方法前先看是否有测试；没有就写。这条 rule 让测试 = communication medium —— 任何方法的使用记录都有测试背书。
- **Targeted Testing**：写完 characterization 之后，针对 change point 看测试是否覆盖。FuelShare / ZonedHawthorneLease 的例子讲的是：分支 / 转换 / 边界值都要 trigger。
- **Double / Float / Int 转换陷阱**：Java/C# 自动从 double 转 int 会截断。Feathers 警告：选测试输入时必须让"如果误用类型"会立即露出。**Test for behavior, but design input to fail loudly when wrong**。
- **Refactoring 工具的 quirk**：自动 extract-method 工具对 instance variable 处理不一致（可能 `add(y)` 而不是 `add(x,y)`）—— 测试必须 verify 方法签名。
- **A Heuristic for Writing Characterization Tests**（3 步）：(1) 在改动区域写够多测试直到理解；(2) 针对要改的具体处再写；(3) verify 行为存在且连接正确（exercise conversions）。
- **When You Find Bugs**：behavior 错误的处理 —— deployed 系统慎改（可能有人依赖"bug"），undeployed 直接修。Feathers 偏"找到就修"，但要 escalate。
- **与 ch8 对照**：ch8 是 Add a Feature（用 test-driven 给新行为写 spec）；ch13 是 Change/Refactor legacy（用 characterization test 给既有行为写 spec）。两者方向相反。

## 二、核心 Takeaways

### Takeaway 1：Characterization test 是"行为快照"而非"行为规范"

- **是什么**：测试目的从"判断正确性"转为"记录现状"。断言不写"应该是什么"，写"现在是什么"。
- **为什么重要**：legacy code 的 spec 通常不存在 / 不可信 —— characterization test 把"现状"当作 spec，让未来改动触发告警。
- **解决什么问题**：在没有 spec 的代码库里建立可观察的行为记录 —— refactor 时用它作 safety net。
- **适用场景**：任何对 legacy code 的改动；任何需要重构但缺乏 spec 的项目。

### Takeaway 2：5 步算法是 characterization test 的标准节奏

- **是什么**：(1) 跑代码；(2) 写必然失败断言；(3) 让失败告诉你行为；(4) 改成真实值；(5) 重复。这是 ch13 p.186 给的硬话。
- **为什么重要**：避免"凭想象写断言" —— 让软件本身告诉你行为。
- **解决什么问题**：characterization test 不需要先理解代码 —— 先跑起来，让跑的结果驱动理解。
- **适用场景**：第一次接触陌生 legacy 类；对方法行为不确定时。

### Takeaway 3：放弃"道德权威"是写 characterization test 的心理转换

- **是什么**：传统测试把自己当"权威"（"代码必须这样"）；characterization test 放弃权威，把"现状"当 spec。**测试不再判断对错，只记录差异**。
- **为什么重要**：让测试从"开发 vs 代码"的对立变成"开发 + 测试 vs 未知"的合作。
- **解决什么问题**：让团队敢于"记录可能错的现状"，把判断推迟到有充分信息时。
- **适用场景**：第一次写 characterization test；引入 characterization test 到团队时。

### Takeaway 4：Characterizing Classes 的 4 条启发式

- **是什么**：(a) 看 tangled 逻辑，用 sensing variable trigger；(b) 列可能出错的点；(c) 输入边界值；(d) 写 invariant 测试。
- **为什么重要**：把"无穷可能输入"压成"4 类必测"。
- **解决什么问题**：避免"该测什么"成为开放式问题。
- **适用场景**：任何 characterization test session 的开头。

### Takeaway 5：Targeted Testing = characterization 之后针对 change point 加测

- **是什么**：写完 characterization 不代表覆盖 change point。**针对要改的具体分支 / 转换写测试**，确保改动被 sense 到。
- **为什么重要**：characterization 覆盖"行为存在"，targeted testing 覆盖"行为对 change 敏感"。
- **解决什么问题**：避免"测试 100 行代码但漏了要改的那 3 行"。
- **适用场景**：任何 refactor / bugfix 前的最后一步。

### Takexture 6：测试输入必须 trigger 误用时的失败

- **是什么**：选测试输入时考虑"如果方法被错误实现（比如参数类型错），输入是否能让差异立即可见"。例：FuelShare 的 `1.2 * price` 如果误用 `int` 而非 `double`，应选让截断可见的输入。
- **为什么重要**：让 silent failure 变成 loud failure。
- **解决什么问题**：避免"测试通过但实际 bug 已被掩盖"。
- **适用场景**：任何涉及类型转换 / 精度 / 边界的代码。

### Takeaway 7：Method Use Rule 让测试 = communication medium

- **是什么**：使用 legacy 方法前先看是否有测试；没有就写。**任何方法的使用都有测试背书** = 任何调用者都看得见"这个方法做什么、怎么用"。
- **为什么重要**：把测试从"开发工具"提升为"团队 communication medium"。
- **解决什么问题**：避免新人"读代码猜行为"。
- **适用场景**：团队 onboarding；legacy code 长期维护。

## 三、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **Characterization test 在 kernel = LTP / KUnit / kselftest 的本质**。LTP 的"功能正确性"测试就是 characterization test：跑一段代码，看输出是不是预期 —— 没有 formal spec，预期由"现状"驱动。
- **Syscall behavior 的 characterization**：Linux UAPI 一旦稳定，新 kernel release 必须保持 syscall 行为。LTP syscall testcase 就是 characterization test —— 记录当前行为，作为 release gate。
- **`/proc` / `sysfs` 输出的 characterization**：内核 ABI 包括 `/proc/net/dev` 输出格式。**ftrace + kselftest 经常用作 characterization**：跑一段操作，读 `/proc` 输出，作为后续版本的 snapshot。
- **`refcount_t` / `atomic_t` 的 characterization**：内核里的 refcount 边界（ref=0 → release）有专门的 KUnit testcase。**这些是典型的 targeted testing**：行为正确性 + 边界 trigger。
- **systemd unit test = characterization**：systemd 的 test harness 在改完一段配置后，跑整个 daemon 看 service 状态是否符合预期 —— 这是 character + targeted 的混合。

#### Linux 系统 — ch13 case × 系统侧映射表

| ch13 概念               | Linux / kernel 映射                                | 工具/手法                          |
| ----------------------- | -------------------------------------------------- | ---------------------------------- |
| Characterization test   | LTP / KUnit / kselftest                            | snapshot-based regression          |
| 5-step algorithm        | 跑 code → 看 dmesg → 写 expected → commit         | LTP `testcase.sh` 模板             |
| Sensing variable        | tracepoint / ftrace                                | `trace_printk` + parser             |
| Targeted testing        | bugfix 前的 LTP testcase                           | `add_testcase.sh`                  |
| Type conversion pitfall | `atomic_t` vs `int`                                | sparse + `Wconversion`             |
| Method use rule         | 每个新 syscall / kfunc 必须有 KUnit testcase       | `Documentation/process/` rule      |

### 3.2 机器人软件视角

- **ROS2 launch test = characterization**：ros2 launch test 跑一段 launch 文件，看节点启动是否如预期 —— 这是 ch13 的 ROS 侧版本。
- **Behavior tree snapshot = characterization test**：behavior tree 的执行序列在 production 中被记录；refactor 后必须保持同样的序列。这正是 ch13 Takeaway 1 的应用。
- **`/diagnostics` msg 的 characterization**：节点的 diagnostic msg 一旦稳定，新 release 必须保持同样 schema。ros2 doctor / `diagnostic_msgs` 的 CI = characterization。
- **robot hal test**：HAL interface 的 characterization = 跑同一组 motor commands，看 odometry / encoder 是否一致。这是 ch13 Takeaway 5（targeted testing）的机器人侧特化 —— 改 driver 前必须有 HAL snapshot。
- **motion planning 的 characterization**：plan path → execute → 反馈。改 planner 后必须保持"相同 start/goal → 相同 path"。**这是 targeted testing 的关键边界条件**。

### 3.3 初级 vs 高级工程师对比

| 维度                       | 初级工程师                                                       | 高级工程师                                                       |
| -------------------------- | ---------------------------------------------------------------- | ---------------------------------------------------------------- |
| 测试目的                   | "测这个方法对不对"                                                | "记录这个方法现在做什么"                                          |
| 断言写法                   | "我以为应该返回 X"                                               | 跑代码看真实返回值，写真实期望                                    |
| 心理障碍                   | "记录可能错的现状 = 默认 bug"                                    | "记录现状 = 后续 refactor 的 safety net"                         |
| 测试选址                   | "改哪个测哪个"                                                  | characterization 覆盖全类 + targeted testing 针对 change point    |
| Method use rule            | "测是 QA 的事"                                                  | "用之前先写测试，把测试当文档"                                   |
| 输入设计                   | 选 happy path                                                    | 选能 trigger 误用（类型错 / 边界）的输入                          |
| Bug 发现时                 | "skip 这个 test，bug 等下修"                                     | "标记 suspicious，escalate，立刻修或挂 ticket"                   |

> **关键差异**：高级把 characterization test 当 *主动捕获工具*；初级把它当 *可选 QA*。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

- 仍然极重要。原因：
  - **AI 写"对"的测试但不写"现状"的测试**：LLM 默认按 spec 写断言，而 legacy 没有 spec。**characterization test 是 AI 必须人工参与才能写的测试类型**。
  - **AI 不擅长"看失败信息反推行为"**：5 步算法的第 3 步（让失败告诉你行为）需要"读懂失败 + 改写期望"。LLM 能做但容易幻觉 —— 它可能"修正"成符合自己想象的行为。
  - **Targeted testing 的"选 trigger 输入"仍由人**：LLM 选 happy path，不擅长选边界 / 类型错敏感输入。
  - **Method Use Rule 在 AI 流水线里被绕过**：AI 用一个 legacy 方法时不会问"测试呢？" —— 它直接调。**这条 rule 必须在团队规约里 enforce**。

### 4.2 AI 已经能做的（具体到 ch13 主题）

- **生成测试骨架** —— 包括 happy path + 几条边界。
- **执行 5 步算法第 3 步** —— 跑失败测试，读输出，写期望。
- **扫描 codebase 找"没有测试就调用"的方法** —— 用 call graph + test file glob 匹配，列出"method use without test"。
- **检测类型转换 pitfall** —— grep `int <-> double` / `long <-> float` 边界，提示"输入应选能让截断可见的值"。

### 4.3 AI 不能替代的（具体到 ch13 主题）

- **判断"现状"是不是 bug** —— 这是 judgment call，由人定。
- **选能 trigger 误用的输入** —— 需要理解"如果实现错，会出什么事"，AI 不擅长这种对抗性思维。
- **Method Use Rule 的执行** —— AI 用方法不查测试，由人 enforce。
- **bug 修复的 escalation 决策** —— deployed vs undeployed，影响修复策略，AI 不懂业务。

### 4.4 AI 经常写错的地方

针对 ch13 characterization-test 主题，AI 的典型误判：

| 错误模式                                          | 例子                                                                                                                | 后果                                                                  |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| **把 characterization 当 unit test 写**           | AI 给 PageGenerator 写 `assertEquals("expected HTML", generator.generate())` —— 这是 spec test，不是 characterization      | 测试失败 = 行为错误（其实是 spec 错误），误报警                        |
| **幻觉"现状"**                                    | AI 没跑代码，凭想象写 `assertEquals("fred", ...)`，然后改 `assertEquals("fred", ...)` 假装 PASS                       | 测试通过但实际行为没被记录                                              |
| **漏掉边界 / 类型转换 trigger**                    | AI 给 FuelShare 的 `1.2 * price` 写测试选 gallons=1（不会触发 double→int 截断差异）                                  | Refactor 误用 int 时测试仍 PASS，bug 被掩盖                            |
| **不 escalate bug**                               | AI 写 characterization test 时发现 `corpBase` 是 12.0 但实际写 13.0，AI 默默接受                                    | Bug 永远不被 fix                                                       |
| **跳过 sensing variable**                         | AI 写 targeted test 时不 introduce sensing variable，导致测试通过但 change point 没被覆盖                            | 测试覆盖率 100%，实际 change point 未 trigger                          |
| **Method Use Rule 缺位**                          | AI 调用 legacy method 时不查 "tests for it"，直接调                                                                | 新代码引用了无测试 legacy method，债滚下去                             |
| **Refactoring 工具 quirk 未 verify**              | AI 自动 extract-method 后不 assert method signature 是否仍正确                                                      | Refactor 后调用方签名错，编译挂 / 行为变                              |

### 4.5 子段：AI 辅助遗留代码理解

适用本会话锁定视角（Linux 系统 + 机器人）。

- **Linux kernel characterization 的 AI 辅助**：
  - 用 AI 给一个 syscall 写 "现状快照" 测试 —— 接收当前 syscall 输出，写成 KUnit / LTP testcase。这是 ch13 5 步算法的自动化。
  - 用 AI 给一段 kernel 代码生成"sensing variable" —— 例如在 refcount 减前后 trace refcount 值。
  - **关键限制**：AI 给的 testing 输入倾向于 happy path，缺少类型转换 / race 的 trigger —— 必须由人 review。
- **ROS/ROS2 节点的 AI 辅助**：
  - 用 AI 给一个 ROS2 node 生成 characterization test —— 跑 launch 文件，读 `/diagnostics`，写 snapshot。这是 behavior preservation 的自动化。
  - 用 AI 给 motion planner 生成 "相同 input → 相同 path" 的 test。
  - **关键限制**：behavior tree 的隐含时序契约（"先 plan 后 execute"），AI 给的 snapshot 测试可能漏掉时序边界。

### 4.6 工程师必须保留的核心能力

- **跑测试读失败 → 改期望的 5 步算法执行**（必须人工 —— AI 容易幻觉现状）。
- **判断"现状"是不是 bug**（必须人工 —— AI 接受现状不 escalate）。
- **选 trigger 误用的测试输入**（必须人工 —— AI 倾向 happy path）。
- **Method Use Rule 的 enforce**（必须人工 —— AI 不查测试直接调）。

## 五、实践行动项

> ch13 是技法章，下面 4 项把 characterization test 落地为可编译运行的小程序。

### A1 — 5 步算法 CLI: 跑代码 → 写必然失败断言 → 读失败 → 改期望

```bash
mkdir -p /tmp/ch13-5step && cd /tmp/ch13-5step

# 5 步算法的小工具: 给一段 C 函数, 跑一次, 拿到输出, 让用户确认这就是现状.
cat > characterization_5step.sh <<'BASH'
#!/usr/bin/env bash
# characterization_5step.sh — ch13 5 步算法的最小 CLI 演示
# 跑一个二进制, 接收 stdout, 输出到 "captured.txt",
# 然后用 captured.txt 替换 assert 中的 expected 值.
set -euo pipefail

step="${1:-help}"
binary="${2:-}"

case "$step" in
    1)  # step 1: 跑代码 (无断言)
        if [[ -z "$binary" ]]; then echo "usage: $0 1 <binary>"; exit 2; fi
        echo "[step 1] running $binary..."
        out=$("$binary")
        echo "[step 1] captured output:"
        echo "$out"
        echo "$out" > captured.txt
        echo "[step 1] saved to captured.txt"
        ;;
    2)  # step 2: 写必然失败断言
        # 自动生成一个 assertEqual(EXPECTED, captured) 但 EXPECTED 是错的
        if [[ ! -f captured.txt ]]; then echo "run step 1 first"; exit 2; fi
        actual=$(cat captured.txt)
        cat > test_char.c <<EOF
#include <assert.h>
#include <stdio.h>
#include <string.h>
int main(void) {
    /* step 2: 必然失败的断言 — "WRONG_EXPECTED" */
    const char *expected = "WRONG_EXPECTED";
    const char *actual   = "$actual";
    printf("step 2: assertEquals(\"%s\", actual)\\n", expected);
    assert(strcmp(expected, actual) == 0);
    return 0;
}
EOF
        cc -std=c17 -Wall -Wextra -o test_char test_char.c
        echo "[step 2] test_char compiled, EXPECTED=WRONG_EXPECTED"
        ;;
    3)  # step 3: 跑测试, 让失败告诉你行为
        if [[ ! -x ./test_char ]]; then echo "run step 2 first"; exit 2; fi
        echo "[step 3] running test_char (expecting FAILURE)..."
        ./test_char 2>&1 || true
        echo
        echo "[step 3] ↑ failure message tells us actual behavior"
        ;;
    4)  # step 4: 改 expected 为真实行为
        if [[ ! -f captured.txt ]]; then echo "run step 1 first"; exit 2; fi
        actual=$(cat captured.txt)
        cat > test_char.c <<EOF
#include <assert.h>
#include <stdio.h>
#include <string.h>
int main(void) {
    /* step 4: 改成真实行为 — characterization test PASS. */
    const char *expected = "$actual";
    const char *actual   = "$actual";
    assert(strcmp(expected, actual) == 0);
    return 0;
}
EOF
        cc -std=c17 -Wall -Wextra -o test_char test_char.c
        echo "[step 4] test_char re-compiled with EXPECTED=actual"
        ;;
    5)  # step 5: 跑测试, 验证 PASS
        if [[ ! -x ./test_char ]]; then echo "run step 4 first"; exit 2; fi
        echo "[step 5] running test_char (expecting PASS)..."
        ./test_char && echo "[step 5] PASS — characterization recorded"
        ;;
    *)
        echo "usage: $0 {1|2|3|4|5} [binary]"
        echo "  1 = run binary, capture output"
        echo "  2 = write failing assertion"
        echo "  3 = run failing test"
        echo "  4 = update assertion to actual"
        echo "  5 = re-run, expect PASS"
        exit 2
        ;;
esac
BASH
chmod +x characterization_5step.sh

# 模拟 PageGenerator: 一个简单的输出函数.
cat > page_generator.c <<'EOF'
#include <stdio.h>
/* 模拟 ch13 p.188 PageGenerator:
 *  - 空 generate() 返回 ""
 *  - 不暴露行为 spec, 必须 characterization 才能知道
 */
const char *page_generator_generate(void) {
    /* 当前行为: 返回 "" — 但 spec 没写, 真实行为靠 5 步算法探出. */
    return "";
}
int main(void) {
    printf("%s\n", page_generator_generate());
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o page_generator page_generator.c

# 跑 5 步
echo "=== STEP 1: capture ==="
./characterization_5step.sh 1 ./page_generator
echo
echo "=== STEP 2: write failing assertion ==="
./characterization_5step.sh 2
echo
echo "=== STEP 3: run, expect FAILURE ==="
./characterization_5step.sh 3
echo
echo "=== STEP 4: update assertion ==="
./characterization_5step.sh 4
echo
echo "=== STEP 5: re-run, expect PASS ==="
./characterization_5step.sh 5
echo "rc=$?"
```

**验收**：
- Step 3 输出 assertion failure 信息（含 "WRONG_EXPECTED"）。
- Step 5 输出 `PASS — characterization recorded`，`rc=0`。

### A2 — Type Conversion Pitfall: 选 trigger 误用的输入

```bash
mkdir -p /tmp/ch13-type-conv && cd /tmp/ch13-type-conv

# 复刻 ch13 p.215-216 FuelShare 例子: 1.2 * price 误用 int 会截断,
# 选 gallons=1 让差异立即可见.
cat > fuel_share.h <<'EOF'
#ifndef FUEL_SHARE_H
#define FUEL_SHARE_H

typedef struct {
    long cost;        /* cents */
    double corp_base; /* CORP base surcharge */
} FuelShare;

void fuel_share_init(FuelShare *fs);
void fuel_share_add_reading(FuelShare *fs, int gallons, double price_per_gallon);
long fuel_share_get_cost(const FuelShare *fs);

#endif
EOF

cat > fuel_share.c <<'EOF'
#include "fuel_share.h"

void fuel_share_init(FuelShare *fs) {
    fs->cost = 0;
    fs->corp_base = 12.0;
}

void fuel_share_add_reading(FuelShare *fs, int gallons, double price_per_gallon) {
    /* ch13 p.215: 1.2 * priceForGallons. 如果实现错用 int, 截断发生.
     * 选 gallons = 100, price = 0.5 → 1.2 * 50 = 60.0. 截断仍 60.
     * 选 gallons = 100, price = 0.99 → 1.2 * 99 = 118.8. 截断 118 vs 118.8 差 0.8.
     * 选 gallons = 100, price = 0.999 → 1.2 * 99.9 = 119.88. 截断 119 vs 119.88 差 0.88.
     * 我们用一个明确的 trigger: 让 cost 累计 3 次后是整数 + 分数. */
    double total_price = (double)gallons * price_per_gallon;
    fs->cost += (long)(1.2 * total_price * 100);  /* cents */
}

long fuel_share_get_cost(const FuelShare *fs) { return fs->cost; }
EOF

cat > test_fuel_share.c <<'EOF'
/* ch13 Takeaway 6: 选 trigger 误用 (int vs double) 的输入.
 * 这里用 price_per_gallon = 0.83:
 *   - 正确: 1.2 * 100 * 0.83 = 99.6 → cents = 9960
 *   - 误用 int: 1.2 * (int)(100*0.83) = 1.2 * 83 = 99.6, 但如果 price=0.99:
 *       - 正确: 1.2 * 100 * 0.99 = 118.8 → 11880 cents
 *       - 误用: 1.2 * (int)(100*0.99) = 1.2 * 99 = 118.8, OK
 * 真正 trigger 选 price_per_gallon = 0.7:
 *   - 正确: 1.2 * 100 * 0.7 = 84.0 → 8400 cents
 *   - 误用 int trunc: 1.2 * 70 = 84.0 → OK, 仍同.
 * 更敏感: price=0.3, gallons=1 → 1.2 * 1 * 0.3 = 0.36 cents → 36.
 *   int trunc 在 (int)(0.36) = 0 时丢精度.
 */
#include "fuel_share.h"
#include <assert.h>

int main(void) {
    /* case 1: 整数结果, 不敏感 */
    FuelShare fs1;
    fuel_share_init(&fs1);
    fuel_share_add_reading(&fs1, 100, 1.0);   /* 1.2 * 100 = 120 cents = 12000 */
    assert(fuel_share_get_cost(&fs1) == 12000);

    /* case 2: 小数结果, 触发类型转换敏感 */
    FuelShare fs2;
    fuel_share_init(&fs2);
    fuel_share_add_reading(&fs2, 1, 0.83);    /* 1.2 * 1 * 0.83 * 100 = 99.6 → (long) = 99 cents */
    /* 这里 trigger 误用: 如果实现错误地把 price 截断成 int (int)(0.83) = 0,
     * cost += 0; vs 正确 cost += 99. 差异明显. */
    long cost = fuel_share_get_cost(&fs2);
    printf("case 2 cost = %ld cents (should be 99)\n", cost);
    assert(cost == 99);   /* characterization: 真实行为记录 */

    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_fuel_share fuel_share.c test_fuel_share.c
./test_fuel_share
echo "rc=$?"

# 也展示如果误用 int 截断的版本: 改 fuel_share.c 把 price 截断, 看测试是否 fail.
cat > fuel_share_buggy.c <<'EOF'
#include "fuel_share.h"
void fuel_share_init(FuelShare *fs) { fs->cost = 0; fs->corp_base = 12.0; }
/* BUGGY: 把 price_per_gallon 直接截断成 int (0.83 → 0). */
void fuel_share_add_reading(FuelShare *fs, int gallons, double price_per_gallon) {
    int trunc_price = (int)price_per_gallon;   /* 0.83 → 0 */
    fs->cost += (long)(1.2 * trunc_price * 100);
}
long fuel_share_get_cost(const FuelShare *fs) { return fs->cost; }
EOF

echo "=== sanity: with buggy version, case 2 should FAIL ==="
cc -std=c17 -Wall -Wextra -o test_fuel_share_buggy fuel_share_buggy.c test_fuel_share.c
./test_fuel_share_buggy ; echo "buggy rc=$?"  # 非 0 (case 2 fail)
```

**验收**：
- `test_fuel_share` `rc=0`（characterization recorded）。
- `test_fuel_share_buggy` `rc != 0`（price 截断 bug 被触发）。
- 注释说明：选 `price_per_gallon = 0.83` 是 Takeaway 6 的"输入能 trigger 误用"。

### A3 — Sensing Variable: 在 refcount / counter 改动时 trace 真实行为

```bash
mkdir -p /tmp/ch13-sensing && cd /tmp/ch13-sensing

# 复刻 ch13 Takeaway 4 (a): tangled 逻辑用 sensing variable 看.
# 这里模拟一个 refcount 边界条件.
cat > refcount.h <<'EOF'
#ifndef REFCOUNT_H
#define REFCOUNT_H
typedef struct { int refcount; int alloc_id; } Object;

Object *object_new(int alloc_id);
void    object_acquire(Object *o);   /* refcount++ */
void    object_release(Object *o);   /* refcount-- */
int     object_is_alive(const Object *o);   /* refcount > 0 */
int     object_get_refcount(const Object *o);
int     object_get_alloc_id(const Object *o);

#endif
EOF

cat > refcount.c <<'EOF'
#include "refcount.h"
#include <stdlib.h>
#include <stdio.h>

static int g_n_alive = 0;   /* sensing variable: 看 live 对象数 */

Object *object_new(int alloc_id) {
    Object *o = calloc(1, sizeof(*o));
    if (o) { o->refcount = 1; o->alloc_id = alloc_id; g_n_alive++; }
    return o;
}
void object_acquire(Object *o) {
    if (o) o->refcount++;
}
void object_release(Object *o) {
    if (!o) return;
    if (--o->refcount == 0) { free(o); g_n_alive--; }
}
int object_is_alive(const Object *o) { return o && o->refcount > 0; }
int object_get_refcount(const Object *o) { return o ? o->refcount : -1; }
int object_get_alloc_id(const Object *o)  { return o ? o->alloc_id : -1; }

/* Test-only: 看全局 live 数. */
int refcount_test_n_alive(void) { return g_n_alive; }
EOF

cat > test_refcount.c <<'EOF'
/* ch13 Takeaway 4 (a): 用 sensing variable 看 "refcount=0 → release" 行为.
 * g_n_alive 全局计数器作为 sensing, 让 test 能 verify:
 *   - alloc 后 alive = N
 *   - release refcount=0 后 alive = N-1
 *   - acquire 不释放
 */
#include "refcount.h"
#include <assert.h>
#include <stddef.h>

extern int refcount_test_n_alive(void);

int main(void) {
    int n0 = refcount_test_n_alive();
    Object *o = object_new(42);
    assert(o != NULL);
    assert(object_get_refcount(o) == 1);
    assert(object_get_alloc_id(o) == 42);
    assert(refcount_test_n_alive() == n0 + 1);    /* sensing: alloc 后 alive 增 */

    /* acquire: refcount=2, alive 不变 */
    object_acquire(o);
    assert(object_get_refcount(o) == 2);
    assert(refcount_test_n_alive() == n0 + 1);

    /* release 一次: refcount=1, alive 不变 */
    object_release(o);
    assert(object_get_refcount(o) == 1);
    assert(object_is_alive(o) == 1);
    assert(refcount_test_n_alive() == n0 + 1);

    /* release 二次: refcount=0, alive 减 (sensing 触发) */
    object_release(o);
    assert(refcount_test_n_alive() == n0);   /* sensing: 减回去了 */
    /* 此时 o 是 dangling, 不再访问. */

    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_refcount refcount.c test_refcount.c
./test_refcount
echo "rc=$?"
```

**验收**：
- `rc=0`，4 个 sensing assertions 全部通过 —— `g_n_alive` 作为全局 sensing variable 让"refcount=0 → release"这条隐藏逻辑可观察。
- 注释说明：这是 ch13 Takeaway 4 (a) 的"tangled 逻辑用 sensing variable"在 refcount 上的工业实例（kernel `refcount_t` 的 KUnit testcase 同理）。

### A4 — Method Use Rule: 测试覆盖度 = "调用每个方法前查测试"

```bash
mkdir -p /tmp/ch13-mur && cd /tmp/ch13-mur

# 模拟 ch13 Method Use Rule: 在调用一个 legacy 方法前先检查测试.
# 用一个 mini grep 工具 + 几个示例函数来演示.
cat > legacy_module.h <<'EOF'
#ifndef LEGACY_MODULE_H
#define LEGACY_MODULE_H
int legacy_compute(int x);
int legacy_validate(int x);
void legacy_log_state(int x);
#endif
EOF

cat > legacy_module.c <<'EOF'
#include "legacy_module.h"
int legacy_compute(int x) { return x * 2; }
int legacy_validate(int x) { return x >= 0; }
void legacy_log_state(int x) { (void)x; }
EOF

cat > test_legacy_compute.c <<'EOF'
#include "legacy_module.h"
#include <assert.h>
int main(void) { assert(legacy_compute(5) == 10); return 0; }
EOF
cat > test_legacy_validate.c <<'EOF'
#include "legacy_module.h"
#include <assert.h>
int main(void) { assert(legacy_validate(1) == 1); assert(legacy_validate(-1) == 0); return 0; }
EOF

cat > method_use_rule.py <<'PY'
#!/usr/bin/env python3
"""method_use_rule.py — ch13 Method Use Rule 的最小 check.
扫描:
  1. legacy_module.h 中每个 extern 函数.
  2. 对每个函数, 在 tests/ 下找 test_<funcname>.c 或 test_<funcname>.py.
  3. 列出 'used without test' 的函数.

这是 ch13 Takeaway 7 的 enforce 工具雏形.
"""
import os, re, sys

def main():
    header = sys.argv[1] if len(sys.argv) > 1 else "legacy_module.h"
    testdir = sys.argv[2] if len(sys.argv) > 2 else "tests"
    os.makedirs(testdir, exist_ok=True)
    funcs = []
    with open(header) as f:
        for line in f:
            pat = (r"\b(?:int|void|char|long|float|double|size_t|"
                   r"Object\s*\*|FuelShare|InMemoryDirectory)"
                   r"\s+(\w+)\s*\(")
            m = re.search(pat, line)
            if m: funcs.append(m.group(1))
    print(f"# scanned {header}, found {len(funcs)} functions: {funcs}")
    print(f"# test directory: {testdir}/")
    missing = []
    for fn in funcs:
        cands = [f"{testdir}/test_{fn}.c", f"{testdir}/test_{fn}.py"]
        if not any(os.path.exists(p) for p in cands):
            missing.append(fn)
            print(f"  WARN: {fn} used without test (no {cands[0]})")
        else:
            print(f"  OK:   {fn} has test")
    print()
    if missing:
        print(f"# ACTION: write characterization tests for {len(missing)} functions:")
        for fn in missing:
            print(f"  - {testdir}/test_{fn}.c")
        return 1
    print("# Method Use Rule: PASS — all functions have tests")
    return 0

if __name__ == "__main__":
    sys.exit(main())
PY
chmod +x method_use_rule.py

mkdir -p tests
cp test_legacy_compute.c tests/
cp test_legacy_validate.c tests/

./method_use_rule.py legacy_module.h tests
echo "rc=$?"

echo
echo "=== now remove test_legacy_validate.c, expect WARN ==="
mv tests/test_legacy_validate.c /tmp/
./method_use_rule.py legacy_module.h tests
echo "rc=$?"
mv /tmp/test_legacy_validate.c tests/

echo
echo "=== write missing test, expect PASS ==="
cat > tests/test_legacy_log_state.c <<'EOF'
#include "legacy_module.h"
int main(void) { legacy_log_state(0); return 0; }
EOF
./method_use_rule.py legacy_module.h tests
echo "rc=$?"
```

**验收**：
- 第一次跑：3 个函数中 `legacy_log_state` 缺测试，输出 WARN，`rc=1`。
- 第三次跑（补上 test）：全部 OK，`rc=0`。
- 注释说明：这是 Method Use Rule 的自动化 enforce 雏形 —— 在 CI / pre-commit hook 里跑 `method_use_rule.py` 确保无测试的方法不被使用。

## 六、值得深入思考的问题

### Q1: Characterization test 记录的"现状"是不是 bug 时怎么办？

Takeaway 1 + "When You Find Bugs" 给了两条路：deployed 慎改，undeployed 直接修。**关键问题**：如何 escalate？谁决定"deployed 系统的 bug 是不是有人依赖"？AI 找到的 bug 是否走同一条 escalate 流程？

### Q2: 5 步算法的"现状"被 AI 幻觉替代怎么办？

AI 跑 5 步算法的 step 3 时可能"修正"成符合自己想象的输出。**关键问题**：怎么 verify 5 步算法的 step 3 输出是真的"现状"而不是 AI 的 hallucination？是不是必须有独立 ground truth（如日志 / 调试器）？

### Q3: Targeted testing 的输入设计谁负责？

Takeaway 6 给了选 trigger 误用的输入的指导。**关键问题**：团队里"输入设计"是 developer 责任还是 reviewer 责任？AI 给的 happy path 输入如何 reject？

### Q4: Method Use Rule 的执行成本

Takeaway 7 让"用之前先写测试"成为硬规则。**关键问题**：探索性 prototyping 期间，强制写测试会拖慢迭代。什么时候 enforce 这条 rule？prototyping → production 的转换点怎么划？

### Q5: AI 主导的 characterization test 是否需要"人 review"

Takeaway 7 + "AI 经常写错"段都暗示 characterization test 不能纯 AI 写。**关键问题**：AI 写完 characterization test 后，人 review 该看什么？什么信号表明 AI 把"现状"误读成 spec 修正？

---

*下一章预告*: **Chapter 14 — Dependencies on Libraries Are Killing Me** —— 极短一章（2 页），但引入 ch11/ch12/ch13 之后的下一个症状：当你对**第三方库**的依赖让你改不了 / 测不了 / 编译不过时怎么办。ch14 是 ch25 catalog 中 "Library Chapter" 的入门 —— `Skin and Wrap the API`、`Adapt Parameter` 等 refactoring 在跨库边界上的应用。

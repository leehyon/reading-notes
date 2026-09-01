# Chapter 12 — I Need to Make Many Changes in One Area

> **PDF**: p.195-206（12 页）
> **定位**: ch11 教"在哪写测试（方法级）"；ch12 把选址放大到"区域级"：当改动横跨 3-5 个紧密相关的类时，**interception point** 与 **pinch point** 是两个核心武器。ch12 短小但概念密度高 —— 它决定"测试覆盖的 ROI 何时最高"。

## 〇、第一性原理思考

**这章做了什么**：在 ch11 effect sketch 之上，引入 interception point 与 pinch point 两个武器，专门解决 3-5 个紧密相关类同时改动的测试选址。

**为什么这样拆**：方法级测试选址（ch11）只解决"测这条链上的谁"；区域级改动需要找测试 ROI 最高的点，作者把 pinch point 定义为 effect sketch 的自然收窄，让一组改动一次性被 sense。

**最值钱的洞见**：Pinch point 不是结构决定的 —— 同一段代码在这次改动下是 pinch point，下次改动下可能就不是；pinch point 只是 change-driven 的测试形态，不是真正的封装边界，所以需要后续拆分用更窄的 unit test 替代。
## 一、章节概述

- **Interception Point**：能 sense 到某次改动效果的程序位置。它可以是 change point 邻接的 public method，也可以跨多个对象。**每个 effect sketch 的端点都是候选 interception point**。
- **The Simple Case**（Invoice. getValue）：单 change point，effect sketch 单链。Interception point 可以是 getValue（紧邻 change point），也可以是 BillingStatement. makeStatement（更上层的 call site）。
- **Higher-Level Interception Points**：选哪一层 = 风险 / 成本权衡。**靠近 change point = 测试 setup 简单，但 effect 链短；远 = setup 复杂，但一组 change 一起覆盖**。
- **Pinch Point**：effect sketch 的自然收窄 —— 一个点能 sense 多个对象的 effect。**图 12.5 的 BillingStatement. makeStatement 就是 pinch point**。Pinch point 是测试 ROI 最高的地点。
- **Pinch Point 的两个关键性质**：(a) 由 change points 决定（不是结构决定 —— 同样结构在不同 change 下 pinch point 可能不同）；(b) 多个 pinch point 联合起来仍算一个 pinch point（如图 12.7 的 run + makeStatement）。
- **找不到 pinch point 怎么办？** 重新审视 change points —— 是不是做得太多？分两批做，先做能 find pinch point 的那批。
- **Judging Design with Pinch Points**：**pinch point 是天然 encapsulation 边界**。找到 pinch point = 找到"该把哪些字段 / 方法聚成新类"的位置。
- **Using Effect Sketches to Find Hidden Classes**：大类的拆分用 effect sketch 找隐藏边界 —— 哪些字段 / 方法只在彼此之间共享 effect，外部根本看不到。
- **Pinch Point Traps**：高扇入的 pinch point 测试容易退化成 mini-integration test。**Pinch point 是 *临时* 测试形态**，测试就位后要做拆分、用更窄的 unit test 替代。

## 二、核心 Takeaways

### Takeaway 1：每个 effect sketch 端点都是 interception point 候选

- **是什么**：Interception point = 能 sense 改动的位置。Effect sketch 的每个端点都是候选。挑哪个 = 判断"setup 成本 / effect 覆盖完整性 / 距 change point 距离"三因素的权衡。
- **为什么重要**：把"在哪里写测试"从"找 change point 邻接"扩展到"找 effect 链任意可达端点"。这给大规模改动提供了测试点选址的搜索空间。
- **解决什么问题**：避免"只能在紧邻 change point 测，结果 setup 成本高但覆盖少"。
- **适用场景**：跨多类的成片改动；大类的内部 helper 改动；framework 集成层改动。

### Takeaway 2：靠近 change point 的 interception point 更安全但覆盖少

- **是什么**：Interception point 距 change point 越近，setup 越简单（不需要构造完整 call chain），但 effect 链短（漏掉中间链上的其它 effect）。
- **为什么重要**：测试 setup 复杂度与 effect 覆盖度反相关 —— 没有"既简单又全"的 interception point，必须权衡。
- **解决什么问题**：让团队在"先求 coverage 还是先求 setup 简洁"间做有意识的取舍。
- **适用场景**：change point 本身可测（首选）；change point 不可测（被迫上移）。

### Takeaway 3：Pinch point = effect sketch 的自然收窄

- **是什么**：多个 change points 的 effect 在某一点汇聚 —— 这就是 pinch point。测这一个点等于测一组改动。
- **为什么重要**：把"每改一处都要写一个测试"变成"一组改动共用一个测试点"。这是测试 ROI 最高的几何形态。
- **解决什么问题**：避免"5 个 change point × 5 个测试"的重复劳动；用 1 个 pinch point 覆盖 5 个 change points。
- **适用场景**：成片改动（ch12 主场景）；framework 集成层；跨子系统的 facade。

### Takeaway 4：Pinch point 由 change points 决定，不由结构决定

- **是什么**：同一个类在不同 change 下可能不是 pinch point。Change points 改变，pinch point 跟着变。**图 12.6 vs 12.7 同结构但 pinch point 不同**。
- **为什么重要**：避免"看到 BillingStatement 就把它当 pinch point" —— 如果 change 涉及 InventoryControl，BillingStatement 不再覆盖全部 effect。
- **解决什么问题**：防止"结构看起来好就以为测试也到位"的错觉。
- **适用场景**：评估现有测试是否覆盖即将到来的改动。

### Takeaway 5：Pinch point 是天然 encapsulation 边界

- **是什么**：effect 在 pinch point 收窄意味着 —— 外部 caller 通过 pinch point 与内部状态解耦。这是封装好的代码的几何特征。**反过来：用 effect sketch 找 pinch point = 找天然的拆分边界**。
- **为什么重要**：让 refactor（ch20 拆分大类）有客观依据 —— 不靠命名或经验，靠 effect propagation 图。
- **解决什么问题**：避免"凭感觉拆类"，给拆分提供 effect-graph 证据。
- **适用场景**：拆分 monolithic 类；微服务 / 模块边界设计；ROS 节点的子职责提取。

### Takeaway 6：找不到 pinch point 时，缩小 change scope

- **是什么**：如果改动范围太大以至于 sketch 散成树状而无收窄，**把改动拆成两批**，先做能 find pinch point 的那批。这是 ch12 给的退路。
- **为什么重要**：把"测试 setup 困难"作为"change scope 太大"的信号 —— 比硬上复杂测试更工程友好。
- **解决什么问题**：避免"5 天搭测试 / 改 1 行代码"的窘境。
- **适用场景**：大 feature 的拆分；agile 迭代中的"垂直切片"。

### Takeaway 7：Pinch point 测试是过渡态，不是终态

- **是什么**：pinch point 测试本质是 mini-integration test。**有 pinch point 测试 = refactor 自由，但 unit test 不充分**。Feathers 明确说："eventually, the tests at the pinch point can go away"，换成更窄的 unit test。
- **为什么重要**：避免"pinch point 测试留下来"变成长期 debt —— 它们会越来越慢、越来越难调试。
- **解决什么问题**：让 pinch point 测试有明确的退役计划，而不是无限膨胀。
- **适用场景**：建立 pinch point 测试后做 ch20 大类拆分。

## 三、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **`/proc` / `sysfs` 是 kernel 侧的天然 pinch point**。一个内核子系统的所有 effect（计数 / 状态 / 配置）经常汇总到 `/proc/net/*`、`/sys/class/*`。**LTP testcase 经常以 `/proc/net/dev` 等为 interception point** —— 这是 pinch point 的内核特化。
- **tracepoint 是 effect propagation 的"pinch point 探针"**：`trace_printk` / ftrace event 把多个 kernel 子系统的 effect 收窄到一个 trace buffer。**当一处改动要覆盖多个子系统时，写一个 tracepoint-based test 比写多个 unit test 更省力** —— 这就是 pinch point 的运行时版本。
- **`kobject` / `kset` 层级 = effect sketch 的几何对应**：内核对象模型中 parent-child 关系经常体现 effect 的"汇流"。**Sysfs 顶层目录 = pinch point**；子系统内部细节 = 多 effect source。
- **systemd unit file 的 `[Install]` section 是 pinch point 配置点**：daemon 之间的 dependency 通过 `[Install] WantedBy=` 声明。**测试一个 systemd unit 完整启动链路**经常以 `systemctl status xxx` 的输出为 interception point。
- **LTP / kselftest 的"高扇出测试"陷阱**：LTP 的某些 testcase 同时触碰多个 kernel 子系统，看似覆盖广，实则是 pinch point trap —— 跑得慢、失败时难定位。ch12 Takeaway 7 直接命中。

#### Linux 系统 — ch12 case × 系统侧映射表

| ch12 概念               | Linux / kernel 映射                                | 工具/手法                          |
| ----------------------- | -------------------------------------------------- | ---------------------------------- |
| Interception point      | `/proc/*` / `/sys/*` / tracepoint                  | LTP + kselftest                    |
| Pinch point             | `/proc/net/dev` / `tcp_rr` 结果                    | 高扇入 syscall test                |
| Pinch point trap        | LTP multi-subsystem testcase                       | 拆分到 KUnit + subsystem-specific |
| Hidden class detection  | `kobject` 子树                                     | coccinelle + `kobj_type` 分析      |
| Encapsulation boundary  | `struct file_operations` vs `inode_operations`      | VFS split                          |

### 3.2 机器人软件视角

- **`/diagnostics` topic 是 ROS2 节点的 pinch point**：节点状态、错误码、heartbeat 汇总到 `/diagnostics`。**测一个节点的完整状态变化 = 订阅 `/diagnostics` 一个 topic**。
- **`tf2` 的 `/tf_static` 是多 sensor 的 pinch point**：所有 sensor 的 calibration 结果汇总到 tf tree。**改一个 sensor 的 extrinsics → 测试一个 tf tree**。
- **`ros2_control` 的 controller_manager 是 hardware + control 的 pinch point**：controller 与 hardware interface 的所有交互汇总到 `controller_manager`。**ch12 Takeaway 3 的机器人侧特化**。
- **`behavior tree` 的 root 是 pinch point，但也是 trap**：一个 BT root 的 tick 触发所有子节点。**测 root = 测整棵树**；但调试时难定位哪个子树失败。
- **micro-ROS / embedded ROS 的 HAL = pinch point**：硬件抽象层把所有 sensor / actuator 收窄到一组 interface。**改 driver = 测 HAL**。

### 3.3 初级 vs 高级工程师对比

| 维度               | 初级工程师                                                       | 高级工程师                                                  |
| ------------------ | ---------------------------------------------------------------- | ----------------------------------------------------------- |
| 测试选址           | "在改的方法上测"                                                 | 画 effect sketch，找 pinch point                            |
| Pinch point 评估   | "结构上看起来像 pinch point 就用"                                | "change points 决定 pinch point，结构只是参考"              |
| 测试 setup         | 复制整套 call chain                                              | 优先选靠近 change point 的 interception；远端只是备选       |
| 找不到 pinch point | "加更多 mock 把链路连起来"                                       | 重新审视 change scope，拆批做                              |
| 拆分大类依据       | "这个类太大了"                                                  | "effect sketch 显示这些字段 / 方法天然成簇"                |
| Pinch point 测试   | 留下来当 integration test 用                                     | 明确退役计划：拆类后用窄 unit test 替代                   |

> **关键差异**：高级把 pinch point 当作 *临时* 投资；初级当作 *永久* 测试形态。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

- 仍然极重要。原因：
  - **AI 写代码更倾向单点改动**，但现代系统（特别是 microservices / ROS2）的改动常常跨多个 component。pinch point 在 AI 流水线里被低估。
  - **AI 不擅长画 effect sketch 的"收窄"几何**：LLM 给的 effect 描述是文字段落，看不出 pinch point。**几何判断仍由人**。
  - **Pinch point trap 在 AI 生成测试里被放大**：AI 写的 "覆盖完整调用链" 测试本质是 mini-integration test，跑得慢、调试难。**Feathers 警告"eventually go away"在 AI 时代更必须执行**。
  - **找到 pinch point = 找到拆分边界**：AI 给的 class 拆分建议通常基于命名相似度，不如 effect sketch 准确。

### 4.2 AI 已经能做的（具体到 ch12 主题）

- **生成 call-graph 并高亮 pinch point 候选** —— 基于"fan-in 高 / fan-out 低"启发式。
- **基于 change set 推荐 interception point** —— 列出"哪些公开方法能 sense 到这次改动"。
- **检测 pinch point trap** —— 标记"测试 setup 涉及 ≥3 个非 mock 对象"的测试，建议拆分。
- **基于 effect sketch 给出大类拆分建议** —— 列出"哪些字段 / 方法可以聚成新类"。

### 4.3 AI 不能替代的（具体到 ch12 主题）

- **判断 pinch point 是否 *真* 收窄** —— AI 给的高 fan-in 节点可能只是巧合（多个 caller 调用但 effect 不重叠）。
- **change scope 是否过大的判断** —— AI 不理解产品节奏。
- **pinch point 测试的退役时机** —— AI 不跟踪"测试到位后是否还负担过重"。
- **跨子系统 encapsulation contract** —— 哪些 effect 通道是核心契约 vs 偶然耦合，AI 模型不知道历史。

### 4.4 AI 经常写错的地方

针对 ch12 pinch-point 主题，AI 的典型误判：

| 错误模式                                  | 例子                                                                                                                | 后果                                                                  |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| **把 fan-in 当 pinch point**              | AI 推荐"测 BillingStatement.makeStatement 因为它有 5 个 caller" —— 但这些 caller 的 effect 不重叠                       | Pinch point 是错觉，写完测试漏 effect                                |
| **忽略 change-specific pinch point**      | AI 推荐"用 makeStatement 测试 supplier 改动" —— 但 supplier 改动 effect 还到 InventoryControl.run                  | Pinch point 选错，覆盖不全                                            |
| **建议用 mini-integration test 而非 unit test** | AI 写一个测试同时 mock 3 个对象，因为它们"都被改动影响"                                                              | Pinch point trap，测试跑得慢、调试难                                  |
| **拆分大类基于命名相似度而非 effect**     | AI 推荐"把 `display*` 字段 / 方法拆成 DisplayHelper"                                                                | 拆分后 effect 通道仍交叉，封装改善有限                                |
| **漏算 superclass / subclass effect**     | AI 给的 pinch point 在 change point 紧邻，漏了 subclass 的 effect                                                    | Test coverage 显示 100%，但 subclass 行为悄悄变                        |
| **忽视 effect 通道的 transient vs persistent** | AI 把 transient 状态（缓存）的 effect 当成 persistent effect，pinch point 选错                                       | 测试 flaky（cache hit / miss 不同）                                  |
| **推荐过早退役 pinch point 测试**         | AI 在 refactor 还没稳定时就把 pinch point 测试删了                                                                  | Refactor 失败时无安全网                                               |

### 4.5 子段：AI 辅助遗留代码理解

适用本会话锁定视角（Linux 系统 + 机器人）。

- **Linux kernel pinch point 的 AI 辅助**：
  - 用 AI 给一段 kernel patch 列出"哪些 `/proc`/`/sys` entry 是 pinch point 候选" —— 这是 LTP testcase 选址的客观化。
  - 用 AI 给一个 kobject 树画 effect sketch —— 揭示"哪些 attribute 是真正的 pinch point"。
  - **关键限制**：kernel 中的 ftrace / kprobe 路径涉及动态 effect，AI 给的 static sketch 漏掉运行时 effect。
- **ROS/ROS2 节点的 AI 辅助**：
  - 用 AI 给一个 launch 文件推荐 pinch point topic —— 这是 ROS2 test 选址的客观化。
  - 用 AI 给 `/diagnostics` 的 msg 画 effect sketch —— 揭示哪些字段是 pinch point 候选。
  - **关键限制**：behavior tree 的隐含时序优先级，AI 给不出"哪条边是核心契约"。

### 4.6 工程师必须保留的核心能力

- **画出 effect sketch 并识别 pinch point**（必须人工 —— AI 给文字描述，几何判断由人）。
- **判断 pinch point 是否真收窄**（必须人工 —— AI 给的 fan-in 可能是巧合）。
- **拆分 change scope**（必须人工 —— AI 不懂产品节奏）。
- **pinch point 测试退役的时机判断**（必须人工 —— AI 不跟踪 debt 进度）。

## 五、实践行动项

> ch12 是概念章，下面 4 项把 pinch point / interception point 落地为可编译运行的小程序。

### A1 — Pinch Point Detector: 从 call-graph 自动识别 pinch point 候选

```bash
mkdir -p /tmp/ch12-pinch && cd /tmp/ch12-pinch

# 接收一段 call-graph 描述, 用 fan-in/fan-out 启发式列出 pinch point 候选.
cat > pinch_detect.py <<'PY'
#!/usr/bin/env python3
"""pinch_detect.py — ch12 pinch point 启发式检测器.
格式: caller -> callee  每行一对.
输出: 按 fan-in 排序的 pinch point 候选 + 警告 (traps).
"""
import sys, re
from collections import defaultdict

def main():
    # 如果 argv < 2 且 stdin 是 TTY (没 pipe 输入), 才报错
    if len(sys.argv) < 2 and sys.stdin.isatty():
        print("usage: pinch_detect.py < graph.txt   或   ./pinch_detect.py < graph.txt")
        print("  graph.txt 格式: 'caller -> callee' 每行一对")
        sys.exit(2)
    text = " ".join(sys.argv[1:]) if len(sys.argv) > 1 else sys.stdin.read()
    fan_in  = defaultdict(set)   # callee -> {callers}
    fan_out = defaultdict(set)   # caller -> {callees}
    for line in text.splitlines():
        m = re.match(r"\s*(\w+)\s*->\s*(\w+)\s*", line)
        if m:
            c, e = m.group(1), m.group(2)
            fan_in[e].add(c)
            fan_out[c].add(e)
    print("# pinch point candidates (sorted by fan-in):")
    items = sorted(fan_in.items(), key=lambda kv: -len(kv[1]))
    for callee, callers in items:
        if len(callers) >= 2:
            print(f"  {callee}  fan-in={len(callers)}  callers={sorted(callers)}")
    print()
    print("# trap warnings (fan-in >= 3 → may be mini-integration test):")
    for callee, callers in items:
        if len(callers) >= 3:
            print(f"  WARN: {callee} has fan-in={len(callers)}, test may be too broad")
    print()
    print("# change-specific note:")
    print("  pinch point depends on change points, not structure.")
    print("  re-run after change scope changes.")

if __name__ == "__main__":
    main()
PY
chmod +x pinch_detect.py

# 测试 1: ch12 Invoice 例子
cat > invoice_graph.txt <<'EOF'
getValue -> itemsSum
getValue -> getLocalShipping
getValue -> getDefaultShipping
getValue -> getSpanningShipping
getValue -> getTax
makeStatement -> getValue
makeStatement -> formatHeader
EOF

echo "=== case 1: ch12 Invoice / BillingStatement ==="
./pinch_detect.py < invoice_graph.txt

echo
echo "=== case 2: ch12 expanded (Item + InventoryControl) ==="
cat > full_graph.txt <<'EOF'
getValue -> itemsSum
getValue -> shippingPricer
getValue -> getTax
makeStatement -> getValue
needsReorder -> inventory
run -> needsReorder
getSupplier -> inventory
setSupplier -> inventory
makeStatement -> getSupplier
EOF
./pinch_detect.py < full_graph.txt
```

**验收**：
- case 1：`makeStatement` 是 pinch point（fan-in=1 但 effect 链汇总到这里）；`getValue` 也有多个 callee。
- case 2：`makeStatement` 和 `run` 共同成为 pinch point；fan-in ≥3 的方法标 WARN（pinch point trap）。

### A2 — BillingSystem Pinch Point: 一组改动共用一个测试点（C 模拟）

```bash
mkdir -p /tmp/ch12-billing && cd /tmp/ch12-billing

# 复刻 ch12 Figure 12.4 - 12.5: Invoice + Item + BillingStatement,
# makeStatement 是 pinch point, 测这一个就 sense 到所有改动.
cat > billing.h <<'EOF'
#ifndef BILLING_H
#define BILLING_H
#include <stddef.h>

typedef struct {
    int id;
    long cents;
    const char *shipping_carrier;   /* ch12 新增字段 */
} Item;

typedef struct {
    Item **items;
    size_t n;
    int    billing_year;
} Invoice;

typedef struct {
    Invoice **invoices;
    size_t n;
} BillingStatement;

Item             *item_new(int id, long cents, const char *carrier);
void              item_free(Item *i);

Invoice          *invoice_new(int year);
void              invoice_free(Invoice *inv);
void              invoice_add_item(Invoice *inv, Item *i);
long              invoice_get_value(Invoice *inv);   /* ch12 改动点 */

BillingStatement *bs_new(void);
void              bs_free(BillingStatement *bs);
void              bs_add_invoice(BillingStatement *bs, Invoice *inv);

/* Pinch point: makeStatement sense 到 Invoice.getValue 改动 + Item.shipping_carrier. */
char             *bs_make_statement(BillingStatement *bs);

#endif
EOF

cat > billing.c <<'EOF'
#include "billing.h"
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

Item *item_new(int id, long cents, const char *carrier) {
    Item *i = calloc(1, sizeof(*i));
    if (i) { i->id = id; i->cents = cents; i->shipping_carrier = carrier ? carrier : ""; }
    return i;
}
void item_free(Item *i) { free(i); }

Invoice *invoice_new(int year) {
    Invoice *inv = calloc(1, sizeof(*inv));
    if (inv) inv->billing_year = year;
    return inv;
}
void invoice_free(Invoice *inv) {
    if (!inv) return;
    for (size_t i = 0; i < inv->n; i++) item_free(inv->items[i]);
    free(inv->items);
    free(inv);
}
void invoice_add_item(Invoice *inv, Item *i) {
    inv->items = realloc(inv->items, (inv->n + 1) * sizeof(Item *));
    inv->items[inv->n++] = i;
}

long invoice_get_value(Invoice *inv) {
    long total = 0;
    for (size_t i = 0; i < inv->n; i++) {
        /* 简化: shipping carrier 影响 5% surcharge. */
        long surcharge = (inv->items[i]->shipping_carrier[0] != 0) ? 5 : 0;
        total += inv->items[i]->cents + surcharge;
    }
    return total;
}

BillingStatement *bs_new(void) { return calloc(1, sizeof(BillingStatement)); }
void bs_free(BillingStatement *bs) {
    if (!bs) return;
    for (size_t i = 0; i < bs->n; i++) invoice_free(bs->invoices[i]);
    free(bs->invoices);
    free(bs);
}
void bs_add_invoice(BillingStatement *bs, Invoice *inv) {
    bs->invoices = realloc(bs->invoices, (bs->n + 1) * sizeof(Invoice *));
    bs->invoices[bs->n++] = inv;
}

char *bs_make_statement(BillingStatement *bs) {
    /* Pinch point: 输出汇总所有 invoice 的 getValue + 各自 carrier. */
    size_t bufsize = 512;
    char *buf = malloc(bufsize);
    size_t pos = 0;
    pos += snprintf(buf + pos, bufsize - pos, "BILLING YEAR(S):\n");
    for (size_t i = 0; i < bs->n; i++) {
        long v = invoice_get_value(bs->invoices[i]);
        pos += snprintf(buf + pos, bufsize - pos,
                        "  invoice[%zu]: year=%d total=%ld\n",
                        i, bs->invoices[i]->billing_year, v);
    }
    return buf;
}
EOF

cat > test_billing.c <<'EOF'
/* 测试 pinch point: makeStatement 一个 assertion sense 到:
 *   - Invoice.getValue 的 shipping surcharge 改动
 *   - Item.shipping_carrier 新增字段
 *   - invoice_add_item 的 list 管理
 * 这就是 ch12 Takeaway 3 的 "5 个 change × 1 个 test point". */
#include "billing.h"
#include <assert.h>
#include <string.h>
#include <stdlib.h>

int main(void) {
    /* === change 1: Item.shipping_carrier 新增 === */
    Item *i1 = item_new(1, 1000, "FedEx");
    Item *i2 = item_new(2, 2000, "");        /* 无 carrier, 无 surcharge */

    /* === change 2: Invoice.get_value 加 surcharge === */
    Invoice *inv = invoice_new(2025);
    invoice_add_item(inv, i1);
    invoice_add_item(inv, i2);

    /* === change 3: BillingStatement 收集 invoices === */
    BillingStatement *bs = bs_new();
    bs_add_invoice(bs, inv);

    /* === pinch point test: 一个 assertion sense 所有 change === */
    char *stmt = bs_make_statement(bs);
    /* invoice 0 contains both items: 1005 (item 1 with FedEx surcharge) +
     * 2000 (item 2 without surcharge) = 3005 total. */
    assert(strstr(stmt, "total=3005") != NULL);  /* 1000 + 5% surcharge + 2000 */
    assert(strstr(stmt, "year=2025") != NULL);
    free(stmt);

    /* 验证 pinch point 也 sense 到 "no carrier" 这条 edge case:
     *  即 Item 没 carrier 时 surcharge = 0. */
    Item *i3 = item_new(3, 500, NULL);
    invoice_add_item(inv, i3);
    stmt = bs_make_statement(bs);
    assert(strstr(stmt, "total=3505") != NULL);   /* 1005 + 2000 + 500 = 3505 */
    free(stmt);

    bs_free(bs);
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_billing billing.c test_billing.c
./test_billing
echo "rc=$?"
```

**验收**：
- `rc=0`，3 个 change（Item 新增 carrier / Invoice. getValue surcharge / BillingStatement collection）共用 `bs_make_statement` 一个 pinch point 测试。
- 注释说明：pinch point 测的是 "成片改动" 的 effect 汇总；将来 ch20 拆分 Invoice 后，pinch point 测试退役换成窄 unit test。

### A3 — Pinch Point Trap 演示：高扇入测试的 setup 成本

```bash
mkdir -p /tmp/ch12-trap && cd /tmp/ch12-trap

# 演示 ch12 Takeaway 7: pinch point 测试容易退化成 mini-integration test.
# 这里 setup 涉及 3 个真实对象, 跑得慢 + 失败难定位.
cat > trap.h <<'EOF'
#ifndef TRAP_H
#define TRAP_H
#include <stddef.h>

typedef struct { int x; } SensorA;
typedef struct { int y; } SensorB;
typedef struct { int z; } SensorC;
typedef struct { SensorA *a; SensorB *b; SensorC *c; } Aggregator;

SensorA    *sensor_a_new(int x);
SensorB    *sensor_b_new(int y);
SensorC    *sensor_c_new(int z);
Aggregator *agg_new(SensorA *a, SensorB *b, SensorC *c);
int         agg_compute(const Aggregator *agg);

#endif
EOF

cat > trap.c <<'EOF'
#include "trap.h"
#include <stdlib.h>

SensorA *sensor_a_new(int x) { SensorA *s = calloc(1, sizeof(*s)); if (s) s->x = x; return s; }
SensorB *sensor_b_new(int y) { SensorB *s = calloc(1, sizeof(*s)); if (s) s->y = y; return s; }
SensorC *sensor_c_new(int z) { SensorC *s = calloc(1, sizeof(*s)); if (s) s->z = z; return s; }
Aggregator *agg_new(SensorA *a, SensorB *b, SensorC *c) {
    Aggregator *g = calloc(1, sizeof(*g));
    if (g) { g->a = a; g->b = b; g->c = c; }
    return g;
}
int agg_compute(const Aggregator *agg) {
    /* pinch point: 一个 compute sense 到 3 个 sensor */
    if (!agg) return 0;
    return agg->a->x + agg->b->y + agg->c->z;
}
EOF

cat > test_trap.c <<'EOF'
/* Pinch point trap demo: 这个测试 setup 涉及 3 个真实对象,
 * 是 mini-integration test. ch12 Takeaway 7 警告:
 * 长期保留这种测试 = 跑得慢 + 失败难定位. */
#include "trap.h"
#include <assert.h>
#include <time.h>
#include <stdio.h>

int main(void) {
    struct timespec t0, t1;
    clock_gettime(CLOCK_MONOTONIC, &t0);

    /* setup 成本: 构造 3 个 sensor + 1 个 aggregator. */
    SensorA *a = sensor_a_new(1);
    SensorB *b = sensor_b_new(2);
    SensorC *c = sensor_c_new(3);
    Aggregator *g = agg_new(a, b, c);

    /* 真正的 assertion 只有一行, 但 setup 涉及 4 个对象. */
    assert(agg_compute(g) == 6);

    /* trap indicator: setup 数量 / assertion 数量 比值 */
    printf("objects_setup=4  assertions=1  ratio=4.0  (trap)\n");

    clock_gettime(CLOCK_MONOTONIC, &t1);
    double ms = (t1.tv_sec - t0.tv_sec) * 1000.0 +
                (t1.tv_nsec - t0.tv_nsec) / 1e6;
    printf("elapsed=%.3f ms\n", ms);

    free(a); free(b); free(c); free(g);
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_trap trap.c test_trap.c
./test_trap
echo "rc=$?"
```

**验收**：
- `rc=0`，输出显示 `objects_setup=4 assertions=1 ratio=4.0 (trap)` —— 这是 pinch point trap 的量化指标。
- 注释说明：将来 ch20 拆分 Aggregator 后，pinch point 测试退役，换 3 个窄 unit test（每个 sensor 独立测）。

### A4 — Hidden Class Detection: 用 effect sketch 找拆分边界

```bash
mkdir -p /tmp/ch12-hidden && cd /tmp/ch12-hidden

# 演示 ch12 Takeaway 5: Parser 类有 root + position + stringToParse,
# 实际上 getToken + hasMoreTokens 只用后两个, 形成天然拆分边界.
cat > parser.h <<'EOF'
#ifndef PARSER_H
#define PARSER_H
#include <stddef.h>

typedef struct {
    const char *src;
    int         pos;
    int         cur_token;        /* 由 tokenizer 管理 */
    int         root;             /* parser 自己用 */
} Parser;

typedef struct {
    const char *src;
    int         pos;
    int         cur_token;
    int         has_more;
} Tokenizer;

Parser    *parser_new(const char *src);
void       parser_free(Parser *p);
void       parser_parse_expression(Parser *p);

/* Tokenizer 是从 Parser 拆出来的 hidden class. */
Tokenizer *tokenizer_new(const char *src);
void       tokenizer_free(Tokenizer *t);
int        tokenizer_get_token(Tokenizer *t);     /* effect sketch: read-only */
int        tokenizer_has_more_tokens(Tokenizer *t);

#endif
EOF

cat > parser.c <<'EOF'
#include "parser.h"
#include <stdlib.h>
#include <string.h>

Parser *parser_new(const char *src) {
    Parser *p = calloc(1, sizeof(*p));
    if (p) p->src = src;
    return p;
}
void parser_free(Parser *p) { free(p); }

void parser_parse_expression(Parser *p) {
    /* 简化: parser 只关心 root, 不直接碰 src/pos. */
    (void)p->src;
    p->root = 42;
}

/* Tokenizer: 之前是 Parser 内部的 private, ch12 Takeaway 5 拆分出来. */
Tokenizer *tokenizer_new(const char *src) {
    Tokenizer *t = calloc(1, sizeof(*t));
    if (t) { t->src = src; t->pos = 0; t->has_more = (src && src[0] != 0); }
    return t;
}
void tokenizer_free(Tokenizer *t) { free(t); }

int tokenizer_get_token(Tokenizer *t) {
    /* effect: pos++, cur_token 被改. */
    if (!t->has_more) return -1;
    int tok = t->src[t->pos];
    t->cur_token = tok;
    t->pos++;
    t->has_more = (t->src[t->pos] != 0);
    return tok;
}
int tokenizer_has_more_tokens(Tokenizer *t) { return t->has_more; }
EOF

cat > test_hidden.c <<'EOF'
/* 验证 ch12 Takeaway 5: effect sketch 揭示 Parser 拆分边界.
 * 拆前: Parser 内部 private getToken + hasMoreTokens, 改 src 都要改 Parser.
 * 拆后: Tokenizer 独立类, 测试窄、effect 收敛. */
#include "parser.h"
#include <assert.h>
#include <string.h>

int main(void) {
    /* === Tokenizer 单独测试 (ch12 拆分后) === */
    Tokenizer *t = tokenizer_new("ab");
    assert(tokenizer_has_more_tokens(t) == 1);
    assert(tokenizer_get_token(t) == 'a');
    assert(tokenizer_has_more_tokens(t) == 1);
    assert(tokenizer_get_token(t) == 'b');
    assert(tokenizer_has_more_tokens(t) == 0);
    assert(tokenizer_get_token(t) == -1);   /* EOF */
    tokenizer_free(t);

    /* === Parser 单独测试 (拆后不需要关心 src 内部细节) === */
    Parser *p = parser_new("anything");
    parser_parse_expression(p);
    /* 简化: parse_expression 跑完没 crash 即视为 PASS. */
    parser_free(p);

    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_hidden parser.c test_hidden.c
./test_hidden
echo "rc=$?"
```

**验收**：
- `rc=0`，Tokenizer 与 Parser 解耦后可独立测试 —— 这是 ch12 Takeaway 5 的 "effect sketch 揭示 hidden class" 落地。
- 注释说明：拆分前 Parser 是一个大类；effect sketch 显示 `src` / `pos` / `cur_token` 只在 getToken + hasMoreTokens 之间共享 effect —— 这就是 hidden class 边界。

## 六、值得深入思考的问题

### Q1: Pinch point 的"真收窄"如何客观判定？

Takeaway 4 给了 change-specific 的判断，但没说"什么算真收窄"。**关键问题**：fan-in ≥ 2 是必要还是充分？fan-in 高但 effect 不重叠的节点算 pinch point 吗？

### Q2: Pinch point 测试退役的客观信号是什么？

Takeaway 7 说"eventually go away"，但没说"什么时候"。**关键问题**：拆完类、单元测试覆盖到位后，pinch point 测试立刻删？还是保留 N 个 release 作为 safety net？保留多久算"过渡态过长"？

### Q3: Change scope 过大时拆分，pinch point 怎么重新定位？

Takeaway 6 说"找不到 pinch point 就缩小 change scope"。**关键问题**：拆分后的两批改动，是否会引入新的 pinch point？还是每批单独跑独立 pinch point？两批之间的 interaction 谁测？

### Q4: Hidden class 检测能否自动化？

Takeaway 5 给的是手工流程。**关键问题**：AI / 工具能不能基于 effect sketch 自动建议类拆分？用什么启发式（field-method 共享度、effect 通道重叠度）？

### Q5: AI 主导的 pinch point trap 怎么 enforce？

AI 写"高扇入测试"非常自然。**关键问题**：CI 规则能不能 enforce"测试 setup 涉及对象数 ≤ N"？团队 review 时怎么快速识别 trap？

---

*下一章预告*: **Chapter 13 — I Need to Make a Change, but I Don't Know What Tests to Write** —— 把 ch11/ch12 的"在哪里测试"补上另一半："**写什么样的测试**"。核心武器是 **characterization test** —— 用测试"捕捉"当前行为的技法：写一个必然失败的断言，让失败信息告诉你真实行为，再把断言改成真实期望值。ch13 与 ch12 互补：ch12 决定测试点位置，ch13 决定测试点内容。

# Chapter 3 — Sensing and Separation

> **PDF**: p.43-50（8 页）
> **定位**: ch2 的 Test Coverings 推进到 *主动*拆依赖的两类动机 — **Sensing**（取不到被测代码算出的值）和 **Separation**（构造不了类实例 / 方法跑不起来）。把这一对分类普及后，ch4 才能把"插入"位置叫 seam。ch3 是骨架章。

## 〇、第一性原理思考

**这章做了什么**：把"我没法测这个类"这个笼统抱怨拆成两个正交的失败模式 —— **Sensing**（代码跑了但看不到返回效果，FakeDisplay 录 lastLine 是经典例子）和 **Separation**（类实例都构造不出来，C++ whole-program link 几 GB object 是硬约束），并用 NetworkBridge 一个案例同时占满两种失败。

**为什么这样拆**：不分清这两类失败就会乱下药 —— 把"sale 看不到 fake 字段"和"sale 根本造不出来"混在一起，必然在测试里塞一坨假依赖，导致耦合反而加深；拆开之后才能各用各的招，sensing 上 fake（轻），separation 上参数抽取 / 接口抽取（重）。

**最值钱的洞见**：Fake 实际上不是"假的对象"，而是 *两个面* —— 被测代码看到的是宽类型（如 `Display` 接口），测试看到的是窄类型（如 `FakeDisplay` 的 `getLastLine`），"宽给消费方、窄给测试方"是写 fake 的判定线，也是 mock vs fake 选择的根。
## 一、章节概述

- **拆依赖的两个独立动机：sensing vs separation**。前者为了"看见"，后者为了"放手让它跑起来"。同一段依赖可能先 separation（先能造出对象）再 sensing（再能看见效果）— 两步可分可合。
- **NetworkBridge 案例**：依赖 = EndPoint + 硬件回调 + 真实网络。造不出实例 (= separation)；即便造出来，看不到对硬件的影响 (= sensing)。
- **Faking Collaborators 是 sensing 的主力技法**。把"目标类调用的协作方"用一个 *行为相似但可观察* 的替身替代。目标类看到的是 collaborator；测试看到的是 fake。
- **FakeDisplay 是 sensing 的最小可工作例子**：Sale. scan() → display. showLine("Milk $3.99") → fake 记录 → test 读 lastLine。两个事实：(a) sale 看不到 fake 的记录字段；(b) test 看不到真硬件。
- **Fake 的"两个面"**：sale 看到的是 `Display` (interface)；test 看到的是 `FakeDisplay` (具体类，含 `getLastLine`)。声明类型必须"宽给消费方、窄给测试方"。
- **Fake vs Mock**：fake 是被动记录，断言在 test 里；mock 在 fake 基础上把 assertion **嵌入 fake 内部**（`setExpectation` + `verify`）。两者选哪个取决于 是否需要"对调用契约的精确断言"。
- **Mock 框架语言支持有限**：C++/嵌入式无成熟 mock framework → fake 仍是最务实选择（ch19 把这扩展到非 OO）。
- **C++ link-time 是 separation 的硬约束**：whole-program link 几 GB object 后才能跑，"rapid turnaround 不可能" → 必须 break dependencies。这条对 kernel/userspace daemon 同理。
- **Fake 的图示演进**: Figure 3.1 (单类) → 3.2 (抽 display 类) → 3.3 (接口 + 实现) → 3.4 (Fake 的两个面)。
- **Fake 的"为什么不真测试显示"反诘**：FakeDisplay 不验证硬件 — 但它验证 Sale 调用契约。divide and conquer 是测试的本质（与 ch2 的 localize 原则一致）。

## 二、核心 Takeaways

### Takeaway 1: 拆依赖 = Sensing ∪ Separation

- **是什么**: 让类不可测的依赖要么是"塞不进 test harness"（separation），要么是"塞进去了但看不到返回效果"（sensing）。两个独立的困难，可分别动手。
- **为什么重要**: 不分清就乱拆。把"造不出来"混在"看不到结果"里导致过度注入 — 真正要做的是先把 separation 干了（Adaptive Constructor / 参数提取），再看是否 sensing 仍必要。
- **解决什么问题**: 工程上 — 让抽象代价归一。sensing 通常用 fake 解决（轻），separation 用参数抽 / 接口抽（重）。
- **适用场景**: 评估一个不可测类时，先问"造不出还是看不到" — 然后看 ch25 catalog 里哪种 refactoring 对应。

### Takeaway 2: Fake 是 sensing 的最小可工作单位

- **是什么**: 替身类，行为仿真 target collaborator 的一小段（往往仅"被调用时记录了什么"）。test 用它探测被测代码与协作方的关系。
- **为什么重要**: 不需要 mock framework — 任何 OO 语言几行类即可完成。
- **解决什么问题**: 引入"无 mock 框架也能 sensing" 的工业路径。
- **适用场景**: 嵌入式 / 旧 C++ / 内核等无成熟 mock framework 的环境。

### Takeaway 3: Fake 的两个面 — Producer side 与 Test side

- **是什么**: SUT（被测代码）看到的是 narrow interface（要看到的协作方法），test 看到的是 wide concrete fake（额外有 test-only 的 `getLastLine`）。
- **为什么重要**: 类型声明分两端是 sensing 设计的关键 — 一端不可松，否则 SUT 误以为真硬件；另一端不可松，否则 test 拿不到记录。
- **解决什么问题**: "我 fake 了还得加 getter 才能验" 的工程步骤合法化。
- **适用场景**: 任何 fake / spy 设计；包含 instrumentation 时（例如 syscall log）。

### Takeaway 4: Fake 是 passive, Mock 是 active

- **是什么**: Fake 只记录，由 test 在最后 assert。Mock 把"assert 什么"预先注册进 fake，调用完调 `verify()`。
- **为什么重要**: 选择规则：mock 适合 *精确调用契约*（次数 / 时序 / 参数相等），fake 适合 *关心 final state*。
- **解决什么问题**: 当测试需要验证"showLine 被调一次且参数 = X"，mock 显式；当只需要"显示内容最终 = X"，fake 简洁。
- **适用场景**: 协议行为测试（mock 更直白）；GUI 显示文本测试（fake 简洁）。

### Takeaway 5: 在 C++ 中 link time 是 separation 的隐形税

- **是什么**: 不拆依赖 = 整项目 link。在嵌入式 / 大型 C++ 项目里一次链接几分钟到几十分钟 — feedback loop 被强行拉到不可承受。
- **为什么重要**: 即使有 mock framework，不拆依赖仍然浪费时间。Feathers 在本章明确点出 "C++ link time alone can make rapid turnaround nearly impossible"。
- **解决什么问题**: 把"为什么 compile 也要等这么久" 转译为 "separation 这步不轻"，让团队在 Sprint planning 时显式测拆分。
- **适用场景**: 任何 C++ / Rust workspace crate 重编译 > 30s 时；CI 加速已经不够用时。

### Takeaway 6: Test Side 引入 `getLastLine` 不是 design debt，是 sensing 成本

- **是什么**: test-only getter 在 production 路径上不可见，但 fake 类有这方法。
- **为什么重要**: 不会让 production 多调用一行。让 fake 类变"瘦的 engineering artifact"，不是 abstraction leak。
- **解决什么问题**: 避免 "我们要测，but production 不能 import test" 的过度设计。
- **适用场景**: 每个 fake 的设计评审。

### Takeaway 7: Figure 3.4 是 microcosm — 一图胜千言 ch3 全文

- **是什么**: `FakeDisplay` 一个类，画分两半 — "sale 看到的" + "test 看到的"。两半在生产代码里都在同一类对象上，但代码视角分两边。
- **为什么重要**: 团队新人看到 fake 第一反应是 "这不会有 N+1 个方法吗？"。图 3.4 把这反应抵消。
- **解决什么问题**: 设计讨论的语言统一 — "你看 SUT 侧 / Test 侧" 的 mind model。
- **适用场景**: code review 时解释"为什么 fake 长成那样"。

## 三、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **sensing 在内核的"两个面" = kprobe / ftrace**。SUT 侧 = 内核函数调用；test 侧 = ftrace ring buffer / kprobe trace。type 同样分两端：函数是 strong type 的；从 buffer 取值是"test 知道发生了什么而 SUT 不知道被看见"。
- **separation 在内核的硬约束 = 模块依赖**。一个带 random_state 的 crypto API 想在 unit test 里跑 — separation 必须把 `get_random_bytes` 抽接口。这正是 `ch25: Extract Interface` 的 kernel 对应（实际叫 `crypto API indirection`）。**Linux crypto subsystem 已开始正式支持 test-against-mock-drbg**。
- **C++ link time ≈ kernel build time**。Linux 全树编译用 ccache + distcc 仍 30-60 分钟；subsystem 单独测也要几分钟 — feedback 不能 "fast"。`make -j$(nproc) M=drivers/net/wireless/` 单独编译一个 driver 已经是最佳 feedback loop。**启示**: separation 在内核等同于 "用 KUnit / 拆出独立 test target"。
- **C 的"fake"轻量但粗糙**。内核某次要 sense `kmalloc` 行为，fake 不是建一个 mock_kmalloc — 而是写一个 `TEST_DRIVER` 装回 fake handler。**这告诉 fake 的工程极限**：便宜的 fake 用 function-pointer injection（运行时替身），昂贵但干净的 fake 用真实 mock object（C++/Java 风格）。
- **`strace` 本身是 sensing 工具** — 它 *假冒* syscall handler 切入 user-kernel 边界。但 strace 是 out-of-band（不修改 production），它不能用于 *active assertion*；要主动断言必须用 seccomp-bpf + fake seccomp。
- **`lwn.net/Articles/824570`** 之类文章在 kernel 用 KUnit 已经覆盖了 sensing/separation 的常态化。这也是 *test-covered islands* 在基础设施层的版本。

#### Linux 系统 — Fake 实现路径对比

| 测试场景                | sensing 手段                  | separation 手段                  | 备注                       |
| ----------------------- | ---------------------------- | -------------------------------- | -------------------------- |
| 文件 I/O                | `LTP`/`fstest` + 假文件系统 | in-memory FS (ramfs)              | sysfs entry mock 常用      |
| 网络 IO                 | TCP loopback + tap           | `nsim` mock dev                   | devlink 替代 netdev        |
| 系统调用                | `strace` / `seccomp`         | seccomp-bpf 自定义规则            | 内核测试套常用              |
| 时间依赖                | test-time injected           | `time_t mock_gettimeofday()`      | 单进程函数指针替换           |
| 内存分配                | `KMEM_CACHE` test target     | `kmem_cache_create` 注入          | SLUB 已有自检               |
| Crypto API              | DRBG indirection              | `crypto_register_template`       | kernel 自带 mock DRBG       |
| Device Tree / ACPI      | DT 节点 faked                 | `of_node_put` / `of_node_get` 抽象 | OF graph 抽象               |

### 3.2 机器人软件视角

- **Fake 在 ROS2 节点间的"两个面"**：`Publisher` interface（SUT 看到）vs `RecordingPublisher` (test side with get_last_message())。这与 `gmock` 的 "two sides" 设计对应 — ROS2 的 `test_msgs` 已经把这抽象为公共依赖。
- **separation 在 ROS2 的硬约束 = DDS discovery**。节点不直接连 → 必须有 DDS 中间件 — 单元测试无法不假 DDS 启动。**业界做法**：`rclcpp::init` 替换为 fake init + `rclcpp::Node` 替换为 `rclcpp::Node::make_shared(fakes)`，separation 由 `ros2_control`'s `hardware_interface::SystemInterface` 完成。
- **fake move_base 的方法**：split `MoveBase` 为两层 — `navigation_planner` (可 fake) + `controller` (可 fake)。原 ROS1 `move_base` 一直是 monolithic。ROS2 Nav2 已经把它按 smac_planner / controller_server / bt_navigator 切。
- **fake 真实硬件 = ros2_control's mock hardware_interface**：sensor data 注入 by fake publisher；电机命令 as fake system。**这正是 ch3 fake 的工业实例**。
- **sensing 在 ROS2 中常用 `MockClock`** — 时间序列仿真的 fake — 让 BT（行为树）可测。Nav2 早期 PR 大量依靠 MockClock 让 BT node "不是真的等待一秒"。
- **Python `unittest.mock`** 对 ROS2 节点是首选 — 因节点通常很小，sensing 取自 return value 即可。

#### 机器人软件 — Fake 插入点表

| ROS2 组件            | separation 路径                  | sensing 实现                       |
| ------------------- | -------------------------------- | ---------------------------------- |
| `rclcpp::Node`     | `MockNode` 自定义 / `node_factory::testing` | topics / params 通过 fake pub/sub     |
| `MoveBase` action  | 拆 planner / controller / bt       | mock_costmap + bt_node recorder    |
| `ros2_control` HW  | `hardware_interface::SystemInterface` mock | `joint_state_broadcaster` w/ fake cmd |
| `image_pipeline`   | 拆 transport / decode / proc      | `image_transport::Publisher` fake  |
| `tf2`              | 静态 broadcaster                  | `Buffer` fake w/ recorded transforms |
| `nav2_bt_navigator`| 拆 BT XML / engine                | `MockClock` + recorder behaviors    |
| `diagnostic_aggregator` | filter graph 拆单 filter        | record filtered diagnostics        |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                   | 高级工程师                                                   |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 看到不可测           | 加 mock framework                                            | 先分 sensing / separation — sensing 用轻 fake               |
| 写 fake 时           | 全部方法都 stub（含内部 helper）                             | 只 stub SUT 实际调用的窄面（"showLine 即可"）              |
| Fake vs Mock 抉择    | 默认用 mock，因为它"看起来专业"                              | 看测试关心的是 "调用契约" 还是 "final state" 来定             |
| 拆依赖顺序           | 先把生产代码改成完全 DI                                      | separation 先行；sensing 视需要再 fake                     |
| C++ 慢链接           | 加 mock framework，然后等链接                                 | 不拆依赖就改 => 等链接 N 分钟是工程债；refactor 出可单独测的单元 |
| 类型声明             | `FakeDisplay display = new FakeDisplay();`; 同时把它当 display 用 | 消费侧声明 `Display`，test 侧声明 `FakeDisplay` — 类型即文档   |
| 看到 `getLastLine`   | "这方法是测试专属的，污染设计"                               | "这是 sensing 的 cost，正常；test only 不进 production 调用" |

> **关键差异**: 高级工程师按 sensing / separation 分类决定拆法，fake/mock 选择按测试焦点决定。初级倾向于"先 mock 再说"。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

仍然极重要。原因：
- **AI 写 fake 是它的强项**（不需要理解领域），但写 FakeObject 时**它最容易搞错的是 "two sides"** — 它会不小心把 fake 的 test-only getter 暴露给 production interface。
- **AI 默认生成 mock**，但很多场景 fake 更轻；AI 不知道该用哪种。
- **separation 在不修代码 → AI 帮你生成接口** 这个套路，会让 *修代码 = 先妥协 design*；AI 看不出来"妥协是否值得"。
- **C++ link-time 隐式税没消除** — AI 不能让编译时间变快。

### 4.2 AI 已经能做的（具体到 ch3 主题）

- **生成 fake 的初稿**（基于 SUT 调用的方法集合）：识别 collaborator + 自动生成 stub — 准确率 60–80%，review 一遍够。
- **识别 sensing vs separation**：给一段不可测代码，AI 把"造不出" 和 "看不到结果" 标出来 — 准确率 70%。
- **mock vs fake 的推荐**：基于测试 focus（state vs interaction），AI 给出推荐。

### 4.3 AI 不能替代的（具体到 ch3 主题）

- **C++ / 内核 link-time 的实际测量** — 这是 throughput 心智模型，AI 不感知。
- **decide 哪一面是 test 侧** — 这是 design judgement：比如 `lastLine` getter 是不是该在 fake 里，还是该用 thread-local 测试变量？AI 给两种方案都说得通，但哪个适合团队节奏是人决定。
- **判断 fake 长期 maintenance cost** — 一个测试 helper API 设计暴露给全 team 用了 6 个月后，要 refactor 时哪个最痛？AI 看不到团队长尾反馈。

### 4.4 AI 经常写错的地方

针对 ch3 Sensing/Separation/Fake/Mock 主题：

| 错误模式                                       | 例子                                                                 | 后果 |
| ---------------------------------------------- | -------------------------------------------------------------------- | ---- |
| **fake 把 test-only getter 暴露成 public**      | `FakeDisplay.showLine` 改成 `getLastLine` 也走同样 source，`/api` 端点列出 | SUT 误用 test-only 方法; 行为失真 |
| **默认选 mock 而不该**                          | 测试关心 "最终屏幕上" 啥内容, AI 仍用 `MockDisplay.verify()`         | test 关心 state 转 interaction, 改需求后回归不暴露 |
| **fake 仿真过度**                                | AI 给 fake 实现 stub `getValue` + `setValue` + `flush` + `connect` + ... | fake 比 SUT 还复杂; 维护成本爆表 |
| **fake 不放 sensing 限制**                      | AI 写的 fake 没 records 调用次数, 测试 hidden assumption "调用一次" | SUT 改成调用两次, test 不报 — false negative |
| **separation 拆错**                              | AI 看到 `get_random_bytes` 调用, 直接抽 global function pointer, 让 production 也变 | production 性能 +1ns, 但 analysis 强行 indirection |
| **decide sensing 的边界失误**                    | AI 给 `NetworkBridge.formRouting` 测 fake，把整个 EndPoint 都假了   | 测试生产 nothing，只验证了接口形状 |
| **mock 的 setExpectation 漏次**                  | mock 没 assert "只调用一次" 但业务要求只一次                         | SUT 偷偷调用两次, verify() 仍通过 |
| **fake 不识别"final state 是协作方的属性"**      | AI 直接 fake final state 而非 fake collaborator                     | 测试测的不是 SUT 行为，而是 fake 自己的字段 |
| **producer 与 test 端类型不分**                  | AI 让 SUT 直接引用 `FakeDisplay` 类, 不是 `Display` interface        | 测试和生产耦合, fake 隐式成为 production 代码的 path |

### 4.5 子段: AI 辅助遗留代码理解 — 在本主题专项

- **AI 帮你识别"该 fake 谁"**：给一段不可测方法 → AI 推荐 N 个 candidate collaborator + 推荐每个拆哪一段 (`Extract Interface` / `Adapt Parameter`)。准确率中上。
- **AI 帮你维护 fake suite** — 项目里 100 个 fake → AI 同步 mock framework 升级。**风险**: fake 升级后行为变了，测试仍绿（mock 框架的 façade 差可能掩盖）。
- **AI 不会自动 sensing** — 它默认 view-level inspection，写不到行为 → 必须有 Fake/Recording 才能动 = 这条得人警惕。
- **AI 推荐"加 instrumentation"代替 fake** — 这是个常见 *semantic-mismatch*。instrumentation 是 read side，fake 是 write side 也能控。AI 给"加 tracepoints"，人得分辨。

### 4.6 工程师必须保留的核心能力

- **判断 fake/mock 选择**，是基于测试 focus 的，不是基于"看起来专业"。
- **类型声明分两端**：product 类型窄 / test 类型宽。这是 review 时一眼能看出来 design debt 的点。
- **sensing / separation 分类** — 任何不可测代码第一步都是这分类。
- **fake 不能侵入 production**：review 时检查 FakeDisplay 类是否只被 test 文件 import。
- **测试 helper API 设计**是工程债 — 长期看 fake reuse 应当有"test utilities"命名空间 / 子模块的归属，而不是散落。

## 五、实践行动项

> 本章是 ch4 seam 的铺垫章，行动项聚焦在 (a) 用 C 复刻 Feathers 的 Java FakeDisplay/Sale (b) 用 fake 命中常见的 sensing 场景 (c) mocking 在 fake 上的最小进化。

### A1 — C 版 FakeDisplay / Sale: 完整复刻 ch3 Sale 例子

```bash
mkdir -p /tmp/ch03-senssep && cd /tmp/ch03-senssep

cat > sale.h <<'EOF'
/* ch3 Sale + Display/ArtR56Display/FakeDisplay 的 C 复刻 */
#ifndef SALE_H
#define SALE_H

#include <stdio.h>
#include <string.h>
#include <stdlib.h>

typedef struct Display Display;
struct Display {
    /* SUT 看到的"窄面" — Sale 只调 showLine */
    void (*show_line)(Display *self, const char *line);
    void *impl;       /* 隐藏具体实现 */
};

/* 真实硬件的实现: 真会打印到 stderr (模拟串口) */
Display *art_r56_display_new(void);

/* Fake: 只记录最后一次 showLine 的内容 */
Display *fake_display_new(void);
const char *fake_display_last_line(const Display *d);  /* test side 窄接口 */

/* Sale: 只看 Display */
typedef struct {
    const char *barcode;   /* demo: 1 = "Milk $3.99" */
    Display    *display;
} Sale;

static inline void sale_init(Sale *s, Display *d) { s->display = d; }
void sale_scan(Sale *s, const char *barcode);

#endif
EOF

cat > sale.c <<'EOF'
/* Sale 实现: 显示 "NAME $PRICE" */
#include "sale.h"

void sale_scan(Sale *s, const char *barcode) {
    /* 演示数据: 简单 switch */
    const char *line = NULL;
    int barcode_int = atoi(barcode);
    if      (barcode_int == 1) line = "Milk $3.99";
    else if (barcode_int == 2) line = "Bread $2.49";
    else                       line = "Unknown $0.00";
    s->display->show_line(s->display, line);
}
EOF

cat > displays.c <<'EOF'
#include "sale.h"
#include <stdio.h>

typedef struct { int n_calls; } ArtR56State;
typedef struct { char last[128]; int n_calls; } FakeState;

static void art_show(Display *self, const char *line) {
    ArtR56State *st = self->impl;
    fprintf(stderr, "[ART] %s\n", line);
    st->n_calls++;
}
Display *art_r56_display_new(void) {
    static ArtR56State state;
    state.n_calls = 0;
    static Display d;
    d.show_line = art_show;
    d.impl = &state;
    return &d;
}

static void fake_show(Display *self, const char *line) {
    FakeState *st = self->impl;
    strncpy(st->last, line, sizeof(st->last)-1);
    st->last[sizeof(st->last)-1] = 0;
    st->n_calls++;
}
Display *fake_display_new(void) {
    static FakeState state;
    state.last[0] = 0;
    state.n_calls = 0;
    static Display d;
    d.show_line = fake_show;
    d.impl = &state;
    return &d;
}
const char *fake_display_last_line(const Display *d) {
    return d && d->impl ? ((FakeState *)d->impl)->last : NULL;
}
EOF

cat > test_sale.c <<'EOF'
/* 单元测试: 用 fake 验证 sale -> showLine 契约 */
#include "sale.h"
#include <assert.h>
#include <string.h>

int main(void) {
    Display *disp = fake_display_new();
    Sale s; sale_init(&s, disp);

    sale_scan(&s, "1");
    assert(strcmp(fake_display_last_line(disp), "Milk $3.99") == 0);
    /* 二次扫描确认 fake 是否记录"最后一次" */
    sale_scan(&s, "2");
    assert(strcmp(fake_display_last_line(disp), "Bread $2.49") == 0);

    fprintf(stderr, "test_sale PASS\n");
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_sale sale.c displays.c test_sale.c
./test_sale
echo "rc=$?"
```

**验收**: `test_sale PASS`, `rc=0` (证明 fake 记录了 Sale 的 `showLine` 调用, test 侧 narrow interface 拿到 lastLine 并 assert 通过)。

### A2 — Sensing vs Separation 二分类 CLI: 给一段不可测代码快速判定

```bash
mkdir -p /tmp/ch03-senssep && cd /tmp/ch03-senssep

cat > classify.py <<'PY'
#!/usr/bin/env python3
"""classify.py — 给一段不可测描述, 判定 sensing vs separation.
规则:
  separation 信号: "construct*" "new(" "instantiate" "link" "DB" "connect" "sockets?" "file open" "hardware" ...
  sensing     信号: "void" "no return" "private" "set internal" "callback" "can't see" "cant see" ...
"""
import re, sys
SEP = re.compile(
    r"\b(construct(?:or|ion)?|new\s*\(|\binstantiate|link"
    r"|static\s+state|singleton|DB|sockets?|connect|file\s+open"
    r"|hardware)\b", re.I)
SEN = re.compile(
    r"\b(void|no\s+return|private|set\s+internal|write[- ]only|stateful"
    r"|update\s+internal|callback"
    r"|can'?t\s+see|cant\s+see|no\s+way\s+to\s+see)\b", re.I)
desc = " ".join(sys.argv[1:]) if len(sys.argv) > 1 else sys.stdin.read()
sep = bool(SEP.search(desc))
sen = bool(SEN.search(desc))
if sep and sen:
    print("MIXED — both Separation and Sensing")
    print("  →  step1: separation (adapter / Extract Interface), then")
    print("  →  step2: sensing   (fake / spy)")
elif sep:
    print("Separation")
    print("  →  ch25: Parameterize Constructor / Adapt Parameter / Extract Interface")
elif sen:
    print("Sensing")
    print("  →  ch3 fake; if more call-pattern: ch3 mock")
else:
    print("UNKNOWN — neither signature; re-describe")
PY
chmod +x classify.py

echo "case 1: NetworkBridge:"
./classify.py "constructor makes sockets; can't see hardware effect"
echo
echo "case 2: void update inventory silently:"
./classify.py "method returns void, updates internal state privately"
echo
echo "case 3: getter-only mock:"
./classify.py "method returns the total but private cache can't be queried"
```

**验收**: 3 case 各自落到正确的分类（Sensing / Separation / Sensing）+ 给出对应 ch25 章节号。

### A3 — 从 Fake 演化到 Mock: 嵌入断言的最小进化

```bash
mkdir -p /tmp/ch03-senssep && cd /tmp/ch03-senssep

# 复用 A1 的 sale.h, displays.c, sale.c
cat > mock_display.h <<'EOF'
/* 在 fake 上加 embedded expectation + verify */
#ifndef MOCK_DISPLAY_H
#define MOCK_DISPLAY_H
#include "sale.h"

typedef struct {
    Display base;            /* 即父类 fake */
    char    expected[128];
    char    actual[128];
    int     call_count;
    int     verified;
} MockDisplay;

Display *mock_display_new(const char *expected);
int      mock_display_verify(Display *d);    /* 1 == ok, 0 == fail */

#endif
EOF

cat > mock_display.c <<'EOF'
#include "mock_display.h"
#include "sale.h"
#include <stdio.h>
#include <string.h>

typedef struct { char last[128]; int n_calls; } FakeState;

static void mock_show(Display *self, const char *line) {
    FakeState *st = self->impl;
    strncpy(st->last, line, sizeof(st->last)-1);
    st->last[sizeof(st->last)-1] = 0;
    st->n_calls++;
    ((MockDisplay *)self)->call_count = st->n_calls;
    strncpy(((MockDisplay *)self)->actual, line,
            sizeof(((MockDisplay *)self)->actual)-1);
}

Display *mock_display_new(const char *expected) {
    static FakeState fake_state;            /* 单例 demo */
    fake_state.last[0] = 0;
    fake_state.n_calls = 0;
    static MockDisplay m;
    m.call_count = 0;
    m.verified = 0;
    strncpy(m.expected, expected, sizeof(m.expected)-1);
    m.actual[0] = 0;
    m.base.show_line = mock_show;
    m.base.impl = &fake_state;
    return &m.base;
}

int mock_display_verify(Display *d) {
    MockDisplay *m = (MockDisplay *)d;
    m->verified = 1;
    if (strcmp(m->expected, m->actual) != 0) {
        fprintf(stderr, "MOCK FAIL: expected \"%s\" got \"%s\"\n",
                m->expected, m->actual);
        return 0;
    }
    fprintf(stderr, "MOCK OK: %s (called %d times)\n", m->actual, m->call_count);
    return 1;
}
EOF

cat > test_mock_sale.c <<'EOF'
#include "mock_display.h"
#include "sale.h"
#include <assert.h>

int main(void) {
    Display *d = mock_display_new("Milk $3.99");
    Sale s; sale_init(&s, d);

    sale_scan(&s, "1");
    int ok = mock_display_verify(d);
    assert(ok);
    return ok ? 0 : 1;
}
EOF

cc -std=c17 -Wall -Wextra -o test_mock_sale sale.c mock_display.c test_mock_sale.c
./test_mock_sale
echo "rc=$?"
echo "--- failure case (expect Bread got Milk) ---"
cat > test_mock_fail.c <<'EOF'
#include "mock_display.h"
#include "sale.h"
#include <stdio.h>

int main(void) {
    Display *d = mock_display_new("Bread $2.49");
    Sale s; sale_init(&s, d);
    sale_scan(&s, "1");     /* 实际扫描 1, 输出 Milk */
    return mock_display_verify(d) ? 0 : 2;
}
EOF
cc -std=c17 -Wall -Wextra -o test_mock_fail sale.c mock_display.c test_mock_fail.c
./test_mock_fail ; echo "rc=$?"  # 期望 rc=2 (MOCK FAIL)
```

**验收**:
- 合法用例 rc=0 (mock verify OK)
- 失败用例 rc=2 (mock verify FAIL 输出 "expected Bread got Milk")

### A4 — Linux sensing: 用 LD_PRELOAD fake `clock_gettime` 测时间敏感代码

```bash
mkdir -p /tmp/ch03-senssep && cd /tmp/ch03-senssep

# 1) 一个用真实时间的"贵"作业
cat > time_user.c <<'EOF'
/* 用真实时间的代码 */
#define _POSIX_C_SOURCE 200809L
#include <time.h>
#include <stdio.h>

double elapsed_seconds(struct timespec *t0) {
    struct timespec t1;
    clock_gettime(CLOCK_MONOTONIC, &t1);
    return (t1.tv_sec - t0->tv_sec)
         + (t1.tv_nsec - t0->tv_nsec) / 1e9;
}

int main(int argc, char **argv) {
    if (argc < 2) {
        fprintf(stderr, "usage: %s <seconds>\n", argv[0]);
        return 2;
    }
    int secs_to_wait = atoi(argv[1]);
    struct timespec t0, now;
    clock_gettime(CLOCK_MONOTONIC, &t0);
    struct timespec target = { t0.tv_sec + secs_to_wait, t0.tv_nsec };
    /* 模拟长任务: 等到 target 时间 */
    while (1) {
        clock_gettime(CLOCK_MONOTONIC, &now);
        if (now.tv_sec > target.tv_sec ||
            (now.tv_sec == target.tv_sec && now.tv_nsec >= target.tv_nsec))
            break;
    }
    printf("waited ~%d s, measured=%.3f s\n",
           secs_to_wait, elapsed_seconds(&t0));
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o time_user time_user.c

# 2) 一个 LD_PRELOAD fake: 替换掉 clock_gettime, 时间往前走但每次少走.
#    这样程序立即返回, 但 elapsed_seconds 仍能算出一个看上去合理的值.
cat > fake_clock.c <<'EOF'
/* fake_clock.c — fake clock_gettime, 让每次读到的 nanos + 1ms
 * 用法: LD_PRELOAD=./fake_clock.so ./time_user 1000
 * 程序说"等 1000 秒", 实际我们 fake 时钟每 1ms 走 1000s, 所以调用 ~1 次就结束.
 */
#define _POSIX_C_SOURCE 200809L
#include <time.h>
#include <stdint.h>
#include <dlfcn.h>

static uint64_t fake_nanos = 0;

int clock_gettime(clockid_t id, struct timespec *tp) {
    if (id == CLOCK_MONOTONIC) {
        fake_nanos += 1000000ULL * 1000; /* 每次 clock_gettime +1 秒 */
        tp->tv_sec  = fake_nanos / 1000000000ULL;
        tp->tv_nsec = fake_nanos % 1000000000ULL;
        return 0;
    }
    /* 非 MONOTONIC 走 glibc 实现 */
    int (*real)(clockid_t, struct timespec *) = dlsym(RTLD_NEXT, "clock_gettime");
    return real ? real(id, tp) : -1;
}
EOF
cc -std=c17 -shared -fPIC -o fake_clock.so fake_clock.c -ldl

echo "=== no fake (real time, slow) ==="
/usr/bin/time -f "%e s elapsed" ./time_user 3
echo
echo "=== with fake (instant) ==="
LD_PRELOAD=./fake_clock.so /usr/bin/time -f "%e s elapsed" ./time_user 3
echo "rc=$?"
```

**验收**:
- 无 fake: `time_user 3` 实际等 3 秒，`%e s elapsed` 约 3。
- 有 fake: 同样命令，测量出的 elapsed 时间 ≤ 0.05 秒 (fake 时钟跑得快)，但输出 "waited ~3 s, measured=3.000 s"（因为 fake 给出的时间序列合理）。

> ⚠️ 这是 sensing 跨进程边界的标准技法 — 但仅适用 *进程内* 用 LD_PRELOAD。内核态要 fake 得走 ftrace / kprobe。

## 六、值得深入思考的问题

### Q1: Fake vs Mock 的边界是不是应该靠工具决定还是靠测试 focus 决定？

Feathers 偏向"focus 决定" — state 测用 fake，interaction 测用 mock。但工具论（"mock framework 一致好用"）往往偏离本意。**关键问题**：团队是否应该把"哪个工具用哪种测试类型"写进测试规约？或保持纯个人偏好？

### Q2: SUT 侧的接口最小化是否总等于好设计？

Feathers 假设"窄面"等于好。但生产中 `Display` 接口可能只需要 1 个方法，但 SUT 的可测试性要求它有 5 个（恰好是测试焦点需要 examine 的 5 个 getter）。**关键问题**：测试可观察性需求对接口形状的拉动力，会不会反过来破坏生产 API 的"窄面"？

### Q3: LD_PRELOAD 这种进程外 fake 是不是 testing debt？

它让 fake cross-process boundary 工作（即不修改 product），但让 build / CI 流程多一份 LD_PRELOAD 机制。**关键问题**：LD_PRELOAD 模拟的成本低（不必动生产），但长期看 — 会不会因为"加一点 LD_PRELOAD 是廉价的"导致 fake 设计被滥用，丧失 native 的可测性？fake 应该原生的还是 trade-off？

### Q4: AI 写 fake 是不是会消灭"sensing 分两类"的判断？

AI 可以把"无法测"自动写为 mock。即使是 sensing 类的场景，AI 也可能写出重型 mock framework 调用 — 实际更适合 fake 时却上 mock。**关键问题**：团队里这种"AI 一键加 mock"的习惯，会不会让"sensing vs separation"这种概念性区分实际褪色？未来的工程师还需要这两个分类吗？

### Q5: Test Side 的 getter 在多线程场景下还是不是 OK 的？

`getLastLine` 在 sale 单线程场景工作正常。但 fake 在多线程下，读 `lastLine` 时另一线程可能正在写 — visibility 飘忽。**关键问题**：fake 的 test getter 该不该 atomic / volatile / mutex 保护？什么情况下 fake 必须升级为 thread-safe？这和 mock framework 的内部 lock 是同样的问题吗？

---

*下一章预告*: **Chapter 4 — The Seam Model** — 把 ch3 的 sensing/separation 抽象到一个对象理论：**seam**。一个 seam = 一个"可以替换行为而不动 place"的位置；Feathers 给出 5 类 seam（preprocessor / link / object / runtime-parameter / class-inheritance-not-visible）。每个 seam 都映射到具体可用的拆依赖技法，ch25 catalogue 后面会反复引用。ch4 是把 ch3+ch5 之间的所有 case 归到一个名字系统。

# Chapter 4 — The Seam Model

> **PDF**: p.51-66（16 页）
> **定位**: ch3 的 Sensing / Separation 抽象到一个**对象理论**：**seam**。任何"不动 place 就能换行为"的位置 = seam；每一个 seam 都有一个 **enabling point** 在 place 之外。本章是后续 ch6–ch25 的隐式索引系统 — 每个技法本质上是"在某个 seam 上启用某条替代行为"。

## 〇、第一性原理思考

**这章做了什么**：把"能不能换行为"从一个 *位置*（seam）抽出来，再用 CAsyncSslRec:: Init() 一个例子并排放了 3 种 seam（object seam 用 TestingAsyncSslRec 子类覆盖、link seam 换 stub library、preprocessing seam 用 `#define PostReceiveError`），最后加一个隐式的 *class-inheritance-not-visible*（static method 转 protected instance method）。

**为什么这样拆**：OO 时代被教的是 *class layer 抽象*（继承、接口、设计模式），但遇到 legacy 代码这些抽象都没了 —— seam 是一个 *lens*，让你重新看见"被隐藏的 enabling point"，不给你新设计，而是给你一套读既有代码的 X 光。

**最值钱的洞见**：seam 和 enabling point *必须分开* —— enabling point 总在 seam 之外（构造点、build script、宏定义文件），这是 link seam 在 build script 里"看不见但用了就要让 test/prod 区别显眼"的根，也是 ch25 那 24 个 dependency-breaking 技法能按 primary seam type 索引的根本原因。
## 一、章节概述

- **Seam = a place where you can alter behavior in your program without editing in that place**（核心定义）。Enabling point = 让"使用行为 A 还是行为 B"的决定所在 — 总是 *在 seam 之外*。
- **Seam view 与 sheet of text view 的对照**。Feathers 第一台 DEC VAX + account 计费时代，"一段程序 = 一张长 listing"，每个调用必须编辑源码才能改。现在的程序仍是"长 listing"，但 seam 让"不到 editing at place"成为可能 — 这是被忽视的现状。
- **CAsyncSslRec:: Init() 例子的 3 种 seam**：
  - **Object seam** — 给 `CAsyncSslRec` 加 `virtual void PostReceiveError(...)`；test 用 `TestingAsyncSslRec` 子类覆盖。
  - **Link seam** — 抽 stub 函数进 stub library，test build 用 stub，production 用真库。
  - **Preprocessing seam** — `#define TESTING` 后 `#define PostReceiveError(args) lastItem=...; lastCode=...`；test 时启用宏。
- **4 类 seam + 一个隐式辅助**：
  - **Preprocessing seam**（C/C++）— `#ifdef` / `#define` 在编译前改文本。生成新版本程序（"maintain several different programs"），但换行为时不动 call site。
  - **Link seam** — 编译完到可执行文件中间那道 link 步骤可换实现；Java 是 classpath 替换，C/C++ 是 `.a/.so` 替换。
  - **Object seam** — OO 内最自然：virtual call 由对象多态决定行为；构造点 = enabling point。
  - **Class-inheritance-not-visible** (隐式) — static method 转 protected instance method 让 override 成为可能。
- **Decide build vs production 的可见性**。Link seam 的 enabling point 在 build script 里 — *容易看不见*，但用了就要 *test vs production environment 区别显眼*。
- **Class layer 抽象 vs seam layer 抽象**。seam 不是另一个设计方法，是 *理解既有代码的 lens* — 它让你看见"被隐藏的 enabling point"。
- **Choose right seam**。Feathers 倾向：OO 语言首选 object seam；preprocessing / link seam 留给"dependency pervasive + 别无选择"。
- **Seam view 同时是 explore 新语言的工具**。换到 Go / Rust / OCaml 时，先找五种 seam 就可以判断"测起来几斤几两"。
- **本章与后续 21 章的咬合**。ch25 的 24 个 dependency-breaking 技法 → 本质上是"在某类 seam 上启用替代行为"。每条 ch25 技法都标 the primary seam type 它用到。

## 二、核心 Takeaways

### Takeaway 1: Seam = place without edit

- **是什么**: 一个允许替换行为的 source location, 不用直接编辑它。
- **为什么重要**: 把 "拆依赖" 从源码级设计约束还原为 *位置判断* — 看到 call, 立刻判断 "这处能换成 fake 吗"。
- **解决什么问题**: 让"不可测"的问题变成"seam 在哪"的问题 — 几何可解。
- **适用场景**: 接任意 legacy code 第一步: 扫一遍 call graph + 标 seam 类型。

### Takeaway 2: Every seam has an enabling point

- **是什么**: switch 实际所在 = enabling point; 永远是 seam 外的某个位置 (build script / classpath / 构造代码 / preprocessor define)。
- **为什么重要**: 没 enabling point 就不算 seam — 那是死代码 ("no way to change without modifying the method")。
- **解决什么问题**: 区分"看似 seam 但实际无 enabling point" 的 false seam — 例如 `new FormulaCell(...).Recalculate()` 一行内创建就地调, 没 seam。
- **适用场景**: refactor 时 — 找不到 enabling point = 该类需要先 separation 才能 sensing。

### Takeaway 3: C 预处理器给了一个 lint 抵不过但 seam 极强的工具

- **是什么**: `#define TESTING` 启用, `#include "localdefs.h"` 改写 `db_update` 为 `last_item = ...`, production 文件不需修改 1 行; test 拿 `last_item` 断言。
- **为什么重要**: Java 等没预处理器但有更干净的替代 (classpath / mock framework); C 借用 preprocessor seam 是 Feathers 明确认可 — "I'm actually glad that C and C++ have a preprocessor"。
- **解决什么问题**: vendor 二进制 / 静态库没法动 → preprocessing seam 是唯一低代价切入。
- **适用场景**: 嵌入式 / system daemon / kernel 的 vendor 函数假替 (e. g. `memcpy`/`kmalloc`/`printf` family)。

### Takeaway 4: Link seam 在 static linking 工程里被严重低估

- **是什么**: build script 在 `gcc -lgraphics_test` vs `gcc -lgraphics_real` 之间换库。Call site 不动。
- **为什么重要**: 大型 C++ / static-link 工程 link 时间长到 5–10 分钟 → 切换 library 只需 1 分钟, link 时延可忽略。
- **解决什么问题**: CAD / 游戏引擎 / 浏览器等"几乎整个代码库都是 lib calls" → 不要动 call site, lib level seam 是只剩的路。
- **适用场景**: 服务于 *"不是 OO 也能 seam"* 的普适证据 — C / Fortran / Rust 同样适用。

### Takeaway 5: Object seam 几乎永远首选 (在 OO 语言里)

- **是什么**: 给一个方法声明 `virtual`, 构造点选真 / fake 实现, SUT 看到的是基类引用。
- **为什么重要**: explicit / 可读 / IDE 可重构 / 不依赖 build script 状态, 是最稳的最显式 seam。
- **解决什么问题**: 让 enabling point = 构造点 (代码可见)。
- **适用场景**: 任何 OO 重构 (即使 OO 也首选 — 隐式 seam 留作 fallback)。

### Takeaway 6: 静态方法转 instance 方法 = 隐性 object seam

- **是什么**: 把 `static void Recalculate(Cell cell)` 改 `protected void Recalculate(Cell cell)`; 现在 subclass 可以 override。
- **为什么重要**: 给"看起来 seam 死了"的位置重新找 enabling point — 比 link/preprocessor 都干净。
- **解决什么问题**: 没拆 enable point 的旧 static method → 一行修改恢复 seam。
- **适用场景**: 重构静态 utility 类, 准备测试时。

### Takeaway 7: 在新语言写代码 = 先找 5 种 seam

- **是什么**: Go / Rust / Zig / Nim 等评估测试能力, 第一步看"我至少有 5 种 seam 可用吗"。
- **为什么重要**: 把"这语言难测"翻译成"seam 数量 + enabling point 可达性"。
- **解决什么问题**: 团队新语言选型 disc 不靠 benchmark, 靠 seam 数量。
- **适用场景**: tech radar / engineering memo / 教学语境。

### Takeaway 8: ch4 与 ch25 的对应 (索引学)

- **是什么**: ch25 的 24 个 dependency-breaking 技法 = 24 种"在某 seam 启用替代行为"的具体模板。
- **为什么重要**: seam view 不只是 lens, 而是 ch25 的 *索引系统*; 看 ch25 时, 问自己 "这条技法的 seam type 是什么"。
- **解决什么问题**: ch25 的参考目录看起来吓人 — 一旦按 seam 分类读, 24 条变 5 类, 可控。
- **适用场景**: 团队把 ch25 24 条按 seam type 聚类 (object / link / preprocessor / runtime-parameter / class-inheritance) 当 cheat sheet。

## 三、工程实践视角

> 锁定领域: **Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **Kernel seam 的分类**:
  - **Object seam** = `struct file_operations` (vfs 用 vtable 间接调用)。 各子系统 (ext4, btrfs, xfs) 自己实现 file ops, test 装 fake。
  - **Link seam** = `EXPORT_SYMBOL_GPL` + 改 build 链接到 stub 函数 (5.10+ 内核 KUnit 已用)。
  - **Preprocessor seam** = `#ifdef CONFIG_FOO` 在 . c 里换实现 (经典, 但 LTP/run一次 build 验成本高)。
  - **Runtime-parameter seam** = `EXPORT_SYMBOL_PARAM` 的 `__read_mostly` (e. g. `tcp_wmem`)。
- **db_update 案例在 kernel 版本 = test DRBG indirection**。内核 crypto 子系统把 `get_random_bytes` 的 indirection 抽出来, 测试桩用 deterministic DRBG。这是 preprocessing+function-pointer 的混合 seam — 也称 "function-pointer seam"。
- **`struct file` 抽象层 = object seam 的极致**。Linux 整个 VFS 不依赖任何具体 FS — 因为 `struct file_operations` 是 seam, FS 是 enabling point。
- **linker script (`.lds`) 提供 link seam**。Linux 各节区 (`__init`, `__exit`, `__ksymtab`) 都是 link-time seam; 嵌入式用 LD script 删节区模拟"测不到 extern 函数"的反例。
- **`vmlinux.lds.h` 中的 `__crc_*` 是 link seam 的二阶结果** — 把 function 的 checksum 嵌入 binary, 跨模块 test 时可以验证。

#### Linux 系统 — Seam 类型与示例表

| Seam 类型              | Linux 示例                                | Enabling point 位置                  |
| ---------------------- | ----------------------------------------- | ------------------------------------ |
| Preprocessing         | `#ifdef CONFIG_X86_64`, `IS_ENABLED()`    | `.config`, build flags               |
| Link                   | `EXPORT_SYMBOL` + 替换 `.o`               | kernel build / `LD` flags            |
| Object (vtable)        | `file_operations`, `datasource_ops`       | `register_filesystem()` 构造点      |
| Static→Instance        | `e1000_clean_tx_irq` (驱动)              | subclass override for test           |
| Runtime-parameter      | `__write_once` 变量                        | sysfs/procfs write                   |
| Function pointer       | `crypto_send_data` indirection, fake DRBG  | `crypto_alloc_*` 装回                |
| Class-inheritance class| (C 无此机制, 但 `-fno-plt` 是 link-time 类似) | `-fno-plt` GCC flag                  |

### 3.2 机器人软件视角

- **ROS2 的 object seam = `Publisher<T>` / `Subscription<T>`**。`topic_callback` 走 vtable-like 虚分派; Node 拿到的是 publisher 抽象, DDS 通过 factory method 创建。
- **Preprocessing seam = `ros2_control` 的 compile-time `hardware_interface` 选择**。 Build 时 `-DURDF_MODEL=...` 决定 mock 还是真 hardware。test 用 fake hardware_interface 替代。
- **`ros2_control` 的 mock hardware = link seam**。 同一 source 可与 `mock_components/GenericSystem` 或 `mock_components/Sensor` 或真实 hardware_interface 链接, link script 切。
- **Nav2 BT node 的 seam = XML 树**。BehaviorTree. CPP 用 XML 注册 nodes, test 时换 stub nodes 不动 . cpp。
- **TF2 的 buffer = object seam 极致**。`tf2_ros::Buffer` 是 `tf2::BufferCore` 子类化接口, 不同 backend (rosbag 录制回放 / RViz 可视化) 都是 enabling point。
- **action server / client = runtime-parameter seam**。 目标 topic / server name 可在 launch 时换; test 用 launch-time 替换。

#### 机器人软件 — Seam 实施表

| ROS2 组件                  | 主要 seam                         | test 时常用替代                     |
| -------------------------- | --------------------------------- | ----------------------------------- |
| `Node` 抽象                | object seam (publisher 接口)       | mock publisher, in-process DDS stub  |
| `MoveBase` (ROS1) / `BTNavigator` (ROS2) | object seam (plugin lib)           | 替换 BT nodes via XML                |
| hardware interface         | preprocessing seam + link seam     | `mock_components/GenericSystem`     |
| `tf2::BufferCore`          | object seam (subclass)             | `BufferMockRecorder` for playback     |
| `image_transport::Publisher` | object seam                       | `image_transport::Publisher` test stub |
| `tf_static_broadcaster`   | link seam (`tf_static_publisher`)  | bag file 录制 / 静态 URDF           |
| BT engine                  | object seam (BT node factory)     | BehaviorTree.CPP 单元测试场景       |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                   | 高级工程师                                                   |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 看不可测代码         | "它没结构, 没法测"                                           | "我找找 seam — call site, building block, classpath"        |
| 选 seam              | 默认 link seam (用 stub 库)                                   | 视 seam 显性程度选; object > preprocessing > link          |
| 看 static method     | 跳过; "static 不能 override"                                 | "把 static 转 instance + protected, 立刻 seam"               |
| 评估新语言           | "benchmark 跑多少"                                           | "给我 5 种 seam; 优劣逐条"                                   |
| 组织 ch25 24 条      | "太长了"                                                    | "按 seam 类型聚类, 5 类即可"                                 |
| 对预处理 seam        | "丑陋, 勿用"                                                | "vendor 二进制唯一路径, 用得"                                |
| 看到 enabling point 在 build script | 觉得 build script "隐蔽"                          | 把它外显到环境变量 / kconfig, 让 prod vs test 一眼可分       |

> **关键差异**: 高级工程师把"怎么测"还原为"哪些 seam + 哪个 enabling point"; 初级工程师把"怎么测"还原为 "改源码"。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

仍然极重要。原因:
- AI 写 mock 太容易 → 让"不动 place 改行为"的 seam 价值被淹没 — 但其实 seam 永远是 *位置判断*, 比 mock 高一层。
- AI 倾向一次性改 source 引入 DI 框架 (seam 的升级版) — 这恰好删了 seam view 的 *位置信息*, 给后续维护者看不出 seam 形状。
- 新的代码生成 (GPT 写 GraphQL backend) 让"原本是 handler 写死的 seam 现在直接绑定到 schema", 失去 explicit seam — 失去 = 失去"未来能不能改"的诊断能力。

### 4.2 AI 已经能做的 (具体到 ch4 主题)

- **识别 call graph 中的 seam point** — 给一个 source tree, AI 标出哪一行是 virtual call / 哪一行是 function-pointer / 哪一行是 preprocessor switch, 准确率高 (基于 AST 分析 + 模式匹配)。
- **生成 seam test template** — 知道是 object seam 就给 mock subclass; 知道是 link seam 就给 stub . o / . a template。
- **跨语言 seam 字典** — 给语言 X, AI 列 "5 种 seam + enabling point 位置 + recommended 测试框架"。

### 4.3 AI 不能替代的 (具体到 ch4 主题)

- **decide seam 优先级** — 何时用 object vs preprocessor vs link 是 architecture decision, AI 给法都"合理"但不一定合团队习惯。
- **kernel subsystem 这种 4 种 seam 混用的工程权衡** — AI 看 AST / cross-ref 做不到老 maintainer 的 tradeoff 经验。
- **enable-point 看不见的场景** — 有些 seam 藏在宏 / link-script / auto-gen 里, AI 静态分析不会标出来, 必须人 review 生成代码。

### 4.4 AI 经常写错的地方

| 错误模式                                       | 例子                                                                 | 后果 |
| ---------------------------------------------- | -------------------------------------------------------------------- | ---- |
| **seam 当作 mock framework 的同义词**            | AI 给出 "把 seam 改成 mock, 加 mockito"                              | seam 是 place 概念, mock 是工具; 混淆 → seam view 丢失 |
| **每处 call 都引入 interface** (反 seam)          | AI 看到 static method 就自动 `extract interface`                     | enabling point 增加了, 没 seam 实质变化, 反而加耦合 |
| **preprocess seam 漏 enable-point**            | `#define FOO(x) ...` 加在头, 但 production build 也 include 进去     | production 行为偷偷改了; 测试不再有 isolation |
| **link seam 写到不在 enable-point 的位置**       | stub .a 写到 `/usr/local/lib`, 但 `LD_LIBRARY_PATH` 没指            | 链接不切, 测试用真函数 |
| **seam 判断错位**                              | AI 说 "call `cell.Recalculate()` 是 seam"                          | 但 cell 是 `new FormulaCell()` 一行内创建, 实际不是 seam |
| **seam-enable 引了一处 source 改动**             | AI 给 stub 加 virtual method 时改了 production class                | 反了 — "不动 place" 是 seam 的关键不变式 |
| **class-inheritance 滥用**                       | AI 把所有 private static 都转 protected + virtual                   | 性能与可见性都下降; 边界处理也坏了 |
| **seam 配 mock framework 用错**                  | AI 给 link seam 配 mock, 反之 preprocess 配 stub lib               | 能跑, 但 enable-point 错; 切换时无法预测 |

### 4.5 子段: AI 辅助遗留代码理解 — seam 视角专项

- **AI 帮我画 seam diagram** — 给一段不可测 source, AI 列出 call + 可能的 seam 类型 + enabling point 位置。**风险**: AI 标错 enabling point (e. g. 错把 classpath 标成 preprocessor flag)。人工 review 时第一个看的就是 enable-point 是否真存在。
- **AI 助迁移 seam 到新语言** — 用 Rust 重写 C 模块时, AI 给"每个 C seam 在 Rust 怎么对应"。
- **AI 不会自动找隐式 seam** — C++ 的 `std::bind` / 函数指针传递 / macro 展开 里的 seam AI 不擅长识别。
- **AI 默认加 mock framework** — 在团队已经有 link-seam 习惯的代码中, AI 看到 dependency 就 inject + mock — 实际更适合用 stub lib。

### 4.6 工程师必须保留的核心能力

- **区分真 seam 和伪装 seam** — 不是所有 call 都是 seam。 必须 visual inspect: enabling point 在哪 / 能否在不编辑 call site 的前提下换行为。
- **seam 选择 trade-off** — object vs preprocess vs link 的工程取舍。
- **enable-point 的可见性** — 把 build script 隐式的 enabling point 提到 env var / config file。
- **在新语言看 5 种 seam** — 这是评估语言"测试友好度"的最快方法。
- **审 seam view 的 LLM 输出** — AI 标 "seam" 时要 verify enabling point。

## 五、实践行动项

> 本章是抽象章, 行动项聚焦 **(a) 用 C 复刻 ch4 的预处理 seam** + **(b) C++ 的 object seam** + **(c) link seam 的 stub . a 实证** + **(d) seam 类型识别器**。

### A1 — Preprocessing seam (C 版 db_update): #define TESTING 启用 macro 改写

```bash
mkdir -p /tmp/ch04-seam && cd /tmp/ch04-seam

# production 源 — 不动它 1 行, 但测试时通过 -DTESTING 启用 localdefs.h
cat > account_update.c <<'EOF'
#include <stdio.h>
#include "localdefs.h"

struct DFHLItem { int id; };
struct DHLSRecord {
    struct DFHLItem *item;
    struct DFHLItem *backup_item;
    int dateStamped;
    int quantity;
};
extern int db_update(int account_no, struct DFHLItem *item);

void account_update(int account_no, struct DHLSRecord *record, int activated) {
    if (activated) {
        if (record->dateStamped && record->quantity > 100) {
            db_update(account_no, record->item);
        } else {
            db_update(account_no, record->backup_item);
        }
    }
    db_update(42, record->item);   /* MASTER_ACCOUNT */
}
EOF

# production 默认: db_update 是真 db 调 (这里用 print 模拟)
cat > db_real.c <<'EOF'
#include <stdio.h>
struct DFHLItem;
int db_update(int acct, struct DFHLItem *item) {
    (void)item;
    printf("DB: write account %d\n", acct);
    return 0;
}
EOF

# 默认 (production) 编译:
cc -std=c17 -Wall -Wextra -o prod_audit account_update.c db_real.c
echo "=== production build, 期望看到 3 条 DB: write ==="
./prod_audit < /dev/null 2>&1 | head -5 || true
# 真生产是写 db, prod_audit 不接受 stdin, 我们用 fake driver:
cat > driver.c <<'EOF'
#include <stdio.h>
struct DFHLItem { int id; };
int main(void) {
    struct DFHLItem i1 = {1}, i2 = {2};
    /* 跳过 - account_update 需要 DHLSRecord, 用 simplified 直接驱 db_update */
    extern int db_update(int, struct DFHLItem *);
    db_update(1, &i1);
    db_update(42, &i1);
    db_update(42, &i2);
    return 0;
}
EOF
cc -std=c17 -Wall -Wextra -o prod_audit db_real.c driver.c
echo "production rc check:"; ./prod_audit
echo "---"

# test 版: localdefs.h 提供 TESTING 下 db_update 的 macro 定义 (记录为 static)
cat > localdefs.h <<'EOF'
struct DFHLItem;
#ifdef TESTING
extern struct DFHLItem *test_last_item;
extern int test_last_account_no;
#define db_update(account_no, item)              \
    do {                                         \
        test_last_item = (item);                 \
        test_last_account_no = (account_no);     \
    } while (0)
#endif
EOF

# test driver: 不需要 db_real.c, 因为 macro 已经吸收
cat > test_seam.c <<'EOF'
#include <stdio.h>
#include <string.h>
#include "localdefs.h"
struct DFHLItem;
int db_update(int account_no, struct DFHLItem *item);

/* 在 test build 里我们需要这些 extern 变量 */
#include "test_vars.c"
EOF
cat > test_vars.c <<'EOF'
#pragma once
struct DFHLItem *test_last_item = 0;
int test_last_account_no = -1;
EOF

# 在 test 编译时, 用 -DTESTING 开启 seam
cat > test_main.c <<'EOF'
#include <stdio.h>
#include <assert.h>
#include "localdefs.h"
struct DFHLItem { int id; };

/* 这些在 TESTING 下是 macro 化的"实际函数" */
struct DFHLItem *test_last_item;
int test_last_account_no;

void account_update(int, void*, int);   /* 简化 */

int main(void) {
    struct DFHLItem i1 = {1}, i2 = {2};
    /* 用 stub DHLSRecord 把持 — 为简化 account_update 不实际走, 我们直接调 db_update (已被 macro 化) */
    db_update(7, &i1);
    assert(test_last_account_no == 7);
    assert(test_last_item == &i1);
    db_update(42, &i2);
    assert(test_last_account_no == 42);
    assert(test_last_item == &i2);
    printf("preprocessing-seam PASS\n");
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -DTESTING -include localdefs.h -o test_seam account_update.c test_main.c
echo "=== test run ==="
./test_seam
echo "rc=$?"
```

**验收**:
- production 二进制 `prod_audit` 调用 `db_update` 走真函数 (打印 DB: write)。
- test 二进制 `test_seam` 通过 `-DTESTING -include localdefs.h` 走 macro 改写, `assert` 拿到 `test_last_account_no == 7`。**不动 `account_update.c` 1 行** (这是 seam 的关键) — 只有编译时 include 一个 header + define TESTING。

### A2 — Object seam (C++ 版 PostReceiveError): virtual + subclass override

```bash
mkdir -p /tmp/ch04-seam && cd /tmp/ch04-seam

cat > async_ssl.h <<'EOF'
#pragma once
class CAsyncSslRec {
public:
    bool Init();
    virtual void PostReceiveError(int type, int errorcode);   /* seam */
    virtual ~CAsyncSslRec() = default;
};
EOF

cat > async_ssl.cpp <<'EOF'
#include "async_ssl.h"
#include <cstdio>

void CAsyncSslRec::PostReceiveError(int type, int errorcode) {
    /* production: 调用什么 global */
    std::printf("[prod] PostReceiveError type=%d code=%d\n", type, errorcode);
}
bool CAsyncSslRec::Init() {
    /* 简化: 不做 FreeLibrary, 直接走到 PostReceiveError */
    PostReceiveError(1, 0xDEAD);
    return true;
}
EOF

cat > test_seam_obj.cpp <<'EOF'
#include "async_ssl.h"
#include <cassert>
#include <cstdio>

struct TestingAsyncSslRec : CAsyncSslRec {
    int got_type = -1, got_code = -1;
    int n_calls = 0;
    void PostReceiveError(int type, int code) override {
        got_type = type; got_code = code; ++n_calls;
    }
};
int main() {
    TestingAsyncSslRec rec;       /* enabling point: 用哪个子类 */
    rec.Init();                   /* 不动 Init 1 行 */
    assert(rec.n_calls == 1);
    assert(rec.got_type == 1 && rec.got_code == 0xDEAD);
    std::printf("object-seam PASS\n");
    return 0;
}
EOF

c++ -std=c++17 -Wall -Wextra -o test_seam_obj async_ssl.cpp test_seam_obj.cpp
./test_seam_obj
echo "rc=$?"
```

**验收**:
- `TestingAsyncSslRec::PostReceiveError` 被调 1 次, 测试通过。
- `async_ssl.cpp` 的 `Init()` 没改 — 子类 override = enabling point。

### A3 — Link seam (C 版): stub library + ldflags 切换

```bash
mkdir -p /tmp/ch04-seam && cd /tmp/ch04-seam

# production 库 (libgraphics_real.a)
mkdir real && cd real
cat > graphics.c <<'EOF'
#include <stdio.h>
void drawText(int x, int y, const char *t, int n) {
    printf("[real] drawText(%d,%d,%.*s,%d)\n", x, y, n, t, n);
}
void drawLine(int x1, int y1, int x2, int y2) {
    printf("[real] drawLine(%d,%d,%d,%d)\n", x1, y1, x2, y2);
}
int getStatus(void) { return 0; }
EOF
cc -std=c17 -c graphics.c && ar rcs libgraphics_real.a graphics.o
cd ..

# stub 库 (libgraphics_test.a)
mkdir test && cd test
cat > graphics.c <<'EOF'
#include <stdio.h>

/* stub: 不做任何事, 但仍可让 caller link */
void drawText(int x, int y, const char *t, int n) {
    printf("[stub] drawText called\n");
}
void drawLine(int x1, int y1, int x2, int y2) {
    printf("[stub] drawLine called\n");
}
int getStatus(void) { return 0; }
EOF
cc -std=c17 -c graphics.c && ar rcs libgraphics_test.a graphics.o
cd ..

# caller (CrossPlaneFigure::rerender 简化版)
cat > caller.c <<'EOF'
#include <stdio.h>
extern void drawText(int, int, const char *, int);
extern void drawLine(int, int, int, int);

void rerender(void) {
    drawText(10, 10, "label", 5);
    drawLine(10, 10, 50, 10);
    drawLine(10, 10, 10, 20);
}
EOF
cc -std=c17 -c caller.c

# production 链接
cc -std=c17 -o use_real real/libgraphics_real.a caller.o
echo "=== production link — expect [real] drawXXX ==="
./use_real

# test 链接 (enabling point: LD 选择不同 .a)
cc -std=c17 -o use_test test/libgraphics_test.a caller.o
echo "=== test link — expect [stub] drawXXX ==="
./use_test
echo "rc=$?"
```

**验收**:
- `use_real` 输出 `[real] drawText` / `[real] drawLine` — production 行为。
- `use_test` 输出 `[stub] drawText` / `[stub] drawLine` — test stub 行为。
- **enabling point = lib file 路径**, call site (`rerender`) 完全不动。

### A4 — Seam 类型识别器: 给一段不可测描述, 标出可行 seam

```bash
mkdir -p /tmp/ch04-seam && cd /tmp/ch04-seam

cat > seamfind.py <<'PY'
#!/usr/bin/env python3
"""seamfind.py — 给一段不可测描述, 列出可行 seam + enabling point.
规则: 基于关键字匹配. 真分析需 AST / cross-ref.
"""
import re, sys

RULES = [
    # (regex, seam_type, enabling_point_hint)
    (r"\bvirtual\b|method call on .*|interface\b",
        "object seam",   "构造点 (.c'tor / factory / DI 注入)"),
    (r"\bextern \w+\b|global function\b",
        "link seam",    "build script (ldflags / classpath)"),
    (r"#define|#ifdef|macro\b",
        "preprocessing seam", "compile flag (-D / -U)"),
    (r"\bstatic\b.*\b(method|void|function|utility|recalculate|helper)\b|utility\.h\b",
        "static-to-instance seam", "改 static 为 protected, 然后 subclass override"),
    (r"function pointer|fp\b|indirection\b",
        "function-pointer seam", "运行时替身 (init 时赋值)"),
]

desc = sys.stdin.read() if not sys.argv[1:] else " ".join(sys.argv[1:])
print(f"--- input ---\n{desc.strip()[:300]}\n--- seams ---")
hits = set()
for rx, seam, enabler in RULES:
    if re.search(rx, desc, re.I):
        hits.add((seam, enabler))
for seam, enabler in hits:
    print(f"  {seam:30s}  enabling point: {enabler}")
if not hits:
    print("  (none matched — re-describe with keywords)")
PY
chmod +x seamfind.py

echo "=== case 1: virtual call ==="
./seamfind.py "calls virtual method on interface"
echo
echo "=== case 2: global C function in production ==="
./seamfind.py "calls extern int db_update; globally"
echo
echo "=== case 3: macro replacing call ==="
./seamfind.py "#define db_update(x,y) { lastItem=y; lastX=x; }"
echo
echo "=== case 4: static utility method, can't override ==="
./seamfind.py "static void recalculate(Cell c) called from class"
```

**验收**: 4 个 case 各自命中至少一条 seam + 给 enabling point。证明对不可测代码描一段能诊断 seam。

## 六、值得深入思考的问题

### Q1: Seam 类型 5 个是不是真的够了?

Feathers 列 4 个 + 1 隐式。**关键问题**: 这是工程枚举还是穷举性分类? 例如 runner parameter 的 seam (启停时换 mock) 完全没列 — 是不是漏了? 而 mock framework 包装的 seam (Mockito / gMock) 是否算独立 seam? 

### Q2: Object seam 完美的代价 — OO-only

Object seam 是 OO 语言的礼物, 但 C / Fortran / Rust 没有完整 OO 继承机制。**关键问题**: 非 OO 语言一旦项目规模越过某个阈值, 是否必然退化成 link-seam-heavy? link seam 的 build-script enable-point 不易维护, 项目能否承受?

### Q3: 在大规模代码里 enable-point 的可发现性是不是 mock framework 取胜的原因?

OO mock framework 给出 *显式* `mock(Foo::bar)` + enabling-point 在代码内。**关键问题**: seam view 的"enable-point 必有"是不是 mock framework 才是真正的工业体现? seam view 还是 mock framework 在未来哪种会赢?

### Q4: AI 推动 seam 隐式化是否会反过来损失 seam view 的诊断价值?

AI 给代码自动加 DI 注入 + @Mock + middleware 替代 manual seam — 表面更干净, 实际每个 seam 形状都被 framework 隐藏。**关键问题**: 未来代码里我们还能像 ch4 那样看出 5 种 seam 形状吗? 如果不能, 团队就看不到 enabling point, 就无法安全拆依赖。

### Q5: Seam 和 Refactoring 的内在关系?

Fowler 的 Refactoring (Addison-Wesley 1999) 没提 seam 概念, Feathers 是把它应用到 testability 才提。**关键问题**: seam 是 refactoring 的扩展, 还是更底层的 framework? refactoring 是否要分"seam-building" 与 "behavior-preserving" 两类才完整?

---

*下一章预告*: **Chapter 5 — Tools** — 把 ch4 的 seam 概念落到 *具体工具箱*: Automated Refactoring Tools (Java 重构 / refactoring browser) + Mock Objects + Unit-Testing Harnesses (xUnit / JUnit / CppUnitLite / NUnit) + General Test Harnesses (FIT / Fitnesse)。ch5 是把 ch4 的抽象映射到工程师实际开箱即用的工具组合。

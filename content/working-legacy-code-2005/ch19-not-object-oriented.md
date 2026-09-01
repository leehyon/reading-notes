# Chapter 19 — My Project Is Not Object-Oriented. How Do I Make Safe Changes?

> **PDF**: p.253-266（14 页）
> **定位**: Part II 实战章 — ch3 的 sensing/separation 在 non-OO 项目里如何落地。本章是 ch18 的延伸(ch18 讲 test layout, ch19 讲 non-OO 项目里 *test 不存在* 时的拆依赖技法)。关键命题:**non-OO 项目里的"behavior site"** — 一段代码逻辑出现的位置, 与 OO 里 method 的边界不一样;**全局变量 + 记录**(global + record)是非 OO 项目的 *fake 替身*。

## 〇、第一性原理思考

**这章做了什么**: 在 C / COBOL / Fortran 这种没有 class 边界的 procedural 项目里, 演示 link seam (fake lib) + preprocessing seam (`#define TESTING` / `#ifdef`) + file inclusion seam (`#include "testscanner.tst"`) + function pointer struct 四种破依赖路径, 把 `scan_packets` 这种调 vendor lib 的 hard case 变成可测。

**为什么这样拆**: Feathers 用 "non-OO 不等于不能测, 只是 seam 较少" 这个命题, 把"测试做不了"的借口换成"用 link/preprocess seam 测得了"的具体工具 — 并用 `calculate_loan_interest` 这个 db_retrieve / db_update 塞在同一函数的例子, 把 hard 函数的破解路径落到代码层。

**最值钱的洞见**: 即便 100 个 procedural functions 没有显式 seam, 把 extern 全局函数逐个搬进 class (从 `ksr_notify` → `ResultNotifier::ksr_notify`, 再 Parameterize Constructor)就是 incremental 的 OO 迁移, 实际上不需要大爆炸重写 — `class program` 包一层就是测试 wedge。

## 一、章节概述

- **non-OO 不等于"不能做安全修改"**。Feathers 的开场命题: 任何语言都可以做安全修改, 只是 OO 让一些事情更容易。重点是 *在 procedural 语言里拆依赖的难度更高*。
- **C/COBOL/FORTRAN/Pascal/BASIC** 是 procedural 主流; 它们比 OO 难测试, 因为 *没有 class 边界* (没有封装/继承/动态分发)→ seam 较少。
- **测试 dilemma**: procedural 项目里 "think really hard, patch, hope" 是常见做法, 但这等于 Edit and Pray (ch1)。**替代路径**是引入大量 link seam + preprocessing seam, 在 language boundary 上建立 *可测试 islands*。
- **Easy case vs Hard case**: Easy = function 只依赖可设的 global (`jiffies` 可赋值 + struct field 可读写), 直接构造 instance 测。Hard = function 调用 hard-side-effect (vendor lib / I/O)。
- **Hard case 三种破解路径**:
  1. **Link seam** (ch4) — 用 *fakes library* 在 link 时替身; 缺点是每个 executable 一种 fakes。
  2. **Preprocessing seam** (ch4) — C 的 `#define TESTING` 让函数体内调用被替换, 文件末 `#include "testscanner.tst"` 让 test 与 prod 共存。这是 *C 独有的 seam*。
  3. **File inclusion seam** — `#include "scannertestdefs.h"` + 末尾 `#include "testscanner.tst"`, 让生产代码 *看起来* 没变, 实际测试逻辑在最末。
- **Adding new behavior**: procedural 项目里 *新写函数* 比 *修改老函数* 更安全, 因为新函数可写测试。Feathers 把 send_command 拆为 form_command (纯逻辑) + send_command (thin wrapper) — 这是 *Extract Function* 的 procedural 版本。
- **`calculate_loan_interest` 案例**: db_retrieve / db_update 等"很重的 dependency"塞在同一函数里 — 这种函数 *没法直接测试*。C 的解法 = **function pointer struct**: 把 db_retrieve / db_update 作为 struct 字段, 生产时填真函数, 测试时填 fake。
- **Taking Advantage of OO Migration**: C → C++ / COBOL → OO COBOL / Fortran → OO Fortran / Visual Basic → OO VB。把 non-OO 函数包进 class 是 incremental 的: 从 extern `ksr_notify` → `ResultNotifier::ksr_notify` 包装方法(Preserve Signatures)→ `Scanner` class with Parameterize Constructor。
- **"It's All Object Oriented"**: Feathers 的硬命题 — 即使是 100 个 procedural functions, 把它们都搬进一个 `class program` 也不改变行为。这是 *Encapsulate Global References* (ch25 p.339) 的极端版, 作为 *测试 wedge*。
- **真正可 OO 化的语言 = 渐进过程**: VB 最近才"fully OO"、COBOL/Fortran 有 OO 扩展。**关键建议**: 如果你的语言有 OO 后继,**朝它迁移**, 因为 object seams (ch4 p.40) 是测试 / 设计的双重 gain。

## 三、核心 Takeaways

### Takeaway 1: non-OO = seam 较少, 但不是没有 seam

- **是什么**: C 等 procedural 语言没有 class 边界, 但仍有 *preprocessing seam* (`#define`)、*link seam* (fake lib)、*runtime parameter seam* (function pointer struct)。这三条足以建立可测试 islands。
- **为什么重要**: 很多 procedural 项目默认"没法测"而放弃; 实际 C 项目 *完全可以测*, 只是测法与 OO 不同。
- **解决什么问题**: 让 C/COBOL 工程师摆脱"测试做不了"的借口, 转而"用 link/preprocess seam"。
- **适用场景**: 任何 C / COBOL / Fortran 项目; 嵌入式 firmware; legacy 系统编程。

### Takeaway 2: Hard Case = 函数调"调一下就有副作用"的 hard-side-effect 函数

- **是什么**:`scan_packets` 调 `ksr_notify`, 后者写 notification 到第三方系统 — 这是 hard side effect, 因为测试不能让它真写。
- **为什么重要**: Hard case 不可"直接测" — 必须先 break the dependency。三种 seam 都能破。
- **解决什么问题**: 让"vendor lib / IO 函数"在 test 时被 fake。
- **适用场景**: 任何调 vendor lib / 系统调用 / hardware 的 C 函数。

### Takeaway 3: Link Seam = 给每个 executable 一套 fakes

- **是什么**: 做一个 `libksr_fakes.a`, 函数体是空(或 record); test 时链 fake lib 替代真 lib。`ksr_notify` 在 fake lib 中什么都不做。
- **为什么重要**: 简单粗暴 — 不动 source 任何一行, 只换 link 命令。
- **解决什么问题**: C 里 *没有* 动态 mock framework → fake lib 是工业标准。
- **局限**: 每个 test 一个 fake lib 配置; 函数行为靠 fake lib 中的"if 条件分支"控制, 繁琐。
- **适用场景**: 少量 hard 函数; 无需细回 fakes 的返回值。

### Takeaway 4: Preprocessing Seam = C 独有的 `#define` 替身

- **是什么**:`#ifdef TESTING` 包一段代码, 在 production build 不编, 在 testing build 替换 call 或加测试代码。
- **为什么重要**: C 独有 seam — OO/其他 procedural 语言没有。这是 *测试放在 source 末尾* 的工业版本。
- **解决什么问题**: 让 test 与 prod 共处一文件, 但互不可见(`#ifdef` 隔离)。
- **示例**:`#ifdef TESTING #define ksr_notify(code, packet) #endif` + 文件末 `#ifdef TESTING int main() { ... } #endif`。
- **适用场景**: C / C++ 项目; 嵌入式 firmware 测试。

### Takeaway 5: File Inclusion Seam = 把 test 藏在末尾 include

- **是什么**: production source 末尾 `#include "testscanner.tst"`。`testscanner.tst` 在 testing build 时被加, production build 时是空文件。
- **为什么重要**:`#include` 是 C 的 *文本合并* seam, 比 `#ifdef` 更干净 — production source 看起来 *没* test infra。
- **解决什么问题**: 让 navigation 维持 — review production 时不看到 test 噪音。
- **示例**: Feathers ch19 给出 `scan_packets` + 末尾 `#include "testscanner.tst"` 的真实例子。
- **适用场景**: C / C++ 复杂 file; legacy 工程。

### Takeaway 6: Adding New Behavior = 倾向"写新函数"而非"改老函数"

- **是什么**:`send_command` → 拆为 `form_command`(纯逻辑, 可测) + `send_command`(thin wrapper, 调 mart_key_send)。`calculate_loan_interest` → 把 `db_*` 调用抽到 function pointer struct。
- **为什么重要**: 新函数可写测试; 老函数不可测。procedural 项目里, 优先"用 Extract Function 把可测的逻辑剥出"。
- **解决什么问题**: 让*可测逻辑* 与 *副作用调用* 在不同函数, 后者测试时靠 fake seam 替身。
- **适用场景**: 任何 procedural 项目添加新代码时。

### Takeaway 7: C 的 function pointer struct = class 的前身

- **是什么**:`struct database { void (*retrieve)(...); void (*update)(...); }; extern struct database db;` — db 是 global, 但生产时填真函数, test 时填 fake。
- **为什么重要**: 这是 *struct-as-class* 的 procedural 替代。`db.retrieve(id)` 就是 OO message send, 只是语法不同。
- **解决什么问题**: 让 db_retrieve / db_update 在 test 时被 fake(无需 link seam 重编 lib)。
- **适用场景**: C 项目里 *多个函数共享一个 dependency* 的场景。

### Takeaway 8: Preserve Signatures + Encapsulate Global References = 把 C 函数变 C++ 方法的"安全路径"

- **是什么**: 把 `void ksr_notify(int, struct rnode_packet*)` 包成 `class ResultNotifier::ksr_notify`, Preserve Signatures (ch25 p.312) 保不变行为; 然后 Encapsulate Global References (ch25 p.339) 把 global 函数引用挪进 class。
- **为什么重要**: 这种 *包装* 不改任何调用语法, 只加一层间接。**安全到不需要测试**就能改。
- **解决什么问题**: 让 C 项目"渐进式"获得 OO seam, 而*不需要大规模重写*。
- **适用场景**: C → C++ 迁移; legacy 项目想引入 seam。

### Takeaway 9: 把整个 program 包进 `class program` 是 *mechanical*

- **是什么**:100 个 procedural functions → 全部塞进 `class program`;`int main() { program the_program; return the_program.main(ac, av); }`。**Feathers 的硬命题**: 这不改任何行为。
- **为什么重要**: 它让 *整个 program* 成为一个可 fake 的对象 — Encapsulate Global References 的极端版。
- **解决什么问题**: 让"程序"作为一个 unit 可被测试 (用 testing subclass override method)。
- **适用场景**: 极 legacy 的 C 项目; 无法拆成多 class 的 monolithic binary。

### Takeaway 10: Object seams 远优于 link/preprocessing seam

- **是什么**: Object seam 不仅是 *测试 seam*, 更是 *设计 seam*。link/preprocess seam 仅用于测试, 不改设计。
- **为什么重要**: 如果你的语言有 OO 扩展,**值得朝 OO 迁移**, 因为 object seam 长期收益更大。
- **解决什么问题**: 在 Sprint planning 上把"语言迁移"作为可投资项, 而不是被视为禁区。
- **适用场景**:5+ 年长存活的 C 项目; 有 OO 扩展的语言(COBOL/Fortran/VB)。

## 三、工程实践视角

> 锁定领域:**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **kernel C 是 non-OO 的极致**。Linux kernel 100% C, 几乎不用 OO, 但有完整测试网(KUnit + LTP + syzkaller)。**Feathers 论点验证 — seam 不必是 OO seam**。
- **kernel 的 link seam 实例**:`tools/testing/selftests/` 自带许多 fakes — 例如 `ksr_notify` 等同 kernel `netif_rx` / `printk` 等, test 时用 `selftests/net/lib/fake_*` 函数。
- **kernel 的 preprocessing seam 实例**:`#ifdef CONFIG_FOO_TEST` 在 source 内部加 self-test hook; 内核编译时 `CONFIG_FOO_TEST=y` 才编入。这是 ch19 Takeaway 4 的工业实例。
- **`__weak` symbol + alternative link** = kernel 的 *function pointer injection*。kernel 里许多 subsystem 提供 `__weak` 默认实现, test driver override 注入 fake。
- **`EXPORT_SYMBOL_GPL` + module replacement** = kernel 的 *module-level fake*。test module 提供同名 symbol, insmod 时覆盖真 symbol — 等同 ch19 Takeaway 3 的 fake lib。
- **kernel 没有 C++ 的 class 抽象**, 但 *struct + function pointer* 处处存在:`struct file_operations`, `struct i2c_algorithm`, `struct net_device_ops`。这正是 ch19 Takeaway 7 的 *function pointer struct* 在工业代码中的化身。
- **`struct seq_operations` 的单方法 fake** 是 kernel 测 *probe / debugfs* 的常用技法。Fake `seq_operations.start/stop/next/show` 替身, test 验 show 输出。

#### Linux 系统 — non-OO 测法映射表

| 技法                | kernel 实例                            | 等价 OO 概念           |
| ------------------- | ------------------------------------- | ---------------------- |
| Link seam           | kselftest fake library                  | Mock library (gmock)   |
| Preprocess seam     | `CONFIG_FOO_TEST`                      | `#ifdef` C 独有        |
| File inclusion      | `#include "selftest.h"`                | test fixture include   |
| `__weak` symbol     | driver load + override                  | subclass override       |
| `EXPORT_SYMBOL`     | module replace                          | dynamic linking        |
| `struct file_ops`   | filesystem driver / char device         | interface + impl        |
| `seq_operations`    | debugfs / proc fake                    | iterator stub          |
| KUnit `kunit_*`     | kernel 内 unit test                     | CppUnit                |

### 3.2 机器人软件视角

- **ROS 1 → ROS 2 的迁移** = ch19 Takeaway 8-10 的工业实例。ROS 1 是 Python/C++ mix 但 callback 不强制 class; ROS 2 强制 Node 是 class(`rclcpp::Node`)。这是 *非 OO → OO* 的迁移, 目的是让 test fixture 可注入。
- **ros2_control 的 `hardware_interface::SystemInterface`** = ch19 Takeaway 7 的 function pointer struct 在机器人中的化身。一个 robot 有多个 actuator, 每个 actuator 暴露 `read()` / `write()` function pointer; test 时 fake hardware_interface 把 function pointer 重定向。
- **micro-ROS (MCU)** = ch19 Takeaway 4 的 preprocess seam 实例。micro-ROS 在嵌入式端编译时常带 `MICRO_ROS_TESTS=ON` 标志, test code 通过 `#ifdef` 编译入。这是 *deployment size 紧* 的场景 — test 必须不污染 firmware。
- **MoveIt 2 的 planning pipeline** = ch19 Takeaway 7 的实例。`planning_pipeline::PlanningPipeline` 接受 `PlannerManager` plugin list (function pointer / vtable), test 时 fake `OMPL` / `CHOMP` plugin 让 deterministic output。
- **ros2 nav2 的 `bt_navigator`** = ch19 Takeaway 7 的 Behavior Tree plugin。BT XML 行为节点是 *late binding*(运行时选), test 时用 `MockClock` + `BehaviorTreeTest` 替身。这是 *function pointer injection* 在 ROS2 的实例。
- **rosbridge (ROS 1 + WebSocket)** 是 C++ class 包装 C 函数: 它把 `ros::Publisher` (C++) 转成 JSON over WebSocket。如果包装得当, WebSocket 层可单独 unit test (用 fake `ros::Publisher`)。

#### 机器人软件 — non-OO 测法映射表

| 技法                | 机器人实例                              | 等价工业标准     |
| ------------------- | -------------------------------------- | --------------- |
| Link seam           | micro-ROS test driver                   | Kselftest       |
| Preprocess seam     | `MICRO_ROS_TESTS=ON`                   | `CONFIG_*_TEST` |
| Function pointer    | `hardware_interface::SystemInterface`  | `file_operations` |
| Plugin injection    | BT navigator `Behavior` plugin         | dlopen plugin   |
| Class migration     | ROS 1 → ROS 2 Node                     | C → C++         |
| Time fake           | `MockClock` (Nav2)                      | `LD_PRELOAD`    |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                | 高级工程师                                                  |
| -------------------- | --------------------------------------------------------- | ----------------------------------------------------------- |
| 拿到 non-OO legacy   | "这没法测,只能靠 print + 试"                              | 先找 seam:preprocess / link / function pointer             |
| Hard side effect 函数 | 直接写 mock (但 C 没有 mock framework)                    | 用 fake lib 或 `#define` 替身                                |
| 加新功能            | 直接改老函数                                              | 倾向"写新函数 + Extract Function"                           |
| 看到 `db_retrieve`  | "这是 DB 库,不能动"                                       | "包 function pointer struct,test 时填 fake"                |
| 评估 C → C++ 迁移    | "代价太大,不做"                                          | "Encapsulate Global References + Preserve Signatures,逐步来" |
| 看到 100 function 的 `program` | "重构 OO 设计吧"                            | "先把所有 function 塞进 `class program`,mechanical,不改行为" |
| 对 #define TESTING   | "这污染 production"                                       | "它是 C 独有的 seam,只在 test 时展开,production 0 影响"      |

> **关键差异**: 高级工程师把 *seam 类型* 当语言资源(asset), 初级把 non-OO 当限制(limitation)。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

仍然重要。原因:
- **AI 写的 procedural code 越来越多**。LLM 在 Python/C++ 上偏向 OO, 但在嵌入式 / firmware / kernel / data engineering 中仍大量生成 procedural C / Go(Go 不是 OO, 是 procedural-like)。ch19 的技法仍是 *日常工具*。
- **AI 不知道 link seam 的局限** — 它推荐 "用 `dlopen` + function pointer" 但忘了 *每个 executable 一套 fakes* 的痛。
- **AI 推荐的 `#ifdef TESTING` 通常过度** — 它会无差别在所有函数加 `#ifdef`, production build 出意外行为。**正确的 `#ifdef` 是 surgical 的**, 只 wrap *hard-side-effect* 调用。
- **AI 不知道语言迁移成本** — C → C++ 看起来是 *compile flag* 改变, 实际牵涉到 *头文件包裹*、*name mangling*、*constructor 调用*、*lifecycle*。Feathers 的 Preserve Signatures (ch25 p.312) 流程是 *安全* 路径, AI 不知道。

### 4.2 AI 已经能做的(具体到 ch19 主题)

- **识别 hard-side-effect 调用** — `grep` vendor lib / IO 函数, 标出 *需要 fake* 的位置。准确率 ~80%。
- **推荐 fake lib 实现** — 给定 function signature, 生成 empty body + record-only body。准确率 ~90%。
- **推荐 function pointer struct 抽象** — 给定 N 个 db_* 调用, 推荐 `struct db_iface` 设计。准确率 ~75%。
- **生成 Preserve Signatures wrapper** — 把 C 函数包成 C++ class 方法, 签名不变。准确率 ~85%。

### 4.3 AI 不能替代的(具体到 ch19 主题)

- **判断 `#ifdef` 的 surgical 范围** — 哪些行 wrap, 哪些不 wrap。这是 design judgement。
- **判断 C → C++ migration 的可逆性** — C++ 的 RAII / 异常处理会改 C 的内存语义。AI 看不到这一层。
- **评估 fake lib 的 link 复杂度** — 每个 test 一个 fake lib, 工程上有 O(N) 配置文件。AI 不感知 deployment。
- **决定 Preserve Signatures vs 重构签名** — 哪个更值得长期投资。AI 看不到团队 5 年计划。
- **non-OO 项目的 seam 边界判定** — "这个 #ifdef 是 seam, 那个不是"。AI 经常 *过度识别* seam, 导致 production build 行为偏离。

### 4.4 AI 经常写错的地方

针对 ch19 non-OO / seam 主题:

| 错误模式                                       | 例子                                                                 | 后果 |
| ---------------------------------------------- | -------------------------------------------------------------------- | ---- |
| **过度 `#ifdef TESTING`**                       | AI 给所有函数加 `#ifdef TESTING` 包括纯计算函数                      | production build 行为差异(register alloc 不同),debug 时崩 |
| **`#ifdef` 替换调用但保留副作用**                | `#define db_retrieve(x) 0` 但内部仍 `db_update()`                    | test 跑过但 production 数据不一致 |
| **fake lib 函数体仍调真函数**                   | fake `ksr_notify` 里 `fprintf(stderr, ...)` + 调真 notify             | 真 notify 仍触发,测试失败 |
| **function pointer struct 漏 init**            | 生产代码 `db.retrieve = NULL;` 后续 deref,segfault                   | 启动崩溃 |
| **C++ Preserve Signatures 后改 struct layout**  | 把 `struct rnode_packet` 加字段,不更新 caller layout                  | ABI break,运行时崩 |
| **link seam 多个 test 用同一 fake**              | test1 要 fake `ksr_notify` 返回 1,test2 要返回 0,共用同一 fake       | 后跑的 test 被前一个污染 |
| **Encapsulate Global References 后忘 extern**  | C++ 类方法调 `::ksr_notify`,但 `extern "C"` 没声明                   | link 失败 |
| **`class program` 改造后 `the_program.main()` 没继承原 main 行为** | AI 直接 stub main,跳过 argc/argv 处理 | CLI 协议 break |
| **`#include "testscanner.tst"` production 仍包** | production build 没正确 `--no-include-test`                          | production binary 含 test main,启动崩溃 |
| **function pointer 失去 const-correctness**     | fake 函数 cast 掉 const,生产代码仍读 const                          | undefined behavior |

### 4.5 子段: AI 辅助遗留代码理解 — 在本主题专项

- **AI 帮你 audit hard-side-effect calls** — 给定 C source, 统计 `printf/fprintf/write/syscall/vfork` 等调用, 标"需要 fake"。
- **AI 推荐 `#ifdef` surgical 范围** — 只 wrap 函数 *出口*(return 前的 side-effect call), 不 wrap 计算。
- **AI 帮你写 fake lib** — `libfakes.a` 提供 stub 实现, AI 给定 function signature 生成。**风险**: fake lib 维护成本。
- **AI 推荐 Preserve Signatures wrapper** — 把 C 函数转 C++ 方法, 签名 0 改。**风险**: AI 偶尔改名 (parameter name), 编译器 warning 不报但 ABI 变。
- **AI 不知道 "mechanical migration"** — 它倾向 *建议大改* (整个 OO redesign), Feathers 的"先全塞进 class program"是 *最小动作*, AI 看不到。

### 4.6 工程师必须保留的核心能力

- **判断 seam 类型** — preprocess / link / function pointer / object seam, 根据 *测试 focus* 选。
- **写 surgical `#ifdef`** — 只 wrap hard-side-effect。
- **写 fake lib** — 每个 test 一份, fake body 简短。
- **C → C++ migration Preserve Signatures** — 包一层, 不改签名。
- **Encapsulate Global References** — 把 global 函数挪进 instance。
- **评估 migration cost** — 知道哪些 seam 给"测试 gain" vs "设计 gain"。
- **对 AI 给的 `#ifdef` 范围 review** — 排除"AI 过度 wrap"导致 production 行为漂移。

## 五、实践行动项

> 本章是 ch3 (sensing/separation) 的 non-OO 实例化。行动项聚焦在 (a) C 复刻 ch19 的三个 case (easy / hard / new behavior);(b) C → C++ migration 路径(用 g++ 演示 Preserve Signatures);(c) Python 演示 procedural 项目的 Extract Function + function pointer struct。

### A1 — C easy case: `set_writetime` 在 test harness 中可设 jiffies + buffer

```bash
mkdir -p /tmp/ch19-easy && cd /tmp/ch19-easy

cat > writetime.h <<'EOF'
/* ch19 "Easy Case" — set_writetime 复刻
 * Linux 内核 buffer_head 风格的 function: 输入 buffer + flag,输出更新 b_flushtime.
 * 测试时可设 jiffies + buffer_head 内容,断言 b_flushtime.
 */
#ifndef WRITETIME_H
#define WRITETIME_H

typedef struct {
    int dirty;          /* 1 = 需更新 flushtime */
    int b_flushtime;
} buffer_head_t;

typedef struct {
    int age_super;      /* dirty=true, flag=true 用 */
    int age_buffer;     /* dirty=true, flag=false 用 */
} bdf_prm_t;

extern unsigned long jiffies;        /* global — test 时可设 */
extern bdf_prm_t bdf_prm;

void set_writetime(buffer_head_t *buf, int flag);
#endif
EOF

cat > writetime.c <<'EOF'
#include "writetime.h"
void set_writetime(buffer_head_t *buf, int flag) {
    if (buf->dirty) {
        int newtime = (int)jiffies + (flag ? bdf_prm.age_super : bdf_prm.age_buffer);
        if (!buf->b_flushtime || buf->b_flushtime > newtime)
            buf->b_flushtime = newtime;
    } else {
        buf->b_flushtime = 0;
    }
}
EOF

cat > test_easy.c <<'EOF'
#include "writetime.h"
#include <assert.h>

/* test-only redefines — global 可写,这就是 easy case 的关键 */
unsigned long jiffies = 1000;
bdf_prm_t bdf_prm = { .age_super = 500, .age_buffer = 100 };

int main(void) {
    /* case 1: dirty + flag=1 (super), jiffies+age_super = 1500 */
    buffer_head_t b1 = { .dirty = 1, .b_flushtime = 0 };
    set_writetime(&b1, 1);
    assert(b1.b_flushtime == 1500);

    /* case 2: dirty + flag=0 (buffer), jiffies+age_buffer = 1100 */
    buffer_head_t b2 = { .dirty = 1, .b_flushtime = 9999 };
    set_writetime(&b2, 0);
    assert(b2.b_flushtime == 1100);

    /* case 3: not dirty → b_flushtime = 0 */
    buffer_head_t b3 = { .dirty = 0, .b_flushtime = 9999 };
    set_writetime(&b3, 1);
    assert(b3.b_flushtime == 0);

    /* case 4: dirty + existing flushtime already earlier → keep new */
    jiffies = 2000;
    buffer_head_t b4 = { .dirty = 1, .b_flushtime = 5000 };
    set_writetime(&b4, 0);  /* 2000 + 100 = 2100 < 5000 */
    assert(b4.b_flushtime == 2100);

    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_easy writetime.c test_easy.c && ./test_easy
echo "rc=$?"
```

**验收**: `rc=0` (4 个 case 全部 assert 通过 — easy case 在 test harness 中直接可设 global 即可测)

### A2 — C hard case with preprocess seam: `#define ksr_notify` 替换 + 末尾 test include

```bash
mkdir -p /tmp/ch19-hard && cd /tmp/ch19-hard

# 1) vendor "library" — ksrlib.h + ksr_notify 真实现(写 stderr 模拟"通知第三方")
cat > ksrlib.h <<'EOF'
#ifndef KSRLIB_H
#define KSRLIB_H
struct rnode_packet { int body; struct rnode_packet *next; };
#define INVALID_PORT 0x04

/* 真实现: 写 stderr — testing 时不希望它真发生 */
void ksr_notify(int scan_code, struct rnode_packet *packet);
int loc_scan(int body, int flag);
#endif
EOF

cat > ksrlib.c <<'EOF'
#include "ksrlib.h"
#include <stdio.h>
void ksr_notify(int scan_code, struct rnode_packet *packet) {
    fprintf(stderr, "[KSR_NOTIFY] code=%d packet=%p\n", scan_code, (void*)packet);
}
int loc_scan(int body, int flag) {
    /* 简单规则: flag=0 全 PASS, 否则 INVALID_PORT */
    if (flag == 0xFF) return INVALID_PORT;
    return 0;
}
EOF

# 2) scanner.c — production source, 末尾 #include test stub
#    当 TESTING 时, ksr_notify 被 #define 为空; test stub 提供 main()
cat > scanner.c <<'EOF'
#include "ksrlib.h"

/* ch19 Takeaway 4: preprocess seam */
#ifdef TESTING
#define ksr_notify(code, packet) ((void)0)
#endif

int scan_packets(struct rnode_packet *packet, int flag) {
    struct rnode_packet *cur = packet;
    int err = 0;
    while (cur) {
        int sr = loc_scan(cur->body, flag);
        if (sr & INVALID_PORT) {
            ksr_notify(sr, cur);
            err |= sr;
        }
        cur = cur->next;
    }
    return err;
}

/* ch19 Takeaway 5: file-inclusion seam — test 在末尾被包进来 */
#ifdef TESTING
#include <assert.h>
#include <stdio.h>
int main(void) {
    /* 单 packet, flag=0xFF → INVALID_PORT 应被触发 */
    struct rnode_packet p1 = { .body = 1, .next = NULL };
    int rc = scan_packets(&p1, 0xFF);
    assert(rc & INVALID_PORT);
    /* 多 packet 链, flag=0 → 无 invalid */
    struct rnode_packet p2 = { .body = 2, .next = NULL };
    struct rnode_packet p3 = { .body = 3, .next = &p2 };
    rc = scan_packets(&p3, 0);
    assert((rc & INVALID_PORT) == 0);
    printf("ch19 hard-case preprocess seam PASS\n");
    return 0;
}
#endif
EOF

# 3) build & test
echo "=== production build (no TESTING, ksr_notify 真调用) ==="
cc -std=c17 -Wall -Wextra -c -o ksrlib.o ksrlib.c
cc -std=c17 -Wall -Wextra -c -o scanner_no_test.o scanner.c
cc -std=c17 -Wall -Wextra -o scanner_prod ksrlib.o scanner_no_test.o
ls -la scanner_prod
echo "✅ production binary built without test main"

echo
echo "=== testing build (TESTING=1, #define ksr_notify 替换 + main from include) ==="
cc -std=c17 -Wall -Wextra -DTESTING=1 -o scanner_test ksrlib.c scanner.c 2>&1
./scanner_test
echo "rc=$?"

echo
echo "=== 验证: production binary 没有 test main(用 strings grep) ==="
strings scanner_prod | grep -c "ch19 hard-case" || true
test $(strings scanner_prod | grep -c "ch19 hard-case") -eq 0 && \
    echo "✅ production binary 不含 test 字符串 (preprocess seam 隔离成功)"

echo
echo "=== 验证: testing binary 包含 test main ==="
strings scanner_test | grep -c "ch19 hard-case"
```

**验收**:
- `scanner_prod` 是 production binary, 运行 OK 且 *不* 含 test main
- `scanner_test` 跑出 PASS (`ch19 hard-case preprocess seam PASS`)
- `strings` 验证 production 不含 test 字符串

### A3 — C function pointer struct: procedural db interface → fakeable

```bash
mkdir -p /tmp/ch19-fn-ptr && cd /tmp/ch19-fn-ptr

# 1) db interface 抽象为 function pointer struct
cat > db_iface.h <<'EOF'
#ifndef DB_IFACE_H
#define DB_IFACE_H

typedef struct { int id; } record_id_t;
typedef struct { int lender_id; int amount; double interest; } record_set_t;

typedef struct db_iface {
    void (*retrieve)(record_id_t id, record_set_t *out);
    void (*update)(record_id_t id, const record_set_t *rec);
    int  call_count_retrieve;
    int  call_count_update;
} db_iface_t;

/* production 用真 db; test 用 fake db */
extern db_iface_t *g_db;

void db_iface_init_default(db_iface_t *iface);   /* production setup */
void db_iface_init_fake(db_iface_t *iface);       /* test setup */
#endif
EOF

cat > db_iface.c <<'EOF'
#include "db_iface.h"
#include <stdio.h>
#include <string.h>

static db_iface_t default_iface;
db_iface_t *g_db = &default_iface;

static void prod_retrieve(record_id_t id, record_set_t *out) {
    g_db->call_count_retrieve++;
    /* 真实 DB 调用 */
    out->lender_id = 100;
    out->amount = 1000;
    out->interest = 0;
}
static void prod_update(record_id_t id, const record_set_t *rec) {
    g_db->call_count_update++;
    /* 真实 DB update */
}
void db_iface_init_default(db_iface_t *iface) {
    iface->retrieve = prod_retrieve;
    iface->update = prod_update;
    iface->call_count_retrieve = 0;
    iface->call_count_update = 0;
    db_iface_init_default(&default_iface);
}

/* fake 实现 — test 用 */
static void fake_retrieve(record_id_t id, record_set_t *out) {
    g_db->call_count_retrieve++;
    out->lender_id = 999;
    out->amount = id.id * 100;
    out->interest = 0.05;
}
static void fake_update(record_id_t id, const record_set_t *rec) {
    g_db->call_count_update++;
    /* fake: 啥也不做,但记录 call count */
}
void db_iface_init_fake(db_iface_t *iface) {
    iface->retrieve = fake_retrieve;
    iface->update = fake_update;
    iface->call_count_retrieve = 0;
    iface->call_count_update = 0;
    g_db = iface;
}
EOF

# 2) procedural 函数: 用 function pointer struct 调用 db
cat > loan.c <<'EOF'
#include "db_iface.h"

int calculate_loan_interest(record_id_t loan_id, int calc_type) {
    record_set_t loan;
    /* 通过 function pointer 调 db — production 真 / test fake */
    g_db->retrieve(loan_id, &loan);
    /* 计算 interest (简单 calc) */
    double rate = (calc_type == 1) ? 0.10 : 0.05;
    loan.interest = loan.amount * rate;
    /* update 回去 */
    record_set_t updated = loan;
    g_db->update(loan_id, &updated);
    return (int)loan.interest;
}
EOF

# 3) test: 用 fake db 验证 call count + 计算正确
cat > test_loan.c <<'EOF'
#include "db_iface.h"
#include <assert.h>
#include <stdio.h>

int main(void) {
    db_iface_t fake;
    db_iface_init_fake(&fake);

    int interest = calculate_loan_interest((record_id_t){42}, 1);
    /* fake_retrieve 设 amount = 42*100 = 4200; rate=0.10 → 420 */
    assert(interest == 420);
    assert(fake.call_count_retrieve == 1);
    assert(fake.call_count_update == 1);
    printf("ch19 function-pointer fake PASS (call_count_retrieve=%d, interest=%d)\n",
           fake.call_count_retrieve, interest);

    /* 复位, 测 calc_type=0 */
    fake.call_count_retrieve = fake.call_count_update = 0;
    interest = calculate_loan_interest((record_id_t){100}, 0);
    assert(interest == 500);   /* 100*100 * 0.05 = 500 */
    assert(fake.call_count_retrieve == 1);
    printf("PASS calc_type=0 (interest=%d)\n", interest);
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_loan db_iface.c loan.c test_loan.c && ./test_loan
echo "rc=$?"
```

**验收**: `rc=0`, 两次调用 fake db 后 interest 计算正确 (420, 500), call_count 都是 1。

### A4 — C → C++ migration 路径: Preserve Signatures + Encapsulate Global References

```bash
mkdir -p /tmp/ch19-migrate && cd /tmp/ch19-migrate

# 1) 原 C 函数
cat > ksrlib.h <<'EOF'
#ifndef KSRLIB_H
#define KSRLIB_H
struct rnode_packet { int body; struct rnode_packet *next; };
extern "C" void ksr_notify(int scan_code, struct rnode_packet *packet);
#endif
EOF
cat > ksrlib.c <<'EOF'
#include "ksrlib.h"
#include <stdio.h>
extern "C" void ksr_notify(int scan_code, struct rnode_packet *packet) {
    fprintf(stderr, "[KSR_NOTIFY] code=%d\n", scan_code);
}
EOF

# 2) C++ wrapper — Preserve Signatures: 签名不变, 只是放进 class
cat > result_notifier.h <<'EOF'
#ifndef RESULT_NOTIFIER_H
#define RESULT_NOTIFIER_H
#include "ksrlib.h"

class ResultNotifier {
public:
    virtual void ksr_notify(int scan_code, struct rnode_packet *packet);
    virtual ~ResultNotifier() = default;
};

/* global instance — production 默认实现 */
extern ResultNotifier *global_result_notifier;

/* test 用的 fake */
class FakeNotifier : public ResultNotifier {
public:
    int call_count = 0;
    int last_code = 0;
    void ksr_notify(int scan_code, struct rnode_packet *packet) override {
        call_count++;
        last_code = scan_code;
    }
};
#endif
EOF
cat > result_notifier.cpp <<'EOF'
#include "result_notifier.h"

class DefaultNotifier : public ResultNotifier {
public:
    void ksr_notify(int scan_code, struct rnode_packet *packet) override {
        ::ksr_notify(scan_code, packet);  /* 委托给原 C 函数 */
    }
};

static DefaultNotifier g_default;
ResultNotifier *global_result_notifier = &g_default;
EOF

# 3) scanner: 使用 global_result_notifier->ksr_notify 替代直接 ::ksr_notify
cat > scanner.cpp <<'EOF'
#include "result_notifier.h"
#include "ksrlib.h"
#include <cassert>

int scan_packets(struct rnode_packet *packet, int flag) {
    struct rnode_packet *cur = packet;
    int err = 0;
    while (cur) {
        int sr = (cur->body & 0x04) ? 0x04 : 0;
        if (sr) {
            global_result_notifier->ksr_notify(sr, cur);
            err |= sr;
        }
        cur = cur->next;
    }
    return err;
}

/* test main: 用 FakeNotifier 替身 */
int main() {
    /* 替换 global */
    FakeNotifier fake;
    ResultNotifier *old = global_result_notifier;
    global_result_notifier = &fake;

    /* 测: 不真写 stderr, 用 fake */
    struct rnode_packet p = { .body = 0xFF, .next = NULL };
    int rc = scan_packets(&p, 0);
    assert(rc & 0x04);
    assert(fake.call_count == 1);
    assert(fake.last_code == 0x04);

    global_result_notifier = old;
    return 0;
}
EOF

g++ -std=c++17 -Wall -Wextra -o test_migrate ksrlib.c result_notifier.cpp scanner.cpp && ./test_migrate
echo "rc=$?"
```

**验收**: `rc=0`, FakeNotifier 接收到 1 次 ksr_notify 调用, last_code=0x04。这是 C → C++ 的 Preserve Signatures + Encapsulate Global References 实例。

> 💡 **g++ 命令可能不在容器内** — 如果 `g++` 不存在, 把 cc 当 g++ 用 (`cc -x c++` 等)。若都不行, 标注 SKIP 在报告里。

## 六、值得深入思考的问题

### Q1: `#ifdef TESTING` 是 seam 还是设计债务?

C 圈传统观点认为 `#ifdef` 是"丑陋但实用"的 seam。问题是: production build 与 testing build 行为是否真的等价? 寄存器分配、branch prediction、inlining 都会变。**关键问题**: `#ifdef TESTING` wrap 的范围多大时, production 行为仍然等价?

### Q2: function pointer struct 是 class 的前身, 还是反 OO?

`struct database { void (*retrieve)(...); ...}` 像 OO vtable, 但缺少 *封装*(全局 `g_db` 可写)。**关键问题**: 用 function pointer struct 是不是"假装 OO, 实际更脆弱"? 什么时候该直接切到 C++?

### Q3: C → C++ migration 的 *mechanical* vs *semantic*

Feathers 说 `class program` 是 mechanical。但实际: C 的 `extern` 函数与 C++ 的 name mangling 冲突; C++ 的 constructor/destructor 在 C 函数里没有对应; C++ 的 `try/catch` 与 C 的 `setjmp/longjmp` 不互通。**关键问题**: 哪些"mechanical"是真的, 哪些需要重新设计?

### Q4: link seam 与 dynamic linking (dlopen) 的成本对比

传统 link seam 重新链 binary, 毫秒级; dlopen 是运行时加载, 秒级。**关键问题**: 大型系统(数百万 LOC C)用 link seam 重新链一次的时间成本 vs dlopen 一次的时间成本, 选哪个?

### Q5: AI 推荐 "包成 C++ class" 时, 是否考虑了 ABI 兼容?

AI 写的 wrapper class 可能改 struct layout, 导致 binary 不兼容老 client。**关键问题**: team 怎么 enforce "Preserve Signatures, also Preserve Layout"? 这是 review checklist 还是 build system check?

---

*下一章预告*: **Chapter 20 — This Class Is Too Big and I Don't Want It to Get Any Bigger** — ch19 给 non-OO 项目拆依赖的 seam 三件套, ch20 进入 *已经 OO* 但 *class 太胖* 的场景。核心议题:(a) Sprout Class + Sprout Method 当 crisis-mode;(b) 7 个 heuristic 找隐藏职责(method grouping / hidden methods / decisions / internal relationships / primary responsibility / scratch refactoring / current work);(c) **Feature Sketch** — 画 circle-and-line 表示 method 用 instance variable 关系, 找 pinch point 决定 extract 边界;(d) **Interface Segregation Principle** 给"implementation-level SRP"还是"interface-level SRP"提供区分;(e) "职责优先 / 依赖其次" 的 extract class 顺序 — 推测 vs 实测的次序。ch20 是 ch21 (changing same code all over) 的前提。
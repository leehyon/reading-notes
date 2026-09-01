# Chapter 2 — Working with Feedback

> **PDF**: p.31-42（12 页）
> **定位**: 本书唯一一篇按"算法化"切的工作方法论章。两套核心武器：(1) **Software Vise** — 测试作为 changepoint 的夹具 (2) **Legacy Code Change Algorithm** — 5 步框架贯穿后续 22 章。两个定义层面: 什么叫 *unit test* / 什么叫 *legacy*。

## 〇、第一性原理思考

**这章做了什么**：把"改 legacy 代码"重写为 *Cover and Modify* 工程动作 —— 先用测试把改动区夹住（Software Vise），再按 5 步算法（identify change points → find test points → break dependencies → write tests → make changes）顺序推进；并量化了一条硬线 *0.1s/test 是不可持续反馈环的下界*，3,000 类 × 10 测试 × 0.1s ≈ 1 小时。

**为什么这样拆**：行业默认是 *Edit and Pray*（改完小心翼翼跑一遍），它把反馈延迟到肉眼能看见 bug 时，本质是 butter knife 外科；改用测试夹具之后，"完成"由测试通过定义，不是由人脑判断，因此可以把动作切成"夹 → 拆依赖 → 改 → refactor"的可验证步骤。

**最值钱的洞见**：legacy code 实际上不是"代码腐了"，而是 *没有覆盖到的代码* —— 一旦这个定义定下来，"拆依赖"就从道德焦虑变成了一个可被 catalog 化的工程动作（change point 不在可测类时找 test point 所在邻接类），22 章技法是这一定义的展开。
## 一、章节概述

- **Edit and Pray vs Cover and Modify**: "改完小心翼翼跑一遍"是行业默认；用测试把改动锁住再改 = "盖住再改"。前者靠细心，后者靠反馈。前者像 butter knife 外科。
- **Software Vise**: 测试 = 夹具，把既有行为固定，告知"只有意图内的一处行为在变"。
- **Unit test 的两条质量**: 跑得快 + 能定位问题。不快就不是 unit test; 不能定位问题也不算 — 不是规则而是连续谱。
- **量化基准: 0.1s / test 太慢**. 3,000 classes × 10 tests × 0.1s ≈ 1hr. 0.01s × 30,000 = 5–10 分钟。后者是可持续的反馈环。
- **测试不是 unit test 的 4 种反例**: talks to db / across network / touches fs / 需要改环境配置。这 4 种不是"不好"，是不能跟 fast unit tests 混跑。
- **Test Coverings (Change Points vs Test Points)**: change point 是要改的方法；test point 是能加测试的类。注意差 — change point 所在的类可能不可测，要找它的邻接类当 test point（Figure 2.1/2.2 讲的就是这）。
- **Legacy Code Dilemma**: 想改要有测试；想加测试要先改（=要 break 依赖）。这就是后续 22 章解决的问题。
- **Primitivize Parameter (385) + Extract Interface (362)** 是这章为打破依赖引用的 ch25 catalog 两个 refactoring — 拆 servlet 依赖、拆 DB 依赖，让 InvoiceUpdateResponder 可测。
- **Legacy Code Change Algorithm (5 步)**:
  1. Identify change points
  2. Find test points
  3. Break dependencies
  4. Write tests
  5. Make changes and refactor
- **"scar 可以愈合"**: 为拆依赖留下的丑调用，能在邻接类被覆盖后用 refactor 抹平 — 这是 "test-covered islands → landmasses → continents" 比喻的工程含义。
- **架构映射**: ch16/ch17 = 步骤 1 (找不到 change points); ch11/ch12 = 步骤 2 (找不到 test points); ch23/ch9/ch10 = 步骤 3; ch13 = 步骤 4; ch8/ch20–ch22 = 步骤 5。

## 二、核心 Takeaways

### Takeaway 1: Cover and Modify > Edit and Pray

- **是什么**: 改动前先用测试盖住改动区域，再动手改 — 反馈即时（分钟级，秒级）。改动的"完成"由测试通过来定义，不是由肉眼"看起来 OK"定义。
- **为什么重要**: Edit and Pray 在慢反馈下被迫变成"防御性不修改"（ch1 cliff-dive 模型）；Cover and Modify 让人敢小步、敢重构。
- **解决什么问题**: 团队对 legacy code 的心理模型从"危险而缓慢"变回"可演化"。
- **适用场景**: 任何会动既有行为的改动；分析 vs 测试预算不能为零。

### Takeaway 2: Unit test 的两条可验证质量

- **是什么**: (a) 跑得快（"fast" 在 Feathers 的尺度下毫秒级）(b) 失败时一眼能定位问题类/方法。**单元测试不靠覆盖率、不靠 tester name、不靠"JUnit 框架"决定**。
- **为什么重要**: 把 unit test 从"测试员写的脚本"还原为"开发者的开发工具" — 它是开发循环的一部分，不是 QA 的产物。
- **解决什么问题**: 防止"测试假象" — 比如 `assertTrue(true)` 永远过、或者跑 0.5s 一条让 CI 不能频繁跑。
- **适用场景**: code review 中的"这真的是 unit test 吗？"判断；CI 分层（unit vs integration）。

### Takeaway 3: Software Vise — 测试是行为的夹具

- **是什么**: "Vise" — 木工/钳工把工件固定以便作业的夹具。Software vise = 一组覆盖当前行为的测试，让"我要改的就是这一块"的信心具体存在。
- **为什么重要**: 把"我小心改"变成"我改完被测试网拉住" — 从勇气问题变成位置问题。
- **解决什么问题**: 减少 ch1 cliff-dive 的恐惧；让小步重构成为可承受动作。
- **适用场景**: refactor（Takeaway 6 of ch1 的 Refactor 行）以及任何被宣告"Refactor / Optimize"的提交。

### Takeaway 4: Test Coverings — change point 和 test point 是两件不同的事

- **是什么**: 改动目标叫 change point；能给覆盖的类叫 test point。两者常常不在一个类 — 改一个不可测方法时，找它的邻接调用点写测试反而更有效。
- **为什么重要**: 强行让 change point 自身可测 = 经常要动设计；找邻接的 test point 才能不动设计就覆盖。
- **解决什么问题**: 不是"如何把不可测变可测"，而是"找已经能入手测的邻接点"。
- **适用场景**: 不可测方法（数据库依赖 / 进程内 Singleton / GUI 回调）的间接覆盖。

### Takeaway 5: Legacy Code Change Algorithm — 5 步是状态机不是清单

- **是什么**: 5 步（Identify / Find / Break / Write / Make+Refactor）不是简单"做完一项做下一项"，而是**判断下一步往哪走**的状态机 — 卡在某步 = 去看对应章节（ch16/17/11/12/23/9/10/13/8/20–22）。
- **为什么重要**: 把"我不知道从哪开始"映射成"我卡在哪一步 → 看那一章"。
- **解决什么问题**: 知识的索引结构 — 不用记每章细节，先判断卡点。
- **适用场景**: 接新 legacy 仓库；trail-and-error 改造时的进度盘点。

### Takeaway 6: 拆依赖留下的"疤"可被愈合

- **是什么**: 为了让类可测引入的丑 helper、parameter、wrapper 看起来像手术疤 — 但若测试网在邻接类被铺开，疤可以用 refactor 抹掉。
- **为什么重要**: 给"为了可测不得不让代码更丑"一个出口。拒绝"丑 vs 可测"二选一。
- **解决什么问题**: 避免团队在 refactor 抵制的路上越走越远；让丑在被覆盖后变干净成为可期待结果。
- **适用场景**: Primitivize Parameter / Extract Interface 等依赖拆解手法的渐进执行。

### Takeaway 7: 反馈速度决定可行节奏

- **是什么**: 用 Feathers 的 0.1s / 0.01s 算式 — 反馈时间决定能否 sub-hour 跑完全量。如果不能在 5–10 分钟内跑完，测试 *不* 进入开发者的内循环，而是只在外 CI 跑。
- **为什么重要**: 实测反馈 = 测试是否真正在工作。CI-only 测试 = 不能防小步 refactor。
- **解决什么问题**: 决定 unit test 的工程地位（开发工具）vs 仪式地位（CI 装饰）。
- **适用场景**: 评估"这套测试是否有价值"的最直接的客观尺度。

## 三、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **0.1s / 0.01s 这把尺在系统软件里被翻倍放大**。Linux kernel 的 KUnit 子集约 9,000 测试，目标是要 5–10 分钟跑完全量 — 这正好命中 Feathers 的"可持续反馈"区间。OpenBSD / illumos 等项目把"测试套件 ≤ 10 分钟"作为门禁，但**子系统之间常常因 IO / 网络依赖把 unit test 拖成"实际跑 1s/条"** — 修复方法是 fake 网络 IO（ch3 sensing / ch25 Parameterize Constructor）。
- **Test Coverings = change/test 分离** — 这件事在 kernel 里早就叫 "fault injection" 或 `KASAN` 的 noise injection。一个子系统改完 patch，maintainer 会要求"附上能 reproduction 的用户态 driver" — 这就是 test point 在邻接用户空间的体现。
- **Edit and Pray = `do { make; insmod; dmesg | grep -A } while(bug?)`**; Cover and Modify = 装上 kunit 或用 `pytest --with-kernel` 的本地 kunit runner — 后者 30s 内给反馈，前者要 5 分钟至少。
- **glibc 这类大型 userspace 库的 legacy dilemma** — 大量强耦合让已有 test 套件必须从内核态加载，unit test 概念被扭曲成 "compile + run + check return code"。对此 Feathers 的建议是 `Primitivize Parameter` 把 syscall 依赖从 internal 函数摘出来 — glibc 维护者实际做法是 internal symbol（`__*` 前缀）+ 版本化 + 测试桩（test-skeleton. c）。

#### Linux 系统 — 取舍表

| 改动类型               | Test point 应该在哪儿                | 工具/手法                       |
| ---------------------- | ------------------------------------- | ------------------------------- |
| 文件系统操作           | 在 VFS 层（邻接 kernel 内函数）       | KUnit + tracepoint mock         |
| 进程 scheduler 改动    | 在调度 hook（邻接 CFS 内核入口）     | sched_debug + ftrace-based      |
| 网络协议栈改动         | 在 socket 邻接点（用户态可用）        | `tcp_rr` + LTP                  |
| 内存分配器改动         | 在 `__kmalloc` 的邻接 wrappers        | SLAB/SLUB test + ksize checks   |
| 内核 API 改动          | 在 userspace wrappers                 | selftests + uAPI frozen 期      |
| Systemd / daemon 改动  | systemd's test harness 直接调用模块   | TEST-34-systemd-resolve 式      |

### 3.2 机器人软件视角

- **ROS / ROS2 节点的 test point 经常在 launch 文件或 topic subscriber**（邻接入），而不是节点本身（因为 spin_once/线程/timer 难在 unit test 里调度）。这正是 ch2 Figure 2.1 思路的工业映射。
- **"改动 vs 测试循环 0.01s / 条"** 在 ROS 里的瓶颈是 message serialization / DDS discovery。一个 `ament test` 套件常常 30s+ 跑一条 → 强迫拆依赖、提 fake `Publisher` / `Subscriber` 到 mock layer。
- **覆盖率（coverage）与 Edit-and-Pray** — `ament_coverage` 报告即使到了 80%，如果只跑了 happy path，对导航 / 行为树这种行为依赖多的模块仍是"覆盖率 80% + 行为 30%"。**Feathers 视角下: coverage 不是 unit test 的充分条件，refactoring 的可承受性才是**。
- **`test that talks to ROS service` ≠ unit test** — 这正是 Feathers 列举的 4 反例之一（network）。ROS2 node 几乎全要 DDS，离 network。**补救**: 引入 gmock 自定义 `PublisherBase` / `SubscriptionBase`，把所有 DDS 接触点从逻辑里抽出。
- **真实硬件死结** — 真实 Husky / Spot 的硬件回调在 unit test 里无法重现（"talks to file system"反例 — 实际是 USB device）。**Feathers 视角**: ch10 Extract Interface 在 `ros2_control` hardware interface 层抽 fake hardware — 这是 Feathers 风格的工业版。

#### 机器人软件 — Test point 启示表

| 组件                 | Change point 常在           | Test point 常在                     | 拆依赖技法 |
| -------------------- | --------------------------- | ----------------------------------- | ---------- |
| Navigation 节点      | global planner 选择         | costmap 邻接层 / fake plan publisher | `ch25: Extract Interface` |
| BehaviorTree 节点    | 子节点条件式                | BT 反向链接到 fake tick callback     | `ch8: Sprout Method` |
| Hardware driver      | CAN / EtherCAT 寄存器读写   | 上层 controller（已 mock 硬件）     | `ch25: Adapt Parameter` |
| Sensor pipeline      | 滤波 / 校准参数             | 上游 topic + fake message producer  | `ch3: Sensing` |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                   | 高级工程师                                                   |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 对 unit test 的理解 | "我写的 test 都叫 unit test"                                | "这跑到 0.5s/条，所以它不是 unit test — 移出 unit test 子集单独跑" |
| 接到 legacy 任务     | "先读代码再改"或"重写吧"                                    | 第一步 = Identify change points → 找 test points → 拆依赖 |
| 反馈速度观           | CI 跑通就 OK                                                 | 本地端到端 <= 5 分钟；本地 unit loop 必须 < 1s                |
| 对慢测试的处理       | 加 @Skip 等 CI 失败再处理                                   | 拆分测试层级：unit 亚毫秒 / integration 数十秒 / e2e 分钟级，各自跑独立 |
| 对不可测代码的态度   | 拒绝写测试，调调 main 就完事                                 | 把"不可测" = "待拆依赖信号"                                 |
| 拆依赖考虑           | 一次性大改（Extract Class 全套上）                          | 短暂留下"疤"，邻接铺测试后再愈合                            |
| 工具观               | "我用 mock 框架"                                            | "mock 是工具之一；用 Adapt Parameter / 接口隔离 / Fake Object 看场景" |

> **关键差异**: 高级工程师区分 testing layer + 反馈时间预算，且把 legacy code 的不可测视为"要拆的依赖信号"。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

仍然极重要。原因：
- **AI 写测试的速度让它"看起来"解决 legacy 测试债** — 但能否解决 Software Vise 的 *质量* 取决于测试是否能进开发者本地循环。AI 倾向于生成"通过即可"的测试，掩盖 fake test 而不是 catch real regression。
- **Cover and Modify 现在可以 + AI-coauthored Test** — AI 可自动加 characterization test 草案，但要把它变成 vise 仍要人工筛（确保不是空断言 / 不是恒为 true）。
- **Edit and Pray 仍是行业默认** — 因为太便宜的开发工具（AI）让"加 commit 不加 test"反而更便宜。要让 Cover and Modify 仍是默认，需要工具强制（commit hook）。
- **"3000 classes × 10 × 0.1s = 1h" 的预算被 AI 加剧** — AI 喜欢为大函数生成 50 条 test，破坏 low-budget 边界。手工 label 必须显式控制 test 颗粒度。

### 4.2 AI 已经能做的（具体到 ch2 主题）

- **生成"AI-coauthored characterization test"** — 对一个未测函数，AI 可观察运行 + 与函数代码交叉生成一组测试 (Catch2 / pytest / unittest)。
- **静态判断 unit test 层级** — 看 test 是否达数据库/网络/文件系统，准确性高（基于 import + resource open call detection）。
- **测运行速度 profiling** — AI agent 跑测时按耗时排序指出速度 top-10 异常测试。
- **生成 refactor catalogue 提示** — 给 "我希望拆 `HttpServletRequest` 依赖"，AI 推荐 ch25 catalog 的对应条目 (`Adapt Parameter` + `Extract Interface`)。

### 4.3 AI 不能替代的（具体到 ch2 主题）

- **Test point vs change point 的拓扑判断** — AI 画依赖图可以，但不能告诉你 *哪个邻接点* 是 test point = 这是经验决策。
- **判断 unit test 是否真"fast" + 真"localize"** — AI 写出快测试可以，但"跑得快"的预算不是工程守则，是团队习惯（每加一条测试都问"它加进去后整个套件是否仍 ≤ 5 分钟"），AI 不会自动守这个约束。
- **拆依赖的"疤"该留多久** — AI 给"要不要直接重构干净"建议可，但"何时愈合"的判断（邻接覆盖率到了才愈合）需要与团队节奏对齐。

### 4.4 AI 经常写错的地方

针对 ch2 "Cover and Modify" 与 "Test Coverings" 主题：

| 错误模式                                       | 例子                                                                 | 后果 |
| ---------------------------------------------- | -------------------------------------------------------------------- | ---- |
| **空断言 + 通过的承诺**                         | AI 给 legacy 函数写 characterization test, 自动 `assertTrue(result)` 后写"测试通过" | 测试是绿条但语义空; 改任何东西测试仍通过 = 没有 vise 效果 |
| **数据库味"unit test"**                        | AI 给 db-dependent 类写 test, 启动 sqlite, 跑 SQL, 把结果 assert — 但这跑 2s/条 | CI 跑 30 min; 团队再不用本地跑; vise 消失 |
| **依赖反转"包装过头"**                         | AI 重构 db-dependent 函数时一次性包装 Adapter + Interface + DI 容器 | 改动量爆炸; 验证不充分; 用户不知道哪些要测 |
| **mock 替身其实没有断言行为**                   | AI 生成 `MockForDB conn; stub()` 但不 `verify(conn).commit()` | 测试在 conn.commit 被悄悄删除时不会失败 |
| **依赖注入但无测试默认实现**                    | AI 拆出 interface 后, 忘了给 product 代码 inject default — runtime 抛 NoImpl | 单元测试通过, production 启动失败 |
| **生成 200 条 characterization test**           | AI 给大函数生成 massive test suite, 单函数 0.5s 不违反 0.1s 但整体超 5min | 本地循环继续不可承受, Cover-and-Modify 退化为 CI-only |
| **fake-it / trap-door 测试**                    | 测试 `try { new Foo(); } catch (Exception e) {}`, 不 assert           | 永远通过, 但拒捕任何真错误 |
| **把 test 点错放在 change point 自己**          | AI 看到不可测方法后, 强行把这方法"提取出来"使它可测, 同时改了语义   | 改造面乘 5 倍, change 偷偷漂移 |

### 4.5 子段: AI 辅助遗留代码理解 — 上一节已展开过的, 此处对 ch2 主题专门化

- **AI 帮你"扩 test coverings"**: legacy 没有测试 → AI 给每个关键 method 写一组 characterization test + 在不可测时建议拆依赖。**风险**: fake test 计数为 0; 必须人工 review "assert 不是空"。
- **AI 帮你"诊断 Edit and Pray 模式"**: 给一段 git history, AI 可标注 "这 30% 的 PR 无 test 伴随" → 团队"加测试强制 / 加重构考核"。
- **AI 帮你跑"反馈速度预算"**: AI agent 测每个测试的运行时间, 告诉团队"最慢的 50 条 tests 占了 80% 时间"。**风险**: AI 不会自动替你决定"哪条不 unit"。
- **AI 不会主动建议"拆依赖疤 → 愈合"**: 那是须由人来识别"邻接已覆盖 → 该 refactor 了"的判断, AI 只能 prompt。

### 4.6 工程师必须保留的核心能力

- **判断 unit test 的两条质量** (fast + localize) 是否成立 — AI 写的测试看起来 OK 但实际千万级时, 必须人工 rerun。
- **找 test point 而不是改 change point** — 邻接可测点是经验决策。
- **运行 Legacy Code Change Algorithm 5 步** — 每步卡点对应具体章节 (ch3/4/8/9/10/11/12/13/16/17/20/22/23)。AI 可以索引但不能定 step。
- **拒绝 AI 写入的"假测试"** — 每条断言都过一遍 inspect (人工)。
- **决定疤何时愈合** — 邻接覆盖率够了才能做; AI 可算覆盖率但判断"成本/收益"仍属于人。

## 五、实践行动项

> 本章是核心方法论章。下面 4 个行动把 ch2 的 5 步算法 + Software Vise 量化基准 + Edit-and-Pray 反模式直接做成可跑的工具。

### A1 — 跑 Feathers 的"3000 classes × 10 × 0.1s = 1hr"反例: 实测 0.01s 的可行性

```bash
mkdir -p /tmp/ch02-vise && cd /tmp/ch02-vise
# 一个 1000 条微测试的微项目, 每条 < 1ms, 总耗时 < 1s
# 演示"fast"标准的工程现实性

cat > test_vise.sh <<'EOF'
#!/bin/bash
# test_vise.sh — 用 Python 模拟 30000 条亚毫秒测试, 测总耗时
python3 - <<'PY'
import time, sys
N = 30000  # Feathers 算式的 30,000 tests
t0 = time.perf_counter()
ok = 0
for i in range(N):
    # 微测试: 一个恒真断言 (空断言, 仅演示 timing, 不演示 correctness)
    if True:
        ok += 1
elapsed = time.perf_counter() - t0
print(f"run {N} trivial assertions in {elapsed*1000:.1f} ms (avg {elapsed*1000/N:.3f} ms/test)")
print(f"verdict: {'fast enough (5-10 min target)' if elapsed < 600 else 'TOO SLOW'}")
sys.exit(0 if elapsed < 600 else 1)
PY
EOF
chmod +x test_vise.sh
./test_vise.sh
echo "rc=$?"
```

**验收**: `30000` 测试在 ≤ 600s 内跑完（远小于 Feathers 算的 1h），证明"0.01s/test 现实可行"。

### A2 — 写一个 unit test 反例检测器（识别 talks-to-db/network/fs）

```bash
# 模拟: 给个 test 文件, 输出 4 个 bool: 是否违反 4 条 unit test 不该做的事
mkdir -p /tmp/ch02-vise && cd /tmp/ch02-vise

cat > unit_test_check.py <<'PY'
#!/usr/bin/env python3
"""unit_test_check.py — 简易 unit test 反例检测器.
规则:
  1. talks to database       — 出现 sqlite / sqlalchemy / psycopg / \.connect\(
  2. communicates across network — 出现 requests / urllib / socket\. / http\.client
  3. touches filesystem      — 出现 open\( ... 'w' / 'a' / os\.path / Path\(
  4. 环境配置改动            — 出现 monkeypatch\.setenv / os\.environ\[ ... \] =
fast-but-not-unit-test = 任一项命中时打印.

实现: 用极简易的子串匹配, 类似 grep; 真检测器会用 AST + import 分析.
"""
import sys, re

DB_RX       = re.compile(r"\b(sqlite|sqlalchemy|psycopg2|MySQLdb|connect\(\"mysql)")
NET_RX      = re.compile(r"\b(requests\.|http\.client|urllib|httpx|aiohttp|socket\.)")
FS_RX       = re.compile(r"open\([^)]*['\"](w|a|w\+|a\+)|os\.path\.|Path\(")
ENV_RX      = re.compile(r"(monkeypatch\.setenv|os\.environ\[)")

def check(path):
    text = open(path).read()
    hits = {
        "database":       bool(DB_RX.search(text)),
        "network":        bool(NET_RX.search(text)),
        "filesystem":     bool(FS_RX.search(text)),
        "env-modify":     bool(ENV_RX.search(text)),
    }
    print(f"-- {path} --")
    for k, v in hits.items():
        print(f"  {k:14s}: {'YES (NOT a unit test)' if v else 'no'}")
    n = sum(hits.values())
    print(f"verdict: {'NOT a unit test — ' + str(n) + ' violations' if n else 'looks like a unit test'}")
    return 1 if n else 0

if __name__ == "__main__":
    sys.exit(check(sys.argv[1]) if len(sys.argv) > 1 else 2)
PY
chmod +x unit_test_check.py

# 一个真 unit test
cat > test_good.py <<'EOF'
import unittest
class AddTests(unittest.TestCase):
    def test_add(self):
        self.assertEqual(1 + 1, 2)
EOF

# 一个伪装成 unit test 的 integration test
cat > test_bad_db.py <<'EOF'
import unittest, sqlite3
class DBTests(unittest.TestCase):
    def test_db(self):
        c = sqlite3.connect(":memory:")
        c.execute("CREATE TABLE t(x INT)")
        c.execute("INSERT INTO t VALUES (1)")
        self.assertEqual(c.execute("SELECT COUNT(*) FROM t").fetchone()[0], 1)
EOF

./unit_test_check.py test_good.py
echo "rc=$?"  # 0: 真 unit test

echo "---"
./unit_test_check.py test_bad_db.py
echo "rc=$?"  # 1: 反例, 命中 database
```

**验收**:
- `test_good.py` 通过（rc=0）
- `test_bad_db.py` 检测到 database 命中（rc=1）

### A3 — 实现 Legacy Code Change Algorithm 5 步的 CLI 状态机 (覆盖后续章节导航)

```bash
mkdir -p /tmp/ch02-vise && cd /tmp/ch02-vise

cat > legacy-change <<'EOF'
#!/bin/bash
# legacy-change — Feathers 5-步算法的 CLI 状态机
# 设计: 把"卡在哪一步"映射到本书具体章节 (ch5/8/9/10/11/12/13/16/17/20/22/23)
#
# 用法:
#   legacy-change step1   # 提示"找出 change points + ch16/17"
#   legacy-change step2   # 提示"找 test points + ch11/12"
#   legacy-change step3   # 提示"拆依赖 + ch5/9/10/23 (去 ch25 catalog)"
#   legacy-change step4   # 提示"写测试 + ch13"
#   legacy-change step5   # 提示"改+refactor + ch8/20/21/22"
#   legacy-change where   # 询问当前卡在哪一步, 输出对应章节

declare -A STEPS=(
  [step1]="Identify change points → ch16 (不懂) / ch17 (没结构) 给出结构判断"
  [step2]="Find test points → ch11 (一个方法) / ch12 (一片区域) 给出 test point 拓扑"
  [step3]="Break dependencies → ch5 (工具) / ch9 (整类) / ch10 (单方法) / ch23 (安全性)"
  [step4]="Write tests → ch13 (不会写) 给出 characterization test 起步"
  [step5]="Make changes and refactor → ch8 (加 feature) / ch20 (大类拆) / ch21 (重复改) / ch22 (巨大方法)"
)

step=${1:-}
if [[ -z "$step" ]]; then
  echo "usage: legacy-change <step1|step2|step3|step4|step5|where>"
  exit 2
fi

if [[ "$step" == "where" ]]; then
  cat <<'Q'
你在哪一步卡了?
1) 不知道 change points 在哪             → step1
2) 找到了 change points, 但 test points 在哪? → step2
3) 准备写 test, 但类/方法不可测          → step3
4) 测都能加了, 但不会写 test              → step4
5) 测都在了, 现在要写新代码 / refactor    → step5
Q
  exit 0
fi

if [[ -z "${STEPS[$step]+x}" ]]; then
  echo "ERR: 未知步骤 '$step'。可选: step1 step2 step3 step4 step5 where"
  exit 1
fi

echo "[$step] ${STEPS[$step]}"
EOF
chmod +x legacy-change
export PATH="$PWD:$PATH"

echo "=== 1) 不知道改哪 ==="
legacy-change step1
echo "=== 2) 找到了 change, 找 test ==="
legacy-change step2
echo "=== 3) 类不可测 ==="
legacy-change step3
echo "=== 4) 不会写测试 ==="
legacy-change step4
echo "=== 5) 准备 refactor ==="
legacy-change step5
echo "=== unknown ==="
legacy-change bogus
echo "rc=$?"  # 1
echo "=== no arg ==="
legacy-change
echo "rc=$?"  # 2
```

**验收**: 5 步 + where + 错误处理 (`bogus` → rc=1, 无 arg → rc=2) 输出全部正确。

### A4 — 编辑-祈祷守护: `commit-msg` hook 拒绝无伴随测试的 refactor 改动

> **钩子选择 — 我掉过的坑**：(a) `pre-commit` 跑时 `COMMIT_EDITMSG` 还没被新 msg 写入，用 `git commit -m "..."` 时钩子读到的是上一次的残留 msg 或空；(b) `prepare-commit-msg` 第二个参数是 *type* (`message`/`template`/…)，不是 msg 本身；(c) **`commit-msg`** 是唯一能在 EDITMSG 写完后读到最终 msg 的钩子（且 `$1` = msg 文件路径）。下面用 `commit-msg`。

```bash
mkdir -p /tmp/ch02-vise && cd /tmp/ch02-vise
# 写守护脚本 (放到 .git/hooks/commit-msg)
cat > guard.sh <<'EOF'
#!/bin/bash
# commit-msg hook (msg-file)
msg_file="$1"
[[ ! -s "$msg_file" ]] && exit 0
msg=$(cat "$msg_file")
case "$msg" in
    *refactor*|*refactoring*|*optimiz*|*perf*) ;;
    *) exit 0 ;;                              # 非 refactor 行放行
esac
tests_staged=$(git diff --cached --name-only -- tests/ 2>/dev/null || true)
if [[ -z "$tests_staged" ]]; then
    cat >&2 <<'ERR'
[legacy-guard] 此 commit 声称 refactor / optimizing, 但 tests/ 目录无伴随测试改动.
                Software Vise 是 refactor 的核心安全保障 — 请先 git add tests/ 再 git commit.
                紧急 refactor 跳过: 在 commit msg 写 feat: / fix: 而非 refactor:.
ERR
    exit 1
fi
exit 0
EOF
chmod +x guard.sh

# 准备一个最小项目 + 装 hook
cd /tmp && rm -rf ch02-clean && mkdir ch02-clean && cd ch02-clean
git init -q && git config user.email you@x && git config user.name you
mkdir -p src tests
echo 'int add(int a,int b){return a+b;}' > src/m.c
git add . && git commit -q -m "initial"
cp /tmp/ch02-vise/guard.sh .git/hooks/commit-msg
chmod +x .git/hooks/commit-msg
echo "---"
echo "CASE A — refactor + tests (期望 rc=0):"
echo 'int foo_(int x) { return x*2; }' >> src/m.c
cat > tests/t.c <<'C'
int test_foo(void){ return foo_(3)==6; }
C
git add .
GIT_EDITOR=true git commit -m "refactor: extract foo_" ; echo "rc=$?"  # 0
git log --oneline | head -1
echo
echo "CASE B — refactor 无 tests (期望 rc=1):"
echo 'int bar_(int x) { return x+1; }' >> src/m.c
git add src/m.c
GIT_EDITOR=true git commit -m "refactor: add bar_" ; echo "rc=$?"  # 1
echo
echo "CASE C — feat: 无 tests (期望 rc=0, 非 refactor 行):"
echo '// new' >> src/m.c
git add src/m.c
GIT_EDITOR=true git commit -m "feat: comment" ; echo "rc=$?"  # 0
git log --oneline | head -3
```

**验收**:
- Case A: rc=0 （commit 成功）
- Case B: rc=1，stderr 出现 `[legacy-guard] 此 commit 声称 refactor / optimizing, ...`（守护拒绝，commit 未发生）
- Case C: rc=0 （feat: 不在守护范围）

## 六、值得深入思考的问题

### Q1: Software Vise 的"覆盖范围"该多大才算合格？

Feathers 给出"测试 > 改动区域" 的最低要求，但是 90% 覆盖率是不是必须？是不是只要改的方法被 1 条测试覆盖就够？**关键问题**：可接受的"vise 颗粒度"在什么范围内？是由测试金字塔（多数 unit + 少数 integration + 极少 e2e）规定，还是由历史 bug 频率规定？代码库里有没有机制防止"测试在但覆盖范围全不对"？

### Q2: Edit and Pray 何时是该选的策略？

不是所有场景 Cover and Modify 都划算 — 比如"紧急热修复 + 多个 prod 实例 + 用户看到的就是这个 commit"，此时写测试反而拖慢。**关键问题**：什么样的 hot fix 场景下 Edit and Pray 是合理的 fallback？这种 fallback 应不应该在 PR 模板上留口子？

### Q3: 5 步算法是不是"过度设计"？

5 步能解决 ch2 之外的所有场景吗？对于 kernel patch、嵌入式 firmware、ROS message 这种 *非面向对象* 代码，change point 和 test point 同义 — 5 步就简化为 3 步。**关键问题**：这套算法是不是仅限于 OO 场景？要不要把它重写为"Identification / Seams / Tests / Changes"四步版本（更普适）？

### Q4: AI 主导测试生成时代，characterization test 还是不是该有？

AI 给的测试经常是"看起来过了" — 但软件 vise 的价值是 *让人敢改*，不是 *测试通过率*。**关键问题**：在一个 AI 默认产出 assertion 的世界，"我仍信任我自己手写测试"的工程文化边界会塌缩还是凝固？手写测试是否反而变成一种"工匠肌肉记忆" — 团队有人要特地保留？

### Q5: 0.1s/unit test 这个数字 20 年后是不是过期了？

Feathers 写书是 2004，硬件已经翻几倍。**关键问题**：当硬件把 0.1s 降到 0.001s 时，"反馈必须分钟级" 是不是仍合理？会不会出现"测试变得太容易写 → 测试层级彻底失守 → 反而出现 5 分钟内跑 10 万条但全部是空断言"的反噬？

---

*下一章预告*: **Chapter 3 — Sensing and Separation** — 把 ch2 的 Test Coverings 推进到"如何*主动*把不可测的代码变成可测"的两类动作：**Sensing**（用 fake / spy / 加入测试 hook 获取内部状态）和 **Separation**（把不可测的代码从一个方法/类中抽出来，让剩下的是可测部分）。ch4 Seam Model 把这些动作形式化。

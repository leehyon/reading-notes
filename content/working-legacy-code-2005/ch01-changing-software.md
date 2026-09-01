# Chapter 1 — Changing Software

> **PDF**: p.25-30（6 页）
> **定位**: 全书概念底盘。本章不教技术，只立"为什么改 = 改四种东西里的一种"和"改 = 三道题"的判断框架；后续 24 章都是这个框架的症状化展开。

## 〇、第一性原理思考

**这章做了什么**：把"修改软件"这件大家凭直觉做的事，拆成一张二维表 — 横轴是改动理由（加功能 / 修 bug / 重构 / 优化），纵轴是不变量（结构 / 新功能 / 旧功能 / 资源占用）。每一格里写明这一类改动"动了哪些、没动哪些"。

**为什么这样拆**：因为工程师每次 PR 实际都在问两个问题 — 我这一改动是哪种？我承诺把哪些保住，不动？拆开之后这两个问题可以写在 PR 模板里被同行 review，不再是个人直觉。后续 24 章每条具体方法（怎么加测试、怎么拆依赖、怎么写 mock），本质上都是在第一格的列里做*更精细的拆法*。

**这章最值钱的洞见**：保留既有行为（保住旧功能不破）的难度，往往远大于加一个新行为。改动的工时应该按"保住多少"算，不是按"改多少行"算。

## 一、章节概述

- **改变 = 在结构、新功能、既有功能、资源占用这四者之间挑哪几个动**：把"修改代码"这个动作从含混的"做改动"还原成结构化判断 — 究竟动的是什么，保住的是什么。
- **四种理由对四张保表（4 种改动 × 4 类不变量）**：加功能保 1 条不变（除了结构 + 新功能 + 既有功能 + 资源占用里挑一条），修 bug 保 2 条，重构保 2 条，优化保 2 条；只有重构和优化严格保功能。
- **行为比"特性还是缺陷"的分类更根本**：公司追踪缺陷和特性是财务需求；对写代码的人来说，"加行为 vs 改行为"才是分界线 — 前者加一个新方法就行，后者得改既有调用链。
- **改动 = 三道题**：要改什么 / 怎样算改对了 / 怎样算没改坏不相关的。第三题最难的，因为不知道多少既有行为处于风险区。
- **不主动改 = 制造更大风险**："不出问题就不修"把短期风险延后成长期结构腐化；不常拆大类的工程师手生；对修改的恐惧累积成"悬崖跳"式开发。
- **保留行为比改行为难**：保留的范围比改变的范围大；认识不到自己不知道什么 = 改代码的最大风险来源。
- **Figure 1.1 那张表是后续 6–24 章的隐藏打分表** — 每条 legacy 修改建议最后都要回到"这四条不变量里，具体保了哪几条"。
- 本章与后面章节的关系：第 2 章解决第二题和第三题（测试）；第 3 章解决"在不破坏既有结构的前提下加测试"。

## 二、核心 Takeaways

### Takeaway 1: 改变 = 在三大类不变式里找缺口

- **是什么**：把每次代码改动看作对三个维度（structure / functionality / resource usage）的状态机转换；大多数改动同时动其中几样，但**功能 vs 非功能**的切分是稳定可教的。
- **为什么重要**：把"改对了"从直觉变成可验证 — 对每一个维度都可以问"它现在和改动前是否一致？"
- **解决什么问题**：让 team 对"改动范围"有共同语言，避免"我以为只是 refactor 你以为是新功能"的争论。
- **适用场景**：code review 时讨论改动影响面；估算改动工作量；判断要不要在合并前加测试。

### Takeaway 2: 改 = 三道题（what / right / not broken）

- **是什么**：对任何改动，先问 (1) 做什么 (2) 怎样知道做对了 (3) 怎样知道没把不相关的弄坏。第三题是真正的难点。
- **为什么重要**：第一题靠文档，第二题靠 spec/test，第三题靠 regression 测试覆盖 — 三题对应不同的工程工具链。
- **解决什么问题**：把"我能不能直接改"变成"我在哪道题上有缺口"；缺口定位 = 缺的工具 / 缺的测试。
- **适用场景**：进入未知遗留代码前；做新功能时和 PM 对齐"边界"；规划 release 前的 risk review。

### Takeaway 3: 添加 vs 改变 — 比 feature vs bug 更工程

- **是什么**：把"加一个 method"和"改一个 method"区分开 — 前者默认不破坏既有行为，后者必须先建立回归网。这与会计上的 feature/bug 分类正交。
- **为什么重要**：让"该不该加测试"由行为变化方向决定，不由 PM 的标签决定。
- **解决什么问题**：防止 PM 在 backlog 上把破坏性改动标成"小任务"绕过测试。
- **适用场景**：评估 ticket、改 PR description、回顾事故时定位"我们当时为什么没测"。

### Takeaway 4: 规避改 = 制造更大风险

- **是什么**："if it's not broke don't fix it" 把短期风险延期成长期结构腐化；不经常拆大类的开发者技术生疏；对修改的恐惧累积成"cliff dive"式开发。
- **为什么重要**：工程的健康度不会因为不主动改而守住，只会在改的反复训练中守住。
- **解决什么问题**：让团队摆脱"修不如不修"的瘫痪状态，但同时要承认 — 没有测试网络时这种瘫痪是理性的。
- **适用场景**：季度规划、回顾会议、对新人讲"为什么我们要先花两周搭测试"。

### Takeaway 5: 保留行为的复杂度 ≫ 引入新行为的复杂度

- **是什么**：新行为可以由新代码承载，保留行为需要让既有代码的所有调用者、所有隐含不变式、所有 race / 异常路径都不变。这是不对称的。
- **为什么重要**：定价工时的常见错误是按改动行数估，而保留行为的开销随代码使用面放大。
- **解决什么问题**：把"改这个 API 要几小时"换成"改这个 API 要多少回归覆盖"再换算工时。
- **适用场景**：估算 ticket、解释"为什么只是 5 行业务代码改完测了三天"。

### Takeaway 6 (隐式 — Figure 1.1 矩阵): 四种改动 × 不变式

把 ch1 p.28 那张表还原成下面的 3×4 矩阵（**这是后续 ch6-ch24 的隐藏打分表**）：

| 改动类型            | Structure | 新增 Functionality | 既有 Functionality | Resource Usage |
| ------------------- | :-------: | :----------------: | :----------------: | :------------: |
| **Adding a Feature**    |    变     |         变         |        **保**      |       保       |
| **Fixing a Bug**        |    变     |         保         |         变         |       保       |
| **Refactoring**         |    变     |         保         |        **保**      |       保       |
| **Optimizing**          |    保     |         保         |        **保**      |       变       |

> 行按"什么不变"分类：feature=1 不变, bug=2, refactor=2, optimize=2。Refactor + optimize 共有的"功能强不变"是后面讲 seam / mock 的根 — 因为一旦保功能不变，**网络测试**才能让我们敢动 structure。

## 三、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**（按本会话惯例）。

### 3.1 Linux 系统开发视角

- **避开改动的代价在系统软件里被放大**。Linux kernel 一旦错过某个 release window 就得排队到下一个；用户态 daemon 因为没人敢 refactor 越变越大。**Feathers 视角下**：kernel 之所以能持续演化恰是因为它有最严密的测试网（KUnit + LTP + syzkaller + LKD/bisect），而不只是因为它 review 严。
- **API 稳定性 = existing functionality 不变。** 这正是 Takeaway 6 矩阵中"Adding a Feature"的保表那一列。Linux UAPI 一旦设了 `errno` 值就不能改 — 因为改 = bug。**问题：errno 再不合理的值（如 1995 年定下的 `ENOTSUPP`）也得永远支持**，对应工程动作：不要用保留 errno 数做协议魔数、不要做"假设 errno 一定不在某个区间"这种 hack。
- **三大类不变式做 review checklist**：每次 patch 提交前，提交者自己先答两题 — (1) 此次提交属于矩阵中哪一行？(2) 我对它声称"保"的那几列，写了测试吗？内核子系统 maintainer 不接受口头 claim。
- **规避改动在 daemon 里很常见**。systemd / OpenSSH / NetworkManager 都经历过"先 hold 测试债再重构"的窗口期，启示：**测试网到位前不应该接 refactor 任务** — 否则就掉进"cliff dive"模型。
- **惯例**：嵌入式/系统侧新人最常见的失败模式是把"动一个小行为"做成"快速 PR" 不附 regression test；按 Takeaway 6 这种 PR 在 adding/bug 行上等于不达标。

### 3.2 机器人软件视角

- **ROS/ROS2 节点是典型的高扇出 behavioral 集合**。一个 `MoveBase` 行为保住不动的难度 = 它和 costmap、tf、AMCL、controller 之间的契约都保住。**Feathers 视角**：ROS 节点功能本身的可用性测试远少于 seam 测试（节点-参数 / 节点-话题 / 节点-硬件 mock），这直接限制了 ROS2 代码库的可演化速度。
- **`launch` 文件 + 参数动态加载是为 seam 而设的**（不知 Feathers 是否有意，但这和 ch3 的 Sensing/Separation 高度对齐）。启示：**为机器人代码做 unit test 时，要从 launch 文件反向追溯"哪些可注入点没利用上"**，这比硬塞 mock 更省力。
- **真实硬件依赖 = 嵌入式死结**。Feathers 在 ch3 ch14 把"kill me by a library" 当核心症状；机器人侧 = "kill me by serial port / EtherCAT / CAN bus"。**做法**：用 ch25 的 `Adapt Parameter` 在 HAL 层之下插入 wrapper，等同于 `ros2_control` 的 hardware interface 抽象。
- **机器人软件行为保留的隐性税极高**：一条动作（如"导航到目标点"）涉及行为树 + costmap + 控制器三重决策链路。"改 navigation 行为"看起来是 1 行代码变动，**保留行为 = 数天的集成测试** — 这正好印证 Takeaway 5（保留行为的开销随代码使用面放大）。

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                   | 高级工程师                                                   |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 接到改动任务         | "大概不会破坏什么吧"或"改完再测"                            | 先答"在矩阵哪一行"——进 refactor 行就先写 characterization test |
| 看到 legacy 代码     | "代码太烂，建议重写"                                        | "这块代码丑但有 10 处行为是我不知道的，重写 = 丢 10 处"      |
| 估算工时             | 按 LOC 估，或者按"我读懂用了多久"                           | "改动 5 行 + 配套 regression 套件 3 天"                      |
| 风险观感             | 改 1 个文件 = 改 1 个文件的风险                              | 改 1 个文件 = 触发该文件所有 caller 的潜在回归               |
| 对测试的偏好         | 觉得"测试拖慢迭代"                                          | 觉得"没测试网的开发 = 自我减缓迭代速度"                     |
| 用的诊断工具         | 调试器、print                                                | git bisect、coverage、characterization test、microbenchmark   |
| 对"我以为只改一个东西"的恐惧 | 无感                                                       | 看到 PR 描述说"只改 X"就警觉（要么不实，要么隐含更大改动） |

> **关键差异**：高级工程师把"改"和"保"是分开估量的；初级工程师把"改完"当成目标，忽略"保"的开销。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

仍然极重要。原因：
- **LLM 写代码的速度让改动更频繁** — 加速了矩阵中"Adding a Feature"行的密度。
- **LLM 的安全性没有内置回归网** — 它写出绿条测试不代表等价修改，更不代表既有行为保留。
- **legacy 不会消失** — AI 不能消除既有的紧耦合、状态机、隐含协议；只会让"修补"变快，把 debt 滚到下一次。
- **"行为保留"在 AI 主导的 PR 流里更脆弱** — LLM 倾向做局部可见的最小改动，但跨文件行为链路是它看不透的。Takeaway 3（adding vs changing）的边界因此需要人工把关。

### 4.2 AI 已经能做的（具体到 ch1 主题）

- **自动分析 PR 类型并归到 4-理由矩阵的哪一行** — 看 message + diff 即可分类（Adding/Refactor/Optimize/Bugfix），精度 ~85%。
- **生成 characterization tests**（接收未知遗留代码 → 输出现有行为的黑盒快照式测试）— 这是 ch2 的预告，AI 现在已能批量产出。
- **绘制 caller graph + 标注风险面** — 替代"凭直觉知道改这个文件影响很大"，代码分析工具 + AI 总结能瞬间给出。
- **回归测试选择性补足** — diff 之后 AI 根据语义建议"这个改动应该新增哪些回归用例"，粒度到函数级。

### 4.3 AI 不能替代的（具体到 ch1 主题）

- **判断一改动属于哪一行 + 这一行的隐含承诺**（"我承诺保 functionality 不变"）— 这是签字式的责任，AI 不能为团队做 commit。
- **既有行为的真正等价性** — LLM 写的等价改写在语义边界（race、异常路径、依赖反演）上不可证。
- **对历史 commit 链的"行为稳定"判断** — "这段代码曾经被无数次重新实现"只有老成员能讲，AI 没记忆。
- **跨子系统的行为契约** — 例如 ROS 节点行为在 costmap/控制器/tf 三处的契约，AI 模型即使读得完，也无法判断其中哪些是 *必需的契约* vs *偶然的耦合*。

### 4.4 AI 经常写错的地方

针对 ch1"改动 = 三道题"框架，AI 的典型误判：

| 错误模式                                       | 例子                                                                 | 后果 |
| ---------------------------------------------- | -------------------------------------------------------------------- | ---- |
| **混淆"不动"和"等价改"**                       | 把 `if (x == NULL) return -EINVAL;` "refactor" 成 `if (!x) return -EINVAL;` — 字面等价但语义上把"显式 NULL"换成"任何 falsy 值"，跨整型路径下不同的行为。 | 维护性看似更好，行为悄悄变（Takeaway 6 的 Refactor 行被悄悄违反）。 |
| **claim 功能不变，但漏 race / 异常路径**        | AI 重写一个文件级函数时，把 `pthread_mutex_lock/unlock` 优化成 lockless 版本，省略了竞态错误返回码。 | 测试套件全绿但生产偶发 dead lock / 数据竞争。 |
| **把"添加 feature"的 PR 当作 refactor 写**      | 用户要"加字段"，AI "顺便"调整了构造函数参数顺序和默认值（"让它更优雅"），声称只是 refactor。 | PR 的 invariant 矩阵行悄悄从 Adding 变成 Refactoring + Adding，测试覆盖按 Adding 套则 Refactoring 部分无覆盖。 |
| **用"读起来更短"作为保功能的证据**              | AI 给出"more idiomatic"改写，注释解释为行为不变 — 但因为没跑过对比测试就是没法验证。 | review 容易过，但用户复现一个 corner case 时发现行为不同。 |
| **改 PR description 来掩盖 invariant 失守**     | 真正的 refactor 改完产生了 perf 改善，AI 自动续写 description "Optimizes hot path"，但行为也被改了一点。 | release notes 显示是"纯优化"，出 bug 后排查反而被绕过。 |
| **伪造 characterization test 结果**             | LLM 写完测试，自动"验证"自己写的就是预期，把 OP 状态写死成实现的内嵌值。 | 测试 100% 通过但实际是空断言。 |
| **跳过 3-questions 中的"如何不破坏"**           | AI 写 PR 时只问 "如何做"和"做对了吗"，自动省略 "如何不破坏" — 因为这一步没法从 diff 直接生成。 | 回退不掉的 regression。 |

### 4.5 子段：AI 辅助遗留代码理解

适用本会话锁定视角（Linux 系统 + 机器人）。

- **Linux kernel / 系统 daemon 的遗留代码AI辅助**：
  - 用 AI 给一段 syscall handler 写一个"它什么时候返回什么 errno"的归纳 — 这就替代了几小时文档阅读。
  - 但 AI 对 syscall 表格的"边界即错"的判断不行 — 例如 `epoll_ctl` 在 EPOLL_CTL_DEL 时的 race，AI 经常推荐"等价修复"但实际掩盖 bug。
- **ROS/ROS2 节点的遗留代码 AI 辅助**：
  - 用 AI 把节点的 callback 列表 + publisher/subscriber 依赖画成图 → 替代手工 launch 文件分析。
  - 用 AI 给 costmap 层"什么会改变它的 footprint"做 spec-tour —— 这是行为定义，不是实现。
  - **关键限制**：behavior tree 的隐含优先级 + state machine 的合法转换，AI 给不出可信的最小不变量集。它能罗列 transitions，但不能告诉你"哪些是核心契约哪些是偶然"。

### 4.6 工程师必须保留的核心能力

- 判断改动属于 4-理由矩阵的哪一行（**必须人工** — 这是 commit message 的前置）。
- 写出能 *捕捉* 既有行为的 characterization 测试（不能只用 AI 生成的——必须人工验证它们不是空断言）。
- 评估一个看似等价的改写是否真等价 — AI 给的 "simplification" 默认需要被反编译一遍验证。
- 对 release note / PR description 做 invariant 维度的 sanity check（"Optimizing" 行不能附带 functionality 改）— 这是规约性责任。

## 五、实践行动项

> 本章是开篇概念章，没有演示代码。下面 4 项是把 ch1 的判断框架翻译成可重复的工程动作。

### A1 — 给一段 diff 跑"4-理由矩阵自检"

把 `git show HEAD` 的输出丢给 A2 里写的 `clf.py`（下面 A2 的微工具），或直接用 git 内置 fixture 走一遍：

```bash
# 零依赖 fixture: 用 git 自身的源码作为样本 (任何装有 git 的机器都行)
GIT_REPO=$(git --exec-path | sed 's|/libexec/git-core||')/..
# 或者用本机的临时仓库:
WORK=/tmp/ch01-self-check && rm -rf "$WORK" && mkdir -p "$WORK" && cd "$WORK"
git init -q
git config user.email "you@example.com" && git config user.name "you"
# 加一个会"假装 refactor 实际添加 feature"的提交 (正是 ch1 容易滑进去的反例)
cat > main.c <<'EOF'
int square(int n) { return n * n; }
EOF
git add main.c && git commit -q -m "feat: initial"
# 然后这提交故意声称是 refactor, 但偷偷加了 cube()
cat >> main.c <<'EOF'
int cube(int n) { return n * n * n; }
EOF
git add main.c && git commit -q -m "refactor: rename & cleanup"
git log --oneline
# 自己答: 此次提交变化是 structure / new functionality / existing functionality / resource 中的哪几个？
#   - 这次在 4×4 矩阵 (Adding/Bugfix/Refactor/Optimize × Structure/New Func/Existing Func/Resource)
#     中实际声称保的列是什么？
#   - 准则: 声称的"保"必须有对应 regression test 指向它, 否则该列就是 silently violated。
```

**验收**：
- `git log --oneline` 拿到 2 个 commits。
- 把"refactor: rename & cleanup" 提交按 4-理由矩阵分类（答案：因为加了 `cube()`，应判为 `Adding a Feature`，不是 `Refactoring`）。
- 列出这个仓库里还有哪些"声称 refactor 实际 adding"的类似 PR，标注 `UNCOVERED` 列。

### A2 — 写一个微型的 4-reason 矩阵打分工具

```bash
# prompts/clf.py — 一个 50 行的 Python 工具，用关键字+diff 信号判断 PR 类型
# 规则草案（手册规则，AI 调用作参考而非依赖）：
#   - diff 行数 < 10 且新 > 旧（含函数/方法签名）             → Adding a Feature
#   - diff 主要改控制流、变量名、抽函数, 无新增依赖/接口       → Refactoring
#   - diff 含 loop 改写、cache、asymptotic 优化, 无新接口       → Optimizing
#   - 其它                                                    → Fixing a Bug
# 输出：矩阵中那一行 + 自检 checklist。
```

> **不要让 AI 决定**，AI 只能辅助。本工具输出 PR 行 + checklist 之后，仍由人判断 invariant 列。

**验收**：

```bash
python3 prompts/clf.py path/to/diff.patch
# 输出一行 PR-type + 4-7 行的 invariant checklist
# 你手工改一个 refactor PR 让它"偷偷加了 feature"，再跑工具，确认工具还能警告（不是默认放行）
```

### A3 — 找一个自己仓库的"jumping off cliff"模块做声明

```bash
# 在你维护的项目里找出至少一个模块，让 3 个团队成员各自独立给它打分 1-5：
#   (1) 修改该模块的恐惧度
#   (2) 改该模块的回归测试覆盖密度估计
#   (3) 该模块的"声称保的 invariant"是否能立刻列出
# 三个分数都 ≤ 2 的模块即候选做 ch2 之后的 characterization test 实施对象。
```

**验收**：输出一个 `legacy-cliff-candidates.md`，列 ≥1 个候选模块 + 3 人评分 + 下一步建议（接 ch2 Working with Feedback 的 characterization test 流程）。

### A4 — 写一个最小 C 程序，复刻 Figure 1.1 的不变式矩阵 (Linux 系统)

```c
// samples/ch01/invariant-matrix.c
// 演示：在 changelog 提交信息里强制作者声明 4-理由矩阵中保住的列
// 编译：gcc -std=c17 -Wall -Wextra -o invariant-matrix samples/ch01/invariant-matrix.c
// 运行：./invariant-matrix                                          （打印一份模板 message）
//       ./invariant-matrix <word> <col:change|hold> ...             （分类 + 校验）

#include <stdio.h>
#include <string.h>

static const char *ROWS[] = {"adding", "fixing", "refactor", "optimizing"};
static const char *COLS[] = {"structure", "new-functionality",
                              "existing-functionality", "resource-usage"};

static int row_index(const char *msg) {
    for (int i = 0; i < 4; i++) {
        /* 单词边界: 行关键字前后必须是空格或字符串首尾 */
        char needle[32];
        snprintf(needle, sizeof(needle), " %s ", ROWS[i]);
        if (strstr(msg, needle)) return i;
        if (strncmp(msg, ROWS[i], strlen(ROWS[i])) == 0 &&
            (msg[strlen(ROWS[i])] == ' ' || msg[strlen(ROWS[i])] == '\0'))
            return i;
    }
    return -1;
}

/* 要求 message 里声明所有 4 列 — 否则报错, 强制作者填完整 */
static int check_columns_declared(const char *msg) {
    int missing = 0;
    for (int i = 0; i < 4; i++) {
        char needle[64];
        snprintf(needle, sizeof(needle), "%s:", COLS[i]);
        if (!strstr(msg, needle)) {
            fprintf(stderr, "ERR: 缺少列声明 '%s:'。\n", COLS[i]);
            missing++;
        }
    }
    return missing != 0;
}

/* Refactor / Optimize 行禁止改动 existing functionality
 * — 与 ch1 Takeaway 6 矩阵一致。 */
static int check_invariant(const char *msg) {
    int r = row_index(msg);
    if (r < 0) return 0;                       /* 未知行不强制 */
    int changed_existing = strstr(msg, "existing-functionality:change") != NULL;
    if ((r == 2 || r == 3) && changed_existing) {
        fprintf(stderr, "ERR: '%s' 行不应改动 existing-functionality。\n",
                ROWS[r]);
        return 1;
    }
    return 0;
}

int main(int argc, char **argv) {
    if (argc < 2) {
        printf("# 提交模板:\n");
        printf("#   <row> <col>:<change|hold> ...\n");
        printf("#   例: refactor structure:change new-functionality:hold "
               "\\\n#        existing-functionality:hold resource-usage:hold\n");
        return 0;
    }
    char buf[1024] = {0};
    for (int i = 1; i < argc; i++) {
        strncat(buf, argv[i], sizeof(buf) - strlen(buf) - 2);
        strncat(buf, " ", sizeof(buf) - strlen(buf) - 2);
    }
    if (check_columns_declared(buf)) return 2;
    return check_invariant(buf);
}
```

```bash
mkdir -p /tmp/ch01 && cd /tmp/ch01
# 复制 samples 里那份 invariant-matrix.c
cp /home/koh/dev/notes/working-legacy-code-2005/samples/ch01/invariant-matrix.c .
cc -std=c17 -Wall -Wextra -o invariant-matrix invariant-matrix.c
# 合法
./invariant-matrix refactor structure:change existing-functionality:hold \
                    new-functionality:hold resource-usage:hold
echo "rc=$?"   # 0

# 非法: refactor 行声称 existing-functionality:change
./invariant-matrix refactor structure:change existing-functionality:change \
                    new-functionality:hold resource-usage:hold
echo "rc=$?"   # 1

# 不全声明 4 列 → rc=2
./invariant-matrix refactor structure:change
echo "rc=$?"   # 2
```

**验收**：
- 上面 4 行命令全部在 `/tmp/ch01` 下 `bash` 即可`rc=0/1/2` 顺序对。
- `cat invariant-matrix.c | wc -l` > 50 (确认文件确实写了，不是空文件兜底)。

> ⚠️ 这只是 ch1 概念章的极简回声 — 后续 ch2 会引入真正的"feedback = tests"工具。

## 六、值得深入思考的问题

### Q1: 把改动归到 4-理由矩阵的"哪一行"由谁决定？

如果 PM 决定（卡 / ticket 标签），那么 technical 团队的判断被绕过；如果 tech lead 决定，PM 的 backlog 就和实际改动脱节；如果 AI 自动决定，标签不可信。**关键问题**：这个分类责任如何在流程上挂在 *能够理解既有行为* 的人身上？

### Q2: Refactor 和 Optimize 的边界常常塌陷

二者都"保 functionality"，Optimize 改 resource、Refactor 改 structure。但实际生产中我们做的 70% 都是混合的（顺手把代码改了又顺手把算法换了）。**关键问题**：Feathers 是不是低估了"严格 refactor"的难度？是不是该默认承认没有*纯* refactor 这种东西？

### Q3: "用什么知道没破坏"在没有覆盖的 legacy 里是空集

第三题的答案在 ch2 是"写测试"，但 legacy 的悖论 = "改不了所以没测 / 没测所以不敢改"。**关键问题**：测试债的形成是渐进的、偿还也应是渐进的，那么一个组织应该在哪个时间点开始 *部分* 偿还，而不是等"有 6 个月空闲"才动手？

### Q4: 行为保留 vs 行为增强在版本号上等价吗？

semver 假设"行为不变"对应 patch、"行为增强"对应 minor。**关键问题**：能不能形式化"行为不变"到机器可验证？这是不是 ROS 节点级 contract testing 的工业意义所在？还是说形式化行为永远只对极小子集可行？

### Q5: AI 主导 PR 流的现实里，Takeaway 6 矩阵这一端还是工程端的工具？

AI 写 80% 的 PR，但分类与 invariant 声明仍然由人写。**关键问题**：我们是不是在制造一个"AI 的 prompt 工程 = 老的工程师精神模型"的人力市场？这一代的矩阵是否会自动塌陷成空话？

---

*下一章预告*: **Chapter 2 — Working with Feedback** —— 第二/三题"改对 + 没改坏"的工程答案：unit test 的位置（不是覆盖率而是与设计的耦合）、characterization test 写在遗留代码上的具体节奏，以及一条贯穿全书的 Legacy Code Change Algorithm。

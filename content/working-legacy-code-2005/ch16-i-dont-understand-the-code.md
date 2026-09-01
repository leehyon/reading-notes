# Chapter 16 — I Don't Understand the Code Well Enough to Change It

> **PDF**: p.231-236（6 页）
> **定位**: ch15 教你 *识别自己代码* 的核心逻辑；ch16 转向 *理解别人的代码*。Feathers 给的不是 IDE 工具，是四个**低成本、低技术、慢但有效**的人类技巧：**Notes / Sketching**（随手画图） + **Listing Markup**（打印代码用笔画块） + **Scratch Refactoring**（checkout 一份丢进去乱改，看完扔掉） + **Delete Unused Code**（删！VCS 是 backup）。这章的隐线：**理解代码不需要懂代码，要先动起来**。

## 〇、第一性原理思考

**这章做了什么**: 针对"我读不懂这段代码"的恐惧, 给出 4 个不需要 IDE / 编译 / 测试的低技术技法 —— Sketching(纸笔画图)、Listing Markup(打印代码用笔画块)、Scratch Refactoring(checkout 一份乱改看完扔掉)、Delete Unused Code(删了 VCS 看得到)。

**为什么这样拆**: Feathers 不用工具解决理解问题, 反而把"理解"动作物化、可见化、可 push back —— 4 个技法的共同点是输出都是可见物(纸/打印件/working copy/commit log), 所以"我懂没懂"不再靠自我感觉。

**最值钱的洞见**: 实际上理解不是一次性读完的状态, 而是 activity —— scratch refactor 的两个风险(错认 / attachment)暴露的是"理解必须可丢弃", 不 check in 就是这条规则的物理保障。
## 一、章节概述

- **进入陌生代码的恐惧**："你不知道改动是 5 分钟还是 5 天"——ch1 的 central dilemma 在每个工程师身上重演。Feathers 不给 IDE 教程，给的是 *4 个让你开始动手* 的技巧。
- **Notes / Sketching**：拿笔在纸上 / 备忘录背面画图。Feathers 例子：图 16.1 是他在 memo 背面画的——一团 Level A / Level B / dispatcher / offline / local / remote + C Child —— 别人看不懂，但 *当场的两个人对话够用了*。
- **画图的"传染性"**：不用推团队采用。等你在跟人 pair 时随手画，对方会被带动。
- **Listing Markup**：打印代码 + 用笔画。4 个目的：
  - **Separating Responsibilities**：同类语句用同色标。
  - **Understanding Method Structure**：长方法的 block 从内向外对齐（标 brace pair）。
  - **Extract Methods**：圈出想抽的代码 + 写 *coupling count*（ch22 详谈）。
  - **Understand the Effects of a Change**：在要改的行做标记 → 标出会受影响的变量 / 方法调用 → 标出会受那些变量影响的东西……直到传播停止。
- **Scratch Refactoring** —— ch16 的明星技法：从 VCS checkout → 不用写测试 → 随意抽 method / 移动变量 / 重命名 → *不 check in* → 看完扔掉。**两个风险**：(a) 抽出错的方法 → 误以为代码做了什么别的；(b) 对最终结构产生 attachment。
- **Delete Unused Code**：判断某段代码没用 = 删。VCS 是历史；不要怕删。
- **ch16 不教任何自动化工具**。它的反话："Spending time trying to understand something looks and feels suspiciously like not working." 看完 ch16 你应该知道"理解"是 *可观察的活动*，不是"读完所有代码"的神话。
- **Chapter 16 的隐含承诺**：理解 ≠ 一次性完成。理解 = 反复 sketch + 反复 refactor scratch + 反复删——直到你"敢动"。

## 二、核心 Takeaways

### Takeaway 1: 理解 ≠ 一次读完。理解 = 反复动起来。

- **是什么**：Feathers 给的核心 insight。读代码时被恐惧缠住（"我读不完怎么办"），不如 *开始动*：画图 / 标 / 抽 method / 删。**理解是 activity，不是 state。**
- **为什么重要**：打破"我得先全懂才能改"的瘫痪循环。
- **解决什么问题**：让"我不懂"从 *freeze* 变成 *kickoff*。
- **适用场景**：新人进 legacy 仓库；trail-and-error 的第一次接触。

### Takeaway 2: Sketching — 纸 + 笔，5 分钟可见效

- **是什么**：拿笔画图。Figure 16.1 那个例子——Level A / Level B / dispatcher / offline / local / remote + C Child——别人看不懂，但当场对话够用。**画图的语言不必 UML**，"blob + 线条" 就够。
- **为什么重要**：画图把"脑内模型" 投影到外部，让别人能 challenge。你说"A 调 B"，画出来发现 A 实际调 C。
- **解决什么问题**：让 pair 工作时的对话从"我脑子里的图你能想象吗" 变成 "看这"。
- **适用场景**：onboarding；排查跨文件 bug；review 一段陌生代码。

### Takeaway 3: Listing Markup — 4 种目的对应 4 种笔法

- **是什么**：打印代码 + 用笔画。Feathers 列 4 个目的：
  1. **Separating Responsibilities** — 同色标同类语句。
  2. **Understanding Method Structure** — 长方法的 block 从内向外对齐。
  3. **Extract Methods** — 圈代码 + 写 coupling count。
  4. **Understand the Effects of a Change** — 在要改的行做标记，扩散到变量/方法调用，直到传播停止。
- **为什么重要**：它把"理解"动作 *物化* —— 不必再问"我懂了没"，看 markup 的覆盖密度就懂。
- **解决什么问题**：让"我以为我懂" 暴露成"这行我没标 → 我不确定"。
- **适用场景**：处理长方法；评估改动影响面；ch22 大方法的预演。

### Takeaway 4: Scratch Refactoring — checkout 一份，看完扔掉

- **是什么**：从 VCS checkout → 不用写测试 → 随意抽 method / 移动变量 / 重命名 → *不 check in* → 看完扔掉。**两个风险**：
  1. **错认**：抽出错的方法 → 误以为代码做了别的 → 后续真的改基于错误模型。
  2. **attachment**：对最终结构产生 attachment → 失去后续 refactor 的洞察。
- **为什么重要**：Scratch 是 *理解工具*，不是 refactor 计划。**核心约束**：不 check in。
- **解决什么问题**：让"乱抽 method" 安全 —— 因为不必保存结果。
- **适用场景**：评估一段复杂代码；ch22 的预演；新人自学。

### Takeaway 5: Delete Unused Code — VCS 是 backup

- **是什么**：判断某段代码没用 = 删。VCS 保存历史。**反对意见**："有人花时间写的"——答："git 看得到"。
- **为什么重要**：删代码是 *低成本理解动作* —— 删的过程要 trace 调用链，trace 完就懂。
- **解决什么问题**：让 legacy 里 "看起来 dead 但不敢删" 的代码消失。
- **适用场景**：refactor 前；代码 review；trail-and-error。

### Takeaway 6: 4 个技法的共同特征 = 低技术 + 可观察

- **是什么**：4 个技法都不需要 IDE / 工具 / 安装。纸 + 笔 + VCS + 打印。**它们的输出是 *可见物*** —— sketch 在纸上，markup 在打印件上，scratch refactor 在 working copy 上，deletion 在 commit log 上。
- **为什么重要**：可观察 = 可教学 / 可 push back / 可复盘。
- **解决什么问题**：让"理解" 动作可被同事看到，不只是脑内。
- **适用场景**：onboarding；新人 pair 工作；legacy refactor 会议。

### Takeaway 7: Scratch Refactoring 的"两个风险" = 工程规约

- **是什么**：
  - 风险 1：错认（refactor 出错 → 误以为系统做了什么别的）。
  - 风险 2：attachment（对最终结构 attachment → 失去后续洞察）。
- **为什么重要**：把"乱改"的安全边界画清楚——*不 check in* 是关键规约。
- **解决什么问题**：让团队对 scratch refactor 有规约可循，不是"野路子"。
- **适用场景**：工程文化；新人培训。

### Takeaway 8: 理解 = 与代码 *交谈*，不是 *阅读*

- **是什么**：Feathers 整章隐线：理解代码 = 反复 *提问 + 验证*。Sketching 是把"我脑子里的问题" 变成图；markup 是把"我打算改什么" 标到代码；scratch refactor 是把"代码如果这样写" 演练一遍；delete 是把"这段真的没人用吗" 试一下。
- **为什么重要**：把"理解" 从静态阅读变成动态对话。
- **解决什么问题**：让"读代码慢" 不再是个人缺陷，而是工程方法选择。
- **适用场景**：任何 *陌生代码 + 改的需求* 场景。

### Takeaway 9: ch16 与 ch17 的接力

- **是什么**：ch16 = 微观技法（4 个技法对单段代码 / 单文件）。ch17 = 系统层技法（"我不懂整个应用的架构"）。**ch17 把 ch16 的 sketch 推到 *Telling the Story of the System***。
- **为什么重要**：从"读单文件" 到"理解系统" 的递进。
- **解决什么问题**：把"我不懂整片代码" 也变成可解的工程问题。
- **适用场景**：长期项目；新人 trailing。

## 三、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **kernel 开发者的 sketch = ftrace / bpftrace 现场图**。当一个 syscall 行为奇怪，开发者不会"读完所有 call path"——他们用 `ftrace` / `bpftrace` 抓一次 stack trace，*当场画出"哪几个函数被调" 的图*。这正是 ch16 sketch 的工业级映射：从脑内 → 笔 → 工具输出。
- **systemd / dbus 的 listing markup**。这两者的代码库 *长度* 不算夸张（systemd ~1M LOC），但单个文件常 500-2000 LOC。一个 patch 涉及多个文件时，开发者打印 + highlight 关键行 = 工业版 listing markup。**真实工作流**：很多人会拿 iPad / Surface 做 markup 而不是打印——但动作形态一致。
- **kernel scratch refactoring = 改完测一遍然后 revert**。kernel 不接受 scratch commit 进 mainline，但 maintainer 经常本地改一堆看 kernel 行为，然后 `git reset --hard origin/master` 丢弃。**scratch refactoring 的工程版本 = exploratory debugging**。
- **delete unused code 在 kernel 经常是 RSD / DT 节点 / syscall**。kernel 维护者过去会"保留未使用 syscall 以防未来"——结果遗留大量 dead code。社区现在转 *"我们不保留死代码；future use 不构成保留理由"*。**VCS = backup** 在 kernel 是真理。
- **"syscall 行为有 race" 的 listing markup 实战**：开发者打印 `epoll_wait` 实现 + 用 4 色标 4 个 race 入口（`ep_poll_callback` / `__remove_wait_queue` / `list_del_init` / `wake_up`）——4 个目的（Separating / Structure / Extract / Effects of Change）全部命中。

#### Linux 系统 — ch16 技法映射表

| Feathers 技法      | Linux 系统等价做法                          | 工具 / 输出                          |
| ------------------ | ------------------------------------------- | ------------------------------------ |
| Notes / Sketching  | ftrace / bpftrace 输出 + 现场手绘图         | `bpftrace -e 'kprobe:... { @[comm]=count(); }'` |
| Listing Markup     | 打印 + 高亮 / iPad markup                   | `git diff` + manual highlight         |
| Scratch Refactoring| 改完测一遍 + `git reset --hard` 丢弃         | `git stash` + exploratory patch        |
| Delete Unused Code | remove dead syscall / DT node / RSD entry    | `git rm` + commit                     |

### 3.2 机器人软件视角

- **ROS 节点的 listing markup**：ROS 节点常见 500-1500 LOC（一个 navigation 节点）。新人要改某个 behavior 时，打印 + 用 3 色标 3 个 layer（perception / planning / control）。**这是 ch16 listing markup 的工业版**。
- **Nav2 的 scratch refactoring**：`nav2_bt_navigator` 复杂（BT XML + C++ 引擎 + ROS action）。开发者本地改 `nav2_bt_navigator` 看 BT 行为，不 commit 进 mainline → scratch 实战。
- **MoveIt2 的 sketch = planning scene 现场图**。MoveIt 调试时开发者 *画 collision object / attached object / world* 的位置关系图——ch16 sketch 的几何版本。
- **ros2_control 的"delete unused joint"**。机器人项目迭代中"加 sensor 后又移除"很常见；保留 dead joint 让 hardware interface 复杂化。**delete unused code 的工程实例**。
- **机器人 multi-node 系统的 *Telling the Story* 提前到 ch16**。机器人系统是天然的 multi-process（ROS launch 启动 N 个 node）；理解 = 用 sketch 画 launch 文件 → 节点 → topic 的图。
- **Nav2 BT 调试的 scratch refactoring**：开发者改 BT XML 试 condition 顺序，看 navigation 表现是否更好。改的 XML 不一定 commit（可能用作 issue 反馈）；scratch 行为。

#### 机器人软件 — ch16 技法映射表

| Feathers 技法      | 机器人软件等价做法                       | 工具 / 输出                          |
| ------------------ | ---------------------------------------- | ------------------------------------ |
| Notes / Sketching  | launch 文件 + topic 关系图 + BT XML 草图 | mermaid / 纸笔 + rviz 数据流          |
| Listing Markup     | ROS 节点代码 + 4 色标 4 个 layer         | 打印 / iPad + `rqt_graph` 高亮        |
| Scratch Refactoring| 改 BT XML / 节点参数 → 看 rviz 行为     | ros2 launch + `ros2 bag` play        |
| Delete Unused Code | remove dead topic / param / launch arg   | `ros2 param delete` + `git rm`       |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                   | 高级工程师                                                   |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 读陌生代码           | 反复从头读                                                    | 第一动作 = 打印 + markup（动作化）                          |
| 怕改坏              | 不敢动                                                      | checkout scratch branch → 改 → 验证 → revert               |
| 死代码             | 不敢删，"也许以后用"                                          | "git 看得到，删"                                             |
| Sketch               | 不画                                                          | 随手画（5-10 分钟可见效）                                    |
| 长方法              | 一直读直到"读完"                                              | 打印 + 从内向外对齐 brace                                    |
| 改动影响评估         | 凭直觉                                                        | listing markup "Effects of Change" 标签                     |
| 与新人 pair 时      | 给代码不给解释                                                | 边画图边讲                                                  |
| 风险观感             | "读完再改" 是好习惯                                            | "读完" 是神话                                              |

> **关键差异**：高级工程师把"理解"动作化、可见化；初级把它当成"读完"的神话状态。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

仍然极重要。原因：
- **AI 不能替你"理解"**——AI 给总结 = 文本版的脑内模型，但 Feathers 整章的隐线是"理解是动作"——AI 给不了动作。
- **AI 让"读代码"变得更慢**——AI 输出长篇解释，工程师沉入阅读而不是动作。**ch16 的反 AI 思路**：停止阅读，开始动手。
- **Listing markup 在 AI 时代更显重要**——AI 给的总结常有 hallucination，唯一保险是人工标 + 自己 trace。
- **Scratch refactoring 在 AI 时代被异化**——AI 给"refactor 建议"，但建议 ≠ scratch。AI 建议要 commit 才"有用"，但 scratch 不 commit。AI 不会自动 scratch。

### 4.2 AI 已经能做的（具体到 ch16 主题）

- **生成 sketch 草图**：给一段代码，AI 输出 mermaid / dot graph 当场使用。
- **生成 listing markup 提示**：识别 method 长度 + complexity，推荐 highlight 哪些行。
- **自动 scratch refactor + revert**：用 AI agent 在 scratch branch 上乱改，运行测试，看是否 broken，报告给工程师。**风险**：AI 自动 revert = scratch 不是真的 scratch。
- **识别 dead code**：AI 静态扫描 + 给"这段代码 N 月没被引用，建议删除"。
- **给"Effects of a Change" 影响图**：AI 给一个 method 的 call graph + 标红所有受影响的 caller。

### 4.3 AI 不能替代的（具体到 ch16 主题）

- **替代不了"画图思考"**：sketching 的核心是 *边画边想*，不是图本身。AI 给的图是"已完成的图"，丢了"画的过程"。
- **替代不了 scratch refactor 的"出错体验"**：scratch 抽错 method 时 *你自己* 才学到代码结构。AI 给"正确 refactor" = 没学到。
- **替代不了 delete 的 *决定过程***：AI 标 dead code = 候选；删不删是工程 judgment（看 commit log / 看 blame）。
- **替代不了 listing markup 的 *物化* 动作**：高亮代码 = 思考物化，AI 给高亮建议 ≠ 真的高亮。

### 4.4 AI 经常写错的地方

针对 ch16 "Notes/Sketching/Listing Markup/Scratch Refactor/Delete Unused Code" 主题：

| 错误模式                                       | 例子                                                                 | 后果 |
| ---------------------------------------------- | -------------------------------------------------------------------- | ---- |
| **AI 给"完美的 sketch"，丢了"画的过程"**         | AI 输出 clean mermaid 图，工程师用它讲解——但 *没画* 的思考过程       | 团队失去"画的过程"中的洞察                          |
| **AI 自动 scratch refactor 后 commit**          | AI 看到代码乱，改完，commit 到 mainline                               | scratch 的安全边界被破；下次 refactor 不知基准是什么 |
| **AI "识别 dead code" 标错**                   | AI 说 `old_method` 没人用，但有动态调用 (`__getattr__` / reflection) | 删了 runtime 崩                                       |
| **AI 生成的 listing markup 推荐 highlight 错行** | AI 用 static analysis 推荐高亮，但工程师关注的 control flow 行没标    | markup 形同虚设                                       |
| **AI "Effects of a Change" 影响图漏 race**      | AI 标 caller graph 但漏了 thread context / signal handler            | listing markup 给假安全感                            |
| **AI 拒绝删 dead code**                         | AI 标 dead 后说"也许未来用，先保留"——等于没标                        | dead code 持续积累                                    |
| **AI 自动生成 *coupling count* 不准**            | AI 给 method 的 coupling = 5，实际 = 12                              | ch22 估算失真                                          |
| **AI 把 scratch refactor 写成 *production plan*** | AI 输出 "建议抽 method X / Y / Z" 列入 backlog                       | scratch 的 "看完扔掉" 精神丢失                       |
| **AI sketch 抽象度过高**                        | AI 生成的图过于"标准 UML"，新人看不懂，反而没帮助                   | sketch 失去"自由 blob + 线" 的对话能力              |
| **AI "画图" 实际是写 markdown 表格**            | AI 给"类 A 调用 B 调用 C" 的 markdown table 当 sketch               | 没图形 = 不算 sketch                                  |

### 4.5 子段: AI 辅助遗留代码理解

适用本会话锁定视角（Linux 系统 + 机器人）。

- **Linux kernel / 系统 daemon 的 ch16 AI 辅助**：
  - AI 给一个 syscall handler 输出 mermaid 图（call stack 形式）。**风险**：图反映 *单次 trace*，不代表所有路径。要人工加条件分支。
  - AI 推荐 highlight 哪些行 = 提议，但工程师要亲自画（动作化不可替代）。
- **ROS/ROS2 节点的 ch16 AI 辅助**：
  - AI 给一个 navigation 节点生成 launch file 关系图。**好**：加速 onboarding。**坏**：图反映 launch 文件实际拓扑，不反映节点内部状态机。
  - AI 标 dead topic / param —— 但 ROS2 允许 dynamic subscription，AI 看不到动态创建的 subscriber。
- **AI 的 ch16 角色是 *启动器***，不是 *替代者*。AI 给"画什么 / 标什么 / scratch 改什么" 的建议，但工程师要 *做* 这些动作。

### 4.6 工程师必须保留的核心能力

- **动作化的理解**：读代码时同时打印 / 画图 / scratch。AI 给文本 ≠ 动作。
- **Scratch 的纪律**：scratch = 不 commit。AI 给的 refactor 建议当 scratch = 不 commit。
- **判断 dead code 的勇气**：AI 标 dead ≠ 删除；要看 blame + commit history + reflection 风险。
- **Listing markup 的物化思维**：打印代码 + 高亮 = 思考物化。AI 给高亮建议 ≠ 真的高亮。
- **Effects of a Change 的多跳扩散**：手动标 1 跳 → 2 跳 → 3 跳 直到停止。AI 给 caller graph 但漏 race / reflection / dynamic dispatch。

## 五、实践行动项

> ch16 的 4 个技法都是 *人 + 笔 / 纸 / VCS*。下面 4 个行动用 bash / python / git 把这些动作机械化、便于跑通。

### A1 — Notes / Sketching: 给一段陌生 C 代码生成 mermaid 图

> 复刻 ch16 Figure 16.1 那种 *blob + line* 风格 —— 不必 UML，能让 pair 工作对话够用就行。

```bash
mkdir -p /tmp/ch16-scratch && cd /tmp/ch16-scratch

# 准备一段"陌生代码" — 我们让它复杂一点
cat > legacy_nav.c <<'EOF'
/* 假想的 navigation 节点 — 5 个函数 + 一些交叉调用 */
#include <stdio.h>

static int read_sensor(int id) {
    printf("read sensor %d\n", id);
    return id * 10;
}

static int fuse_sensors(int a, int b) {
    return (a + b) / 2;
}

static int plan_path(int start, int goal) {
    int cost = 0;
    for (int i = start; i < goal; i++) cost += i;
    return cost;
}

static int execute(int path_cost) {
    return path_cost > 0 ? 1 : 0;
}

int main_loop(int sensor_a_id, int sensor_b_id, int start, int goal) {
    int a = read_sensor(sensor_a_id);
    int b = read_sensor(sensor_b_id);
    int fused = fuse_sensors(a, b);
    int path = plan_path(start, goal);
    int ok = execute(path);
    (void)fused;
    return ok;
}
EOF

# 用 grep + sed 把每个函数 + 调用关系抠出来
cat > sketch.sh <<'EOF'
#!/bin/bash
# sketch.sh — 给一个 C 文件, 生成 mermaid 类图
# 提取每个 function 名, 提取每个调用别的 function 的位置
# 输出 mermaid 语法 (可在 mermaid.live 渲染)
F="$1"
echo "graph LR"
echo "  main[main]"
# 提取所有 static 和非 static 函数
FUNCS=$(grep -E '^(static\s+)?[a-zA-Z_]+\s+[a-zA-Z_]+\s*\(' "$F" \
        | sed -E 's/.*\s([a-zA-Z_]+)\s*\(.*/\1/' | sort -u)
# 每个函数生成一个节点 + label
for FN in $FUNCS; do
    echo "  ${FN}[\"$FN\"]"
done
# 提取每个函数体里的调用
IN_FN=""
while IFS= read -r LINE; do
    if echo "$LINE" | grep -qE '^(static\s+)?[a-zA-Z_]+\s+[a-zA-Z_]+\s*\('; then
        IN_FN=$(echo "$LINE" | sed -E 's/.*\s([a-zA-Z_]+)\s*\(.*/\1/')
        continue
    fi
    if [[ -n "$IN_FN" ]]; then
        for FN in $FUNCS; do
            if echo "$LINE" | grep -qE "\\b$FN\\s*\\(" && \
               [[ "$LINE" != *"$IN_FN"* || "$FN" != "$IN_FN" ]]; then
                echo "  $IN_FN --> $FN"
            fi
        done
    fi
done < "$F"
EOF
chmod +x sketch.sh

./sketch.sh legacy_nav.c
echo "---"
echo "(copy-paste 上面的 mermaid 输出到 https://mermaid.live 看图)"
```

**验收**：
- 输出 mermaid `graph LR` + 节点 + 箭头
- 图显示 `main_loop --> read_sensor` / `main_loop --> fuse_sensors` / `main_loop --> plan_path` / `main_loop --> execute` / `execute --> read_sensor`（如有）等关系
- 这就是 ch16 Figure 16.1 的"自由 blob + 线" 风格的工业版

### A2 — Listing Markup: 模拟打印 + 高亮长方法

> 演示 ch16 提到的 4 个 markup 目的 —— 长方法的 brace 对齐 + 责任分组 + extract 候选 + Effects of Change 扩散。

```bash
mkdir -p /tmp/ch16-scratch && cd /tmp/ch16-scratch

cat > long_method.c <<'EOF'
/* 长方法 — ch22 那一类, 这里做 demo 用 */
#include <stdio.h>
int process_all(int *arr, int n, int mode) {
    int sum = 0;
    int max = 0;
    int min = 0;
    int err = 0;
    if (mode == 1) {
        for (int i = 0; i < n; i++) {
            sum += arr[i];
            if (arr[i] > max) max = arr[i];
            if (arr[i] < min) min = arr[i];
        }
    } else if (mode == 2) {
        for (int i = 0; i < n; i++) {
            sum += arr[i] * 2;
            if (arr[i] % 7 == 0) err++;
        }
    } else {
        return -1;
    }
    printf("sum=%d max=%d min=%d err=%d\n", sum, max, min, err);
    return sum;
}
EOF

cat > markup.sh <<'EOF'
#!/bin/bash
# markup.sh — Listing Markup 的 4 个目的
# 1. Separating Responsibilities: 同色标同类语句
# 2. Understanding Method Structure: brace 对齐
# 3. Extract Methods: 圈代码 + coupling count
# 4. Effects of Change: 标记受影响行
F="$1"
echo "=== 1. Separating Responsibilities (按语句类型分色) ==="
echo "[ASSIGN] sum/max/min/err 是状态聚合"
grep -nE '\s+(sum|max|min|err)\s*([+\-*/%]=?)' "$F" | head -10
echo "[CONTROL] if/for 是控制流"
grep -nE '\s*(if|else|for)\b' "$F"
echo "[IO] printf 是输出"
grep -nE 'printf\(' "$F"

echo
echo "=== 2. Method Structure (brace 对齐 — 第 1 个 open 找 match) ==="
# 简易 brace balance
awk '
/\{/  { for(i=1;i<=length($0);i++) if(substr($0,i,1)=="{") depth++; print NR": +"depth" "$0; next}
/\}/  { for(i=1;i<=length($0);i++) if(substr($0,i,1)=="}") depth--; print NR": -"depth" "$0; next}
{print NR":      "$0}
' "$F" | head -25

echo
echo "=== 3. Extract Methods (按 block 选候选) ==="
# 找出独立的 for + 周边 4 行作为抽 method 候选
echo "block: for (int i...) loop  → 候选抽 method 'accumulate_stats'"
grep -nE 'for \(int i = 0' "$F"

echo
echo "=== 4. Effects of Change (改 printf 那行, 看下游影响) ==="
echo "line N: printf(...)  → 影响 stdout (无 caller)"
echo "       sum return value → 影响 process_all 的所有 caller"
grep -nE 'process_all' "$F"
EOF
chmod +x markup.sh

./markup.sh long_method.c
```

**验收**：
- 4 个 section 都输出对应信息
- 这是 ch16 Listing Markup 4 个目的的 bash 机械化版

### A3 — Scratch Refactoring: 抽出 method 看，然后 git reset 扔掉

> 复刻 ch16 核心技法 —— checkout scratch，乱抽 method，看完 revert。

```bash
mkdir -p /tmp/ch16-scratch && cd /tmp/ch16-scratch

# 准备一个 git 仓库 + 长方法
rm -rf scratch-demo && mkdir scratch-demo && cd scratch-demo
git init -q
git config user.email "you@x" && git config user.name "you"

cat > main.c <<'EOF'
#include <stdio.h>
int process_all(int *arr, int n, int mode) {
    int sum = 0, max = 0, min = 0, err = 0;
    if (mode == 1) {
        for (int i = 0; i < n; i++) {
            sum += arr[i];
            if (arr[i] > max) max = arr[i];
            if (arr[i] < min) min = arr[i];
        }
    } else if (mode == 2) {
        for (int i = 0; i < n; i++) {
            sum += arr[i] * 2;
            if (arr[i] % 7 == 0) err++;
        }
    } else {
        return -1;
    }
    printf("sum=%d max=%d min=%d err=%d\n", sum, max, min, err);
    return sum;
}
EOF
git add . && git commit -q -m "initial: long method"
echo "=== 1) mainline 状态 ==="
git log --oneline

# === Scratch refactor ===
echo
echo "=== 2) 抽出 accumulate_stats_helper, 不 commit ==="
# 改成更清晰的结构 — *scratch 阶段不 commit*
cat > main.c <<'EOF'
#include <stdio.h>
static void accumulate_stats(int *arr, int n, int *sum, int *max,
                             int *min) {
    *sum = 0; *max = 0; *min = 0;
    for (int i = 0; i < n; i++) {
        *sum += arr[i];
        if (arr[i] > *max) *max = arr[i];
        if (arr[i] < *min) *min = arr[i];
    }
}
int process_all(int *arr, int n, int mode) {
    int sum = 0, max = 0, min = 0, err = 0;
    if (mode == 1) {
        accumulate_stats(arr, n, &sum, &max, &min);
    } else if (mode == 2) {
        for (int i = 0; i < n; i++) {
            sum += arr[i] * 2;
            if (arr[i] % 7 == 0) err++;
        }
    } else {
        return -1;
    }
    printf("sum=%d max=%d min=%d err=%d\n", sum, max, min, err);
    return sum;
}
EOF
cc -std=c17 -Wall -Wextra -c main.c -o main.o && echo "scratch refactor compiles OK"
echo "git status:"
git status -s
echo "(注意: git status 显示 modified — 但我们 *不 commit*)"

echo
echo "=== 3) git reset --hard 扔掉 scratch ==="
git reset --hard --quiet
git log --oneline
echo "main.c 现在是:"
head -5 main.c
echo "..."
```

**验收**：
- `git log --oneline` 仍只有 `initial: long method`（scratch 没 commit）
- `main.c` 恢复原样（`head -5` 显示原 long method 第一段）
- 这就是 ch16 核心技法 —— scratch 看完扔掉，mainline 不动

### A4 — Delete Unused Code: 检测 + 真删（git 是 backup）

> 复刻 ch16 最后一节 —— 用 VCS 当 backup，删 dead code。

```bash
mkdir -p /tmp/ch16-scratch && cd /tmp/ch16-scratch

rm -rf delete-demo && mkdir delete-demo && cd delete-demo
git init -q
git config user.email "you@x" && git config user.name "you"

# 准备一段有 dead code 的代码
cat > main.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
int used_helper(int x) { return x * 2; }          /* 非 static, 防止 unused 警告 */
static int dead_helper(int x) { return x * 999; }   /* nobody calls this */
static int also_dead(int x) { return -x; }            /* nobody calls this */
int main(int argc, char **argv) {
    if (argc < 2) return 1;
    int v = atoi(argv[1]);
    printf("%d\n", used_helper(v));
    return 0;
}
EOF
git add . && git commit -q -m "initial"
echo "=== 1) 检测 dead functions ==="
# 用 nm 查每个 static function 是否被引用
# 简易版: 用 grep 看 function body 是否被任何其它代码提及
for FN in used_helper dead_helper also_dead; do
    HITS=$(grep -c "\\b$FN\\b" main.c)
    if [[ "$HITS" -le 1 ]]; then
        echo "DEAD: $FN (only $HITS occurrence — the definition itself)"
    else
        echo "USED: $FN ($HITS occurrences)"
    fi
done

echo
echo "=== 2) 删 dead functions ==="
# 真删 — 用 awk 而不是 sed 多行删除, 因为 `}` 通常跟代码同行, `^}` 不匹配
# 用 perl 一次性 awk 替代: 删除从 "static int FUNC" 到下一个 "}" 闭括号之间所有行
perl -i -0pe 's/static int dead_helper[^\n]*\n.*?\n\}//s' main.c
perl -i -0pe 's/static int also_dead[^\n]*\n.*?\n\}//s' main.c

cat main.c
echo "---"

echo "=== 3) 编译验证 ==="
cc -std=c17 -Wall -Wextra -o main main.c
./main 5
echo "rc=$?"

echo
echo "=== 4) commit 删除 (VCS 是 backup) ==="
git add . && git commit -q -m "delete dead helpers (dead_helper, also_dead)"
git log --oneline
echo "(git blame / git show 仍能看到历史)"
```

**验收**：
- 检测阶段标 `dead_helper` + `also_dead` 为 DEAD
- 删后代码只剩 `used_helper` + main，编译 + 运行通过
- `git log` 显示两次 commit：`initial` + `delete dead helpers`
- VCS 是 backup 演示完整

## 六、值得深入思考的问题

### Q1: Scratch Refactoring 是不是被高估了？

ch16 把 scratch refactor 放在 *明星位置*。**问**：scratch 在大型 OOP 项目里真的有 30 分钟见效吗？还是其实要几小时？而且 scratch 不写测试 = 改错时 *不知道*。**关键问题**：scratch 是不是只在 *C / 函数式* 这类"小颗粒度"代码上有效？

### Q2: Sketching 在远程 pair 里怎么 work？

Feathers 写 ch16 时远程 pair 不普及。图 16.1 那种"纸 + 笔" 在 Zoom 里看不见。**关键问题**：远程 pair 时代，sketching 的等价动作是什么？（excalidraw / FigJam / 共享屏幕画）？这些工具是不是反而让 sketch 变 *正式* 失去 *自由 blob* 的对话性？

### Q3: Delete Unused Code 在 framework / public library 里能用吗？

公共库不能"删 unused"——下游可能依赖。**关键问题**：什么时候 delete 是安全的（内部项目 / 不发布 / 内部 API），什么时候应该保留（公共 API / framework / OSS）？开源项目维护者怎么决定"反正没人 issue = 删"？

### Q4: AI 时代的 Listing Markup 是不是该 *重新发明*？

Feathers 的 markup 是 *纸 + 笔*。AI 时代 IDE 给"hover doc / jump to definition" 已经替代了部分 markup。**关键问题**：未来工程师还需要 *打印 + 高亮* 吗？还是 markup 应该转成 *思维工具链*（hover + bookmark + comment）？哪些 markup 价值无可替代（brace 对齐 / Effects of Change 扩散）？

### Q5: Scratch Refactor 是不是让"乱改代码"正当化？

Feathers 说"乱改" 安全因为不 commit。**关键问题**：当团队约定"scratch OK" 时，是不是给"乱改"开绿灯？什么情况下 scratch 应该 *commit to a branch*（feature/scratch-2025-09-01）而不是直接 revert？

### Q6: Sketching 与代码 review 的关系

review 时常用 mermaid / UML 解释设计 —— 这和 sketch 一致还是不同？**关键问题**：code review 时画的图是不是该 *进 commit*（作为 design doc 的一部分）？还是保持"当场"的非正式性？

### Q7: 当 *没有任何* 同事能 pair 时 scratch refactor 是不是孤独？

远程独立开发者 / 单人维护项目时 scratch refactor 没 pair。**关键问题**：单人维护时 scratch 还是好工具吗？是不是单人应该 *写下来* 思考过程而不是靠 pair 对话？

### Q8: Delete Unused Code 与 *注释掉的代码* 的关系

legacy 里常见 `/* old_code_here */` 或 `// removed because X` —— 这些是"delete 的妥协"。**关键问题**：注释代码该不该也删？什么时候保留（比如有 bug history）？有没有工程规约说"dead code 包括 commented-out code"？

---

*下一章预告*: **Chapter 17 — My Application Has No Structure** — ch16 是 *单文件级* 的理解技法；ch17 推到 *整个应用* 的架构层。具体技法：**Telling the Story of the System**（强制用 1-2 句话讲架构） + **Naked CRC**（用 index cards 在桌上摆出对象实例） + **Conversation Scrutiny**（倾听日常对话，看代码里的概念和对话里的是否对齐）。ch17 是 ch16 的"系统级"版本。

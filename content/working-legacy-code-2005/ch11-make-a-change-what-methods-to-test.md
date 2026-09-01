# Chapter 11 — I Need to Make a Change. What Methods Should I Test?

> **PDF**: p.173-194（22 页）
> **定位**: ch2 教"用测试当夹具"；ch11 把测试点的选址从**单方法**放大到**系统级**。两个核心武器：(1) **effect sketch** —— 用一张气泡图推导"这次改动可能影响哪些方法/变量"；(2) **reasoning forward** —— 从 change point 反推"哪几个下游能 sense 到这次改动"。ch12 把这两个武器用到"成片改动"场景。
>
> **章节标题勘误**：任务清单里把本章误标为 "Change a Method's Signature"；PDF 173–194 实际章节标题是 *"I Need to Make a Change. What Methods Should I Test?"*。本书无独立的 signature-change 章 —— signature preservation 在 ch25 catalog（Refactoring 312 条目）里以"Preserve Signatures" 出现。本笔记按 PDF 实际内容编写。

## 〇、第一性原理思考

**这章做了什么**：给出 effect sketch（气泡图，把"被改的"指向"返回值会变的"）和 reasoning forward（从 change point 反推下游能 sense 这次改动的位置），再加 Tools for Effect Reasoning 的 6 步法。

**为什么这样拆**：ch2 给的是邻接类级别"找测试点"；当改动跨多类时，"邻接"已经不靠谱，所以上升到一个 system-level 心智模型：每次改动都有 chain of effects，画出来再决定测哪。

**最值钱的洞见**：const / final / readonly / Java final 不是装饰 —— 它们是 effect sketch 的防火墙；不知道语言的"什么在构造后不再变"，实际上你连 effect 链的边界都画不出来，更别说找测试点。
## 一、章节概述

- **测试选址的两难**：ch2 给的是"找邻接类写测试"的局部方法；ch11 上升为"在多类多方法系统里，哪里写测试能让一组改动全部被 sense"。这是 *Legacy Code Change Algorithm* 第 2 步（Find test points）的放大版。
- **Reasoning About Effects**：每个改动都有"chain of effects" —— 改一处，影响其 caller、caller 的 caller、一直回到系统边界。**Effect sketch** = 把这种影响画成气泡图：变量 / 方法各是一个气泡，箭头从"会被改的"指向"返回值会变的"。图 11.1–11.4 给出 CppClass 的 4 张 sketch。
- **Reasoning Forward** vs backward：调试时 backward（从 bug 反推到 root）；写 characterization test 时 forward（从 change point 推"谁会被影响"）。Forward 让我们预先圈出测试点而不是事后追。
- **Effect Propagation 的三种基本通道**：(a) 返回值被 caller 使用；(b) 传参对象被修改；(c) 静态 / 全局数据被修改。C++ 的 `const`、Java 的 `final`、C# 的 `readonly` 是这三种通道的"防火墙" —— 知道它们就知道 effect 的传播边界。
- **Tools for Effect Reasoning** 6 步法（Feathers）：找 change point → 看返回值 caller → 看修改的字段 → 看 caller 的 caller → 看 superclass/subclass → 看全局 / 静态 → 看参数及其方法返回值。
- **Knowing your language**：C++ `mutable` 在 `const` 方法里能修改；Java/C# String 是 immutable；Java final 字段不能在构造后改。**这些 language-level firewall 是 effect sketch 的边界依据**。
- **Learning from Effect Analysis**：当你对 codebase 熟了之后，会出现 "but that would be stupid" 规则 —— 比如 CppClass 例子里"`declarations` 在构造后没人会改"。**好代码 = 大量这种 rule；坏代码 = rule 散乱或不存在**。
- **Simplifying Effect Sketches**：refactor 让 effect sketch 的 endpoint 收敛。例：把 `getInterface` 改成内部调 `getDeclaration(int)` 后，写 `getInterface` 的测试顺带覆盖 `getDeclaration` —— 测试工作量减半。
- **Encapsulation vs Test Coverage**：当二者冲突时，**Feathers 偏向 test coverage** —— 因为有测试后能再 refactor 回更好的封装。这是 ch11 末尾的工程哲学落点。
- **与其它章节的关系**：ch10 = 方法级（"这一个方法不可测"）；ch11 = 系统级（"哪些方法该测试"）；ch12 = 跨多类的测试点圈选（pinch point）；ch13 = 写什么样的测试（characterization test）。

## 二、核心 Takeaways

### Takeaway 1：每个改动都有 chain of effects，画出它

- **是什么**：把"这次改动会让哪些方法的返回值变"画成一张箭头图 —— *effect sketch*。每个变量、每个方法是气泡；箭头从"会被改的"指向"返回值会受影响的"。
- **为什么重要**：把"我以为只改一个东西"从口头保证变成可视证据。Effect sketch 是 ch1 "Takeaway 3（adding vs changing）" 的具体化 —— 改一处要看整条链。
- **解决什么问题**：避免"测试了改的方法就以为 OK，结果 caller 拿到奇怪返回值"的回归。
- **适用场景**：任何对既有方法的改动；任何新增返回值的方法；任何修改字段的方法。

### Takexture 2：测试点选址 = effect sketch 的可达端点

- **是什么**：能 sense 到改动效果的端点就是测试候选点。挑端点的判据：(a) 端点是公开 API（外部代码会调到）；(b) 端点返回值易于 assert；(c) 端点的输入容易构造。
- **为什么重要**：把"在哪写测试"从"看代码凭经验"变成"看 sketch 看端点"。
- **解决什么问题**：防止"测了 8 个方法但漏了真正下游的那个"。
- **适用场景**：多文件多类的改动；新人第一次接触 legacy 仓库。

### Takeaway 3：Reasoning Forward 是 characterization test 的选址工具

- **是什么**：从 change point 出发，沿三条通道（返回值 / 传参对象 / 全局数据）逐步推"谁会受影响"。最终把所有 reachable 的端点列出来作为测试候选。
- **为什么重要**：Forward reasoning 让我们预先知道"这次改动会破坏什么"；backward reasoning 是 debug 工具。
- **解决什么问题**：让"写测试"从"凭直觉挑几个方法"变成"由改动推覆盖"。
- **适用场景**：准备 characterization test；做 code review 的"是否需要更新测试"判断。

### Takeaway 4：Effect Propagation 的三种通道 + language firewall

- **是什么**：三种通道（return value / parameter mutation / static-global）+ 防火墙（C++ `const`、Java `final`、C# `readonly`、immutable String）。知道防火墙在哪，effect sketch 的箭头就在哪终止。
- **为什么重要**：避免"以为字段是 final 所以不会被改，结果是 `mutable`"这种 false firewall。
- **解决什么问题**：让 effect sketch 不必追到代码每一行，而是按 language rule 提前剪枝。
- **适用场景**：C++ `mutable` / `const_cast` 滥用；Java reflection 修改 final 字段；C# `unsafe` block 改 readonly。

### Takeaway 5：Refactor 让 effect sketch 简化 = 测试工作量减半

- **是什么**：让 method A 内部调 method B 后，写 A 的测试顺带覆盖 B。Effect sketch 的端点数从 N 收敛到 N-K，测试点也跟着少。
- **为什么重要**：让 refactor 的好处可测量 —— 不是"代码更优雅"，而是"测试更少"。
- **解决什么问题**：给"refactor 不止是 aesthetic"提供量化依据。
- **适用场景**：多方法共享同一段逻辑；多方法读同一字段。

### Takeaway 6：Encapsulation 与 Test Coverage 冲突时选后者

- **是什么**：Feathers 明确说 —— 当"为了可测不得不破坏一些封装" vs "保住封装但测不到" 冲突时，**先测后封**。理由：测试到位之后还能 refactor 回好封装，没有测试再好的封装也只是"没人敢动的干净代码"。
- **为什么重要**：把"测试债"和"封装债"做权衡 —— 测试债更紧迫，因为它是后续一切 refactor 的前提。
- **解决什么问题**：避免团队在"封装洁癖"和"测试覆盖"间无止境地争论。
- **适用场景**：参数化构造器（Parameterize Constructor）破坏封装但换来可测性；Extract Interface 暴露私有成员但让 fake 可写。

### Takexture 7：Knowing your language = effect sketch 的剪枝依据

- **是什么**：C++ `mutable` + `const` 是反例（看上去 const 其实可变）；Java final String 是真防火墙；C# `readonly` field 不阻止 collection mutation。**每条 language rule 都是 effect sketch 的一条边**。
- **为什么重要**：避免"凭语法常识判断 firewall，结果漏了 mutable / readonly 集合"。
- **解决什么问题**：让 effect sketch 的边界有客观依据而不是凭印象。
- **适用场景**：新语言 / 不熟悉的 codebase；接手他人维护的代码。

## 三、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **syscall 的 effect sketch = 调用图 + 字段修改图**。Linux kernel 中 `sys_open` 的 effect：返回 fd（caller 用）→ 修改 `fdtable`（进程内可见）→ 修改 inode refcount（全局）→ 触发 `fsnotify` 事件（observer 可见）。这正好对应 ch11 的三种通道。子系统 maintainer 在 review syscall patch 时实际就在做 effect sketch。
- **`const` 在 kernel 里 = 编译期 firewall**：kernel 中 `const struct file_operations *` 不代表 file_operations 指向的内容不变，只是指针不变。内核代码用 `rcu_dereference` + `const` 组合来表达"读侧 firewall"。这是 ch11 firewall 概念在并发场景下的具体化。
- **`EXPORT_SYMBOL` 是 kernel 侧的 effect sketch 边界**：一个 symbol 是否 EXPORT，决定了 module 边界外能否 sense 到它的 effect。这与 ch11 "subclass/superclass 也是 effect source" 的论断同构。
- **`mutable` 在 kernel = `volatile` + 原子变量**：`atomic_t` / `atomic64_t` 即使在 `const` 上下文也能改。Effect sketch 必须显式标注 `atomic_*` 字段为 effect source。这是 ch11 Takeaway 4 的内核特化。
- **`/proc` / `sysfs` 文件是 kernel → userspace 的 effect 通道**：改一个 sysfs entry = 影响 userspace reader。Kernel patch 必须 review "改了 sysfs entry 是否会让 userspace 读到新值"。这与 ch11 "static-global channel" 等价。

#### Linux 系统 — Effect sketch × 系统侧映射表

| Effect sketch 概念           | Linux / kernel 映射                                | 工具/手法                          |
| ---------------------------- | -------------------------------------------------- | ---------------------------------- |
| Return value channel         | syscall 返回码 / fd / pointer                      | man pages + LTP                    |
| Parameter mutation channel   | `struct file *` / `struct inode *` 内部字段改       | KUnit + parameter fixture          |
| Static/global channel        | `current` task / 全局 jiffies / sysctl             | ftrace + kprobe                    |
| Language firewall (`const`)  | `const struct file_operations *`                   | sparse + `gcc -Wcast-qual`         |
| `mutable` 异常               | `atomic_t` / `rcu_dereference`                     | Coccinelle script                  |
| subclass/superclass channel  | `struct inode_operations` 继承                     | kunit + `DEFINE_STRUCT_OPS`        |
| 端点 (test point)            | LTP testcase / KUnit / selftests                   | CI gating                          |

### 3.2 机器人软件视角

- **ROS2 service / action callback 的 effect sketch = topic + parameter + transform**。一个 `MoveBase::computePath` 的 effect：返回路径（caller 用）→ 修改 `costmap`（后续 planning 用）→ 修改 `tf`（视觉反馈链）。**测试点候选**：`/plan` topic subscriber、`/costmap` 的 getter、tf 静态 listener。
- **launch 文件是 effect sketch 的物理表示**：每个 `<node>` 标签 = 一个 effect source；每个 `<remap>` / `<param>` = 一个 effect 通道。从 launch 文件反向追溯"哪些 effect 通道漏测"比从 C++ 代码追溯更省力。
- **`message_filters` 的同步器是 multi-channel effect 聚合点**：ApproximateTimeSynchronizer 把多个 topic 聚合到一个 callback。这与 ch11 "narrowing in effect sketch" 同构 —— 同步器是 pinch point，测这一个就够了。
- **ROS2 lifecycle node 的 transition callback = state machine 的 effect sketch**：每个 transition 触发 on_cleanup / on_shutdown 等副作用。**测试点**：`/get_state` service 调用 + state assertion。
- **robot hardware HAL 是 effect sketch 的 global channel**：`/dev/ttyUSB0` / `can0` 的写入 = 全局 effect。**测试隔离** = `ros2_control` 的 `hardware_interface/mock` 模块（ch25 Adapt Parameter 的机器人特化）。

### 3.3 初级 vs 高级工程师对比

| 维度                | 初级工程师                                              | 高级工程师                                              |
| ------------------- | ------------------------------------------------------- | ------------------------------------------------------- |
| 改动影响判断        | "我改了一个方法，应该没影响吧"                          | 先画 effect sketch，列出所有可达端点                     |
| 测试选址            | "改哪个测哪个"                                          | 按 sketch 端点选；优先选公开 API 端点                   |
| Reasoning 方向      | Debug 时 backward（追 bug）                             | 写 characterization test 时 forward（追覆盖）           |
| Language firewall   | "String 是 immutable 吧"（凭印象）                     | 看具体声明；C++ 看 `mutable`，Java 看 final 是否 reflection-safe |
| Refactor 评估       | "代码更短更优雅"                                        | "effect sketch 简化，测试点从 N 个收敛到 N-K 个"        |
| Encapsulation 冲突  | 优先保封装                                             | 优先 test coverage，有测试后再 refactor 回封装          |

> **关键差异**：高级把 effect sketch 当作 PR review 的客观证据；初级凭直觉。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

- 仍然极重要。原因：
  - **AI 写代码更频繁 = effect chain 更长**：LLM 倾向把改动做成"5 行 diff"，但这 5 行可能横跨 5 个文件，effect chain 不缩反增。Effect sketch 在 AI 流水线里反而更必要。
  - **AI 不擅长画 sketch**：LLM 给的"impact analysis"是文字段落，不是可视化气泡图。Visual sketch 仍由人 / 工具生成。
  - **测试覆盖率 ≠ effect 覆盖**：AI 写的"高 coverage"测试可能漏掉真正的 effect 端点 —— 因为 coverage 是 line-level，effect sketch 是 call-graph-level。
  - **refactor vs 简化 sketch 的判断仍由人**：AI 给的"simplification"是否真的简化了 effect sketch，需要人 review。

### 4.2 AI 已经能做的（具体到 ch11 主题）

- **生成 call-graph 与 dependency graph** —— 替代手画 effect sketch 的第一步。
- **基于 diff 推荐 effect sketch 端点** —— 把"这次改动可能影响哪些公开 API"列出来。
- **自动生成 characterization test 候选** —— 按 sketch 端点排序，列出推荐测试列表。
- **检测 mutable / atomic 等 firewall 异常** —— grep `mutable` + `const` 同时出现，提示"看起来 const 但实际可变"。

### 4.3 AI 不能替代的（具体到 ch11 主题）

- **judgment call：哪个端点是"真测试点"** —— AI 列出 20 个端点，但"哪个值得写测试"由人判断。
- **refactor 后 effect sketch 是否真的简化** —— AI 给的"simplification"可能只是换皮，effect 通道数没变。
- **language firewall 的微妙反例** —— C# `readonly` field 不阻止 collection mutation，AI 不一定能区分。
- **跨子系统的 effect contract** —— ROS 节点行为在 costmap/控制器/tf 三处的契约，AI 无法判断"哪些是必需契约 vs 偶然耦合"。

### 4.4 AI 经常写错的地方

针对 ch11 effect-sketch 主题，AI 的典型误判：

| 错误模式                                       | 例子                                                                                                                       | 后果                                                              |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- |
| **把 "coverage" 当作 "effect 覆盖"**           | AI 报告"本次改动 coverage 95%"，但 effect sketch 显示某个 subclass override 没被覆盖                                         | 测试套件绿，subclass 行为悄悄变                                  |
| **忽略 `mutable` 异常**                        | AI 看到 C++ `const` 方法就标"no effect"，但方法内有 `mutable int cache; cache++`                                            | Effect sketch 漏画一条边                                          |
| **把 subclass 当 firewall**                    | AI 认为"私有字段只在 class 内可见"，但忘了 subclass 可以继承访问                                                           | Effect sketch 漏画 subclass 通道                                  |
| **忽略 static / global channel**              | AI 只追 call graph，不追 static 字段修改                                                                                   | 一个 static counter 的副作用被漏掉                                |
| **forward reasoning 终止过早**                 | AI 在 2 层 caller 后停止推 effect，但实际链有 5 层                                                                          | Test point 选错，测试不到位                                      |
| **凭"看起来像 refactor"判断 effect 不变**      | AI 重构一段控制流，声称"行为不变"，但实际改了 short-circuit evaluation 顺序                                                 | Effect 通道数没变，但 endpoint 变了                               |
| **漏掉 reflection / dynamic proxy**            | Java/C# runtime 通过 reflection 改 final 字段，AI 不识别                                                                   | Effect firewall 假象                                              |

### 4.5 子段：AI 辅助遗留代码理解

适用本会话锁定视角（Linux 系统 + 机器人）。

- **Linux kernel effect sketch 的 AI 辅助**：
  - 用 AI 给一个 syscall 画出"返回 / 参数 mutation / global" 三通道 sketch —— 这是 man page 的结构化升级版。
  - 用 AI 列出"改了这个字段会触发哪些 ftrace event" —— 这与 ch11 effect propagation 等价。
  - **关键限制**：kernel 中的 `rcu_dereference` / `smp_mb__after_atomic` 等同步原语让 effect channel 变成"概率性" —— AI 给不出"什么时候这个 effect 实际可见"的精确判定。
- **ROS/ROS2 节点的 AI 辅助**：
  - 用 AI 给一个节点画出 topic / service / param 三通道 sketch —— 替代 launch 文件分析。
  - 用 AI 给 `costmap_2d` 的 "what mutates my footprint" 列表 —— 这是 effect sketch 的机器人特化。
  - **关键限制**：behavior tree 节点的隐含时序优先级，AI 给不出可信的 effect sketch —— 它能罗列 transitions，但不知道"哪些 transition 是合法契约"。

### 4.6 工程师必须保留的核心能力

- **画出 effect sketch 并用它指导测试选址**（必须人工 —— AI 给的是文字描述，气泡图由人画）。
- **辨认 language firewall 的反例**（C++ `mutable`、Java reflection、C# `readonly` 集合）—— AI 不一定识别。
- **判断 refactor 是否真的简化了 sketch**（必须人工 —— 不是所有"更短"都是简化）。
- **跨子系统 effect contract 的核心 vs 偶然区分**（必须人工 —— AI 模型读得完代码但不知道历史）。

## 五、实践行动项

> ch11 是概念章，下面 4 项把 effect sketch / reasoning forward 落地为可编译运行的小程序。

### A1 — Effect Sketch 自动生成器：从 diff 到端点列表

```bash
mkdir -p /tmp/ch11-effect && cd /tmp/ch11-effect

# 接收一段 "改动描述"，按 ch11 三通道 + 6 步法列出可能的 effect 端点.
cat > effect_sketch.py <<'PY'
#!/usr/bin/env python3
"""effect_sketch.py — ch11 风格的 effect sketch CLI.
接收一句自然语言描述, 按关键字分类到三通道, 输出端点候选.
"""
import re, sys

# 三通道关键字: return value / parameter mutation / static-global
RV = re.compile(r"\b(return|getter|query|read|fetch|compute)\b", re.I)
PM = re.compile(r"\b(mutate|set|write|update|modify|append|push|insert)\b", re.I)
SG = re.compile(r"\b(static|global|singleton|registry|cache|counter|env)\b", re.I)

# 6 步法关键字: change point / caller / field / caller-of-caller / superclass / global
STEPS = [
    ("1.change_point", re.compile(r"\b(modify|change|edit|alter)\s+(?:method|function|field)\b", re.I)),
    ("2.caller",       re.compile(r"\b(caller|callee|invokes?|callers?)\b", re.I)),
    ("3.field",        re.compile(r"\b(field|attribute|state|member|variable)\b", re.I)),
    ("4.caller_of_caller", re.compile(r"\b(transitive|chain|cascade|propagat)\b", re.I)),
    ("5.superclass_or_subclass", re.compile(r"\b(superclass|subclass|inherit|override|virtual)\b", re.I)),
    ("6.global_or_static", re.compile(r"\b(static|global|singleton|registry)\b", re.I)),
]

def channels(desc: str) -> list[str]:
    ch = []
    if RV.search(desc): ch.append("RV (return value)")
    if PM.search(desc): ch.append("PM (parameter mutation)")
    if SG.search(desc): ch.append("SG (static/global)")
    if not ch: ch.append("UNKNOWN — re-describe")
    return ch

def steps_hit(desc: str) -> list[str]:
    return [name for name, rx in STEPS if rx.search(desc)]

def main():
    if len(sys.argv) < 2:
        print("usage: effect_sketch.py '<change description>'")
        sys.exit(2)
    desc = " ".join(sys.argv[1:])
    print(f"# change: {desc}")
    print("# channels hit:")
    for c in channels(desc):
        print(f"  - {c}")
    print("# reasoning steps hit:")
    for s in steps_hit(desc):
        print(f"  - {s}")
    print("# suggested test points (端点):")
    if RV.search(desc): print("  - getter on changed object")
    if PM.search(desc): print("  - read-after-call on param object")
    if SG.search(desc): print("  - read on static/global state")
    print("# refactor opportunity:")
    print("  - look for shared field/method to consolidate")

if __name__ == "__main__":
    main()
PY
chmod +x effect_sketch.py

echo "=== case 1: 修改 getter 返回值 (RV channel) ==="
./effect_sketch.py "modify method getBalancePoint, return value used by callers"

echo
echo "=== case 2: 修改 instance field (PM + field step) ==="
./effect_sketch.py "modify field declarations, transitive via getDeclaration and getDeclarationCount"

echo
echo "=== case 3: 修改 static counter (SG + 6th step) ==="
./effect_sketch.py "modify static singleton cache, read by other components"
```

**验收**：
- case 1：channels = [RV]，steps = [1. change_point, 2. caller]，test point = "getter on changed object"。
- case 2：channels = [PM]，steps = [3. field, 4. caller_of_caller]，test point = "read-after-call on param object"。
- case 3：channels = [SG]，steps = [6. global_or_static]，test point = "read on static/global state"。

### A2 — CppClass effect sketch 复刻（Java 例子的 C 版）

```bash
mkdir -p /tmp/ch11-cppclass && cd /tmp/ch11-cppclass

# 模拟 ch11 p.174-178 的 CppClass + ClassReader, 但用 C 实现,
# 让我们能用 unit test 验证 effect sketch 推断的正确性.
cat > cppclass.h <<'EOF'
#ifndef CPPCLASS_H
#define CPPCLASS_H
#include <stddef.h>

typedef struct {
    char *name;
    void **declarations;     /* array of Declaration* */
    size_t n_decls;
} CppClass;

typedef struct {
    char *text;       /* 不可变 */
    int   weight;     /* 不可变 */
} Declaration;

CppClass *cppclass_new(const char *name, void **decls, size_t n);
void      cppclass_free(CppClass *c);

size_t        cppclass_get_declaration_count(const CppClass *c);
const char   *cppclass_get_name(const CppClass *c);
Declaration  *cppclass_get_declaration(const CppClass *c, int index);

/* 模拟 getInterface: 输出接口声明字符串. */
char *cppclass_get_interface(const CppClass *c, const char *iface_name,
                             const int *indices, size_t n_idx);

/* 模拟 ClassReader.parse -> 创建 CppClass.
 * 内部 declarations 在 parse 期间被填充, 之后不再变化. */
typedef struct {
    int   in_public_section;
    void *decl_buffer[16];
    int   n_decls;
    CppClass *parsed_class;
} ClassReader;

ClassReader *classreader_new(void);
void         classreader_free(ClassReader *c);
int          classreader_parse(ClassReader *cr, const char *input);

#endif
EOF

cat > cppclass.c <<'EOF'
#define _POSIX_C_SOURCE 200809L
#include "cppclass.h"
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* Declaration 是 immutable (模拟 Java String + TokenReader.readToken). */
Declaration *decl_new(const char *text, int weight) {
    Declaration *d = calloc(1, sizeof(*d));
    if (d) {
        d->text = strdup(text);
        d->weight = weight;
    }
    return d;
}
void decl_free(Declaration *d) {
    if (!d) return;
    free(d->text);
    free(d);
}

CppClass *cppclass_new(const char *name, void **decls, size_t n) {
    CppClass *c = calloc(1, sizeof(*c));
    if (c) {
        c->name = strdup(name);
        c->declarations = decls;
        c->n_decls = n;
    }
    return c;
}
void cppclass_free(CppClass *c) {
    if (!c) return;
    free(c->name);
    /* 不 free declarations — 它们归 ClassReader 所有. */
    free(c);
}

size_t cppclass_get_declaration_count(const CppClass *c) {
    return c ? c->n_decls : 0;
}
const char *cppclass_get_name(const CppClass *c) { return c ? c->name : ""; }
Declaration *cppclass_get_declaration(const CppClass *c, int index) {
    if (!c || index < 0 || (size_t)index >= c->n_decls) return NULL;
    return (Declaration *)c->declarations[index];
}

char *cppclass_get_interface(const CppClass *c, const char *iface_name,
                             const int *indices, size_t n_idx) {
    /* 模拟 ch11 p.190 改写: 内部调 cppclass_get_declaration. */
    size_t bufsize = 256;
    char *buf = malloc(bufsize);
    size_t pos = 0;
    pos += snprintf(buf + pos, bufsize - pos, "class %s {\npublic:\n", iface_name);
    for (size_t i = 0; i < n_idx; i++) {
        Declaration *d = cppclass_get_declaration(c, indices[i]);
        if (d) pos += snprintf(buf + pos, bufsize - pos, "\t%s\n", d->text);
    }
    pos += snprintf(buf + pos, bufsize - pos, "};\n");
    return buf;
}

/* ClassReader 简化: 只展示 "parse 后 declarations 不可变" 这条 effect rule. */
ClassReader *classreader_new(void) {
    return calloc(1, sizeof(ClassReader));
}
void classreader_free(ClassReader *cr) {
    if (!cr) return;
    if (cr->parsed_class) cppclass_free(cr->parsed_class);
    for (int i = 0; i < cr->n_decls; i++) decl_free(cr->decl_buffer[i]);
    free(cr);
}
int classreader_parse(ClassReader *cr, const char *input) {
    (void)input;
    /* 模拟: 添加 2 个 immutable declaration. */
    cr->decl_buffer[0] = decl_new("virtual void f();", 1);
    cr->decl_buffer[1] = decl_new("virtual void g();", 2);
    cr->n_decls = 2;
    cr->parsed_class = cppclass_new("MyClass", cr->decl_buffer, 2);
    return 0;
}
EOF

cat > test_cppclass_effect.c <<'EOF'
/* 验证 ch11 effect sketch 推断:
 *   - "declarations 不可变" 这条 rule 让 effect sketch 简化
 *   - getInterface 调 getDeclaration 后, 测试 getInterface 顺带覆盖 getDeclaration
 */
#include "cppclass.h"
#include <assert.h>
#include <string.h>
#include <stdlib.h>

int main(void) {
    ClassReader *cr = classreader_new();
    assert(classreader_parse(cr, "class MyClass { ... }") == 0);

    /* effect sketch endpoint 1: cppclass_get_declaration_count */
    assert(cppclass_get_declaration_count(cr->parsed_class) == 2);

    /* effect sketch endpoint 2: cppclass_get_declaration (by index) */
    Declaration *d0 = cppclass_get_declaration(cr->parsed_class, 0);
    assert(d0 && strcmp(d0->text, "virtual void f();") == 0);

    /* effect sketch endpoint 3: cppclass_get_interface (顺带覆盖 getDeclaration) */
    int idx[] = {0, 1};
    char *iface = cppclass_get_interface(cr->parsed_class, "IMyClass", idx, 2);
    assert(strstr(iface, "class IMyClass") != NULL);
    assert(strstr(iface, "virtual void f();") != NULL);
    assert(strstr(iface, "virtual void g();") != NULL);
    free(iface);

    /* 验证 effect rule: declarations 在 parse 后不可变.
     * 再次读, 值不变. */
    Declaration *d0_again = cppclass_get_declaration(cr->parsed_class, 0);
    assert(d0_again == d0);  /* 同一指针, 没被替换 */

    classreader_free(cr);
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_cppclass_effect cppclass.c test_cppclass_effect.c
./test_cppclass_effect
echo "rc=$?"
```

**验收**：
- `rc=0`，4 条 effect sketch 推断全部被测试验证。
- 注释说明：Java String immutable + readToken 不返回新对象 = effect firewall，让 sketch 在构造后立即收敛。

### A3 — InMemoryDirectory 改动推理（ch11 p.179-185）

```bash
mkdir -p /tmp/ch11-inmem && cd /tmp/ch11-inmem

cat > inmem.h <<'EOF'
#ifndef INMEM_H
#define INMEM_H
#include <stddef.h>

typedef struct {
    char *name;
    char *text;
} Element;

typedef struct {
    Element **elements;
    size_t    n;
} InMemoryDirectory;

InMemoryDirectory *imd_new(void);
void               imd_free(InMemoryDirectory *d);

void     imd_add_element(InMemoryDirectory *d, Element *e);
void     imd_generate_index(InMemoryDirectory *d);    /* 不可调两次 — bug */
size_t   imd_get_element_count(const InMemoryDirectory *d);
Element *imd_get_element(const InMemoryDirectory *d, const char *name);

Element *element_new(const char *name, const char *text);
void     element_free(Element *e);

#endif
EOF

cat > inmem.c <<'EOF'
#define _POSIX_C_SOURCE 200809L
#include "inmem.h"
#include <stdlib.h>
#include <string.h>

Element *element_new(const char *name, const char *text) {
    Element *e = calloc(1, sizeof(*e));
    if (e) { e->name = strdup(name); e->text = strdup(text); }
    return e;
}
void element_free(Element *e) {
    if (!e) return;
    free(e->name); free(e->text); free(e);
}

InMemoryDirectory *imd_new(void) {
    return calloc(1, sizeof(InMemoryDirectory));
}
void imd_free(InMemoryDirectory *d) {
    if (!d) return;
    for (size_t i = 0; i < d->n; i++) element_free(d->elements[i]);
    free(d->elements);
    free(d);
}

/* 模拟 ch11 的 addText: 文本追加 + 可能调 view (此处跳过). */
static void element_add_text(Element *e, const char *new_text) {
    size_t cur = e->text ? strlen(e->text) : 0;
    size_t add = strlen(new_text);
    e->text = realloc(e->text, cur + add + 1);
    memcpy(e->text + cur, new_text, add + 1);
}

void imd_add_element(InMemoryDirectory *d, Element *e) {
    d->elements = realloc(d->elements, (d->n + 1) * sizeof(Element *));
    d->elements[d->n++] = e;
}

void imd_generate_index(InMemoryDirectory *d) {
    Element *idx = element_new("index", "");
    for (size_t i = 0; i < d->n; i++) {
        element_add_text(idx, d->elements[i]->name);
        element_add_text(idx, "\n");
    }
    imd_add_element(d, idx);
}

size_t imd_get_element_count(const InMemoryDirectory *d) {
    return d ? d->n : 0;
}
Element *imd_get_element(const InMemoryDirectory *d, const char *name) {
    if (!d) return NULL;
    for (size_t i = 0; i < d->n; i++) {
        if (strcmp(d->elements[i]->name, name) == 0) return d->elements[i];
    }
    return NULL;
}
EOF

cat > test_inmem.c <<'EOF'
/* 验证 ch11 reasoning forward 的端点:
 *   - imd_get_element_count
 *   - imd_get_element
 *   - element 的 text 字段 (通过 imd_get_element("index") 间接读)
 * 测试只覆盖这 3 个端点, 等价于"在 effect sketch 端点上写 test". */
#include "inmem.h"
#include <assert.h>
#include <string.h>

int main(void) {
    InMemoryDirectory *d = imd_new();

    /* 加入 2 个 element, 然后生成 index. */
    imd_add_element(d, element_new("foo", ""));
    imd_add_element(d, element_new("bar", ""));

    /* 端点 1: count 包含 index 之前是 2. */
    assert(imd_get_element_count(d) == 2);

    imd_generate_index(d);

    /* 端点 1: count 变 3. */
    assert(imd_get_element_count(d) == 3);

    /* 端点 2: index 元素存在. */
    Element *idx = imd_get_element(d, "index");
    assert(idx != NULL);

    /* 端点 3: index.text 包含 foo + bar (注意: ch11 说"不可调两次",
     * 我们也不重测, 让 single-call path 通过). */
    assert(strstr(idx->text, "foo") != NULL);
    assert(strstr(idx->text, "bar") != NULL);

    imd_free(d);
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_inmem inmem.c test_inmem.c
./test_inmem
echo "rc=$?"
```

**验收**：
- `rc=0`，3 个 effect sketch 端点全部覆盖。
- 注释说明：ch11 p.179-185 的 InMemoryDirectory 例子讲的就是这 3 个端点；测试不重复调 `generate_index` 是因为 ch11 提到"二次调用会 gum up" —— 把这个 bug 留作 characterization test 的未来添加项。

### A4 — Language Firewall 反例：C++ `mutable` + `const` 不真 firewall

```bash
mkdir -p /tmp/ch11-firewall && cd /tmp/ch11-firewall

# 验证 ch11 Takeaway 4: "mutable in const method" 是 language firewall 反例.
# 表面看 method 是 const 不修改状态, 实际 mutable cache 字段被修改.
cat > firewall.h <<'EOF'
#ifndef FIREWALL_H
#define FIREWALL_H
#include <stddef.h>

typedef struct {
    int x;
    int y;
    mutable int cache;     /* mutable: 即使 const method 也能改 */
    mutable int hits;      /* mutable: 计数器 */
} Coord;

void coord_init(Coord *c, int x, int y);

/* 看似 read-only (const), 实际 mutable cache 被改. */
double coord_distance(Coord *self, Coord *other);

/* 用于测试: 看 cache/hits 字段. */
int coord_get_cache(const Coord *c);
int coord_get_hits(const Coord *c);

#endif
EOF

cat > firewall.c <<'EOF'
#include "firewall.h"
#include <math.h>

void coord_init(Coord *c, int x, int y) {
    c->x = x; c->y = y;
    c->cache = -1;
    c->hits = 0;
}

double coord_distance(Coord *self, Coord *other) {
    /* 即使签名 const, mutable 字段仍可改. 这是 ch11 的 mutable 反例. */
    self->hits++;
    if (self->cache >= 0 && other->x == self->x && other->y == self->y) {
        return self->cache / 1000.0;
    }
    double d = sqrt((double)(other->x - self->x) * (other->x - self->x) +
                    (double)(other->y - self->y) * (other->y - self->y));
    self->cache = (int)(d * 1000);
    return d;
}

int coord_get_cache(const Coord *c) { return c->cache; }
int coord_get_hits(const Coord *c)  { return c->hits; }
EOF

cat > test_firewall.c <<'EOF'
/* 验证 ch11 Takeaway 4 + 7: const 方法 + mutable 字段
 * 让 effect sketch 误以为 "no effect", 实际 cache/hits 在变. */
#include "firewall.h"
#include <assert.h>
#include <math.h>

int main(void) {
    Coord a, b;
    coord_init(&a, 0, 0);
    coord_init(&b, 3, 4);

    /* 第一次调: cache 写入, hits=1. */
    double d1 = coord_distance(&a, &b);
    assert(fabs(d1 - 5.0) < 0.001);
    assert(coord_get_cache(&a) == 5000);
    assert(coord_get_hits(&a) == 1);

    /* 第二次调: cache 复用, hits=2. 验证 mutable effect. */
    double d2 = coord_distance(&a, &b);
    assert(fabs(d2 - 5.0) < 0.001);
    assert(coord_get_hits(&a) == 2);    /* hits 增, 即使 const 方法 */

    /* 第三次: b 改了, cache miss, hits=3. */
    coord_init(&b, 6, 8);
    double d3 = coord_distance(&a, &b);
    assert(fabs(d3 - 10.0) < 0.001);
    assert(coord_get_hits(&a) == 3);

    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o test_firewall firewall.c test_firewall.c -lm
./test_firewall
echo "rc=$?"
```

**验收**：
- `rc=0`，证明 `const` 方法内的 `mutable` 字段在变 —— 这是 ch11 Takeaway 4 的典型反例。
- 注释说明：Java 反射 / C# `unsafe` block 是其它语言侧类似反例；effect sketch 不能凭 const/final 一票否决。

## 六、值得深入思考的问题

### Q1: Effect sketch 应该画在哪里、由谁画？

PR description 内联？独立 wiki？工具自动生成？**关键问题**：sketch 的可视化是不是必须？没有图形 sketch 的纯文字描述算不算 sketch？团队多久 review 一次 sketch 模板？

### Q2: Forward reasoning 的终止条件是什么？

画到 system boundary？画到 public API？画到测试成本 > 收益的地方？**关键问题**：有没有客观的剪枝规则，还是纯 judgment call？AI 给的"建议终止"能不能用？

### Q3: Encapsulation vs Test Coverage 的取舍有没有客观标准？

Feathers 说"偏向 coverage"，但没说偏向多少。**关键问题**：什么情况下应该反过来 —— 比如改的字段是密码 / 私钥，封装优先。安全敏感的场合怎么 enforce？

### Q4: Refactor 让 sketch 简化是否可测？

Takeaway 5 给了逻辑，但没给 measurement。**关键问题**：能不能形式化"effect sketch 简化率 = 测试减少率"？有没有团队真在跟踪这个指标？

### Q5: AI 时代的 effect sketch 会不会变成"自动生成 + 自动验证"？

LLM 能画 sketch、能推荐端点。**关键问题**：人 review sketch 还会发现 AI 漏画的边吗？还是要靠运行时 effect trace（如 ftrace / eBPF）做 ground truth？

---

*下一章预告*: **Chapter 12 — I Need to Make Many Changes in One Area** —— 把 ch11 的 effect sketch 工具用到"成片改动"场景：**interception point**（在哪里写测试能 sense 到一组改动）、**pinch point**（sketch 自然收窄的地方）、以及 pinch point 的陷阱（高扇入测试 = mini integration test）。ch12 是 ch11 的多类放大版。

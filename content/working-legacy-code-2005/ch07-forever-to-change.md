# Chapter 7 — It Takes Forever to Make a Change

> **PDF**: p.99-108（10 页）
> **定位**: ch6 是"代码改动"问题，ch7 是 *"build 时间"* 问题。同一个症状（"改个东西要好久"），但根不在代码，在 *build graph*。本章给出 **dependent-build 解构**：把"改一个文件要 recompile N 个" 降到"改一个文件只 recompile 它自己"。这是 ch4 seam 模型在 *build time* 维度上的对应。

## 〇、第一性原理思考

**这章做了什么**: 用 Figure 7.1 → 7.5 的演进展示 dependent-build 解构 — 把 `ConsultantSchedulerDB` 从具体类抽成接口再拆包, 改一个文件不再触发全项目 recompile。

**为什么这样拆**: 沿 ch4 seam 模型在 *build time* 维度重做一次 — 编译防火墙 = build 版的 seam, 接口稳定则调用方不感知实现变化。

**最值钱的洞见**: 拆包后整体 build *稍微变慢*, 但平均 build *明显变快*; TDD 的节奏成立的前提是 lag < 5–10 秒, 否则反馈循环已破坏。

Spirit rover 地-火 14 分钟延迟和我们 5–30 分钟编译+测试 lag 本质同构 — lag = 难以 debug, 所以 build time 不是工程洁癖, 是 debug 速度的硬约束。

## 一、章节概述

- **核心论断**：改个东西太久，90% 不是因为你"读不懂代码"，而是 *feedback lag* — 从 commit 到看到结果的时间。**Feathers 举 Spirit rover 类比**：地-火 14 分钟通讯延迟 vs *我们* 5-30 分钟编译+测试反馈。**本质相同**：lag = 难以 debug。
- **关键数字**：在大多数语言里，**10 秒内 compile + run 一个类**是可行的。极端可达 5 秒内。**今天大多数项目做不到**，因为 build graph 没有"局部化"。
- **三个原因**：(1) **Understanding** — 项目大了就读不懂；(2) **Lag Time** — 改到看到结果要 30 分钟；(3) **Breaking Dependencies** 卡死 — 第一个不可测类卡住整个 sprint。
- **Build Dependencies**：解构依赖不是 *只为了可测*，*也是为了 build 快*。`Extract Interface` / `Extract Implementer`（ch25 技法）不只是让 fake 注入；它还**让被依赖方代码改了不强制重新编译你**。
- **Figure 7.1 → 7.5 的演进**：AddOpportunityFormHandler 直接依赖 `ConsultantSchedulerDB`（具体类） → Extract Implementer 把 `ConsultantSchedulerDB` 变成接口，`Impl` 是实现 → 再 Extract `OpportunityItem` → 拆 package / library。这是 *依赖反向原理*（DIP）在 build 上的版本。
- **拆包后整体 build *稍微变慢*，但平均 build *明显变快***。这是关键 trade-off：整体 = 编译全部项目；平均 = 编译一次 commit 实际影响的范围。**经验**：拆出来的包越多，平均 build 越快。
- **编译防火墙（compilation firewall）**：当 A 类的 interface 不变时，A 的所有使用方**不需要重编译**。这与 ch4 seam 的"对象 seam" 共享核心思想 —— *边界稳定 = 内部变化不可见*。
- **增量 build 时间目标**：理想 < 5 秒，普通 < 30 秒，可接受 < 1 分钟。**1 分钟以上**：开发节奏已被破坏，必须拆包。
- **Chapter 8 衔接**：build time 拆好后，ch8 的 TDD 添加 feature 才可行 — 不然 test-driven 写一个 method 等 5 分钟编译，TDD 节奏崩。

## 二、核心 Takeaways

### Takeaway 1: Lag time 不是宿命，是工程问题

- **是什么**：改代码到看到结果（compile + run + assert）的时间。10 秒内可行；30 秒难受；5 分钟以上 = 不可持续。
- **为什么重要**：feedback lag 直接影响 *bug 修复速度*。Spirit rover 类比 — 14 分钟延迟 = 一次 iteration 至少 28 分钟；我们 5 分钟 lag 一次 commit 同样要 5 分钟看到结果。lag 长 = bugs 累积。
- **解决什么问题**：把"为什么这个 PR 要测 1 小时"换成"哪些 build step 可以拆出来独立运行"。
- **适用场景**：评估 CI 时间、评估 IDE 的 incremental build 时间、评估单文件 rebuild 时间。

### Takeaway 2: 拆依赖是 build time 的核心杠杆

- **是什么**：把"改 A 就要重编 B/C/D" 通过抽接口 / 抽实现 / 拆包变成"改 A 只重编 A 自身 + 必要的 cpp"。
- **为什么重要**：依赖图越紧凑，recompile 传播越远。**DIP（依赖反转原理）在这里体现**：依赖接口/抽象 = 接口稳定时调用方不变。
- **解决什么问题**：增量 build 时间从"分钟级"压到"秒级"。
- **适用场景**：任意 C++ / Rust workspace crate 项目、Java Maven multi-module、Go module 拆分。

### Takeaway 3: Extract Interface / Extract Implementer 是 build 解构主力

- **是什么**：ch25 的两个技法 — 把具体类抽成接口（Extract Interface，C++ 用纯虚基类；Java/C# 用 interface），或者抽成 implementer（保留原类 + 抽出基类）。
- **为什么重要**：调用方只看接口/基类时，具体实现类变了**调用方不重新编译**。**对 build time 而言**，这等于"具体实现类的 header 是 stable"。
- **解决什么问题**：把"改具体实现 → 全项目重链" 的链条断掉。
- **适用场景**：任何被多个调用方引用的具体类。

### Takeaway 4: 拆 package / library 是粒度更粗的 build 隔离

- **是什么**：把一组相关类独立成包 / 库（Java . jar / Rust crate / Go module / C++ static lib）。包的 *interface* 暴露给其他包；包内部实现可以自由改。
- **为什么重要**：包级隔离 > 类级隔离。包越多，recompile 边界越细。**注意**：拆包有"整体 build 略变慢"的代价 — 包越多，连接器要做的工作越多。
- **解决什么问题**：把"动一个文件就动整个 monolithic tree" 拆成"动一个文件最多动一个 lib"。
- **适用场景**：所有多包构建系统。

### Takeaway 5: 编译防火墙 = seam 在 build 上的映射

- **是什么**：当你依赖接口（而不是实现）时，你的代码 *永远不需要* 因为实现变化而 recompile — *除非接口变了*。这就是 *编译防火墙*。
- **为什么重要**：每个编译防火墙 = 一个 *seam* —— 内部行为可变而外部不可见。ch4 seam 模型在 build 维度的具体实例。
- **解决什么问题**：让"实现层优化"（如 database driver 重写）"调用层代码"零修改 + 零重编。
- **适用场景**：稳定 API 与内部实现的边界；plugin 系统；硬件抽象层。

### Takeaway 6: "理解" 与 "lag time" 是两个独立瓶颈

- **是什么**：Feathers 区分 *understanding lag*（"我读不懂代码要改哪"）与 *build lag*（"我改了要等多久看到结果"）。两个瓶颈**独立存在**。
- **为什么重要**：治"理解"靠 ch16（更好的命名）/ ch17（结构化）。治"lag time" 靠 *本章技法*。
- **解决什么问题**：让团队知道 *该把工程精力花在哪*。
- **适用场景**：估算 sprint 时区分两类问题。

### Takeaway 7: 拆包后整体 build 略变慢 — 但这是合理的 trade

- **是什么**：拆 10 个包 = 10 次 link（vs monolithic 1 次）。整体 build 略慢 5-15%。但平均 build 显著快（因为平均只动 1-2 个包）。
- **为什么重要**：让团队接受"拆包" 不会让 *CI 全量 build* 变快；变快的是 *local incremental build*。这与"快 = 全量" 的直觉反着来。
- **解决什么问题**：拆包决策的 *政治* —— 工程师 vs CI 管理员的拉锯。
- **适用场景**：CI 全量时间 vs 本地增量时间的权衡。

## 三、工程实践视角

> 锁定领域：**Linux 系统开发 + 机器人软件**。

### 3.1 Linux 系统开发视角

- **头文件包含是 C/C++ build 的最大杀手**。Linux kernel `include/linux/list.h` 被几乎所有 . c 包含；改它等于重编 kernel 大半。这正是 ch7 警告的 "transitive include" —— 一个核心 header 的修改 = 重 build 整个项目。
- **`__user` / `__kernel` / `__iomem` 标注** = 内核的"接口稳定化"。它们不改类型含义，但让 sparse / 编译器知道"这是契约" —— 调用方不需要关心具体实现。
- **`EXPORT_SYMBOL_GPL`** = API 稳定性边界。改 GPL 导出符号 = 重新编译所有依赖模块；改 internal static 函数 = 只需重编所在模块。**这是 *编译防火墙* 的最直接实例**。
- **`make M=drivers/net/wireless/`** = ch7 拆包的工业版本。子系统单独编译：subsystem 内部 rebuild 几十秒；全树 rebuild 30-60 分钟。
- **`forward declarations` vs `#include`**：ch7 隐含的 C/C++ 实践 —— 在 header 里 *前向声明* 类型（而不是 #include 整个文件）= 调用方不需要被依赖方的完整定义。这是 *Extract Interface* 在 C 的对应。

#### Linux — Build 解构的工程映射

| ch7 技法                | Linux 内核对应                              | 工程动作                         |
| ----------------------- | ---------------------------------------- | ------------------------------ |
| Extract Interface       | `struct file_operations` / `net_device_ops` | 函数指针 ops table；driver 单独编译 |
| Extract Implementer     | 多态 ops 注册 (`register_netdev`)            | ops 注册时挂入；dev 注册时挂入      |
| 拆 package / library    | `M=somedir` 子系统编译                       | 单 subsystem 编译               |
| 编译防火墙             | `EXPORT_SYMBOL` vs static                  | GPL 边界稳定；internal 自由改     |
| 头文件前向声明          | `struct inode;` 不 include inode.h          | 调用方只需 pointer，不需要 full def |

### 3.2 机器人软件视角

- **ROS 节点间是 *编译期独立* 的**。`MoveBase` / `costmap_2d` / `tf2` 都是独立 catkin packages —— 它们之间只通过 message 接口耦合。**这就是 ch7 编译防火墙的工业实例**：改 `costmap_2d` 不重编 `MoveBase`。
- **ament / colcon build graph** = 编译图。ROS2 的 build 是有向图：package 之间只依赖 message + 头文件；source 改动只传播给 reverse-deps。**核心指标**：单个 package rebuild < 30s。
- **pluginlib 在 build 上 = 强 seam**。`nav2_core::GlobalPlanner` 是抽象基类；具体 planner 编译成 shared library，运行时 pluginlib 加载。**改 planner 不需要 recompile nav2_core 或 nav2_bt_navigator**。
- **ros2_control hardware interface** = Extract Interface 工业版。`hardware_interface::SystemInterface` 是 interface；`mock_system_interface` 是 testing implementer；真实 hardware driver 是 production implementer。
- **rosbag2 / ros2 bag record** = "可独立 build + 独立 deploy" 的工具链样本。`ros2 bag play` 离线 replay 是 ch7 "理解 + lag time" 的合解：先录下真机数据，离线 fast-iterate。

#### 机器人 — Build 解构的工程映射

| ch7 技法                | ROS/ROS2 对应                                  | 工程动作                         |
| ----------------------- | --------------------------------------------- | ------------------------------ |
| Extract Interface       | `nav2_core::GlobalPlanner` / `BT::ActionNode` | 抽象基类 + 多种实现                |
| Extract Implementer     | pluginlib 注册                                 | shared lib 注册；运行时加载       |
| 拆 package / library    | catkin / ament package                          | 单 pkg 编译                      |
| 编译防火墙             | ROS message 接口 (`.msg` 文件)                  | 改 message = 重新生成所有依赖     |
| 头文件前向声明          | `rclcpp::Node` 前向声明                        | plugin 只需 header               |

### 3.3 初级 vs 高级工程师对比

| 维度                 | 初级工程师                                                   | 高级工程师                                                    |
| -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| build 慢的原因       | "机器烂 / CI 太慢"                                          | "build graph 没拆；改 header 1 行要重编 50 个 .cpp"         |
| 看到不可拆的依赖     | "C++ 就是这样"                                              | "用 PIMPL / 接口基类 / forward decl 切断"                    |
| 反馈速度            | 接受 30 秒 compile + 5 分钟 link                            | "我本地 < 5 秒；CI 全量可以接受 10 分钟"                   |
| 拆包决策            | "拆包麻烦，少拆"                                            | "拆一个 5 秒的包，节省 N 个开发者每天 30 分钟"               |
| 看到 transitive include | 删 header line                                              | 引入 forward declaration + interface header                 |

> **关键差异**：高级工程师把 *build graph* 看作 *依赖图*，把 *header* 看作 *seam*。初级工程师把 build 看作 *机器问题* 或 *CMake 问题*。

## 四、AI 时代视角

### 4.1 这个知识今天仍重要吗

仍然极重要。原因：
- **AI 写代码让 build 频率升高** — LLM 生成代码的速度意味着 daily build 触发更多。
- **增量 build 仍是 反馈 loop 的核心** — AI 帮不了 build 时间。
- **依赖图分析是 AI 强项** — `tree-sitter` + call graph 分析能瞬间给出"改这个 header 影响哪些 . cpp"，但 *该抽哪个接口* 仍是人决定。
- **整体 build vs 增量 build 的政治学**：CI 工程师与产品工程师的拉锯 — AI 可以给数据，但无法定夺。

### 4.2 AI 已经能做的（具体到 ch7 主题）

- **生成依赖图 + 标注 cascade 重编风险**：自动分析代码库的 `#include` 拓扑，给出"改 X header 会重编 N 个 . cpp" 的报告。准确率高。
- **建议拆包方案**：基于依赖密度 + 改动频率，给"该把哪些类抽到新包" 的建议。准确率 60-70%。
- **生成 forward declaration header**：从 transitive include 自动抽出 forward decl header。
- **CI 缓存策略推荐**：基于历史 build 数据，给"哪些 artifact 应该 cache" 的建议。

### 4.3 AI 不能替代的（具体到 ch7 主题）

- **判断"该不该拆包"**：拆包有政治成本（团队 / CI / deploy）。什么时候值得拆？这是项目管理判断。
- **Extract Interface 的接口形状**：抽什么方法到接口？这是设计判断，AI 给的是 *plausible*，不是 *correct*。
- **整体 build 变慢的可接受阈值**：5% 可接受还是 15%？与 CI / artifact storage 策略相关。
- **跨包依赖的层次约束**：上层包能否依赖下层包的"内部" header？这是 package governance。

### 4.4 AI 经常写错的地方

针对 ch7 build 解构主题：

| 错误模式                                          | 例子                                                                                                  | 后果 |
| ------------------------------------------------- | ----------------------------------------------------------------------------------------------------- | ---- |
| **拆包后接口选择错误**                              | AI 把 *所有 public 方法* 都抽到 interface，结果接口臃肿（5 个方法只 1 个被 mock 调用）               | 抽象泄漏；具体类小改 = 接口也要改 = 调用方重编 |
| **forward declaration 用了 incomplete type 操作** | AI 写 `class Foo; Foo* f;` 然后 `*f` 操作                                                              | 编译错；运行时 undefined behavior                              |
| **建议拆包但不留接口兼容**                          | AI 把类拆 A/B 两半，但中间留依赖 — A 改时 B 还要重编                                                | 拆包无 build time 收益                                          |
| **建议增量 build cache 但 cache key 错**            | AI 给 `ccache` 配置，但 hash 包含的不是 source + flags 而是 wall-clock                               | cache 命中率接近 0；build 仍慢                                  |
| **拆包后整体 link 变慢不警告**                      | AI 拆 30 个 .so 不知道 link overhead 累积                                                          | CI 全量 build 变慢 50%                                         |
| **忽略 transitive header include**                 | AI 改 .h 时不分析 .cpp 是否 #include 它                                                            | 实际改 1 行触发重编 200 个 .cpp                                 |
| **建议插件化但 runtime 链接风险不说**              | AI 推荐 pluginlib 但没说 `dlopen` 失败如何 fallback                                                  | 生产 plugin 加载失败 = 系统降级运行                            |

### 4.5 子段：AI 辅助遗留代码理解 — 在本主题专项

- **AI 帮你"画出 build cascade 图"**：从 `#include` 拓扑自动构图，标红"高扇出" header。这是 ch7 Figure 7.1 的自动版。
- **AI 帮你"识别拆包 pinch point"**：基于改动频率 + 依赖密度，自动建议"该把这 5 个类抽到 lib X"。准确率 60%，人工 review 必要。
- **AI 帮你"评估拆包收益"**：模拟"包 X 抽出去后，平均 build 时间从 30 秒 → 8 秒" 的预测。**风险**：模型基于 commit history，commit 模式如果变了，预测失效。
- **AI 不会替你做 *package governance***：谁有权往某个包加类？这是组织治理问题。

### 4.6 工程师必须保留的核心能力

- **判断是否值得拆包**：政治成本 / 收益分析，与 AI 无关。
- **Extract Interface 的接口形状选择**：把"哪些方法"抽到接口 — 这是 ch7 隐藏在 build time 背后的 *设计决策*。
- **forward declaration 的纪律**：C/C++ 头文件的"最少信息"原则 — header 只暴露指针 + 函数签名，不暴露完整定义。
- **CI 缓存策略**：哪些 artifact 该 cache，cache key 怎么设 — 这是 build system 工程。
- **评估增量 vs 全量 build 的真实时间**：用真实测量，不用 AI 给的预测。

## 五、实践行动项

> ch7 核心是 *build graph 解构*。4 个 demo 演示编译防火墙与拆包隔离，每个在 `/tmp/ch07-build*` 下真实测量。

### A1 — 头文件 transitive include 演示：1 行修改触发的重编

```bash
mkdir -p /tmp/ch07-build && cd /tmp/ch07-build

# lib.h — 经常被包含的头文件
cat > lib.h <<'EOF'
#ifndef LIB_H
#define LIB_H
static int lib_version = 1;   /* 改这一行 → 所有 includer 重编 */
int lib_get_version(void);
#endif
EOF

cat > lib.c <<'EOF'
#include "lib.h"
int lib_get_version(void) { return lib_version; }
EOF

# user.c — 大量引用方
cat > user.h <<'EOF'
#ifndef USER_H
#define USER_H
#include "lib.h"
int user_do_work(void);
#endif
EOF

cat > user.c <<'EOF'
#include "user.h"
int user_do_work(void) { return lib_get_version() + 1; }
EOF

cat > main.c <<'EOF'
#include <stdio.h>
#include "user.h"
int main(void) { printf("user=%d\n", user_do_work()); return 0; }
EOF

# 编译一次记 baseline 时间
echo "=== baseline build ==="
cc -std=c17 -Wall -Wextra -o app main.c user.c lib.c
echo "rc=$?"

# 修改 lib.h 一行 (改 lib_version)
echo "=== after change lib_version 1→2 (same content, but mtime update) ==="
touch lib.h
START=$(date +%s%N)
cc -std=c17 -Wall -Wextra -o app main.c user.c lib.c 2>&1
END=$(date +%s%N)
echo "rc=$?"
echo "elapsed_ms=$(( (END - START) / 1000000 ))"
```

**验收**：
- `app` 输出 `user=2`。
- `elapsed_ms` < 3000。
- 改 lib. h 后整树重编耗时记录 — 演示 *transitive include* 触发 cascade。

### A2 — 用 forward declaration 切断传递依赖

```bash
mkdir -p /tmp/ch07-build && cd /tmp/ch07-build

# 重写 user.h：不再 #include "lib.h"
cat > lib.h <<'EOF'
#ifndef LIB_H
#define LIB_H
int lib_get_version(void);
#endif
EOF

cat > lib.c <<'EOF'
int lib_get_version(void) { return 42; }
EOF

# user.h 只声明外部符号，不 include
cat > user.h <<'EOF'
#ifndef USER_H
#define USER_H
/* 关键: 之前 #include "lib.h" 现在移除 */
int user_do_work(void);
#endif
EOF

cat > user.c <<'EOF'
#include "user.h"
extern int lib_get_version(void);
int user_do_work(void) { return lib_get_version() + 1; }
EOF

cat > main.c <<'EOF'
#include <stdio.h>
#include "user.h"
int main(void) { printf("user=%d\n", user_do_work()); return 0; }
EOF

cc -std=c17 -Wall -Wextra -o app main.c user.c lib.c
./app
echo "rc=$?"

# 改 lib.h 后只重编 lib.c
echo "=== 修改 lib 后只重编 lib.c ==="
cat > lib.c <<'EOF'
int lib_get_version(void) { return 100; }
EOF
START=$(date +%s%N)
cc -std=c17 -Wall -Wextra -o app main.c user.c lib.c 2>&1
END=$(date +%s%N)
echo "rc=$?  elapsed_ms=$(( (END - START) / 1000000 ))"
./app   # 输出 user=101
```

**验收**：`user=101` + `elapsed_ms` < 3000。证明 forward declaration 让 user. h 不再依赖 lib. h 的内容。

### A3 — Extract Interface: 用 ops table 模拟编译防火墙

```bash
mkdir -p /tmp/ch07-build && cd /tmp/ch07-build

# interface.h — 不变的契约
cat > interface.h <<'EOF'
#ifndef INTERFACE_H
#define INTERFACE_H
typedef struct Iface {
    int  (*get)(struct Iface *self);
    void (*set)(struct Iface *self, int v);
} Iface;
#endif
EOF

# impl_v1.c — 第一个实现
cat > impl_v1.c <<'EOF'
#include "interface.h"
#include <stdlib.h>
static int v1_get(Iface *s) { return s ? 1 : 0; }
static void v1_set(Iface *s, int v) { (void)s; (void)v; }
Iface *impl_v1_new(void) {
    static Iface i; i.get = v1_get; i.set = v1_set; return &i;
}
EOF

# client.c — 只依赖 interface.h
cat > client.c <<'EOF'
#include <stdio.h>
#include "interface.h"
int client_get(Iface *i) { return i->get(i); }
EOF

cat > main.c <<'EOF'
#include <stdio.h>
#include "interface.h"
extern Iface *impl_v1_new(void);
extern int    client_get(Iface *);
int main(void) {
    Iface *i = impl_v1_new();
    printf("v1=%d\n", client_get(i));
    return 0;
}
EOF

cc -std=c17 -Wall -Wextra -o app main.c client.c impl_v1.c
./app
echo "rc=$?"

# 现在改 impl_v2.c — 接口不变，client 不重编
cat > impl_v2.c <<'EOF'
#include "interface.h"
static int v2_get(Iface *s) { return s ? 200 : 0; }
static void v2_set(Iface *s, int v) { (void)s; (void)v; }
Iface *impl_v2_new(void) {
    static Iface i; i.get = v2_get; i.set = v2_set; return &i;
}
EOF

cat > main2.c <<'EOF'
#include <stdio.h>
#include "interface.h"
extern Iface *impl_v2_new(void);
extern int    client_get(Iface *);
int main(void) {
    Iface *i = impl_v2_new();
    printf("v2=%d\n", client_get(i));
    return 0;
}
EOF

# 只重编 main2 + impl_v2, client.c 不动
cc -std=c17 -Wall -Wextra -o app2 main2.c client.c impl_v2.c
./app2
echo "rc=$?"
```

**验收**：
- `./app` 输出 `v1=1`。
- `./app2` 输出 `v2=200`。
- `client.c` 没改也没重编 — 编译防火墙建立。

### A4 — 拆 lib: 静态库 + 头文件

```bash
mkdir -p /tmp/ch07-build && rm -rf /tmp/ch07-build/*
mkdir -p /tmp/ch07-build/libsub /tmp/ch07-build/appdir
cd /tmp/ch07-build

# 子库 libsub
cat > libsub/sub.h <<'EOF'
#ifndef SUB_H
#define SUB_H
int sub_compute(int x);
#endif
EOF
cat > libsub/sub.c <<'EOF'
int sub_compute(int x) { return x * 2 + 1; }
EOF

# 编译为静态库
cc -std=c17 -Wall -Wextra -c -o libsub/sub.o libsub/sub.c
ar rcs libsub/libsub.a libsub/sub.o

# app 包: 只依赖 sub.h
cat > appdir/main.c <<'EOF'
#include <stdio.h>
#include "sub.h"
int main(void) { printf("res=%d\n", sub_compute(20)); return 0; }
EOF

# 编译 app + 链接 libsub
cc -std=c17 -Wall -Wextra -I libsub -o appdir/app appdir/main.c libsub/libsub.a
./appdir/app
echo "rc=$?"

# 改 libsub 后只重编 libsub，不动 app 源
cat > libsub/sub.c <<'EOF'
int sub_compute(int x) { return x * 3; }   /* 行为变了 */
EOF
echo "=== 修改 libsub 后只重编 libsub + relink ==="
START=$(date +%s%N)
cc -std=c17 -Wall -Wextra -c -o libsub/sub.o libsub/sub.c
cc -std=c17 -Wall -Wextra -I libsub -o appdir/app appdir/main.c libsub/libsub.a
END=$(date +%s%N)
echo "rc=$?  elapsed_ms=$(( (END - START) / 1000000 ))"
./appdir/app   # 输出 res=60 (20*3)
```

**验收**：
- `./appdir/app` 第一次输出 `res=41`（20*2+1）。
- 改 libsub 后第二次输出 `res=60`（20*3）。
- 证明 libsub 内部修改不要求 app 源改动，演示 *库级隔离*。

## 六、值得深入思考的问题

### Q1: 拆包的"整体 build 略慢"怎么度量？

Feathers 说拆包后整体 build 略慢 5-15%。但实际项目里有人报告慢 50%。**关键问题**：是 Feathers 的数据基于理想实现（incremental link、ccache），还是 *通用* ？如何让团队接受这个 trade-off？

### Q2: Extract Interface 抽什么方法到接口？

AI 给的是 *plausible* — "所有 public 方法"。但真实工程里 5 个方法可能只有 1 个被 test 或 mock 调用。**关键问题**：*抽太多* vs *抽太少* 的判断准则是什么？测试覆盖范围？还是协作对象依赖？

### Q3: forward declaration 在 C++ 项目里的纪律

C++ 项目里的"少 include" 是纪律，不是工具。**关键问题**：如何在大团队里强制 *header 不该 transitive include* 的代码规范？靠 review？靠 lint？靠 include-what-you-use (IWYU)？

### Q4: AI 写新代码时是否"应该自动"考虑 build cascade？

AI 写 . cpp 时，是否有义务做 *依赖影响* 分析？**关键问题**：是否要把 *build cascade report* 写进 PR CI — 任何 PR 改 . h 必须给出 cascade 影响列表？

### Q5: pluginlib / 动态加载的 build time trade-off 何时值得？

Plugin 系统让 build 隔离，但 runtime 加载失败风险 + 类型检查 + 性能损耗。**关键问题**：什么时候 pluginlib 比 monolithic build 划算？这是产品工程 vs 性能工程 vs 部署工程的三角权衡。

### Q6: CI 全量 build 时间 vs 本地增量 build 时间的资源分配

CI 全量 30 分钟，本地增量 5 秒。CI 工程师想压全量（省钱），开发者想压增量（加速反馈）。**关键问题**：在 CI 是云端 ephemeral 资源的今天，是否还应该优化 *全量 build*？还是 *本地增量* 优先？

---

*下一章预告*: **Chapter 8 — How Do I Add a Feature?** —— 在 ch6（避开老代码改）+ ch7（让 build 快）的铺垫后，ch8 是 *正面添加 feature*：TDD 完整 cycle + Programming by Difference（继承添加）+ 何时从继承收敛回 composition。ch8 是 Part II 中 *"改动方法"* 章节的代表作 —— 教你"如何用 TDD 在 legacy 上加 feature"。